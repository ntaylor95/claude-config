# Implementation Plan: Phase 1 - Stamps Account Sync (Full Fan-Out Pattern)

**Goal:** Refactor the Stamps account sync job to use a fan-out pattern where the CronJob enqueues 170,000 individual messages (one per seller) for serial processing by multiple QueueService workers.

**Expected Result:** Job completes in ~14 hours (acceptable for nightly job), with zero risk since this matches the proven `FindCPPAccountsTask` pattern.

---

## Overview

**Current Problem:**
- Single message processes all 170k sellers serially in one handler = 94 hours
- All-or-nothing failure, no progress tracking, will timeout

**Phase 1 Solution:**
- CronJob queries for 170k seller IDs and enqueues 170k individual messages
- Each message contains ONE SellerProviderId
- Multiple QueueService workers process messages in parallel (5 workers recommended)
- Each worker processes messages serially (no concurrency per worker)
- Per-seller retry granularity, best observability

**Why Phase 1 (not jumping to Phase 2):**
- Zero risk - matches existing `FindCPPAccountsTask` daily task pattern
- Proven to work with SWSIM API (no rate limit concerns)
- Simplest implementation (no concurrency complexity)
- Best observability (see every seller in Hangfire dashboard)

---

## Changes Required

### 1. Create Modern.Data Repository for SellerProvider Data

**File:** `Modern.Data/Repositories/SellerProviderRepository.cs` (NEW FILE)

**Purpose:** Provide data access for the CronJob to query active Stamps seller IDs. CronJobs can only use Modern.Data, not SS.Data.

**Implementation:**
```csharp
using Dapper;
using Modern.Data.Utilities;

namespace Modern.Data.Repositories;

public interface ISellerProviderRepository
{
    /// <summary>
    /// Gets all active Stamps.com seller provider IDs for nightly sync.
    /// </summary>
    /// <returns>List of SellerProviderID values for active Stamps accounts.</returns>
    Task<List<int>> GetActiveStampsSellerProviderIds();
}

public class SellerProviderRepository : EnvironmentDapperRepositoryBase, ISellerProviderRepository
{
    public SellerProviderRepository(IEnvironmentConnectionStringProvider connectionStringProvider)
        : base(connectionStringProvider)
    {
    }

    public async Task<List<int>> GetActiveStampsSellerProviderIds()
    {
        return await ExecuteOnEnvironmentAsync(async connection =>
        {
            const string sql = @"
                SELECT sp.SellerProviderID
                FROM dbo.SellerProvider sp WITH (NOLOCK)
                INNER JOIN dbo.Seller s WITH (NOLOCK) ON sp.SellerID = s.SellerID
                WHERE s.CancelDate IS NULL
                  AND s.MaintenanceDate IS NULL
                  AND s.ExcludeReasonID IS NULL
                  AND sp.ProviderID = 2
                  AND sp.SellerProviderStatusID = 1";

            var results = await connection.QueryAsync<int>(sql);
            return results.AsList();
        },
        new[]
        {
            SqlRetryPolicies.RetryOnTimeoutAsyncPolicy,
            SqlRetryPolicies.RetryOnDeadlockAsyncPolicy
        });
    }
}
```

**Register in DI:**
- File: `Modern.Data/DependencyInjection/ServiceCollectionExtensions.cs`
- Add to `AddTransientModernData()` method around line 49:
  ```csharp
  services.AddTransient<ISellerProviderRepository, SellerProviderRepository>();
  ```

---

### 2. Modify Message to Support Single-Seller Processing

**File:** `V3QueueClient/Messages/SyncStampsAccountProgramDetailsMessage.cs` (MODIFY)

**Current:**
```csharp
public class SyncStampsAccountProgramDetailsMessage : IMessageType
{
    public string MessageKey => QueueConstants.MessageKey.SyncStampsAccountProgramDetails;

    public int SellerId { get; set; } = 0;  // 0 = process all sellers
    public Guid UserId { get; set; } = Guid.Empty;
    public int BatchSize { get; set; } = 100;
}
```

**Change to:**
```csharp
public class SyncStampsAccountProgramDetailsMessage : IMessageType
{
    public string MessageKey => QueueConstants.MessageKey.SyncStampsAccountProgramDetails;

    public int SellerId { get; set; } = 0;  // Required by IMessageType
    public Guid UserId { get; set; } = Guid.Empty;  // Required by IMessageType

    /// <summary>
    /// The specific SellerProviderID to sync. Each message processes ONE seller.
    /// </summary>
    public int SellerProviderId { get; set; }
}
```

**Notes:**
- Remove `BatchSize` property (no longer needed for single-seller processing)
- Add `SellerProviderId` property to identify which seller to process
- Keep `SellerId` and `UserId` for IMessageType interface compliance

---

### 3. Refactor Action Handler to Process Single Seller

**File:** `V3Core/AsyncActions/MessageAction/SyncStampsAccountProgramDetailsAction.cs` (MODIFY)

**Current Implementation:** Fetches ALL 170k sellers and processes them in a loop

**Change to:** Process only the ONE seller specified in the message

**New Implementation:**
```csharp
public class SyncStampsAccountProgramDetailsAction : MessageActionBase<SyncStampsAccountProgramDetailsMessage>
{
    private readonly ILogger log = Log.ForContext<SyncStampsAccountProgramDetailsAction>();
    private readonly ISellerProviderRepository sellerProviderRepository;
    private readonly IStampsAccountProgramDetailsUpdater accountProgramDetailsUpdater;

    public SyncStampsAccountProgramDetailsAction(
        ISellerProviderRepository sellerProviderRepository,
        IStampsAccountProgramDetailsUpdater accountProgramDetailsUpdater)
    {
        this.sellerProviderRepository = sellerProviderRepository;
        this.accountProgramDetailsUpdater = accountProgramDetailsUpdater;
    }

    protected override async Task<IMessageActionResult> Execute(SyncStampsAccountProgramDetailsMessage message)
    {
        log.Debug("Processing SellerProviderId {SellerProviderId}", message.SellerProviderId);

        try
        {
            // Step 1: Get THIS seller's provider
            var sellerProvider = await sellerProviderRepository.GetSellerProviderAsync(message.SellerProviderId);

            if (sellerProvider == null)
            {
                log.Warning("SellerProvider {SellerProviderId} not found", message.SellerProviderId);
                return new MessageActionResult(true); // Don't retry - seller doesn't exist
            }

            // Step 2: Get account info from SWSIM for THIS seller
            var sdcService = (SDCService)PartnerFactory.GetShippingProvider(sellerProvider);

            // CRITICAL: Use async version to avoid blocking threads
            var accountInfo = await sdcService.GetAccountInfoAsync();

            // Step 3: Build and update details for THIS seller
            var details = accountProgramDetailsUpdater.BuildAccountProgramDetails(sellerProvider, accountInfo);

            if (details != null)
            {
                accountProgramDetailsUpdater.UpdateAccountProgramDetails(details);
                log.Debug("Updated SellerProviderId {SellerProviderId} with RateSetType {RateSetType}",
                    message.SellerProviderId, details.RateSetType);
            }
            else
            {
                log.Warning("No account program details built for SellerProviderId {SellerProviderId}",
                    message.SellerProviderId);
            }

            return new MessageActionResult(true);
        }
        catch (StampsAuthenticationException ex)
        {
            // Authentication failure - don't retry
            log.Warning(ex, "Authentication failed for SellerProviderId {SellerProviderId}", message.SellerProviderId);
            return new MessageActionResult(true); // Mark as success to prevent retry
        }
        catch (Exception ex)
        {
            log.Error(ex, "Error syncing SellerProviderId {SellerProviderId}", message.SellerProviderId);
            throw; // Let QueueService retry this ONE seller
        }
    }
}
```

**Key Changes:**
- Remove `GetActiveStampsSellerProviders()` call (no longer fetches all)
- Remove foreach loop (processes single seller)
- Use `message.SellerProviderId` to identify which seller to process
- Call `GetAccountInfoAsync()` (async version, not blocking)
- Update single seller's settings
- Authentication errors don't retry (prevents infinite retry loop)
- Other errors throw for QueueService retry

**New Dependency Required:**
Need to add `GetSellerProviderAsync(int sellerProviderId)` method to `ISellerProviderRepository` in SS.Data (existing repository, not Modern.Data).

---

### 4. Update CronJob to Fan-Out Messages

**File:** `ScheduledJobs.Shared/Jobs/SyncStampsAccountStatusCronJob.cs` (MODIFY)

**Current Implementation:** Enqueues single message with SellerId=0

**Change to:** Query all seller IDs and enqueue one message per seller

**New Implementation:**
```csharp
using Serilog;
using V3QueueClient.Messages;
using V3QueueClient.Queue;
using Modern.Data.Repositories;  // NEW: Modern.Data dependency

namespace ScheduledJobs.Shared.Jobs;

public interface ISyncStampsAccountStatusCronJob : ICronJob
{
    Task Run();
}

/// <summary>
/// Cron job for syncing Stamps.com account status information to SellerProviderSettings.
/// Queries all active Stamps accounts and enqueues one message per account.
/// </summary>
public class SyncStampsAccountStatusCronJob : ISyncStampsAccountStatusCronJob
{
    private static readonly ILogger Logger = Log.ForContext<SyncStampsAccountStatusCronJob>();
    private readonly IAsyncAction asyncAction;
    private readonly ISellerProviderRepository sellerProviderRepository;  // NEW: Modern.Data repository

    public SyncStampsAccountStatusCronJob(
        IAsyncAction asyncAction,
        ISellerProviderRepository sellerProviderRepository)  // NEW: Inject Modern.Data repository
    {
        this.asyncAction = asyncAction;
        this.sellerProviderRepository = sellerProviderRepository;
    }

    public string CronJobId { get; set; } = "SyncStampsAccountStatusCronJob";

    /// <summary>
    /// Scheduled to run at 2:00 AM PST daily.
    /// </summary>
    public Func<string> CronExpression { get; set; } = () => "0 2 * * *"; // 2:00 AM PST daily

    public async Task Run()
    {
        Logger.Information("SyncStampsAccountStatusCronJob: Starting - fetching active Stamps seller IDs");

        try
        {
            // Step 1: Get all active Stamps seller provider IDs from Modern.Data repository
            var sellerProviderIds = await sellerProviderRepository.GetActiveStampsSellerProviderIds();

            Logger.Information("SyncStampsAccountStatusCronJob: Found {Count} active Stamps accounts to sync",
                sellerProviderIds.Count);

            if (sellerProviderIds.Count == 0)
            {
                Logger.Warning("SyncStampsAccountStatusCronJob: No active Stamps accounts found");
                return;
            }

            // Step 2: Fan-out - enqueue ONE message per seller
            var enqueueCount = 0;
            foreach (var sellerProviderId in sellerProviderIds)
            {
                await asyncAction.Queue(new SyncStampsAccountProgramDetailsMessage
                {
                    SellerProviderId = sellerProviderId,
                    SellerId = 0,  // Required by IMessageType
                    UserId = Guid.Empty  // Required by IMessageType
                });
                enqueueCount++;

                // Log progress every 10k messages
                if (enqueueCount % 10000 == 0)
                {
                    Logger.Information("SyncStampsAccountStatusCronJob: Enqueued {Count}/{Total} messages",
                        enqueueCount, sellerProviderIds.Count);
                }
            }

            Logger.Information("SyncStampsAccountStatusCronJob: Successfully enqueued {Count} messages for processing",
                enqueueCount);
        }
        catch (Exception ex)
        {
            Logger.Error(ex, "SyncStampsAccountStatusCronJob: Error syncing Stamps.com account statuses");
            throw; // Let Hangfire handle the retry
        }
    }
}
```

**Key Changes:**
- Inject `ISellerProviderRepository` from Modern.Data
- Call `GetActiveStampsSellerProviderIds()` to get list of seller IDs
- Loop through IDs and enqueue ONE message per seller
- Log progress every 10k messages (helps monitor enqueue phase)
- Total enqueue time: ~2-5 minutes for 170k messages

---

### 5. Add GetSellerProviderAsync Method to SS.Data Repository

**File:** `SS.Data/Repositories/ISellerProviderRepository.cs` (MODIFY)

Add method signature:
```csharp
Task<SellerProvider> GetSellerProviderAsync(int sellerProviderId);
```

**File:** `SS.Data/Repositories/SellerProviderRepository.cs` (MODIFY)

Add implementation:
```csharp
public async Task<SellerProvider> GetSellerProviderAsync(int sellerProviderId)
{
    const string sqlQuery = @"
        SELECT sp.*
        FROM dbo.SellerProvider sp WITH (NOLOCK)
        WHERE sp.SellerProviderID = @sellerProviderId";

    return await ExecuteOnShipStationAsync(connection =>
        connection.QueryFirstOrDefaultAsync<SellerProvider>(
            sqlQuery,
            new { sellerProviderId }));
}
```

**Purpose:** The action handler needs to fetch the full SellerProvider object to create the SDCService.

---

### 6. Create GetAccountInfoAsync Wrapper (If Not Present)

**File:** `SS.Business/Shipping/SDCManager.cs` (CHECK/ADD)

**Check if exists:** Search for `GetAccountInfoAsync` method in SDCManager.cs

**If NOT present, add:**
```csharp
/// <summary>
/// Async version of GetAccountInfo for non-blocking SWSIM API calls.
/// </summary>
public virtual async Task<SdcAccountInfo> GetAccountInfoAsync()
{
    try
    {
        var client = stampsCarrierConfigSettingResolver.GetSoapClient();
        var credentials = GetCredentials();

        var request = new GetAccountInfoRequest
        {
            Authenticator = credentials
        };

        var response = await client.GetAccountInfoAsync(request);

        var sdcAccountInfo = new SdcAccountInfo
        {
            AccountInfo = response.AccountInfo,
            Address = response.Address,
            Email = response.Email,
            AccountStatus = response.AccountStatus,
            DateAdvanceConfig = response.DateAdvanceConfig,
            VerificationPhoneNumber = response.VerificationPhoneNumber,
            VerificationPhoneNumberExtension = response.VerificationPhoneNumberExtension
        };

        if (sdcAccountInfo.AccountInfo == null)
        {
            Log.Error($"Account info null for SellerId: {SellerProvider?.SellerID} SellerProviderId: {SellerProvider?.SellerProviderID}");
            throw new ShippingProviderException($"Account info not populated on GetAccountInfo call");
        }

        // Fire off a task to try to save the stamps.com CustomerId
        if (!SellerProvider.SellerProviderSettings?.Any(x => x.Name == StampsConstants.CustomerIdSetting) ?? true)
        {
            Task.Run(() => TrySaveCustomerId(SellerProvider.SellerID, SellerProvider.SellerProviderID, sdcAccountInfo.AccountInfo.CustomerID));
        }

        if (!string.IsNullOrWhiteSpace(sdcAccountInfo?.AccountInfo?.AccountType) &&
            (!SellerProvider.SellerProviderSettings?.Any(x => x.Name == StampsConstants.SDCAccountTypeSetting) ?? true))
        {
            Task.Run(() => TrySaveSDCAccountType(SellerProvider.SellerID, SellerProvider.SellerProviderID, sdcAccountInfo.AccountInfo.AccountType));
        }

        return sdcAccountInfo;
    }
    catch (Exception ex)
    {
        if (ex.Message.StartsWith("Authentication failed"))
        {
            throw new StampsAuthenticationException(Provider.Stamps_com, SellerProvider?.SellerProviderID,
                "Your Stamps.com username or password are invalid", ex);
        }

        if (ex.Message.Contains("Registration in Progress"))
        {
            throw new StampsRegistrationInProgressException();
        }

        throw;
    }
}
```

**Note:** The async SOAP proxy method `GetAccountInfoAsync` exists in `SS.Proxies.SDC.SwsimV157SoapClient` - we're just wrapping it.

---

## Files to Modify/Create

### NEW Files:
1. `Modern.Data/Repositories/SellerProviderRepository.cs` - New repository for Modern.Data

### MODIFIED Files:
1. `V3QueueClient/Messages/SyncStampsAccountProgramDetailsMessage.cs` - Add SellerProviderId property
2. `V3Core/AsyncActions/MessageAction/SyncStampsAccountProgramDetailsAction.cs` - Process single seller
3. `ScheduledJobs.Shared/Jobs/SyncStampsAccountStatusCronJob.cs` - Fan-out to 170k messages
4. `SS.Data/Repositories/ISellerProviderRepository.cs` - Add GetSellerProviderAsync method
5. `SS.Data/Repositories/SellerProviderRepository.cs` - Implement GetSellerProviderAsync
6. `Modern.Data/DependencyInjection/ServiceCollectionExtensions.cs` - Register new repository
7. `SS.Business/Shipping/SDCManager.cs` - Add GetAccountInfoAsync wrapper (if not present)

### NO CHANGES NEEDED:
- `V3Core/AsyncActions/InjectableMessageActionResolver.cs` - Message routing already registered
- `ScheduledJobs.Shared/ScheduledJobsSharedBuilderExtension.cs` - CronJob already registered
- `ScheduledJobs.Client/Hangfire/HangfireBuilderExtensions.cs` - Already registered

---

## Testing Strategy

### Unit Tests

**Test File:** `V3Core.Tests/AsyncActions/MessageAction/SyncStampsAccountProgramDetailsActionTests.cs` (if exists, modify)

**Test Cases:**
1. `Execute_ValidSellerProviderId_CallsGetAccountInfoAndUpdatesSettings`
2. `Execute_SellerProviderNotFound_ReturnsSuccessWithoutRetry`
3. `Execute_AuthenticationFailure_ReturnsSuccessWithoutRetry`
4. `Execute_UnexpectedException_ThrowsForRetry`
5. `Execute_NullAccountDetails_LogsWarningAndReturnsSuccess`

**Mock Dependencies:**
- `ISellerProviderRepository.GetSellerProviderAsync()` - Return test SellerProvider
- `SDCService.GetAccountInfoAsync()` - Return test account info
- `IStampsAccountProgramDetailsUpdater.BuildAccountProgramDetails()` - Return test details
- `IStampsAccountProgramDetailsUpdater.UpdateAccountProgramDetails()` - Verify called

### Integration Testing (Manual)

**Phase 1: Local Development**
1. Query for 10 test Stamps sellers from local DB
2. Modify cron job to process only those 10 sellers
3. Run cron job manually, verify 10 messages enqueued
4. Start QueueService worker, verify 10 messages processed
5. Check SellerProviderSettings table for updates

**Phase 2: Staging Environment**
1. Deploy to staging
2. Trigger cron job manually (don't wait for 2am)
3. Monitor Hangfire dashboard:
   - Verify messages enqueue successfully (~2-5 min for 170k)
   - Watch "Processing" count increase
   - Check "Succeeded" and "Failed" tabs
4. Sample check: Verify 10-20 random sellers have updated settings
5. Monitor logs for errors

**Phase 3: Production Rollout**
1. Deploy on a Friday (allows weekend monitoring)
2. Let it run overnight at 2am
3. Check next morning:
   - Hangfire dashboard: Success/failure counts
   - Logs: Any error patterns
   - Database: Random sample of updated settings
4. Monitor for 3 nights before declaring success

---

## Monitoring & Observability

### Hangfire Dashboard
- URL: `http://localhost:9002/hangfire` (local) or environment-specific URL
- **Recurring Jobs** tab: See the cron job schedule
- **Processing** tab: See active message processing (expect ~150-500 at a time)
- **Succeeded** tab: Count of successfully processed sellers
- **Failed** tab: Count of failed sellers (investigate if > 1%)
- **Scheduled** tab: Messages waiting to execute

### Logs to Monitor
```bash
# CronJob logs (enqueue phase)
grep "SyncStampsAccountStatusCronJob" logs/scheduled-jobs-client.log

# Action handler logs (processing phase)
grep "SyncStampsAccountProgramDetailsAction" logs/queueservice.log

# Look for error patterns
grep "Error syncing SellerProviderId" logs/queueservice.log
```

### Success Metrics
- **Enqueue phase:** All 170k messages enqueued in < 5 minutes
- **Processing phase:** ~14 hours to complete (10 messages/min per worker × 5 workers = 50/min = ~57 hours... adjust workers as needed)
- **Error rate:** < 1% (< 1,700 failed sellers)
- **Data accuracy:** Random sample shows RateSetType updated correctly

---

## Rollback Plan

**If error rate > 5% or critical issues:**
1. Stop cron job: Disable in Hangfire dashboard
2. Investigate failed messages in Hangfire "Failed" tab
3. Check logs for error patterns
4. Fix underlying issue (authentication, SWSIM API, DB connectivity)
5. Re-run failed messages: Hangfire "Requeue" button

**If error rate > 10%:**
1. Rollback code changes
2. Restore original single-message implementation
3. Investigate SWSIM API issues or connection problems

---

## Timeline Estimate

**Development:** 2-3 hours
- Create Modern.Data repository (30 min)
- Modify message and action handler (1 hour)
- Update cron job (30 min)
- Add async wrapper if needed (30 min)
- Write unit tests (30 min)

**Testing:** 2 hours
- Local testing with 10 sellers (30 min)
- Staging testing with full load (1 hour)
- Code review and adjustments (30 min)

**Deployment:** 1 hour
- Deploy to staging
- Monitor first run
- Deploy to production

**Total:** 5-6 hours

---

## Success Criteria (Phase 1 Complete)

- [ ] CronJob enqueues 170,000 individual messages successfully
- [ ] Hangfire dashboard shows all messages
- [ ] QueueService workers process messages without errors
- [ ] Error rate < 1% (< 1,700 failed sellers)
- [ ] Job completes in ~14 hours
- [ ] SellerProviderSettings table shows updated RateSetType values
- [ ] No SWSIM API rate limit errors
- [ ] No database performance issues
- [ ] Job runs successfully for 3 consecutive nights

**After Phase 1 success, proceed to Phase 2** (add concurrency gradually: 10 → 20 → 30 → 50 concurrent calls per worker)

---

## Notes

- **Why not Phase 2 immediately?** Risk. We don't know SWSIM's rate limits. Phase 1 matches the proven `FindCPPAccountsTask` pattern.
- **14 hours acceptable?** Yes, for nightly job running at 2am PST, completes by 4pm PST next day.
- **Memory concerns?** None - single seller per message, not loading all 170k into memory.
- **Database concerns?** Existing batch update logic handles this (100 updates per batch).
- **What if it takes longer than 14 hours?** Add more QueueService workers (5 → 10 → 20).

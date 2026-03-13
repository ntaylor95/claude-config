# Spec: Seller Recommendations Endpoint
*Generated: 2026-03-06*

## Intent

A production-grade seller recommendations API is shipped — replacing the hackathon BigQuery prototype — backed by DynamoDB, following V3 conventions, with a typed metadata system extensible by any engineer without modifying existing code.

---

## Context

**Trigger:** Engineer invokes the executor agent against this spec on branch `SPD/24707-SELLER_RECOMMENDATIONS_ENDPOINT`.

**Starting state:**
- Hackathon PR #25200 exists as reference only (BigQuery implementation — do not port)
- `V3Core/Recommendations/` and `WebV3Api/Seller/SellerRecommendationsController.cs` do not yet exist
- `AWSSDK.DynamoDBv2 v3.7.405.13` already referenced in `V3Core.csproj` — no new packages needed
- Pattern reference for DynamoDB repository: `V3Core/Rates/Services/Recommendations/AlternativeRatesRepository.cs`
- `IDynamoDbClient` interface: `SS.Business/DynamoDB/IDynamoDbClient.cs`
- Nearest controller example: any file in `WebV3Api/Seller/` (e.g. `SellerBrandsController.cs`)

**Constraints:**
- .NET Framework 4.8 — no modern C# features beyond what's already used in the project
- NSubstitute only for mocking — never Moq
- FluentValidation 6.4.1 for any request validation
- `sellerId` must always come from `ShipstationIdentity` — never a route parameter
- No feature flag — endpoint goes live to all sellers on deploy
- No server-side caching — UI loads into Redux once on app init (one call per session)
- Do NOT bump `Microsoft.Extensions.DependencyInjection` (hackathon bumped to v10 — revert to 2.2.0)
- Table name from `ConfigurationManager.AppSettings[AppConstants.ConfigurationKeys.SellerRecommendationsTableName]`; local/dev value = `seller-recommendations-dev`
- All controllers inherit from `BaseApiController` (which inherits `BaseController` with `[Authorize]`)
- Serilog static logger: `private static readonly ILogger Log = Serilog.Log.ForContext<T>()`
- Use `WITH (NOLOCK)` on all SQL SELECT queries (not applicable here — DynamoDB only)
- All string comparisons use `StringComparison.OrdinalIgnoreCase`

---

## Success Criteria

Opus self-evaluates against all of the following before handing off to reviewer:

1. `GET /api/seller/recommendations` returns `200` with `{ "recommendations": [] }` for a seller with no DynamoDB records
2. `POST /api/seller/recommendations/{recommendationType}/dismissed` returns `204`, and the DynamoDB record reflects `isDismissed=true`, incremented `dismissedCount`, and a populated `dismissedAt` UTC timestamp
3. `POST /api/seller/recommendations/{recommendationType}/actioned` returns `204`, and the DynamoDB record reflects `isActioned=true` and a populated `actionedAt` UTC timestamp
4. Cross-seller requests are impossible — `sellerId` is sourced exclusively from `ShipstationIdentity`, never from caller input
5. Solution builds clean via `./localdev/remote-build.sh "V3Core/V3Core.csproj"` and `./localdev/remote-build.sh "WebV3Api/WebV3Api.csproj"`
6. `SellerRecommendationsServiceTests` pass via `./localdev/remote-test.sh "SellerRecommendationsServiceTests"`
7. `EnableSmartFillWeight` is implemented as the working example: enum value, `EnableSmartFillWeightMetadata` class with `OrderCount30d` (int) and `MatchingShipmentCount` (int) properties, registered in `RecommendationMetadataResolver`, and a GET response correctly deserializes its `metadata` JSON to that typed model
8. Adding a future recommendation type requires only: (a) new enum value, (b) new metadata class inheriting `RecommendationMetadata`, (c) one line in `RecommendationMetadataResolver` — zero changes to controller, service, or repository

---

## Failure Modes

| Trigger | Action |
|---------|--------|
| `isDismissed`, `isActioned`, or `dismissedCount` absent from DynamoDB record (newly seeded records) | Default to `false`, `false`, `0` via C# property defaults on `SellerRecommendationRecord` — not an error |
| DynamoDB query throws any exception on GET | Log error with `sellerId`, return `200` with `{ "recommendations": [] }` — UI loads cleanly |
| DynamoDB write throws any exception on dismiss/action | Log error with `sellerId` + `recommendationType`, return `500` |
| DynamoDB record not found on dismiss/action | Silently succeed — return `204` (idempotent) |
| `recommendationType` route param does not parse to a valid `RecommendationType` enum value | Return `400` |
| Metadata JSON in DynamoDB record is null or empty | Set `Metadata = null` on response object, log warning — do not fail the recommendation |
| `RecommendationMetadataResolver` has no mapping for a parsed type | Log warning with `recommendationType` + `sellerId`, set `Metadata = null` on the recommendation — do not fail the recommendation |

---

## Task Decomposition

Execute steps in order. Each step produces files on disk. Next step reads those files as input.

| # | Step | What it does | Capability | Agent | Input | Output |
|---|------|-------------|-----------|-------|-------|--------|
| 1 | Add config key | Add `SellerRecommendationsTableName = "SellerRecommendationsTableName"` inside `AppConstants.ConfigurationKeys` | code | executor | `SS.Configuration/AppConstants.cs` | updated `AppConstants.cs` |
| 2 | DynamoDB record model | Create `SellerRecommendationRecord` POCO with all table fields; `IsDismissed = false`, `IsActioned = false`, `DismissedCount = 0` as C# defaults | code | executor | DynamoDB table schema | `V3Core/Recommendations/DataAccess/Models/SellerRecommendationRecord.cs` |
| 3 | RecommendationType enum | Create string-backed enum with `[JsonConverter(typeof(StringEnumConverter))]`; add `EnableSmartFillWeight` as first value | code | executor | — | `V3Core/Recommendations/Models/RecommendationType.cs` |
| 4 | Metadata base + example | Create `RecommendationMetadata` abstract base class; create `EnableSmartFillWeightMetadata : RecommendationMetadata` with `OrderCount30d` (int) and `MatchingShipmentCount` (int) | code | executor | Step 3 output | `V3Core/Recommendations/Models/Metadata/RecommendationMetadata.cs`, `EnableSmartFillWeightMetadata.cs` |
| 5 | Metadata resolver | Create `RecommendationMetadataResolver` with static dictionary mapping `RecommendationType` → `System.Type`; register `EnableSmartFillWeight → EnableSmartFillWeightMetadata` | code | executor | Steps 3–4 output | `V3Core/Recommendations/Models/Metadata/RecommendationMetadataResolver.cs` |
| 6 | Domain model | Create `SellerRecommendation` with `RecommendationType`, `string RawMetadata`, and resolved `RecommendationMetadata Metadata` | code | executor | Steps 2–4 output | `V3Core/Recommendations/Models/SellerRecommendation.cs` |
| 7 | Repository interface + impl | Create `ISellerRecommendationsRepository` and `SellerRecommendationsRepository` using `IDynamoDbClient`; GET uses `QueryTable<SellerRecommendationRecord>` by `sellerId`; dismiss/action uses `GetRecordAsync` (hashKey=sellerId, rangeKey=recommendationType, `saveHashKeyAsNumeric: false`) then `PutItemAsync` | code | executor | Steps 1–2 output, `IDynamoDbClient` | `V3Core/Recommendations/DataAccess/ISellerRecommendationsRepository.cs`, `SellerRecommendationsRepository.cs` |
| 8 | Service interface + impl | Create `ISellerRecommendationsService` and `SellerRecommendationsService`; maps records to domain models using `RecommendationMetadataResolver`; handles all failure modes from Section 4 | code | executor | Steps 5–7 output | `V3Core/Recommendations/Services/ISellerRecommendationsService.cs`, `SellerRecommendationsService.cs` |
| 9 | Public contracts | Create `GetRecommendationsResponse` (wraps list) and `RecommendationResponse` in `WebV3Api/PublicContracts/Recommendations/`; `RecommendationResponse` exposes all record state: `string RecommendationType`, `object Metadata`, `bool IsDismissed`, `bool IsActioned`, `int DismissedCount`, `DateTime? DismissedAt`, `DateTime? ActionedAt` — no V3Core type references | code | executor | Step 6 output | `GetRecommendationsResponse.cs`, `RecommendationResponse.cs` |
| 10 | Controller | Create `SellerRecommendationsController` in `WebV3Api/Seller/`; route `api/seller/recommendations`; GET returns `200`/empty list; POST dismiss + POST actioned return `204`; `sellerId` from `identity.SellerId` only | code | executor | Steps 8–9 output | `WebV3Api/Seller/SellerRecommendationsController.cs` |
| 11 | DI registration | Add `RegisterRecommendationsServices(services)` call in `DIConfig.cs` main method; implement private method registering repo and service as `AddSingleton` | code | executor | Steps 7–8 output | updated `WebV3Api/DependencyInjection/DIConfig.cs` |
| 12 | Unit tests | Write `SellerRecommendationsServiceTests` covering: happy path GET, empty result, DynamoDB exception on GET returns empty list, record-not-found on dismiss returns without error, metadata deserialization to `EnableSmartFillWeightMetadata`, unknown type falls back to dictionary | code | tester | Steps 6–8 output | `V3Core/Recommendations/Tests/SellerRecommendationsServiceTests.cs` |
| 13 | README | Write `V3Core/Recommendations/README.md` documenting how to add a new recommendation type (enum → metadata class → resolver registration), with `EnableSmartFillWeight` as the worked example | write | executor | All above | `V3Core/Recommendations/README.md` |
| 14 | Review | Verify all success criteria; check all files against `CLAUDE.md` conventions (NSubstitute, no Moq, no inline validation, singleton DI lifetimes, Serilog pattern, no commented-out code) | analyze | reviewer | All files | review report |

---

## Decision Points

```
IF recommendationType string from route does not parse to a valid RecommendationType enum value
  THEN return 400
  ELSE proceed

IF RecommendationMetadataResolver has no mapping for the parsed type
  THEN log warning with recommendationType + sellerId, set Metadata = null
  ELSE deserialize to registered concrete type

IF metadata JSON in DynamoDB record is null or empty
  THEN set Metadata = null on response and log warning
  ELSE deserialize using resolver

IF DynamoDB GET throws any exception
  THEN log error with sellerId and return 200 with empty recommendations list
  ELSE map records to domain models and return

IF DynamoDB write throws any exception (dismiss/action)
  THEN log error with sellerId + recommendationType and return 500
  ELSE return 204

IF DynamoDB GetRecord returns null on dismiss/action (record not found)
  THEN silently return 204 (idempotent — do not error)
  ELSE apply mutation and write back via PutItemAsync

IF SellerRecommendationRecord is missing isDismissed / isActioned / dismissedCount fields
  THEN C# property defaults (false / false / 0) apply automatically — treat as new record
```

---

## Handoff Protocol

**Between steps:** Each executor step produces `.cs` files written to disk on branch `SPD/24707-SELLER_RECOMMENDATIONS_ENDPOINT` at `/Users/nicole.taylor/Documents/Repos/shipstation`. Next step reads those files directly — no in-memory passing between agents.

**Step completion trigger:** A step is complete when its output file(s) exist and the project compiles. Build verification: `./localdev/remote-build.sh`.

**Test trigger:** Tester agent runs after Step 12 (all implementation files complete).
Command: `./localdev/remote-test.sh "SellerRecommendationsServiceTests"`
Failure loops back to executor with the failing test output.

**Review trigger:** Reviewer runs after tests pass.
Checks: all new files against `CLAUDE.md` conventions. Required changes loop back to executor. Approval proceeds to final output.

**Final deliverable:**

```
New files (13):
  SS.Configuration/AppConstants.cs                                          (modified)
  V3Core/Recommendations/DataAccess/Models/SellerRecommendationRecord.cs
  V3Core/Recommendations/DataAccess/ISellerRecommendationsRepository.cs
  V3Core/Recommendations/DataAccess/SellerRecommendationsRepository.cs
  V3Core/Recommendations/Models/RecommendationType.cs
  V3Core/Recommendations/Models/SellerRecommendation.cs
  V3Core/Recommendations/Models/Metadata/RecommendationMetadata.cs
  V3Core/Recommendations/Models/Metadata/EnableSmartFillWeightMetadata.cs
  V3Core/Recommendations/Models/Metadata/RecommendationMetadataResolver.cs
  V3Core/Recommendations/Services/ISellerRecommendationsService.cs
  V3Core/Recommendations/Services/SellerRecommendationsService.cs
  V3Core/Recommendations/Tests/SellerRecommendationsServiceTests.cs
  V3Core/Recommendations/README.md
  WebV3Api/PublicContracts/Recommendations/GetRecommendationsResponse.cs
  WebV3Api/PublicContracts/Recommendations/RecommendationResponse.cs
  WebV3Api/Seller/SellerRecommendationsController.cs
  WebV3Api/DependencyInjection/DIConfig.cs                                  (modified)

Branch: SPD/24707-SELLER_RECOMMENDATIONS_ENDPOINT
State: builds clean, tests green, reviewer approved — ready for PR against master
```

---

*Spec Version 1.0 — Ready for Opus 4.6*

# Seller Recommendations Endpoint

## Context

A hackathon PR (#25200) prototyped a recommendations API backed by BigQuery. This productionizes that feature with proper conventions: correct controller location, DynamoDB as the data store (table `seller-recommendations-<env>`), typed polymorphic metadata per recommendation type, and a developer guide for extending recommendations over time.

---

## Endpoints

```
GET  /api/seller/recommendations                              → all recommendations for authenticated seller
POST /api/seller/recommendations/{recommendationType}/dismissed → mark as dismissed, increment count, set timestamp
POST /api/seller/recommendations/{recommendationType}/actioned  → mark as actioned, set timestamp
```

- `sellerId` always comes from `ShipstationIdentity` (never a route param)
- GET returns all records regardless of dismissed/actioned state
- POST operations return `204 No Content`

---

## Files to Create

### V3Core

```
V3Core/Recommendations/
├── DataAccess/
│   ├── ISellerRecommendationsRepository.cs
│   ├── SellerRecommendationsRepository.cs
│   └── Models/
│       └── SellerRecommendationRecord.cs       ← DynamoDB POCO
├── Models/
│   ├── RecommendationType.cs                   ← string-backed enum
│   ├── SellerRecommendation.cs                 ← domain model
│   └── Metadata/
│       └── RecommendationMetadata.cs           ← abstract base class
├── Services/
│   ├── ISellerRecommendationsService.cs
│   └── SellerRecommendationsService.cs
├── Tests/
│   └── SellerRecommendationsServiceTests.cs
└── README.md                                   ← how to add a new recommendation type
```

### WebV3Api

```
WebV3Api/
├── Seller/
│   └── SellerRecommendationsController.cs
└── PublicContracts/
    └── Recommendations/
        ├── GetRecommendationsResponse.cs
        └── RecommendationResponse.cs
```

---

## Files to Modify

| File | Change |
|------|--------|
| `SS.Configuration/AppConstants.cs` | Add `SellerRecommendationsTableName = "SellerRecommendationsTableName"` inside `ConfigurationKeys` |
| `WebV3Api/DependencyInjection/DIConfig.cs` | Add `RegisterRecommendationsServices(services)` call + private method |
| `WebV3Api/appsettings.json` (or equivalent local config) | Add `"SellerRecommendationsTableName": "seller-recommendations-dev"` |

> Note: `AWSSDK.DynamoDBv2` is already in `V3Core.csproj` — no new NuGet packages needed. Do NOT bump `Microsoft.Extensions.DependencyInjection` to v10 as the hackathon did.

---

## Key Implementation Details

### DynamoDB Table Schema
- Table: `seller-recommendations-<env>`
- PK: `sellerId` (String)
- SK: `recommendationType` (String)
- Attributes: `createdAt`, `metadata` (JSON string), `syncedAt`, `dismissedCount`, `isDismissed`, `isActioned`
- GSI: `seller-visibility-index` (sellerId + visibilitySortKey) — not used in v1

### SellerRecommendationRecord (DynamoDB POCO)
```csharp
// V3Core/Recommendations/DataAccess/Models/SellerRecommendationRecord.cs
[JsonProperty] attributes on all fields.
Fields: SellerId, RecommendationType, CreatedAt, Metadata (string - raw JSON),
        SyncedAt, DismissedCount, IsDismissed, IsActioned, DismissedAt, ActionedAt
```

### Repository Pattern
Follows `AlternativeRatesRepository` (`V3Core/Rates/Services/Recommendations/AlternativeRatesRepository.cs`):
- Inject `IDynamoDbClient` (from `SS.Business/DynamoDB/IDynamoDbClient.cs`)
- Table name from `ConfigurationManager.AppSettings[AppConstants.ConfigurationKeys.SellerRecommendationsTableName]`
- **GET all**: `QueryTable<SellerRecommendationRecord>(tableName, new QueryOperationConfig { Filter = new QueryFilter("sellerId", QueryOperator.Equal, sellerId.ToString()) })`
- **GET one** (for update): `GetRecordAsync<SellerRecommendationRecord>(tableName, sellerId.ToString(), recommendationType, null, saveHashKeyAsNumeric: false)`
- **Write back**: `PutItemAsync(tableName, updatedRecord)` — read-modify-write pattern for dismiss/action

### RecommendationType Enum
```csharp
// V3Core/Recommendations/Models/RecommendationType.cs
[JsonConverter(typeof(StringEnumConverter))]
public enum RecommendationType
{
    // Values added here as recommendations are built out
    // Example: SetupInventory, EnableAutomation
}
```

### Metadata Polymorphism
```
RecommendationMetadata (abstract base) — V3Core/Recommendations/Models/Metadata/RecommendationMetadata.cs
  └── ConcreteTypeMetadata : RecommendationMetadata  ← added per recommendation type

RecommendationMetadataResolver (static) — maps RecommendationType → System.Type
Service uses: JsonConvert.DeserializeObject(record.Metadata, resolver.Resolve(type))
```
Response contract `RecommendationResponse.Metadata` is typed as `RecommendationMetadata` — serializes to the concrete runtime type's properties automatically.

### Controller
```csharp
// WebV3Api/Seller/SellerRecommendationsController.cs
[Route("api/seller/recommendations")]
public class SellerRecommendationsController : BaseApiController
{
    // Inject: ShipstationIdentity, ISellerRecommendationsService
    // GET → identity.SellerId, no route param
    // POST /{recommendationType}/dismissed → 204
    // POST /{recommendationType}/actioned  → 204
}
```

### DI Registration
```csharp
// In DIConfig.cs
private static void RegisterRecommendationsServices(IServiceCollection services)
{
    services.AddSingleton<ISellerRecommendationsRepository, SellerRecommendationsRepository>();
    services.AddSingleton<ISellerRecommendationsService, SellerRecommendationsService>();
}
```
Both are Singleton: no per-request state, `IDynamoDbClient` is already Singleton.

---

## README Content (V3Core/Recommendations/README.md)

Covers:
1. How to add a new `RecommendationType` enum value
2. How to create a `XxxMetadata : RecommendationMetadata` class in `Models/Metadata/`
3. How to register the type in `RecommendationMetadataResolver`
4. How the DynamoDB record's `metadata` field is populated (by the data pipeline, not this service)
5. Testing checklist for new recommendation types

---

## Verification

1. Build: `./localdev/remote-build.sh "V3Core/V3Core.csproj"` then `"WebV3Api/WebV3Api.csproj"`
2. Tests: `./localdev/remote-test.sh "SellerRecommendationsServiceTests"`
3. Manual: Start local services (`docker compose up -d`), call `GET /api/seller/recommendations` with an authenticated session — expect `200` with `{ "recommendations": [] }` against dev DynamoDB
4. Verify POST `/api/seller/recommendations/{type}/dismissed` returns `204` and record in DynamoDB has `isDismissed=true`, incremented `dismissedCount`, and `dismissedAt` timestamp
5. Verify POST `/api/seller/recommendations/{type}/actioned` returns `204` with `isActioned=true` and `actionedAt`

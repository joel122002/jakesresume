# MaskingNameClient — Architecture & Design

This document describes how **`Mmv.MaskingNameClient`** works: dependency injection, MySQL loading, RabbitMQ-driven incremental updates, in-memory keying, and how host apps (for example Carwale / BikeWale) consume it. File paths are relative to the `MaskingNameClient` project unless noted.

---

## 1. Purpose

**Masking names** are URL- and SEO-friendly slugs (e.g. make / model / version) stored in MMV alias tables. Resolving a slug to **MakeId, ModelId, version id, MmvStatus, body styles, etc.** on every request would be slow and would hammer the database.

The client:

1. **Bootstraps** a full in-memory map from the **read-only MySQL** MMV database (per `applicationId` / vehicle type).
2. **Subscribes** to a **RabbitMQ fanout** exchange for change events and **patches** the in-memory map using small, targeted SQL reads.
3. Exposes **`IMaskingNames`**: fast, **read-only** lookups with **no per-request database access** on the hot path.

If a lookup returns empty, host applications typically fall back to **gRPC via API Gateway** (e.g. `GetModelByMaskingName`); that path is **not** in this package but is the expected safety net.

---

## 2. Public API: `IMaskingNames`

The application-facing contract lives in `IMaskingNames.cs`:

```csharp
public interface IMaskingNames
{
    public MaskingNameDetails GetMmvDetails(string makeMaskingName, int applicationId);
    public MaskingNameDetails GetMmvDetails(string makeMaskingName, string modelMaskingName, int applicationId);
    public MaskingNameDetails GetRootDetails(string makeMaskingName, string rootMaskingName, int applicationId);
    public MaskingNameDetails GetHyphenatedMakeRootDetails(string maskingName, int applicationId);
    public MaskingNameDetails GetMmvDetails(string makeMaskingName, string modelMaskingName, string versionMaskingName, int applicationId);
    public MaskingNameDetails GetMmvDetails(string makeMaskingName, string modelMaskingName, string trimMaskingName, bool isTrim, int applicationId);
    List<ModelInfo> GetMakeModelData(string makeMaskingName, int applicationId);
}
```

Implementation: `MaskingNames`. It reads from a **static** `MaskingNameSnapshot` owned by `MaskingNameManager` (see §5). Missing keys return a **new empty** `MaskingNameDetails()`.

---

## 3. Dependency injection: `AddMaskignNameService`

Registration is intentionally explicit: options, DB connection factory, an **unbounded `System.Threading.Channels` channel** (single reader / single writer), the manager, the public `IMaskingNames` facade, and **two** hosted services.

> **Note:** The method name `AddMaskignNameService` (typo) is the public API for backward compatibility.

```csharp
public static IServiceCollection AddMaskignNameService(this IServiceCollection services,
                                                       IConfiguration configuration,
                                                       int applicationId)
{
    // ...
    services.AddOptions();
    services.AddLogging();
    services.AddAeplCoreQueue(configuration);
    services.AddDbConnectionString<ConnectionStringOption, IOptions<ConnectionStringOption>>(configuration);
    services.AddSingleton<ConnectionFactory<IOptions<ConnectionStringOption>>>();
    services.AddSingleton<ConnStrRepo<IOptions<ConnectionStringOption>>>();
    services.Configure<MaskingNameOptions>(configuration.GetSection(MaskingNameOptions.MaskingName));

    services.AddSingleton(Channel.CreateUnbounded<MaskingNameChangeEventDataProto>(
        new UnboundedChannelOptions { SingleReader = true, SingleWriter = true }));
    services.AddSingleton(svc => svc.GetRequiredService<Channel<MaskingNameChangeEventDataProto>>().Reader);
    services.AddSingleton(svc => svc.GetRequiredService<Channel<MaskingNameChangeEventDataProto>>().Writer);

    services.AddSingleton<IMaskingNameManager, MaskingNameManager>();
    services.AddSingleton<IMaskingNames, MaskingNames>();

    services.AddHostedService(svc => new MaskingNameManagerInitializer(
        svc.GetRequiredService<IOptions<MaskingNameOptions>>(),
        svc.GetRequiredService<IMaskingNameManager>(), applicationId));

    services.AddHostedService(svc => new MaskingNameUpdatesConsumer(
        svc.GetRequiredService<IOptions<MaskingNameOptions>>(),
        svc.GetRequiredService<IMaskingNameManager>(),
        svc.GetRequiredService<ChannelWriter<MaskingNameChangeEventDataProto>>()));

    return services;
}
```

**Why `applicationId`?** The hosted initializer passes it into `InitializeAsync` so the correct **application → vehicle type** mapping and MMV rows are loaded (e.g. CarWale vs BikeWale).

---

## 4. Configuration: `MaskingNameOptions`

```csharp
public class MaskingNameOptions
{
    public static readonly string MaskingName = "MaskingName";
    public string UpdatesExchangeName { get; set; } = "MMVMASKINGNAMECHANGEEVENTEXCHANGE";
    public string ReadOnlyConnectionString { get; set; } = "ConnectionMySqlReadMMV";
    public string MasterConnectionString { get; set; } = "ConnectionMySqlMasterMMV";
}
```

- **`ReadOnlyConnectionString`**: key into your connection-string configuration (not the raw string). `MaskingNameManager` uses it for all reads in this client.
- **`UpdatesExchangeName`**: RabbitMQ **fanout** exchange for `MaskingNameChangeEventDataProto` messages.
- **`MasterConnectionString`**: reserved for future or other use; bulk and incremental paths in the current code use the read-only key.

Host `appsettings` must define the `MaskingName` section and the connection string entry the key points to.

---

## 5. In-memory model: `MaskingNameSnapshot`

```csharp
internal class MaskingNameSnapshot
{
    public DateTime TimeStamp { get; set; }
    public ConcurrentDictionary<string, MaskingNameDetails> MaskingNames { get; set; } = new();
    public ConcurrentDictionary<string, string> HyphenatedMaskingNameDictionary { get; set; } = new();
    public ConcurrentDictionary<Tuple<string, int>, ConcurrentBag<ModelInfo>> MakeToModelMap { get; set; } = new();
}
```

- **`MaskingNames`**: composite string key → `MaskingNameDetails` (ids, status, body styles, etc.).
- **`HyphenatedMaskingNameDictionary`**: maps hyphenated root-style keys to the canonical underscore key in `MaskingNames` (for URL variants).
- **`MakeToModelMap`**: supports `GetMakeModelData` (models under a make with status).

The snapshot is **`internal static`** on `MaskingNameManager`, so there is **one** logical cache per **process**.

---

## 6. Key formats (how lookups align with URLs)

Built during `GetAllMaskingNameDictionary` and maintained during incremental updates:

| Entity | Key pattern |
|--------|-------------|
| Make | `{makeMaskingName}_{applicationId}` |
| Model | `{makeMaskingName}_{modelMaskingName}_{applicationId}` |
| Version | `{makeMaskingName}_{modelMaskingName}_{versionMaskingName}_{applicationId}` |
| Trim | `{make}_{model}_{trim}_{applicationId}_trim` |
| Root | `Root_{makeMaskingName}_{rootMaskingName}_{applicationId}` |
| Hyphenated root | `Root_{make}-{root}_{applicationId}` → maps to canonical root key string |

`MaskingNames.GetMmvDetails` / `GetRootDetails` / `GetHyphenatedMakeRootDetails` mirror these formats exactly.

---

## 7. Cold start: `MaskingNameManagerInitializer` + `InitializeAsync`

On host startup, `MaskingNameManagerInitializer` validates `applicationId` and awaits **`InitializeAsync`**:

```csharp
public async Task StartAsync(CancellationToken cancellationToken)
{
    if (_applicationId < 1)
    {
        _logger.LogError("applicationId can't be less than 1");
        return;
    }
    _manager = _maskingNameManager.InitializeAsync(_applicationId, cancellationToken);
    await _manager.ConfigureAwait(false);
}
```

`InitializeAsync`:

1. Calls **`GetVehicleTypeByApplicationId`** — loads **`vehicletypesmapping`** (`ApplicationId` → `VehicleType`). On failure, falls back to hardcoded defaults (Car/Bike mapping) and logs.
2. Calls **`GetAllMaskingNameDictionary(applicationId)`**, which:
   - Queries **versions**, **roots**, **models**, **makes**, **trims** (each via Dapper + MySqlConnector).
   - Filters by **`VehicleType`** and **`ApplicationId`** on alias rows.
   - Populates the composite keys above and **`MakeToModelMap`** for models.
3. Merges the result into **`maskingNameSnapshot.MaskingNames`** and sets **`TimeStamp`**.

Bulk query helpers use **`retryCount`** (default 3) with **`Task.Delay(2000)`** between retries on transient MySQL errors.

---

## 8. Incremental updates: RabbitMQ + channel + `ProcessMaskingNameUpdate`

### 8.1 Consumer (`MaskingNameUpdatesConsumer`)

`ExecuteAsync` starts **two** long-running tasks:

```csharp
protected override async Task ExecuteAsync(CancellationToken cancellationToken)
{
    _cancellationToken = cancellationToken;
    Task consumer = StartConsumerWithRetry();
    Task manager = _maskingNameManager.StartProcessingUpdates(cancellationToken);
    await Task.WhenAll(consumer, manager).ConfigureAwait(false);
}
```

1. **`StartConsumerWithRetry`**: connects via **`ChannelManager.Instance`** (AEPLCore.Queue), declares the **fanout** exchange, creates an **exclusive auto-delete** queue, binds, consumes. Each message is deserialized to **`MaskingNameChangeEventDataProto`** and **`TryWrite`** to the channel, then **ACK**. On consumer cancellation, it restarts the subscription loop.

2. **`StartProcessingUpdates`**: loops `WaitToReadAsync` / `TryRead` on the **`ChannelReader`** and calls **`ProcessMaskingNameUpdate`** per item.

### 8.2 Patch logic

`ProcessMaskingNameUpdate` switches on **`MmvEntity`** (Version, Root, Model, Make, Trim):

- For each case, it loads **current** rows from MySQL using helpers such as `GetMakeDetailsByMaskingname`, `GetModelDetailsByMaskingname`, `GetVersionDetailsByMaskingname`, etc. Old vs new masking name depends on **`Operation`** (Create / Update / Delete / Discontinue).
- It builds **`oldMaskingNameKeyDictionary`** (keys to remove) and **`newMaskingNameKeyDictionary`** (keys to add). Make renames **cascade**: models, roots, versions, and trims under that make get key updates.
- It removes all old keys from **`maskingNameSnapshot.MaskingNames`** (and hyphenated entries). If operation is not **Delete**, it inserts new keys.

**Model** events also update **`MakeToModelMap`** via `UpdateMakeToModelMap`.

---

## 9. Read path: no database

`MaskingNames.GetMmvDetails(...)` only checks `maskingNameSnapshot`. There is **no** SQL here; throughput depends on in-memory dictionary performance.

---

## 10. Integration in host applications (Carwale / BikeWale)

In ASP.NET Core startup, register the client **with the correct `applicationId`**:

```csharp
services.AddMaskignNameService(Configuration, (int)Application.CarWale);
// or BikeWale
```

Static helpers (for example in **`Carwale.Utility.MaskingNameUtils`**) resolve `IMaskingNames` from the service provider and delegate:

```csharp
IMaskingNames maskingNames = ServiceRegistryUtils.ServiceProviderForStaticAccess.GetRequiredService<IMaskingNames>();
return maskingNames.GetMmvDetails(makeMaskingName, modelMaskingName, applicationId);
```

**Typical page flow:**

1. Call `GetMmvDetails` with URL segments.
2. If `MakeId` / `ModelId` (or equivalent) are missing → treat as **miss** → call MMV over **API Gateway** (`ModelByMaskingNameAdapter`, etc.) and map the gRPC response into the same shape as `MaskingNameDetails`.

---

## 11. Operational concerns

| Topic | Behavior |
|-------|----------|
| **Ordering** | Single reader/writer channel serializes update processing per process. |
| **Consistency** | Eventually consistent with MMV DB + message lag; full consistency requires successful processing of all events or restart (full bulk reload). |
| **Missed messages** | No built-in periodic full refresh; long RabbitMQ outages can leave memory stale until redeploy or manual reload if you add one. |
| **Multi-instance** | Each instance keeps its own snapshot; all must see the same DB and queue messages. |
| **Static snapshot** | Tests or multiple logical apps in one process should not assume isolation unless they use separate processes or refactor static state for tests. |

---

## 12. Source file map

| File | Role |
|------|------|
| `ServiceCollectionExtensions.cs` | `AddMaskignNameService` DI registration |
| `Options.cs` | `MaskingNameOptions` |
| `IMaskingNames.cs` / `MaskingNames.cs` | Public lookup API |
| `IMaskingNameManager.cs` / `MaskingNameManager.cs` | DB load, snapshot, channel processing, SQL |
| `MaskingNameManagerInitializer.cs` | Hosted startup bulk load |
| `MaskingNameUpdatesConsumer.cs` | RabbitMQ → channel |
| `Entities.cs` | `MaskingNameSnapshot`, DTOs, enums |

---

## 13. Package / proto dependency

The client depends on **`MaskingNameChangeEventDataProto`** from **`MMV.Service.ProtoClass`** (shared protobuf definitions). Queue payloads wrap this proto inside the AEPL **`QueueMessage`** envelope.

---

*Generated as internal documentation for maintainers. Keep in sync when changing registration, key formats, or transport.*

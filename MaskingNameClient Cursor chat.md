# MaskingNameClient usage and flow
_Exported on 5/2/2026 at 20:54:58 GMT+5:30 from Cursor (2.5.22)_

---

**User**

Can you tell me how is masking MaskingNameClient being used and the entire flow basically how this thing works refer to the two folders in the workspace to know how this thing works and give me the entire flow

---

**Cursor**

Tracing `MaskingNameClient` usage and the masking flow across the codebase.



---

**User**

Can you explain how @MMV/MaskingNameClient/ServiceCollectionExtensions.cs also plays a role?

---

**Cursor**

Here is how `ServiceCollectionExtensions.cs` fits in and what it does.

## What this file is

`AddMaskignNameService` is the **extension method** that CarWale/BikeWale call from `Startup.cs` (`services.AddMaskignNameService(Configuration, (int)Application.CarWale)` / `BikeWale`). It wires the **local masking-name resolution path** from the `Mmv.MaskingNameClient` package: config, DB access, a queue, in-memory state, and background services that **load and keep that state up to date**.

## What it registers (in order of role)

1. **Infrastructure the client needs**  
   - `AddOptions`, `AddLogging`  
   - `AddAeplCoreQueue` – message queue used for **change events**  
   - `AddDbConnectionString` + `ConnectionFactory` + `ConnStrRepo` – so the client can read masking data from a **database** (per `MaskingNameOptions` section)

2. **Configuration**  
   - `Configure<MaskingNameOptions>(configuration.GetSection(MaskingNameOptions.MaskingName))` – binds the `MaskingName` config block (connection string, module names, etc., as defined in that options class).

3. **In-process event pipeline**  
   - A **single-reader, single-writer** `Channel<MaskingNameChangeEventDataProto>` and registered `ChannelReader` / `ChannelWriter` – used to pass **proto-shaped “masking name changed” events** from a consumer into the manager.

4. **The two main services your app uses**  
   - `IMaskingNameManager` → `MaskingNameManager` – owns the **in-memory model** and **update** logic (load, apply changes).  
   - `IMaskingNames` → `MaskingNames` – the **query API** that `MaskingNameUtils` resolves via `GetRequiredService<IMaskingNames>()` and calls `GetMmvDetails(...)`, `GetMakeModelData(...)`, etc.

5. **Background workers (why resolution is “local” and fresh)**  
   - **`MaskingNameManagerInitializer`** (hosted) – runs at startup: uses `IMaskingNameManager` + `applicationId` (the int you pass from Startup) to **initialize** the local masking data for that app (CarWale vs BikeWale, etc.).  
   - **`MaskingNameUpdatesConsumer`** (hosted) – subscribes to the **queue** and, when the MMV service publishes **masking-name change events**, feeds them through the channel so `MaskingNameManager` can **refresh** without needing a full redeploy for every catalog change.

## How this connects to the flow we discussed earlier

- **Happy path:** Request comes in with URL segments like make/model **masking names** → `MaskingNameUtils.GetMmvDetailsByMaskingName(...)` → **`IMaskingNames`** (registered here) answers from **in-memory data** populated via DB init + queue updates. Fast, no MMV gRPC for that step.

- **Fallback path:** If that returns empty / invalid IDs, Carwale code calls **`ModelByMaskingNameAdapter`** (API Gateway → MMV **GetModelByMaskingName**) and builds a `MaskingNameDetails` from the response.

So **`ServiceCollectionExtensions.cs` does not call the MMV gRPC adapter itself**; it **bootstraps and maintains** the client library’s **`IMaskingNames`** implementation so that most slug→ID resolution stays **local and eventually consistent** with the MMV source of truth, while `AddMaskignNameService`’s **typo** is preserved in the public API for backward compatibility.

---

**User**

Yea can you tell me in detail how it fetches from the db, how the background worker work and stuff, Like DIfferent scenarios, how the intial stuff is populated etc

---

**Cursor**

Tracing the MaskingNameClient implementation: managers, initializers, DB access, and background consumers.



Here is a concrete walkthrough based on the MMV `MaskingNameClient` sources you have locally (`MaskingNameManager`, `MaskingNameManagerInitializer`, `MaskingNameUpdatesConsumer`, `MaskingNames`, `Entities`).

## 1. Where the data lives in memory

Everything resolves through a **single static snapshot**:

```23:23:/home/joelvarghese/Code/MMV/MaskingNameClient/MaskingNameManager.cs
        internal static MaskingNameSnapshot maskingNameSnapshot = new MaskingNameSnapshot();
```

`MaskingNameSnapshot` holds:

- **`MaskingNames`**: `ConcurrentDictionary<string, MaskingNameDetails>` — canonical slug keys → IDs/status/body styles.
- **`HyphenatedMaskingNameDictionary`**: maps hyphenated root keys to the underscore key used in `MaskingNames` (for URLs like `make-root` vs `make_root`).
- **`MakeToModelMap`**: `(makeMaskingName, applicationId)` → list of `ModelInfo` for `GetMakeModelData`.

`IMaskingNames` / `MaskingNames` only **looks up** these structures; it does **not** hit the database on the hot path.

---

## 2. Startup: how the first load is populated

**Hosted service:** `MaskingNameManagerInitializer` runs at app startup and calls `InitializeAsync` for the **`applicationId`** passed from `AddMaskignNameService` (CarWale vs BikeWale maps to different rows in `vehicletypesmapping`).

```29:37:/home/joelvarghese/Code/MMV/MaskingNameClient/MaskingNameManagerInitializer.cs
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

**`InitializeAsync`** does two things:

1. **`GetVehicleTypeByApplicationId`** — ensures `VehicleTypeByApplicationId` is filled from **`vehicletypesmapping`** (`ApplicationId` → `VehicleType`, e.g. Car vs Bike). If that query fails or returns nothing, it falls back to hardcoded defaults (1→Car, 2→Bike, 3→Car) and logs an error.

2. **`GetAllMaskingNameDictionary(applicationId)`** — this is the **full bulk load**. It runs **five** MySQL read queries (via Dapper), all scoped by `@ApplicationId` and `@VehicleType`:

   - All **versions** (joins `versions`, `versionsalias`, `models`/`modelsalias`, `makes`/`makesalias`, body styles; excludes inactive versions).
   - All **roots** (`modelroots` + aliases + makes).
   - All **models** (with `GROUP_CONCAT` body styles).
   - All **makes**.
   - All **trims** (`modeltrims`).

Those lists are folded into **string keys** and `MaskingNameDetails` DTOs, for example:

- Make: `{makeMaskingName}_{applicationId}`
- Model: `{make}_{model}_{applicationId}`
- Version: `{make}_{model}_{version}_{applicationId}`
- Trim: `{make}_{model}_{trim}_{applicationId}_trim`
- Root: `Root_{make}_{root}_{applicationId}` plus hyphenated alias entries.

Then `InitializeAsync` **copies** every entry into `maskingNameSnapshot.MaskingNames` and sets `TimeStamp`. If the dictionary is still empty, it logs that initialization failed.

**DB connection:** all reads use `_dbConnectionFactory.GetConnection(_maskingNameOptions.ReadOnlyConnectionString)`. The option value is the **config key name** (default `ConnectionMySqlReadMMV`), not the raw connection string — resolved through your existing AEPLCore connection-string plumbing.

**Resilience:** each bulk `GetAll*ListAsync` retries up to **3** times with **2s** delay on MySQL / protocol / NRE-style failures.

---

## 3. Background workers: two cooperating loops

`MaskingNameUpdatesConsumer` is a `BackgroundService`. Its `ExecuteAsync` runs **two tasks in parallel**:

```117:123:/home/joelvarghese/Code/MMV/MaskingNameClient/MaskingNameUpdatesConsumer.cs
        protected override async Task ExecuteAsync(CancellationToken cancellationToken)
        {
            _cancellationToken = cancellationToken;
            Task consumer = StartConsumerWithRetry();
            Task manager = _maskingNameManager.StartProcessingUpdates(cancellationToken);
            await Task.WhenAll(consumer, manager).ConfigureAwait(false);
        }
```

### A) RabbitMQ → in-memory channel (producer side)

- Uses **`ChannelManager`** (from `AEPLCore.Queue`) to get a RabbitMQ channel.
- Declares a **fanout** exchange named by config: default **`MMVMASKINGNAMECHANGEEVENTEXCHANGE`** (`MaskingNameOptions.UpdatesExchangeName`).
- Creates an **exclusive, auto-delete, server-named queue**, binds it to that exchange, and starts consuming.
- On each message: deserialize `QueueMessage` → payload as **`MaskingNameChangeEventDataProto`**, then **`TryWrite`** to the unbounded `Channel<MaskingNameChangeEventDataProto>`, then **ACK**.
- If the consumer is cancelled, it **restarts** the consumer in a loop (`StartConsumerWithRetry` with 5s backoff on failure).

So: **MMV (or another publisher)** broadcasts “masking name changed” events; this worker **only** forwards them into the in-process channel.

### B) Channel reader → incremental DB-backed patch (consumer side)

`MaskingNameManager.StartProcessingUpdates` loops:

```136:152:/home/joelvarghese/Code/MMV/MaskingNameClient/MaskingNameManager.cs
        private async Task StartProcessingUpdates(CancellationToken cancellationToken)
        {
            try
            {
                while (await _maskingNameUpdatesReader.WaitToReadAsync() && !cancellationToken.IsCancellationRequested)
                {
                    while (_maskingNameUpdatesReader.TryRead(out MaskingNameChangeEventDataProto item))
                    {
                        await ProcessMaskingNameUpdate(item);
                    }
                }
            }
```

For **each** event, `ProcessMaskingNameUpdate`:

1. Switches on **`MmvEntity`** (Version, Root, Model, Make, Trim).
2. For that entity, runs **targeted** SQL (e.g. `GetMakeDetailsByMaskingname`, `GetModelDetailsByMaskingname`, …) using **old vs new** masking name depending on **`Operation`** (Create/Update/Delete/Discontinue).
3. Builds **old** keys (to remove) and **new

**Remove vs add:**  
It fills temporary concurrent dictionaries for keys to remove and keys to add. Then it **removes** all `old*` entries from `maskingNameSnapshot.MaskingNames` (and hyphenated map). For non-**Delete** operations, it **adds** the `new*` entries.

For **Make** renames, the logic is heavier: after reloading the make, it reloads **all models under that make**, **all roots**, **all versions and trims** for affected model IDs so dependent keys stay aligned.

For **Model** events, it also calls **`UpdateMakeToModelMap`**, which may add/update/remove a `ModelInfo` in `MakeToModelMap` (or remove the make’s bag on delete).

So incremental updates are **“event + read current row(s) from MySQL + surgically replace keys in the snapshot”**, not a second full `GetAllMaskingNameDictionary` unless you treat startup as the only full rebuild.

---

## 4. Read path (after data is in memory)

`MaskingNames` only does **dictionary lookup** by the same key patterns used at build time. If the key is missing, it returns an **empty** `MaskingNameDetails` (default instance), which is what drives the Carwale “miss → call `ModelByMaskingNameAdapter`” behavior.

There is **no** per-request DB call in `GetMmvDetails`.

---

## 5. Scenarios (mental model)

| Scenario | What happens |
|----------|----------------|
| **Cold start** | `MaskingNameManagerInitializer` runs → bulk MySQL reads → fills `maskingNameSnapshot`. Until this completes, lookups can be empty (race with first requests depends on host startup ordering). |
| **Steady state / cache hit** | URL slugs hit `IMaskingNames` → O(1) in-memory lookup → no DB, no queue. |
| **CMS changes a make/model/version trim/root** | MMV publishes to RabbitMQ fanout → `MaskingNameUpdatesConsumer` writes proto to channel → `ProcessMaskingNameUpdate` runs focused queries and patches dictionaries. |
| **Rename make** | Event loads make + cascades to models, roots, versions, trims under that make; old keys removed, new keys inserted. |
| **Delete / discontinue** | Uses **old** masking name in queries (`Operation.Delete` relaxes some `Status <> 'inactive'` filters so rows can still be found). Keys removed; for deletes, new keys are not added. |
| **DB unavailable at startup** | Retries on bulk queries; if still failing, lists may be null/empty → weak snapshot and error logs. |
| **Queue down / consumer broken** | In-memory data stays at **last successful** bulk load + any updates received before failure; **no** automatic full refresh from queue alone. |
| **Wrong `applicationId` in `AddMaskignNameService`** | Wrong `VehicleType` → wrong filtered rows → wrong or missing keys for that app. |

---

## 6. How this ties together

- **DB** is the **source of truth** for both the initial bulk snapshot and each incremental patch after a queue event.
- **RabbitMQ** is the **signal** that something changed so the process does not need periodic full-table polling.
- **`IMaskingNames`** is a **read-through cache** implemented as in-memory concurrent dictionaries, shared process-wide via the static `maskingNameSnapshot`.

That is the full picture of how fetch, workers, and scenarios fit together in this client.

---

**User**

Can you make a robust document on how this was created with code snippets?

---

**Cursor**

Creating architecture documentation in the MMV `MaskingNameClient` project, pulling accurate snippets from the source files.



Here is what was added.

## Document location

**`/home/joelvarghese/Code/MMV/MaskingNameClient/ARCHITECTURE.md`**

It is a single reference doc that covers:

1. **Purpose** — in-memory slug resolution vs DB/gRPC fallback  
2. **`IMaskingNames`** — interface summary  
3. **`AddMaskignNameService`** — full registration snippet (options, queue, channel, singletons, two hosted services)  
4. **`MaskingNameOptions`** — config keys and RabbitMQ exchange default  
5. **`MaskingNameSnapshot`** — the three dictionaries and what they store  
6. **Key formats** — table of string key patterns (make/model/version/trim/root/hyphenated)  
7. **Cold start** — initializer + `InitializeAsync` + bulk MySQL reads + retries  
8. **Incremental updates** — fanout consumer, channel, `ProcessMaskingNameUpdate` / entity types / cascade on make rename  
9. **Read path** — no DB on lookup  
10. **Host integration** — Carwale/BikeWale-style `Startup` + static `MaskingNameUtils` pattern  
11. **Operations** — consistency, multi-instance, static snapshot caveats  
12. **Source file map** — which file owns what  
13. **Proto / queue dependency** — `MaskingNameChangeEventDataProto`, `QueueMessage`

Snippets are taken from your project (registration block, initializer, consumer `ExecuteAsync`, options, snapshot class, key table). Long SQL stays in code; the doc points to `MaskingNameManager` for details.

If you want this duplicated under `carwaleweb` (e.g. `docs/masking-name-client.md`), say where you prefer it and we can add a short pointer file or copy.

---

**User**

AddMaskignNameService so this would remain in the client right? How would the client have access to queue events?

---

**Cursor**

Yes. **`AddMaskignNameService` lives in the client** (`ServiceCollectionExtensions.cs` in the `Mmv.MaskingNameClient` package). Host apps only call it from `Startup` and pass `IConfiguration` + `applicationId`.

## How the client “gets” queue events

The client does **not** embed a broker. It is a **RabbitMQ consumer** in the same process as the web app. Access works like this:

1. **Registration**  
   Inside `AddMaskignNameService`, the first step is `services.AddAeplCoreQueue(configuration)`. That (from your **AEPLCore.Queue** stack) registers whatever is needed so **`ChannelManager.Instance.GetChannel()`** can open a real connection/session to **RabbitMQ** using values from **configuration** (host, vhost, credentials, etc.—usually the same style of config other services use).

2. **At runtime**  
   `MaskingNameUpdatesConsumer` calls `ChannelManager.Instance.GetChannel()`, then:
   - declares the **fanout** exchange named in `MaskingNameOptions.UpdatesExchangeName` (default `MMVMASKINGNAMECHANGEEVENTEXCHANGE`),
   - creates an **anonymous, exclusive, auto-delete** queue,
   - **binds** that queue to the exchange,
   - **consumes** and deserializes each message to `MaskingNameChangeEventDataProto`, then `TryWrite` to the in-process `Channel`.

3. **Who produces events?**  
   The **MMV (or another) service** is expected to **publish** to that same exchange when masking names or related entities change. Fanout = every bound queue (including each instance of your app) gets a copy. The client only **subscribes**; it does not own the exchange definition on the server beyond `ExchangeDeclare` (idempotent in RabbitMQ).

So: **the client has “access” to queue events because the deployed app is configured with valid RabbitMQ settings and is allowed to connect and consume**; the **masking change events** are **published elsewhere** and delivered by the broker to this consumer.

If RabbitMQ is unreachable, `StartConsumerWithRetry` keeps retrying; in-memory data stays at the last good bulk load + any messages already processed.

---

**User**

So the client also makes call to the DB correct?

---

**Cursor**

Yes. The **client process** talks to **MySQL** directly (read-only connection by key, default `ConnectionMySqlReadMMV`).

- **Startup:** `MaskingNameManager.InitializeAsync` → `GetAllMaskingNameDictionary` runs the big **bulk** queries (makes, models, versions, roots, trims, `vehicletypesmapping`, etc.) via Dapper.
- **After a RabbitMQ event:** `ProcessMaskingNameUpdate` runs **smaller, targeted** queries (e.g. by make/model/version masking name) to build the new key set and patch the in-memory snapshot.

`IMaskingNames` lookups are **in-memory only**; the **DB** is used to **fill and update** that snapshot, not for each `GetMmvDetails` call.

---

**User**

Can you recurs through the object and output the entire masking name dictionary object which is used as a json with data types?

---

**Cursor**

Gathering all types that compose the masking snapshot and serialized JSON shape.



Here is a recursive breakdown of the **in-memory snapshot** type (`MaskingNameSnapshot` in `Entities.cs`), including **`[JsonProperty]` names** where defined, **C# types**, and how they map to **JSON value types**.

---

### Root: `MaskingNameSnapshot` (internal)

| JSON property (`JsonProperty`) | C# property | C# type | JSON shape |
|-------------------------------|---------------|---------|------------|
| `"timestamp"` | `TimeStamp` | `DateTime` | string (ISO 8601) or date object per serializer settings |
| `"maskingNamesDictionary"` | `MaskingNames` | `ConcurrentDictionary<string, MaskingNameDetails>` | **object**: keys = string, values = object (see below) |
| `"hyphenatedMaskingNameDictionary"` | `HyphenatedMaskingNameDictionary` | `ConcurrentDictionary<string, string>` | **object**: keys = string, values = string |
| `"makeModelDictionary"` | `MakeToModelMap` | `ConcurrentDictionary<Tuple<string, int>, ConcurrentBag<ModelInfo>>` | **object**, but keys are **not** plain strings — see note below |

---

### Nested: `MaskingNameDetails` (values in `maskingNamesDictionary`)

| Field | C# type | JSON type | Notes |
|-------|---------|-----------|--------|
| `Id` | `int` | number | Often version id; used where entity has one |
| `MakeId` | `int` | number | |
| `ModelId` | `int` | number | |
| `TrimId` | `int` | number | |
| `MmvStatus` | `MmvStatus` (enum) | number | Serialized as integer unless you use string enum converter |
| `RootId` | `int` | number | |
| `BodyStylesIds` | `List<int>` | array of numbers | May be `null` if not set |

**`MmvStatus` enum** (`int` in JSON unless customized):

| Name | Value |
|------|-------|
| `None` | 0 |
| `Futuristic` | 1 |
| `New` | 2 |
| `Discontinued` | 3 |
| `InActive` | 4 |

---

### Nested: `ModelInfo` (items inside each bag in `makeModelDictionary`)

| Field | C# type | JSON type |
|-------|---------|-----------|
| `ModelId` | `int` | number |
| `Status` | `MmvStatus` | number |

---

### Full tree (conceptual)

```text
MaskingNameSnapshot
├── timestamp                 : DateTime
├── maskingNamesDictionary    : { [string]: MaskingNameDetails }
│   └── MaskingNameDetails
│       ├── Id               : int
│       ├── MakeId           : int
│       ├── ModelId          : int
│       ├── TrimId           : int
│       ├── MmvStatus        : MmvStatus (enum → number)
│       ├── RootId           : int
│       └── BodyStylesIds    : List<int> | null
├── hyphenatedMaskingNameDictionary : { [string]: string }
└── makeModelDictionary       : { key → ConcurrentBag<ModelInfo> }
    └── ModelInfo[]
        ├── ModelId          : int
        └── Status           : MmvStatus (enum → number)
```

---

### Example JSON **shape** (illustrative — keys shortened)

```json
{
  "timestamp": "2026-05-02T12:00:00",
  "maskingNamesDictionary": {
    "maruti-dzire_1": {
      "Id": 0,
      "MakeId": 10,
      "ModelId": 200,
      "TrimId": 0,
      "MmvStatus": 2,
      "RootId": 5,
      "BodyStylesIds": [1, 3]
    }
  },
  "hyphenatedMaskingNameDictionary": {
    "maruti-swift-2024_1": "Root_maruti-swift_2024_1"
  },
  "makeModelDictionary": { }
}
```

---

### Important caveat: `makeModelDictionary`

Keys are **`Tuple<string, int>`** (make masking name + `applicationId`). **Newtonsoft.Json** does not serialize `ConcurrentDictionary<Tuple<…>, …>` as a clean `{ "make_appId": … }` map unless you use a custom converter. Often you get key strings like `"(make, 1)"` or other `ToString()`-like forms, or the type may be **skipped / not used** in whatever code path actually serializes the snapshot. For a **portable** JSON contract, you would normally use **`string` keys** (e.g. `"make_1"`) or a **list of objects** instead of `Tuple` keys.

---

### Types **not** part of the snapshot JSON but related in `Entities.cs`

- **`VersionDetails` / `RootDetails` / `VersionDetailsResponse` / `RootDetailsResponse` / `VehicleTypeData`**: used for **Dapper** / internal processing, not the `JsonProperty` surface of `MaskingNameSnapshot`.
- **`Operation` enum**: used in **update** processing, not in the snapshot DTO.

If you need this as a **formal JSON Schema** or a **C# DTO** that round-trips `makeModelDictionary` cleanly, say which serializer settings you use (Newtonsoft default vs `StringEnumConverter` vs custom) and we can align the schema.

---

**User**

How did it differentiate between make model and version?

---

**Cursor**

It does **not** use a separate “entity type” field on the value. It differentiates by **how the lookup key is built** and by **which API you call**.

## 1. Different **key string** patterns (namespace in one flat dictionary)

All of these live in the same `maskingNameSnapshot.MaskingNames` map, but the **key** encodes the level:

| Level   | Key pattern | Example (conceptual) |
|--------|-------------|----------------------|
| **Make**   | `{make}_{appId}` | `maruti_1` |
| **Model**  | `{make}_{model}_{appId}` | `maruti_dzire_1` |
| **Version** | `{make}_{model}_{version}_{appId}` | `maruti_dzire_vxi_1` |
| **Trim** | `{make}_{model}_{trim}_{appId}_trim` | `maruti_dzire-zxi_1_trim` |
| **Root** | `Root_{make}_{root}_{appId}` | `Root_maruti_swift_1` |

So **make vs model vs version** is distinguished by **how many slug parts** are in the key (and by special prefixes/suffixes like `Root_` and `_trim`).

## 2. Different **entry points** on `IMaskingNames`

Callers pick the overload that matches the URL / intent; each overload formats **one** of the patterns above:

```10:30:/home/joelvarghese/Code/MMV/MaskingNameClient/MaskingNames.cs
        public MaskingNameDetails GetMmvDetails(string makeMaskingName, int applicationId)
        {
            string maskingNameKey = $"{makeMaskingName}_{applicationId}";
            // ...
        }

        public MaskingNameDetails GetMmvDetails(string makeMaskingName, string modelMaskingName, int applicationId)
        {
            string maskingNameKey = $"{makeMaskingName}_{modelMaskingName}_{applicationId}";
            // ...
        }
```

```60:68:/home/joelvarghese/Code/MMV/MaskingNameClient/MaskingNames.cs
        public MaskingNameDetails GetMmvDetails(string makeMaskingName, string modelMaskingName, string versionMaskingName, int applicationId)
        {
            string maskingNameKey = $"{makeMaskingName}_{modelMaskingName}_{versionMaskingName}_{applicationId}";
            // ...
        }
```

So: **make-only** → one string + app id; **model** → two slugs + app id; **version** → three slugs + app id. No overlap as long as masking names don’t collide across levels in a way that produces the same composite string (that’s a data/design constraint).

## 3. The **value** (`MaskingNameDetails`) is shared shape

`MaskingNameDetails` is the same type for every key; **which properties matter** depends on what was stored when that key was built (e.g. version rows populate `Id`/variant id; make-level rows may emphasize `MakeId` and body styles). The client doesn’t store “this row is a version” separately—it’s implied by **which key** you used to retrieve it.

**Summary:** Differentiation is **key composition + overload**, not a discriminator enum on the dictionary value.


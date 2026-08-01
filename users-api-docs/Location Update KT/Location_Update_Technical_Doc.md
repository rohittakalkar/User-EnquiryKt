# Location Update — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`Location_Update_Business_Doc.md`](./Location_Update_Business_Doc.md)
dekho.

**Repos**: `service-api-go-production` (write only). **Koi dedicated read-controller ya
consumer nahi mila** — write-only, insert-only endpoint. Yeh is KT-series ka **most
heavily refactored/modern-style** controller hai — goroutine-based concurrent batch-
processing, structured stats-tracking, decomposed helper-functions (atypical of the
rest of this codebase's largely-monolithic-controller style).

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write | write | [`UserLocationsUpdateController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserLocationsUpdateController.go), [`UserLocationsUpdateModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserLocationsUpdateModel.go) (`UserLocationsDataInsert`, `UserLocationsInputValidation`) |
| Validation | write | `MandatoryParamsCheckUserLocationsUpdate` — [`UserUtilsMandatory.go:993`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go), `ValidateUserLocationsUpdate` — [`UsersValidationMaps.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |

---

## 2. Routes

| Method | serviceName | Repo | Controller |
|---|---|---|---|
| POST | `USER_MULTIPLE_LOCATIONS_UPSERT` | write | `UserLocationsUpdateController` |

---

## 3. Data Model — Table

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_GEO_ADDT_CONTACT` | cslpg | Supplier geo-location + detailed address record, insert-only, multi-row-per-user | `FK_GLUSR_USR_ID`, `GLUSR_GEO_ADDT_LATITUDE`/`_LONGITUDE` (`NUMERIC(15,8)`), `GLUSR_GEO_ADDT_SOURCE` (modid), `GLUSR_GEO_ADDT_LATLONG_ADD_TYP` (loc_type), `GLUSR_GEO_ADDT_VERIFIED_STATUS` (status), `GLUSR_GEO_ADDT_ORIG_ADD`/`_UPDATED_ADD`, `GLUSR_GEO_ADDT_ROUTE`, `_STRT_ADDRESS`, `_PREMISE`, `_INTERSECTION`, `_SUBLOCALITY1..5`, `_AREA_LEVEL1..5`, `_ZIP`, `_LOCALITY`, `GLUSR_GEO_ADD_CITY`, `GLUSR_GEO_ADDT_STATE`, `_COUNTRY`, `_DIVISION`, `_MAP_URL`, `_DATE` (`CURRENT_TIMESTAMP`) — [`UserLocationsUpdateModel.go:124`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserLocationsUpdateModel.go) |

**Note**: `GLUSR_GEO_ADD_CITY` (no `T` before `_CITY`, unlike every other `GLUSR_GEO_ADDT_*`
column) is a naming inconsistency in the schema itself, not a code bug.

---

## 4. Business Rules & Validation (code se)

1. **Gateway allowlist**: `ANDROID`, `IOS` only.
   [`UserUtilsMandatory.go:999`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
2. **On successful gateway-validation, `token`/`modid` are silently overwritten** in
   `inputParams` (`token = "imobile@15061981"` hardcoded, `modid = gate`) — a hardcoded
   token-constant embedded directly in validation logic.
   [`UserUtilsMandatory.go:1001-1004`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
3. **Mandatory fields**: `latitude`, `longitude`, `usr_id`, `modid`, `token`,
   `orig_add`, `flag`, `loc_type`, `status` — all numeric-checked where applicable
   (`usr_id`, `latitude`, `longitude`).
4. **`flag` must be `I` or `U`**, but **only `I` (Insert) is actually implemented** —
   any other value (including the technically-valid `U`) returns `"Only Insertion is
   allowed"` without touching the DB. `loc_type` must be 1-50, `status` must be 0-2.
5. **DB-level duplicate-protection via `ON CONFLICT (FK_GLUSR_USR_ID,
   GLUSR_GEO_ADDT_LATITUDE, GLUSR_GEO_ADDT_LONGITUDE) DO NOTHING`** — a genuine unique
   constraint backs this, so `rowsAffected==0` reliably indicates a true duplicate, not a
   race-condition-guess.
   [`UserLocationsUpdateModel.go:124,136-141`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserLocationsUpdateModel.go)
6. **Batch-mode (`is_multiple=1` + `addr_arr`)**:
   - Each address in the array is validated sequentially first (preserving order for
     stats/logging).
   - **In-request duplicate-detection** happens via a `seenRequestCoords` map keyed on a
     formatted `"lat|long"` string (8-decimal precision via `NumberFormat`) — catches
     duplicates *within the same batch* before they ever reach the DB.
   - **Only the non-duplicate, valid addresses are dispatched as concurrent goroutine
     tasks** (`runUserLocationBatchInsertTasks`), worker-pool capped at
     `min(taskCount, 4, dbConn.MaxOpenConnections)`.
   - Each address inherits `usr_id`/`token`/`modid`/`VALIDATION_KEY` from the parent
     request and always gets `flag="I"` forced (`setBatchAddrDefaults`) — sub-items
     can't independently request update, only insert.
   [`UserLocationsUpdateController.go:232-313`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserLocationsUpdateController.go)
7. **Single-mode (no `is_multiple`)**: same validation + insert path, just one address,
   no concurrency needed.

---

## 5. RabbitMQ / Kafka / Redis

**Koi RabbitMQ, Kafka, ya Redis usage nahi mila.**

---

## 6. End-to-End Technical Flow

### Single location
```
Supplier (Android/iOS)
    │
    ▼
[API — write]  POST serviceName=USER_MULTIPLE_LOCATIONS_UPSERT
                {usr_id, latitude, longitude, flag="I", loc_type, status, orig_add, ...}
    │  UserLocationsUpdateController.go — Gateway(ANDROID/IOS), token/modid overwrite
    ▼
UserLocationsInputValidation()  — mandatory + type/range checks
    ▼
UserLocationsDataInsert()
    │  INSERT INTO GLUSR_GEO_ADDT_CONTACT (...) ON CONFLICT (user, lat, long) DO NOTHING
    ▼
[DB — cslpg]
    ▼
Response {"Record Successfully Inserted" / " Duplicate data"}
```

### Multiple locations (batch)
```
Supplier (Android/iOS)
    │
    ▼
[API — write]  POST {is_multiple=1, addr_arr: [{lat,long,...}, {lat,long,...}, ...]}
    │  buildUserLocationRequestContext() — parse addr_arr
    ▼
handleMultipleLocations()
    │  for each address (sequential):
    │      setBatchAddrDefaults() — inherit usr_id/token/modid, force flag="I"
    │      UserLocationsInputValidation()
    │      ├─ invalid → record failure, skip
    │      └─ valid →
    │            buildCoordKey() → check seenRequestCoords (in-batch dedup)
    │            ├─ duplicate-in-request → mark "Duplicate data", skip DB
    │            └─ unique → queue as goroutine task
    ▼
runUserLocationBatchInsertTasks()  — worker-pool (≤4, ≤MaxOpenConnections)
    │  concurrent UserLocationsDataInsert() calls, each own ON CONFLICT DO NOTHING
    ▼
[DB — cslpg]  GLUSR_GEO_ADDT_CONTACT  (parallel inserts)
    ▼
Response {" Postgres result: <outcome per address, pipe-separated>"}
```

---

## 7. Optimization Scope — DB Response-Time Contribution

### Already well-optimized (positive findings)

1. **Concurrent worker-pool for batch-inserts** — this is the most explicitly
   performance-engineered write-path in this entire KT series: bounded parallelism (via
   `dbConn.MaxOpenConnections`), avoiding both serial-slowness and connection-pool
   exhaustion.
2. **Two-layer duplicate-detection (in-request map + DB `ON CONFLICT`)** — avoids
   wasting DB round-trips on batch-internal duplicates while still relying on the DB as
   the source of truth for cross-request duplicates.
3. **Detailed micro-benchmarking already built into the response/logs** (validation-time,
   per-item-processing-time, unaccounted-time-estimation) — suggests this endpoint has
   already been through a performance-tuning pass; any future optimization should build
   on this existing instrumentation rather than re-deriving it.

### Low impact

4. **Worker-pool cap of 4 is a hardcoded constant** — if this endpoint regularly receives
   large batches and DB capacity allows more concurrency, this cap could be revisited;
   equally, if `dbConn.MaxOpenConnections` isn't configured, `getUserLocationWorkerCount`
   would use `taskCount` capped only at 4, so a burst of very small batches wouldn't
   under-utilize.

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora Location Update flowchart yahan dekho](https://lucid.app/lucidchart/65aef49a-62b1-4500-9d62-ebab17a54aa8/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **`flag="U"` is validated as "allowed" by `MandatoryParamsCheckUserLocationsUpdate`
   but functionally rejected later** with "Only Insertion is allowed" — validation and
   actual-capability are inconsistent; a caller reading only the mandatory-check logic
   might assume update is supported.
2. **Hardcoded token-constant `"imobile@15061981"`** embedded in validation logic — a
   credential-like string in code, worth confirming this isn't sensitive/rotatable
   secret material that should be externalized.
3. **`GLUSR_GEO_ADD_CITY` naming inconsistency** in the schema (missing `T`) — cosmetic,
   but a common source of copy-paste bugs if a future column is added by pattern-matching
   the wrong prefix.
4. **Concurrent inserts inside one request could hit `dbConn.MaxOpenConnections` limits
   from OTHER concurrent requests too** — the worker-count calculation only considers
   this single request's view of `MaxOpenConnections`, not current in-flight usage from
   other simultaneous callers.

---

## 10. Open Questions

1. Why does `flag` accept `"U"` in validation when update isn't implemented — dead code-
   path, or planned-but-unfinished feature?
2. Is `"imobile@15061981"` a rotatable secret, and should it be in config instead of
   inline code?
3. Any consumer of `GLUSR_GEO_ADDT_CONTACT` data (search/recommendation systems) — not
   traceable from this codebase; likely read directly by another service.
4. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`Location_Update_Business_Doc.md`](./Location_Update_Business_Doc.md) — product perspective

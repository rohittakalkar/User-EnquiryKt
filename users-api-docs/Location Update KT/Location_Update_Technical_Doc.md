# Location Update — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`Location_Update_Business_Doc.md`](./Location_Update_Business_Doc.md)
dekho. **Methodology**: har claim neeche real source code se trace kiya gaya hai (file path
diya gaya hai har jagah). Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — confirm with team]** likh diya hai, guess nahi kiya.

**Repos**: `service-api-go-production` (write), `users-api-go-production` (read — newly
found in this pass, purana doc mein missing tha). **Koi dedicated consumer nahi mila** jo
specifically `GLUSR_GEO_ADDT_CONTACT` table ko process karta ho — write-side insert-only hai,
read-side seedha DB hit karta hai.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write — multi-location batch insert | write | [`UserLocationsUpdateController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserLocationsUpdateController.go), [`UserLocationsUpdateModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserLocationsUpdateModel.go) (`UserLocationsDataInsert`, `UserLocationsInputValidation`) |
| Validation (mandatory params + type/range) | write | `MandatoryParamsCheckUserLocationsUpdate` — [`UserUtilsMandatory.go:993-1066`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go), `ValidateUserLocationsUpdate` — [`UsersValidationMaps.go:2107-2145`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |
| Field/length maps | write | `UserLocationsUpdateMap70`, `UserLocationsUpdateMap250` — [`UsersValidationMaps.go:83-113`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |
| DB-exec helper (endpoint-specific) | write | `ExecuteQueryUserLocationUpdate` — [`globalfunctions.go:941-957`](../../service-api-go-production/service-api-go-production/pkg/utils/globalfunctions.go) |
| **Read — `GLUSR_GEO_ADDT_CONTACT` (newly found)** | read | [`LatLongController.go`](../../users-api-go-production/internal/controllers/UsersControllers/LatLongController.go) → [`LatLongtoAddressModel.go`](../../users-api-go-production/internal/models/users/LatLongtoAddressModel.go), function `GetData()` (only when `type=3`) |
| **Adjacent-but-distinct write path (same domain, different table — do not conflate)** | write | [`UserAddLatLongController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserAddLatLongController.go) → [`UserAddLatLongModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddLatLongModel.go) (`InsertLatLong`) — writes `GLUSR_USR_ADDRESS_LAT_LONG`, not `GLUSR_GEO_ADDT_CONTACT` |
| **Adjacent-but-distinct consumer (same table family, generic profile-address trigger, not this feature's queue)** | consumers | [`USER_LATLONG_VERIFICATION.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_LATLONG_VERIFICATION.go) — fires on general profile address-field changes (`GLUSR_USR_ADD1/ADD2/CITY/STATE/LOCALITY/ZIP`), calls `LocalizationProcess`, which itself writes `GLUSR_USR_ADDRESS_LAT_LONG` ([`additional.go:2522,2541`](../../user-temp-consumers-production/user-temp-consumers-production/pkg/utils/additional.go)) |

**Why the two extra rows matter**: grepping "location"/"lat"/"long" across all three repos
surfaces a *second*, older, single-row-per-user geo table (`GLUSR_USR_ADDRESS_LAT_LONG`,
written via `POST /user/addlatlong` and via the `USER_LATLONG_VERIFICATION` consumer) that is
**not** the same feature as this doc's subject (`GLUSR_GEO_ADDT_CONTACT`, multi-row
insert-only via `POST /userlocations/update`). They are easy to confuse because both store
lat/long for a supplier. This doc's scope is **`GLUSR_GEO_ADDT_CONTACT`** only; the other
table is flagged here so nobody assumes they're the same pipeline.

---

## 2. Routes

| Method | Path / serviceName | Repo | Controller |
|---|---|---|---|
| POST | `/userlocations/update` (serviceName `USER_MULTIPLE_LOCATIONS_UPSERT`) | write | [`UserLocationsUpdateController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserLocationsUpdateController.go) — [`router.go:168,248,336`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) |
| GET/POST | `/wservce/latlongtoaddress/addressfields/*params` (serviceName `LATLONG_TO_ADDRESS`) — **the read endpoint that returns `GLUSR_GEO_ADDT_CONTACT` rows, but only when `type=3`** | read | [`LatlongToAddress`](../../users-api-go-production/internal/controllers/UsersControllers/LatLongController.go) — [`routerUsers.go:311,560,679`](../../users-api-go-production/internal/api/users_router/routerUsers.go) |
| POST | `/user/addlatlong` (serviceName `ADD_LAT_LONG_SERVICE`) — **different table, see section 1** | write | [`UserAddLatLongController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserAddLatLongController.go) — [`router.go:250,348`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) |

**Note on `/wservce/latlongtoaddress/addressfields`**: this is a multi-purpose endpoint —
`type=1`/`type=2` call external geocoding APIs (Google Maps, MMI) and don't touch
`GLUSR_GEO_ADDT_CONTACT` at all; only `type=3` reads the DB table this feature writes to.
[`LatLongController.go:74-129`](../../users-api-go-production/internal/controllers/UsersControllers/LatLongController.go),
[`LatLongtoAddressModel.go:77-115`](../../users-api-go-production/internal/models/users/LatLongtoAddressModel.go)

---

## 3. Data Model — Table

> **Verification note**: table/column names Go code ke andar embedded SQL strings se liye
> gaye hain. Live DB schema se cross-verify **nahi** kiya gaya hai.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_GEO_ADDT_CONTACT` | cslpg | Supplier geo-location + detailed address record, insert-only, multi-row-per-user | `FK_GLUSR_USR_ID`, `GLUSR_GEO_ADDT_LATITUDE`/`_LONGITUDE` (`NUMERIC(15,8)`), `GLUSR_GEO_ADDT_SOURCE` (modid), `GLUSR_GEO_ADDT_LATLONG_ADD_TYP` (loc_type), `GLUSR_GEO_ADDT_VERIFIED_STATUS` (status), `GLUSR_GEO_ADDT_ORIG_ADD`/`_UPDATED_ADD`, `GLUSR_GEO_ADDT_ROUTE`, `_STRT_ADDRESS`, `_PREMISE`, `_INTERSECTION`, `_SUBLOCALITY1..5`, `_AREA_LEVEL1..5`, `_ZIP`, `_LOCALITY`, `GLUSR_GEO_ADD_CITY`, `GLUSR_GEO_ADDT_STATE`, `_COUNTRY`, `_DIVISION`, `_MAP_URL`, `_DATE` (`CURRENT_TIMESTAMP`) — [`UserLocationsUpdateModel.go:124`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserLocationsUpdateModel.go); read-side confirms same columns lowercase — [`LatLongtoAddressModel.go:238`](../../users-api-go-production/internal/models/users/LatLongtoAddressModel.go) |
| `GLUSR_USR_ADDRESS_LAT_LONG` (**different feature, documented for disambiguation only**) | cslpg | Single/latest lat-long per user, written by `/user/addlatlong` and by the `USER_LATLONG_VERIFICATION` consumer's `LocalizationProcess` | `GLUSR_USR_ID`, `GLUSR_USR_LATITUDE`/`_LONGITUDE`, `GLUSR_LAT_LONG_CAPTURED_DATE`, `GLUSR_ADD_CHANGE_FLAG`, `GLUSR_USR_ACCURACY`, `FK_GLUSR_USR_METHOD_ID`, `glusr_lat_long_accuracy_type`, `FK_GLUSR_USR_SOURCE_ID`, `fk_iil_lat_long_verf_mast_id` — [`UserAddLatLongModel.go:21`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddLatLongModel.go), [`additional.go:2522,2541`](../../user-temp-consumers-production/user-temp-consumers-production/pkg/utils/additional.go) |

**Note**: `GLUSR_GEO_ADD_CITY` (no `T` before `_CITY`, unlike every other `GLUSR_GEO_ADDT_*`
column) is a naming inconsistency in the schema itself, not a code bug.

---

## 4. Decode-This-Magic-Value

No numeric "verification source ID"-style enum exists in this feature (unlike GST's
`FK_GST_VERIFICATION_SRC_ID`). The two magic ranges that do exist are plain
input-validated integers, not lookup codes:

| Field | Valid range | Meaning | Evidence |
|---|---|---|---|
| `loc_type` (`GLUSR_GEO_ADDT_LATLONG_ADD_TYP`) | 1-50 | **[INFERRED — confirm with team]**: no enum/lookup table found in any of the three repos for what each of the 50 values represents (e.g. "1 = factory", "2 = office"). Only the numeric bound is enforced. | [`UserUtilsMandatory.go:1051-1054`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go) |
| `status` (`GLUSR_GEO_ADDT_VERIFIED_STATUS`) | 0-2 | **[INFERRED — confirm with team]**: on the read-side, `GetData()` only returns rows where status is `0` or `1` — `2` is implicitly filtered out of `LatlongToAddress` results, which suggests `2` may mean "rejected/invalid," but this is not confirmed by any comment or named constant. | Write bound: [`UserUtilsMandatory.go:1056-1060`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go); read-side filter: [`LatLongtoAddressModel.go:270`](../../users-api-go-production/internal/models/users/LatLongtoAddressModel.go) (`if ... v["glusr_geo_addt_verified_status"].(int64) == 0 \|\| ... == 1`) |
| `flag` | `I` or `U` (validated), only `I` actually implemented | See Business Rules #4 below | [`UserUtilsMandatory.go:1048-1049`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go), [`UserLocationsUpdateController.go:18,326-329,291`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserLocationsUpdateController.go) |

---

## 5. Business Rules & Validation (code se, exhaustive)

1. **Gateway allowlist**: only `ANDROID`, `IOS` client types can call this endpoint.
   [`UserUtilsMandatory.go:999`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
2. **On successful gateway-validation, `token`/`modid` are silently overwritten** in
   `inputParams` (`token = "imobile@15061981"` hardcoded, `modid = gate`) — a hardcoded
   token-constant embedded directly in validation logic, regardless of what the caller sent.
   [`UserUtilsMandatory.go:1001-1004`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
3. **Mandatory fields** (single-mode): `latitude`, `longitude`, `usr_id`, `modid`, `token`,
   `orig_add`, `flag`, `loc_type`, `status` — missing any → `"Please Enter
   Mandatory(MODID/USERID/LATITUDE/LONGITUDE/UPDATED_ADDRESS/TOKEN/AK/ACTION_FLAG/
   LOCATION_STATUS/LOCATION_TYPE) Fields"`.
   [`UserUtilsMandatory.go:1044-1045`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
4. **Numeric checks**: `usr_id` must be numeric, `latitude`/`longitude` must be
   float-numeric, else `"USR_ID/Latitude/Longitude should be numeric"`.
   [`UserUtilsMandatory.go:1046-1047`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
5. **`flag` must be `I` or `U`** at the mandatory-check level (`"Invalid Flag Value"`
   otherwise), **but only `I` (Insert) is actually implemented downstream** — both single-mode
   (`UserLocationsUpdateController.go:326-329`) and batch-mode
   (`UserLocationsUpdateController.go:271-292`, where non-`I` sub-items get
   `onlyInsertionAllowedCount++` and the literal message `"Only Insertion is allowed"`)
   reject any `flag` that isn't exactly `"I"` without touching the DB.
6. **`loc_type` must be 1-50, `status` must be 0-2** (see section 4) — out-of-range →
   `"Invalid Location type for User"` / `"Invalid Status type for User"`.
   [`UserUtilsMandatory.go:1051-1060`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
7. **Field length limits**: `usr_id`/`latitude`/`longitude`/`zip` max 15 chars; a 70-char
   group (`modid`, `st_add`, `route`, `premise`, `intersection`, `sub_loc1-5`,
   `area_lev1-5`, `locality`, `state`, `country`, `division`); a 250-char group (`map_url`,
   `orig_add`). [`UsersValidationMaps.go:83-113, 2107-2145`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go)
8. **DB-level duplicate-protection via a genuine unique constraint**: `ON CONFLICT
   (FK_GLUSR_USR_ID, GLUSR_GEO_ADDT_LATITUDE, GLUSR_GEO_ADDT_LONGITUDE) DO NOTHING` —
   `rowsAffected==0` reliably means a true duplicate (same user, same exact lat/long already
   stored), not a race-condition guess.
   [`UserLocationsUpdateModel.go:124,136-141`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserLocationsUpdateModel.go)
9. **Batch-mode (`is_multiple=1` + `addr_arr`)**:
   - Each address is validated sequentially first, preserving order for stats/logging.
   - **In-request duplicate-detection** via a `seenRequestCoords` map keyed on a formatted
     `"lat|long"` string (8-decimal precision via `NumberFormat`) — catches duplicates
     *within the same batch* before they reach the DB.
   - **Only non-duplicate, valid addresses are dispatched as concurrent goroutine tasks**
     (`runUserLocationBatchInsertTasks`), worker-pool capped at
     `min(taskCount, 4, dbConn.MaxOpenConnections)` — see `getUserLocationWorkerCount`.
   - Each address inherits `usr_id`/`token`/`modid`/`VALIDATION_KEY` from the parent request
     and always gets `flag="I"` forced (`setBatchAddrDefaults`) — sub-items cannot
     independently request update, only insert.
   [`UserLocationsUpdateController.go:99-105,150-187,232-313`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserLocationsUpdateController.go)
10. **Single-mode (no `is_multiple`)**: same validation + insert path, one address, no
    concurrency, via `handleSingleLocation`.
    [`UserLocationsUpdateController.go:315-332`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserLocationsUpdateController.go)
11. **Read-side (`type=3` on `/wservce/latlongtoaddress/addressfields`) only surfaces rows
    with `status` `0` or `1`** — `status=2` rows are silently excluded from results (section 4).
    [`LatLongtoAddressModel.go:270`](../../users-api-go-production/internal/models/users/LatLongtoAddressModel.go)
12. **Read-side proximity gating**: if a lat/long is passed to the read endpoint, it computes
    a haversine distance to each stored row; if the closest stored row is within 200 meters,
    external geocoding API calls (Google/MMI) are **skipped** — the DB record is considered
    "close enough" to be authoritative, saving an external API round-trip.
    [`LatLongtoAddressModel.go:275-282`](../../users-api-go-production/internal/models/users/LatLongtoAddressModel.go)

---

## 6. RabbitMQ / Kafka / Redis

**Koi RabbitMQ, Kafka, ya Redis usage nahi mila** in the write-path files
(`UserLocationsUpdateController.go`, `UserLocationsUpdateModel.go`) or in the read-path files
(`LatLongController.go`, `LatLongtoAddressModel.go`). Verified via:

```
grep -n "PushToQueue\|Kafka\|Redis\|redis\|kafka" UserLocationsUpdateController.go UserLocationsUpdateModel.go
→ no matches
```

This is write-only/read-only, synchronous-DB, no async fan-out. Contrast with GST, which
fans out to 4+ databases and 2 messaging systems per write — Location Update is a much
simpler, self-contained feature.

The `USER_LATLONG_VERIFICATION` RabbitMQ consumer exists in `user-temp-consumers-production`,
but as noted in section 1, it operates on the **other** table
(`GLUSR_USR_ADDRESS_LAT_LONG`) triggered by generic profile-address changes — it is not part
of this feature's write or read path.

---

## 7. End-to-End Technical Flows

### Flow A — Single location insert

```
Supplier (Android/iOS)
    │
    ▼
[API — write]  POST /userlocations/update
                {usr_id, latitude, longitude, flag="I", loc_type, status, orig_add, ...}
    │  UserLocationsUpdateController.go
    │  Gateway(ANDROID/IOS) → token/modid silently overwritten
    ▼
UserLocationsInputValidation()  — mandatory + type/range + length checks
    │
    ├─ invalid → return validation error message, no DB call
    │
    └─ valid, flag != "I" → "Only Insertion is allowed", no DB call
    │
    └─ valid, flag == "I" →
    ▼
UserLocationsDataInsert()
    │  INSERT INTO GLUSR_GEO_ADDT_CONTACT (...) ON CONFLICT (user, lat, long) DO NOTHING
    ▼
[DB — cslpg]
    ▼
Response {"Record Successfully Inserted" / " Duplicate data"}
```

### Flow B — Multiple locations (batch)

```
Supplier (Android/iOS)
    │
    ▼
[API — write]  POST {is_multiple=1, addr_arr: [{lat,long,...}, {lat,long,...}, ...]}
    │  buildUserLocationRequestContext() — parse addr_arr out of inputParams
    ▼
handleMultipleLocations()
    │  for each address (sequential, order preserved):
    │      setBatchAddrDefaults() — inherit usr_id/token/modid, force flag="I"
    │      UserLocationsInputValidation()
    │      ├─ invalid → record failure (validationFailureCount++), skip
    │      └─ valid →
    │            buildCoordKey() → check seenRequestCoords (in-batch dedup)
    │            ├─ duplicate-in-request → mark " Duplicate data", skip DB
    │            └─ unique → queue as goroutine task
    ▼
runUserLocationBatchInsertTasks()  — worker-pool (≤4, ≤dbConn.MaxOpenConnections)
    │  concurrent UserLocationsDataInsert() calls, each own ON CONFLICT DO NOTHING
    ▼
[DB — cslpg]  GLUSR_GEO_ADDT_CONTACT  (parallel inserts)
    ▼
Response {" Postgres result: <outcome per address, pipe-separated>"}
```

### Flow C — Read back stored locations (`type=3`)

```
Caller (internal service or app)
    │
    ▼
[API — read]  GET/POST /wservce/latlongtoaddress/addressfields  {type=3, glid, ...}
    │  LatLongController.go (LatlongToAddress)
    │  validation: token/modid check, screen_name required, type in {1,2,3},
    │  glid required + numeric when type=3
    ▼
LatlongToAddressModel() → GetData()  [type == "3" branch]
    │  SELECT ... FROM glusr_geo_addt_contact WHERE fk_glusr_usr_id = $1
    │  filters to status IN (0, 1) only — status=2 rows excluded
    │  if a lat/long was also passed: haversine-distance check against each row;
    │  if closest row < 200m → skip external geocoding API calls (Google/MMI)
    ▼
[DB — csl_pg]
    ▼
Response — grouped buckets (Address AutoFill / Verified / Computed) × (Google / MMI source)
```

**Note**: `type=1` and `type=2` on this same endpoint call external geocoding APIs
(`MMIAPI`, `GoogleAPI`) directly and do **not** touch `GLUSR_GEO_ADDT_CONTACT` — only
`type=3` is a genuine read of this feature's data.
[`LatLongtoAddressModel.go:77-130`](../../users-api-go-production/internal/models/users/LatLongtoAddressModel.go)

---

## 8. Flow-wise DB & Table Usage

### Flow A — Single Insert

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | cslpg | `GLUSR_GEO_ADDT_CONTACT` | INSERT ... ON CONFLICT DO NOTHING | Single round-trip — insert the address, DB itself resolves duplicate-vs-new via the unique constraint on `(FK_GLUSR_USR_ID, LATITUDE, LONGITUDE)` |

**Total DB round-trips: 1.** This is the leanest write flow across every KT doc in this
series so far — no lookups, no secondary tables, no fan-out.

### Flow B — Batch Insert (N addresses)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1..N | cslpg | `GLUSR_GEO_ADDT_CONTACT` | INSERT ... ON CONFLICT DO NOTHING, one per non-duplicate valid address, run **concurrently** (worker-pool ≤4) | Each address gets its own conflict-checked insert; duplicates (in-DB or in-request) never reach the DB at all — filtered before dispatch |

**Total DB round-trips: up to N (one per unique, valid address in the batch)**, but they run
in parallel batches of ≤4, not N sequential round-trips — this is the main reason batch
requests don't scale linearly with size.

### Flow C — Read (`type=3`)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | csl_pg | `GLUSR_GEO_ADDT_CONTACT` | SELECT (13 columns) WHERE `fk_glusr_usr_id = $1` | Fetch all stored locations for a user, then filter/format in Go (status filter, distance calc, grouping) |
| 2 (conditional) | csl_pg | `gl_city`/`gl_state`/`gl_country` (via `getCityIds`) | SELECT with up to 20 `OR upper(...) IN (...)` clauses, batched 4 names at a time | Only runs in `analyseResult()` post-processing to resolve a `cityId` from free-text city/area-level fields — not part of the core Flow C read, but triggered by it when `zip`-lookup doesn't resolve a city |
| 3 (conditional, external) | — | Google Maps / MapmyIndia (MMI) API | HTTP GET/POST | Only when `type != 3`, OR when `type=3` and the closest stored row is ≥200m from the passed lat/long (i.e. DB data isn't "close enough") |

---

## 9. Optimization Scope — DB Response-Time Contribution

### Already well-optimized (positive findings)

1. **Concurrent worker-pool for batch-inserts** — bounded parallelism
   (`min(taskCount, 4, dbConn.MaxOpenConnections)`), avoiding both serial-slowness and
   connection-pool exhaustion.
   [`UserLocationsUpdateController.go:169-187`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserLocationsUpdateController.go)
2. **Two-layer duplicate-detection (in-request map + DB `ON CONFLICT`)** — avoids wasting DB
   round-trips on batch-internal duplicates while still relying on the DB as the source of
   truth for cross-request duplicates.
3. **Detailed micro-benchmarking already built into the response/logs**
   (`VALIDATION_TIME_SUM_MICRO`, `ITEM_PROCESS_TIME_SUM_MICRO`,
   `UNACCOUNTED_REQUEST_TIME_MICRO_EST`, etc. — [`buildUserLocationMiscData`,
   `UserLocationsUpdateController.go:379-403`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserLocationsUpdateController.go)) — suggests this endpoint has already been through a
   performance-tuning pass; any future optimization should build on this existing
   instrumentation.
4. **Read-side 200m proximity short-circuit** (`distanceCalc`,
   [`LatLongtoAddressModel.go:275-282`](../../users-api-go-production/internal/models/users/LatLongtoAddressModel.go)) avoids two external API calls (Google + MMI)
   when a DB row is already close enough — a genuine latency/cost saver on the read path.

### Medium impact

5. **`analyseResult()`'s city-resolution fallback (`getCityIds`) runs a wide, `OR`-heavy
   query against `gl_city`/`gl_state`/`gl_country`** for every read result whose `zip`
   lookup doesn't resolve a city — up to 20 placeholder params per batch of names, run in a
   loop of batches of 4. On a user with several stored locations lacking clean zip data,
   this could mean multiple such queries per single read request. **Concrete fix**: cache
   city-name→city-id resolution (locality/city names change rarely) instead of re-querying
   per request. [`LatLongtoAddressModel.go:1105-1225`](../../users-api-go-production/internal/models/users/LatLongtoAddressModel.go)
6. **No caching on the read endpoint** — `GET /wservce/latlongtoaddress/addressfields`
   (`type=3`) hits `cslpg`/`csl_pg` live on every call. Stored locations change relatively
   infrequently (insert-only, no update path — section 5.5) which makes this a reasonable
   caching candidate if read volume is high. **[INFERRED — confirm actual read QPS with
   team before prioritizing]**.

### Low impact

7. **Worker-pool cap of 4 is a hardcoded constant** — if this endpoint regularly receives
   large batches and DB capacity allows more concurrency, this cap could be revisited;
   conversely, `getUserLocationWorkerCount` only reads `dbConn.Stats().MaxOpenConnections`
   at the moment of the call, not current in-flight usage from other simultaneous callers
   sharing the same pool.
   [`UserLocationsUpdateController.go:169-187`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserLocationsUpdateController.go)

---

## 10. Cron Inventory

Grepped both `service-api-go-production/crons/` and
`user-temp-consumers-production/internal/Workers/*cron*` for anything touching
`location`/`GEO_ADDT`/`LATLONG`:

```
find service-api-go-production/service-api-go-production -iname "*cron*"
→ recommend/autogenerate_json_cron.go, cron_tracker.go, gst_tact_veri_cron.go,
  mcatcrontest.go, rating_suspect_cron.go, Supp_verify_log_cron.go
→ none reference GLUSR_GEO_ADDT_CONTACT, location update, or lat/long

find user-temp-consumers-production/user-temp-consumers-production -iname "*cron*"
→ gst_tact_cron_sync.go (GST-only, see GST_Technical_Doc.md)
```

**No cron touches this feature.** Confirmed absent, not assumed absent.

---

## 11. Edge Cases & Gotchas (technical POV)

1. **`flag="U"` passes the mandatory-check but is functionally rejected later** with
   `"Only Insertion is allowed"` — validation and actual-capability are inconsistent; a
   caller reading only `MandatoryParamsCheckUserLocationsUpdate` might assume update is
   supported.
2. **Hardcoded token-constant `"imobile@15061981"`** embedded in validation logic, and
   silently written into every request's `inputParams` after gateway validation — worth
   confirming this isn't sensitive/rotatable secret material that should be externalized.
   [`UserUtilsMandatory.go:1002`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
3. **`GLUSR_GEO_ADD_CITY` naming inconsistency** in the schema (missing `T`) — cosmetic,
   but a common source of copy-paste bugs if a future column is added by pattern-matching
   the wrong prefix.
4. **Two lat/long tables exist in this codebase family** (`GLUSR_GEO_ADDT_CONTACT` vs
   `GLUSR_USR_ADDRESS_LAT_LONG`, section 1) — easy to confuse when investigating a location
   bug; check which endpoint/consumer is actually implicated before assuming it's this
   feature.
5. **`loc_type` (1-50) has no documented enum anywhere in the three repos** — the meaning
   of specific values is entirely tribal knowledge outside this codebase (section 4).
6. **`status=2` rows are silently excluded from reads** with no code comment explaining why
   — behaves like a soft-delete/reject flag but this is inferred, not confirmed (section 4).
7. **Concurrent inserts inside one batch request could interact with `MaxOpenConnections`
   pressure from OTHER concurrent requests too** — the worker-count calculation only
   considers this single request's snapshot of `dbConn.Stats().MaxOpenConnections`, not
   current in-flight usage from other simultaneous callers on the shared pool.
8. **The read endpoint (`type=3`) is far more complex than the write endpoint** — it
   involves haversine distance math, external API fallback, city-ID resolution via a wide
   `OR`-heavy query, and response bucketing by source — none of which is visible from the
   write side, so a "why is this read slow" investigation needs a different mental model
   than the batch-insert write path.

---

## 12. Open Questions

1. Why does `flag` accept `"U"` in validation when update isn't implemented — dead code
   path, or planned-but-unfinished feature?
2. Is `"imobile@15061981"` a rotatable secret, and should it be in config instead of inline
   code?
3. What do the 50 `loc_type` values mean (factory / office / warehouse / etc.)? No enum
   found in any of the three repos.
4. What does `status=2` mean on `GLUSR_GEO_ADDT_VERIFIED_STATUS`? Read-side excludes it but
   no comment/constant confirms the intended meaning (rejected? pending-deletion? something
   else?).
5. Any downstream consumer of `GLUSR_GEO_ADDT_CONTACT` data (search/recommendation/geo-index
   systems)? Not traceable from these three repos — likely read directly by another service
   not in scope of this KT.
6. Live DB schema verification (column types, nullability, indexes, constraints) — this doc
   reflects only what Go SQL strings imply.

---

## See also

- [`Location_Update_Business_Doc.md`](./Location_Update_Business_Doc.md) — product perspective, no code
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — depth/structure reference used to rebuild this doc

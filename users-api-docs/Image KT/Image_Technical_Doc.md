# User Image — Technical Doc (Code-Level Deep Dive)

**Scope note**: yeh doc sirf **Image** (`GLUSR_USR_IMAGE`) cover karta hai — **Logo**
(`GLUSR_USR_LOGO`, apna approval-workflow ke saath) genuinely alag table/system hai, dekho
[`../Logo KT/Logo_Technical_Doc.md`](../Logo%20KT/Logo_Technical_Doc.md). Dono
`FK_GLUSR_USR_ID` se supplier se link hain, ek doosre se FK-linked nahi.

Business/product perspective ke liye [`Image_Business_Doc.md`](./Image_Business_Doc.md) dekho.

**Repos involved**: `service-api-go-production` (write), `users-api-go-production` (read —
missed by the earlier pass of this doc), `user-temp-consumers-production` (generic
replication/CDC workers — touched, but **not specific to this feature**, see §6).

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Image write (controller) | write | [`UserImageController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserImageController.go) |
| Image write (model/query) | write | [`UserImageModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserImageModel.go) |
| Field-map / validation rules | write | [`UsersValidationMaps.go:349-373`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) (`UserImageMap`), [`UsersValidationMaps.go:2176-2185`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) (`ValidationUserImage`) |
| Mandatory-field check | write | [`UserUtilsMandatory.go:1277-1280`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go) (`MandatoryFieldsUserImage`) |
| Route registration | write | [`router.go:150,318`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) |
| **Image read** (via shared `/detail` endpoint) | read | [`UserDetailController.go`](../../users-api-go-production/internal/controllers/UsersControllers/UserDetailController.go), [`UserDetailModel.go:380-388,545-563,664-670,822-831,1233-1248,1515-1560`](../../users-api-go-production/internal/models/users/UserDetailModel.go) |
| Read route registration | read | [`routerUsers.go:172-173,496-497,602-603`](../../users-api-go-production/internal/api/users_router/routerUsers.go) |
| Generic core-profile replication worker (touches a DB nicknamed "imagePg" — **not** the `GLUSR_USR_IMAGE` table, see §6) | consumer | [`USER_UPSERT_CONSUMERS.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_CONSUMERS.go) |
| Generic Debezium/Kafka CDC sync worker (same "imagePg" DB nickname) | consumer | [`DebeziumSyncScript_GlusrUsr.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/DebeziumSyncScript_GlusrUsr.go) |

**What the earlier pass of this doc missed**: it listed only the write controller/model and
concluded "no consumer found, no read side found." Both were wrong/incomplete:
1. A read path **does** exist — `GLUSR_USR_IMAGE` is read as a sub-query inside the shared
   `/detail` (`GET`/`POST`) endpoint in `users-api-go-production`, not a dedicated
   `GET /user/image`.
2. Workers named around "imagePg"/`IMAGE_PG` **do** exist in `user-temp-consumers-production`,
   but tracing them (§6) shows they replicate the **core `glusr_usr` table** into a Postgres
   instance that is historically *nicknamed* "Image PG" — a naming collision, not the
   `GLUSR_USR_IMAGE` table used by this feature. This is called out explicitly so nobody
   re-discovers it and assumes a queue-based fan-out exists for Image uploads (it doesn't).

---

## 2. Routes

| Method | Path | Repo | Controller | Purpose |
|---|---|---|---|---|
| POST | `/user/image` | write (`service-api-go-production`) | `UserImageController` — [`router.go:150`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) (also registered again at [`router.go:318`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) inside `SetupUsersFailoverCluster2()` — **same controller, failover cluster wiring, not a second endpoint**) | Upsert image record |
| GET / POST | `/detail/*params` | read (`users-api-go-production`) | `UserDetailController` — [`routerUsers.go:172-173`](../../users-api-go-production/internal/api/users_router/routerUsers.go) (repeated at lines 496-497, 602-603 for other cluster setups) | Returns full supplier profile, **including image sub-object** when `others=ALL` or `logo=1` is passed |

---

## 3. Data Model — Table

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_USR_IMAGE` | `meshpg` (write side, [`UserImageController.go:110`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserImageController.go)); read side uses the connection wired to `users-api-go-production`'s `/detail` query, table referenced in lowercase as `glusr_usr_image` — [`UserDetailModel.go:386,668`](../../users-api-go-production/internal/models/users/UserDetailModel.go) | General branding-image record, **no approval-ID/history-versioning** (unlike Logo) | `FK_GLUSR_USR_ID`, `GLUSR_USR_IMAGE_IMG_32X32`/`_32X32_WH`, `_64X64`/`_64X64_WH`, `_125X125`/`_125X125_WH`, `_250X250`/`_250X250_WH`, `_IMG_ORIG`/`_ORIG_WH`, `GLUSR_USR_IMAGE_STATUS`, `GLUSR_USR_IMAGE_REJECT_REASON`, `GLUSR_USR_IMAGE_UPDATEDBY`, `GLUSR_USR_IMAGE_UPDATEDBY_ID` (present in field-map, [`UsersValidationMaps.go:363`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go), **but not written by `UpsertImagePg`** — see Edge Cases §9.3), `_UPDATESCREEN`, `_IP`/`_IP_COUNTRY`, `_UPDATEDUSING`, `_HIST_COMMENTS`, `_UPDATED_TIME`, `_UPDATEDBY_URL`, `_UPDATE_AGENCY` (in map, also unused by write query) — [`UserImageModel.go:137-157,179-200`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserImageModel.go), [`UsersValidationMaps.go:349-373`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |

**Verification note**: column/table names above come purely from SQL strings embedded in the
Go source, not from a live-checked DB schema — actual DDL may have additional
columns/constraints not visible here.

**Note**: har image-size ke saath ek `_WH` (width-height) companion-column bhi hai — Logo table
mein yeh nahi tha, ek extra tracking-detail jo Image-specific hai.

---

## 4. Read-Side Behaviour — "decode this" section

The read path (`/detail`) decides **which image resolution to return** based on two request
params, `logo` and `others`, both string flags:

| Param value | Behaviour | Evidence |
|---|---|---|
| `others == "ALL"` (case-insensitive) | Sub-query selects `glusr_usr_image_img_250x250, _125x125, _64x64, updated_time, status`, then `getGlusrprofileimage()` picks the **first non-empty of 250→125→64** (largest-available preference) and returns it as `GLUSR_USR_IMAGE` / `q_result["image"]` | [`UserDetailModel.go:380-388,545-563,1544-1557`](../../users-api-go-production/internal/models/users/UserDetailModel.go) |
| `logo == "1"` | Same sub-query columns, gated by the `logo` param name (**misleadingly named** — it is actually gating the `GLUSR_USR_IMAGE` sub-query, not the separate Logo/`GLUSR_USR_LOGO` table) — returns `64x64` image plus `GLUSR_USR_IMAGE_STATUS`/`_UPDATED_TIME` (formatted `DD-MON-YY`) — [`UserDetailModel.go:664-670,822-831,1518-1543`](../../users-api-go-production/internal/models/users/UserDetailModel.go) | **[INFERRED — confirm with team]** whether `logo=1` was originally meant to read the Logo table and was repointed to Image, or was always intentionally an Image-only flag despite the name. |
| Neither flag set | `image` key defaults to `""` empty string in the response — [`UserDetailModel.go:559,562`](../../users-api-go-production/internal/models/users/UserDetailModel.go) | — |

This resolution-fallback (`250 → 125 → 64`, largest available wins) is the only "decode this
magic value" logic found for Image; `STATUS` itself is a plain passthrough of whatever was
written (see §5.2), not decoded/mapped to a label anywhere in the read path.

---

## 5. Business Rules & Validation (code se)

1. **UPDATE-first, INSERT-fallback pattern**: query pehle
   `UPDATE GLUSR_USR_IMAGE ... WHERE FK_GLUSR_USR_ID=$20` try karti hai; `affectedRows > 0`
   check karke, agar 0 hon, tabhi `INSERT INTO GLUSR_USR_IMAGE (...)` chalta hai — the caller
   never has to send an explicit insert/update flag.
   [`UserImageModel.go:137,157,175-179`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserImageModel.go)
2. **Mandatory fields**: `USR_ID`, `UPDATEDBY`, `UPDATESCREEN`, `VALIDATION_KEY`, `STATUS` — if
   any is missing, request fails with
   `"Please Enter Mandatory(USR_ID/UPDATEDBY/UPDATESCREEN/VALIDATION_KEY/STATUS) Fields"`.
   [`UserImageController.go:118,136-139`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserImageController.go)
3. **`STATUS` must be one of `P`, `D`, `A`, `Y`** (Pending/Deleted/Approved/Yes-equivalent —
   exact meanings **[INFERRED — confirm with team]**, no comment/mapping found in code);
   anything else → `"Invalid Status"`.
   [`UserImageController.go:119,152-155`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserImageController.go)
4. **Gateway/validation-key check**: `VALIDATION_KEY` must resolve via `Gateway_v1` against the
   allow-list `["BUYERMY", "MAPI", "SELLERMY", "ANDROID", "IOS"]`; failing this →
   `"Input gateway not validated"`.
   [`UserImageController.go:92-93,141-142`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserImageController.go)
5. **Length/type validation** against `UserImageMap` (statuses `["1","2","3"]` passed to
   `LengthAndTypeValidations_v2` — meaning of these status-codes for the validator itself is
   internal to `utils`, not re-derived here) — e.g. `STATUS` max length 1,
   `REJECT_REASON` max length 255, image-URL columns max length 500,
   `_WH` columns max length 200.
   [`UsersValidationMaps.go:2176-2185,349-373`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go)
6. **Optional employee-lookup side read**: if `UPDATEDBY_ID` is present and non-empty/non-"0",
   `Employee_mesh_pg()` is called before validation — an extra DB round-trip purely to validate
   the employee ID exists; failure here aborts the whole request before the image
   upsert even runs.
   [`UserImageController.go:127-130`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserImageController.go)
7. **`STATUS`/`REJECT_REASON` columns exist, but there is no separate approval-ID/versioning-
   history** table or column set the way Logo has — a simpler, single-record moderation model.
   No moderation-consumer or admin-tool that actively sets these was found in this pass (see
   Open Questions §12).
8. **`_WH` (width-height) companion columns per resolution** — accepted as free-form strings,
   no numeric/format validation beyond the generic 200-char length cap.
   [`UserImageModel.go:29-33,41-45,53-57,65-69,77-81`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserImageModel.go)
9. **Response normalisation**: internally `"INSERT SUCCESS"` and `"UPDATE SUCCESS"` are distinct
   outputs, but the controller **rewrites `INSERT SUCCESS` to `UPDATE SUCCESS`** before sending
   the response body — callers cannot distinguish insert vs. update from the message text (only
   from server-side Kibana logs, which retain the real value in `PG_OUTPUT`).
   [`UserImageController.go:222-225,245`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserImageController.go)

---

## 6. RabbitMQ

**No RabbitMQ publish/consume found inside the Image write path itself**
(`UserImageController.go`/`UserImageModel.go` — grepped for `rabbitmq|amqp|Publish`, no hits).

However, a **generic, queue-name-driven replication consumer** exists in
`user-temp-consumers-production` that is wired to a DB connection nicknamed `imagePg`:

| Queue / consumer group | Purpose (as coded) | Evidence |
|---|---|---|
| `USER_UPSERT_IMAGEPG`, `USER_UPSERT_IMAGEPG_BULK` | Consumes core-profile column changes (via `UserImagePgMap`, which lists fields like `GLUSR_USR_FIRSTNAME`, `GLUSR_USR_EMAIL`, `GLUSR_USR_COMPANYNAME`, `GLUSR_USR_ADD1/2`, etc. — **not** any `GLUSR_USR_IMAGE_*` column), and upserts them into `glusr_usr` (default table branch, no `IMAGEPG`-specific table override) on the DB connection named `imagePg` | [`USER_UPSERT_CONSUMERS.go:32-33,128-130,299-311`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_CONSUMERS.go), [`map_index.go:371+`](../../user-temp-consumers-production/user-temp-consumers-production/pkg/utils/map_index.go) |

**Conclusion (important, corrects a likely misread)**: `imagePg` is a **physical Postgres
instance nickname** ("Image PG" as an infra/replica name), used to replicate the *core user
profile row* (`glusr_usr`) — it is **not** a queue for the `GLUSR_USR_IMAGE` table and has no
relationship to supplier photo uploads. **[INFERRED — confirm with team]**: the historical
reason this replica DB is called "Image PG" is not discoverable from code; flagged to Open
Questions rather than guessed at.

**Net finding**: the Image-upload write itself is purely synchronous, single-table, no
RabbitMQ fan-out — same conclusion as the earlier pass, but now backed by an explicit trace of
the only queue names that share the word "image" in this codebase, ruling them out rather than
just not finding them.

---

## 7. Kafka

A **generic Debezium/Kafka CDC sync worker** (`DebeziumSyncScript_GlusrUsr.go`, using
`github.com/segmentio/kafka-go`) also has an `IMAGE_PG` branch:

```go
} else if DB_NAME == "IMAGE_PG" {
    ConsumerGroup = "GLUSR_USR_DEBEZIUM_SYNC_IMAGEPG"
    subject       = "GLUSR_USR_DEBEZIUM_SYNC_IMAGEPG"
    db_conn_name  = "imagePg"
```
[`DebeziumSyncScript_GlusrUsr.go:73-76`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/DebeziumSyncScript_GlusrUsr.go)

Same conclusion as §6 — this is CDC sync of the core `GLUSR_USR` row into the "imagePg" replica
instance (source query is `SELECT * FROM GLUSR_USR WHERE GLUSR_USR_ID = $1`,
[`DebeziumSyncScript_GlusrUsr.go:369`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/DebeziumSyncScript_GlusrUsr.go)),
**not** a Kafka pipeline for `GLUSR_USR_IMAGE`/photo data. **No Kafka usage was found that is
actually scoped to the Image-upload feature** (`UserImageController`/`UserImageModel` have zero
Kafka imports or references).

---

## 8. Redis

**No Redis usage found** in `UserImageController.go`, `UserImageModel.go`, or the read-side
image-fetch code in `UserDetailModel.go` (grepped `redis|Redis`, no hits in any of these
files). Image data appears to be read fresh from Postgres on every `/detail` call — no caching
layer confirmed for this specific sub-object.

---

## 9. End-to-End Technical Flows

### 9.1 Write — Supplier uploads/updates image

```
Supplier / client app
    │
    ▼
[API — write, service-api-go-production]  POST /user/image
    {USR_ID, IMG_32X32, IMG_32X32_WH, ..., IMG_ORIG, IMG_ORIG_WH, STATUS, REJECT_REASON,
     UPDATEDBY, UPDATESCREEN, VALIDATION_KEY, UPDATEDBY_ID?, ...}
    │  UserImageController.go
    ▼
[Gateway check] VALIDATION_KEY against BUYERMY/MAPI/SELLERMY/ANDROID/IOS allow-list
    │
    ▼
[Optional DB]  Employee_mesh_pg(UPDATEDBY_ID)  — only if UPDATEDBY_ID present & != "0"
    │
    ▼
[Mandatory-field check]  USR_ID/UPDATEDBY/UPDATESCREEN/VALIDATION_KEY/STATUS
    │
    ▼
[Length/type validation]  against UserImageMap
    │
    ▼
[STATUS enum check]  must be one of P/D/A/Y
    │
    ▼
[DB]  UPDATE GLUSR_USR_IMAGE SET ... WHERE FK_GLUSR_USR_ID=$N
    │
    ├─ Rows affected > 0 → "UPDATE SUCCESS"
    │
    └─ 0 rows affected →
          [DB]  INSERT INTO GLUSR_USR_IMAGE (...)
          ├─ success → "INSERT SUCCESS" (but rewritten to "UPDATE SUCCESS" in response body)
          └─ failure → "INSERTION QUERY FAILED IN PG: <err>"
    ▼
Response {STATUS, CODE, MESSAGE, SERVICE_NAME, RESPONSE_DATA}
```

### 9.2 Read — Supplier profile fetch includes image

```
Client (buyer-facing storefront / app / supplier dashboard)
    │
    ▼
[API — read, users-api-go-production]  GET or POST /detail/{glusrid}?others=ALL  (or &logo=1)
    │  UserDetailController.go → users.Getusercontactdetail()
    ▼
[DB]  single combined query joins glusr_usr, glusr_usr_ext, and a correlated sub-query:
      SELECT ... json_agg(...) FROM (
        SELECT glusr_usr_image_img_250x250, _125x125, _64x64, updated_time, status
        FROM glusr_usr_image WHERE fk_glusr_usr_id = $1
      ) AS glusr_usr_images
    │
    ▼
[In-memory]  getGlusrprofileimage() picks:
      - others=ALL  → first non-empty of 250x250 → 125x125 → 64x64
      - logo=1      → 64x64 image + STATUS + UPDATED_TIME (formatted DD-MON-YY)
    │
    ▼
Response  {..., "image": "<url or empty string>", ...}
```

---

## 10. Flow-wise DB & Table Usage Matrix

### Flow 9.1 — Write (first-time upload, i.e. UPDATE misses)

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1 | meshpg | *(none — employee lookup helper's own table, out of scope)* | SELECT | Only if `UPDATEDBY_ID` present — validates employee ID via `Employee_mesh_pg()` [`UserImageController.go:129`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserImageController.go) |
| 2 | meshpg | `GLUSR_USR_IMAGE` | UPDATE (attempted) | Update-first pattern; 0 rows affected for a new supplier [`UserImageModel.go:137-157`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserImageModel.go) |
| 3 | meshpg | `GLUSR_USR_IMAGE` | INSERT | Fallback after failed UPDATE [`UserImageModel.go:179-200`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserImageModel.go) |

### Flow 9.1 — Write (returning supplier, UPDATE hits)

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1 | meshpg | *(employee lookup, optional)* | SELECT | Same as above, only if `UPDATEDBY_ID` present |
| 2 | meshpg | `GLUSR_USR_IMAGE` | UPDATE | Existing record found, 1 round-trip total [`UserImageModel.go:137-157,175-177`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserImageModel.go) |

### Flow 9.2 — Read (`/detail`, others=ALL)

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1 | (read DB behind `users-api-go-production`, not separately confirmed as meshpg by this pass) | `glusr_usr`, `glusr_usr_ext`, `glusr_usr_image` (correlated sub-query) | SELECT (single combined query) | Fetch full profile plus image sub-object in one round-trip [`UserDetailModel.go:380-388`](../../users-api-go-production/internal/models/users/UserDetailModel.go) |

---

## 11. Optimization Scope — DB / Response-Time Contributors

### Medium-impact

1. **Optional employee-lookup (`Employee_mesh_pg`) runs before validation, not after** — if the
   employee ID is invalid, the request has already paid for an extra DB round-trip before any
   of the cheap in-memory validations (mandatory-fields, length, STATUS-enum) run. Reordering
   to validate-cheap-first would save a DB hit on malformed requests.
   [`UserImageController.go:127-139`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserImageController.go)

### Low-impact

2. **Update-then-insert-fallback is a predictable 1-2-query pattern** — first-time upload = 2
   round-trips (failed UPDATE + INSERT), later updates = 1 round-trip. If first-time-upload is
   a common case (new suppliers), an `INSERT ... ON CONFLICT (FK_GLUSR_USR_ID) DO UPDATE` would
   make it always 1 round-trip — same suggestion already made in GST/Rating/Social-Review docs
   for this exact pattern, so likely a repo-wide easy win if a unique constraint on
   `FK_GLUSR_USR_ID` exists (unconfirmed, see Open Questions).
3. **`/detail` combines profile + image in a single query with a `json_agg` sub-select**
   — reasonably efficient (1 round-trip for both), no obvious optimization opportunity found
   here; flagged only as good-practice-already-present.

### Low-impact / good practice already present

4. **No async complexity, no consumer fan-out for the write path** — simple, predictable,
   low-risk architecture, same as Blocking/Social-Review modules.

---

## 12. Cron Inventory

**No cron job specific to `GLUSR_USR_IMAGE` was found.** Searched
`user-temp-consumers-production` and `users-api-go-production` for cron-style schedulers
touching image-related files/tables; none found. This does not rule out a cron elsewhere in
the broader IndiaMART estate that was not part of the repos scoped for this KT.

---

## 13. Edge Cases & Gotchas (technical POV)

1. **Update-then-insert race-condition risk**: if two concurrent requests hit a brand-new
   supplier (both UPDATE calls return 0 rows), both could INSERT — if `FK_GLUSR_USR_ID` has no
   unique constraint, duplicate rows are possible. **[INFERRED — confirm with team]**, no DDL
   available to verify.
2. **`GLUSR_USR_IMAGE_STATUS`/`REJECT_REASON` write path exists, but no consumer/admin-tool
   that actively *sets* them was found** in this pass beyond the client-supplied value on
   `/user/image` itself — worth confirming whether a moderation workflow exists outside these
   repos.
3. **`UPDATEDBY_ID`, `UPDATE_AGENCY` are defined in `UserImageMap`/validated, but never appear
   in the actual `UPDATE`/`INSERT` SQL built by `UpsertImagePg`** — they're accepted and
   validated but silently dropped, never persisted.
   [`UsersValidationMaps.go:363,365`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) vs.
   [`UserImageModel.go:137-157,179-200`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go)
4. **Response message masking**: `INSERT SUCCESS` is rewritten to `UPDATE SUCCESS` before being
   returned to the caller — anyone debugging client-observed behaviour purely from the response
   body (rather than Kibana logs) will not be able to tell insert vs. update apart.
   [`UserImageController.go:222-225`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserImageController.go)
5. **The `logo` request param on `/detail` is a misleading name** — it gates the
   `GLUSR_USR_IMAGE` sub-query (not the actual Logo/`GLUSR_USR_LOGO` table), which could confuse
   anyone reading only the read-side code without this doc.
   [`UserDetailModel.go:664-670,822-831`](../../users-api-go-production/internal/models/users/UserDetailModel.go)
6. **"imagePg"/`IMAGE_PG` DB nickname in the consumers repo is a false lead** — it replicates
   core `glusr_usr` fields, not `GLUSR_USR_IMAGE`; documented explicitly in §6/§7 to prevent
   future confusion.

---

## 14. Open Questions

1. Does `FK_GLUSR_USR_ID` on `GLUSR_USR_IMAGE` have a unique constraint (race-condition
   prevention for concurrent first-time uploads)? Not verifiable from Go code alone.
2. Who/what sets `STATUS`/`REJECT_REASON` in practice — is there a moderation admin-tool outside
   the three scoped repos?
3. What does `STATUS` values `P`/`D`/`A`/`Y` mean exactly (Pending/Deleted/Approved/... )? No
   comment or mapping found in code.
4. Why is the Postgres replica in `user-temp-consumers-production` nicknamed "imagePg" if it
   only carries core `glusr_usr` fields and has nothing to do with `GLUSR_USR_IMAGE`? Historical
   naming, not discoverable from code.
5. Is there a resize/moderation pipeline (actual image processing — thumbnailing to
   32/64/125/250) upstream of this API, e.g. in a CDN/asset service not in these three repos?
   The controller only accepts pre-sized URLs as strings; no image-processing code was found in
   scope.
6. Live DB schema verification — this doc, like the previous pass, reflects only what Go SQL
   strings imply, not a live-checked schema.
7. Which physical DB backs the `users-api-go-production` `/detail` read query (confirmed same
   `meshpg`-family Postgres as the write side, or a replica)? Not conclusively confirmed in this
   pass.

---

## See also

- [`Image_Business_Doc.md`](./Image_Business_Doc.md) — product perspective
- [`../Logo KT/Logo_Technical_Doc.md`](../Logo%20KT/Logo_Technical_Doc.md) — related but
  separate approval-workflow-based logo system
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — depth/structure
  reference this doc was rebuilt against

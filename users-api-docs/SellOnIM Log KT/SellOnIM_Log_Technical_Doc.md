# SellOnIM Log — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`SellOnIM_Log_Business_Doc.md`](./SellOnIM_Log_Business_Doc.md)
dekho — dono docs same flows cover karte hain, bas alag audience ke liye.

**Repos**: `service-api-go-production` (write), `user-temp-consumers-production`
(RabbitMQ-driven calling-eligibility pipeline). **Koi read-controller `sellonim_log` ke liye
kisi bhi repo mein nahi mila** (`users-api-go-production` mein bhi `sellonim` string kahin
match nahi hui, sirf docs mein) — yeh ek write-only journey-log hai jisme ek conditional
downstream fan-out attach hai.

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file path diya
gaya hai har jagah). Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — team se confirm karo]** likh diya hai, guess nahi kiya.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write — controller | write | [`UserSellonimLogController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSellonimLogController.go) — `UserSellonimLogContrller` |
| Write — model | write | [`UserSellonimlogModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSellonimlogModel.go) — `SellonIMLogUpdateintoBD`, `getDBValuesforGlid`, `SellonIMRabbitMQ`, `ValidRFC3339Date` |
| Validation map | write | `MapSellonIM` — [`UsersValidationMaps.go:566-583`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go), consumed via `LengthAndTypeValidations_v1` |
| Route registration | write | [`router.go:156,284,324`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) — `POST /user/sellonimlog` registered **three times** across what look like separate router-group blocks (see section 2 note) |
| JWT/AK-middleware exemption | write | `exception_service` list — [`requestValidation.go:26`](../../service-api-go-production/service-api-go-production/pkg/middleware/requestValidation.go) includes `/user/sellonimlog`, and is checked at [`requestValidation.go:723`](../../service-api-go-production/service-api-go-production/pkg/middleware/requestValidation.go) to skip the outer JWT/AK-signature validation entirely for this route — the controller's own `VALIDATION_KEY`/`Gateway_v1` allowlist check (section 4, rule 1) is the *only* auth gate on this endpoint |
| Queue-routing config | write | `serviceToQueueMap` — [`rabbitmq.go:30`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go): `"USER_SELLONIM_LOG": "USER_SELLONIM_CALLING"` |
| Consumer registration | consumers | [`IntializeMsgBroker.go:62`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go) maps `"USER_SELLONIM_CALLING"` queue-name → `workers.UserSellOnImCalling`; [`router.go:219`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Router/router.go) maps the same string → `dbActionUserSellOnImCalling` (the actual per-message handler) |
| Consumer — calling-eligibility pipeline | consumers | [`USER_SELLONIM_CALLING.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SELLONIM_CALLING.go) — `UserSellOnImCalling` (bootstrap), `dbActionUserSellOnImCalling` (dispatcher), plus 5 gate-check helpers: `getVerifcationLogData`, `getFcpDetailsData`, `getDuplicateEmailData`, `getDuplicateGSTData`, `getDuplicateGlid`, and the final writer `insertDetail_supp_verification` |

---

## 2. Routes

| Method | Path | serviceName (in payload, not URL) | Repo | Controller |
|---|---|---|---|---|
| POST | `/user/sellonimlog` | `USER_SELLONIM_LOG` | write | `UserSellonimLogContrller` |
| — | RabbitMQ queue `USER_SELLONIM_CALLING` | `USER_SELLONIM_LOG` (as the `SERVICENAME` field inside the published message, routed to this queue by `serviceToQueueMap`) | consumers | `dbActionUserSellOnImCalling` |

**Note on triple registration**: `router.go` registers `POST /user/sellonimlog →
UserSellonimLogContrller` at three separate line numbers (156, 284, 324) inside what appear to
be three different `gin.RouterGroup` blocks (one uses `middleware.Validation`, comment
`//approval pg` at line 284). **[INFERRED — confirm with team]**: this is very likely the
same route mounted under different middleware-group prefixes/environments (e.g. versioned
API groups) rather than a genuine duplicate/conflict — not conclusively traced in this pass.

---

## 3. Data Model — Tables

> **Verification note**: yeh sab table/column names Go code ke andar embedded SQL strings se
> liye gaye hain (har row ke saamne file:line diya hai). Live DB schema se pgAdmin pe
> cross-verify **nahi** kiya gaya hai.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `sellonim_log` | approvalpg (write) | Onboarding-journey progress log — one row per (GLID, LOG_ID) | `sellonim_log_id` (PK, `RETURNING` on INSERT), `fk_glusr_usr_id`, `sellonim_mob_number`, `sellonim_fcp_status`, `sellonim_custtype_id`, `sellonim_journey_completed`, `sellonim_modid`, `sellonim_best_time_to_call`, `is_usr_name_entered`, `is_companyname_entered`, `is_email_entered`, `total_products_entered`, `is_address_entered`, `is_gst_entered`, `glusr_usr_approv`, `fk_parent_glusr_id`, `sellonim_add_date`, `sellonim_upd_date` — [`UserSellonimlogModel.go:117-119`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSellonimlogModel.go), column-name mapping in `MapSellonIM` [`UsersValidationMaps.go:566-583`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |
| `glusr_usr` | approvalpg (write-side lookup) | Source of `FK_PARENT_GLUSR_ID`/`GLUSR_USR_APPROV` merged into every `sellonim_log` write | queried by `glusr_usr_id=$1` — [`UserSellonimlogModel.go:180-206`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSellonimlogModel.go) |
| `iil_supp_verification_log` | meshPg (consumer, Gate 1 read-only) | Existing verification-flag history, used as a disqualification-gate | `FK_GLUSR_USR_ID`, `VERIFY_FLAG` (`-1`/`-2`/`-7` = disqualifying) — [`USER_SELLONIM_CALLING.go:169`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SELLONIM_CALLING.go) |
| `active_fcp_details` | meshPg (consumer, Gate 2 read-only) | FCP priority-range lookup | `FK_GLUSR_USR_ID`, `PRIORITY_RANGE` (must be `0.8` or `0.9` to pass) — [`USER_SELLONIM_CALLING.go:196`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SELLONIM_CALLING.go) |
| `glusr_usr` | meshPg (consumer, Gate 3 read-only — separate connection/DB from the write-side `glusr_usr` above) | Duplicate-email detection (self-join) | `glusr_usr_id`, `GLUSR_USR_EMAIL_DUP`, `fcp_flag`, `GLUSR_USR_CUSTTYPE_ID` — [`USER_SELLONIM_CALLING.go:225`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SELLONIM_CALLING.go) |
| `custtype` | meshPg (consumer, Gate 3 read-only, subquery) | Paid-customer-type lookup used inside the Gate 3 query | `CUSTTYPE_ID`, `paid` (`-1` = paid) — [`USER_SELLONIM_CALLING.go:225`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SELLONIM_CALLING.go) |
| `GLUSR_USR_COMP_REGISTRATIONS` | meshPg (consumer, Gate 4 read-only) | Duplicate-GST detection (self-join) | `gst`, `FK_GLUSR_USR_ID` — [`USER_SELLONIM_CALLING.go:249`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SELLONIM_CALLING.go) |
| `iil_supp_verification` | meshPg (consumer — Gate 5 read + final write target) | The actual "calling/verification queue" — final destination once all 5 gates pass | `Iil_supp_verif_glusr_id`, `IIL_SUPP_VERIF_ALLOCAT_DATE`, `Iil_supp_verif_data_source` (`'QGFCP ACTIVE'`), `IIL_SUPP_VERIF_CUSTTYPE_WT` (`1399`), `iil_supp_verif_priority` (`1`), `iil_supp_verif_type` (`'VGFCP'`), `BEST_TIME_TO_CALL` — Gate 5 read at [`USER_SELLONIM_CALLING.go:273`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SELLONIM_CALLING.go), final INSERT at [`USER_SELLONIM_CALLING.go:295-308`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SELLONIM_CALLING.go) |

---

## 4. Business Rules & Validation (code se, exhaustive)

### Write-path (`UserSellonimLogContrller` / `SellonIMLogUpdateintoBD`)

1. **Auth model is two-layer, and unusual**: `/user/sellonimlog` is listed in
   `exception_service` [`requestValidation.go:26`](../../service-api-go-production/service-api-go-production/pkg/middleware/requestValidation.go),
   which is checked at [`requestValidation.go:723`](../../service-api-go-production/service-api-go-production/pkg/middleware/requestValidation.go)
   to **skip the outer JWT/AK-signature middleware entirely**. The only auth check on this
   endpoint is the controller's own inline `Gateway_v1` allowlist against the
   `VALIDATION_KEY` field: `GLADMIN`, `MAPI`, `M.INDIAMART.COM`, `SELLERMY`, `IMOB`,
   `ANDROID`, `IOS`. [`UserSellonimLogController.go:95-99`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSellonimLogController.go)
2. **Mandatory fields**: `GLID`, `MODID`, `VALIDATION_KEY` (checked via `CheckMandatoryParams`
   after the gateway check passes). [`UserSellonimLogController.go:88-105`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSellonimLogController.go)
3. **`BEST_TIME_TO_CALL` format gate**: if present, must parse as valid RFC3339
   (`ValidRFC3339Date`) — checked **before** the gateway/mandatory checks even run, so a
   malformed date short-circuits earlier than an auth failure would.
   [`UserSellonimLogController.go:91-93`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSellonimLogController.go)
4. **`BEST_TIME_TO_CALL` reformatting**: after passing `LengthAndTypeValidations_v1`, a
   valid RFC3339 value is re-parsed and reformatted to `YYYY-MM-DD HH24:MI:SS` before being
   stored in `inputParams` — this reformatted string is what actually reaches the DB (and,
   if `CUSTTYPE_ID==39`, what gets published to RabbitMQ too).
   [`UserSellonimLogController.go:111-116`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSellonimLogController.go)
5. **Insert-vs-update decided purely by `LOG_ID` presence** (not a pre-check SELECT) —
   `LOG_ID` empty/absent → `INSERT`, else → `UPDATE`.
   [`UserSellonimLogController.go:142-151`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSellonimLogController.go)
6. **Every write does a preliminary lookup** — `getDBValuesforGlid()` runs
   `SELECT FK_PARENT_GLUSR_ID, GLUSR_USR_APPROV FROM glusr_usr WHERE glusr_usr_id=$1` against
   **approvalpg** and merges the result into `inputParams`, so these two derived columns get
   written into `sellonim_log` too, even though the caller never sends them.
   [`UserSellonimlogModel.go:51,59,75-77,180-206`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSellonimlogModel.go)
7. **On UPDATE, `LOG_ID`, `GLID`, `MOBILE` are deleted from the dynamic-column-set** before
   building the `SET` clause — these three are treated as identity/lookup keys, not
   updatable fields, on UPDATE (on INSERT they *are* still written as normal columns).
   [`UserSellonimlogModel.go:80-84`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSellonimlogModel.go)
8. **`BEST_TIME_TO_CALL` gets special SQL treatment** in both INSERT and UPDATE — wrapped in
   `to_timestamp($n, 'yyyy-mm-dd hh24:mi:ss')` rather than bound as a plain parameter, since
   the column is a timestamp type but the app sends a formatted string.
   [`UserSellonimlogModel.go:88-96`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSellonimlogModel.go)
9. **UPDATE is scoped to `sellonim_log_id=$N AND fk_glusr_usr_id=$M`** — an update only
   succeeds if the `(LOG_ID, GLID)` pair matches an existing row; a `LOG_ID` that belongs to
   a different GLID silently affects 0 rows, which is treated as a failure
   (`"No Rows updated IN PG"`). [`UserSellonimlogModel.go:119,151,169-173`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSellonimlogModel.go)
10. **INSERT failure on zero rows returned**: even though the INSERT statement itself might
    not error, if the `RETURNING sellonim_log_id` scan loop finds zero rows, the function
    still returns a `"No record Inserted"` error. [`UserSellonimlogModel.go:139-147`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSellonimlogModel.go)
11. **On success (`output == "INSERT SUCCESS"` or `"UPDATE SUCCESS"`) AND
    `CUSTTYPE_ID=="39"` (string comparison)** → `SellonIMRabbitMQ()` fires, pushing the
    **full `inputParams` map** (including the merged `FK_PARENT_GLUSR_ID`/`GLUSR_USR_APPROV`
    from rule 6) to RabbitMQ, keyed by `GLID` for partition/routing.
    [`UserSellonimLogController.go:173-186`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSellonimLogController.go),
    [`UserSellonimlogModel.go:208-222`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSellonimlogModel.go)
12. **`rabbit_response` in the failure branch reads an out-of-scope variable** — in the
    `else` (non-success) response block, `response["rabbit_response"]` refers to the
    outer-scope `rabbitResponse` (declared at function top, never assigned in the failure
    path), not the inner `rabbitResponse :=` shadow declared inside the success `if` block —
    this is almost certainly always an empty string on failure, a minor code-quality
    observation rather than a business-rule.
    [`UserSellonimLogController.go:70,174,199-208`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSellonimLogController.go)

### Consumer-path (`dbActionUserSellOnImCalling`) — 5-gate eligibility chain

13. **Message shape expected**: top-level `SERVICENAME`, `MESSAGE`, `UNIQUE_LOGGING_ID` keys
    mandatory; `MESSAGE.GLID` mandatory. Missing any of these → `d.Ack` (message dropped, not
    retried) with a `FAILURE`-logged Kibana entry.
    [`USER_SELLONIM_CALLING.go:64-75`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SELLONIM_CALLING.go)
14. **Gate 1 — `getVerifcationLogData`**: `SELECT VERIFY_FLAG FROM
    iil_supp_verification_log WHERE FK_GLUSR_USR_ID=$1`. Fails (returns `false`) the moment
    ANY row has `VERIFY_FLAG` in `{-1, -2, -7}`; passes (`true`) only if all rows (or zero
    rows) avoid those three values. [`USER_SELLONIM_CALLING.go:166-191`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SELLONIM_CALLING.go)
15. **Gate 2 — `getFcpDetailsData`**: `SELECT PRIORITY_RANGE FROM active_fcp_details WHERE
    FK_GLUSR_USR_ID=$1`. Passes (`true`) on the **first** row where `PRIORITY_RANGE` is
    exactly `0.8` or `0.9` (NULLs are skipped, not counted as fail). If no row matches —
    including zero rows total — returns `false`. Any other priority value (0.1-0.7, 1.0,
    etc.) does not pass this gate. [`USER_SELLONIM_CALLING.go:193-220`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SELLONIM_CALLING.go)
16. **Gate 3 — `getDuplicateEmailData`**: `SELECT glusr_usr_id FROM glusr_usr WHERE
    GLUSR_USR_EMAIL_DUP = (SELECT GLUSR_USR_EMAIL_DUP FROM glusr_usr WHERE glusr_usr_id=$1)
    AND glusr_usr_id<>$2 AND (fcp_flag=1 OR GLUSR_USR_CUSTTYPE_ID IN (SELECT CUSTTYPE_ID FROM
    custtype WHERE paid=-1))`. **Inverted-boolean naming**: if this query returns **any**
    row (i.e. a duplicate-email account that is fcp-flagged or paid-type exists), the
    function returns `false` (disqualify); if it returns **zero** rows, it returns `true`
    (pass). A function literally named "getDuplicate..." returning `true` for "no duplicate
    found" is a readability trap for anyone reading call-sites without checking the body.
    [`USER_SELLONIM_CALLING.go:222-244`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SELLONIM_CALLING.go)
17. **Gate 4 — `getDuplicateGSTData`**: `SELECT FK_GLUSR_USR_ID FROM
    GLUSR_USR_COMP_REGISTRATIONS WHERE gst IN (SELECT gst FROM
    GLUSR_USR_COMP_REGISTRATIONS WHERE FK_GLUSR_USR_ID=$1) AND FK_GLUSR_USR_ID<>$2`. Same
    inverted pattern as Gate 3 — passes (`true`) only if no *other* `FK_GLUSR_USR_ID` shares
    this user's GST value(s). [`USER_SELLONIM_CALLING.go:246-268`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SELLONIM_CALLING.go)
18. **Gate 5 — `getDuplicateGlid`**: `SELECT IIL_SUPP_VERIF_GLUSR_ID FROM
    iil_supp_verification WHERE IIL_SUPP_VERIF_ALLOCAT_DATE > CURRENT_DATE-30 AND
    IIL_SUPP_VERIF_GLUSR_ID=$1`. Same inverted pattern — passes only if this GLID has **not**
    already been allocated to the verification/calling queue in the last 30 days.
    [`USER_SELLONIM_CALLING.go:270-292`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SELLONIM_CALLING.go)
19. **All 5 gates are chained sequentially with nested short-circuit `if err == nil &&
    result...` blocks** — each gate only runs if the previous one both errored-free AND
    passed. A DB-error at any gate sets `finalError=true` and stops the chain (`d.Nack(false,
    false)` — no requeue-to-front, standard Nack with `requeue=false`... **note**: `Nack`
    with `requeue=false` typically means the message is dropped/dead-lettered, not retried,
    unless the queue has a DLQ configured — see Open Questions). A business-rule failure
    (a gate returns `false`, no error) leaves `finalStatus=false` and the message is `d.Ack`'d
    — "processed but not qualified," not a failure.
    [`USER_SELLONIM_CALLING.go:89-159`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SELLONIM_CALLING.go)
20. **Final insert into `iil_supp_verification`** (`insertDetail_supp_verification`)
    hardcodes several constant-fields: `Iil_supp_verif_data_source='QGFCP ACTIVE'`,
    `IIL_SUPP_VERIF_CUSTTYPE_WT=1399`, `iil_supp_verif_priority=1`,
    `iil_supp_verif_type='VGFCP'`. `BEST_TIME_TO_CALL` is `NULL` if not supplied in the
    message, else parsed via `TO_DATE($2, 'YYYY-MM-DD HH24:MI:SS')` (matching the format the
    write-side reformatted it to in rule 4). [`USER_SELLONIM_CALLING.go:295-308`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SELLONIM_CALLING.go)
21. **`insertDetail_supp_verification`'s return-value plumbing is unusual**:
    `utils.ExecuteDMLCommand` returns `(string, ...)` where the string appears to double as
    an error-message; the function then does `return err, errors.New(err)` — i.e. it wraps
    that same string into both the `string` and `error` return positions. Downstream
    (`dbActionUserSellOnImCalling`) treats a non-nil `err` as failure regardless of the string
    content, so this only matters if `ExecuteDMLCommand` ever returns an **empty** string on
    success — in which case `errors.New("")` is technically a non-nil error and would be
    misread as a failure. **[INFERRED — confirm `ExecuteDMLCommand`'s success-path return
    value with team before assuming this path is bug-free.]**

---

## 5. RabbitMQ

| Queue / `SERVICENAME` | Publisher | Consumer | Purpose |
|---|---|---|---|
| `USER_SELLONIM_LOG` (SERVICENAME) → resolves to queue `USER_SELLONIM_CALLING` via `serviceToQueueMap` — [`rabbitmq.go:30`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go) | `UserSellonimlogModel.go` (`SellonIMRabbitMQ`) — **only when `CUSTTYPE_ID=="39"` and the DB write succeeded** | [`USER_SELLONIM_CALLING.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SELLONIM_CALLING.go) (`dbActionUserSellOnImCalling`, registered as consumer for queue `USER_SELLONIM_CALLING` in both [`IntializeMsgBroker.go:62`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go) and [`router.go:219`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Router/router.go)) | 5-gate calling-eligibility pipeline, final write to `iil_supp_verification` |

This is the **only** queue in scope for the SellOnIM Log feature. No other publish/consume
call referencing `sellonim`/`SELLONIM` was found in any of the three repos searched
(`users-api-go-production`, `service-api-go-production`, `user-temp-consumers-production`).

---

## 6. Kafka

**Koi Kafka usage nahi mila.** Grepping `sellonim`/`SELLONIM` across all three repos surfaces
only the RabbitMQ path (section 5) and the write/consumer files already listed in the File
Map — no Kafka topic, producer, or `InitializeKafka`-style call references this feature.

---

## 7. Redis

**Koi Redis usage nahi mila.** None of `UserSellonimLogController.go`,
`UserSellonimlogModel.go`, or `USER_SELLONIM_CALLING.go` reference any Redis
client/`RedisGet`/`RedisSet`-style call. Both the write-path (DB read + DB write) and the
5-gate consumer chain hit Postgres directly on every call, with no caching layer.

---

## 8. End-to-End Technical Flow

```
New supplier / GLADMIN / MAPI / M.INDIAMART.COM / SELLERMY / IMOB / Android / iOS
    │
    ▼
[API — write]  POST /user/sellonimlog   (serviceName=USER_SELLONIM_LOG in body)
                {GLID, MODID, LOG_ID?, CUSTTYPE_ID, JOURNEY_COMPLETED, IS_USR_NAME,
                 IS_COMPANYNAME, IS_EMAIL, IS_ADDRESS, IS_GST, PROD_CNT, MOBILE?,
                 FCP_STATUS?, BEST_TIME_TO_CALL?, VALIDATION_KEY}
    │  UserSellonimLogController.go
    │  - route is exempt from outer JWT/AK middleware (exception_service list)
    │  - RFC3339 check on BEST_TIME_TO_CALL (if present) — BEFORE gateway check
    │  - Gateway_v1 allowlist check on VALIDATION_KEY (the only auth on this route)
    │  - mandatory-field check (GLID, MODID, VALIDATION_KEY)
    │  - LengthAndTypeValidations_v1 against MapSellonIM
    │  - BEST_TIME_TO_CALL reformatted RFC3339 → 'YYYY-MM-DD HH24:MI:SS'
    ▼
getDBValuesforGlid()  — SELECT FK_PARENT_GLUSR_ID, GLUSR_USR_APPROV FROM glusr_usr
                         WHERE glusr_usr_id=$1 (approvalpg), merge into inputParams
    ▼
SellonIMLogUpdateintoBD()
    ├─ LOG_ID empty  → INSERT INTO sellonim_log (...) RETURNING sellonim_log_id
    └─ LOG_ID given  → UPDATE sellonim_log SET ... WHERE sellonim_log_id=$N AND
                        fk_glusr_usr_id=$M   (LOG_ID/GLID/MOBILE excluded from SET)
    ▼
[DB — approvalpg]  sellonim_log
    │
    │  if output == "INSERT SUCCESS"/"UPDATE SUCCESS"  AND  CUSTTYPE_ID=="39":
    ▼
[RabbitMQ]  SellonIMRabbitMQ() → SERVICENAME=USER_SELLONIM_LOG → queue USER_SELLONIM_CALLING
            (payload = full inputParams incl. merged FK_PARENT_GLUSR_ID/GLUSR_USR_APPROV)
    ▼
[CONSUME]  dbActionUserSellOnImCalling()
    │  validates SERVICENAME/MESSAGE/UNIQUE_LOGGING_ID/GLID present, else Ack+drop
    │
    │  Gate 1: getVerifcationLogData()   — no VERIFY_FLAG in {-1,-2,-7}?          (meshPg)
    │  Gate 2: getFcpDetailsData()       — PRIORITY_RANGE in {0.8, 0.9}?          (meshPg)
    │  Gate 3: getDuplicateEmailData()   — no risky duplicate-email account?      (meshPg)
    │  Gate 4: getDuplicateGSTData()     — no duplicate GST?                      (meshPg)
    │  Gate 5: getDuplicateGlid()        — not verification-allocated in last 30d?(meshPg)
    │  (strictly sequential, short-circuit on first failure or first DB error)
    │
    ├─ any gate DB-errors  → d.Nack(false, false)  (dropped/DLQ, not business-retried)
    ├─ any gate fails (no error) → d.Ack(false)  ("processed, not eligible")
    └─ all 5 pass → insertDetail_supp_verification()
                    INSERT INTO iil_supp_verification (..., BEST_TIME_TO_CALL)
                    → d.Ack(false)
    ▼
[DB — meshPg]  iil_supp_verification  (final "calling queue" for internal sales/calling team)
```

---

## 9. Flow-wise DB & Table Usage

### Write-path — `POST /user/sellonimlog`

| # | DB (physical) | Table | Operation | Kya nikala/likha jaata hai, aur kyun |
|---|---|---|---|---|
| 1 | approvalpg | `glusr_usr` | SELECT (`getDBValuesforGlid`) | `FK_PARENT_GLUSR_ID`, `GLUSR_USR_APPROV` fetch karta hai iss GLID ke liye, taaki yeh derived-columns bhi `sellonim_log` mein likhe ja sakein — caller inhe khud nahi bhejta |
| 2 | approvalpg | `sellonim_log` | INSERT (`RETURNING sellonim_log_id`) **or** UPDATE (`WHERE sellonim_log_id=$N AND fk_glusr_usr_id=$M`) | Journey-progress record insert ya update karta hai — flag purely `LOG_ID` presence se decide hota hai |

**Total DB round-trips for one write: 2**, both on the same `approvalpg` connection
(sequential, not parallel).

### Consumer — 5-gate eligibility chain (`USER_SELLONIM_CALLING`, only fires for `CUSTTYPE_ID==39` writes)

| # | DB (physical) | Table | Operation | Kya nikala/likha jaata hai, aur kyun |
|---|---|---|---|---|
| 1 | meshPg | `iil_supp_verification_log` | SELECT | Gate 1 — is GLID ka koi disqualifying `VERIFY_FLAG` (`-1`/`-2`/`-7`) history hai kya, check karta hai |
| 2 | meshPg | `active_fcp_details` | SELECT | Gate 2 — is GLID ka `PRIORITY_RANGE` `0.8`/`0.9` range mein hai kya, check karta hai |
| 3 | meshPg | `glusr_usr` self-join + `custtype` subquery | SELECT | Gate 3 — same `GLUSR_USR_EMAIL_DUP` share karne wala koi doosra `fcp_flag=1`/paid-custtype account hai kya, check karta hai (duplicate-email risk) |
| 4 | meshPg | `GLUSR_USR_COMP_REGISTRATIONS` self-join | SELECT | Gate 4 — same GST share karne wala koi doosra `FK_GLUSR_USR_ID` hai kya, check karta hai |
| 5 | meshPg | `iil_supp_verification` | SELECT | Gate 5 — is GLID ka koi verification-allocation last 30 din mein already hua hai kya, check karta hai |
| 6 | meshPg | `iil_supp_verification` | INSERT | Sab 5 gates pass hone par — final "calling-queue" entry likhta hai, hardcoded `'QGFCP ACTIVE'`/`1399`/`1`/`'VGFCP'` constants ke saath, plus `BEST_TIME_TO_CALL` (ya `NULL`) |

**Total DB round-trips per message: best-case 1 (Gate 1 fails), worst-case 6 (all 5 gates
pass + final insert)** — inherently a gating-pipeline, each gate only runs if all prior
gates passed. All 6 possible round-trips hit the **same physical DB** (meshPg), unlike GST's
multi-database fan-out.

---

## 10. Optimization Scope — DB Response-Time Contribution

### Medium impact

1. **Write-path does 2 sequential DB round-trips** (`getDBValuesforGlid` SELECT, then the
   INSERT/UPDATE) on the same connection — could be combined if `FK_PARENT_GLUSR_ID`/
   `GLUSR_USR_APPROV` were resolved via a subquery inside the main INSERT/UPDATE statement
   instead of a separate round-trip. Same pattern as flagged in GST's Flow A (mainPg
   `Get_gst_pan_cin_tan` + separate write).
   [`UserSellonimlogModel.go:26-77`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSellonimlogModel.go)
2. **Consumer's 5-gate chain is up to 5 sequential DB queries per message (plus a 6th for the
   final insert)**, each only running if the prior passed. Best-case 1 query, worst-case 6.
   This is inherently a gating-pipeline, so full parallelization isn't straightforward
   (gates are meant to short-circuit on the cheapest/most-likely-to-disqualify check first)
   — but **the current gate order (verification-log → FCP-priority → email-dup → GST-dup →
   GLID-recency) is not obviously ordered by selectivity or query-cost**. If gate-failure
   rates were measured, reordering to fail fast on the cheapest/most-frequently-failing
   check could reduce average query-count. [`USER_SELLONIM_CALLING.go:89-147`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SELLONIM_CALLING.go)
3. **Gates 3 and 4 use non-indexed-looking subquery patterns** (`WHERE gst IN (SELECT gst
   FROM ... WHERE FK_GLUSR_USR_ID=$1)`, `WHERE GLUSR_USR_EMAIL_DUP = (SELECT ... WHERE
   glusr_usr_id=$1)`) — a self-referencing subquery per row rather than a pre-fetched
   value bound directly. **[INFERRED]**: whether this is actually a performance issue
   depends on the live query plan/indexes on `gst`/`GLUSR_USR_EMAIL_DUP`, which wasn't
   checked in this pass — flagging as a candidate for `EXPLAIN ANALYZE` review, not a
   confirmed problem.

### Low impact

4. **Hardcoded constant-fields in the final insert** (`'QGFCP ACTIVE'`, `1399`, `'VGFCP'`,
   `1`) — not a performance concern, but worth flagging as config-in-code (would need a
   code-deploy to change these values for this specific lead-source).
   [`USER_SELLONIM_CALLING.go:295-308`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SELLONIM_CALLING.go)
5. **No caching anywhere in this feature** (section 7) — given the feature is write-heavy
   with no read endpoint at all, this is low-priority; caching would only help the consumer's
   repeated `glusr_usr`/`GLUSR_USR_COMP_REGISTRATIONS` reads if the same GLID gets
   re-evaluated frequently, which the 30-day Gate-5 window suggests is uncommon by design.

---

## 11. Cron Inventory

**No cron job found for this feature.** Searched all `.go` files under any `cron`-named
directory/path across the three repos for `sellonim`/`SELLONIM` — no matches. Unlike GST
(which has a nightly BigQuery-driven `gst_tact_veri_cron.go`), SellOnIM Log's journey-tracking
and calling-eligibility pipeline is purely event-driven (API write → conditional RabbitMQ
publish → consumer), with no periodic re-evaluation mechanism.

---

## 12. Edge Cases & Gotchas (technical POV)

1. **Gate-function naming is misleading** — `getDuplicateEmailData`/`getDuplicateGSTData`/
   `getDuplicateGlid` return `true` when NO duplicate is found (i.e., "pass"), not when a
   duplicate exists — easy to misread during debugging or when adding a 6th gate later.
2. **A gate-failure (business-rule "not eligible") is `d.Ack`'d, not `d.Nack`'d** — only a
   genuine DB/query-error triggers a `Nack`. This is correct behavior (don't retry a
   legitimate ineligibility) but means there's **no automatic re-evaluation** if the
   supplier's disqualifying condition later resolves (e.g., duplicate-GST cleared, or
   `VERIFY_FLAG` history changes) — they'd need a fresh `USER_SELLONIM_LOG` write with
   `CUSTTYPE_ID==39` to re-trigger the whole pipeline. No cron exists to periodically retry
   (section 11), unlike GST's daily reconciliation job.
3. **`d.Nack(false, false)` on DB-errors means dropped, not necessarily retried** — the
   second `false` is `requeue=false`; whether this message is truly lost or lands in a
   dead-letter queue depends on the RabbitMQ queue/exchange configuration, which was not
   inspected in this pass. **[INFERRED — confirm DLQ setup with infra/platform team]**.
4. **`CUSTTYPE_ID=="39"` is a hardcoded magic-string gate** for the entire downstream
   pipeline — no comment/constant explaining what customer-type 39 represents, and the
   comparison is against a string (`"39"`), not a parsed int.
   [`UserSellonimLogController.go:175`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSellonimLogController.go)
5. **Route bypasses the standard JWT/AK middleware entirely** (section 4, rule 1) —
   security posture rests solely on the `VALIDATION_KEY` allowlist string-match inside the
   controller, which is a materially different (and weaker) auth mechanism than most other
   endpoints in this codebase use. Worth a security-review flag if not already reviewed.
6. **Route registered three times** in `router.go` (lines 156, 284, 324) — not conclusively
   explained as duplicate-vs-intentional in this pass (section 2).
7. **UPDATE silently no-ops on GLID/LOG_ID mismatch** — if a caller sends a `LOG_ID` that
   belongs to a different `GLID` than the one in the request, the UPDATE affects 0 rows and
   is reported as `"Insert/Update Failed"` with no distinguishing detail from other DB
   failures — from the API response alone, a caller can't tell "wrong LOG_ID/GLID pairing"
   from "DB connectivity issue."
8. **Koi read-endpoint nahi mila** — journey-progress ko kaise view/monitor kiya jaata hai
   (dashboard, direct-DB-query, BI tool) confirm nahi hua is pass mein, in any of the three
   repos searched.
9. **`insertDetail_supp_verification`'s error-wrapping is unusual** (section 4, rule 21) —
   `return err, errors.New(err)` pattern means an empty success-string from
   `ExecuteDMLCommand` would still produce a non-nil `error` value; worth confirming this
   function's actual success-path return contract before relying on it as a "was it really
   inserted" signal.

---

## 13. Open Questions

1. `CUSTTYPE_ID=="39"` — what customer-type does this represent, business-wise, and why is
   it the sole trigger for the calling-eligibility pipeline?
2. Consumer's disqualification is permanent per-message — is there truly no periodic
   re-evaluation/retry mechanism elsewhere for suppliers who become eligible later (no cron
   found for this feature, section 11), or does some other system re-trigger
   `USER_SELLONIM_LOG` writes on a schedule?
3. Hardcoded constants in `insertDetail_supp_verification` (`'QGFCP ACTIVE'`, `1399`,
   `'VGFCP'`) — are these specific to this one lead-source, or shared config that should be
   centralized/looked-up instead?
4. Is there any read/reporting surface for `sellonim_log` outside these three repos (BI
   dashboard, direct SQL access for the calling team, etc.)?
5. `d.Nack(false, false)` on gate DB-errors — does the `USER_SELLONIM_CALLING` queue have a
   dead-letter-exchange configured, or are these messages genuinely dropped on DB failure?
6. Triple route registration in `router.go` (section 2, section 12.6) — intentional
   (env/version-scoped groups) or leftover duplication?
7. Why does this specific route (`/user/sellonimlog`) bypass the standard JWT/AK middleware
   via `exception_service`, when comparable write endpoints (e.g. GST's `/details`) go
   through it?
8. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai; koi live pgAdmin cross-check nahi hua.

---

## See also

- [`SellOnIM_Log_Business_Doc.md`](./SellOnIM_Log_Business_Doc.md) — product perspective
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — duplicate-GST check
  (Gate 4) reads from the same `GLUSR_USR_COMP_REGISTRATIONS` table documented there

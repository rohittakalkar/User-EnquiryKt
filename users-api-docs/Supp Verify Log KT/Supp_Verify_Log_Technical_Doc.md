# Supplier Verification Log — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`Supp_Verify_Log_Business_Doc.md`](./Supp_Verify_Log_Business_Doc.md)
dekho. Dono docs same feature cover karte hain, bas alag audience ke liye.

**Scope note**: Related to [`../SellOnIM Log KT/`](../SellOnIM%20Log%20KT/) — SellOnIM's
`USER_SELLONIM_CALLING` consumer populates `iil_supp_verification` (the calling-**queue**);
this KT covers `IIL_SUPP_VERIFICATION_LOG` (the verification-**outcome**-log) — a different
table. No FK relationship between the two tables was found in code; they are sequential in
business-flow terms (queue → outcome-log) but not joined anywhere seen in this review.

**Repos**: `service-api-go-production` (write API + a standalone nightly cron),
`users-api-go-production` (read API). **Koi RabbitMQ/Kafka/Redis consumer nahi mila** for
this feature anywhere in `user-temp-consumers-production` either — purely synchronous CRUD
plus an independent nightly batch-cron that talks straight to Postgres.

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file:line diya
gaya hai). Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — confirm with team]** likh diya hai, guess nahi kiya.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write — controller | write | [`UserSuppVerifyLogController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSuppVerifyLogController.go) |
| Write — model | write | [`UserSuppVerifyLogModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSuppVerifyLogModel.go) (`SuppVerifyLog`, `callHistoryPackageMeshpg`) |
| Write — mandatory-params validation | write | `MandatoryParamsSuppVerifyLog` — [`UserUtilsMandatory.go:1421-1453`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go) |
| Write — type/length validation | write | `ValidationSuppVerifyLog` + `UserSuppVerifyLogMap` — [`UsersValidationMaps.go:398-422,2226-2239`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |
| Read — controller | read | [`SuppVerifyLogController.go`](../../users-api-go-production/internal/controllers/UsersControllers/SuppVerifyLogController.go) |
| Read — model | read | [`UsersSuppVerifyLogModel.go`](../../users-api-go-production/internal/models/users/UsersSuppVerifyLogModel.go) (`SuppVerifyLogModel`) |
| Automated batch-populator cron | write repo, `crons/` | [`Supp_verify_log_cron.go`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) (`populateSuppVerificationData` + 9 `process*` sub-processors, `main()`) |
| Cron shell wrapper | write repo, `crons/` | [`Supp_verify_log_cron.sh`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.sh) — `go run Supp_verify_log_cron.go ${1}` |
| Router — write | write | [`router.go:158,326`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) |
| Router — read | read | [`routerUsers.go:402-403,540-541,739-740`](../internal/api/users_router/routerUsers.go) — route registered 3 times (likely 3 env/version blocks) |

**Related but out-of-scope**: `USER_SELLONIM_CALLING` consumer —
[`USER_SELLONIM_CALLING.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SELLONIM_CALLING.go)
— inserts into `iil_supp_verification` (the queue table), not `IIL_SUPP_VERIFICATION_LOG`.
Grepped for `IIL_SUPP_VERIFICATION_LOG` inside that file — **no match**, confirming the two
tables are not directly linked in code.

---

## 2. Routes

| Method | Path | serviceName | Repo | Controller |
|---|---|---|---|---|
| POST | `/supp_verify_log` | `SUPP_VERIFY_LOG` | write | `UserControllers.SuppVerifyLog` — [`router.go:158,326`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) |
| GET / POST | `supp_verify_log/*params` | `SUPPVERIFYLOG` | read | `UsersControllers.SuppVerifyLogController` — [`routerUsers.go:402-403`](../internal/api/users_router/routerUsers.go) |
| — | (standalone binary, not HTTP) | — | cron | `main()` in `Supp_verify_log_cron.go`, invoked via `Supp_verify_log_cron.sh` |

---

## 3. Data Model — Tables

> **Verification note**: yeh sab table/column names Go code ke andar embedded SQL strings se
> liye gaye hain (har row ke saamne file:line diya hai). Live DB schema se pgAdmin pe
> cross-verify **nahi** kiya gaya hai — kisi migration/integration se pehle woh zaroor karo.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `IIL_SUPP_VERIFICATION_LOG` | write/cron: `meshpg` (Postgres) / read: `mesh_pg_user` | **Verification-call/auto-rule outcome log** — one row per verification event (manual call ya automated cron-rule) | `VERIFY_LOG_ID` (PK, `RETURNING` on insert — [`UserSuppVerifyLogModel.go:75`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSuppVerifyLogModel.go)), `FK_GLUSR_USR_ID`, `VERIFY_FLAG`, `DISPOSITION_TYPE`, `DISPOSITION_TYPE_ACTIVITY`, `CALL_COUNTER`, `PROCESS_BY`, `PROCESS_DATE`, `PROCESS_BY_EMPID`, `VOICE_LOG_URL`, `EMAIL_OPEN_STATUS`, `IIL_SUPP_FIRST_VERIFY_DATE`, `IIL_SUPP_FIRST_VERIFY_BY`, `PRD_LINE_VERIFY`, `OTHER_ATTACHMENTS` (JSON-array-as-string, max 5), `SUPP_VERF_UPDATE_DATE`, `SUPP_VERF_UPDATED_BY_ID`/`_NAME`/`_AGENCY`/`_SCREEN`/`_URL`/`_IP`/`_IP_COUNTRY`, `CALL_RESCHEDULE_TIME` — [`UserSuppVerifyLogModel.go:75,252-272`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSuppVerifyLogModel.go), read columns confirmed at [`UsersSuppVerifyLogModel.go:34`](../internal/models/users/UsersSuppVerifyLogModel.go) |
| `iil_supp_verification` | `meshpg`/`mesh_pg_user` | **Different table** — the upstream calling-**queue** (SellOnIM's domain, not this feature's write path), but the cron's `cntBeforeReceipt`/`cntAfterReceipt`/`processReceiptUsers`/`processCountryRejection`/`finalizeAndSendMail` sub-steps all read/delete from it too — [`Supp_verify_log_cron.go:330-425`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) | `iil_supp_verif_glusr_id`, `iil_supp_verif_allocat_date`, `iil_supp_verif_type` (`'VGFCP'` seen), `iil_supp_verif_data_source` |
| `GLUSR_USR_CASS_UPDATE_MAIN` | `mainpg` | Main-DB source table the cron reads from to find recently-disabled suppliers | `FK_GLUSR_USR_ID`, `GLUSR_USR_CASS_UPDATEDUSING`, `GL_ATTRIBUTE_UPDATE_DATE` — [`Supp_verify_log_cron.go:108`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) |
| `cust_to_serv` / `service` / `custtype` / `glusr_usr` / `pns_defaulter` | `mainpg` | Joined by the PNS-defaulter sub-processor to find suppliers whose catalog service recently ended | see query at [`Supp_verify_log_cron.go:180-204`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) |
| `ar_receipts` / `ar_receipt_types` | `mainpg` | Joined by the receipt-users sub-processor to find suppliers with a recent actual payment receipt | [`Supp_verify_log_cron.go:341-343`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) |
| `glusr_usr` (lowercase alias, meshpg) | `meshpg` | Used by `processFreeVerified` (dormant, see section 12) to filter by `fcp_flag`/`glusr_usr_custtype_id` | [`Supp_verify_log_cron.go:315-317`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) |

---

## 4. Decoding `VERIFY_FLAG` — Magic Values

No enum/mapping was found anywhere in code for `VERIFY_FLAG`. The values below are
reconstructed purely from the literal comparisons/assignments seen — treat all business
meanings as **[INFERRED — confirm with team]** unless noted otherwise.

| Value | Where it's set/read | Meaning (inferred) |
|---|---|---|
| `-1` | Set by `processCassUpdateMain` ([`Supp_verify_log_cron.go:137`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go)) with `DISPOSITION_TYPE='Verified'`, `DISPOSITION_TYPE_ACTIVITY='Paid Catalog To FCP'`. Also set by `processPNSDefaulterRecord` when `paidStatus==true` ([`:278`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go)) with same disposition strings. | "Verified" outcome, specifically tied to a paid-catalog-to-FCP disposition |
| `-5` | Set by `processPNSDefaulterRecord` when `paidStatus==false` ([`:282`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go)), `DISPOSITION_TYPE='Verification Rejected'`, `DISPOSITION_TYPE_ACTIVITY='Catalog PNS Defaulter'` | Rejected outcome — supplier is a PNS defaulter |
| `-7` | Set by `processFreeVerified` ([`:315`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go), **currently dormant/uncalled**, see section 12) — downgrades rows currently at `-1`/`-2` that are >330 days old, `fcp_flag=1`, custtype in `(14,32,34,35)` | Free-tier-verified downgrade of a stale paid-verification |
| `-2` | Never set/read anywhere in this feature's own code. Referenced only as a "disqualifying" value in **SellOnIM's** consumer per the existing SellOnIM Log KT doc. | **[INFERRED — confirm with team]**, likely a shared convention across both features but not produced by this cron |
| `-3` | Appears only as an **exclusion condition** in `processPNSDefaulterRecord`'s UPDATE — `... where verify_log_id = (...) and verify_flag <> -3` ([`Supp_verify_log_cron.go:293`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go)) — the cron will not overwrite a row currently flagged `-3` | **[INFERRED — confirm with team]** — likely represents a manually-locked/protected state that the automated PNS rule is explicitly forbidden from touching, but no code sets `-3` anywhere in this file |
| `-20` | Never set by the cron. Appears only in the write-API's mandatory-params check: `ACTION=INSERT` + `VERIFY_FLAG="-20"` forces `REQUEST_URL` to be mandatory — [`UserUtilsMandatory.go:1448-1449`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go) | **[INFERRED — confirm with team]** — some specific manual-insert scenario that requires an audit URL, business meaning unclear from code alone |

---

## 5. Business Rules & Validation (code se, exhaustive)

### Write-path (`SuppVerifyLog` controller / model)

1. **Mandatory-params gate runs before everything else**: `GLUSR_ID`, `VALIDATION_KEY`, `IP`,
   `IP_COUNTRY`, `ACTION`, `PROCESS_BY`, `PROCESS_BY_EMPID` are always required; `ACTION` must
   be one of `INSERT`/`UPDATE`/`DELETE`; per-action extra requirements (`INSERT` additionally
   needs `VERIFY_FLAG`; `UPDATE` needs `VERIFY_LOG_ID`; `DELETE` needs `VERIFY_LOG_ID`).
   [`UserUtilsMandatory.go:1421-1453`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
2. **`OLD_VERIFY_FLAG`, if supplied, must be numeric** — otherwise rejected with "OLD_VERIFY_FLAG
   should be numeric". Note: this field is validated but **never actually used** anywhere else
   in the write model (`UserSuppVerifyLogModel.go`) — dead input, no downstream read of it found.
   [`UserUtilsMandatory.go:1446-1447`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
3. **`ACTION=INSERT` + `VERIFY_FLAG="-20"` forces `REQUEST_URL` to be mandatory**, a special-case
   requirement no other `VERIFY_FLAG` value triggers.
   [`UserUtilsMandatory.go:1448-1449`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
4. **Gateway allowlist**: `GLADMIN` only (plus a stray empty-string entry in the array —
   `[]string{"GLADMIN", ""}` — likely a leftover/no-op, functionally harmless).
   [`UserSuppVerifyLogController.go:75`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSuppVerifyLogController.go)
5. **`MULTIPLE_ATTACHMENTS` capped at 5 comma-separated values** (each trimmed of whitespace),
   converted to a JSON array-string (`OTHER_ATTACHMENTS` column) before storage; empty input
   is stored as empty string, not `null`.
   [`UserSuppVerifyLogController.go:79-95`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSuppVerifyLogController.go)
6. **Field-level type/length validation** runs via `ValidationSuppVerifyLog` →
   `LengthAndTypeValidations_v2` against `UserSuppVerifyLogMap` (22 named fields, with explicit
   type=`number`/`string` and max-length per field) — runs on every field **except**
   `VALIDATION_KEY`, `IP`, `IP_COUNTRY`, `ACTION`, `OLD_VERIFY_FLAG`, `REQUEST_URL`, `unique_id`
   (those are stripped before this check).
   [`UsersValidationMaps.go:398-422,2226-2239`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go)
7. **`ACTION=INSERT`**: full 21-column insert (see section 3), `CALL_COUNTER` hardcoded to `1`
   on first insert, `PROCESS_DATE` and `SUPP_VERF_UPDATE_DATE` both hardcoded to
   `CURRENT_TIMESTAMP` (not client-supplied), `VERIFY_LOG_ID` returned via `RETURNING`.
   [`UserSuppVerifyLogModel.go:75-248`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSuppVerifyLogModel.go)
8. **`ACTION=UPDATE`**: dynamically-built via a `updateMap` whitelist (20 input-key →
   DB-column pairs) — only keys present in the request get included in the `SET` clause.
   `PROCESS_DATE`/`SUPP_VERF_UPDATE_DATE` are **always** forced to `CURRENT_TIMESTAMP`
   regardless of whether the client supplied a value for those specific keys (client-supplied
   values for those two keys are silently discarded, `CURRENT_TIMESTAMP` wins); if
   `PROCESS_DATE` key wasn't present at all in the request, it's appended as an extra `SET`
   clause. Matched on `VERIFY_LOG_ID` only (not scoped by `FK_GLUSR_USR_ID` — an UPDATE can
   touch any row given its ID, no ownership check).
   [`UserSuppVerifyLogModel.go:250-299`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSuppVerifyLogModel.go)
9. **`ACTION=DELETE`**: matched on `(FK_GLUSR_USR_ID, VERIFY_LOG_ID)` (both must match, unlike
   UPDATE), and **always** followed by a call to `callHistoryPackageMeshpg()` — invokes stored
   procedure `PCK_GL_HISTORY_SP_ADD_CUSTOM_HISTORY(...)` with a hardcoded comment
   `"Gladmin (Data Validation Process)"`, transaction-type `'D'`, and tag
   `'IIL_SUPP_VERIFICATION_LOG-VERIFY_FLAG'` — logs the deletion into a shared history/audit
   mechanism regardless of whether the DELETE itself actually affected any row (the history
   call fires unconditionally after the DELETE query executes, no rows-affected check gates it).
   [`UserSuppVerifyLogModel.go:301-350,359-432`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSuppVerifyLogModel.go)
10. **A quirky response-message override**: if the model returns `"INSERT SUCCESS IN MESH PG"`,
    the controller overwrites the outward-facing `MESSAGE` to say `"UPDATE SUCCESS IN MESH PG"`
    instead — response-text doesn't distinguish insert from update for the caller. The success
    check itself (`STATUS`/`CODE`) is based on the **original** (pre-override) `output` string
    matching one of the three success strings, so this only affects the human-readable message,
    not the status code.
    [`UserSuppVerifyLogController.go:137-161`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSuppVerifyLogController.go)
11. **1-second query timeout** on both the main INSERT/UPDATE/DELETE and the history-package
    call — [`UserSuppVerifyLogModel.go:221,325,420`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSuppVerifyLogModel.go)

### Read-path (`SuppVerifyLogModel`)

12. **Returns ALL log-rows for a given `FK_GLUSR_USR_ID`** — no single-latest-row filtering
    (unlike Disposition KT's default-latest-only behavior); full history always returned, no
    `LIMIT`/pagination/`ORDER BY` in the query at all.
    [`UsersSuppVerifyLogModel.go:34`](../internal/models/users/UsersSuppVerifyLogModel.go)
13. **Explicit 11-column `SELECT`** (not `SELECT *`): `VERIFY_LOG_ID`, `CALL_COUNTER`,
    `VERIFY_FLAG`, `IIL_SUPP_FIRST_VERIFY_DATE`, `DISPOSITION_TYPE_ACTIVITY`, `PROCESS_BY`,
    `PROCESS_DATE`, `PROCESS_BY_EMPID`, `CALL_RESCHEDULE_TIME`, `VOICE_LOG_URL`,
    `OTHER_ATTACHMENTS` — notably **`DISPOSITION_TYPE` itself (not just `_ACTIVITY`) is not
    returned by the read API**, only the write API stores/sees it.
    [`UsersSuppVerifyLogModel.go:34`](../internal/models/users/UsersSuppVerifyLogModel.go)
14. **100ms query timeout** on the read — noticeably tighter than the write path's 1s.
    [`UsersSuppVerifyLogModel.go:42`](../internal/models/users/UsersSuppVerifyLogModel.go)
15. **Date-fields (`PROCESS_DATE`, `IIL_SUPP_FIRST_VERIFY_DATE`) reformatted to `DD-MON-YY`
    uppercase**, `OTHER_ATTACHMENTS` JSON-unmarshaled back into an array for the response; if
    JSON-unmarshal fails, the field is silently set to `nil` rather than erroring the request.
    [`UsersSuppVerifyLogModel.go:60-80`](../internal/models/users/UsersSuppVerifyLogModel.go)
16. **`glusrid` and `modid` (gateway/permission ID) both mandatory** on the read call, checked
    via `utils.CheckValidity` before the model is even invoked — `token` param also validated
    though not obviously used downstream in the model itself.
    [`SuppVerifyLogController.go:43-59`](../internal/controllers/UsersControllers/SuppVerifyLogController.go)

### Automated batch-cron (`Supp_verify_log_cron.go`)

17. See section 12 (Cron Inventory) for the full, per-sub-processor breakdown — this is the
    richest part of the feature and is broken out separately below rather than repeated here.

---

## 6. RabbitMQ

**Koi RabbitMQ usage nahi mila** for this feature — neither in the write controller/model, the
read controller/model, nor the cron. Grep for `PushToQueue`/`PubAPI`/`Requeue` inside
`UserSuppVerifyLogController.go`, `UserSuppVerifyLogModel.go`, and `Supp_verify_log_cron.go`
returned no matches. This is a purely synchronous-CRUD-plus-batch-cron feature.

## 7. Kafka

**Koi Kafka usage nahi mila** — same grep, same result, no matches anywhere in this feature's
files.

## 8. Redis

**Koi Redis usage nahi mila** — no caching layer anywhere in this feature. The read endpoint
hits `mesh_pg_user` Postgres directly on every request (section 5.12-14); the write endpoint
writes straight to `meshpg` Postgres; the cron talks straight to `meshpg` + `mainpg`.

---

## 9. End-to-End Technical Flows

### Flow A — Manual write (GLADMIN, INSERT/UPDATE/DELETE)

```
Internal-verification-team (GLADMIN, via internal tool)
    │
    ▼
[API — write]  POST /supp_verify_log  serviceName=SUPP_VERIFY_LOG
                {GLUSR_ID, VALIDATION_KEY, IP, IP_COUNTRY, ACTION(INSERT/UPDATE/DELETE),
                 PROCESS_BY, PROCESS_BY_EMPID, VERIFY_LOG_ID?, VERIFY_FLAG, DISPOSITION_TYPE,
                 VOICE_LOG_URL, MULTIPLE_ATTACHMENTS(max 5), ...}
    │  UserSuppVerifyLogController.go
    │  1. MandatoryParamsSuppVerifyLog() — required fields + ACTION-specific requirements
    │  2. Gateway_v1(["GLADMIN",""]) — permission check
    │  3. MULTIPLE_ATTACHMENTS → split, trim, cap at 5, JSON-array-string
    ▼
SuppVerifyLog() model — ValidationSuppVerifyLog() (type/length check)
    ├─ ACTION=INSERT → INSERT INTO IIL_SUPP_VERIFICATION_LOG (21 cols) RETURNING VERIFY_LOG_ID
    ├─ ACTION=UPDATE → dynamic UPDATE (whitelist-driven SET clause, WHERE VERIFY_LOG_ID=$n)
    └─ ACTION=DELETE → DELETE ... WHERE (FK_GLUSR_USR_ID, VERIFY_LOG_ID)
                        → callHistoryPackageMeshpg() → CALL PCK_GL_HISTORY_SP_ADD_CUSTOM_HISTORY(...)
    ▼
[DB — meshpg]  IIL_SUPP_VERIFICATION_LOG (+ GL_ALL_MASTER_HISTORY-equivalent via the SP, on DELETE)
    ▼
Response — MESSAGE overridden to "UPDATE SUCCESS..." even on successful INSERT (section 5.10)
```

### Flow B — Read

```
Any caller (internal tool)
    │
    ▼
[API — read]  GET/POST supp_verify_log/*params  serviceName=SUPPVERIFYLOG  {glusrid, modid, token}
    │  SuppVerifyLogController.go — CheckValidity(token, glusrid, modid)
    ▼
SuppVerifyLogModel(glid)
    ▼
SELECT VERIFY_LOG_ID, CALL_COUNTER, VERIFY_FLAG, IIL_SUPP_FIRST_VERIFY_DATE,
       DISPOSITION_TYPE_ACTIVITY, PROCESS_BY, PROCESS_DATE, PROCESS_BY_EMPID,
       CALL_RESCHEDULE_TIME, VOICE_LOG_URL, OTHER_ATTACHMENTS
FROM IIL_SUPP_VERIFICATION_LOG WHERE FK_GLUSR_USR_ID=$1   (all rows, full history, 100ms timeout)
    ▼
[DB — mesh_pg_user]
    ▼
Response — array of log-entries, dates reformatted to DD-MON-YY, attachments JSON-parsed
```

### Flow C — Automated nightly cron (see section 12 for full per-sub-processor detail)

```
[External scheduler]  → Supp_verify_log_cron.sh → go run Supp_verify_log_cron.go <mode>
    │
    ▼
main()  — connects to meshpg + mainpg; on connection failure, alerts via Google Chat and exits
    ▼
populateSuppVerificationData()
    ├─ processCassUpdateMain()          — ACTIVE — disabled-user detection (mainpg → meshpg insert)
    ├─ processPNSDefaulterRejection()   — ACTIVE — PNS-defaulter verify/reject (mainpg → meshpg insert/update)
    ├─ cntBeforeReceipt()               — ACTIVE — bookkeeping counts from iil_supp_verification
    ├─ processReceiptUsers()            — ACTIVE — receipt-triggered cleanup (mainpg → meshpg delete)
    ├─ cntAfterReceipt()                — ACTIVE — bookkeeping counts from iil_supp_verification
    └─ processCountryRejection()        — ACTIVE — country-based auto-rejection (meshpg delete only)
    │
    │  (5 more sub-processors exist as methods but are NOT invoked here — see section 12:
    │   processFreeVerified [dormant, fully coded], processDataCopy/processC2CVerification/
    │   processHotLeadVerification/processQGFCPVGFCP [dead, bodies fully commented out])
    ▼
finalizeAndSendMail() → builds an HTML summary table (source-wise counts from iil_supp_verification)
                         → Google Chat webhook (SendMessageToGoogleSpace), always, success or failure
```

---

## 10. Flow-wise DB & Table Usage

### Flow A — Manual Write (INSERT / UPDATE / DELETE)

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1 | meshpg | `IIL_SUPP_VERIFICATION_LOG` | INSERT `RETURNING VERIFY_LOG_ID` (ACTION=INSERT only) | Creates the new verification-outcome row, gets back its generated ID |
| 1' | meshpg | `IIL_SUPP_VERIFICATION_LOG` | UPDATE by `VERIFY_LOG_ID` (ACTION=UPDATE only) | Modifies an existing row's whitelisted fields |
| 1'' | meshpg | `IIL_SUPP_VERIFICATION_LOG` | DELETE by `(FK_GLUSR_USR_ID, VERIFY_LOG_ID)` (ACTION=DELETE only) | Removes the row |
| 2 | meshpg | (via stored procedure `PCK_GL_HISTORY_SP_ADD_CUSTOM_HISTORY`) | CALL (ACTION=DELETE only) | Writes an audit-trail entry for the deletion into the shared cross-domain history mechanism (same pattern GST uses for `GL_ALL_MASTER_HISTORY`, though the exact table this SP writes to wasn't directly inspected in this pass) |

**Total DB round-trips per write**: 1 for INSERT/UPDATE, 2 for DELETE (main op + history SP
call) — the simplest flow in this feature.

### Flow B — Read

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1 | mesh_pg_user | `IIL_SUPP_VERIFICATION_LOG` | SELECT (11 columns, all rows for a GLID, no LIMIT) | Returns full verification-history for the supplier |

### Flow C — Nightly Cron (by far the most DB-heavy flow)

| # | DB | Table(s) | Operation | Why |
|---|---|---|---|---|
| 1 | mainpg | `GLUSR_USR_CASS_UPDATE_MAIN` | SELECT | `processCassUpdateMain` — finds GLIDs disabled via `PCK_CUST_SERV_NEW.SP_DISABLE` in the last day |
| 2 | meshpg | `IIL_SUPP_VERIFICATION_LOG` | INSERT (looped, one txn, per matched GLID) | Writes a `VERIFY_FLAG=-1` "Verified/Paid Catalog To FCP" row for each disabled user found |
| 3 | mainpg | `cust_to_serv` + `service` + `custtype` + `glusr_usr` + `pns_defaulter` (5-way join) | SELECT | `processPNSDefaulterRejection` — finds suppliers whose catalog service ended 3-10 days ago, flags whether they're a PNS defaulter |
| 4 | meshpg | `iil_supp_verification_log` | SELECT `count(1)` (per row, inside loop) | `processPNSDefaulterRecord` — checks if this GLID already has a log row (decides INSERT vs UPDATE) |
| 5 | meshpg | `iil_supp_verification_log` | INSERT or UPDATE (per row, same txn) | Writes/updates the verify outcome — `-1` if paid, `-5` if PNS-defaulter-rejected; UPDATE path explicitly skips rows currently at `verify_flag=-3` |
| 6 | meshpg | `iil_supp_verification` | SELECT `count(1)` x2 (`cntBeforeReceipt`) | Bookkeeping: today's VGFCP-type count and today's total-allocated count, for the summary email — no business-logic effect |
| 7 | mainpg | `ar_receipts` + `ar_receipt_types` | SELECT | `processReceiptUsers` — finds suppliers with an actual payment receipt (status 1/2) in the last 60 days |
| 8 | meshpg | `iil_supp_verification` | DELETE (looped, one txn, per matched GLID) | `processReceiptUserRecord` — removes these suppliers from the **calling queue** (not the log table) — presumably because a paid receipt means they no longer need calling-verification |
| 9 | meshpg | `iil_supp_verification` | SELECT `count(1)` (`cntAfterReceipt`) | Bookkeeping: yesterday's allocated count, for the summary email |
| 10 | meshpg | `iil_supp_verification` | DELETE | `processCountryRejection` — removes today's queue-entries whose supplier's `glusr_usr_ip_country` is in a 9-country blocklist (Cameroon, Nigeria, Benin, Ghana, Togo, Cote d'Ivoire, Sierra Leone, Senegal, Burkina Faso) |
| 11 | meshpg | `iil_supp_verification` | SELECT `count(1)` (`finalizeAndSendMail`, total) | Today's total-allocated count for the summary email |
| 12 | meshpg | `iil_supp_verification` | SELECT `count(1) GROUP BY iil_supp_verif_data_source` | Per-source breakdown table for the summary email |

**Total DB round-trips for one cron run**: at minimum ~12 distinct queries/statements across
2 physical databases, **plus one extra SELECT+INSERT/UPDATE pair per matched GLID inside the
PNS-defaulter loop** (steps 4-5 run once per row from step 3, not once total) — this is a
classic N+1 pattern inside a batch job, see Optimization Scope below.

---

## 11. Optimization Scope — DB/Response-Time Contributors

### High-impact

1. **N+1 query pattern inside `processPNSDefaulterRejection`**: for every row returned by the
   `mainpg` join query, `processPNSDefaulterRecord` issues a separate `SELECT count(1)` on
   `meshpg` followed by a separate `INSERT`/`UPDATE` — all inside one shared transaction but as
   fully sequential round-trips, one row at a time. If the daily disabled/defaulter list is
   large, this could dominate cron runtime. **Suggestion**: batch the existence-check (a single
   `WHERE fk_glusr_usr_id = ANY($1)` query) and use `INSERT ... ON CONFLICT` semantics if the
   schema allows, to collapse the per-row check+write into fewer round-trips.
   [`Supp_verify_log_cron.go:257-309`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go)
2. **`processCassUpdateMain` also loops row-by-row for its INSERT** ([`Supp_verify_log_cron.go:148-159`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go))
   and **`processReceiptUsers` loops row-by-row for its DELETE** ([`:363-372`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go))
   — same pattern, lower risk since these are single-statement-per-row (no extra SELECT), but
   still N round-trips instead of 1 batched statement (`unnest($1::bigint[])` style INSERT, or
   `WHERE x = ANY($1)` DELETE).

### Medium-impact

3. **Read endpoint returns full history, no pagination/limit** (section 5.12) — for suppliers
   with a long verification-history (many calls/reschedules/cron-writes), this could return a
   large row-set every time; consider a `LIMIT`/pagination if any caller only needs recent
   entries.
4. **Read endpoint's 100ms timeout is aggressive relative to the write endpoint's 1s** — if
   `IIL_SUPP_VERIFICATION_LOG` grows large per-GLID (plausible given the cron writes to it
   independently of manual calls) and the read has no `LIMIT`, this timeout could start
   tripping under load; worth monitoring together with point 3.

### Low-impact / inherent

5. **DELETE always incurs a second sequential call** (`callHistoryPackageMeshpg`, a
   stored-procedure call) on the write path — inherent to the audit-requirement, not really
   optimizable without dropping the audit-trail guarantee.
6. **Cron's active sub-processors run strictly sequentially** in one batch-job
   (`processCassUpdateMain` → `processPNSDefaulterRejection` → `processReceiptUsers` →
   `processCountryRejection`) — if any one is slow (most likely candidate: the PNS N+1 loop,
   point 1), it dominates total runtime; per-sub-processor timing is already partially logged
   (`fmtMainExecTime`/`fmtPGExecTime`) into the summary Google Chat message, so this is
   observable without extra instrumentation work.

---

## 12. Cron Inventory

| Cron | Live? | Trigger | What it does |
|---|---|---|---|
| `Supp_verify_log_cron.go` | **Yes, live** | External scheduler (not found in-repo — likely crontab on the app server, invoked via `Supp_verify_log_cron.sh`) | Runs `populateSuppVerificationData()` — see the 9-sub-processor breakdown below |

### The 9 `process*` sub-processors — individually traced

Only **4 of the 9** defined sub-processor methods are actually invoked from
`populateSuppVerificationData()` ([`Supp_verify_log_cron.go:89-99`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go)).
This is a materially different picture than "8 sub-processors run nightly" — most of the
business rules coded here are currently **not executing at all**.

| # | Method | Status | What it does (when it runs) |
|---|---|---|---|
| 1 | `processCassUpdateMain()` | **ACTIVE** — called at [`:89`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) | Finds `mainpg` GLIDs disabled via `PCK_CUST_SERV_NEW.SP_DISABLE` in the last day, inserts a `VERIFY_FLAG=-1`, `DISPOSITION_TYPE='Verified'`/`'Paid Catalog To FCP'` row into `IIL_SUPP_VERIFICATION_LOG` per GLID, in one transaction. [`:104-173`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) |
| 2 | `processPNSDefaulterRejection()` + `processPNSDefaulterRecord()` | **ACTIVE** — called at [`:93`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) | Finds `mainpg` suppliers whose catalog service ended 3-10 days ago (5-way join), cross-checks `pns_defaulter` status, then per-GLID either INSERTs (if no existing log row) or UPDATEs (if one exists and isn't `VERIFY_FLAG=-3`) with `-1`/"Verified" if paid, `-5`/"Verification Rejected"/"Catalog PNS Defaulter" if a defaulter. Hardcoded `pnsEmpID=26486`, `pnsEmpName="Manoj Baranwal"` as the attributed processor for every row this rule writes. [`:175-309`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) |
| 3 | `processFreeVerified()` | **DORMANT** — method fully implemented, but the call is commented out at [`:94`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) (`// p.processFreeVerified()`) | Would downgrade `IIL_SUPP_VERIFICATION_LOG` rows currently at `VERIFY_FLAG` `-1`/`-2` to `-7` if they're the latest row for their GLID, older than 330 days, `fcp_flag=1`, and `glusr_usr_custtype_id IN (14,32,34,35)`. Ready to re-enable with a one-line change if the team wants it back. [`:311-328`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) |
| 4 | `cntBeforeReceipt()` / `cntAfterReceipt()` | **ACTIVE** (bookkeeping, not a business rule) — called at [`:96,98`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) | Reads counts from `iil_supp_verification` before/after `processReceiptUsers` runs, purely for the summary email — no writes. [`:329-336`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) |
| 5 | `processReceiptUsers()` + `processReceiptUserRecord()` | **ACTIVE** — called at [`:97`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) | Finds `mainpg` GLIDs with an actual payment receipt (`ar_recpt_status IN (1,2)`, `is_actual_receipt=1`) in the last 60 days, then **deletes** their rows from `iil_supp_verification` (the calling **queue**, not the log table) — one DELETE per GLID inside a shared transaction. [`:337-407`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) |
| 6 | `processCountryRejection()` | **ACTIVE** — called at [`:99`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) | Deletes today's `iil_supp_verification` (queue) rows for suppliers whose `glusr_usr_ip_country` (lowercased) is in a hardcoded 9-country blocklist. No `mainpg` read step — single `meshpg` statement. [`:409-425`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) |
| 7 | `processDataCopy()` | **DEAD** — entire function body is commented out, and the call itself is also commented out at [`:90`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) | Would have copied `ar_receipts_indiamart` GLIDs with `ar_recpt_amount >= 5000` into an `iil_supp_verification_populate_recpt` table. Currently a fully inert no-op function. [`:474-514`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) |
| 8 | `processC2CVerification()` | **DEAD** — body fully commented, call commented out at [`:91`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) | Would have used `CLICK_TO_CALL` + `GLUSR_CLCKSTRM_4HOTLEAD_ARCH` Oracle tables to detect high-frequency click-to-call suppliers over rolling 7/30/60-day windows and update/insert their verification-log row. Inert no-op today. [`:516-632`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) |
| 9 | `processHotLeadVerification()` | **DEAD** — body fully commented, call commented out at [`:92`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) | Would have used `sts_company`/`sts_dsr_sales`/`glusr_clckstrm_4hotlead_arch` to find suppliers flagged as hot-leads in the last 180 days and update/insert their verification-log row. Inert no-op today. [`:634-694`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) |
| — | `processQGFCPVGFCP()` | **DEAD** — body fully commented, call commented out at [`:95`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) | Would have run a `LIMIT 6000` INSERT into `iil_supp_verification` tagged QGFCP/VGFCP — the same tagging convention the SellOnIM consumer uses (per the SellOnIM Log KT doc). This is the piece that would have most directly overlapped with SellOnIM's own queue-population logic; currently inert. [`:696-710`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) |

**Failure/completion alerting**: the cron does not use email or Kibana for alerting — every
path (connection failure, mid-run failure, or successful completion) posts to a **Google Chat
Space webhook** via `utils.SendMessageToGoogleSpace` ([`Supp_verify_log_cron.go:31,38,51,58`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go)).
On success, the message includes an HTML summary table of today's `iil_supp_verification`
counts grouped by `iil_supp_verif_data_source`, plus a text dump of the
before/after-proforma receipt counts and per-sub-processor timing/row-count log
(`recordsCountLog`). On any per-step error, `errorFlag` is set and the **entire accumulated
output buffer is preserved and sent** (normally it's reset and only the summary is sent) —
[`Supp_verify_log_cron.go:456-471`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go).

---

## 13. Edge Cases & Gotchas (technical POV)

1. **5 of the cron's 9 coded business rules are dormant/dead** — anyone reading just the
   function names (`processDataCopy`, `processC2CVerification`, `processHotLeadVerification`,
   `processQGFCPVGFCP`, `processFreeVerified`) would reasonably assume they run nightly; they
   do not. If a business question comes in about "why didn't a hot-lead supplier get
   auto-verified," the answer is: that logic exists in code but has been switched off, not a
   bug in a running path.
2. **Response `MESSAGE` always says "UPDATE SUCCESS" even for a successful INSERT** (section
   5.10) — misleading if caller logic branches on the message text rather than the `ACTION` it
   sent.
3. **`ACTION=UPDATE` is not scoped by `FK_GLUSR_USR_ID`**, only by `VERIFY_LOG_ID` (section
   5.8) — unlike `DELETE`, which requires both. A caller with a valid `VERIFY_LOG_ID` could in
   principle update a row belonging to a different GLID than the one they passed, since GLID
   isn't part of the UPDATE's `WHERE` clause.
4. **`OLD_VERIFY_FLAG` is validated (must be numeric) but never used** anywhere downstream —
   dead input parameter.
5. **`updateMap`-driven UPDATE builds the query via unordered Go map iteration** — column order
   in the generated SQL isn't deterministic run-to-run, though this has no functional impact
   (each column still gets its own placeholder).
6. **This table is fed from at least 3 independent sources**: the manual write-API (GLADMIN),
   the nightly cron's 2 active writing sub-processors (`processCassUpdateMain`,
   `processPNSDefaulterRejection`), and — if `processFreeVerified` is ever re-enabled — a
   fourth. `iil_supp_verification` (the separate queue table) is touched by 3 more cron
   sub-processors (`processReceiptUsers`, `processCountryRejection`, and indirectly the
   bookkeeping counts) plus the unrelated SellOnIM consumer. Full picture of who writes what
   requires reading all sources together.
7. **`processPNSDefaulterRecord`'s UPDATE explicitly protects `verify_flag=-3` rows** from being
   overwritten by this automated rule, but nothing in this codebase ever sets a row to `-3` —
   either that value is set by a process outside this review's scope, or it's a defensive
   guard for a state that no longer occurs. Flagged as an open question below.
8. **Gateway allowlist includes a stray empty-string entry** (`[]string{"GLADMIN", ""}`) on the
   write path — functionally harmless but worth cleanup.
9. **Read query silently swallows a JSON-unmarshal failure on `OTHER_ATTACHMENTS`**, returning
   `nil` for that field rather than surfacing an error — a malformed attachments string would
   be invisible to the caller.
10. **`processReceiptUserRecord`'s DELETE isn't guarded against also deleting a row that
    `processCountryRejection` would have deleted anyway** — no functional bug (DELETE is
    idempotent), but the two active queue-cleanup rules could in principle race/overlap for the
    same GLID within one cron run; not a concern in practice since they run sequentially in the
    same process.

---

## 14. Open Questions

1. What does `VERIFY_FLAG=-3` represent, and what process sets it? It's explicitly protected
   from overwrite in `processPNSDefaulterRecord` but never assigned anywhere in this codebase
   (section 4, 13.7).
2. What does `VERIFY_FLAG=-20` represent, and why does it specifically require `REQUEST_URL`
   on INSERT (section 4, 5.3)? No cron or consumer code sets this value in this review.
3. What does `VERIFY_FLAG=-2` mean here — is it the same convention SellOnIM's consumer treats
   as disqualifying, or independent (section 4)?
4. Why are 5 of the 9 cron sub-processors dormant/dead (section 12)? Is `processFreeVerified`
   intentionally paused, or was it disabled and forgotten? Are `processDataCopy`,
   `processC2CVerification`, `processHotLeadVerification`, `processQGFCPVGFCP` planned for
   removal, or planned to be re-enabled? This materially changes what business rules are
   actually running today vs. what the code suggests at a glance.
5. `PNS` — confirm full expansion (Pay Now Search?) and what a "PNS defaulter" means in
   business terms.
6. `pnsEmpID=26486`/`pnsEmpName="Manoj Baranwal"` are hardcoded as the attributed
   processor for every automated PNS-rule row (section 12, #2) — is this intentional
   (a real employee whose ID represents "system-generated"), or should it be a generic
   system-account ID?
7. Where/how is the cron actually scheduled (crontab entry, Jenkins, or another scheduler)? Not
   found in-repo — only the `.sh` wrapper exists.
8. What does `PCK_GL_HISTORY_SP_ADD_CUSTOM_HISTORY` write to exactly (table name, schema)? Not
   directly inspected in this pass — referenced by name only from the DELETE audit call
   (section 5.9).
9. `meshpg` (write/cron) vs `mesh_pg_user` (read) — same open-question pattern as several other
   recent KTs: are these the same physical database under different connection-pool names, or
   genuinely different instances that could drift?
10. Live DB schema verification — this doc only reflects what the Go SQL strings imply.

---

## See also

- [`Supp_Verify_Log_Business_Doc.md`](./Supp_Verify_Log_Business_Doc.md) — product perspective
- [`../SellOnIM Log KT/SellOnIM_Log_Technical_Doc.md`](../SellOnIM%20Log%20KT/SellOnIM_Log_Technical_Doc.md) —
  upstream calling-queue pipeline that populates `iil_supp_verification`, the table this cron
  also reads/deletes from (via `processReceiptUsers`/`processCountryRejection`/bookkeeping)
  but does not itself populate (`processQGFCPVGFCP`, the sub-processor that would have,
  is dead — section 12)
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — depth/structure
  reference this doc was upgraded to match

# Supplier Verification Log — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`Supp_Verify_Log_Business_Doc.md`](./Supp_Verify_Log_Business_Doc.md)
dekho.

**Scope note**: Related to [`../SellOnIM Log KT/`](../SellOnIM%20Log%20KT/) — SellOnIM's
consumer-pipeline populates `iil_supp_verification` (the calling-queue); this KT covers
`IIL_SUPP_VERIFICATION_LOG` (the verification-outcome-log) — a different table, sequential
concept, not FK-linked in code seen so far.

**Repos**: `service-api-go-production` (write API + a standalone cron), `users-api-go-production`
(read). **Koi RabbitMQ/Kafka/consumer nahi mila** for the write-API path — purely
synchronous CRUD, plus an independent nightly batch-cron.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write | write | [`UserSuppVerifyLogController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSuppVerifyLogController.go), [`UserSuppVerifyLogModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSuppVerifyLogModel.go) (`SuppVerifyLog`, `callHistoryPackageMeshpg`) |
| Write — validation | write | `MandatoryParamsSuppVerifyLog` — [`UserUtilsMandatory.go:1421`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go), `ValidationSuppVerifyLog` — [`UsersValidationMaps.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |
| Read | read | [`SuppVerifyLogController.go`](../../users-api-go-production/internal/controllers/UsersControllers/SuppVerifyLogController.go), [`UsersSuppVerifyLogModel.go`](../../users-api-go-production/internal/models/users/UsersSuppVerifyLogModel.go) (`SuppVerifyLogModel`) |
| Automated batch-populator | write (cron) | [`Supp_verify_log_cron.go`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) (`populateSuppVerificationData` + 8 sub-processors) |

---

## 2. Routes

| Method | serviceName | Repo | Controller |
|---|---|---|---|
| POST | `SUPP_VERIFY_LOG` | write | `SuppVerifyLog` |
| — | `SUPPVERIFYLOG` | read | `SuppVerifyLogController` |
| — | (standalone binary, not HTTP) | cron | `main()` in `Supp_verify_log_cron.go` |

---

## 3. Data Model — Table

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `IIL_SUPP_VERIFICATION_LOG` | write/cron: meshpg / read: mesh_pg_user | Verification-call outcome log | `VERIFY_LOG_ID` (PK, `RETURNING` on insert), `FK_GLUSR_USR_ID`, `VERIFY_FLAG`, `DISPOSITION_TYPE`, `DISPOSITION_TYPE_ACTIVITY`, `CALL_COUNTER`, `PROCESS_BY`, `PROCESS_DATE`, `PROCESS_BY_EMPID`, `VOICE_LOG_URL`, `EMAIL_OPEN_STATUS`, `IIL_SUPP_FIRST_VERIFY_DATE`, `IIL_SUPP_FIRST_VERIFY_BY`, `PRD_LINE_VERIFY`, `OTHER_ATTACHMENTS` (JSON-array-as-string, max 5), `SUPP_VERF_UPDATE_DATE`, `SUPP_VERF_UPDATED_BY_ID`/`_NAME`/`_AGENCY`/`_SCREEN`/`_URL`/`_IP`/`_IP_COUNTRY`, `CALL_RESCHEDULE_TIME` — [`UserSuppVerifyLogModel.go:75,252-272`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSuppVerifyLogModel.go) |

---

## 4. Business Rules & Validation (code se)

### Write-path (`SuppVerifyLog` controller / model)

1. **Gateway allowlist**: `GLADMIN` only (plus a stray empty-string entry in the array —
   `[]string{"GLADMIN", ""}` — likely a leftover/no-op).
   [`UserSuppVerifyLogController.go:75`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSuppVerifyLogController.go)
2. **`MULTIPLE_ATTACHMENTS` capped at 5 comma-separated values**, converted to a JSON
   array-string before storage.
   [`UserSuppVerifyLogController.go:79-95`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSuppVerifyLogController.go)
3. **`ACTION=INSERT`**: full-column insert, `CALL_COUNTER` hardcoded to `1` on first
   insert, `PROCESS_DATE` = `CURRENT_TIMESTAMP`.
4. **`ACTION=UPDATE`**: dynamically-built via a `updateMap` whitelist (input-key →
   DB-column) — only keys present in the request get included in the `SET` clause;
   `PROCESS_DATE`/`SUPP_VERF_UPDATE_DATE` always forced to `CURRENT_TIMESTAMP` if not
   explicitly supplied (`PROCESS_DATE` appended as an extra `SET` clause when absent).
   [`UserSuppVerifyLogModel.go:250-299`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSuppVerifyLogModel.go)
5. **`ACTION=DELETE`**: matched on `(FK_GLUSR_USR_ID, VERIFY_LOG_ID)`, and **always**
   followed by a call to `callHistoryPackageMeshpg()` — invokes a stored-procedure
   `PCK_GL_HISTORY_SP_ADD_CUSTOM_HISTORY(...)` with a hardcoded comment
   `"Gladmin (Data Validation Process)"` and transaction-type `'D'`, logging the deletion
   into a shared history/audit mechanism (`IIL_SUPP_VERIFICATION_LOG-VERIFY_FLAG` tag).
   [`UserSuppVerifyLogModel.go:345-350,359-432`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSuppVerifyLogModel.go)
6. **A quirky response-message override**: if the model returns `"INSERT SUCCESS IN
   MESH PG"`, the controller overwrites the outward-facing `MESSAGE` to say `"UPDATE
   SUCCESS IN MESH PG"` instead — response-text doesn't distinguish insert from update
   for the caller.
   [`UserSuppVerifyLogController.go:157-161`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSuppVerifyLogController.go)

### Read-path (`SuppVerifyLogModel`)

7. **Returns ALL log-rows for a given `FK_GLUSR_USR_ID`** — no single-latest-row
   filtering (unlike Disposition KT's default-latest-only behavior); full history always
   returned.
8. **Date-fields (`PROCESS_DATE`, `IIL_SUPP_FIRST_VERIFY_DATE`) reformatted to
   `DD-MON-YY` uppercase**, `OTHER_ATTACHMENTS` JSON-unmarshaled back into an array for
   the response.

### Automated batch-cron (`Supp_verify_log_cron.go`)

9. **A standalone Go binary (not an HTTP service)**, run on a schedule, connecting to
   both `meshpg` and `mainpg`. On any connection/processing failure, sends an alert via
   `SendMessageToGoogleSpace` (Google Chat webhook) rather than email.
10. **8 independent sub-processors, each implementing a different business-rule for
    auto-generating verification-log/queue entries**:
    - `processCassUpdateMain()` — detects users disabled via
      `PCK_CUST_SERV_NEW.SP_DISABLE` in the last day (`GLUSR_USR_CASS_UPDATE_MAIN`).
    - `processPNSDefaulterRejection()` / `processPNSDefaulterRecord()` — PNS
      (Pay-Now-Search likely) defaulter-based rejection records.
    - `processFreeVerified()` — free-tier-verified suppliers.
    - `processReceiptUsers()` / `processReceiptUserRecord()` — receipt/payment-triggered
      verification.
    - `processCountryRejection()` — country-based auto-rejection.
    - `processDataCopy()` — some data-copy/sync step.
    - `processC2CVerification()` — Click-to-Call-driven verification signals.
    - `processHotLeadVerification()` — hot-lead-based verification.
    - `processQGFCPVGFCP()` — QGFCP/VGFCP-tagged records (same tags seen in SellOnIM's
      `iil_supp_verification` insert — `'QGFCP ACTIVE'`/`'VGFCP'` — confirming this cron
      and the SellOnIM consumer both feed the same underlying verification-pipeline from
      different trigger-sources).
    [`Supp_verify_log_cron.go:104-696`](../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go)
11. **`cntBeforeReceipt()`/`cntAfterReceipt()`/`finalizeAndSendMail()`** — the cron tracks
    before/after record-counts per sub-process and emails/chat-notifies a summary report
    on completion, regardless of success/failure.

---

## 5. RabbitMQ / Kafka / Redis

**Koi RabbitMQ, Kafka, ya Redis usage nahi mila** in the write-API path. The cron is a
scheduled batch-job, not a message-driven consumer.

---

## 6. End-to-End Technical Flow

### Write API (manual, GLADMIN-only)
```
Internal-verification-team (GLADMIN)
    │
    ▼
[API — write]  POST serviceName=SUPP_VERIFY_LOG
                {GLUSR_ID, ACTION(INSERT/UPDATE/DELETE), VERIFY_LOG_ID?, VERIFY_FLAG,
                 DISPOSITION_TYPE, VOICE_LOG_URL, MULTIPLE_ATTACHMENTS(max 5), ...}
    │  SuppVerifyLogController.go — MandatoryParamsSuppVerifyLog() → Gateway(GLADMIN)
    │  attachments → JSON-array-string
    ▼
SuppVerifyLog()  — ValidationSuppVerifyLog()
    ├─ ACTION=INSERT → INSERT INTO IIL_SUPP_VERIFICATION_LOG (...) RETURNING VERIFY_LOG_ID
    ├─ ACTION=UPDATE → dynamic UPDATE (whitelist-driven SET clause)
    └─ ACTION=DELETE → DELETE ... WHERE (user, verify_log_id)
                        → callHistoryPackageMeshpg() → CALL PCK_GL_HISTORY_SP_ADD_CUSTOM_HISTORY(...)
    ▼
[DB — meshpg]  IIL_SUPP_VERIFICATION_LOG
```

### Read
```
Any caller
    │
    ▼
[API — read]  serviceName=SUPPVERIFYLOG  {glusrid, modid}
    │  SuppVerifyLogController.go (read) → SuppVerifyLogModel()
    ▼
SELECT * FROM IIL_SUPP_VERIFICATION_LOG WHERE FK_GLUSR_USR_ID=$1  (all rows, full history)
    ▼
[DB — mesh_pg_user]
    ▼
Response — array of log-entries, dates reformatted, attachments JSON-parsed
```

### Automated nightly cron
```
[Scheduler]
    │
    ▼
Supp_verify_log_cron (standalone binary)
    │  connects to meshpg + mainpg
    ▼
populateSuppVerificationData()
    ├─ processCassUpdateMain()         — disabled-user detection
    ├─ processPNSDefaulterRejection()  — PNS-defaulter rejection
    ├─ processFreeVerified()           — free-tier verified
    ├─ processReceiptUsers()           — receipt-triggered
    ├─ processCountryRejection()       — country-based rejection
    ├─ processDataCopy()               — data-copy step
    ├─ processC2CVerification()        — click-to-call-driven
    ├─ processHotLeadVerification()    — hot-lead-based
    └─ processQGFCPVGFCP()             — QGFCP/VGFCP-tagged (same source as SellOnIM)
    ▼
[DB — meshpg]  IIL_SUPP_VERIFICATION_LOG / iil_supp_verification (varies per sub-process)
    ▼
finalizeAndSendMail() → Google Chat notification (before/after counts, success/failure)
```

---

## 7. Optimization Scope — DB Response-Time Contribution

### Low-Medium impact

1. **Read query returns full history, no pagination/limit** — for suppliers with a long
   verification-history (many calls/reschedules), this could return a large row-set;
   consider a `LIMIT`/pagination if any caller only needs recent entries.
2. **DELETE always incurs a second sequential call** (`callHistoryPackageMeshpg`, a
   stored-procedure call) — inherent to the audit-requirement, not really optimizable
   without dropping the audit-trail guarantee.

### Design-scope note (not performance)

3. **The cron's 8 sub-processors run sequentially in one batch-job** — if any one
   sub-process is slow/heavy (not fully traced in this pass), it could dominate the
   cron's total runtime; worth profiling which sub-process takes longest if the cron's
   overall duration becomes a concern.

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora Supplier Verification Log flowchart yahan dekho](https://lucid.app/lucidchart/ea3b0aa8-cf0f-410d-b709-0893b88aa951/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **Response `MESSAGE` always says "UPDATE SUCCESS" even for a successful INSERT** —
   misleading if caller logic branches on the message text rather than the `ACTION` it
   sent.
2. **Gateway allowlist includes a stray empty-string entry** (`[]string{"GLADMIN", ""}`)
   — functionally harmless (empty validation-key would fail elsewhere) but worth cleanup.
3. **This table is fed from at least 3 independent sources**: the manual write-API
   (GLADMIN), the nightly cron's 8 sub-processors, and (indirectly, via
   `iil_supp_verification` not `IIL_SUPP_VERIFICATION_LOG`) the SellOnIM consumer-pipeline
   — full picture of who's writing what requires reading all sources together.
4. **`updateMap`-driven UPDATE builds the query via unordered Go map iteration** — column
   order in the generated SQL isn't deterministic run-to-run, though this has no
   functional impact (each column still gets its own placeholder).

---

## 10. Open Questions

1. What do `VERIFY_FLAG` values (`-1`, `-2`, `-7` seen as disqualifying in SellOnIM's
   consumer, others unclear) each represent, business-wise? No enum/mapping found.
2. `PNS` — what does it stand for exactly (Pay Now Search? confirm)?
3. Cron's 8 sub-processors — how do they avoid creating duplicate log-entries if a
   supplier matches multiple business-rules in the same run?
4. `meshpg` (write/cron) vs `mesh_pg_user` (read) — same open-question pattern as
   several other recent KTs.
5. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`Supp_Verify_Log_Business_Doc.md`](./Supp_Verify_Log_Business_Doc.md) — product perspective
- [`../SellOnIM Log KT/SellOnIM_Log_Technical_Doc.md`](../SellOnIM%20Log%20KT/SellOnIM_Log_Technical_Doc.md) —
  upstream calling-queue pipeline; shares the `'QGFCP ACTIVE'`/`'VGFCP'` tagging
  convention with this cron's `processQGFCPVGFCP()`

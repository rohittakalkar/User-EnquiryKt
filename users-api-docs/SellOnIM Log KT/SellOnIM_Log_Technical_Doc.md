# SellOnIM Log — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`SellOnIM_Log_Business_Doc.md`](./SellOnIM_Log_Business_Doc.md)
dekho.

**Repos**: `service-api-go-production` (write), `user-temp-consumers-production`
(RabbitMQ-driven eligibility-pipeline). **Koi read-controller kisi bhi repo mein nahi
mila** — write-only journey-log + a conditional downstream fan-out.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write | write | [`UserSellonimLogController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSellonimLogController.go), [`UserSellonimlogModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSellonimlogModel.go) (`SellonIMLogUpdateintoBD`, `SellonIMRabbitMQ`, `getDBValuesforGlid`) |
| Validation | write | `MapSellonIM` (`LengthAndTypeValidations_v1`) — [`UsersValidationMaps.go:566`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |
| Consumer — calling-eligibility pipeline | consumers | [`USER_SELLONIM_CALLING.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SELLONIM_CALLING.go) (`dbActionUserSellOnImCalling`, plus 5 gate-check helper functions) |

---

## 2. Routes

| Method | serviceName | Repo | Controller |
|---|---|---|---|
| POST | `USER_SELLONIM_LOG` | write | `UserSellonimLogContrller` |
| — | `USER_SELLONIM_CALLING` (RabbitMQ queue) | consumers | `dbActionUserSellOnImCalling` |

---

## 3. Data Model — Tables

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `sellonim_log` | approvalpg (write) | Onboarding-journey progress log | `sellonim_log_id` (PK, `RETURNING` on insert), `fk_glusr_usr_id`, `sellonim_mob_number`, `sellonim_fcp_status`, `sellonim_custtype_id`, `sellonim_journey_completed`, `sellonim_modid`, `sellonim_best_time_to_call`, `is_usr_name_entered`, `is_companyname_entered`, `is_email_entered`, `total_products_entered`, `is_address_entered`, `is_gst_entered`, `glusr_usr_approv`, `fk_parent_glusr_id`, `sellonim_add_date`, `sellonim_upd_date` — [`UserSellonimlogModel.go:117-119`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSellonimlogModel.go) |
| `iil_supp_verification_log` | meshPg (consumer, read-only check) | Existing verification-flag history, used as a disqualification-gate | `FK_GLUSR_USR_ID`, `VERIFY_FLAG` (`-1`/`-2`/`-7` = disqualifying) — [`USER_SELLONIM_CALLING.go:169`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SELLONIM_CALLING.go) |
| `active_fcp_details` | meshPg (consumer, read-only check) | FCP priority-range lookup | `FK_GLUSR_USR_ID`, `PRIORITY_RANGE` (must be `0.8` or `0.9` to pass) |
| `glusr_usr` | meshPg (consumer, read-only checks) | Duplicate-email detection, and (write-side) `FK_PARENT_GLUSR_ID`/`GLUSR_USR_APPROV` lookup for the log-write itself | — |
| `GLUSR_USR_COMP_REGISTRATIONS` | meshPg (consumer, read-only check) | Duplicate-GST detection | `gst`, `FK_GLUSR_USR_ID` |
| `iil_supp_verification` | meshPg (consumer — final write target) | The actual "calling/verification queue" — final destination once all 5 gates pass | `Iil_supp_verif_glusr_id`, `IIL_SUPP_VERIF_ALLOCAT_DATE`, `Iil_supp_verif_data_source` (`'QGFCP ACTIVE'`), `IIL_SUPP_VERIF_CUSTTYPE_WT` (`1399`), `iil_supp_verif_priority` (`1`), `iil_supp_verif_type` (`'VGFCP'`), `BEST_TIME_TO_CALL` — [`USER_SELLONIM_CALLING.go:295-308`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SELLONIM_CALLING.go) |

---

## 4. Business Rules & Validation (code se)

### Write-path (`UserSellonimLogContrller` / `SellonIMLogUpdateintoBD`)

1. **Gateway allowlist**: `GLADMIN`, `MAPI`, `M.INDIAMART.COM`, `SELLERMY`, `IMOB`,
   `ANDROID`, `IOS`. [`UserSellonimLogController.go:95`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSellonimLogController.go)
2. **Mandatory**: `GLID`, `MODID`, `VALIDATION_KEY`. `BEST_TIME_TO_CALL` (if present) must
   be valid RFC3339, then reformatted to `YYYY-MM-DD HH24:MI:SS` before storage.
3. **Insert-vs-update decided purely by `LOG_ID` presence** (not a pre-check select) —
   `LOG_ID` empty → `INSERT`, else → `UPDATE`.
   [`UserSellonimLogController.go:142-151`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSellonimLogController.go)
4. **Every write does a preliminary lookup** — `getDBValuesforGlid()` fetches
   `FK_PARENT_GLUSR_ID` and `GLUSR_USR_APPROV` from `glusr_usr` and merges them into
   `inputParams`, so these two derived-columns get written too, even though the caller
   never sends them.
   [`UserSellonimlogModel.go:51,59,75-77`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSellonimlogModel.go)
5. **Update deletes `LOG_ID`, `GLID`, `MOBILE` from the dynamic-column-set before
   building the SET clause** — these are treated as identity/lookup keys, not
   updatable fields, on UPDATE.
   [`UserSellonimlogModel.go:80-84`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSellonimlogModel.go)
6. **On success (`INSERT SUCCESS`/`UPDATE SUCCESS`) AND `CUSTTYPE_ID=="39"`** →
   `SellonIMRabbitMQ()` fires, pushing the full `inputParams` payload (including the
   merged parent-ID/approval-status) to RabbitMQ.
   [`UserSellonimLogController.go:173-186`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSellonimLogController.go)

### Consumer-path (`dbActionUserSellOnImCalling`) — 5-gate eligibility chain

7. **Gate 1 — `getVerifcationLogData`**: fails (disqualifies) if any existing
   `VERIFY_FLAG` for this user is `-1`, `-2`, or `-7`.
8. **Gate 2 — `getFcpDetailsData`**: passes only if `PRIORITY_RANGE` is exactly `0.8` or
   `0.9` — any other value (including valid-but-different priorities) fails this gate.
9. **Gate 3 — `getDuplicateEmailData`**: passes only if NO other `glusr_usr_id` shares
   the same `GLUSR_USR_EMAIL_DUP` value AND is either `fcp_flag=1` or a paid custtype —
   note the **inverted-boolean naming**: a match here returns `false` (disqualify), no
   match returns `true` (pass) — the function name "getDuplicate..." returning `true`
   for "no duplicate found" is a readability trap.
10. **Gate 4 — `getDuplicateGSTData`**: same inverted pattern — passes if no other user
    shares this user's GST.
11. **Gate 5 — `getDuplicateGlid`**: passes if this GLID hasn't been allocated for
    verification in the last 30 days (`iil_supp_verification`).
12. **All 5 gates are chained sequentially with short-circuit `if err == nil &&
    result...`** — each gate only runs if the previous one passed; a single DB-error at
    any gate stops the chain (`d.Nack`, message requeued for retry), while a
    business-rule failure (gate returns `false`, no error) silently `d.Ack`s the
    message with no downstream write — "processed but not qualified", not a failure.
13. **Final insert into `iil_supp_verification`** hardcodes several constant-fields
    (`'QGFCP ACTIVE'`, `1399`, `1`, `'VGFCP'`) — these look like fixed configuration
    for this specific SellOnIM-sourced lead-type, not derived from the message.

---

## 5. RabbitMQ / Kafka / Redis

| Queue / `SERVICENAME` | Publisher | Consumer | Purpose |
|---|---|---|---|
| `USER_SELLONIM_LOG` → routes to `USER_SELLONIM_CALLING` (per domain-wide `serviceToQueueMap`) | `UserSellonimlogModel.go` (`SellonIMRabbitMQ`) — **only when `CUSTTYPE_ID=="39"`** | [`USER_SELLONIM_CALLING.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SELLONIM_CALLING.go) | 5-gate calling-eligibility pipeline, final write to `iil_supp_verification` |

**Koi Kafka ya Redis usage nahi mila.**

---

## 6. End-to-End Technical Flow

```
New supplier / GLADMIN / MAPI / M.INDIAMART.COM / SELLERMY / IMOB / Android / iOS
    │
    ▼
[API — write]  POST serviceName=USER_SELLONIM_LOG
                {GLID, MODID, LOG_ID?, CUSTTYPE_ID, JOURNEY_COMPLETED, IS_USR_NAME,
                 IS_COMPANYNAME, IS_EMAIL, IS_ADDRESS, IS_GST, PROD_CNT,
                 BEST_TIME_TO_CALL?, VALIDATION_KEY}
    │  UserSellonimLogController.go — Gateway check, RFC3339 validation
    ▼
getDBValuesforGlid()  — fetch FK_PARENT_GLUSR_ID, GLUSR_USR_APPROV from glusr_usr,
                         merge into inputParams
    ▼
SellonIMLogUpdateintoBD()
    ├─ LOG_ID empty → INSERT INTO sellonim_log (...) RETURNING sellonim_log_id
    └─ LOG_ID given → UPDATE sellonim_log SET ... WHERE sellonim_log_id=$N AND
                       fk_glusr_usr_id=$M
    ▼
[DB — approvalpg]  sellonim_log
    │
    │  if success AND CUSTTYPE_ID=="39":
    ▼
[RabbitMQ]  SERVICENAME=USER_SELLONIM_LOG → USER_SELLONIM_CALLING
    ▼
[CONSUME]  dbActionUserSellOnImCalling()
    │  Gate 1: getVerifcationLogData()   — VERIFY_FLAG not in {-1,-2,-7}?
    │  Gate 2: getFcpDetailsData()       — PRIORITY_RANGE in {0.8, 0.9}?
    │  Gate 3: getDuplicateEmailData()   — no risky duplicate-email account?
    │  Gate 4: getDuplicateGSTData()     — no duplicate GST?
    │  Gate 5: getDuplicateGlid()        — not verification-allocated in last 30 days?
    │  (all sequential, short-circuit on first failure)
    ▼
  all pass → insertDetail_supp_verification()
             INSERT INTO iil_supp_verification (..., BEST_TIME_TO_CALL)
    ▼
[DB — meshPg]  iil_supp_verification  (final "calling queue")
```

---

## 7. Optimization Scope — DB Response-Time Contribution

### Medium impact

1. **Write-path does 2 sequential DB round-trips** (`getDBValuesforGlid` SELECT, then
   the INSERT/UPDATE) — same pattern flagged in other KTs; could be combined if
   `FK_PARENT_GLUSR_ID`/`GLUSR_USR_APPROV` were resolved via a subquery inside the main
   INSERT/UPDATE instead of a separate round-trip.
2. **Consumer's 5-gate chain is up to 5 sequential DB queries per message**, each only
   running if the prior passed (best-case 1 query, worst-case 5) — this is inherently
   a gating-pipeline so full parallelization isn't straightforward (gates are meant to
   short-circuit), but the **early gates could be reordered by selectivity** (cheapest/
   most-likely-to-fail-first) to minimize average query-count if gate-failure-rates are
   known.

### Low impact

3. **Hardcoded constant-fields in the final insert** (`'QGFCP ACTIVE'`, `1399`, `'VGFCP'`)
   — not a performance concern, but worth flagging as config-in-code (would need a
   code-deploy to change these values for this lead-source).

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora SellOnIM Log flowchart yahan dekho](https://lucid.app/lucidchart/f7567b1e-40c4-456f-a257-1c889102a9e1/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **Gate-function naming is misleading** — `getDuplicateEmailData`/`getDuplicateGSTData`/
   `getDuplicateGlid` return `true` when NO duplicate is found (i.e., "pass"), not when a
   duplicate exists — easy to misread during debugging.
2. **A gate-failure (business-rule "not eligible") is `d.Ack`'d, not `d.Nack`'d** — only
   a genuine DB/query-error triggers a `Nack`/retry. This is correct behavior (don't
   retry a legitimate ineligibility) but means there's no automatic re-evaluation if the
   supplier's disqualifying condition later resolves (e.g., duplicate-GST cleared) —
   they'd need a fresh `USER_SELLONIM_LOG` write to re-trigger the pipeline.
3. **`CUSTTYPE_ID=="39"` is a hardcoded magic-number gate** for the entire downstream
   pipeline — no comment/constant explaining what customer-type 39 represents.
4. **Koi read-endpoint nahi mila** — journey-progress ko kaise view/monitor kiya jaata
   hai (dashboard, direct-DB-query) confirm nahi hua is pass mein.

---

## 10. Open Questions

1. `CUSTTYPE_ID=39` — what customer-type does this represent, business-wise?
2. Consumer's disqualification is permanent per-message — is there a periodic
   re-evaluation/retry mechanism elsewhere for suppliers who become eligible later?
3. Hardcoded constants in `insertDetail_supp_verification` (`'QGFCP ACTIVE'`, `1399`,
   `'VGFCP'`) — are these specific to this one lead-source, or shared config that should
   be centralized?
4. Is there any read/reporting surface for `sellonim_log` outside these 3 repos?
5. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`SellOnIM_Log_Business_Doc.md`](./SellOnIM_Log_Business_Doc.md) — product perspective
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) —
  duplicate-GST check (Gate 4) reads from the same `GLUSR_USR_COMP_REGISTRATIONS` table
  documented there

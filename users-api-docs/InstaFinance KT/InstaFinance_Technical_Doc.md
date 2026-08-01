# InstaFinance — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`InstaFinance_Business_Doc.md`](./InstaFinance_Business_Doc.md)
dekho.

**Repos**: `service-api-go-production` (write only). **Koi read-controller ya dedicated
consumer nahi mila** — write-only toggle-endpoint, RabbitMQ-fan-out ke saath.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write | write | [`UserInstaFinanceController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserInstaFinanceController.go), [`UserInstaFinanceModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserInstaFinanceModel.go) (`UpsertInstaFinance`) |
| Validation | write | `MandatoryParamsCheckInstafinance` — [`UserUtilsMandatory.go:2441`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go), `UserInstaFinanceMap` (`LengthAndTypeValidations_v3`) |

---

## 2. Routes

| Method | serviceName | Repo | Controller |
|---|---|---|---|
| POST | `INSTAFINANCE_SERVICE` | write | `UserInstaFinanceController` |

---

## 3. Data Model — Table

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `COMP_MASTER_FINANCIALS` | meshpg | Company-level financial-services-eligibility record | `COMPANY_CIN` (match-key, not `FK_GLUSR_USR_ID` — company-identity, not user-identity), `COMP_MASTER_FINANCIAL_ENABLED` (`0`=enabled, `-1`=disabled), `update_date` (`CURRENT_TIMESTAMP`) — [`UserInstaFinanceModel.go:28`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserInstaFinanceModel.go) |

**Update-only, no insert path exists in this controller** — the query is always
`UPDATE ... WHERE COMPANY_CIN = $2`; a non-existent CIN simply returns "no record
found," it never creates one.

---

## 4. Business Rules & Validation (code se)

1. **Gateway allowlist is the narrowest seen across this entire KT series**: just
   `["MY"]` — a single caller-identity allowed.
   [`UserUtilsMandatory.go:2469`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
2. **Mandatory fields**: `COMPANY_CIN`, `COMP_MASTER_FINANCIAL_ENABLED`, `UPDATEDBY`,
   `UPDATEDUSING`.
3. **`COMP_MASTER_FINANCIAL_ENABLED` must be exactly `"0"` or `"-1"`** — any other
   numeric value rejected as `"length exceeded"` (a slightly misleading error-message
   for what's actually a value-whitelist check, not a length-check).
4. **`COMPANY_CIN` must be exactly 21 characters** — matches India's standard
   Corporate-Identification-Number format.
5. **Table match-key is `COMPANY_CIN`, not a `glusr_usr_id`** — this is a genuinely
   company-entity-keyed table, distinguishing it from almost every other feature in this
   KT series which keys off the supplier's user-ID.
6. **On successful update (rows-affected > 0), a RabbitMQ event fires** —
   `SERVICENAME=INSTAFINANCE`, `TABLES=COMP_MASTER_FINANCIALS`, `ACTION=UPDATE`, with the
   changed columns embedded.
   [`UserInstaFinanceModel.go:49-77`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserInstaFinanceModel.go)

### Notable implementation issues (not business rules, code-quality findings)

7. **`env := "dev"` is hardcoded in the controller** rather than calling
   `config.GetEnv()` (which every other controller in this codebase uses) — this means
   the New Relic transaction-tracing block (`if env == "prod" { ... }`) **can never
   execute**, even when actually running in production. This is very likely an
   accidental leftover from local-debugging that wasn't reverted before merge.
   [`UserInstaFinanceController.go:18`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserInstaFinanceController.go)
8. **The controller-level `gate` variable is NOT derived from the `Gateway_v1` result**
   — it's just `utils.FormatToString(inputParams["modid"])`, a raw echo of whatever
   `modid` the caller sent, used only for logging. The actual gateway-validation
   (`Gateway_v1([]string{"MY"}, ...)`) happens entirely inside
   `MandatoryParamsCheckInstafinance`, and its resolved value is written back into
   `inputParams["modid"]` on success — this works correctly, but the split between
   "logged gate" and "actually-validated gate" is an unusual pattern compared to other
   controllers where `gate` is the direct `Gateway_v1` return value.

---

## 5. RabbitMQ / Kafka / Redis

| Queue / `SERVICENAME` | Publisher | Consumer | Purpose |
|---|---|---|---|
| `INSTAFINANCE` | `UserInstaFinanceModel.go` (`UpsertInstaFinance`) — only on successful update | **No dedicated consumer found** — `USER_UPSERT_ADDITIONAL_GCP_IN.go` is a generic multi-field consumer referenced in the broader grep results but its handling of this specific `SERVICENAME` wasn't confirmed in this pass | Presumably syncs the enabled/disabled flag to a downstream system (search/directory, financial-partner-integration) |

**Koi Kafka ya Redis usage nahi mila.**

---

## 6. End-to-End Technical Flow

```
Internal-tool ("MY" only)
    │
    ▼
[API — write]  POST serviceName=INSTAFINANCE_SERVICE
                {COMPANY_CIN, COMP_MASTER_FINANCIAL_ENABLED(0/-1), UPDATEDBY,
                 UPDATEDUSING, VALIDATION_KEY}
    │  UserInstaFinanceController.go
    │  MandatoryParamsCheckInstafinance() — Gateway(["MY"]), CIN-length, flag-whitelist
    ▼
UpsertInstaFinance()
    │  UPDATE COMP_MASTER_FINANCIALS SET COMP_MASTER_FINANCIAL_ENABLED=$1,
    │         update_date=CURRENT_TIMESTAMP WHERE COMPANY_CIN=$2
    ▼
[DB — meshpg]
    ├─ 0 rows affected → "NO RECORD FOUND FOR PROVIDED COMPANY_CIN"
    └─ rows affected   → "UPDATE SUCCESS IN MESHPG"
                          → [RabbitMQ] SERVICENAME=INSTAFINANCE
    ▼
Response
```

---

## 7. Optimization Scope — DB Response-Time Contribution

### Low impact / good practice already present

1. **Single-query update, no sequential round-trips** — straightforward, low-risk.

### Correctness-scope note (not performance)

2. **Hardcoded `env := "dev"` (§4, point 7)** — while not a DB-performance issue, this
   is a real production-observability gap: this endpoint's requests never get New Relic
   transaction-tracing, making it harder to diagnose latency/errors for this specific
   service compared to every other endpoint in the codebase.

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora InstaFinance flowchart yahan dekho](https://lucid.app/lucidchart/3aeb690b-a3a6-44b9-bfae-33c3d2814ac3/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **No New Relic tracing in production** (§4, point 7) — this is the most notable
   finding in this KT: `env := "dev"` hardcoded means this endpoint is effectively
   invisible to APM/tracing tooling regardless of actual deployment environment. Worth
   flagging for a fix.
2. **No insert path** — if a company's `COMP_MASTER_FINANCIALS` record doesn't exist
   yet, this endpoint cannot create one; some other process must be responsible for the
   initial record's creation, not traceable from this file alone.
3. **Error-message wording** ("length exceeded" for a value-whitelist failure on
   `COMP_MASTER_FINANCIAL_ENABLED`) could mislead a caller into thinking it's a
   length-validation issue rather than an allowed-values issue.

---

## 10. Open Questions

1. Is `env := "dev"` an intentional debug-leftover that should be fixed to
   `config.GetEnv()`, matching every other controller in this codebase?
2. Which process creates the initial `COMP_MASTER_FINANCIALS` row per company — not
   found in this codebase.
3. Does `USER_UPSERT_ADDITIONAL_GCP_IN.go` (or another consumer) actually handle the
   `INSTAFINANCE` `SERVICENAME` — not confirmed in this pass.
4. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`InstaFinance_Business_Doc.md`](./InstaFinance_Business_Doc.md) — product perspective

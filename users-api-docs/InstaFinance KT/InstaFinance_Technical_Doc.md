# InstaFinance — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`InstaFinance_Business_Doc.md`](./InstaFinance_Business_Doc.md)
dekho — dono docs same flow cover karte hain, bas alag audience ke liye.

**Repos**: `service-api-go-production` (write), `user-temp-consumers-production` (consumer/retry
path). **`users-api-go-production` (read repo) mein InstaFinance ka koi trace nahi mila** — grep
`COMP_MASTER_FINANCIAL|InstaFinance|Instafinance` sirf doc-files mein match karta hai, kisi
controller/model file mein nahi. Isliye yeh **write + async-retry-only feature** hai, koi
dedicated read-endpoint nahi hai.

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file:line diya gaya
hai). Jahan code se pura confirm nahi ho paaya, wahan **[INFERRED — team se confirm karo]**
likha hai.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write — controller | write | [`UserInstaFinanceController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserInstaFinanceController.go) |
| Write — model/query + RabbitMQ publish | write | [`UserInstaFinanceModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserInstaFinanceModel.go) (`UpsertInstaFinance`) |
| Validation — gateway/mandatory-field/value checks | write | `MandatoryParamsCheckInstafinance` — [`UserUtilsMandatory.go:2441-2496`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go) |
| Validation — length/type map | write | `UserInstaFinanceMap` — [`UsersValidationMaps.go:1993-2000`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go), consumed via `LengthAndTypeValidations_v3` |
| Route registration | write | [`router.go:179`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) — `users.POST("/instafinance", UserControllers.UserInstaFinanceController)` |
| RabbitMQ queue resolution (`SERVICENAME`→queue map) | write | [`rabbitmq.go:59`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go) — `"INSTAFINANCE": "user.insta." + modulus` |
| Consumer/retry — dispatch entrypoint | consumers | [`USER_UPSERT_ADDITIONAL_GCP_IN.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_ADDITIONAL_GCP_IN.go), function `ActionUserUpsertAdditionalGcpIn` — dispatches on `COL["COMP_MASTER_FINANCIAL_ENABLED"]` presence (line 128) |
| Consumer/retry — actual DML | consumers | [`additional.go:342-365`](../../user-temp-consumers-production/user-temp-consumers-production/pkg/utils/additional.go), function `Instafinance` |
| Consumer registration (queue-name → worker function) | consumers | [`router.go:93`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Router/router.go) and [`IntializeMsgBroker.go:349`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go) — both keyed `USER_UPSERT_ADDITIONAL_GCP_IN` |

**No read-controller, no dedicated `INSTAFINANCE`-only consumer, no cron found** anywhere in the
three repos searched (grep for `InstaFinance`/`Instafinance`/`INSTAFINANCE`/`COMP_MASTER_FINANCIAL`
case-insensitive across `users-api-go-production`, `service-api-go-production`,
`user-temp-consumers-production`).

---

## 2. Routes

| Method | Path | serviceName (in-body) | Repo | Controller |
|---|---|---|---|---|
| POST | `/instafinance` | `INSTAFINANCE_SERVICE` (log/txn name, hardcoded const) | write | `UserInstaFinanceController` — [`router.go:179`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) |

---

## 3. Data Model — Table

> **Verification note**: table/column names neeche Go code ke andar embedded SQL strings se
> liye gaye hain (har row ke saamne file:line diya hai). Live DB schema se pgAdmin pe
> cross-verify **nahi** kiya gaya hai.

| Table | Physical DB (write path) | Physical DB (consumer/retry path) | Purpose | Key columns (code se) |
|---|---|---|---|---|
| `COMP_MASTER_FINANCIALS` | **meshpg** — `config.GetPGDbConnection("meshpg")` [`UserInstaFinanceModel.go:19`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserInstaFinanceModel.go) | **buyerprofilePg** — `utils.GetDatabaseConnectionv1("buyerprofilePg", ...)` [`USER_UPSERT_ADDITIONAL_GCP_IN.go:28`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_ADDITIONAL_GCP_IN.go), same table/query run against that connection [`additional.go:355-360`](../../user-temp-consumers-production/user-temp-consumers-production/pkg/utils/additional.go) | Company-level financial-services-eligibility record | `COMPANY_CIN` (match-key, not `FK_GLUSR_USR_ID`), `COMP_MASTER_FINANCIAL_ENABLED` (`0`=enabled, `-1`=disabled), `update_date` (`CURRENT_TIMESTAMP`) |

**Update-only, no insert path exists anywhere in this feature (write OR consumer)** — both
queries are `UPDATE ... WHERE COMPANY_CIN = $2`; a non-existent CIN returns "no record found,"
it never creates one.

**Notable inconsistency (flagging, not fabricating a fix)**: the synchronous write path connects
to **meshpg**, but the async consumer/retry path (`Instafinance()` in `additional.go`) runs the
*same* `UPDATE COMP_MASTER_FINANCIALS` statement against a connection obtained under the label
**`buyerprofilePg`** — the `dbname` string `"MESHPG"` passed into `Instafinance()` is only a
log-label, not the actual connection selector; the real connection is `buyerprofilePgconn`
(`dbConnStruct.SqlDbConnSlice[1]`, from `GetDatabaseConnectionv1("buyerprofilePg", ...)` at
[`USER_UPSERT_ADDITIONAL_GCP_IN.go:28`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_ADDITIONAL_GCP_IN.go)).
Whether `meshpg` and `buyerprofilePg` are the same physical Postgres instance under different
connection-string aliases, or genuinely different databases, is **[INFERRED — confirm with
DBA/infra team]** — if they are different databases, a retried InstaFinance update would land in
a different physical table than the original synchronous write, silently diverging from it.

---

## 4. Business Rules & Validation (code se)

1. **Gateway allowlist is the narrowest seen across this KT series**: just `["MY"]` — a single
   caller-identity allowed, via `utils.Gateway_v1([]string{"MY"}, validationKey, "k")`.
   [`UserUtilsMandatory.go:2469`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
2. **Mandatory fields**: `COMPANY_CIN`, `COMP_MASTER_FINANCIAL_ENABLED`, `UPDATEDBY`,
   `UPDATEDUSING` — all four required or request fails with a combined error message.
   [`UserUtilsMandatory.go:2477-2478`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
3. **`COMP_MASTER_FINANCIAL_ENABLED` must be numeric first** (`utils.IsNumeric`), **then**
   exactly `"0"` or `"-1"`** — any other value (including other numerics like `"1"`) rejected as
   `"COMP_MASTER_FINANCIAL_ENABLED length exceeded."` — a misleading message for what's actually
   a value-whitelist check, not a length-check.
   [`UserUtilsMandatory.go:2481-2486`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
4. **`COMPANY_CIN` must be exactly 21 characters** — checked twice: once implicitly via
   `UserInstaFinanceMap`'s `length: 21` type-map entry (`LengthAndTypeValidations_v3`), and once
   explicitly via `len(cin) != 21` in `MandatoryParamsCheckInstafinance`.
   [`UsersValidationMaps.go:1994`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go),
   [`UserUtilsMandatory.go:2487-2488`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
5. **Table match-key is `COMPANY_CIN`, not a `glusr_usr_id`** — a genuinely company-entity-keyed
   table, distinguishing it from most other write-features in this codebase which key off the
   supplier's user-ID (`GLUSR_ID` is present in the input map and validation map, but only used
   for the RabbitMQ event payload and logging — never in the `WHERE` clause of either the write
   or retry query).
   [`UserInstaFinanceModel.go:28-29`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserInstaFinanceModel.go)
6. **Validation order matters for the error message returned**: `jsoncheck` → mandatory-fields
   → gateway → `IsNumeric` → `LengthAndTypeValidations_v3` (`valid`) → value-whitelist
   (`"0"`/`"-1"`) → CIN-length — so, e.g., an empty CIN with a bad flag value returns the
   mandatory-fields message, not the CIN-length one, even though both are wrong.
   [`UserUtilsMandatory.go:2475-2492`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
7. **On successful update (rows-affected > 0), a RabbitMQ event fires unconditionally** —
   `SERVICENAME=INSTAFINANCE`, `MESSAGE[0].TABLES=COMP_MASTER_FINANCIALS`, `ACTION=UPDATE`, with
   `COMP_MASTER_FINANCIAL_ENABLED` and `COMPANY_CIN` embedded in `COLUMNS`. No `GLUSR_ID`
   validation is done before this — whatever was in `inputParams["GLUSR_ID"]` (which is *not* a
   mandatory field, see point 2) goes straight into the message, and also determines which
   sharded queue (`user.insta.<glid%20>`) the message lands on (section 5). If `GLUSR_ID` is
   absent/blank, `InterfacetoInt` on an empty value effectively routes to shard `user.insta.0`.
   [`UserInstaFinanceModel.go:49-77`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserInstaFinanceModel.go)
8. **`ExecuteQuery` has a hard 1-second timeout** (`1*time.Second` passed as the 4th argument) —
   if the meshpg `UPDATE` takes longer than 1s, the request fails/cancels rather than waiting.
   [`UserInstaFinanceModel.go:34`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserInstaFinanceModel.go)

### Notable implementation issues (not business rules, code-quality findings)

9. **`env := "dev"` is hardcoded in the controller** rather than calling `config.GetEnv()` (which
   every other controller in this codebase uses) — this means the New Relic
   transaction-tracing block (`if env == "prod" { ... }`) **can never execute**, even in an
   actual production deployment. Very likely an accidental leftover from local-debugging that
   wasn't reverted before merge.
   [`UserInstaFinanceController.go:18-24`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserInstaFinanceController.go)
10. **The controller-level `gate` variable is NOT derived from the `Gateway_v1` result** — it's
    just `utils.FormatToString(inputParams["modid"])`, a raw echo of whatever `modid` the caller
    sent, used only for logging. The actual gateway-validation (`Gateway_v1(["MY"], ...)`)
    happens entirely inside `MandatoryParamsCheckInstafinance`, and its resolved value is written
    back into `inputParams["modid"]` on success (point 6 above) — this works correctly, but the
    split between "logged gate" and "actually-validated gate" is unusual versus other controllers
    where `gate` is the direct `Gateway_v1` return value.
    [`UserInstaFinanceController.go:28-42`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserInstaFinanceController.go)
11. **Retry/consumer path writes to a differently-labelled DB connection than the sync write
    path** — see section 3's "Notable inconsistency."

---

## 5. RabbitMQ

GST-style dedicated section: this domain's messaging is entirely RabbitMQ, resolved through the
shared `PushToQueue` helper's `serviceToQueueMap`.

| Queue / `SERVICENAME` | Resolves to (actual queue) | Publisher | Consumer | Purpose |
|---|---|---|---|---|
| `INSTAFINANCE` | `user.insta.<GLUSR_ID % 20>` — [`rabbitmq.go:59`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go) | `UserInstaFinanceModel.go` (`UpsertInstaFinance`) — only on successful update, i.e. rows-affected > 0 | `USER_UPSERT_ADDITIONAL_GCP_IN` worker (`ActionUserUpsertAdditionalGcpIn`), dispatched to `utils.Instafinance()` whenever the message's `COLUMNS.COMP_MASTER_FINANCIAL_ENABLED` key is present and non-empty — [`USER_UPSERT_ADDITIONAL_GCP_IN.go:128-148`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_ADDITIONAL_GCP_IN.go) | Re-applies the same `UPDATE COMP_MASTER_FINANCIALS` write, presumably as a durability/sync mechanism so a second physical target (buyerprofilePg-labelled connection, see section 3) stays in sync with the primary write |
| (retry-on-failure) | `requeue_additional_GCP_IN` (config key, resolved value not traced in this pass) | `USER_UPSERT_ADDITIONAL_GCP_IN.go` (`utils.Requeue`), only if `Instafinance()` returns an error | Presumably itself again (self-requeue loop) | If the consumer-side UPDATE fails, the whole original message is republished onto the requeue queue rather than being dropped — [`USER_UPSERT_ADDITIONAL_GCP_IN.go:135-143`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_ADDITIONAL_GCP_IN.go) |

**Important structural note not obvious from the queue name**: the consumer that actually
processes `INSTAFINANCE` messages is registered under the key `USER_UPSERT_ADDITIONAL_GCP_IN`
([`router.go:93`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Router/router.go),
[`IntializeMsgBroker.go:349`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go)),
**not** a queue literally named `INSTAFINANCE` or `user.insta.*` — the actual RabbitMQ
queue-name-to-binding topology (i.e. is a queue physically named `USER_UPSERT_ADDITIONAL_GCP_IN`
bound to the `user.insta.*` topic pattern, or something else) lives in RabbitMQ/infra
configuration, **not in this codebase** — **[INFERRED — confirm actual queue/binding name with
infra team]**. The `serviceName` variable inside the consumer is overwritten from the message's
own `SERVICENAME` field at runtime (`serviceName = message["SERVICENAME"]`), so logs for this
consumer will correctly show `INSTAFINANCE` even though the Go function/file name suggests a
more generic "additional GCP IN" purpose — this worker is a shared multi-purpose dispatcher (it
also handles `VERIFIED_FLAG`/`Glusr_vbb` messages, see [`USER_UPSERT_ADDITIONAL_GCP_IN.go:102-124`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_ADDITIONAL_GCP_IN.go)), not an InstaFinance-exclusive one.

This corrects the previous (shallow) doc's claim of "no dedicated consumer found" — a consumer
**was found** in this deeper pass; it's just shared/generic rather than InstaFinance-exclusive.

---

## 6. Kafka

**Koi Kafka usage InstaFinance ke liye kahin nahi mila.** Grep for `Kafka`/`InitializeKafka`
against `USER_UPSERT_ADDITIONAL_GCP_IN.go` and the write-side controller/model returns no
matches. This feature is 100% RabbitMQ.

---

## 7. Redis

**Koi Redis usage InstaFinance ke liye kahin nahi mila** — write controller, model, and the
consumer dispatch/DML functions have no Redis calls. Reads of `COMP_MASTER_FINANCIALS` (by
whatever downstream system consumes the flag) are not traceable in these three repos at all
(section 1) — no caching layer visible because no read-path is visible.

---

## 8. Cron Inventory

Checked `service-api-go-production/crons/**` (both `ML_Retail` and `recommend`/`users`
subfolders) and `user-temp-consumers-production` for any InstaFinance/`COMP_MASTER_FINANCIAL`
reference.

| Cron | Found? |
|---|---|
| Any InstaFinance-related cron | **Nahi mila.** No cron file references `InstaFinance`, `Instafinance`, `INSTAFINANCE`, or `COMP_MASTER_FINANCIAL` anywhere in the crons directories searched. |

---

## 9. End-to-End Technical Flow

```
Internal-tool ("MY" only)
    │
    ▼
[API — write, service-api-go-production]  POST /instafinance  serviceName=INSTAFINANCE_SERVICE
                {COMPANY_CIN, COMP_MASTER_FINANCIAL_ENABLED(0/-1), UPDATEDBY,
                 UPDATEDUSING, GLUSR_ID(optional), VALIDATION_KEY}
    │  UserInstaFinanceController.go
    │  MandatoryParamsCheckInstafinance():
    │    Gateway(["MY"]) → mandatory-fields → IsNumeric → LengthAndTypeValidations_v3
    │    → flag whitelist (0/-1) → CIN length==21
    ▼
UpsertInstaFinance()
    │  UPDATE COMP_MASTER_FINANCIALS SET COMP_MASTER_FINANCIAL_ENABLED=$1,
    │         update_date=CURRENT_TIMESTAMP WHERE COMPANY_CIN=$2   (meshpg, 1s timeout)
    ▼
[DB — meshpg]
    ├─ 0 rows affected → "NO RECORD FOUND FOR PROVIDED COMPANY_CIN"  (no RabbitMQ publish)
    └─ rows affected>0 → "UPDATE SUCCESS IN MESHPG"
                          │
                          ▼
                     [RabbitMQ publish]  SERVICENAME=INSTAFINANCE → queue user.insta.<glid%20>
                          │
                          ▼
                     [CONSUME — user-temp-consumers-production]
                     ActionUserUpsertAdditionalGcpIn()
                       │  detects COLUMNS.COMP_MASTER_FINANCIAL_ENABLED present
                       ▼
                     utils.Instafinance()
                       │  UPDATE COMP_MASTER_FINANCIALS ... WHERE COMPANY_CIN=$2
                       │  (buyerprofilePg-labelled connection — see section 3 discrepancy)
                       ├─ success → Ack, log success
                       └─ failure → utils.Requeue() onto requeue_additional_GCP_IN → retry loop
    ▼
Response (HTTP 200/500 with STATUS/CODE/MESSAGE)
```

---

## 10. Flow-wise DB & Table Usage

### Flow: Successful InstaFinance flag update

| # | DB (physical) | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `COMP_MASTER_FINANCIALS` | UPDATE (`COMP_MASTER_FINANCIAL_ENABLED`, `update_date`) WHERE `COMPANY_CIN` | Primary synchronous write — the actual toggle |
| 2 | buyerprofilePg-labelled connection (async, via RabbitMQ consumer) | `COMP_MASTER_FINANCIALS` | UPDATE (identical statement) WHERE `COMPANY_CIN` | Async re-application of the same update — role/purpose (redundancy vs. genuinely-separate-DB sync) is **[INFERRED — confirm with team]**, see section 3 |

**Total DB round-trips for one successful InstaFinance change: 2** — one synchronous (blocking
the API response), one asynchronous (via consumer, does not block the caller). This is the
simplest flow in the KT series covered so far — single-table, single-predicate, no
joins/lookups/multi-table fan-out.

### Flow: CIN not found

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `COMP_MASTER_FINANCIALS` | UPDATE (0 rows affected) | Same query runs, but `WHERE COMPANY_CIN=$2` matches nothing — no RabbitMQ publish happens in this branch (point 7, section 4) |

---

## 11. Optimization Scope — DB Response-Time Contribution

### Low impact / good practice already present

1. **Single-query synchronous write, no sequential round-trips in the request path** —
   straightforward, low blocking-latency risk for the caller. The heavier consumer-side retry
   work happens off the request's critical path.
2. **Explicit 1-second query timeout already in place** (`UserInstaFinanceModel.go:34`) — this
   protects the request from hanging indefinitely on a slow meshpg query, a good defensive
   pattern not seen consistently applied everywhere in this codebase.

### Correctness-scope notes (not raw performance, but affect response reliability)

3. **Hardcoded `env := "dev"`** (section 4, point 9) — a real production-observability gap: this
   endpoint's requests never get New Relic transaction-tracing, making latency/error diagnosis
   for this specific endpoint harder than for every other endpoint in the codebase.
4. **Connection-label discrepancy between write and retry paths** (section 3) — if `meshpg` and
   `buyerprofilePg` are genuinely different physical databases, every consumer-side retry writes
   to a different target than the original request, which would silently desync the two and make
   "why did the retry not fix my data" support cases very confusing. Worth a direct infra
   confirmation before assuming this is benign.

---

## 12. Edge Cases & Gotchas (technical POV)

1. **No New Relic tracing in production** (section 4, point 9) — `env := "dev"` hardcoded means
   this endpoint is invisible to APM/tracing tooling regardless of actual deployment environment.
2. **No insert path anywhere** — neither the synchronous write nor the async consumer retry can
   create a `COMP_MASTER_FINANCIALS` row; some other process must be responsible for the initial
   record's creation, not traceable from any of the three repos searched.
3. **Error-message wording** ("length exceeded" for a value-whitelist failure on
   `COMP_MASTER_FINANCIAL_ENABLED`) could mislead a caller into thinking it's a length-validation
   issue rather than an allowed-values issue.
4. **`GLUSR_ID` is optional but silently drives RabbitMQ shard routing** — it's not in the
   mandatory-fields list (section 4, point 2), yet it determines both the outbound message
   payload and which of the 20 `user.insta.N` shards the message lands on. A caller who omits it
   won't get an error, but message distribution/ordering could be affected.
5. **Consumer-side retry connects to a differently-labelled DB than the sync write** (section 3)
   — flagged as the most notable new finding versus the original doc.
6. **The consumer that handles `INSTAFINANCE` messages is a shared, generically-named worker**
   (`USER_UPSERT_ADDITIONAL_GCP_IN`), not InstaFinance-exclusive — it also processes
   `VERIFIED_FLAG`/`Glusr_vbb` (verified-buyer-badge) messages in the same function. A bug or
   slowdown in one code path (e.g. `Glusr_vbb`) could delay processing of InstaFinance messages
   sharing the same consumer/queue.

---

## 13. Open Questions

1. Is `env := "dev"` an intentional debug-leftover that should be fixed to `config.GetEnv()`,
   matching every other controller in this codebase?
2. Which process creates the initial `COMP_MASTER_FINANCIALS` row per company — not found in any
   of the three repos searched.
3. Are `meshpg` (write path) and `buyerprofilePg` (consumer retry path, section 3) the same
   physical Postgres instance under different connection aliases, or genuinely different
   databases? If different, is the consumer-side retry writing to the wrong target?
4. What is the actual RabbitMQ queue name and binding pattern behind the
   `USER_UPSERT_ADDITIONAL_GCP_IN` consumer key, and does it truly bind to the `user.insta.*`
   topic pattern that `PushToQueue` resolves `INSTAFINANCE` to? Not confirmed in this codebase
   (infra/RabbitMQ config, not Go code).
5. What does `requeue_additional_GCP_IN` (the requeue-on-failure queue, section 5) actually
   resolve to at runtime, and is it monitored/alerted on?
6. Which downstream system(s) actually read `COMP_MASTER_FINANCIALS.COMP_MASTER_FINANCIAL_ENABLED`
   to decide financial-services eligibility — no read-path exists in any of the three repos
   searched, so this flag's consumer is entirely outside this codebase's visibility.
7. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi reflect
   kiya hai.

---

## See also

- [`InstaFinance_Business_Doc.md`](./InstaFinance_Business_Doc.md) — product perspective
- [`../user_consumers_reference.md`](../user_consumers_reference.md) — confirms
  `USER_UPSERT_ADDITIONAL_GCP_IN` dispatches to `utils.Instafinance` (financial flag,
  buyerprofilePg)
- [`../service_api_write_reference.md`](../service_api_write_reference.md) — confirms route,
  write target, and the `env := "dev"` bug flag independently

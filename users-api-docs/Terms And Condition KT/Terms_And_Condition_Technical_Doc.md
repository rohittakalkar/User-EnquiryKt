# Terms And Condition Acceptance — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`Terms_And_Condition_Business_Doc.md`](./Terms_And_Condition_Business_Doc.md)
dekho. Yeh doc GST-depth methodology follow karta hai — har claim exact file:line se trace
kiya gaya hai; jahan code se pura confirm nahi ho paaya, wahan **[INFERRED — confirm with
team]** likha hai ya Open Questions mein daala hai.

**Repos checked**: `users-api-go-production` (read — koi T&C-specific controller/model nahi
mila), `service-api-go-production` (write — poora feature yahin hai),
`user-temp-consumers-production` (consumers/crons — koi T&C-specific worker ya cron nahi
mila). Iska matlab yeh **write-only, single-repo feature hai** — is depth-pass mein bhi yeh
confirm hua, purane doc ka finding sahi tha.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write (controller) | write | [`UserTncAcceptanceController.go.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTncAcceptanceController.go.go) — function `UserTncAcceptanceController` |
| Write (model/DB) | write | [`UserTncAcceptanceModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTncAcceptanceModel.go) — function `UpsertTncAcceptance` |
| Mandatory-field validation | write | `MandatoryParamsCheckTncAccp` — [`UserUtilsMandatory.go:200-231`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go) |
| Length/type validation map | write | `UserTncMap` — [`UsersValidationMaps.go:29-35`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go), used inside `LengthAndTypeValidations_v3` via [`UsersValidationMaps.go:2077-2098`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |
| Route registration | write | [`router.go:165`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) and again at [`router.go:333`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) — identical route registered twice in two separate route-groups (pattern also seen for other siblings on the same lines, e.g. `/user/trustseal`, `/rating_usefulness` — **[INFERRED — confirm with team]** this is a versioned/duplicate router-group setup, not a T&C-specific quirk) |
| Queue-name resolution | write | `PushToQueue` / `serviceToQueueMap` / `exchangeSet` — [`rabbitmq.go:20-99`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go) |
| Endpoint registry entry | write | `/wservce/users/tncacceptance` listed in `EndPoints` — [`Validapi.go:224`](../../users-api-go-production/pkg/components/Validapi.go) (note: this repo-level file lives under `users-api-go-production`, referenced from the write repo's request-validation middleware) |

**Read side — actively searched, none found.** Grepped `users-api-go-production/internal`
case-insensitively for `tnc`/`Tnc`/`TERMS`/`terms_cond` — zero controller or model hits.
Grepped all three repos for the table name `IIL_TERMS_COND_ACCEPTANCE` — it appears **only**
in `UserTncAcceptanceModel.go` and in doc/reference files, nowhere else. There is no
`GetTncAcceptance`-style read endpoint anywhere in this codebase.

**Consumer side — actively searched, none found.** Grepped `user-temp-consumers-production`
for `tnc`/`TNC`/`terms_cond`/`SESSION_UPDATE` — the only `SESSION_UPDATE` hit in that repo is
in [`USER_PRIVACYSETTING_ALERTPG.go:244-246`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_ALERTPG.go),
which is a **publisher** to `SESSION_UPDATE` for a completely different feature (Privacy
Setting), not a consumer of it. No file in that repo consumes the `SESSION_UPDATE` queue.

**Filename note**: the controller's filename has a literal double `.go.go` extension in the
repo as-found — almost certainly an accidental duplicate-extension from a rename. Harmless to
Go's build (the file still ends in `.go`), but worth a cleanup PR.

---

## 2. Routes

| Method | Path | serviceName (Kibana/NewRelic transaction name) | Repo | Controller |
|---|---|---|---|---|
| POST | `/user/tncacceptance` (registered twice, identical, in two route-groups — [`router.go:165`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go), [`router.go:333`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go)) | `USER_TERMS_AND_CONDITIONS` | write | `UserTncAcceptanceController` |

---

## 3. Data Model — Table

> **Verification note**: table/column names come from the Go SQL string built in
> `UpsertTncAcceptance` — not cross-checked against a live pgAdmin schema. Confirm before any
> migration work.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `IIL_TERMS_COND_ACCEPTANCE` | meshpg (`config.GetPGDbConnection("meshpg")`) | Append-only per-acceptance compliance log | `fk_glusr_usr_id`, `IIL_TERMS_COND_IP`, `IIL_TERMS_COND_IP_country`, `IIL_TERMS_COND_COUNTRY_ISO`, `IIL_TERMS_COND_USER_AGENT`, `IIL_TERMS_COND_DATE` (app-generated `YYYYMMDD` string via `time.Now().Format("20060102")`, not a DB timestamp function) — [`UserTncAcceptanceModel.go:22,33-45`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTncAcceptanceModel.go) |

Only one table touched by this feature, in one physical DB, by one controller. No
join/read-back happens anywhere in the write path.

---

## 4. Magic-Value Decode — `STATE: "9"`

The success-path RabbitMQ payload hardcodes `"STATE": "9"`
([`UserTncAcceptanceModel.go:72`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTncAcceptanceModel.go)).

Searched this write repo and the consumer repo for any enum, comment, or constant naming
session/GLUSR "states" 0-9 that could explain `9` — **none found**. `SESSION_UPDATE` has no
in-repo consumer (§1), so there is no downstream code in these three repos that interprets
this value either.

**[INFERRED — confirm with team]**: `STATE="9"` most likely corresponds to some
session/account-status enum owned by whatever system consumes `SESSION_UPDATE` (possibly a
legacy PHP/session-management system outside these three repos) — cannot be conclusively
decoded from code available here. Flagged in Open Questions.

---

## 5. Business Rules & Validation (code se, exhaustive)

1. **Mandatory fields**: `MODID`, `VALIDATION_KEY`, `IP`, `USR_ID`, `USER_AGENT` — if any is
   missing/empty, request fails before reaching the gateway check with
   `"Please enter mandatory field (MODID / VALIDATION_KEY / IP / USER_AGENT)"`.
   [`UserUtilsMandatory.go:200-231`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
   Note the error message text omits `USR_ID` even though it is checked in code — a
   copy-paste gap in the message, not the logic.
2. **Gateway allowlist has 33 entries** — the widest "who-can-call-this" pattern also seen on
   Popup Details/Negative Mcat: `MY`, `GLADMIN`, `TOLLFREE`, `Email Marketing`, `HTVENDOR`,
   `Weberp`, `LEAP`, `IPC-Admin`, `NSD Sales`, `MAPI`, `ENQUIRY`, `FREE-WEBSITE`,
   `M.INDIAMART.COM`, `TRADE`, `BL`, `TENDER`, `PAYNOW`, `CREDIT ALLOCATION`, `OVP Process`,
   `SAMPARK Process`, `TOLLFREE Process`, `VENDOR CITY Pin Correction`, `Notification Server`,
   `search`, `FCP`, `PNS`, `Merp`, `IMOB`, `SELLERMY`, `PAYWIM`, `BUYERS_FEEDBACK`, `FLPNS`,
   `LEAPIN`, `ANDROID`, `IOS`. [`UserTncAcceptanceController.go.go:51`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTncAcceptanceController.go.go)
3. **Gateway failure short-circuits with a fixed timing shape**: on gateway rejection, both
   `VALIDATION_TIME` and `RESPONSE_TIME` are set to the same elapsed value at that point (no
   further processing happens) — cosmetic but visible in Kibana timing breakdowns.
   [`UserTncAcceptanceController.go.go:52-57`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTncAcceptanceController.go.go)
4. **Unusual validation-error leniency**: `LengthAndTypeValidations_v3(inputParams,
   UserModels.UserTncMap)` result is treated as passing (insert proceeds) if it is empty
   **OR** if it contains `"is a wrong parameter."` but does **not** also contain
   `" length exceeded."` or `" should be numeric."` — i.e. an unrecognized/extra field name
   alone does not block the insert; only a recognized field failing length or numeric-type
   checks blocks it. This is more lenient than the pattern seen elsewhere in this KT series
   (e.g. GST's controllers hard-fail on any `LengthAndTypeValidations` non-empty result).
   [`UserTncAcceptanceController.go.go:60`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTncAcceptanceController.go.go)
5. **Dynamic column list is driven by `UserTncMap`'s keys, not the request body** — the code
   iterates `UserTncMap` (5 keys: `USR_ID`, `IP`, `IP_COUNTRY`, `IP_COUNTRY_ISO`,
   `USER_AGENT`), and only includes a column in the INSERT if that key is present in the
   incoming request params. `IIL_TERMS_COND_DATE` is always appended regardless, as an
   app-generated value. Any request field NOT in `UserTncMap` is silently ignored for the
   INSERT (though it may still have triggered the leniency path in rule 4 if unrecognized).
   [`UserTncAcceptanceModel.go:31-45`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTncAcceptanceModel.go)
6. **`USR_ID` type mismatch risk**: `UserTncMap["USR_ID"]` declares `"type": "number"`
   ([`UsersValidationMaps.go:30`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go)), yet the insert loop does
   `params[key].(string)` — a hard type-assert to string
   ([`UserTncAcceptanceModel.go:38`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTncAcceptanceModel.go)). If `USR_ID` ever arrived as a
   JSON number rather than a numeric string, this assertion would panic; the panic is caught
   by `defer utils.Recover_Panic(ctx,"USER_TERMS_AND_CONDITIONS")` at the top of the
   controller ([`UserTncAcceptanceController.go.go:18`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTncAcceptanceController.go.go)), so it fails safely rather than
   crashing the process, but the request would still fail. There's a separate explicit
   numeric-string check for `USR_ID` in `UserTncMap`'s validation call
   ([`UsersValidationMaps.go:2079-2085`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go)) which would catch a non-numeric string, but not a
   raw JSON number type — **[INFERRED — confirm with team]** whether `FetchRequestParams`/
   `FormatInputParams` normalizes all incoming values to strings before this point (if so,
   this risk is moot).
7. **Custom string-escaping via `formatinput()`** — uses `strconv.QuoteToASCII` then strips
   the surrounding quotes, layered **on top of** (not instead of) parameterized `$1`/`$2`
   placeholders already used in the query. Redundant but not harmful; a maintenance-clarity
   note, not a security gap. Returns `nil` for an empty string (so an empty field becomes SQL
   `NULL`, not an empty-string literal).
   [`UserTncAcceptanceModel.go:94-103`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTncAcceptanceModel.go)
8. **On INSERT success, exactly one RabbitMQ event fires** — `SERVICENAME=TNC_ACCEPTANCE`,
   `STATE="9"` (§4), `GLUSR_ID`, `TIMESTAMP`. No conditional suppression logic (no DEV-mode
   skip, no per-value skip) — unlike several other write controllers in this domain (e.g. GST
   email suppression) this publish always fires on success.
   [`UserTncAcceptanceModel.go:70-84`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTncAcceptanceModel.go)
9. **Detailed error-diagnostics captured on DB failure**: `getErrorLine()` uses
   `runtime.Caller(1)` to capture file/line of the error and puts it directly into
   `extraParamsForOutput["Error"]`, which flows into both the Kibana log AND is checked for
   presence in the controller (though the raw Response body itself does not directly expose
   `Error` — it goes into `kibanaLogResponse["Errors"]` only, which is logged, not returned to
   the caller). [`UserTncAcceptanceModel.go:58-67`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTncAcceptanceModel.go),
   [`UserTncAcceptanceController.go.go:148-150`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTncAcceptanceController.go.go)
10. **No idempotency/dedup check** — nothing in the code checks whether an identical
    acceptance record already exists for this `USR_ID` before inserting; every call is a
    fresh append. This is a deliberate design (§ Business Doc, "insert-only, no
    overwrite"), not a bug, but worth stating explicitly since it means repeated client
    retries (e.g. a double-tap on a UI button) will create multiple rows for the same
    logical acceptance event.

---

## 6. RabbitMQ

| Queue / `SERVICENAME` | Publisher | Exchange | Consumer | Purpose |
|---|---|---|---|---|
| `SERVICENAME=TNC_ACCEPTANCE` → resolves to queue `SESSION_UPDATE` via `serviceToQueueMap` | `UserTncAcceptanceModel.go` (`UpsertTncAcceptance`), only on insert success | **None** — `TNC_ACCEPTANCE` is present in `serviceToQueueMap` but absent from `exchangeSet` ([`rabbitmq.go:20-91`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go)), so `RabbitEnqueue` sends with `exchange=""` — a direct-to-queue publish via the default exchange, not routed through `USER.topic` like most other domain events | **Not found in `user-temp-consumers-production`** (§1) | Presumably updates the supplier's session/status somewhere downstream; consumer is outside these 3 repos or not traceable from code |

This is the **only** queue interaction in this feature — one publish, no fan-out, no retry
queue, no fail-queue.

## 7. Kafka

**Not used.** Grepped for Kafka client usage (`InitializeKafka`, `sub_topic`,
`consumer_group`) scoped to any T&C-related file — zero hits. This feature is 100%
RabbitMQ (well, technically a single HTTP call to the RabbitMQ-publishing microservice via
`CallPubApi`, §6).

## 8. Redis

**Not used.** Grepped `RedisGet`/`RedisSet`/`redis` case-insensitively across
`UserTncAcceptanceController.go.go` and `UserTncAcceptanceModel.go` — zero hits. No caching
layer anywhere in this write-only path (there being no read endpoint, there is nothing to
cache).

---

## 9. End-to-End Technical Flow

```
Supplier (via any of ~33 allowed internal-apps/channels)
    │
    ▼
[API — write]  POST /user/tncacceptance   serviceName=USER_TERMS_AND_CONDITIONS
                {USR_ID, MODID, VALIDATION_KEY, IP, IP_COUNTRY, IP_COUNTRY_ISO,
                 USER_AGENT}
    │  UserTncAcceptanceController.go.go
    │  1. utils.InputJsonCheck() — basic JSON sanity
    │  2. MandatoryParamsCheckTncAccp() — MODID/VALIDATION_KEY/IP/USR_ID/USER_AGENT present?
    │  3. utils.Gateway_v1(...) — caller in the ~33-entry allowlist?
    │  4. LengthAndTypeValidations_v3(inputParams, UserTncMap) — lenient on
    │     unrecognized-param-name-only errors (§5, rule 4)
    ▼
UpsertTncAcceptance()
    │  builds column list dynamically from UserTncMap ∩ request keys
    │  formatinput() escapes each string value (QuoteToASCII, redundant over $N placeholders)
    ▼
[DB — meshpg]  INSERT INTO IIL_TERMS_COND_ACCEPTANCE (...)
    │
    ├─ on failure → output="INSERT FAILED INTO MESH PG", getErrorLine() captures file:line
    │               → response Code=500, Status=Failed (no RabbitMQ publish)
    │
    └─ on success → output="INSERT SUCCESS INTO MESH PG"
                     ▼
                [RabbitMQ — direct-to-queue, no exchange]
                     SERVICENAME=TNC_ACCEPTANCE → queue SESSION_UPDATE
                     {STATE:"9", GLUSR_ID, TIMESTAMP}
                     ▼
                Response Code=200, Status=Success
```

---

## 10. Flow-wise DB & Table Usage

Only one flow exists in this feature — there is no read path and no consumer to add
additional flows.

### Flow — T&C Acceptance Submit (single DB round-trip)

| # | DB (physical) | Table | Operation | Kya hota hai, aur kyun |
|---|---|---|---|---|
| 1 | meshpg (`config.GetPGDbConnection("meshpg")`) | `IIL_TERMS_COND_ACCEPTANCE` | INSERT | Ek naya append-only compliance-log row likha jaata hai — supplier ID, IP, IP-country, country-ISO, user-agent, aur app-generated date. Koi prior SELECT nahi hota (no dedup/idempotency check, §5 rule 10), koi UPDATE nahi hota (insert-only design) |

**Total DB round-trips per event: exactly 1.** This is the simplest write flow documented in
this KT series so far — single table, single physical DB, single operation, no lock-checks,
no cross-table fan-out, no parallel goroutines. The only non-DB step after the insert is the
single `SESSION_UPDATE` RabbitMQ publish (§6), which is itself an HTTP call to a
RabbitMQ-publishing microservice (`CallPubApi`), not a second DB write.

---

## 11. Optimization Scope — DB Response-Time Contribution

### Low impact / good practice already present

1. **Single-insert, no sequential round-trips, no N+1 pattern** — this is about as
   response-time-efficient as a write endpoint can get; there is effectively nothing to
   optimize on the DB side. [`UserTncAcceptanceModel.go:14-91`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTncAcceptanceModel.go)
2. **Single async fan-out (one RabbitMQ publish), not chained/sequential** — no risk of this
   endpoint's response time growing due to downstream fan-out complexity, unlike domains with
   multi-queue publish chains (e.g. GST's `USER_GST_LAST_MODIFIED`).
3. **1-second query timeout is tight and explicit** (`utils.ExecuteQuery(meshpgconn, query,
   input_params, 1*time.Second)` — [`UserTncAcceptanceModel.go:51`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTncAcceptanceModel.go)) — this bounds worst-case
   latency for the one DB call this endpoint makes, a good defensive practice already in
   place.

### Design-scope note (not performance)

4. **`formatinput()`'s custom escaping layered on top of parameterized queries** (§5, rule 7)
   — not a performance concern (negligible CPU cost for short strings), but a
   maintenance-clarity one: a future reader might assume it is the sole injection-defense and
   be confused about why it coexists with `$N` placeholders.
5. **Direct-to-queue publish (no exchange) is an outlier vs. the rest of the domain** — most
   `SERVICENAME`s in `serviceToQueueMap` also have an `exchangeSet` entry routing through
   `USER.topic`; `TNC_ACCEPTANCE` does not. Not a performance issue per se, but worth
   understanding if this queue is ever migrated/renamed — its routing behaves differently
   from the domain's more common topic-exchange pattern. [`rabbitmq.go:36,64-90`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go)

There is no High or Medium impact optimization item here — this is the leanest flow reviewed
in this KT series; the response-time floor is essentially one DB round-trip plus one
HTTP-to-RabbitMQ-microservice call, both of which are already minimal and already timeout-bounded.

---

## 12. Cron Inventory

Checked `service-api-go-production/crons/` in full (`autogenerate_json_cron.go`,
`cron_tracker.go`, `gst_tact_veri_cron.go`, `mcatcrontest.go`, `rating_suspect_cron.go`,
`Supp_verify_log_cron.go`) and grepped for `tnc`/`terms`/`TERMS_COND` across that directory
and across `user-temp-consumers-production`. **No cron touches T&C acceptance in any way.**
This feature has zero scheduled/batch component — it is purely a synchronous, on-demand
write triggered by the API call.

---

## 13. Edge Cases & Gotchas (technical POV)

1. **Lenient validation-forgiveness for "wrong parameter" errors** (§5, rule 4) — a caller
   could send an unrecognized extra field-name and the insert still succeeds, unlike most
   other write controllers in this codebase where any non-empty validation result blocks the
   write. If this leniency was meant only for genuinely-extra/ignorable fields, confirm it
   doesn't accidentally let a *typo'd* known field name (e.g. `USER_AGNET`) silently get
   dropped from the INSERT rather than flagged as an error to the caller.
2. **`SESSION_UPDATE` consumer not found in any of the 3 repos in scope** (§1, §6) — same
   "queue with no traceable in-repo consumer" pattern seen elsewhere in this KT series (e.g.
   Social Contacts had a missing *producer*; here it's a missing *consumer*). Downstream
   effect of a T&C acceptance on "session/status" cannot be verified from code alone.
3. **`STATE="9"` is an unexplained magic-value** (§4) — no enum/comment anywhere in these 3
   repos describing what session-states exist or what `9` represents specifically for a T&C
   acceptance event (vs. whatever other events might also publish to `SESSION_UPDATE` with
   different `STATE` values).
4. **No idempotency/dedup check** (§5, rule 10) — a double-submit (double-tap, retry-on-timeout
   from a flaky client) creates two rows for the same logical event. This is likely
   intentional per the "append-only audit trail" design, but worth confirming with the team
   that duplicate rows are an acceptable/expected artifact rather than something downstream
   consumers need to de-duplicate themselves.
5. **`USR_ID` type-assertion risk** (§5, rule 6) — a raw JSON-number `USR_ID` (rather than a
   numeric string) would panic inside `UpsertTncAcceptance`'s insert loop; caught safely by
   the top-level `Recover_Panic`, but worth confirming upstream `FetchRequestParams`/
   `FormatInputParams` normalization guarantees this never happens in practice.
6. **Filename typo** (`UserTncAcceptanceController.go.go`) — cosmetic, but worth a cleanup PR
   at some point; also makes this file harder to find via naive `*.go` tooling that assumes a
   single extension.
7. **Route registered twice, identically** (§1, §2) — `router.go:165` and `router.go:333`
   both register `POST /user/tncacceptance` → `UserTncAcceptanceController`. Not a functional
   bug (Gin will just use one route-group or the other depending on setup), but redundant and
   worth understanding why the router file has two near-duplicate route-registration blocks
   before touching either one.

---

## 14. Open Questions

1. What does `STATE="9"` represent in the `SESSION_UPDATE` message, and are there other
   `STATE` values used by other publishers to the same queue?
2. Which system actually consumes the `SESSION_UPDATE` queue, and what does it do with a
   T&C-acceptance event specifically?
3. Is the validation-leniency for "wrong parameter" errors (§5, rule 4) intentional design,
   or an oversight carried over from a shared validation helper not originally written with
   T&C in mind?
4. Why is `/user/tncacceptance` registered twice in `router.go` (lines 165 and 333) — are
   these two genuinely separate route-groups (e.g. different auth/middleware chains), or is
   one a leftover duplicate?
5. Does `FetchRequestParams`/`FormatInputParams` guarantee all values (including `USR_ID`)
   arrive as strings before `UpsertTncAcceptance` runs, eliminating the type-assertion panic
   risk in §5 rule 6?
6. Live DB schema verification — this doc reflects only what the Go SQL string in
   `UpsertTncAcceptance` implies about `IIL_TERMS_COND_ACCEPTANCE`'s columns; no pgAdmin
   cross-check was done.

---

## See also

- [`Terms_And_Condition_Business_Doc.md`](./Terms_And_Condition_Business_Doc.md) — product perspective
- [`../utils_programming_guide.md`](../utils_programming_guide.md) — shared `PushToQueue`/`PubAPI` helpers referenced in this doc
- [`../service_api_write_reference.md`](../service_api_write_reference.md) — one-line summary of this controller alongside every other write controller in the domain

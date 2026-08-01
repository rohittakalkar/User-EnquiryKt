# Enrich Enquiry — Technical Doc (Code-Level Deep Dive)

Yeh doc "Enrich Enquiry" feature ka **technical implementation** cover karta hai — routes, DB
tables, queries, RabbitMQ, consumer, sab kuch code se verify karke. Business/product perspective
ke liye [`Enrich_Enquiry_Business_Doc.md`](./Enrich_Enquiry_Business_Doc.md) dekho — dono docs
same feature cover karte hain, bas alag audience ke liye.

**Repos**: `enq-services-scm` (write API), `enq-consumers` (downstream RabbitMQ consumer).

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file:line diya gaya
hai har jagah). Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — team se confirm karo]** likh diya hai, guess nahi kiya.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Controller (bind, validate, panic-recover, response shaping, 203→200 remap) | enq-services-scm | [`enrichEnquiry.go`](../../internal/api/controllers/enrichEnquiry.go) (110 lines) |
| Route registration | enq-services-scm | [`router.go:81-84`](../../internal/api/router/router.go) |
| Core model — InputParam struct, ValidateStruct, receiver lookup, subject generation, SP call, queue push | enq-services-scm | [`enrichEnquiryModel.go`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go) (709 lines) |
| Downstream consumer — dual-DB write (IMBL + ENQ) from the enrich queue | enq-consumers | [`enq_post_fenq_enrich.go`](../../../enq-consumers/src/Enquiry/enq_post_fenq_enrich.go) (queue name passed as CLI arg at runtime; message published to `ENQUIRY_FENQ_ENRICH`) |

**Note on file naming**: `enq_post_fenq_enrich.go` (this feature's consumer) is a **completely
separate binary/file** from `enq_post_fenq.go` (the FENQ waiting-enquiry bot documented in
[Approve-Reject-Finish Enquiry KT](../Approve-Reject-Finish%20Enquiry%20KT/Approve_Reject_Finish_Enquiry_Technical_Doc.md)).
They share the "fenq" substring in their names and both touch FENQ-related data, but:
- `enq_post_fenq.go` consumes queue `ENQUIRY_FENQ`, writes `DIR_QUERY_WAITING`/`DIR_QUERY_BOUNCED`,
  and makes **synchronous internal HTTP calls to `ApproveEnquiry`/`RejectEnquiry`**
  ([`enq_post_fenq.go:268,284`](../../../enq-consumers/src/Enquiry/enq_post_fenq.go) per the
  Approve-Reject-Finish doc).
- `enq_post_fenq_enrich.go` (this doc) consumes queue `ENQUIRY_FENQ_ENRICH`, calls two stored
  procedures directly (`sp_enrich_fenq` on an "IMBL" Postgres connection, `sp_enrich_enquiry_v1`
  on the "ENQ" Postgres connection) — **no HTTP call back into `enq-services-scm` was found in
  this file.**

No file in `enq-consumers/src/Enquiry` other than `enq_post_fenq_enrich.go` consumes
`ENQUIRY_FENQ_ENRICH` — confirmed via full-directory grep for `enrich` (case-insensitive); the
other hits (`enq_preferred_channel.go:648`, `enq_pns_lms.go:271`, `enq_attachment.go:448-512`) use
the word "enrich(ment)" generically for unrelated LMS/Kafka enrichment (topic
`LMS_LEADS_ENRICHMENT`) and are **not** part of this feature.

---

## 2. Routes (confirmed from router.go)

| Method | Path | Controller |
|---|---|---|
| GET | `/enquiry/enrichEnquiry/` | `controllers.EnrichEnquiry` |
| GET | `/enquiry/enrichEnquiry` | `controllers.EnrichEnquiry` |
| POST | `/enquiry/enrichEnquiry/` | `controllers.EnrichEnquiry` |
| POST | `/enquiry/enrichEnquiry` | `controllers.EnrichEnquiry` |

[`router.go:81-84`](../../internal/api/router/router.go)

**Middleware position**: all four routes are registered **before** `r.Use(middleware.Validation)`
at [`router.go:92`](../../internal/api/router/router.go) — same pattern already noted in Save
Enquiry / Approve-Reject-Finish docs for their routes. `EnrichEnquiry` does **not** go through
`middleware.Validation`; it does its own binding/validation inline in the controller
([`enrichEnquiry.go:42-79`](../../internal/api/controllers/enrichEnquiry.go)).

Who calls this endpoint externally is **not visible from this repo** —
**[INFERRED — confirm with team which upstream caller (web/app form, admin panel, or an
automation) hits `/enrichEnquiry`]**. It is *not* called by `enq_post_fenq_enrich.go` (that
consumer writes to Postgres directly, it does not call this HTTP endpoint) — see section 1.

---

## 3. Data Model — Tables

> **Verification note**: yeh sab table/column/stored-procedure names Go code ke andar embedded
> SQL strings se liye gaye hain (har row ke saamne file path diya hai). Live DB schema ya stored
> procedure internals se cross-verify **nahi** kiya gaya hai — kisi migration/integration se pehle
> woh zaroor karo.

| Table / SP | Physical DB (as configured) | Purpose | Key columns / evidence |
|---|---|---|---|
| `DIR_QUERY_MAP` | write-Postgres (`db.GetDBConn("write")`) | Receiver (`QUERY_RCV_GLUSR_USR_ID`) lookup by `QUERY_ID`, used to shard-resolve the child table for subject generation | [`enrichEnquiryModel.go:242,257`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go) |
| `DIR_QUERY_<shard>` (e.g. `DIR_QUERY_01`) | write-Postgres | Read `R_NAME`, `R_ORGANIZATION`, `DIR_QUERY_MODREF_NAME`, `S_COUNTRY` for subject-line generation, when `queryDestination == 1` | [`enrichEnquiryModel.go:294-308`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go) — shard suffix computed via `helper.GetLastTwoDigits(receiverId)` |
| `DIR_QUERY_WAITING` | write-Postgres | Same subject-lookup read, when `queryDestination == 2` | [`enrichEnquiryModel.go:303`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go) |
| `DIR_QUERY_BOUNCED` | write-Postgres | Same subject-lookup read, when `queryDestination == 3` | [`enrichEnquiryModel.go:305`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go) |
| (write via stored procedure `sp_enrich_enquiry_v6`) — underlying table(s) not directly visible in Go code | write-Postgres (`db.GetDBConn("write")`) | Primary synchronous enrichment write — 26 positional params (enquiry_id, queryDestination, order value, usage, purchase period, purpose, frequency, geo fields, destination port, payment/shipment mode, estimate qty, need_quotation, currency, sender name/email, description, sender city/state, subject, lat/long, query_ref_text) | [`enrichEnquiryModel.go:389-416`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go) — **actual target table(s) inside the SP are [INFERRED — not visible from Go, confirm with DB/DBA team]** |
| (write via stored procedure `sp_enrich_fenq`) — "IMBL" side | Postgres, `imbldb` connection ([`enq_post_fenq_enrich.go:68`](../../../enq-consumers/src/Enquiry/enq_post_fenq_enrich.go)) | Consumer-side conditional enrichment write (19 params, no `subject`/`queryDestination`) | [`enq_post_fenq_enrich.go:345-389`](../../../enq-consumers/src/Enquiry/enq_post_fenq_enrich.go) — target table(s) **[INFERRED]** |
| (write via stored procedure `sp_enrich_enquiry_v1`) — "ENQ" side | Postgres, `enqdb_write` connection ([`enq_post_fenq_enrich.go:75`](../../../enq-consumers/src/Enquiry/enq_post_fenq_enrich.go)) | Consumer-side unconditional enrichment write (23 params, includes `queryDestination`/`subject`) | [`enq_post_fenq_enrich.go:391-439`](../../../enq-consumers/src/Enquiry/enq_post_fenq_enrich.go) — target table(s) **[INFERRED]** |

**Two different DBs, two different SP versions**: the write API uses `sp_enrich_enquiry_v6`
(26 params) synchronously; the consumer later independently calls `sp_enrich_enquiry_v1` (23
params, on `enqdb_write`) and `sp_enrich_fenq` (19 params, on `imbldb`) from the same queued
payload. The version-number mismatch (`v6` vs `v1`) between the synchronous API write and the
async consumer write is **not explained in code** — **[INFERRED — confirm with team whether this
is intentional (different SP scope) or a version drift that should be reconciled]**.

---

## 4. QueryDestination — reused, not redefined

`EnrichEnquiry` consumes the same `queryDestination` values (`1` = normal/`DIR_QUERY_<shard>`,
`2` = waiting/`DIR_QUERY_WAITING`, `3` = bounced/`DIR_QUERY_BOUNCED`) that Save Enquiry defines and
populates — see
[`../Save Enquiry KT/Save_Enquiry_Technical_Doc.md`](../Save%20Enquiry%20KT/Save_Enquiry_Technical_Doc.md#4-querydestination-codes--decode-this-magic-value)
section 4 for the authoritative table. This doc does not redefine those codes; it only uses them
to pick a child table for subject-line lookup ([`enrichEnquiryModel.go:294-306`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go)).

---

## 5. Business Rules & Validation (code se exhaustive list)

1. **`enquiry_id` must be numeric, non-empty, and non-negative**, else `"Invalid Enrich Enquiry
   detail"` is returned before any DB/queue work — [`enrichEnquiryModel.go:444-449`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go) (`ParameterValidation`).
2. **`Description` and `Subject` are URL-query-unescaped** on the way in — [`enrichEnquiryModel.go:450-451`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go).
3. **`S_lat`/`S_long` are both blanked out unless both are numeric** — [`enrichEnquiryModel.go:452-455`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go).
4. **`ValidateStruct` reflection pass** truncates every string field per its `validate:"truncate:N"`
   tag (e.g. `Description` 4000, `req_usage` 2000, `ApprxOrderValue` 200, `estimate_qty` 50,
   `sendername` 80, `senderemail` 60, `destination_port` 100) and validates every
   `validate:"numeric:N"` field for numeric-ness and max digit length — [`enrichEnquiryModel.go:461-589`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go).
5. **UTF-garbage cleanup** (`helper.UtfValidationNew`) runs on every string field during the same
   reflection pass, setting a `utf8_flag` that is later logged as HTTP-like code `"209"` instead
   of the real response code — [`enrichEnquiryModel.go:476-478,500-502`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go), consumed in [`enrichEnquiry.go:87-91`](../../internal/api/controllers/enrichEnquiry.go).
6. **`FenqEnrich` defaults to `0`** if the caller sends an empty value — [`enrichEnquiryModel.go:100-102`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go).
7. **Subject line only auto-generates when both `SenderName` and `EnquiryID` are non-empty** —
   [`enrichEnquiryModel.go:142-147`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go) (`prepareSubject` call gated).
8. **Subject-line wording branches on country/product/sender-name combination** — export wording
   when `senderCountry` is set and not `"INDIA"`/`"IN"`, else domestic wording, else a generic
   "Business Enquiry from ..." fallback — [`enrichEnquiryModel.go:329-365`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go).
9. **Country name is normalized for 3 hardcoded cases** (`united states of america`→`USA`,
   `united kingdom`→`UK`, `united arab emirates`→`UAE`), else passed through unchanged —
   [`enrichEnquiryModel.go:341-350`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go).
10. **DB timeout is environment-dependent**: `1000s` on `local`, `10s` on `dev`/`stg`, `2s`
    otherwise, for both the receiver lookup and subject lookup — [`enrichEnquiryModel.go:246-252,285-291`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go); the `updateDirQuery` timeout is `1000s` on `local`, `2s` for everything else (no separate `dev`/`stg` branch) — [`enrichEnquiryModel.go:381-386`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go).
11. **Duplicate-key SP errors map to code `203`**, which the controller remaps to HTTP `200` with
    error text `"Invalid Enrich Enquiry detail"` — [`enrichEnquiryModel.go:426-428`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go), [`enrichEnquiry.go:100-103`](../../internal/api/controllers/enrichEnquiry.go).
12. **Timeout/deadline-exceeded SP errors map to code `504`** ("Timeout occurred") —
    [`enrichEnquiryModel.go:429-432`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go).
13. **Any other SP error maps to `503`** ("SQL_Statement_Error") — [`enrichEnquiryModel.go:150-151,429`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go).
14. **Queue push only happens after a successful `updateDirQuery` call** — [`enrichEnquiryModel.go:148-159`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go); on SP error the function returns early and `pushIntoEnrichQueue` is never called.
15. **Queue-push failure falls back to a local file backup** (`Email_Open_read_status_rabbit_failure`
    logger file — file name appears reused/generic, not enrich-specific) — [`enrichEnquiryModel.go:182-195`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go). No cron was found in this repo that replays this specific backup file for Enrich Enquiry (see section 13, Cron Inventory).
16. **Panic anywhere in the controller returns HTTP `503`** with `"some error occured"` — [`enrichEnquiry.go:29-40`](../../internal/api/controllers/enrichEnquiry.go).
17. **Consumer-side IMBL write (`sp_enrich_fenq`) is gated**: only runs when `QUEUE_FLAG == "0"`
    **and** (`queryDestination` is `"1"` or `"2"`) **and** `fenq_enrich == 0` — [`enq_post_fenq_enrich.go:204-219`](../../../enq-consumers/src/Enquiry/enq_post_fenq_enrich.go).
18. **Consumer-side ENQ write (`sp_enrich_enquiry_v1`) is unconditional** — always runs regardless
    of the IMBL gate outcome — [`enq_post_fenq_enrich.go:221-228`](../../../enq-consumers/src/Enquiry/enq_post_fenq_enrich.go).
19. **Consumer defaults `fenq_enrich` to `"236"`** (not `"0"`) when the field is missing from the
    message, which would **skip** the IMBL write if it ever hits that default path — [`enq_post_fenq_enrich.go:339-341`](../../../enq-consumers/src/Enquiry/enq_post_fenq_enrich.go). This differs from the write API's own default of `0` ([`enrichEnquiryModel.go:100-102`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go)) — **[INFERRED discrepancy — confirm with team whether `236` is an intentional "not-yet-enriched-but-skip" sentinel or a copy-paste artifact]**.
20. **Consumer message processing always ACKs** (`d.Ack(false)`) regardless of whether either DB
    write succeeded — errors are logged to Kibana but the message is never requeued —
    [`enq_post_fenq_enrich.go:207-229`](../../../enq-consumers/src/Enquiry/enq_post_fenq_enrich.go). A failed DB write here is **silently dropped** from a retry perspective.
21. **Consumer runs up to 5 concurrent goroutines** per process (`gorclock.WaitLow(5)`) — [`enq_post_fenq_enrich.go:127-129`](../../../enq-consumers/src/Enquiry/enq_post_fenq_enrich.go).

---

## 6. RabbitMQ — Queues Touched By Enrich Enquiry

| Queue | Published by | Consumed by | Payload |
|---|---|---|---|
| `ENQUIRY_FENQ_ENRICH` | `pushIntoEnrichQueue` in `enrichEnquiryModel.go` ([:163-209](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go)), via `pubAPIV2.Insert_rabbitmq`, `SERVICENAME="Enquiry_service"` | [`enq_post_fenq_enrich.go`](../../../enq-consumers/src/Enquiry/enq_post_fenq_enrich.go) (queue name passed as a CLI/deploy-config argument — confirmed by name match and by the consumer's own `serviceName = "ENQUIRY_ENRICH"` / Kibana log tag `"ENQ_FENQ_ENRICH_WORKER"`) | The full `InputParam` struct marshaled to JSON, plus `UNIQUE_LOGGING_ID` and `SERVICENAME` injected — [`enrichEnquiryModel.go:167-172`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go) |

On publish failure (`status_code == "203"` from `Insert_rabbitmq`), the payload is instead written
to a local backup file via `persistence.CreateLoggerFile("Email_Open_read_status_rabbit_failure")`
— [`enrichEnquiryModel.go:186-194`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go).
No consumer of a replay-cron for this specific file was found in this repo's `pkg/crons` — see
section 13.

**No other queue is published to or consumed from by this feature.** `enq_post_fenq_enrich.go`
does not publish to any RabbitMQ queue itself (no `channel.Publish`/`Insert_rabbitmq` calls found
in the file) — it only consumes.

---

## 7. Kafka

**No Kafka usage found for this feature.** Grepped `enq-services-scm` and
`enq-consumers/src/Enquiry/enq_post_fenq_enrich.go` for `kafka` (case-insensitive) — no hits in
either the write-API model/controller or the consumer file. (Kafka topic `LMS_LEADS_ENRICHMENT`
exists elsewhere in `enq_attachment.go:512`, but that is a different feature's — Enquiry
Attachment's — LMS-lead enrichment, unrelated to this feature's `ENQUIRY_FENQ_ENRICH` queue.)

---

## 8. Redis

**No Redis usage found for this feature.** Grepped `enq-services-scm` for `redis`
(case-insensitive) with no hits inside the enrich-enquiry files; the only repo-wide hit
(`EnquiryMailer/main.go:590-591`) is an unrelated cron using a local variable literally named
`redisData` (a plain in-memory `map[string]string`, not an actual Redis client call) — confirmed
by inspection, not a real Redis dependency.

---

## 9. End-to-End Technical Flows

### Flow A — Happy path: successful enrichment

```
[CLIENT] POST /enquiry/enrichEnquiry  {enquiry_id, apprx_order_value, ...}
   │
   ▼
[enrichEnquiry.go] ShouldBindBodyWith → ParameterValidation → ValidateStruct (truncate/numeric/UTF)
   │  (all pass)
   ▼
[enrichEnquiryModel.go] EnrichEnquiryModel()
   │  1. LogRawData() — file-based raw-request backup
   │  2. db.GetDBConn("write")
   │  3. getReceiverId()        SELECT DIR_QUERY_MAP  (best-effort, errors swallowed)
   │  4. prepareSubject()       SELECT DIR_QUERY_<shard> | DIR_QUERY_WAITING | DIR_QUERY_BOUNCED
   │                            (only if SenderName + EnquiryID present)
   │  5. updateDirQuery()       select sp_enrich_enquiry_v6(... 26 params ...)   [SYNCHRONOUS WRITE]
   │  6. pushIntoEnrichQueue()  Insert_rabbitmq → ENQUIRY_FENQ_ENRICH             [ASYNC FAN-OUT]
   ▼
[enrichEnquiry.go] Response.Code == 200 → c.JSON(200, Response)

[CONSUME — separate process/pod] enq_post_fenq_enrich.go on ENQUIRY_FENQ_ENRICH
   │  dataclean() — fills any missing map keys with "" (fenq_enrich defaults to "236" if absent)
   │  if QUEUE_FLAG=="0" AND queryDestination in {1,2} AND fenq_enrich==0:
   │     updateFenqIMBL()        select sp_enrich_fenq(... 19 params ...)   on imbldb
   │  updateEnrichDetailsENQ()   select sp_enrich_enquiry_v1(... 23 params ...) on enqdb_write  [ALWAYS]
   │  d.Ack(false)   (always acked, success or failure)
   ▼
[Kibana log] ENQ_FENQ_ENRICH_WORKER — SUCCESS/FAILURE per DB write
```
[`enrichEnquiry.go`](../../internal/api/controllers/enrichEnquiry.go), [`enrichEnquiryModel.go`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go), [`enq_post_fenq_enrich.go`](../../../enq-consumers/src/Enquiry/enq_post_fenq_enrich.go)

### Flow B — Invalid `enquiry_id`

```
[CLIENT] POST /enquiry/enrichEnquiry  {enquiry_id: "", ...}
   │
   ▼
[enrichEnquiryModel.go] ParameterValidation() → "Invalid Enrich Enquiry detail"
   ▼
[enrichEnquiry.go] c.JSON(200, {error:"Invalid Enrich Enquiry detail", CODE:nil})
   (no DB write, no queue push)
```
[`enrichEnquiryModel.go:444-449`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go)

### Flow C — Duplicate-key conflict on `sp_enrich_enquiry_v6`

```
[updateDirQuery] conn.ExecContext(sp_enrich_enquiry_v6, ...) → error containing
   "duplicate key value violates unique constraint"
   │
   ▼
return err, 203
   │
   ▼
[EnrichEnquiryModel] returns CreateResponse(203, err.Error(), -1, "", "", "")
   │
   ▼
[enrichEnquiry.go:100-103] code==203 → code=200, Response.Error = "Invalid Enrich Enquiry detail"
   (queue push never reached — function returned before pushIntoEnrichQueue)
```
[`enrichEnquiryModel.go:148-157,422-428`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go), [`enrichEnquiry.go:99-104`](../../internal/api/controllers/enrichEnquiry.go)

### Flow D — RabbitMQ publish fails, file-backup fallback

```
[updateDirQuery] succeeds (200)
   ▼
[pushIntoEnrichQueue] Insert_rabbitmq(...) → status_code == "203"
   │
   ▼
CreateLoggerFile("Email_Open_read_status_rabbit_failure") → LogTheResponse(...)
   (data augmented with queue_name/r_key/curl_status before writing to file)
   ▼
status_str = "Failed to push into queue, pushed into the File to be processed via TD Agent"
   ▼
[EnrichEnquiryModel] returns CreateResponse(200, "", 1, "", "", status_str)
   (caller still gets a 200 success — the DB write already happened)
```
[`enrichEnquiryModel.go:182-195`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go). No cron in `pkg/crons` was found that reads this specific backup file back for replay — see Open Questions.

### Flow E — Panic mid-request

```
[EnrichEnquiry controller] recover() catches panic anywhere downstream
   ▼
Response = CreateResponse(503, "some error occured", "-1", "", "", "")
   ▼
CentralizedKibanaLoggingEnquiry() logs stack trace
   ▼
c.JSON(503, Response)
```
[`enrichEnquiry.go:29-40`](../../internal/api/controllers/enrichEnquiry.go)

---

## 10. Flow-wise DB & Table Usage

### Flow A — Happy path (write API side)

| # | DB | Table / SP | Operation | Purpose |
|---|---|---|---|---|
| 1 | write-Postgres | `DIR_QUERY_MAP` | SELECT | Receiver GLUSR ID lookup |
| 2 | write-Postgres | `DIR_QUERY_<shard>` / `DIR_QUERY_WAITING` / `DIR_QUERY_BOUNCED` | SELECT | Subject-line source fields |
| 3 | write-Postgres | `sp_enrich_enquiry_v6` (function call) | UPDATE (via SP) | Primary synchronous enrichment write |

### Flow A (continued) — Consumer side (async, separate process)

| # | DB | Table / SP | Operation | Purpose |
|---|---|---|---|---|
| 4 | `imbldb` | `sp_enrich_fenq` (function call) | UPDATE (via SP) | Conditional (gated) IMBL-side enrichment mirror |
| 5 | `enqdb_write` | `sp_enrich_enquiry_v1` (function call) | UPDATE (via SP) | Unconditional ENQ-side enrichment mirror |

### Flow B — Invalid input

No DB calls (validation fails before `EnrichEnquiryModel` reaches any query).

### Flow C — Duplicate key

Only step 3 (`sp_enrich_enquiry_v6`) runs; it errors out. Steps 4-5 never fire (queue push
never happens).

### Flow D — Queue-push failure

Steps 1-3 run normally; step 4-5 (consumer) never fire because the message never reaches the
queue — a local file is written instead.

---

## 11. Optimization Scope

### High-impact

- **Two sequential blocking DB round-trips before the main write** (`getReceiverId` then
  `prepareSubject`, each its own `SELECT` with its own context/timeout) run serially, not in
  parallel, adding latency to every request that has a `SenderName` — [`enrichEnquiryModel.go:140-147`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go). Could be parallelized or the subject-lookup fields folded into the receiver lookup if schema allows.
- **Consumer opens two live Postgres connections at process start** (`imbldb`, `enqdb_write`) and
  always executes on `enqdb_write` even when nothing changed on the IMBL side — no change-detection
  before the unconditional `sp_enrich_enquiry_v1` call — [`enq_post_fenq_enrich.go:221-228`](../../../enq-consumers/src/Enquiry/enq_post_fenq_enrich.go). If callers frequently enrich with no-op data, this is redundant write load on `enqdb_write`.

### Medium-impact

- **`updateDirQuery`'s timeout branching has no separate `dev`/`stg` case** (only `local` vs.
  everything else, both non-`local` collapsing to `2s`) — inconsistent with `getReceiverId`/
  `prepareSubject` which do have a 3-way `local`/`dev`+`stg`/other split — [`enrichEnquiryModel.go:246-252,285-291`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go) vs [`enrichEnquiryModel.go:381-386`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go). Worth reconciling so prod timeout intent is uniform across the model.
- **Reused, generically-named backup logger file** `"Email_Open_read_status_rabbit_failure"` for
  this feature's queue-push failures — [`enrichEnquiryModel.go:187`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go) — makes it hard to distinguish Enrich Enquiry's backup entries from whatever other feature also writes to that same file name. Consider a dedicated file name.

### Low-impact / good-practice already present

- Reflection-based `ValidateStruct`/`validateStructFields` centralizes truncate/numeric/UTF rules
  declaratively via struct tags — consistent, low-maintenance pattern already used elsewhere in
  this codebase (matches Save Enquiry's approach).
- `ResponseTime`/`newrelic.Segment` instrumentation on every DB/queue step gives per-request timing
  visibility (`ResTime.GetReceiverIdTime`, `ResTime.SelectDIR`, `ResTime.UpdateDirQueryRes`,
  `ResTime.EnrichEnqueTime`) — [`enrichEnquiryModel.go:82-92`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go).

---

## 12. Cron Inventory

Checked [`pkg/crons/`](../../pkg/crons/) for anything enrich-related: no cron file references
`enrich`, `ENQUIRY_FENQ_ENRICH`, or the `Email_Open_read_status_rabbit_failure` backup file
(confirmed via glob for `*enrich*` — no matches, and grep of `enrichEnquiry` across the repo — no
hits in `pkg/crons`). **No dedicated replay cron exists for this feature's file-backup fallback**
(Flow D) in this repo — unlike Save Enquiry, which has `QueryParse`/`EnquiryParseManual` crons for
its own backup replay. **[Confirm with team whether Enrich Enquiry's backup file is ever
replayed, by what process, or if it is currently a dead-end on RabbitMQ failure]**.

---

## 13. Edge Cases & Gotchas (technical POV)

1. **`getReceiverId` and `prepareSubject` errors are swallowed** (logged, then function returns
   an empty string) — a receiver-lookup DB error does not fail the request; it silently results in
   no auto-generated subject — [`enrichEnquiryModel.go:261-273,315-322`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go).
2. **`prepareSubject`'s SQL table name is built via string concatenation** (`"DIR_QUERY_" +
   dirTableId`) rather than a fixed identifier list — [`enrichEnquiryModel.go:296-301`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go). The shard suffix comes from `helper.GetLastTwoDigits` on the receiver ID, not user input directly, but it's worth flagging as a non-parameterized identifier pattern.
3. **Response payload strips almost everything before returning to caller** —
   `Response.QueryID`, `DbTime`, `EnqStatusCode`, `EnqueStatus` are all explicitly blanked in the
   controller after logging — [`enrichEnquiry.go:93-98`](../../internal/api/controllers/enrichEnquiry.go) — so the client-visible response is minimal (`status`, `success`, `error`) even though the model computes richer internal timing data.
4. **`Response.Code` is set to `nil` before every `c.JSON` call** — [`enrichEnquiry.go:36,54,76,104`](../../internal/api/controllers/enrichEnquiry.go) — meaning the actual HTTP status code carries the signal, not a body field, consistent with the `omitempty` tag on `Response.Code`.
5. **Consumer's `fenq_enrich` missing-value default (`"236"`) differs from the write-API's own
   default (`0`)** — see Business Rule 19. A message that omits `fenq_enrich` entirely will skip
   the IMBL write under the consumer's default, but would have defaulted to `0` (which *does*
   allow the IMBL write) had the write API been the one filling it in. Since the write API always
   sets `FenqEnrich = 0` before marshaling ([`enrichEnquiryModel.go:100-102`](../../internal/pkg/models/enrichEnquiry/enrichEnquiryModel.go)), the field should never actually be absent from a queue message the write API produced — so this discrepancy would only bite if some other, undiscovered producer publishes directly to `ENQUIRY_FENQ_ENRICH` without setting `fenq_enrich`. No such alternate producer was found in this review.
6. **Consumer always ACKs regardless of DB write outcome** (Business Rule 20) — a transient DB
   error on either `sp_enrich_fenq` or `sp_enrich_enquiry_v1` results in permanent data loss for
   that message (no DLQ/requeue observed) — [`enq_post_fenq_enrich.go:207-229`](../../../enq-consumers/src/Enquiry/enq_post_fenq_enrich.go).
7. **Two different SP major-versions used for the "same" ENQ-side write** (`sp_enrich_enquiry_v6`
   synchronously in the write API vs. `sp_enrich_enquiry_v1` in the consumer) — see section 3
   note. Field/behavior parity between these two SPs is **not verifiable from Go code**.

---

## 14. Open Questions

1. What upstream system(s) actually call `POST /enquiry/enrichEnquiry`? Not traceable from either
   `enq-services-scm` or `enq-consumers`.
2. What are the actual underlying tables written by `sp_enrich_enquiry_v6`, `sp_enrich_enquiry_v1`,
   and `sp_enrich_fenq`? Only visible as opaque function calls in Go.
3. Is the `sp_enrich_enquiry_v6` (write API) vs `sp_enrich_enquiry_v1` (consumer) version gap
   intentional, or should the consumer be updated to call `v6` as well?
4. Is there any replay mechanism for the `Email_Open_read_status_rabbit_failure` backup file when
   this feature's RabbitMQ push fails? None found in `pkg/crons`.
5. What is the business meaning of the `fenq_enrich` flag and the sentinel default `"236"` used by
   the consumer when the field is absent from the message? [`enq_post_fenq_enrich.go:339-341`](../../../enq-consumers/src/Enquiry/enq_post_fenq_enrich.go).
6. Why does the consumer maintain two separate databases (`imbldb` "IMBL" and `enqdb_write` "ENQ")
   with overlapping enrichment data — is one legacy/being deprecated?
7. Is there a dead-letter/retry mechanism at the RabbitMQ infra level for `ENQUIRY_FENQ_ENRICH`
   that compensates for the consumer's always-ACK behavior (edge case 6)? Not visible from
   application code.

---

## See also

- [`Enrich_Enquiry_Business_Doc.md`](./Enrich_Enquiry_Business_Doc.md) — business/product perspective
- [`../Save Enquiry KT/Save_Enquiry_Technical_Doc.md`](../Save%20Enquiry%20KT/Save_Enquiry_Technical_Doc.md) — defines the `queryDestination` codes this feature reuses for subject-lookup table selection
- [`../Approve-Reject-Finish Enquiry KT/Approve_Reject_Finish_Enquiry_Technical_Doc.md`](../Approve-Reject-Finish%20Enquiry%20KT/Approve_Reject_Finish_Enquiry_Technical_Doc.md) — documents `enq_post_fenq.go`, a **different consumer** from this feature's `enq_post_fenq_enrich.go` despite the similar file name (see section 1 for the distinction)

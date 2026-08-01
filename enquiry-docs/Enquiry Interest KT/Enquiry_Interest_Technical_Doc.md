# Enquiry Interest — Technical Doc (Code-Level Deep Dive)

> Every claim below cites `file:line`. Anything not conclusively confirmable from code is marked
> **[INFERRED — confirm with team]** or listed in Open Questions (section 15).

## 1. Enquiry Interest Kahan-Kahan Hai — File Map

| File | Role |
|---|---|
| [`internal/api/router/router.go:55-58`](../../internal/api/router/router.go) | Route registration |
| [`internal/api/controllers/enquiryInterest.go`](../../internal/api/controllers/enquiryInterest.go) | HTTP handler — bind, validate, delegate to model, respond |
| [`internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go) | Input struct, validation/cleanup helpers, RabbitMQ publish logic |
| [`pkg/pubAPI/rabbitPub.go`](../../pkg/pubAPI/rabbitPub.go) | Generic RabbitMQ publish-over-HTTP helper (`Insert_rabbitmq`), retry + file-fallback |
| [`internal/pkg/persistence/common.go`](../../internal/pkg/persistence/common.go) | `Uniqid`, `ErrorLogging`, `CreateLoggerFile`, `LogTheResponse` — shared logging/persistence utilities |
| [`pkg/helper/helper.go`](../../pkg/helper/helper.go) | `IsInputNumeric` and other generic helpers |
| [`../../../enq-consumers/src/Enquiry/enq_post_intent.go`](../../../enq-consumers/src/Enquiry/enq_post_intent.go) | `enq-consumers` RabbitMQ consumer (`ENQUIRY_INTEREST` service) that persists the queued interest into Postgres |

## 2. Routes (confirmed from router.go)

| Method | Path | Controller | Middleware position |
|---|---|---|---|
| POST | `/enquiry/enquiryInterest/` | `controllers.EnquiryInterest` | Before `middleware.Validation` |
| GET | `/enquiry/enquiryInterest/` | `controllers.EnquiryInterest` | Before `middleware.Validation` |
| POST | `/enquiry/enquiryInterest` | `controllers.EnquiryInterest` | Before `middleware.Validation` |
| GET | `/enquiry/enquiryInterest` | `controllers.EnquiryInterest` | Before `middleware.Validation` |

[`router.go:54-58`](../../internal/api/router/router.go) registers all four variants immediately
after `r := app.Group("/enquiry")`. `r.Use(middleware.Validation)` is attached later at
[`router.go:92`](../../internal/api/router/router.go), so — same pattern already documented for
`CallEnquiry`/`UnidentifiedC2C`/`CallEnquiryUnidentified` in the Call Enquiry Technical Doc — all
four `enquiryInterest` routes run **without** `middleware.Validation`. What that middleware
actually checks was not traced in this review (see Open Questions).

Note the controller only binds JSON body (`c.ShouldBindBodyWith(&inputParam, binding.JSON)`)
[`enquiryInterest.go:38`](../../internal/api/controllers/enquiryInterest.go) — despite `GET`
routes being registered, the handler expects a JSON body regardless of HTTP method; there is no
method-specific branching in the controller.

## 3. Data Model — Tables

**Verification note**: no `.sql`/migration/schema file for `IIL_ENQUIRY_INTEREST` was located in
either repo during this review; the table name below is inferred solely from the stored
procedure name and consumer log strings. Column-to-table mapping is **not independently
confirmed** — treat as **[INFERRED — confirm with team / DBA]**.

| Table (inferred name) | Written by | Evidence |
|---|---|---|
| `IIL_ENQUIRY_INTEREST` (warehouse Postgres, DB alias `imbldwhpg`) | `enq-consumers` `ENQUIRY_INTEREST` worker via `sp_insert_iil_enquiry_interest_v4(...)` | [`enq_post_intent.go:67`](../../../enq-consumers/src/Enquiry/enq_post_intent.go) (`utils.GetPG("imbldwhpg")`), [`enq_post_intent.go:344`](../../../enq-consumers/src/Enquiry/enq_post_intent.go) (query text), log strings `"pg query select for IIL_ENQUIRY_INTEREST..."` at [`enq_post_intent.go:347,355`](../../../enq-consumers/src/Enquiry/enq_post_intent.go) |

No table is written directly by `enq-services-scm` for this feature — the write API only
publishes to RabbitMQ and writes local log files
([`enquiryInterestModel.go:64,207-221`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go)).

Stored-procedure positional parameters (27 total, `$1`..`$27`), in call order
[`enq_post_intent.go:345`](../../../enq-consumers/src/Enquiry/enq_post_intent.go):

`interest_rcv_id, interest_sender_id, interest_modid, interest_modrefid, interest_modrefname,
interest_modreftype, interest_sender_ip, interest_sender_ip_country, interest_sender_country_iso,
interest_current_url, interest_type, mail1(always "null"), mail2(always "null"), cat_id, mcat_id,
interest_usr_login_mode, interest_s_long, interest_s_lat, interest_latlong_accuracy,
interest_query_ref_url, interest_query_ref_text, interest_query_actual_url, interest_date,
interest_glb_city_id, interest_ip_city_id, interest_pref_city_id, in_iil_fenq_status(always 0
via `strconv.Atoi("null")` which errors and returns zero-value)`
[`enq_post_intent.go:311-345`](../../../enq-consumers/src/Enquiry/enq_post_intent.go).

Note `mail1`/`mail2` are hardcoded to `"null"` and never populated from the incoming message —
**dead/unused parameters** as far as this consumer is concerned
[`enq_post_intent.go:313-314`](../../../enq-consumers/src/Enquiry/enq_post_intent.go).
`in_iil_fenq_status` is always `0` because `strconv.Atoi("null")` fails and the error is
discarded, leaving the zero value [`enq_post_intent.go:317`](../../../enq-consumers/src/Enquiry/enq_post_intent.go)
— this looks like unintentional dead code (always sends `0` regardless of any real status), flag
in Open Questions.

## 4. Decode This Magic Value — status/type codes found

- **`interest_type`**: `interface{}` in the API payload, converted with `strconv.Atoi` in the
  consumer [`enq_post_intent.go:329`](../../../enq-consumers/src/Enquiry/enq_post_intent.go). No
  enum or mapping table found in either repo for what values mean. **[INFERRED — confirm with
  team]**.
- **`interest_modreftype`**: same treatment, no mapping found
  [`enq_post_intent.go:324`](../../../enq-consumers/src/Enquiry/enq_post_intent.go).
  **[INFERRED — confirm with team]**.
- **`interest_usr_login_mode`**: same, no mapping found
  [`enq_post_intent.go:332`](../../../enq-consumers/src/Enquiry/enq_post_intent.go).
  **[INFERRED — confirm with team]**.
- **`status_code == "203"`** in the RabbitMQ publish helper is used as a magic string meaning
  "publish failed after retry, fall back to file" — not an HTTP status, just an internal
  convention shared with other `enq-services-scm` publishers
  [`enquiryInterestModel.go:122`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go),
  [`rabbitPub.go:46`](../../pkg/pubAPI/rabbitPub.go).
- **`Code`/`Status` fields on the API response**: `Response.Code` and `Response.Status` are
  populated internally with the RabbitMQ push status/time but explicitly blanked (`= ""`) right
  before every `c.JSON(200, ...)` call
  [`enquiryInterest.go:47,63,75-76`](../../internal/api/controllers/enquiryInterest.go) — so the
  caller-visible response essentially never carries these two fields populated, despite the
  struct defining `json:"query_time,omitempty"` / `json:"code,omitempty"`
  [`enquiryInterestModel.go:19,21`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go).
  With `omitempty` and an explicit blank, these keys are simply absent from the JSON response.

## 5. Business Rules & Validation (code se exhaustive list)

1. **JSON body binding**: `c.ShouldBindBodyWith(&inputParam, binding.JSON)`; on error, HTTP 200
   with `success=-1`/error text is still returned (not a 4xx)
   [`enquiryInterest.go:38-49`](../../internal/api/controllers/enquiryInterest.go).
2. **`interest_sender_glusr_id` must be non-empty**, else `"Sender ID is missing"`
   [`enquiryInterestModel.go:197-200`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go).
3. **`interest_query_ref_text == "||"` is rejected** as `"Fake request"`
   [`enquiryInterestModel.go:201-204`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go).
4. **`interest_sender_glusr_id` must be numeric** per `helper.IsInputNumeric`, else `"Invalid
   sender ID"` [`enquiryInterest.go:53-55`](../../internal/api/controllers/enquiryInterest.go).
5. **Lat/long sanitation**: if either `InterestSLat` or `InterestSLong` matches regex
   `[a-z|=]` (case-sensitive lowercase-letter-or-pipe-or-equals check), BOTH are wiped to empty
   string [`enquiryInterestModel.go:156-165`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go).
6. **`InterestCatId`/`InterestMcatId` default to `"0"`** when the incoming value is an empty
   string (only string-typed empty check; since these are `interface{}` fields, a JSON `null`
   would not match `== ""` — worth confirming intent)
   [`enquiryInterestModel.go:166-171`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go).
7. **Field-length truncation** (all silent, no rejection): `InterestModid` → 8 chars,
   `InterestCurrentUrl`/`InterestQueryRefUrl`/`InterestQueryActualUrl` → 500 chars,
   `InterestQueryRefText` → 250 chars, `InterestProductName` → 200 chars
   [`enquiryInterestModel.go:174-192`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go).
8. **Raw request is file-logged** via `LogRawData` before the RabbitMQ push, only on the
   validation-success path (called from inside `EnquiryInterestModel`)
   [`enquiryInterestModel.go:64,207-221`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go),
   [`enquiryInterest.go:69`](../../internal/api/controllers/enquiryInterest.go).
9. **Panic recovery**: any panic in the controller is caught, logged to Kibana with code `"203"`,
   and converted to a generic failure JSON response (HTTP 200)
   [`enquiryInterest.go:27-36`](../../internal/api/controllers/enquiryInterest.go).
10. **On the consumer side**, every reference/context field defaults to the literal string
    `"null"` if absent or empty before DB insert — done individually per field in `dataclean`
    [`enq_post_intent.go:204-284`](../../../enq-consumers/src/Enquiry/enq_post_intent.go).
11. **Consumer rejects (acks without insert) if `interest_sender_glusr_id` resolves to `"null"`**
    post-cleanup [`enq_post_intent.go:176-179`](../../../enq-consumers/src/Enquiry/enq_post_intent.go).
    Note this is a redundant safety net since the API already enforces non-empty sender ID at
    step 2 — this check catches cases where the message reached the queue outside the normal API
    path, or the field failed `strconv.Atoi` in some other way.
12. **On DB error**, the consumer `Nack`s the message with `requeue=false`
    [`enq_post_intent.go:197`](../../../enq-consumers/src/Enquiry/enq_post_intent.go) — a failed
    insert is **not retried**, it is dropped after one attempt (no dead-letter queue configuration
    visible in this file — **[INFERRED — confirm with team]**, not traced beyond this file).
13. **Duplicate detection is delegated entirely to the stored procedure** — a negative returned
    ID is logged as `"Duplicate record found"` but still `Ack`'d as success
    [`enq_post_intent.go:352-358,190-191`](../../../enq-consumers/src/Enquiry/enq_post_intent.go).
14. **`mail1`/`mail2` stored-proc params are hardcoded `"null"` and never derived from the
    message** — effectively dead parameters
    [`enq_post_intent.go:313-314`](../../../enq-consumers/src/Enquiry/enq_post_intent.go).
15. **`in_iil_fenq_status` is always computed as `0`** via a discarded-error `strconv.Atoi("null")`
    call — looks unintentional, flagged in Open Questions
    [`enq_post_intent.go:317`](../../../enq-consumers/src/Enquiry/enq_post_intent.go).

## 6. RabbitMQ — Enquiry Interest Mein Use Hone Waala Queue

| Queue | Exchange | Published by | Consumed by | Evidence |
|---|---|---|---|---|
| `enquiry.post.intent` | `ENQUIRY.topic` | `enq-services-scm`, `PushIntoIntentQueue` | `enq-consumers`, `ENQUIRY_INTEREST` worker (`enq_post_intent.go`, queue name supplied as `os.Args[2]` at process start) | Publish side: [`enquiryInterestModel.go:73-75`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go). Consume side: queue name is a runtime CLI arg, not hardcoded, at [`enq_post_intent.go:52,84`](../../../enq-consumers/src/Enquiry/enq_post_intent.go); the consumer's own log/service tags (`ENQUIRY_INTEREST`, `ENQUIRY_INTENT_WORKER`, `ENQUIRY_POST_INTENT`) at [`enq_post_intent.go:27`](../../../enq-consumers/src/Enquiry/enq_post_intent.go), [`enq_post_intent.go:70`](../../../enq-consumers/src/Enquiry/enq_post_intent.go), and matching file-fallback tag `"queue_name"] = "ENQUIRY_POST_INTENT"` on the publish side at [`enquiryInterestModel.go:124`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go) make it highly likely — but not 100% code-provable, since the queue name is a deploy-time CLI argument, not a compile-time constant — that this file is indeed the consumer for this queue. **Confirm the deployed CLI args for this worker process with the ops/deploy team to close the loop fully.** |

Publish mechanism is not a native AMQP client call from `enq-services-scm` — it goes through an
internal HTTP relay (`pubapi.Insert_rabbitmq` → `POST {rmq-publish-url}/rmq/publish`)
[`rabbitPub.go:17-56`](../../pkg/pubAPI/rabbitPub.go), with environment-specific URLs
[`enquiryInterestModel.go:76-83`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go).
One retry is attempted on failure; if still unsuccessful, the payload is written to a local
`.log` file via `publishThroughTD` as a last-resort durability mechanism
[`rabbitPub.go:36-51,105-151`](../../pkg/pubAPI/rabbitPub.go).

The consumer (`enq_post_intent.go`) connects natively via `streadway/amqp` and `broker.ConnectCluster()`
[`enq_post_intent.go:17,81`](../../../enq-consumers/src/Enquiry/enq_post_intent.go), with manual
ack/nack (`autoAck=false`) [`enq_post_intent.go:87`](../../../enq-consumers/src/Enquiry/enq_post_intent.go),
max 5 concurrent goroutines via `gorclock.WaitLow(5)` [`enq_post_intent.go:120`](../../../enq-consumers/src/Enquiry/enq_post_intent.go).

## 7. Kafka — Explicit Check

**No Kafka usage found** for this feature. Case-insensitive grep for `kafka` across
`enq-services-scm` returned no hits in code (only in unrelated sibling KT docs), and a targeted
grep on `enq_post_intent.go` returned zero matches. This feature is RabbitMQ-only.

## 8. Redis — Explicit Check

**No Redis usage found** for this feature specifically. Case-insensitive grep for `redis` across
`enq-services-scm` matched only `crons/QASapprovedMailer/main.go`, `crons/EnquiryMailer/main.go`,
and `finishEnquiryModel.go` — none of which are part of the Enquiry Interest code path (they
belong to mailer crons and the Finish Enquiry flow, already covered in the Approve-Reject-Finish
KT docs). A targeted grep on `enq_post_intent.go` also returned zero matches. Enquiry Interest
does not touch Redis.

## 9. End-to-End Technical Flows

### Flow A — Happy path

```
[API] POST /enquiry/enquiryInterest  {interest_sender_glusr_id, interest_cat_id, interest_type, ...}
    │  ShouldBindBodyWith(JSON) → OK
    │  CheckSenderIdQueryRefText() → OK, IsInputNumeric(sender_id) → OK
    │  CheckSLatSLong() → sanitize lat/long
    │  TruncateValidationChecks() → truncate/default fields
    ▼
[MODEL] EnquiryInterestModel()
    │  LogRawData() → local log file (raw backup)
    │  PushIntoIntentQueue()
    │      json.Marshal → flatten to map[string]string
    │      pubapi.Insert_rabbitmq(data, "enquiry.post.intent", "ENQUIRY.topic", url, "EnquiryInterest")
    │          POST {rmq-publish-url}/rmq/publish  → status "Success"
    ▼
[API RESPONSE] HTTP 200 {Interest_id:1, success:1, error:"", (Status/EnqueTime/Code blanked)}

--- asynchronously, separate process ---

[CONSUMER enq-consumers/ENQUIRY_INTEREST] channel.Consume("enquiry.post.intent")
    │  json.Unmarshal(body) → message map
    │  dataclean() → default missing fields to "null"
    │  sender_id != "null"? → yes
    │  insertIntoIMBL() → sp_insert_iil_enquiry_interest_v4(...) on imbldwhpg Postgres
    │  success → d.Ack(false)
```
[`enquiryInterest.go`](../../internal/api/controllers/enquiryInterest.go),
[`enquiryInterestModel.go`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go),
[`enq_post_intent.go`](../../../enq-consumers/src/Enquiry/enq_post_intent.go)

### Flow B — Validation failure (missing/non-numeric sender ID, or fake ref text)

```
[API] POST /enquiry/enquiryInterest {interest_sender_glusr_id: ""}
    │  CheckSenderIdQueryRefText() → "Sender ID is missing"
    ▼
[API RESPONSE] HTTP 200 {success:-1, error:"Sender ID is missing"}  — never reaches RabbitMQ
```
[`enquiryInterest.go:51-66`](../../internal/api/controllers/enquiryInterest.go)

### Flow C — RabbitMQ publish failure → file fallback

```
[MODEL] PushIntoIntentQueue()
    │  Insert_rabbitmq() → rabbit_enqueue()
    │      getCurlResponseCB() attempt 1 → status != "Success"
    │      getCurlResponseCB() attempt 2 (retry) → status != "Success"
    │      status_code = "203"
    │      publishThroughTD(rmq_data, serviceName) → append to local .log file
    ▼
[MODEL] status_code == "203" → CreateLoggerFile("Email_Open_read_status_rabbit_failure")
    │  LogTheResponse(...) → write to that file too (double fallback)
    ▼
[API RESPONSE] HTTP 200 {status: "Failed to push into queue, pushed into the File to be processed via TD Agent"}
```
[`enquiryInterestModel.go:72-136`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go),
[`rabbitPub.go:22-56,105-151`](../../pkg/pubAPI/rabbitPub.go)

### Flow D — Consumer-side DB failure

```
[CONSUMER] insertIntoIMBL() → pgConn.QueryRowContext(...).Scan(&interest_id) → err != nil
    │  LogResponseKibana("FAILURE", ...)
    ▼
[CONSUMER] d.Nack(false, false)  — message dropped, not requeued, no DLQ visible in this file
```
[`enq_post_intent.go:181-198,346-350`](../../../enq-consumers/src/Enquiry/enq_post_intent.go)

## 10. Flow-wise DB & Table Usage

| Flow | DB touched | Table/proc | Notes |
|---|---|---|---|
| A — Happy path | `imbldwhpg` (Postgres, warehouse) | `sp_insert_iil_enquiry_interest_v4` (writes to inferred `IIL_ENQUIRY_INTEREST`) | Only touched by the consumer, asynchronously; write API touches no DB |
| B — Validation failure | None | — | Rejected before any queue/DB interaction |
| C — RabbitMQ failure | None (local filesystem only) | — | Local `.log` file fallback, not a DB |
| D — Consumer DB failure | `imbldwhpg` (attempted, failed) | `sp_insert_iil_enquiry_interest_v4` | Insert attempted but errors; message dropped |

## 11. Optimization Scope

### High-impact
- **No retry/DLQ on consumer DB failure** — a transient Postgres blip permanently drops the
  interest signal (`d.Nack(false, false)`, no requeue)
  [`enq_post_intent.go:197`](../../../enq-consumers/src/Enquiry/enq_post_intent.go). Given this
  is meant to be a durable analytics signal, silent data loss on transient DB errors is a real
  gap.
- **`in_iil_fenq_status` is always `0`** due to a discarded parse error on the literal string
  `"null"` — if this parameter is meant to carry a real status, it's currently dead/broken
  [`enq_post_intent.go:317`](../../../enq-consumers/src/Enquiry/enq_post_intent.go).

### Medium-impact
- **Two-tier file fallback on publish failure is redundant**: both `publishThroughTD` (inside
  `rabbitPub.go`) and the model's own `CreateLoggerFile`/`LogTheResponse` calls write near-duplicate
  fallback data to two different files for the same failure event
  [`rabbitPub.go:105-151`](../../pkg/pubAPI/rabbitPub.go),
  [`enquiryInterestModel.go:122-134`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go)
  — consolidating would simplify the failure path and reduce disk I/O.
- **`mail1`/`mail2` stored-proc parameters are always `"null"`** — either remove them from the
  procedure signature or wire them to real data; currently dead weight in every insert call
  [`enq_post_intent.go:313-314`](../../../enq-consumers/src/Enquiry/enq_post_intent.go).

### Low-impact / good-practice already present
- Manual ack/nack with bounded concurrency (`gorclock.WaitLow(5)`) is a reasonable backpressure
  mechanism already in place [`enq_post_intent.go:118-121`](../../../enq-consumers/src/Enquiry/enq_post_intent.go).
- Panic recovery exists at both the controller and consumer message-processing level, preventing
  a single bad payload from crashing the process
  [`enquiryInterest.go:27-36`](../../internal/api/controllers/enquiryInterest.go),
  [`enq_post_intent.go:146-155`](../../../enq-consumers/src/Enquiry/enq_post_intent.go).

## 12. Cron Inventory

Checked [`pkg/crons/`](../../pkg/crons) for anything related to enquiry interest/intent:
`EnquiryMailer`, `EnquiryParseManual`, `QASapprovedMailer`, `QueryParse`, `db/databasecron.go`,
`errorLogging/cronlogging.go`, `setModIds.go`. **None reference `interest` or `intent`** — no
cron exists in `enq-services-scm` for this feature (e.g. no replay job for the RabbitMQ-failure
fallback log files found in section 9, Flow C). This is consistent with the file-fallback data
potentially requiring manual/external (TD Agent) reprocessing — **[INFERRED — confirm with
team]**.

## 13. Edge Cases & Gotchas (technical POV)

1. `GET` routes are registered for this endpoint but the handler unconditionally does
   `ShouldBindBodyWith(&inputParam, binding.JSON)` — a `GET` request with no body will simply
   bind an empty struct and immediately fail sender-ID validation; there's no dedicated
   query-param path despite `form:"..."` tags being present on every field
   [`enquiryInterestModel.go:25-55`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go).
2. `InterestCatId`/`InterestMcatId` are `interface{}`, and the empty-default check is
   `== ""` — a JSON `null` or numeric `0` would not match this check the same way a missing
   string field would [`enquiryInterestModel.go:166-171`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go).
3. `strconv.Atoi` errors are silently discarded throughout `insertIntoIMBL` (e.g.
   `interest_type`, `cat_id`, `mcat_id`) — any non-numeric string sent through the queue becomes
   `0` rather than causing a visible failure [`enq_post_intent.go:319-341`](../../../enq-consumers/src/Enquiry/enq_post_intent.go).
4. `LogRawData` (raw request backup) only runs for requests that pass validation — invalid
   requests are Kibana-logged but not written to the raw-backup file, so the raw-backup log is
   not a complete audit trail of all incoming requests
   [`enquiryInterestModel.go:64`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go),
   [`enquiryInterest.go:69`](../../internal/api/controllers/enquiryInterest.go).
5. Response's `Code`/`Status`/`EnqueTime` fields are always blanked before being returned to the
   caller (section 4) — any consumer of this API relying on those fields for the actual queue
   status will always see them empty; only `success`/`error`/`Interest_id` are meaningful, and
   `Interest_id` is never actually set by this controller (`CreateResponse` is always called with
   a literal `1` or `-2`, never a real ID) [`enquiryInterestModel.go:68,151-154`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go).

## 14. Open Questions

1. **What does `middleware.Validation` check**, and would it matter if `enquiryInterest` were
   also placed behind it? Not traced in this review's scope (same open question already raised
   in the Call Enquiry Technical Doc for its own routes).
2. **Is `enq_post_intent.go` (`ENQUIRY_INTEREST` worker) actually the deployed consumer for the
   `enquiry.post.intent` queue?** The queue name is passed as a CLI argument at process startup
   (`os.Args[2]`), not hardcoded, so this cannot be proven purely from source — strongly implied
   by matching service/log tags but should be confirmed against the actual deployment/process
   manager configuration.
3. **What does `IIL_ENQUIRY_INTEREST` actually get used for downstream** (analytics,
   personalization, lead scoring)? No schema/consumer-of-the-table code was found in either
   repo during this review.
4. **What are the valid values/meanings of `interest_type`, `interest_modreftype`, and
   `interest_usr_login_mode`?** No enum or mapping exists in either repo.
5. **Is `in_iil_fenq_status` always being `0` intentional**, or is this a bug where a real status
   value should have been threaded through? [`enq_post_intent.go:317`](../../../enq-consumers/src/Enquiry/enq_post_intent.go)
6. **Is there any downstream job that replays the local file-fallback logs** written on RabbitMQ
   publish failure? No such cron was found in `pkg/crons`.
7. **What triggers a caller to hit this endpoint** (which frontend pages/events)? Not traceable
   from backend code alone.
8. **Is dropping a message on consumer DB failure (`Nack(false, false)`, no DLQ visible) an
   accepted risk**, or should this be revisited for durability? [`enq_post_intent.go:197`](../../../enq-consumers/src/Enquiry/enq_post_intent.go)

## See also

- [Enquiry Interest — Business Doc](./Enquiry_Interest_Business_Doc.md)
- [Save Enquiry — Technical Doc](../Save%20Enquiry%20KT/Save_Enquiry_Technical_Doc.md) — documents
  the unrelated `intent` field inspected by `enq_preferred_channel.go` for personalization
  (`leap@indiamart.com` → `finishEnquiry`); confirmed during this review to be a distinct concept
  from this feature's `enquiry.post.intent` queue, sharing only the English word "intent".
- [Call Enquiry — Technical Doc](../Call%20Enquiry%20KT/Call_Enquiry_Technical_Doc.md) — same
  before-`middleware.Validation` routing pattern.

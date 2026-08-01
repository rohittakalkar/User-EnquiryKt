# Enquiry Interest — Business Doc (Product Perspective)

> Note: Yeh doc `GET/POST /enquiry/enquiryInterest` endpoint ko cover karta hai, jo `enq-services-scm`
> write-API mein `controllers.EnquiryInterest` se serve hota hai. Har factual claim ke saath
> file:line citation di gayi hai. Jo cheez code se conclusively confirm nahi ho payi, use
> **[INFERRED — confirm with team]** mark kiya gaya hai ya Open Questions mein daala gaya hai
> (technical doc dekhein).

## 1. Yeh feature hai kya, aur kyun zaroori hai

"Enquiry Interest" ek lightweight, fire-and-forget "signal capture" endpoint hai. Jab koi buyer
kisi supplier/product ke prati "interest" dikhata hai — jaise ki ek listing page dekhna, ek
call-to-action click karna, ya kisi query se related koi activity karna — to frontend/upstream
systems is API ko hit karte hain taaki us interaction ka context (kaun, kis product mein, kis URL
se, kaunsi category, IP/geo, login mode, waqt) capture ho sake.

Controller khud ismein koi synchronous DB likhta nahi hai — yeh sirf input ko validate/clean
karta hai aur ek RabbitMQ queue (`enquiry.post.intent`) mein publish kar deta hai
[`enquiryInterestModel.go:72-75`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go).
Actual database insert ek alag downstream consumer (`enq-consumers`) mein hota hai. Isliye API
response bahut fast hai aur caller ko sirf "queue mein chala gaya ya nahi" ka signal milta hai,
DB-confirmed success/failure nahi.

**Kyun zaroori hai**: Yeh raw behavioral/interest data collect karta hai jo baad mein analytics,
personalization, ya lead-scoring jaise downstream use-cases ke liye IIL_ENQUIRY_INTEREST table
mein persist hota hai (dekhein Technical Doc section 3). Business ke liye yeh ek "signal
pipeline" hai, na ki enquiry lifecycle ka core transactional step (Save/Approve/Reject/Finish
Enquiry se alag hai).

## 2. Kaun-kaun involved hai

- **Buyer** — jiska interest capture ho raha hai (`interest_sender_glusr_id`)
  [`enquiryInterestModel.go:32`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go).
- **Receiver (supplier/seller)** — optional, `interest_rcv_glusr_id`
  [`enquiryInterestModel.go:35`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go).
- **Calling system / frontend** — jo yeh API hit karta hai jab koi interest-worthy interaction
  hoti hai (jaise product page view, listing click). Exact caller(s) code se confirm nahi ho
  paaye — **[INFERRED — confirm with team]**.
- **`enq-services-scm` write API** — validate karta hai, RabbitMQ mein publish karta hai
  [`enquiryInterest.go`](../../internal/api/controllers/enquiryInterest.go).
- **RabbitMQ (`ENQUIRY.topic` exchange, `enquiry.post.intent` queue)** — transport layer
  [`enquiryInterestModel.go:74-75`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go).
- **`enq-consumers` (`ENQUIRY_INTEREST` worker, `enq_post_intent.go`)** — queue consume karke
  actual DB row insert karta hai
  [`enq_post_intent.go:27`](../../../enq-consumers/src/Enquiry/enq_post_intent.go).
- **IMBL / warehouse Postgres DB (`imbldwhpg`)** — final storage via stored procedure
  `sp_insert_iil_enquiry_interest_v4`
  [`enq_post_intent.go:67,344`](../../../enq-consumers/src/Enquiry/enq_post_intent.go).

### Important clarification: "Interest" vs Save Enquiry's "intent" personalization concept

Save Enquiry's Technical Doc documents that `enq_preferred_channel.go` inspects a
personalization payload's `intent` field for a `leap@indiamart.com` value, which triggers
`finishEnquiry`. That is a **completely different, unrelated concept** from this "Enquiry
Interest" feature — it is a personalization-service field name that happens to also use the
English word "intent"/"interest". This document's `enq_post_intent.go` consumer is **not** the
same file/logic as `enq_preferred_channel.go`, and this feature's queue (`enquiry.post.intent`)
is not referenced anywhere in the Save Enquiry personalization-intent code path found earlier.
The word "intent" here is genuinely the queue-name label for this Enquiry Interest signal
(`ENQUIRY_POST_INTENT` in file-fallback logging, `ENQUIRY_INTENT_WORKER` as the consumer's log
tag) [`enquiryInterestModel.go:124`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go),
[`enq_post_intent.go:70`](../../../enq-consumers/src/Enquiry/enq_post_intent.go) — **do not
confuse the two "intent" usages.**

## 3. Interest Type / classification (from code)

The input carries an `interest_type` field (`interface{}`, sent to the queue as-is, converted to
int by the consumer) [`enquiryInterestModel.go:49`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go),
[`enq_post_intent.go:329`](../../../enq-consumers/src/Enquiry/enq_post_intent.go). No enum,
lookup table, or comment in either repo defines what numeric values mean — **[INFERRED — confirm
with team]**, treat as an opaque classification code owned by the caller/business team. Same
applies to `interest_modreftype`, `interest_usr_login_mode`, `interest_modid` — all passed
through without server-side meaning decoded in this code (see Technical Doc section 4).

## 4. Business Flows

### Flow A — Happy path: Interest successfully queued and later persisted

1. Buyer performs an interest-worthy action on the frontend (product view / CTA click / etc. —
   exact trigger **[INFERRED — confirm with team]**).
2. Frontend/caller sends a `POST /enquiry/enquiryInterest` request with buyer ID, product/category
   context, URL context, geo/IP info.
3. `enq-services-scm` validates the sender ID is present and numeric, cleans/truncates fields,
   and pushes the payload onto RabbitMQ (`enquiry.post.intent` queue on `ENQUIRY.topic` exchange).
4. API responds immediately (HTTP 200) with a lightweight success payload — it does **not** wait
   for the DB insert.
5. Asynchronously, the `ENQUIRY_INTEREST` consumer in `enq-consumers` picks up the message,
   cleans nulls, and calls a Postgres stored procedure to insert into the interest-tracking table
   on the `imbldwhpg` warehouse database.

**Business impact**: Buyer-side experience is not slowed by backend persistence; a permanent
record of the interest signal accumulates in the warehouse for downstream analytics/personalization
use, subject to the queue delivering successfully.

### Flow B — Sender ID missing or non-numeric (validation failure)

1. Request arrives without `interest_sender_glusr_id`, or with a non-numeric value.
2. Controller runs `CheckSenderIdQueryRefText` and `IsInputNumeric` checks
   [`enquiryInterest.go:52-56`](../../internal/api/controllers/enquiryInterest.go).
3. Request is rejected before ever reaching RabbitMQ; a validation-failure response is returned
   and logged to Kibana.

**Business impact**: No interest is recorded for anonymous/malformed requests — sender identity
is treated as mandatory for a valid interest signal.

### Flow C — "Fake request" detection via `interest_query_ref_text`

1. If `interest_query_ref_text` is exactly `"||"`, the controller treats this as a fake/garbage
   request [`enquiryInterestModel.go:201-203`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go).
2. Request is rejected with `"Fake request"` error, never queued.

**Business impact**: A specific known bad-data pattern (empty-pipe ref text, likely from a buggy
or bot caller) is filtered out before it pollutes the interest dataset. Exact origin of this
`"||"` pattern is **[INFERRED — confirm with team]**.

### Flow D — RabbitMQ publish fails; fallback to file-based logging

1. Request passes validation; controller attempts to publish to RabbitMQ via `pubapi.Insert_rabbitmq`.
2. The RabbitMQ publish HTTP call fails or returns non-Success status twice (one retry is
   attempted) [`rabbitPub.go:36-51`](../../pkg/pubAPI/rabbitPub.go).
3. Instead of losing the data, the payload is written to a local fail-log file
   (`Email_Open_read_status_rabbit_failure` logger file, and a separate TD-agent fallback file)
   [`enquiryInterestModel.go:122-134`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go),
   [`rabbitPub.go:105-151`](../../pkg/pubAPI/rabbitPub.go).
4. API still responds to the caller with a "failed to push, file-logged" status message.

**Business impact**: Interest signals are not silently dropped on a RabbitMQ outage — they are
captured to disk for later reprocessing (assuming an operational process exists to replay these
files — not confirmed in code, **[INFERRED — confirm with team]**).

### Flow E — Consumer rejects duplicate / malformed messages

1. `ENQUIRY_INTEREST` consumer receives the queued message.
2. If `interest_sender_glusr_id` resolves to `"null"` after cleanup, the message is acknowledged
   (removed from queue) without inserting anything
   [`enq_post_intent.go:176-179`](../../../enq-consumers/src/Enquiry/enq_post_intent.go).
3. If the stored procedure detects a duplicate, it returns a negative interest ID, which the
   consumer logs as "Duplicate record found" but still acknowledges the message as processed
   [`enq_post_intent.go:352-358`](../../../enq-consumers/src/Enquiry/enq_post_intent.go).

**Business impact**: The same interest signal is not double-counted in the warehouse table; the
dedup logic lives in the stored procedure (`sp_insert_iil_enquiry_interest_v4`), not in Go code —
its exact dedup key/window is **[INFERRED — confirm with team]**, listed in Open Questions.

## 5. Business Rules — Plain Language Mein

1. **Sender ID is always mandatory.** Without `interest_sender_glusr_id`, the request is rejected
   outright [`enquiryInterest.go:52-56`](../../internal/api/controllers/enquiryInterest.go).
2. **Sender ID must be numeric.** A non-numeric value triggers `"Invalid sender ID"`
   [`enquiryInterest.go:53-55`](../../internal/api/controllers/enquiryInterest.go).
3. **A specific ref-text value (`"||"`) is treated as a bot/fake signal** and rejected
   [`enquiryInterestModel.go:201-203`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go).
4. **Lat/long values that look like they contain letters or `=` are wiped out** rather than
   stored as garbage geo-data
   [`enquiryInterestModel.go:156-164`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go).
5. **Category/sub-category IDs default to `"0"` when blank**, never left empty
   [`enquiryInterestModel.go:166-171`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go).
6. **Several free-text/URL fields are hard-truncated** to protect downstream storage:
   `interest_modid` to 8 chars, URLs to 500 chars, `interest_query_ref_text` to 250 chars,
   `interest_product_name` to 200 chars
   [`enquiryInterestModel.go:174-192`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go).
7. **API response never exposes queue-internal fields to the caller** — `EnqueTime`, `Status`,
   and `Code` are all blanked out right before the JSON response is sent in both the happy path
   and validation-failure paths [`enquiryInterest.go:47,63,74-76`](../../internal/api/controllers/enquiryInterest.go).
   The net effect is the caller mainly sees `success`/`error`/`Interest_id` (though `Interest_id`
   is never actually populated by this controller — see Open Questions).
8. **Only requests that pass validation get raw-logged to a local per-service log file**
   (`LogRawData`) — this runs inside `EnquiryInterestModel`
   [`enquiryInterestModel.go:64`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go),
   which the controller only invokes **after** sender-ID/fake-request validation succeeds
   [`enquiryInterest.go:69`](../../internal/api/controllers/enquiryInterest.go) — so invalid
   requests are NOT raw-logged via this path, they are only Kibana-logged.
9. **A crash anywhere in the controller is recovered** and converted into a generic
   "Some failure on pushing into queue" response rather than a 500 error
   [`enquiryInterest.go:27-36`](../../internal/api/controllers/enquiryInterest.go).
10. **On the consumer side, all relationship/reference fields default to the string `"null"`**
    when missing, which is how the downstream `NULL`-vs-`0` distinction is preserved for the
    stored procedure [`enq_post_intent.go:204-284`](../../../enq-consumers/src/Enquiry/enq_post_intent.go).
11. **Duplicate interest records are silently deduped** by the stored procedure (returns negative
    ID); the consumer treats this as a successful, ack'd outcome, not a failure
    [`enq_post_intent.go:352-358`](../../../enq-consumers/src/Enquiry/enq_post_intent.go).

## 6. Notifications — Kisko kab pata chalta hai

| Trigger | Notification | Evidence |
|---|---|---|
| Interest queued/persisted | None found — no email/SMS/push code in either the controller, model, or consumer | Absence confirmed by full read of [`enquiryInterest.go`](../../internal/api/controllers/enquiryInterest.go), [`enquiryInterestModel.go`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go), [`enq_post_intent.go`](../../../enq-consumers/src/Enquiry/enq_post_intent.go) |
| RabbitMQ publish failure | Internal Kibana log + file-based fallback log only; no outbound notification to any human/system | [`enquiryInterestModel.go:122-134`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go) |
| Consumer panic | Internal `Debugger`/Kibana log only, commented-out `utils.Mail(...)` call present but disabled | [`enq_post_intent.go:37,153`](../../../enq-consumers/src/Enquiry/enq_post_intent.go) |

This feature is a silent, backend-only signal-capture pipeline — no user-facing or team-facing
notification exists in the reviewed code.

## 7. Edge Cases — Business Perspective

1. Buyer submits interest without a receiver ID (`interest_rcv_glusr_id` blank) — allowed; the
   consumer stores `null` for it [`enq_post_intent.go:206-208`](../../../enq-consumers/src/Enquiry/enq_post_intent.go).
2. Lat/long submitted with letters (e.g. accidental URL-encoded junk) — silently wiped rather
   than rejecting the whole request [`enquiryInterestModel.go:156-164`](../../internal/pkg/models/enquiryInterestModel/enquiryInterestModel.go).
3. Category/mcat IDs blank — defaulted to `"0"`, not `null`, which is a different convention than
   most other reference fields (which default to `"null"` in the consumer) — worth confirming
   this asymmetry is intentional (Open Questions).
4. RabbitMQ down for an extended period — interest data accumulates only in local fail-log files
   per app instance; no evidence of an automated replay job in `pkg/crons`
   (see Technical Doc section — cron inventory found no interest/intent-related cron).
5. Duplicate interest for the same buyer/product — deduped at DB level, not at API level; the API
   itself has no idempotency check and will happily re-queue identical payloads.
6. `interest_type`, `interest_modreftype`, `interest_usr_login_mode` sent as non-numeric strings —
   consumer's `strconv.Atoi` silently defaults to `0` on parse failure (Go zero-value behavior,
   error is discarded) [`enq_post_intent.go:324,329,332`](../../../enq-consumers/src/Enquiry/enq_post_intent.go)
   — a business-meaningful "unknown type" could be silently recorded as type `0`.

## 8. Quick Summary — Ek Line Mein Har Cheez

Enquiry Interest is a fire-and-forget signal-capture API: buyer interactions are validated
lightly, pushed onto RabbitMQ (`enquiry.post.intent`), and asynchronously persisted by the
`ENQUIRY_INTEREST` consumer into a warehouse Postgres table via a stored procedure — it is
unrelated to Save Enquiry's `intent` personalization field despite the shared word.

## See also

- [Enquiry Interest — Technical Doc](./Enquiry_Interest_Technical_Doc.md)
- [Save Enquiry — Business Doc](../Save%20Enquiry%20KT/Save_Enquiry_Business_Doc.md) (for the
  unrelated "intent" personalization concept, clarified in section 2 above)
- [Call Enquiry — Business Doc](../Call%20Enquiry%20KT/Call_Enquiry_Business_Doc.md) (same
  repo/routing pattern, registered before `middleware.Validation` similarly)

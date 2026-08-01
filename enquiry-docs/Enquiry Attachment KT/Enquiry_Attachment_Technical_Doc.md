# Enquiry Attachment — Technical Doc (Code-Level Deep Dive)

Yeh doc "Enquiry Attachment" feature ka **technical implementation** cover karta hai — routes,
DB tables, queries, RabbitMQ, Kafka, downstream consumers, sab kuch code se verify karke.
Business/product perspective ke liye
[`Enquiry_Attachment_Business_Doc.md`](./Enquiry_Attachment_Business_Doc.md) dekho.

**Repos**: `enq-services-scm` (write API — single controller/model), `enq-consumers` (multiple
downstream consumers touch the same `DIR_QUERY_ATTACHMENT` table, but via **separate,
independently-triggered pipelines** — see section 8 for why).

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file:line diya gaya
hai har jagah). Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — team se confirm karo]** likh diya hai, guess nahi kiya.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write-API controller (bind, ParameterValidation, ValidateStruct, response/log) | enq-services-scm | [`enquiryAttachment.go`](../../internal/api/controllers/enquiryAttachment.go) |
| Write-API model — direct synchronous INSERT into `DIR_QUERY_ATTACHMENT` | enq-services-scm | [`enquiryAttachmentModel.go`](../../internal/pkg/models/enquiryAttachment/enquiryAttachmentModel.go) |
| Route registration | enq-services-scm | [`router.go:85`](../../internal/api/router/router.go) |
| **`ENQUIRY_ATTACHMENT_WORKER`** — RabbitMQ consumer that independently re-inserts attachment data into `DIR_QUERY_ATTACHMENT` (own schema, own status column), calls an external image-moderation service, an external permanent-URL/upload service, and conditionally publishes to Kafka topic `LMS_LEADS_ENRICHMENT` | enq-consumers | [`enq_attachment.go`](../../../enq-consumers/src/Enquiry/enq_attachment.go) (813 lines) |
| **Finish-Enquiry-triggered attachment transfer** — reads `dir_query_attachment` back out and forwards it to an external "Buy Leads" HTTP API (`transferAttachment` endpoint) when the FENQ bot processes a waiting enquiry with a resolved Offer/Lead ID | enq-consumers | [`enq_post_fenq.go:1118-1250ish`](../../../enq-consumers/src/Enquiry/enq_post_fenq.go) (`callTransferAttachment`, `getEnquiryAttachment`) |
| Config for the Buy-Leads transfer endpoint | enq-consumers | [`notify.yaml`](../../../enq-consumers/src/Enquiry/notify.yaml) (`transferAttachment` key) |

---

## 2. Routes (confirmed from router.go)

| Method | Path | Controller | Middleware position |
|---|---|---|---|
| POST | `/enquiry/enquiryAttachment` | `controllers.EnquiryAttachment` | **Before** `middleware.Validation` |

[`router.go:85,92`](../../internal/api/router/router.go)

**Middleware position matches the Call Enquiry pattern**: `EnquiryAttachment` is registered at
line 85, **before** `r.Use(middleware.Validation)` at line 92 — same as `CallEnquiry`,
`CallEnquiryUnidentified`, and `UnidentifiedC2C` (documented in
[`Call_Enquiry_Technical_Doc.md`](../Call%20Enquiry%20KT/Call_Enquiry_Technical_Doc.md) section
2). `middleware.Validation` therefore does **not** run for this endpoint either — consistent
with 4 of the 5 endpoints in this domain reviewed so far; only `LinkC2CPNS` sits behind it.
What `middleware.Validation` actually checks remains unconfirmed (open question carried over
from Call Enquiry KT).

---

## 3. Data Model — Tables

> **Verification note**: yeh sab table/column names Go code ke andar embedded SQL strings se
> liye gaye hain (har row ke saamne file path diya hai). Live DB schema se cross-verify **nahi**
> kiya gaya hai — kisi migration/integration se pehle woh zaroor karo.

| Table | Written/read by | Purpose | Key columns (code se) |
|---|---|---|---|
| `DIR_QUERY_ATTACHMENT` | **Write path A** — `enquiryAttachmentModel.go` (direct write-API INSERT) | Buyer-uploaded attachment linked to an enquiry | `FK_DIR_QUERY_ID`, `FK_ATTACH_TYPE_ID`, `DIR_QRY_ATTACH_DOC_PATH`, `DIR_QRY_ATTACH_IMG_ORIG`/`_SMALL`/`_MEDIUM`/`_LARGE` + matching `_WIDTH`/`_HEIGHT` columns for each — [`enquiryAttachmentModel.go:106-158`](../../internal/pkg/models/enquiryAttachment/enquiryAttachmentModel.go) |
| `DIR_QUERY_ATTACHMENT` | **Write path B** — `enq_attachment.go` (`ENQUIRY_ATTACHMENT_WORKER` consumer) | A **separate insert**, with a narrower column set plus a moderation-status column not used by write path A | `FK_DIR_QUERY_ID`, `FK_ATTACH_TYPE_ID`, `DIR_QRY_ATTACH_DOC_PATH`, `DIR_QRY_ATTACH_IMG_ORIG`, `DIR_QRY_ATTACH_ORIG_WIDTH`, `DIR_QRY_ATTACH_ORIG_HEIGHT`, `DIR_QRY_ATTACHEMENT_STATUS` (`RETURNING DIR_QRY_ATTACHMENT_ID`) — [`enq_attachment.go:373-400`](../../../enq-consumers/src/Enquiry/enq_attachment.go) |
| `dir_query_attachment` (lowercase — same physical table) | **Read path** — `enq_post_fenq.go` (`getEnquiryAttachment`) | Reads back all attachment rows for a `query_id` when the FENQ bot resolves a waiting enquiry into an Offer/Lead, to forward to an external Buy-Leads system | `fk_attach_type_id`, `dir_qry_attach_orig_width/height`, `dir_qry_attach_small_width/height`, `dir_qry_attach_medium_width/height`, `dir_qry_attach_large_width/height`, `dir_qry_attach_doc_path`, `dir_qry_attach_img_orig/small/medium/large` — [`enq_post_fenq.go:1282-1413`](../../../enq-consumers/src/Enquiry/enq_post_fenq.go) |
| `dir_query` | Read-only, `enq_attachment.go` | Looks up `fk_glusr_usr_id`, `query_rcv_glusr_usr_id`, `DIR_QUERY_MAIL_SEND` for the incoming message's `query_id`, to decide whether Kafka publish should happen and to attach sender/receiver IDs to the Kafka packet | [`enq_attachment.go:234-238`](../../../enq-consumers/src/Enquiry/enq_attachment.go) |

**Two independent, non-identical INSERT paths into the same table — this is the single most
important structural fact in this doc** (elaborated in section 8).

---

## 4. "Decode This Magic Value" — Codes Traced From The Code

### 4.1 `AttachmentFlag` (`flag` field, `InputParam.AttachmentFlag`)

Accepted by the write-API's `InputParam` (`validate:"numeric:1"`,
[`enquiryAttachmentModel.go:47`](../../internal/pkg/models/enquiryAttachment/enquiryAttachmentModel.go))
but **never read anywhere in `EnquiryAttachmentModel()`'s logic** — it is bound and
validated, then not used in any branch, SQL, or response. **[INFERRED — dead/reserved field,
confirm with team whether it's meant for a not-yet-wired use case, e.g. delete/replace
semantics.]**

### 4.2 `DIR_QRY_ATTACHEMENT_STATUS` (consumer-side only)

Only written by the `ENQUIRY_ATTACHMENT_WORKER` consumer (write path B), **not** by the
write-API's own INSERT (write path A never sets this column at all — see section 8).

| Value | Meaning | Evidence |
|---|---|---|
| `"1"` | Approved (documents always get this by default; images get this when the moderation service approves, or when `ImageOrig` is empty so no check runs) | `insertAllAttachments()` — [`enq_attachment.go:311-333`](../../../enq-consumers/src/Enquiry/enq_attachment.go) |
| `"2"` | Rejected / not-yet-validated fallback (default `imageStatus`, and also the fallback on moderation-API error) | Same function, `imageStatus := "2"` default |

`hasApprovedAttachment` in the consumer treats **anything other than `"2"`** as approved
(`record.Status != "2"`) — [`enq_attachment.go:227-231`](../../../enq-consumers/src/Enquiry/enq_attachment.go),
so `"1"` and any other future non-`"2"` value would both count as approved. **Exact status
enum (is there a `"0"`/pending state?) — [INFERRED, only `"1"`/`"2"` observed in this file].**

### 4.3 `validateAttachment()` external moderation response

Calls a "Leap Buyer Photos Review" HTTP API (`utils.String("validateImage", "")` endpoint)
and maps its JSON `response` field:

| External API `response` (lower-cased) | Local status returned |
|---|---|
| `"approve"` | `"1"` |
| anything else (including request/parse errors) | `"2"` |

[`enq_attachment.go:625-667`](../../../enq-consumers/src/Enquiry/enq_attachment.go)

### 4.4 `insertion_type` in the Kafka packet

Hardcoded to `"1"` for every Kafka publish from `enq_attachment.go`
([`enq_attachment.go:456`](../../../enq-consumers/src/Enquiry/enq_attachment.go)) — this is
the **same `insertion_type` field name** used by `enq_pns_lms.go` in the Call Enquiry /
Save Enquiry pipeline (documented in
[`Call_Enquiry_Technical_Doc.md`](../Call%20Enquiry%20KT/Call_Enquiry_Technical_Doc.md) section
4.4), but these two pipelines are **entirely separate Kafka topics and separate publish
mechanisms** (`ENQ_PNS_DATA` via `enq_pns_lms.go`'s `curlKafka()` vs. `LMS_LEADS_ENRICHMENT`
via `enq_attachment.go`'s own `attachmentcurlKafka()`) — the shared field name does not imply
a shared pipeline; confirmed by reading both files in full.

---

## 5. Business Rules & Validation (code se exhaustive list)

### Write-API (`enq-services-scm`)

1. **`FKQueryID` must be numeric, non-empty, and non-negative.**
   [`enquiryAttachmentModel.go:463`](../../internal/pkg/models/enquiryAttachment/enquiryAttachmentModel.go)
2. **`ModId` must be non-empty** (no allowlist check against `helper.IsModIdValid` like Call
   Enquiry does — just a non-empty check).
   [`enquiryAttachmentModel.go:466`](../../internal/pkg/models/enquiryAttachment/enquiryAttachmentModel.go)
3. **At least one of `image_orig` or `doc_path` must be non-empty** — both empty is rejected.
   [`enquiryAttachmentModel.go:469`](../../internal/pkg/models/enquiryAttachment/enquiryAttachmentModel.go)
4. **Every image entry (in `ImageOrig`) and every document entry must carry a non-empty
   `attach_type_id`** — checked per-element in a loop, first empty one fails the whole request.
   [`enquiryAttachmentModel.go:473-486`](../../internal/pkg/models/enquiryAttachment/enquiryAttachmentModel.go)
5. **A second, reflection-based `ValidateStruct()` pass runs after `ParameterValidation`**
   (only if that first check passed) — truncates strings per `truncate:N` tags (e.g. paths to
   250 chars), and rejects non-numeric/oversized values per `numeric:N` tags (e.g.
   `attach_type_id` must fit 2 digits, `image height/width` must fit 5 digits).
   [`enquiryAttachmentModel.go:208-336`](../../internal/pkg/models/enquiryAttachment/enquiryAttachmentModel.go),
   [`enquiryAttachment.go:53-65`](../../internal/api/controllers/enquiryAttachment.go)
6. **Image paths and document paths are URL-unescaped (`url.QueryUnescape`) before insert** —
   client-submitted paths are expected to be URL-encoded on the wire.
   [`enquiryAttachmentModel.go:116,128`](../../internal/pkg/models/enquiryAttachment/enquiryAttachmentModel.go)
7. **`ImageSmall`/`ImageMedium`/`ImageLarge` are only used up to `len(ImageOrig)` — indexed by
   the same loop counter `i` as `ImageOrig`, with an explicit `i < len(...)` bounds check per
   variant** — meaning if a client sends more small/medium/large images than orig images, the
   extras are **silently dropped** (never inserted), since the outer loop is driven by
   `imageCount := len(inputParam.ImageOrig)`.
   [`enquiryAttachmentModel.go:123-149`](../../internal/pkg/models/enquiryAttachment/enquiryAttachmentModel.go)
8. **A single multi-row `INSERT` statement is built for all documents + all images in one
   request** (one INSERT, dynamically-sized placeholder list) — not one INSERT per row.
   [`enquiryAttachmentModel.go:152-162`](../../internal/pkg/models/enquiryAttachment/enquiryAttachmentModel.go)
9. **2-second DB timeout in non-local environments** (100s locally) — `deadline exceeded`/
   `canceling statement`/`timeout` in the error string maps to HTTP model-code `504`, anything
   else to `500`. No `saveAsJSON`-style backup-file write on failure (unlike Call Enquiry /
   GST's pattern) — a failed insert here is only logged via `persistence.ErrorLogging`, not
   preserved for replay.
   [`enquiryAttachmentModel.go:95-100,160-174`](../../internal/pkg/models/enquiryAttachment/enquiryAttachmentModel.go)
10. **No RabbitMQ or Kafka publish anywhere in the write-API's model file** — confirmed by a
    full read of `enquiryAttachmentModel.go`; the write path is a pure synchronous DB insert,
    nothing else. This is a **material difference from every other endpoint documented in
    Save Enquiry / Call Enquiry / Approve-Reject-Finish KT**, all of which push at least one
    queue message per successful write.

### Consumer (`enq_attachment.go`, `ENQUIRY_ATTACHMENT_WORKER`)

11. **Only the first `ImageOrig` element is sent to the moderation API** (`msg.ImageOrig[0]`)
    — its approve/reject verdict (`imageStatus`) is then applied to **every** image record
    inserted from this message (all of `ImageOrig`, since `ImageSmall`/`Medium`/`Large` are
    explicitly commented out of the insert — see finding in section 9).
    [`enq_attachment.go:322-345,363-366`](../../../enq-consumers/src/Enquiry/enq_attachment.go)
12. **`getPermanentURL` is only called when `imageStatus == "1"` (approved)** — rejected images
    never get a permanent-storage URL fetched.
    [`enq_attachment.go:333-345`](../../../enq-consumers/src/Enquiry/enq_attachment.go)
13. **Kafka publish only happens if `DIR_QUERY_MAIL_SEND` is non-NULL and non-zero for the
    enquiry's `dir_query` row, AND at least one inserted attachment has `Status != "2"`** —
    both conditions are required; either being false causes the message to still be
    `Ack`'d as a successful no-op (no Kafka call, no error).
    [`enq_attachment.go:225-294`](../../../enq-consumers/src/Enquiry/enq_attachment.go)
14. **A `sql.ErrNoRows` on the `dir_query` lookup (i.e. `query_id` doesn't exist) is treated as
    a successful, non-retryable outcome** (`d.Ack(false)`, HTTP-style code `204` in the Kibana
    log) — not an error, not a requeue.
    [`enq_attachment.go:239-245`](../../../enq-consumers/src/Enquiry/enq_attachment.go)
15. **On DB insert failure, JSON unmarshal failure of the moderation/upload responses, or a
    Kafka-publish failure, the message is `Nack`'d with requeue=true** (except unmarshal of the
    incoming message itself, which is `Nack`'d with requeue=**false**, i.e. sent to a fail
    queue) — [`enq_attachment.go:202-289`](../../../enq-consumers/src/Enquiry/enq_attachment.go)
16. **`fetchISQ()` is a stub that always returns an empty list** — the real implementation is
    fully commented out, with an explicit code comment: *"Temporarily return empty ISQ list
    until LMS team can bifurcate ISQ provided from Finish Enquiry API and Enquiry Attachment
    API."* [`enq_attachment.go:591-623`](../../../enq-consumers/src/Enquiry/enq_attachment.go)
    — a known, intentionally-parked gap, not a bug.

### Downstream read (`enq_post_fenq.go`, FENQ bot)

17. **`callTransferAttachment` only runs when the FENQ bot has resolved a non-empty
    `offer_id`** (i.e. only for enquiries that successfully became a purchased Business Lead)
    — [`enq_post_fenq.go:344-347`](../../../enq-consumers/src/Enquiry/enq_post_fenq.go); if
    `getEnquiryAttachment` finds no doc path and no orig-image path for that `query_id`, the
    transfer is skipped entirely (no HTTP call made).
    [`enq_post_fenq.go:1132-1147`](../../../enq-consumers/src/Enquiry/enq_post_fenq.go)
18. **This read query does not filter on any approval/status column** — it reads whatever rows
    exist in `dir_query_attachment` for the `query_id`/`offer_id` pair, regardless of which
    insert path wrote them or what status they carry (write path A never sets a status column
    at all — section 8). **[INFERRED — worth confirming this is intentional, since it means an
    attachment inserted via the write API (which never runs moderation) could be transferred to
    the Buy-Leads system without ever passing through the consumer's approval gate — see Open
    Questions.]**

---

## 6. RabbitMQ — Queues Touched By Enquiry Attachment

| Queue | Publisher | Consumer | Purpose |
|---|---|---|---|
| *(name not resolvable from this repo — passed as `os.Args[2]` at consumer startup, not hardcoded)* | **Not found in `enq-services-scm`** — the write-API's `EnquiryAttachmentModel()` has no `PushToQueue`/`Insert_rabbitmq`/similar call anywhere in the file (confirmed by full read) | `enq_attachment.go` (`ENQUIRY_ATTACHMENT_WORKER`, consumes via `channel.Consume(Queue_name, ...)`) | Consumes an `AttachmentMessage` JSON shape (`UNIQUE_LOGGING_ID`, `query_id`, `modid`, `flag`, `image_orig/small/medium/large`, `docs`, `product_title`, `sender_glid`) — [`enq_attachment.go:38-50`](../../../enq-consumers/src/Enquiry/enq_attachment.go) |

**Publisher not found — this is a genuine gap in this review's scope, not an assumption.** The
write-API (`enquiryAttachmentModel.go`) is the only Enquiry-Attachment-specific write path in
`enq-services-scm`, and it does not publish to any queue. The consumer's `AttachmentMessage`
struct shape (`product_title`, `sender_glid` fields not present anywhere in the write-API's
`InputParam`) suggests its publisher is a **different service entirely**, possibly outside the
`enq-services-scm`/`enq-consumers` repo pair reviewed here.
**[INFERRED — confirm with team which service publishes to the queue `enq_attachment.go`
consumes from; this review found no producer in either reviewed repo.]**

**No evidence of Enquiry Attachment's write-API publishing to `ENQ_DATA`, `ENQUIRY_FENQ`,
`R_MESSAGE_CENTER_BIZFEED_LMS<0-4>`, or any other queue documented in the sibling KT docs** —
confirmed via full read of `enquiryAttachmentModel.go`.

---

## 7. Kafka

**The write-API (`enq-services-scm`) has no Kafka usage** — confirmed via a repo-wide grep for
`kafka`/`Kafka` across `enq-services-scm`; no hits in any `.go` source file (only the KT docs
themselves, which are markdown, mention the word).

**The downstream consumer `enq_attachment.go` does use Kafka, directly (not via RabbitMQ
fan-out)**: it makes a synchronous HTTP `POST` to an internal Kafka-publish gateway
(`http://soa-kafka-pub-api.intermesh.net/kafkapublish` in prod,
`http://dev-kafka-pub-api.intermesh.net/kafkapublish` in dev/local) with a JSON payload
`{rservice, data, topic: "LMS_LEADS_ENRICHMENT"}`, 1-second HTTP client timeout.
[`enq_attachment.go:480-590`](../../../enq-consumers/src/Enquiry/enq_attachment.go)

**Straight answer if asked "does Enquiry Attachment use Kafka":** the write API itself does
not. Its independently-triggered downstream consumer (`ENQUIRY_ATTACHMENT_WORKER`) does,
publishing enrichment data to Kafka topic `LMS_LEADS_ENRICHMENT` — a **different topic** from
the `ENQ_PNS_DATA` topic used by Save Enquiry/Call Enquiry/Approve-Reject-Finish's shared
`enq_pns_lms.go` consumer.

---

## 8. Redis

**No Redis usage found** in `enquiryAttachment.go`, `enquiryAttachmentModel.go`, or
`enq_attachment.go` — confirmed via targeted grep for `redis`/`Redis` (case-insensitive)
across `enq-services-scm` (only hits are the already-documented false-positives in
`finishEnquiryModel.go` and the two mailer crons, per Approve/Reject/Finish Technical Doc
section 10 — none in the attachment files) and across `enq-consumers/src/Enquiry/enq_attachment.go`
specifically (zero hits). All attachment writes/reads hit Postgres directly, no caching layer.

---

## 9. The Two-Insert-Paths Problem — Detailed Analysis

This is the single most important structural finding in this feature, so it gets its own
section rather than being buried in Edge Cases.

**Write path A** (`POST /enquiry/enquiryAttachment` → `enquiryAttachmentModel.go`):
- Synchronous, in the HTTP request/response cycle
- Inserts **all** of `ImageOrig`/`ImageSmall`/`ImageMedium`/`ImageLarge`/`Document`, with full
  width/height columns for every image size variant
- **Never sets `DIR_QRY_ATTACHEMENT_STATUS`** (that column isn't in write path A's `INSERT`
  column list at all — [`enquiryAttachmentModel.go:106-110`](../../internal/pkg/models/enquiryAttachment/enquiryAttachmentModel.go))
- **No moderation, no permanent-URL fetch, no Kafka publish** — a pure "trust the client's
  submitted path" insert
- No queue interaction of any kind

**Write path B** (`ENQUIRY_ATTACHMENT_WORKER` consumer, `enq_attachment.go`):
- Asynchronous, triggered by a RabbitMQ message whose publisher was not found in this review
- Inserts only `ImageOrig` (small/medium/large are explicitly commented out —
  [`enq_attachment.go:363-366`](../../../enq-consumers/src/Enquiry/enq_attachment.go)) plus
  `Docs`, with only orig width/height columns
- **Does set `DIR_QRY_ATTACHEMENT_STATUS`** based on external moderation
- Calls an external permanent-URL upload service for approved images
- Conditionally publishes to Kafka `LMS_LEADS_ENRICHMENT`

**What this means practically**: two attachment rows for what could be the same logical
upload could end up looking structurally different in `DIR_QUERY_ATTACHMENT` depending on
which path wrote them — one has no status/moderation, small/medium/large image variants, no
Kafka trace; the other has a status column, only orig-size images, and a Kafka trace.
**Whether write path A and write path B are meant to be alternative entry points for the
*same* logical event (e.g. a legacy/new-client split), or serve genuinely different purposes
(e.g. path A for immediate UI-facing save, path B for a separate later enrichment step fed by
a different upstream trigger) — [INFERRED, could not be conclusively determined from either
repo; this is the single highest-priority item for the Open Questions in section 15.]**

---

## 10. End-to-End Technical Flows

### Flow A — Buyer submits an attachment via the write API (synchronous)

```
Client (WEB/App)
    │
    ▼
[API] POST /enquiry/enquiryAttachment  {query_id, flag, modid, image_orig[], image_small[],
                                          image_medium[], image_large[], doc_path[]}
    │  enquiryAttachment.go
    │  1. c.ShouldBindBodyWith(&inputParam, binding.JSON) — bind failure → 400 (HTTP 200 on wire)
    │  2. ParameterValidation() — query_id numeric/non-empty, modid non-empty, at least one
    │     image/doc, every image/doc has attach_type_id
    │  3. inputParam.ValidateStruct() — reflection-based truncate/numeric pass
    ▼
[MODEL] EnquiryAttachmentModel()
    │  1. LogRawData() — raw backup log write
    │  2. db.GetDBConn("write")
    │  3. Loop over Document[] and ImageOrig[] (URL-unescape paths, parse widths/heights,
    │     pull matching small/medium/large by index if present)
    │  4. Single multi-row INSERT INTO DIR_QUERY_ATTACHMENT (...) VALUES (...), 2s timeout
    ▼
Client receives: {code:200, status:"Success", response:"Attachment saved successfully"}
    (no queue push, no Kafka, no moderation — this is the entire flow)
```

### Flow B — `ENQUIRY_ATTACHMENT_WORKER` processes a queued attachment message (async, publisher unknown)

```
(unidentified publisher) → RabbitMQ queue (name resolved at consumer startup via os.Args[2])
    │
    ▼
[CONSUME] enq_attachment.go — attachmentdataProcessing()
    │  1. json.Unmarshal into AttachmentMessage{query_id, modid, flag, image_orig/small/
    │     medium/large, docs, product_title, sender_glid}
    │  2. insertAllAttachments():
    │     - Docs → status "1" (always approved, no review)
    │     - ImageOrig[0] → validateAttachment() external moderation call → imageStatus "1"/"2"
    │     - if imageStatus=="1": getPermanentURL() external upload-service call, path/width/
    │       height overwritten with permanent values
    │     - Single multi-row INSERT INTO DIR_QUERY_ATTACHMENT (orig-image + docs only,
    │       + DIR_QRY_ATTACHEMENT_STATUS) ... RETURNING DIR_QRY_ATTACHMENT_ID
    │  3. SELECT fk_glusr_usr_id, query_rcv_glusr_usr_id, DIR_QUERY_MAIL_SEND FROM dir_query
    │     WHERE query_id = $1
    │     ├─ sql.ErrNoRows → Ack, done (no Kafka)
    │     └─ found, mailSend valid and != 0, AND at least one non-"2"-status attachment:
    │           buildKafkaPacket() → inject sender_id/receiver_id →
    │           datapublishintoLMSKafka() → HTTP POST to kafka-pub-api gateway,
    │           topic "LMS_LEADS_ENRICHMENT", SERVICENAME "enquiryEnrichment"
    ▼
d.Ack(false) on success; d.Nack(false, true) on DB/Kafka failure (requeue);
d.Nack(false, false) only on initial JSON unmarshal failure (to fail queue)
```

### Flow C — FENQ bot transfers stored attachments to the Buy-Leads system

```
[CONSUME] enq_post_fenq.go on queue ENQUIRY_FENQ (see Approve-Reject-Finish Enquiry KT,
           Flow C — this is the same FENQ bot that auto-approves/rejects waiting enquiries)
    │  ... (approve/reject decision logic, out of this doc's scope) ...
    │  if offer_id != "" (i.e. the enquiry successfully became a purchased Lead):
    ▼
callTransferAttachment(query_id, offer_id, ...)
    │  getEnquiryAttachment(offer_id, query_id, enqPgConn, ...)
    │     SELECT fk_attach_type_id, dir_qry_attach_*_width/height, dir_qry_attach_doc_path,
    │       dir_qry_attach_img_orig/small/medium/large
    │     FROM dir_query_attachment WHERE ... (query_id/offer_id scoped, exact WHERE clause
    │       not fully captured in this review — see Open Questions)
    │  if no doc path AND no orig-image path found → skip, no HTTP call
    │  else → JSON-marshal into {doc_path[], image_orig[], image_small[], image_medium[],
    │       image_large[]} → HTTP POST to transferAttachment endpoint
    │       (stg: http://stg-leads.imutils.com/wservce/buyleads/saveblattach/,
    │        prod: http://leads.imutils.com/wservce/buyleads/saveblattach/ — notify.yaml)
    ▼
AdditionalInfo["Transfer Attachment Status"] logged success/failure (Kibana), no retry/queue
    on failure — this is a best-effort, synchronous, in-line HTTP call inside the FENQ bot's
    per-message processing
```

---

## 11. Flow-wise DB & Table Usage

### Flow A — Write-API Insert

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1 | enquiry-write Postgres | `DIR_QUERY_ATTACHMENT` | INSERT (multi-row, single statement) | Persist buyer-submitted image/doc paths against the enquiry's `FK_DIR_QUERY_ID` |

### Flow B — Consumer Insert + Enrichment

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1 | enqdb_write Postgres | `DIR_QUERY_ATTACHMENT` | INSERT (multi-row, `RETURNING` id) | Persist consumer-side attachment record with moderation status |
| 2 | enqdb_write Postgres | `dir_query` | SELECT | Determine sender/receiver GLID and whether to publish to Kafka (`DIR_QUERY_MAIL_SEND`) |
| 3 | (external HTTP) | Leap Buyer Photos Review API | GET | Moderate the first orig image |
| 4 | (external HTTP) | Permanent-image-upload API | POST (multipart) | Convert a temp image path into a permanent storage URL, for approved images only |
| 5 | (external HTTP, via kafka-pub-api gateway) | Kafka topic `LMS_LEADS_ENRICHMENT` | Publish | Notify LMS enrichment pipeline of new approved attachment(s) |

### Flow C — FENQ Attachment Transfer

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1 | enqdb_write Postgres (`enqPgConn`) | `dir_query_attachment` | SELECT | Read back all stored attachment rows for the resolved enquiry/offer, regardless of which write path (A or B) inserted them |
| 2 | (external HTTP) | Buy-Leads `saveblattach` API | POST | Forward the attachment set to the purchased-lead system |

---

## 12. Optimization Scope — DB/Response-Time Contributors

### High-impact

1. **Write path A has no queue/async boundary at all** — every attachment save is a
   synchronous DB round-trip inside the HTTP request. This is actually the **simplest, lowest-
   latency** design of the three flows in this doc (good for user-facing responsiveness), but
   it also means **any spike in attachment-upload volume directly loads the write-DB with no
   buffering**, unlike write path B which is queue-buffered. Worth confirming this asymmetry
   with the team — it may be intentional (fast synchronous ack to the client, with path B doing
   the "real" processing later) or an accidental duplication (section 9).
2. **Write path B's `validateAttachment()` and `getPermanentURL()` external HTTP calls are
   synchronous and sequential within the same goroutine**, each with its own multi-second
   timeout (5s and 4s respectively) — a slow external moderation/upload service directly
   inflates this consumer's per-message processing time, same category of risk as GST's BI-API
   call and Call Enquiry's bsmapping call.
   [`enq_attachment.go:626-627,676-677`](../../../enq-consumers/src/Enquiry/enq_attachment.go)

### Medium-impact

3. **No caching anywhere in this feature** — write path A, write path B, and the FENQ
   attachment-transfer read all hit Postgres live every time. Attachment reads specifically
   (Flow C) are a natural low-volatility caching candidate (an enquiry's attachments don't
   change after upload) but this was not implemented.
4. **Two structurally-different INSERT paths into the same table (section 9) mean any future
   schema change (new column, new width/height variant) has to be applied in up to three
   places**: the write API's INSERT, the consumer's INSERT, and the FENQ bot's SELECT.

### Low-impact / good-practice already present

5. **Positive finding**: both write path A and write path B build a single multi-row `INSERT`
   for all attachments in one message/request, rather than looping one INSERT per row — this
   avoids N round-trips for N attachments in both flows.

---

## 13. Cron Inventory

No cron job in `enq-services-scm/pkg/crons/` (`EnquiryMailer`, `EnquiryParseManual`,
`QASapprovedMailer`, `QueryParse`, `setModIds.go`) references `enquiryAttachment` or
`EnquiryAttachment` by name — confirmed via targeted grep across the crons directory. Unlike
`callEnquiry`/`callEnquiryUnidentified` (which write `saveAsJSON` backup files with no
confirmed replay cron), Enquiry Attachment's write-API doesn't even have a backup-file write
on failure (section 5, rule 9) — a failed insert is only logged, with no on-disk artifact to
replay from any cron.

---

## 14. Edge Cases & Gotchas (technical POV)

1. **Two independent insert paths into `DIR_QUERY_ATTACHMENT` with different column sets and
   no shared trigger found** (section 9) — this is the single biggest thing to understand
   before touching this feature. Do not assume the write API's insert and the consumer's
   insert are the same logical event; this review could not confirm they are connected at all.
2. **The write API's `AttachmentFlag` (`flag` field) is validated but never used** (section
   4.1) — dead/reserved input, easy to assume it does something (like a delete/replace
   semantic) when it currently doesn't.
3. **`ImageSmall`/`ImageMedium`/`ImageLarge` are silently truncated to `len(ImageOrig)` in
   write path A** (section 5, rule 7) — a client sending more small/medium/large variants than
   orig images will have the extras silently dropped, no error returned.
4. **Write path B only moderates the first orig image but applies its verdict to all orig
   images in the message** (section 5, rule 11) — if a client somehow sends multiple orig
   images in one message, only the first is actually checked for inappropriate content.
5. **The FENQ attachment-transfer (Flow C) reads back attachments without any status filter**
   (section 5, rule 18) — since write path A never sets a status column at all, any attachment
   inserted through the write API is unconditionally eligible for transfer to the Buy-Leads
   system, with **no moderation ever having run on it**. If write path A is a live, actively-used
   entry point (not dead/legacy), this is a potential content-moderation gap worth flagging to
   the team.
6. **`fetchISQ()` is a known, intentionally-stubbed no-op** (section 5, rule 16) — don't assume
   ISQ (buyer-response) data is actually flowing into the Kafka enrichment packet; the code
   comment says this is pending a bifurcation decision by the LMS team.
7. **Every early-rejection path returns HTTP `200` on the wire** (400-equivalent bind/validation
   failures all get `c.JSON(200, Response)`'d) — the real outcome is only visible via the
   `code`/`status` fields in the response body. Matches the house style already documented in
   Save Enquiry, Call Enquiry, and Approve/Reject/Finish KT docs.
   [`enquiryAttachment.go:47,63`](../../internal/api/controllers/enquiryAttachment.go)

---

## 15. Open Questions

1. **Which service publishes to the RabbitMQ queue that `ENQUIRY_ATTACHMENT_WORKER`
   (`enq_attachment.go`) consumes from?** (section 6) — not found in `enq-services-scm` or
   anywhere else in `enq-consumers/src/Enquiry`. The message shape (`product_title`,
   `sender_glid`) doesn't match the write-API's `InputParam` fields, suggesting a different
   producer entirely, possibly outside these two repos.
2. **Are write path A (write-API direct insert) and write path B (consumer insert) meant to be
   redundant/alternative entry points for the same event, or genuinely separate features that
   happen to share a table name?** (section 9) — this is the highest-priority open question in
   this doc; it materially affects whether the moderation gap in edge case 5 (section 14) is a
   real risk or a non-issue.
3. **What is the exact business meaning of `attach_type_id` values?** (section 3) — no enum or
   decode table found in either repo for this field.
4. **What does `AttachmentFlag` (`flag`) do, if anything?** (section 4.1) — bound and
   validated, never read.
5. **What does `middleware.Validation` actually check**, and does its absence on this endpoint
   (same as 3 of 4 Call Enquiry endpoints) matter for Enquiry Attachment specifically? Carried
   over from Call Enquiry KT's open question, not independently re-investigated here.
6. **Exact `WHERE` clause used by `getEnquiryAttachment()` in `enq_post_fenq.go`** — this
   review confirmed the SELECT column list and table but did not fully trace the query's
   filter predicate line-by-line; confirm it's scoped correctly to avoid cross-enquiry leakage.
7. **Is there any attachment delete/update flow?** — no evidence found of one in either repo;
   confirm with product/frontend team whether attachments are ever removed or replaced.
8. **Live DB schema verification** (column types, nullability, indexes, constraints,
   including whether `DIR_QRY_ATTACHEMENT_STATUS` — note the apparent typo "ATTACHEMENT" vs
   "ATTACHMENT" in write path B's SQL — actually exists with that exact spelling in the live
   schema) — this doc only reflects what the Go SQL strings imply.

---

## See also

- [`Enquiry_Attachment_Business_Doc.md`](./Enquiry_Attachment_Business_Doc.md) — same flows,
  product/business perspective, bina code ke
- [`../Save Enquiry KT/Save_Enquiry_Technical_Doc.md`](../Save%20Enquiry%20KT/Save_Enquiry_Technical_Doc.md) — the enquiry-creation flow that produces the `query_id`/`Query ID` every attachment record links back to
- [`../Approve-Reject-Finish Enquiry KT/Approve_Reject_Finish_Enquiry_Technical_Doc.md`](../Approve-Reject-Finish%20Enquiry%20KT/Approve_Reject_Finish_Enquiry_Technical_Doc.md) — documents `enq_post_fenq.go`'s FENQ bot flow in full; this doc's section 10 Flow C is the attachment-specific extension of that same bot's processing (`callTransferAttachment`, not separately documented there)
- [`../Call Enquiry KT/Call_Enquiry_Technical_Doc.md`](../Call%20Enquiry%20KT/Call_Enquiry_Technical_Doc.md) — source of the `middleware.Validation` positioning pattern and the `insertion_type` field-name convention referenced in sections 2 and 4.4

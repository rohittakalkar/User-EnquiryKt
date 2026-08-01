# Save Enquiry — Technical Doc (Code-Level Deep Dive)

Yeh doc "Save Enquiry" feature ka **technical implementation** cover karta hai — routes, DB
tables, queries, RabbitMQ/Kafka, consumers, crons, sab kuch code se verify karke. Business/
product perspective ke liye [`Save_Enquiry_Business_Doc.md`](./Save_Enquiry_Business_Doc.md)
dekho — dono docs same flows cover karte hain, bas alag audience ke liye.

**Repos**: `enq-services-scm` (write API), `enq-consumers` (downstream RabbitMQ consumers).

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file:line diya gaya
hai har jagah). Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — team se confirm karo]** likh diya hai, guess nahi kiya.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Controller (bind, validate, panic-recover, response shaping, code-430→200 & destination 8/9→3 remapping) | enq-services-scm | [`saveEnquiry.go`](../../internal/api/controllers/saveEnquiry.go) |
| Route registration | enq-services-scm | [`router.go:65-68`](../../internal/api/router/router.go) |
| Core model — InputParam struct, ValidateStruct, sender/receiver/product fetch, SP call, queue pushes, banned-keyword flow | enq-services-scm | [`saveEnquiryModel.go`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go) (~2645 lines) |
| Banned-keyword API call + templated-description detection | enq-services-scm | [`common.go`](../../internal/pkg/persistence/common.go) — `CheckBannedKeyword` (L217), `IsTemplatedDescription` (L273) |
| Backup-file replay cron (reads `QueueSyncing`/`saveAsJSON` backup files, POSTs them back to `saveEnquiry`) | enq-services-scm | [`QueryParse/main.go`](../../pkg/crons/QueryParse/main.go), [`EnquiryParseManual/main.go`](../../pkg/crons/EnquiryParseManual/main.go) |
| Downstream fan-out consumer — publishes to Kafka + republishes FENQ/COMM/LMS queues, triggers BL-Transfer-ISQ, triggers FinishEnquiry | enq-consumers | [`enq_preferred_channel.go`](../../../enq-consumers/src/Enquiry/enq_preferred_channel.go) (queue `ENQ_DATA`) |
| FENQ (waiting-enquiry) processing consumer — writes `DIR_QUERY_WAITING`/`DIR_QUERY_BOUNCED` | enq-consumers | [`enq_post_fenq.go`](../../../enq-consumers/src/Enquiry/enq_post_fenq.go) (queue `ENQUIRY_FENQ`) |
| LMS (Lead Management) consumer — publishes to Kafka topic `ENQ_PNS_DATA` | enq-consumers | [`enq_pns_lms.go`](../../../enq-consumers/src/Enquiry/enq_pns_lms.go) (queue `R_MESSAGE_CENTER_BIZFEED_LMS<0-4>`) |
| Personalization consumer — pushes into **Redis** list | enq-consumers | [`enq_data_personalization.go`](../../../enq-consumers/src/Enquiry/enq_data_personalization.go) (queue `ENQ_PERSONALIZATION_OTHER`) |

---

## 2. Routes (confirmed from router.go)

| Method | Path | Controller |
|---|---|---|
| GET | `/enquiry/saveEnquiry/` | `controllers.SaveEnquiry` |
| GET | `/enquiry/saveEnquiry` | `controllers.SaveEnquiry` |
| POST | `/enquiry/saveEnquiry/` | `controllers.SaveEnquiry` |
| POST | `/enquiry/saveEnquiry` | `controllers.SaveEnquiry` |

[`router.go:65-68`](../../internal/api/router/router.go)

---

## 3. Data Model — Tables

> **Verification note**: yeh sab table/column names Go code ke andar embedded SQL strings se
> liye gaye hain (har row ke saamne file path diya hai). Live DB schema se cross-verify
> **nahi** kiya gaya hai — kisi migration/integration se pehle woh zaroor karo.

| Table | Physical DB (as configured) | Purpose | Key columns (code se) |
|---|---|---|---|
| (write via stored procedure `sp_insert_iil_enquiry_v32`) — underlying tables not directly visible in Go code, SP encapsulates the insert | write-Postgres (`dbConn()`) | Primary enquiry-creation write — ~70 positional params covering sender/receiver/product/query metadata | [`saveEnquiryModel.go:1624-1704`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go) — **actual target table(s) INSIDE the stored procedure are [INFERRED — not visible from Go, confirm with DB/DBA team]** |
| (sender lookup, table name not shown — via `GetSenderDetailsPG`) | read-Postgres (`dbConn2`) | Sender's Name/Email/Company/Address/Mobile etc. fetched before insert | [`saveEnquiryModel.go:1432`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go) |
| (receiver lookup — via `GetReceiverDetailsPG`) — likely `GLUSR_USR`-family table given columns like `GLUSR_USR_FIRSTNAME`, `GLUSR_USR_EMAIL`, `FK_GL_CITY_ID` | read-Postgres (`dbConn1`) | Receiver/supplier's Name/Email/City/Custtype fetched before insert | [`saveEnquiryModel.go:1377-1432`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go) — column names strongly resemble `GLUSR_USR` table from other IM domains, but table name itself is **[INFERRED — confirm with team]** |
| (product lookup — via `GetProductDetailPG`) | read-Postgres (`dbconn3`) | Product name/item-code lookup when only `modrefid` is passed (ModRefType `2`) | [`saveEnquiryModel.go:1506`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go) |
| `DIR_QUERY_WAITING` | Postgres (consumer side, `enq-consumers`) | Waiting/FENQ enquiry state updated by downstream FENQ consumer | `fk_traffic_source_id`, `dir_query_fenq_rsn_id`, `QUERY_ID` — [`enq_post_fenq.go:517`](../../../enq-consumers/src/Enquiry/enq_post_fenq.go) |
| `DIR_QUERY_BOUNCED` | Postgres (consumer side) | Bounced-enquiry state updated by downstream FENQ consumer | `DIR_QUERY_FENQ_RSN_ID`, `QUERY_ID`, `query_rcv_glusr_usr_id` — [`enq_post_fenq.go:968`](../../../enq-consumers/src/Enquiry/enq_post_fenq.go) |
| `eto_attribute` | Postgres (`enqdb_write`, consumer side) | ISQ (buyer requirement) attribute record; consumer checks/skip BL-Transfer if already present, inserts result of BL-Transfer-ISQ call | `fk_eto_ofr_display_id`, `fk_im_spec_master_id`, `eto_attribute_mcat_id`, `fk_im_spec_options_id` — [`enq_preferred_channel.go:226-230,966-996`](../../../enq-consumers/src/Enquiry/enq_preferred_channel.go) |

**Why the primary insert table isn't named**: unlike a plain `INSERT INTO table`, Save Enquiry
writes through a **stored procedure call** (`select sp_insert_iil_enquiry_v32(...)`,
[`saveEnquiryModel.go:1628`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go)) — the
actual table(s) it writes to live inside the SP definition on the DB server, not in the Go
code. This is flagged in Open Questions (section 15).

---

## 4. QueryDestination Codes — "Decode This Magic Value"

`QueryDestination` (aka `query_destination`) is the single most important classification value
in this flow. It comes back as the 3rd field from `sp_insert_iil_enquiry_v32`'s result string
([`saveEnquiryModel.go:1744-1765`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go)),
and every literal comparison against it across the codebase was traced:

| Value | Where compared | What the code does with it | Meaning (evidence-based) |
|---|---|---|---|
| `1` | [`saveEnquiryModel.go:2540,2549,2384(commented)`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go), `PushIntoCCSQueue` | Triggers FENQ push (if not intent-enquiry), LMS push with `insertion_type: "2"` | Normal/direct enquiry — visible to supplier immediately. **[Exact business definition INFERRED — confirm with team]** |
| `2` | [`saveEnquiryModel.go:1724 (querySubPostgresNew error path)`, `2540`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go) | Set as fallback `query_destination` on a DB/SP error; also triggers FENQ push in `PushIntoCCSQueue` | Waiting/error/hold state — used both as an error-path default AND as a genuine SP-returned code (banned-content flow sets `InForceDestination=2` upstream, see section 5.11) |
| `3` | [`saveEnquiry.go:124-126`](../../internal/api/controllers/saveEnquiry.go), `PushIntoCCSQueue` LMS branch (`insertion_type: "4"`), also the fallback used on missing-sender/receiver (section 10, early-exit) | LMS push with `insertion_type: "4"`; also the **normalized value that 8 and 9 collapse into** before the response leaves the controller | A "processed"/alternate-valid destination, and also the generic external-facing value |
| `5` | [`saveEnquiryModel.go:894-896`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go) | Immediately rewritten to `"3"` right after the queue-push block, before being returned to the controller | Transient/intermediate SP-side code that this API normalizes away — **[exact SP-side meaning INFERRED, not visible from Go — confirm with DB team]** |
| `8` | [`saveEnquiry.go:122-126`](../../internal/api/controllers/saveEnquiry.go), also `PushIntoCCSQueue` fenq condition | Controller comment: *"export mark seller received indian leads — implemented on 02-Jul-2025"*; remapped to `3` in the final buyer/supplier-facing response; also counted as an FENQ-triggering destination inside `PushIntoCCSQueue` | Export-marked seller classification (per code comment). **[INFERRED — confirm exact business rule with team]** |
| `9` | [`saveEnquiry.go:122-126`](../../internal/api/controllers/saveEnquiry.go), also `PushIntoCCSQueue` fenq condition | Controller comment: *"Handles HRS-marked enquiries — implemented on 04-Jun-2025"*; remapped to `3` in the final response; also FENQ-triggering | HRS (High-Risk-Supplier, presumed) marked classification. **[INFERRED — HRS expansion and exact rule not found in code, confirm with team]** |

**Where the 8/9→3 remap happens** (controller level, after the model call, before response is
sent):
```go
// saveEnquiry.go:122-126
// Handles HRS-marked enquiries where the query destination is 9 — implemented on 04-Jun-2025
// QueryDestination = 8 stand for export mark seller recived indian leads — implemented on 02-Jul-2025
if Response.QueryDestination == "9" || Response.QueryDestination == "8" {
    Response.QueryDestination = "3"
}
```

**Note**: `PushIntoCCSQueue` (the live queue-push path) makes its *own* independent decisions
based on `queryDestinationId` **before** this controller-level remap runs (since it executes
inside `SaveEnquiryModel`, which is called before the remap) — meaning FENQ/LMS routing logic
still sees the raw `8`/`9`/`5` values, only the final HTTP response is normalized. This is worth
flagging to anyone debugging "why did FENQ get triggered when the response said destination 3."

---

## 5. Business Rules & Validation (code se exhaustive list)

1. **Sender ID and Receiver GL-ID are both mandatory** — resolved from `SenderID` or
   `SenderHash.SGlusrID`, and from `GLUserID` respectively. If either is empty, `"-1"`, or
   negative, the request short-circuits with `queryid=-2`, `query_destination=3`, HTTP body
   code `430` — **no DB round-trip happens**.
   [`saveEnquiryModel.go:307-336`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go)
2. **`ValidateStruct()` runs a full reflection-based pass over every field** of `InputParam`
   (and nested structs `Query`, `Other`, `SenderHash` via `validateStructFields`) before the
   model is even called. Two tag-driven rules exist: `truncate:N` (silently cuts strings to N
   chars) and `numeric:N` (rejects non-numeric values, and if `N>0`, rejects values whose
   string-length exceeds N — returns a hard `429` validation error, unlike truncate which is
   silent). [`saveEnquiryModel.go:1948-2193`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go)
3. **Every string field is UTF-8-cleaned on the way through `ValidateStruct`**, via
   `helper.UtfValidationNew` — this also returns a `utf8_flag` that propagates all the way to
   the response's log-code (`209` instead of the real HTTP code) purely for Kibana log
   filtering. [`saveEnquiryModel.go:1965-1966`, `saveEnquiry.go:103-104`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go)
4. **Description is HTML/URL-unescaped and cleaned of any non-ASCII control bytes** in
   `ParameterValidation` — `url.QueryUnescape` then a byte-level filter keeping only bytes
   `< utf8.RuneSelf` and non-zero.
   [`saveEnquiryModel.go:1049-1062`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go)
5. **Empty `ModID` defaults to `"PCAT"`**, and **`ModID == "PNSBLENQ"` forces
   `InForceDestination = 1`** — a hardcoded per-channel override.
   [`saveEnquiryModel.go:1043-1047`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go)
6. **`ANDROID` ModID gets its description HTML-escaped** (`html.EscapeString`) after
   URL-unescaping — a channel-specific double-processing step not applied to other channels.
   [`saveEnquiryModel.go:1073-1078`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go)
7. **`S_lat`/`S_long` are wiped if they contain any lowercase letter or `=`** (regex
   `[a-z|=]`) — a defensive check against non-numeric/garbage geo values.
   [`saveEnquiryModel.go:1081-1086`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go)
8. **Empty `InForceDestination` defaults to `"-1"`.**
   [`saveEnquiryModel.go:1089-1091`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go)
9. **`ModID == "FCP"` appends `CategoryType` to `QueryRefText`**, and **`QueryRefText ==
   "MY-BusinessFeeds"` with `ModID == "MY"` gets normalized to `"MY-BusinessFeed"`** (typo-fix
   style hardcoded remap). [`saveEnquiryModel.go:1093-1100`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go)
10. **`LatLongAccuracy` is wiped if the integer part exceeds 7 digits or decimal part exceeds 8
    digits** — a length-sanity guard before it hits the DB.
    [`saveEnquiryModel.go:1103-1114`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go)
11. **Banned-keyword check is conditionally skipped for templated descriptions.** Only runs
    `persistence.CheckBannedKeyword` if `Description != ""` **and**
    `!persistence.IsTemplatedDescription(...)`. `IsTemplatedDescription` matches against 14
    hardcoded regex templates (e.g. `"I am interested in buying..."`,
    `"...was attempted via WhatsApp"`).
    [`saveEnquiryModel.go:487-508`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go),
    [`common.go:273-298`](../../internal/pkg/persistence/common.go)
12. **A banned-keyword hit force-sets `InForceDestination = 2` and `WaitingreasonId = -999`**
    before the SP call — this is how a spam/abuse-flagged enquiry gets routed to a
    non-visible/waiting state at the DB layer.
    [`saveEnquiryModel.go:503-506`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go)
13. **The SP call has a 2-second timeout in non-local environments** (100s locally) — any
    `deadline exceeded`/`canceling statement`/`timeout` error is mapped to HTTP `503`, anything
    else to `500`. Both cases trigger `saveAsJSON` backup-file write.
    [`saveEnquiryModel.go:1616-1721`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go)
14. **Receiver/Sender/Product DB fetches also run on a short timeout** (2s prod / 100s local),
    fetched via **3 parallel goroutines** (`sync.WaitGroup`) before the SP call.
    [`saveEnquiryModel.go:375-409`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go)
15. **The final SP-call path only runs if**: `receiverGlid != ""` **and**
    `Description != ""` **and** (`senderId != ""` **or** `ModID == "IMOB"` **or**
    `ModID == "IOS"`... — actually only `"IMOB"`/`"ANDROID"` are checked, not `"IOS"`). If none
    of these hold, request short-circuits to the same `queryid=-2`/`query_destination=3`/`430`
    response as the sender/receiver-missing case, with an explanatory `errorAlert` string.
    [`saveEnquiryModel.go:511-561`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go)
16. **Sender name has honorific prefixes stripped** (`Mr.`, `Mrs.`, `Dr.`, `Ms.` via regex) and
    leading whitespace trimmed. [`saveEnquiryModel.go:1257-1263`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go)
17. **Sender mobile normalization is channel- and country-specific**: `"mobile number"`/`"null"`
    (case-insensitive) is wiped; `IOS` gets a `+` prefix forced on if missing; `ANDROID` gets a
    double `++` collapsed to single `+`; parens/spaces stripped globally; then if country is
    India and number is exactly 10 digits, `+91-` is prefixed, else a country-code lookup
    (`getCountryCodePG`) is used to prefix based on `UserCountryISO`.
    [`saveEnquiryModel.go:1265-1282`, `1341-1362`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go)
18. **Field-length caps applied server-side regardless of `ValidateStruct` truncate tags**:
    SName ≤80, SEmail ≤60, SOrganization ≤80, SAddress ≤150, SCity/SState/SCountry ≤50,
    SFax ≤50, SPhone/SMobile ≤60, SPin ≤30, SReferrer ≤200 — enforced a second time inside
    `setSenderHash`, independent of the struct tags on `InputParam`.
    [`saveEnquiryModel.go:1217-1254`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go)
19. **HTTP response code `430` is silently remapped to `200`** by the controller, with
    `Queryid=-2`, `QueryDestination=3` — i.e. every "early guardrail rejection" (missing
    sender/receiver/description) returns HTTP `200` to the caller, distinguishable only by the
    `queryid` value being `-2`. Any other non-`200` code is passed through as-is with
    `Queryid="Pending"`. [`saveEnquiry.go:127-138`](../../internal/api/controllers/saveEnquiry.go)
20. **On any panic in the controller, the response always reports `Queryid=-2`,
    `QueryDestination=3`, HTTP `503`**, and if the DB write had actually already succeeded
    before the panic (`queueStatus["DBStatus"]=="1"` and `Query_id != ""`) but at least one of
    FENQ/LMSQ/PERSONALIZATION status is still empty, `QueueSyncing` is invoked to write a
    backup-recovery file — meaning a downstream-queue panic does **not** lose the already-saved
    enquiry. [`saveEnquiry.go:34-58`](../../internal/api/controllers/saveEnquiry.go)

---

## 6. Dead Code — Legacy Queue-Push Functions

Several queue-push functions are fully implemented but their call sites in `querySubNew` are
**commented out**, replaced by the single `PushIntoCCSQueue` call which now handles FENQ, LMS,
and Personalization data all bundled into one `ENQ_DATA` message (see section 7):

| Function | Status | What it used to do |
|---|---|---|
| `pushEnquiryFenqDataRabbit` | Dead — defined but never called (call site commented at [`saveEnquiryModel.go:835`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go)) | Would push directly to an `ENQUIRY_FENQ`-style queue |
| `PushIntoLMSQueue` | Dead — defined but never called (call site commented at [`saveEnquiryModel.go:842`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go)) | Would push directly to `R_MESSAGE_CENTER_BIZFEED_LMS<0-4>` with `insertion_type` 2 or 4 depending on destination |
| `pushIntoPersonalizationQueue` | Dead — defined but never called (call site commented at [`saveEnquiryModel.go:849,2317`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go)) | Would push directly to `ENQ_PERSONALIZATION_OTHER` |
| Old direct-parallel `PushIntoCCSQueue` variant (commented, ~L2473-2517) | Dead — fully commented-out earlier signature (no fenq/personalization params) | Superseded by the current `PushIntoCCSQueue(txn, inputParam, SERVICENAME, uniqueid, senderId, fenqdata, personalizationData, queryDestinationId, fenqflag, queryId)` signature |

**Current live path**: `querySubNew` runs a **single** goroutine (not three parallel ones like
the dead code implies) that calls `DataForFenqQueue` + `PersonalizationFunction` to build data,
then a single `PushIntoCCSQueue` call bundles FENQ data, LMS data, and Personalization data all
into **one** `ENQ_DATA` RabbitMQ message, letting the downstream `enq_preferred_channel.go`
consumer fan them back out to their respective queues.
[`saveEnquiryModel.go:868-887`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go)

This is a meaningful architecture shift from "push directly to N queues in parallel from the
API" to "push one bundled message, let the consumer fan out" — worth knowing before assuming
the commented code represents current behavior.

---

## 7. RabbitMQ — Queues Touched By Save Enquiry

| Queue / Exchange | Publisher | Consumer | Purpose |
|---|---|---|---|
| `ENQ_DATA` | `PushIntoCCSQueue` in `saveEnquiryModel.go` ([:2529-2645](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go)), `SERVICENAME="EnquirySave"` | [`enq_preferred_channel.go`](../../../enq-consumers/src/Enquiry/enq_preferred_channel.go) | Primary fan-out message per saved enquiry — bundles FENQ/LMS/Personalization/Comm sub-payloads plus `glid`/`modid`/`QueryDestinationId`/`queryid` |
| `ENQUIRY_FENQ` | (nested inside `ENQ_DATA` as `FENQ_Data`, republished by consumer via `publishToRMQ`) | [`enq_post_fenq.go`](../../../enq-consumers/src/Enquiry/enq_post_fenq.go) | Waiting/hold-state enquiry processing — writes `DIR_QUERY_WAITING`/`DIR_QUERY_BOUNCED` |
| `R_MESSAGE_CENTER_BIZFEED_LMS<0-4>` (5-way sharded, random modulus) | (nested inside `ENQ_DATA` as `LMS_Data`, republished by consumer) | [`enq_pns_lms.go`](../../../enq-consumers/src/Enquiry/enq_pns_lms.go) | Lead-Management enquiry intake |
| `LMS_TXN_ENQ` | `enq_preferred_channel.go` (`processAndPublishLMSData`, exchange `lms-gcp.topic`) | Downstream LMS transaction consumer (not traced in this review) | Enriched LMS data (merges `get_query_data_v4` DB result: `enquiry_info`, `isq`, `enrichment`, `transaction_attachments`) |
| `ENQ_PERSONALIZATION_OTHER` | (nested inside `ENQ_DATA` as `Personalization_Data`, republished by consumer) | [`enq_data_personalization.go`](../../../enq-consumers/src/Enquiry/enq_data_personalization.go) | Personalization/recommendation-engine intake — this consumer then pushes into **Redis**, not a further RabbitMQ hop (section 9) |
| `PERSONALIZE_ENQUIRY_DATA<random 0-N>` (Redis list key, **not RabbitMQ**) | `enq_data_personalization.go` `redisEnqueue` | Redis-side consumer (Resque-style, not traced — outside these two repos) | Final personalization payload destination |

**FinishEnquiry side-effect**: when `enq_preferred_channel.go` sees
`Personalization_Queue_Name == "ENQ_PERSONALIZATION_OTHER"` **and**
`QueryDestinationId == "1"` **and** the personalization payload's 4th q_data element equals
`leap@indiamart.com`, it makes an **outbound HTTP call to the `finishEnquiry` API**
(`callFinishEnquiry`) — a save-enquiry-triggered auto-finish path specific to a hardcoded email
recipient. [`enq_preferred_channel.go:266-296,834-886`](../../../enq-consumers/src/Enquiry/enq_preferred_channel.go)

**BL-Transfer-ISQ side-effect**: same consumer, independent of the above, checks
`SERVICENAME` against `FinishEnquiry`/`EnquiryMailerCron`/`QASapprovedMailerCron` (**note: NOT
`EnquirySave`** — meaning Save-Enquiry-originated `ENQ_DATA` messages do **not** trigger this
branch directly; only messages *re-published by those other services* into the same queue
would) — this path is not part of the direct Save Enquiry flow.
[`enq_preferred_channel.go:219-243`](../../../enq-consumers/src/Enquiry/enq_preferred_channel.go)

---

## 8. Kafka

Save Enquiry's write path (`enq-services-scm`) itself has **zero Kafka usage** — confirmed via
repo-wide grep for `kafka`/`Kafka`, no matches in `enq-services-scm`.

However, the **downstream consumer** `enq_preferred_channel.go` (which is triggered by Save
Enquiry's `ENQ_DATA` message) does publish to Kafka in two places:

| Function | Kafka topic | Trigger | Purpose (inferred from code) |
|---|---|---|---|
| `datapublishintoKafka` | `pref_channel` | When the incoming `ENQ_DATA` message has a non-empty `flag` field | Preferred-channel classification signal (e.g. WhatsApp routing) — sends `{flag, glid, modid}` |
| `datapublishintoENQKafka` | `Enquiry_Data` | Inside `processAndPublishLMSData`, after publishing enriched LMS data to `LMS_TXN_ENQ` | Mirrors the enriched LMS payload to Kafka — **exact downstream consumer of this topic not traced in this review [INFERRED]** |

Both use an HTTP-bridge pattern (`POST http://{env}-kafka-pub-api.intermesh.net/kafkapublish`)
rather than a native Kafka client library —
[`enq_preferred_channel.go:319-433,744-833`](../../../enq-consumers/src/Enquiry/enq_preferred_channel.go).

`enq_pns_lms.go` also references a Kafka topic `ENQ_PNS_DATA`
([`enq_pns_lms.go:346`](../../../enq-consumers/src/Enquiry/enq_pns_lms.go)) — same HTTP-bridge
pattern, triggered from the LMS-consumer side of the flow.

**Straight answer if asked "does Save Enquiry use Kafka":** the write API itself does not; two
of its downstream consumers do, via an internal Kafka-publish HTTP proxy, not a direct Kafka
client.

---

## 9. Redis

Save Enquiry's write path (`enq-services-scm`) has **no Redis usage** — confirmed via
repo-wide grep; the "redis" hit inside `QASapprovedMailer/main.go` is a local Go variable named
`redisData` used purely as an in-memory map, unrelated to the actual Redis client — a
false-positive, not a real dependency.

The **downstream Personalization consumer** (`enq_data_personalization.go`, in `enq-consumers`)
**does** use a real Redis client (`github.com/go-redis/redis`):

- Connects via `utils.RedisClient()` at consumer startup.
- Publishes each personalization payload with
  `redisClient.RPush("resque:queue:PERSONALIZE_ENQUIRY_DATA<random_0-N>", jobJSON)` — a
  **Resque-style job queue pattern** (Redis list used as a work queue, presumably consumed by a
  Ruby/Resque-based personalization worker outside these two repos).
  [`enq_data_personalization.go:229-262`](../../../enq-consumers/src/Enquiry/enq_data_personalization.go)

**Gap worth flagging**: Save Enquiry's read-heavy sub-fetches (`GetSenderDetailsPG`,
`GetReceiverDetailsPG`, `GetProductDetailPG`) hit Postgres on every single request with no
caching layer at all, despite Redis infrastructure already existing in the broader
enquiry-domain codebase (via the personalization consumer). See Optimization Scope, section 12.

---

## 10. End-to-End Technical Flows

### Flow A — Happy path: normal enquiry save

```
Client (WEB/IOS/ANDROID/IMOB/...)
    │
    ▼
[API] POST /enquiry/saveEnquiry  (or GET, or trailing-slash variant)
    │  saveEnquiry.go
    │  1. c.ShouldBindBodyWith(&inputParam, binding.JSON) — bind failure → 429, early return
    │  2. inputParam.ValidateStruct() — reflection-based truncate/numeric/UTF8 pass
    ▼
[MODEL] SaveEnquiryModel()
    │  1. Opens 4 separate DB connections (dbConn x4) — connection failure → 504
    │  2. Resolves senderId (SenderID or SenderHash.SGlusrID) and receiverGlid (GLUserID)
    │     — missing/invalid → early exit, code 430, queryid=-2, destination=3
    │  3. setUserHash / setSenderHash / setRecHash — merge input aliases into canonical hashes
    │  4. ParameterValidation() — ModID-specific normalization (section 5.5-5.10)
    │  5. PARALLEL (sync.WaitGroup, 3 goroutines):
    │     ├─ GetReceiverDetailsPG()  (dbconn1, 2s timeout)
    │     ├─ GetSenderDetailsPG()    (dbconn2, 2s timeout)
    │     └─ GetProductDetailPG()    (dbconn3, 2s timeout, only if ModRefType=="2" and no ProductName)
    │  6. addFetchedReceiverDetails / addFetchedSenderDetails — merge DB data into hashes
    │  7. If Description present and NOT templated → persistence.CheckBannedKeyword() (external BAN API, 5s timeout)
    │     └─ banned==1 → InForceDestination=2, WaitingreasonId=-999
    │  8. setSubjectDescription()
    │  9. Guard: receiverGlid != "" && Description != "" && (senderId != "" || ModID in {IMOB,ANDROID})
    │     else → early exit, code 430
    ▼
[DB — synchronous, 2s timeout] querySubPostgresNew()
    │  select sp_insert_iil_enquiry_v32($1..$70)  — ~70 positional args (sender/receiver/product/query metadata)
    │  parses result string "(queryid,errstr,query_destination)"
    │  error → saveAsJSON() backup file written, code 500/503
    ▼
[QUEUE — single goroutine, panic-recovered] querySubNew()
    │  DataForFenqQueue() + PersonalizationFunction() build payloads
    │  PushIntoCCSQueue() → RabbitMQ queue ENQ_DATA (SERVICENAME=EnquirySave)
    │     bundles: FENQ_Data (if applicable), LMS_Data (if destination 1 or 3), Personalization_Data
    ▼
[CONTROLLER] response shaping
    │  destination 9 or 8 → remapped to 3 (buyer/supplier-facing only)
    │  code 430 → remapped to HTTP 200, queryid=-2, destination=3
    │  code 200/430 (non-error) → Response.Error cleared
    │  else → Queryid="Pending"
    ▼
Client receives: {code, queryid, query_destination, UNIQUE_MSG_ID, ...}

    (async, downstream)
    ▼
[CONSUME] enq_preferred_channel.go on ENQ_DATA
    │  1. If FinishEnquiry/EnquiryMailerCron/QASapprovedMailerCron + mcat_id → BL-Transfer-ISQ check
    │     (NOT triggered by plain EnquirySave messages — see section 7)
    │  2. flag != "" → datapublishintoKafka() → Kafka topic "pref_channel"
    │  3. flag in {"", "ENQ"} → publishToRMQ() fans FENQ_Data/COMM_Data/LMS_Data back out to their queues
    │     ├─ LMS_Data → processAndPublishLMSData() → enriches via get_query_data_v4() SP,
    │     │              publishes to LMS_TXN_ENQ + Kafka topic "Enquiry_Data"
    │     └─ destination==1 && intent_queue=="leap@indiamart.com" → calls finishEnquiry API (HTTP)
    ▼
[CONSUME] enq_post_fenq.go on ENQUIRY_FENQ (if FENQ_Data was populated)
    │  writes DIR_QUERY_WAITING / DIR_QUERY_BOUNCED
    ▼
[CONSUME] enq_pns_lms.go on R_MESSAGE_CENTER_BIZFEED_LMS<n> (if LMS_Data was populated)
    │  reads enquiry data, publishes to Kafka topic ENQ_PNS_DATA
    ▼
[CONSUME] enq_data_personalization.go on ENQ_PERSONALIZATION_OTHER
    │  redisClient.RPush("resque:queue:PERSONALIZE_ENQUIRY_DATA<n>", payload)
```

### Flow B — Banned-keyword / spam description

```
Same as Flow A up to step 7, except:
    │
    ▼
CheckBannedKeyword() returns banned==1
    │  InForceDestination = 2, WaitingreasonId = -999
    ▼
sp_insert_iil_enquiry_v32 still called (with these overridden values)
    ▼
Response still returns queryid (enquiry IS saved), but destination reflects the hold-state
```

### Flow C — Early-exit guardrail (missing sender/receiver/description)

```
[MODEL] SaveEnquiryModel()
    │  senderId=="" or receiverGlid=="" or numericSenderId<0 or numericReceiverId<0
    │      OR the section-15 guard (receiverGlid/Description/senderId-or-ModID) fails
    ▼
Return immediately — NO DB round-trip, NO queue push
    output = {queryid: "-2", query_destination: "3", error: "<reason>"}
    code = "430"
    ▼
[CONTROLLER] 430 → remapped to HTTP 200, queryid=-2, destination=3
```

### Flow D — Panic / unrecovered error mid-request

```
[CONTROLLER] defer/recover() catches panic anywhere in SaveEnquiryModel or its callees
    │  Response = {code:503, queryid:-2, query_destination:3}
    │  Kibana log written with stack trace (truncated to 1000 chars)
    │  IF queueStatus["DBStatus"]=="1" AND Query_id != "" AND
    │     (FENQ=="" OR LMSQ=="" OR PERSONALIZATION=="")
    │       → QueueSyncing() writes a backup JSON file (inputParam + queueStatus)
    ▼
Client receives HTTP 503
```

### Flow E — Backup-file recovery (next-day replay)

```
[CRON] QueryParse / EnquiryParseManual (pkg/crons)
    │  reads backup JSON files written by saveAsJSON() (SP-call failures) or
    │  QueueSyncing() (post-save queue-push failures/panics)
    ▼
[HTTP] POSTs each backup payload back to the live saveEnquiry endpoint
    (http://{env}.../enquiry/saveEnquiry/) — effectively replays Flow A from scratch
```

---

## 11. Flow-wise DB & Table Usage

### Flow A — Normal Save (every DB round-trip)

| # | DB (physical) | Table / SP | Operation | Why |
|---|---|---|---|---|
| 1 | write-Postgres (`dbconn1`) | receiver lookup (table name not directly visible — likely `GLUSR_USR`-family, **[INFERRED]**) | SELECT (`GetReceiverDetailsPG`) | Fetch supplier Name/Email/City/Custtype etc. for enrichment before insert |
| 2 | write-Postgres (`dbconn2`) | sender lookup (table name not visible, **[INFERRED]**) | SELECT (`GetSenderDetailsPG`) | Fetch buyer Name/Email/Company/Address before insert |
| 3 | write-Postgres (`dbconn3`) | product lookup (table name not visible, **[INFERRED]**) | SELECT (`GetProductDetailPG`) | Only runs if `ModRefType=="2"` and no `ProductName` supplied — fetches product name from catalog |
| 4 | write-Postgres (`dbconn`) | `sp_insert_iil_enquiry_v32` | Stored-procedure CALL (effectively an INSERT + business logic inside the SP) | The actual enquiry-creation write, ~70 params |
| 5 | (external HTTP, not DB) | BAN API | HTTP GET-with-body | Spam/abuse content check on Description (only if not templated) |
| 6 | RabbitMQ | `ENQ_DATA` | Publish | Single bundled fan-out message for all downstream systems |

**Total round-trips for a single normal save: 3 parallel SELECTs + 1 SP call + 1 external HTTP
call (conditional) + 1 queue publish.** Unlike GST's 9-10 round-trips across 4 physical DBs,
Save Enquiry's DB footprint is comparatively lean — but it fans out to **3+ RabbitMQ queues and
2 Kafka topics** on the consumer side (see below).

### Flow A (continued) — Downstream Consumer DB Usage

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 7 | Postgres (`enqdb_write`, consumer-side) | `eto_attribute` | SELECT (`COUNT(1) WHERE fk_eto_ofr_display_id=$1`) | Check if BL-ISQ records already exist for this query, before deciding to call BL-Transfer-ISQ (only for `FinishEnquiry`/mailer-cron-originated messages, not plain saves) |
| 8 | Postgres (consumer-side) | `eto_attribute` | Bulk INSERT | Only if BL-Transfer-ISQ call returns records |
| 9 | Postgres (LMS consumer-side, `enq_pns_lms.go`) | not traced in detail (SELECT queries at L224,243) | SELECT | Enrichment lookups before LMS Kafka publish |
| 10 | Postgres (`enq_preferred_channel.go`, `processAndPublishLMSData`) | `get_query_data_v4(query_id, query_destination)` (function call) | SELECT (function call) | Fetches `enquiry_info`/`isq`/`enrichment`/`transaction_attachments` to merge into LMS payload before `LMS_TXN_ENQ` publish |
| 11 | Postgres (`enq_post_fenq.go`) | `DIR_QUERY_WAITING` | UPDATE (`fk_traffic_source_id`, `dir_query_fenq_rsn_id`) | Only when FENQ_Data was populated (destination 2/8/9, or 1/5 non-intent) |
| 12 | Postgres (`enq_post_fenq.go`) | `DIR_QUERY_BOUNCED` | UPDATE (`DIR_QUERY_FENQ_RSN_ID`) | Alternate FENQ outcome path |
| 13 | Redis (`enq_data_personalization.go`) | list key `resque:queue:PERSONALIZE_ENQUIRY_DATA<n>` | RPUSH | Final personalization job handoff |

### Flow B — Banned-keyword

| # | DB / Service | Table | Operation | Why |
|---|---|---|---|---|
| 1 | External HTTP (BAN API) | — | POST/GET-with-body | Content moderation check |
| 2-6 | *(same as Flow A steps 1,2,4,5,6 — receiver/sender fetch + SP insert + queue publish)* | — | — | Enquiry still gets created, just with `InForceDestination=2` baked into the SP args |

### Flow C — Early-exit Guardrail

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| — | none | — | — | Request rejected before any DB connection is used for the actual write path (4 `dbConn()` calls still happen unconditionally at the top of `SaveEnquiryModel`, but no query is issued against them if the sender/receiver guard fails first) |

### Flow D — Panic Recovery

| # | DB / Storage | Table / File | Operation | Why |
|---|---|---|---|---|
| 1 | Local filesystem (`GetQueuSyncSaveEnquiryPath()`) | backup JSON file `{date}_{modid}_{glid}_{unixts}.json` | WRITE | Preserve full input + queue status for next-day cron replay, only if the DB write itself already succeeded |

### Flow E — Backup Replay Cron

| # | DB / Service | Table | Operation | Why |
|---|---|---|---|---|
| 1 | Local filesystem | backup JSON files (previous day) | READ | Source of replay payloads |
| 2 | HTTP (self) | `/enquiry/saveEnquiry/` | POST | Re-runs the entire Flow A from scratch for each backed-up payload |

---

## 12. Optimization Scope — DB/Response-Time Contributors

### High-impact

1. **Four separate `dbConn()` calls open unconditionally at the top of `SaveEnquiryModel`**,
   even for requests that will short-circuit on the sender/receiver guard a few lines later.
   Each `dbConn()` call has its own connection-pool-stat logging overhead.
   [`saveEnquiryModel.go:270-289`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go).
   **Suggestion**: move the sender/receiver presence check before connection acquisition, or
   lazily acquire connections only once the guard passes.
2. **The external BAN API call is synchronous and in the critical path** before the SP insert
   — a 5-second client timeout ([`common.go:248`](../../internal/pkg/persistence/common.go))
   means a slow BAN API directly inflates every non-templated enquiry's save latency. Given
   Save Enquiry is the highest-traffic write endpoint in this domain, this is the single
   largest external-dependency risk. **Suggestion**: consider making the BAN check async
   (flag after the fact) if product tolerates a short window of unfiltered visibility, or
   tighten the timeout further with a fast-fail default.
3. **`sp_insert_iil_enquiry_v32` takes ~70 positional parameters** — any schema drift on the SP
   side silently breaks parameter ordering with no compile-time safety (they're raw
   `[]interface{}`). A parameter-order mismatch would not error at the Go layer, only produce
   wrong data. **Suggestion**: introduce a struct-tagged parameter builder or at minimum a unit
   test asserting param count/order against a known-good SP signature.

### Medium-impact

4. **No caching on sender/receiver/product detail fetches** — every single save hits Postgres
   3 times for data (sender info, receiver info, product info) that changes far less often than
   enquiries are submitted (a given supplier's basic profile info is relatively static).
   Redis infrastructure already exists in the broader domain (used by the personalization
   consumer, section 9) but isn't wired into these read paths. **Suggestion**: short-TTL
   cache-aside (2-5 min) on receiver/sender detail lookups would cut 2 of 3 parallel SELECTs
   for high-frequency sender/receiver pairs.
5. **`ENQ_DATA` message is a single large bundled payload** (FENQ+LMS+Personalization+Comm all
   in one message) that the consumer then re-serializes and republishes to up to 3 separate
   queues — this trades "3 parallel publishes from the API" (the commented-out dead-code
   approach, section 6) for "1 publish + 3 sequential-ish republishes downstream," moving
   latency into the consumer rather than removing it. **Not necessarily wrong** (keeps the API
   fast), but worth knowing this is a latency-relocation, not a latency-elimination.
6. **Both `sp_insert_iil_enquiry_v32` timeout and the receiver/sender/product fetch timeouts are
   hardcoded to `2` seconds in non-local environments** (not configurable per-environment via
   env var) — [`saveEnquiryModel.go:1616-1620`, `1369-1373`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go).
   A prod traffic spike causing DB slowness would hit this ceiling uniformly with no tuning
   lever short of a code change.

### Low-impact / good-practice already present

7. **Positive finding**: Receiver, Sender, and (conditional) Product detail fetches already run
   in parallel goroutines (`sync.WaitGroup`), not sequentially — this is a good pattern already
   in place. [`saveEnquiryModel.go:375-409`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go)
8. **Positive finding**: The queue-push step is wrapped in its own panic-recover and runs
   *after* the DB write is confirmed successful — a downstream queue failure cannot roll back or
   corrupt the already-saved enquiry, and the backup-file safety net (Flow D/E) ensures no data
   loss. This is a solid resilience pattern.

---

## 13. Cron Inventory

| Cron | Live? | Trigger | Kya karta hai |
|---|---|---|---|
| `QueryParse` (`pkg/crons/QueryParse/main.go`) | **Yes, live** | Scheduled (reads previous day's backup files by date-stamped filename) | Reads `saveAsJSON`/`QueueSyncing` backup JSON files and POSTs them back to `/enquiry/saveEnquiry/` — the primary backup-replay mechanism for Save Enquiry |
| `EnquiryParseManual` (`pkg/crons/EnquiryParseManual/main.go`) | **Yes, but manual-trigger flavor** (per name and presence of `test.csv` in the same dir) | Manually invoked, likely for backfilling a specific date range or file set | Same core replay logic against `saveEnquiry`, but designed for manual/ad-hoc runs rather than the daily automated schedule — **exact trigger mechanism (cron schedule vs. manual shell invocation) [INFERRED, confirm with team]** |
| `EnquiryMailer` (`pkg/crons/EnquiryMailer/main.go`) | Present, but **not part of the Save Enquiry write path** — publishes messages that the `enq_preferred_channel.go` consumer recognizes via `SERVICENAME=="EnquiryMailerCron"` (section 7) | Out of scope for this doc — flagged only because it shares the downstream `ENQ_DATA`-consuming pipeline | Out of scope |
| `QASapprovedMailer` (`pkg/crons/QASapprovedMailer/main.go`) | Present, same relationship as above (`SERVICENAME=="QASapprovedMailerCron"`) | Out of scope for this doc | Out of scope |
| `SetModIds` (`pkg/crons/setModIds.go`, exposed as `GET /enquiry/SetModIds`) | Live, but unrelated to the save-write path itself | HTTP-triggered | Not investigated in this review — out of scope |

---

## 14. Edge Cases & Gotchas (technical POV)

1. **Dead code overlaps with live code in naming** — `pushEnquiryFenqDataRabbit`,
   `PushIntoLMSQueue`, `pushIntoPersonalizationQueue`, and an older `PushIntoCCSQueue` signature
   are all fully implemented but unreachable (section 6). A future engineer grepping for
   "where does FENQ get pushed" will find 2 candidates (one dead, one live) — the live one is
   the single `PushIntoCCSQueue` call inside the one active goroutine in `querySubNew`.
2. **QueryDestination remap happens at the controller, not the model** — `PushIntoCCSQueue`
   (inside the model layer) sees raw destination values (`8`, `9`, `5`) and makes real routing
   decisions off them; only the final HTTP response gets the `8/9→3` and `5→3` normalization.
   Debugging "why did this get FENQ'd when the API said destination 3" requires knowing this
   split (section 4).
3. **HTTP `430` is invisible to API consumers** — every early-guardrail rejection returns HTTP
   `200` with `queryid=-2` (section 5.19). Any monitoring/alerting based on HTTP status codes
   alone will miss these rejections entirely; they must be tracked via the `queryid` field or
   Kibana logs.
4. **The stored procedure `sp_insert_iil_enquiry_v32` is a black box from the Go codebase** —
   the actual table(s) it writes to, and any additional business logic inside it (e.g. does it
   also decide `query_destination` itself, or defer to a trigger?) are not visible here. Any
   change to enquiry-insert business rules that isn't explained by this Go file likely lives in
   the SP definition on the DB server (Open Questions, section 15).
5. **`ModID == "IOS"` is referenced in section-15's guard condition documentation but the actual
   code only checks `"IMOB"` and `"ANDROID"`** — meaning a plain `"IOS"` ModID request with a
   missing `senderId` will hit the early-exit guardrail, unlike IMOB/ANDROID which are exempted
   from the strict-sender-required rule.
   [`saveEnquiryModel.go:511`](../../internal/pkg/models/saveEnquiry/saveEnquiryModel.go) — this
   asymmetry is easy to miss and worth confirming is intentional.
6. **QueueStatus map keys are pre-seeded with empty strings** in the controller
   (`"FENQ": "", "LMSQ": "", "PERSONALIZATION": "", ...`) but the live code path only ever
   populates `"CCS"` (via `PushIntoCCSQueue`'s single call) — `"FENQ"` and `"LMSQ"` keys are
   effectively **always empty** in the live flow, which means the panic-handler's condition
   `(queueStatus["FENQ"]=="" || queueStatus["LMSQ"]=="" || ...)` for triggering `QueueSyncing`
   backup-writes is **almost always true** by design (since those keys are never set by the
   live single-goroutine path) — worth confirming this doesn't cause excessive/unnecessary
   backup-file writes on every panic, even ones where the actual push succeeded via `CCS`.
   [`saveEnquiry.go:32,49-51`](../../internal/api/controllers/saveEnquiry.go)
7. **BL-Transfer-ISQ and FinishEnquiry auto-trigger logic in the consumer is gated on
   `SERVICENAME` values that plain Save-Enquiry messages don't carry** (`FinishEnquiry`/
   `EnquiryMailerCron`/`QASapprovedMailerCron` for BL-Transfer; `leap@indiamart.com` intent
   check for auto-FinishEnquiry) — meaning most of `enq_preferred_channel.go`'s more complex
   logic is dormant for typical Save-Enquiry-originated traffic and only fires for
   re-published/mailer-originated messages sharing the same `ENQ_DATA`-style queue shape.
   Easy to misread as "every save triggers BL-transfer," which it does not.

---

## 15. Open Questions

1. **What table(s) does `sp_insert_iil_enquiry_v32` actually write to?** Not visible from Go
   code — the SP is called by name only, with positional params. Confirm with DB/DBA team
   before any schema-impacting change.
2. **Exact business/technical meaning of `QueryDestination` codes `2`, `5`, `8`, `9`** — `2` and
   `5` have no explanatory comment anywhere in the code (only inferred from control-flow
   context); `8`/`9` have brief code comments ("export mark," "HRS-marked") but no fuller
   definition. Confirm with the Enquiry/Trust product team.
3. **Table names behind `GetSenderDetailsPG`/`GetReceiverDetailsPG`/`GetProductDetailPG`** —
   SQL strings were read in full but table names weren't printed as bare identifiers in the
   portions reviewed; column-name shape strongly suggests a `GLUSR_USR`-family table for the
   receiver lookup, but this is inference, not confirmed.
4. **`EnquiryParseManual`'s actual trigger mechanism** (scheduled cron vs. purely manual
   shell-invoked tool) — the presence of `test.csv` alongside it suggests a manual/testing tool
   similar in spirit to GST's `gst_tact_cron_sync.go` "dead/CSV-driven" pattern, but this wasn't
   conclusively confirmed.
5. **Downstream consumer of Kafka topics `pref_channel`, `Enquiry_Data`, and `ENQ_PNS_DATA`** —
   not traced in this review (outside the `enq-services-scm`/`enq-consumers` boundary as
   defined for this doc).
6. **Whether `QueueSyncing`'s near-always-true trigger condition (section 14.6) is intentional
   or a latent bug** left over from when FENQ/LMSQ were pushed by separate goroutines (dead
   code, section 6) that did populate those queueStatus keys.
7. **Live DB schema verification** (column types, nullability, indexes, constraints) for
   `DIR_QUERY_WAITING`, `DIR_QUERY_BOUNCED`, and `eto_attribute` — this doc only reflects what
   the Go SQL strings imply.

---

## See also

- [`Save_Enquiry_Business_Doc.md`](./Save_Enquiry_Business_Doc.md) — same flows, product/business
  perspective, bina code ke

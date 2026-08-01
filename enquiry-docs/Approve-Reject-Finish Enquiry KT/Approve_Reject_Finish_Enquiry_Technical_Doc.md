# Approve / Reject / Finish Enquiry — Technical Doc (Code-Level Deep Dive)

Yeh doc teen lifecycle-transition endpoints ka **technical implementation** cover karta hai —
routes, DB tables, queries, RabbitMQ, consumers, sab kuch code se verify karke. Business/product
perspective ke liye
[`Approve_Reject_Finish_Enquiry_Business_Doc.md`](./Approve_Reject_Finish_Enquiry_Business_Doc.md)
dekho — dono docs same flows cover karte hain, bas alag audience ke liye.

**Repos**: `enq-services-scm` (write API — all 3 controllers/models), `enq-consumers`
(downstream consumer that both *originates* waiting-state enquiries and *auto-calls* two of
these three APIs).

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file:line diya gaya
hai har jagah). Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — team se confirm karo]** likh diya hai, guess nahi kiya.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Approve controller (bind, validate, panic-recover, response shaping) | enq-services-scm | [`approveEnquiry.go`](../../internal/api/controllers/approveEnquiry.go) |
| Approve model — waiting-fetch, waiting-update, SP move-to-live, LMS push, duplicate-conflict handling | enq-services-scm | [`approveEnquiryModel.go`](../../internal/pkg/models/approveEnquiryModel/approveEnquiryModel.go) |
| Reject controller | enq-services-scm | [`rejectEnquiry.go`](../../internal/api/controllers/rejectEnquiry.go) |
| Reject model — waiting-fetch, reason-code remap, waiting-update, SP move-to-bounced, LMS push | enq-services-scm | [`rejectEnquiryModel.go`](../../internal/pkg/models/rejectEnquiryModel/rejectEnquiryModel.go) |
| Finish controller (distinct logger type `LogFormatKibanaEnquiry`, `nil`-code response shaping) | enq-services-scm | [`finishEnquiry.go`](../../internal/api/controllers/finishEnquiry.go) |
| Finish model — receiver lookup, sharded-table read, sender/attachment enrichment, dual queue push (LMS + comm), idempotency update | enq-services-scm | [`finishEnquiryModel.go`](../../internal/pkg/models/finishEnquiryModel/finishEnquiryModel.go) (977 lines) |
| Route registration | enq-services-scm | [`router.go:69-80`](../../internal/api/router/router.go) |
| **FENQ waiting-enquiry consumer** — origin of waiting-state enquiries (from Save Enquiry's `ENQUIRY_FENQ` queue), and the **automated caller of both Approve and Reject** via internal HTTP | enq-consumers | [`enq_post_fenq.go`](../../../enq-consumers/src/Enquiry/enq_post_fenq.go) |
| **Preferred-channel consumer** — automated caller of Finish Enquiry under one specific condition | enq-consumers | [`enq_preferred_channel.go:288-296,834-886`](../../../enq-consumers/src/Enquiry/enq_preferred_channel.go) |

---

## 2. Routes (confirmed from router.go)

| Method | Path | Controller |
|---|---|---|
| GET | `/enquiry/rejectEnquiry` | `controllers.RejectEnquiry` |
| GET | `/enquiry/rejectEnquiry/` | `controllers.RejectEnquiry` |
| POST | `/enquiry/rejectEnquiry/` | `controllers.RejectEnquiry` |
| POST | `/enquiry/rejectEnquiry` | `controllers.RejectEnquiry` |
| GET | `/enquiry/approveEnquiry` | `controllers.ApproveEnquiry` |
| GET | `/enquiry/approveEnquiry/` | `controllers.ApproveEnquiry` |
| POST | `/enquiry/approveEnquiry/` | `controllers.ApproveEnquiry` |
| POST | `/enquiry/approveEnquiry` | `controllers.ApproveEnquiry` |
| GET | `/enquiry/finishEnquiry/` | `controllers.FinishEnquiry` |
| GET | `/enquiry/finishEnquiry` | `controllers.FinishEnquiry` |
| POST | `/enquiry/finishEnquiry` | `controllers.FinishEnquiry` |
| POST | `/enquiry/finishEnquiry/` | `controllers.FinishEnquiry` |

[`router.go:69-80`](../../internal/api/router/router.go)

---

## 3. Data Model — Tables

> **Verification note**: yeh sab table/column names Go code ke andar embedded SQL strings se
> liye gaye hain (har row ke saamne file path diya hai). Live DB schema se cross-verify **nahi**
> kiya gaya hai — kisi migration/integration se pehle woh zaroor karo.

| Table / SP | Purpose | Key columns / params (code se) |
|---|---|---|
| `DIR_QUERY_WAITING` | The "waiting-review" record for an enquiry — both Approve and Reject read and update this table before moving the row elsewhere | `QUERY_ID`, `query_rcv_glusr_usr_id`, `fk_glusr_usr_id`, `iil_enquiry_type`, `fk_iil_whatsapp_attempt_id`, `FK_APPROVAL_REASON_ID`, `FK_BOUNCE_REASON_ID`, `ACTION_BY_EMP_ID`, `ACTION_DATE` — [`approveEnquiryModel.go:271,306`](../../internal/pkg/models/approveEnquiryModel/approveEnquiryModel.go), [`rejectEnquiryModel.go:251,285`](../../internal/pkg/models/rejectEnquiryModel/rejectEnquiryModel.go) |
| `SP_PUSH_ENQ_TO_DIR_QUERY_V3(query_id, emp_id, approval_reason)` (stored procedure) | Moves an enquiry from `DIR_QUERY_WAITING` into its live `DIR_QUERY_*` (sharded) table — the actual "approve" write. Returns a `dirQueryId` | [`approveEnquiryModel.go:367,379`](../../internal/pkg/models/approveEnquiryModel/approveEnquiryModel.go) — **table(s) written inside the SP are not visible from Go [INFERRED — confirm with DBA team]** |
| `sp_push_enq_to_dir_bounced_new_v2(query_id)` (stored procedure) | Moves an enquiry from `DIR_QUERY_WAITING` into `DIR_QUERY_BOUNCED` — the actual "reject" write | [`rejectEnquiryModel.go:320,332`](../../internal/pkg/models/rejectEnquiryModel/rejectEnquiryModel.go) — **same SP-opacity caveat as above** |
| `DIR_QUERY_MAP` | Lookup table used by Finish Enquiry to resolve which receiver GLID owns a given `QUERY_ID` (before it can figure out which sharded table to query) | `QUERY_ID`, `QUERY_RCV_GLUSR_USR_ID` — [`finishEnquiryModel.go:292`](../../internal/pkg/models/finishEnquiryModel/finishEnquiryModel.go) |
| `DIR_QUERY_<NN>` (sharded, `NN` = receiver GLID mod 100, zero-padded) | The actual live-enquiry record — Finish Enquiry reads full enquiry detail from here (joined with `GLUSR_USR`, `GL_CITY`, `IIL_MASTER_DATA`) and updates `DIR_QUERY_MAIL_SEND` here as the idempotency flag | [`finishEnquiryModel.go:174-175,339,442`](../../internal/pkg/models/finishEnquiryModel/finishEnquiryModel.go) |
| `GLUSR_USR` | Joined inline in Finish Enquiry's big `instantCommunication` SELECT (sender phone/address/name enrichment when not already on the enquiry row), and queried directly again in `GetSenderDetailPG` | [`finishEnquiryModel.go:339,538-554`](../../internal/pkg/models/finishEnquiryModel/finishEnquiryModel.go) |
| `GL_CITY` (x3 joins), `IIL_MASTER_DATA` (x5 joins), `GL_COUNTRY` | Lookup/master tables joined inline in the same `instantCommunication` query, resolving purchase-period/location/purpose/paymode/shipmode text and 3 named cities | [`finishEnquiryModel.go:339`](../../internal/pkg/models/finishEnquiryModel/finishEnquiryModel.go) |
| `DIR_QUERY_ATTACHMENT` | Enquiry attachments (images/docs), fetched by Finish Enquiry and bundled into the LMS payload | `DIR_QRY_ATTACHMENT_ID`, `FK_ATTACH_TYPE_ID`, `DIR_QRY_ATTACH_DOC_PATH`, `DIR_QRY_ATTACH_IMG_*`, `FK_DIR_QUERY_ID` — [`finishEnquiryModel.go:914-917`](../../internal/pkg/models/finishEnquiryModel/finishEnquiryModel.go) |

---

## 4. "Decode This Magic Value" — Codes Traced From The Code

### 4.1 Approve — hardcoded `ApprovalReason = "6"`

The controller/model **overwrites whatever the client sent** for `approval_reason_id` at the
very top of `ApproveEnquiryModel`:

```go
// approveEnquiryModel.go:62
inputParam.ApprovalReason = "6"
```

So regardless of the `InputParam.ApprovalReason` binding tag (`binding:"omitempty,numeric"`),
every approval performed through this endpoint is recorded with reason `6`.
**Exact business meaning of reason code `6` — [INFERRED, no enum/comment in code — confirm with
team].**

### 4.2 Reject — `reason_id` → internal bounce-reason remap

`ParameterValidation` in `rejectEnquiryModel.go` silently remaps a small set of incoming
`reason_id` values to a different internal ID before anything is written:

```go
// rejectEnquiryModel.go:359-372
reasonMap := map[int]int{
    1:  80,
    3:  81,
    7:  82,
    8:  83,
    9:  84,
    10: 85,
    11: 86,
}
reasonID := helper.ConvertToInt(inputParam.ReasonID)
if _, ok := reasonMap[reasonID]; ok {
    inputParam.ReasonID = reasonMap[reasonID]
}
```

Any `reason_id` **not** in this map (e.g. `13`, `-94` — both used by the FENQ auto-caller,
section 5) passes through **unmapped**, straight into `FK_BOUNCE_REASON_ID`.
**Business meaning of each side of this map (what `1`/`3`/`7`/`8`/`9`/`10`/`11` mean to a
caller vs. what `80`-`86` mean in the DB) — [INFERRED, no enum found in this codebase — confirm
with team].**

### 4.3 Reject — hardcoded system-triggered reason codes

Two reason IDs are hardcoded at call sites **outside** the reasonMap, used only by
system/automated callers:

| Value | Caller | Context |
|---|---|---|
| `-94` | `approveEnquiryModel.go:432` (`curlRejectEnquiry`, invoked from `setQueryId` on a duplicate-key DB error) | An approve attempt that hit a DB-level duplicate-key conflict is automatically redirected to Reject with this reason, as a safety-net (section 6, Flow D) |
| `13` | `enq_post_fenq.go:268` (`callRejectEnquiry`) | The FENQ bot's own abusive-content re-check reject reason (section 6, Flow C) |

**Exact business meaning of `-94` and `13` — [INFERRED — confirm with team; `-94` in particular
looks like a sentinel/negative marker rather than a real catalogued reason].**

### 4.4 Approve/Reject — `is_instant` (LMS `insertion_type`)

Both Approve and Reject read `IsInstant` from the request (default `0` if not supplied) and use
it purely to pick a different LMS `insertion_type` value — no other behavioral branch depends on
it:

| Endpoint | `is_instant == 0` → `insertion_type` | `is_instant != 0` → `insertion_type` |
|---|---|---|
| Approve | `"7"` [`approveEnquiryModel.go:163`](../../internal/pkg/models/approveEnquiryModel/approveEnquiryModel.go) | `"3"` [`approveEnquiryModel.go:174`](../../internal/pkg/models/approveEnquiryModel/approveEnquiryModel.go) |
| Reject | `"6"` [`rejectEnquiryModel.go:143`](../../internal/pkg/models/rejectEnquiryModel/rejectEnquiryModel.go) | `"4"` [`rejectEnquiryModel.go:154`](../../internal/pkg/models/rejectEnquiryModel/rejectEnquiryModel.go) |

**Exact meaning of `insertion_type` codes `3/4/6/7` on the LMS side — [INFERRED, not owned by
this repo — confirm with LMS team].**

### 4.5 Approve — `moveQueryPG` error-branch return codes

`setQueryId` (called when the move-to-live SP errors) returns a synthetic string code that
`moveQueryPG`/`ApproveEnquiryModel` interpret:

| Value | Meaning |
|---|---|
| `"-1"` | Duplicate-key conflict detected, and the enquiry was **successfully** redirected into the bounced table via an internal reject call — response is still `Success`/`200` |
| `"-2"` | Duplicate-key conflict detected, but the redirect-to-bounced call **also failed** — response is `Failure`/`500` |
| `"-3"` | The move-to-live SP call itself **timed out** (`deadline exceeded`/`canceling statement`/`timeout` substring match) — response is `Failure`/`500` |
| `"0"` | Some other, unclassified DB error — response is `Failure`/`500` |

[`approveEnquiryModel.go:424-447`](../../internal/pkg/models/approveEnquiryModel/approveEnquiryModel.go)

### 4.6 Finish Enquiry — `DIR_QUERY_MAIL_SEND` sentinel values

The idempotency-guard column is set to one of two hardcoded negative sentinels, chosen purely by
whether the request was SMS-flagged:

```go
// finishEnquiryModel.go:427-430
mailSend := -7
if emaiSms == "1" {
    mailSend = -9
}
```

`-7` = general/default finish (comm packet built); `-9` = SMS-only finish. The initial SELECT
(`instantCommunication`) only matches rows where `DIR_QUERY_MAIL_SEND IS NULL` — meaning **once
either sentinel is written, no future Finish call for that `QUERY_ID` will find/process the row
again.** [`finishEnquiryModel.go:339`](../../internal/pkg/models/finishEnquiryModel/finishEnquiryModel.go)
**Exact business meaning of `-7` vs `-9` as opposed to other possible `DIR_QUERY_MAIL_SEND`
values used elsewhere in the domain — [INFERRED — confirm with team].**

### 4.7 Employee ID `"17967"` — the automated-bot identity

Both `callApproveEnquiry` and `callRejectEnquiry` in `enq_post_fenq.go` (section 6, Flow C) pass
the literal string `"17967"` as `emp_id` — meaning every automated approve/reject performed by
the FENQ bot is recorded as if a specific employee performed it.
[`enq_post_fenq.go:268,284`](../../../enq-consumers/src/Enquiry/enq_post_fenq.go)
**Whether `17967` is a real/dedicated "system" employee account, or a shared/legacy ID —
[INFERRED — confirm with team, especially before using it in any audit/reporting logic that
assumes emp_id implies a human reviewer].**

---

## 5. Business Rules & Validation (code se exhaustive list)

1. **Approve: `query_id` and `emp_id` are mandatory, both must be numeric.**
   `ParameterValidation` returns early on the first failing check.
   [`approveEnquiryModel.go:408-423`](../../internal/pkg/models/approveEnquiryModel/approveEnquiryModel.go)
2. **Approve: `ApprovalReason` is unconditionally forced to `"6"`** regardless of what the
   client sent (section 4.1). [`approveEnquiryModel.go:62`](../../internal/pkg/models/approveEnquiryModel/approveEnquiryModel.go)
3. **Approve: a missing `Query Not Found` case (SQL `sql.ErrNoRows` on the `DIR_QUERY_WAITING`
   lookup) returns HTTP `204`→remapped-to-`200`, not an error** — the controller explicitly
   remaps `Response.Code==204` to `c.JSON(200, ...)`.
   [`approveEnquiry.go:76-79`](../../internal/api/controllers/approveEnquiry.go),
   [`approveEnquiryModel.go:96-99`](../../internal/pkg/models/approveEnquiryModel/approveEnquiryModel.go)
4. **Approve: the `DIR_QUERY_WAITING` row is updated (approval reason set, bounce reason
   cleared) BEFORE the move-to-live SP call** — meaning if the SP call subsequently fails, the
   waiting row is left in an "approved-but-not-moved" intermediate state until a retry succeeds
   or the duplicate-key fallback (section 4.5, `"-1"`) kicks in.
   [`approveEnquiryModel.go:112-130`](../../internal/pkg/models/approveEnquiryModel/approveEnquiryModel.go)
5. **Approve: on a duplicate-key DB error from the move-SP, the code (a) resets the waiting row
   back to unapproved** (`updateTablePG(..., flag=0)` — clears both approval and bounce reason)
   **and (b) makes an internal HTTP call to its own `rejectEnquiry` endpoint** with
   `emp_id="17967"`, `reason_id="-94"`, `is_instant=1` — moving the conflicting enquiry into the
   bounced table instead of leaving it stuck. [`approveEnquiryModel.go:424-491`](../../internal/pkg/models/approveEnquiryModel/approveEnquiryModel.go)
6. **Approve/move-SP call has a 2-second timeout in non-local environments** (1000s locally) —
   `deadline exceeded`/`canceling statement`/`timeout` substring match maps to code `"-3"`
   (section 4.5). [`approveEnquiryModel.go:369-378`](../../internal/pkg/models/approveEnquiryModel/approveEnquiryModel.go)
7. **Reject: `query_id`, `emp_id`, and `reason_id` are all mandatory**, all must be numeric.
   [`rejectEnquiryModel.go:341-358`](../../internal/pkg/models/rejectEnquiryModel/rejectEnquiryModel.go)
8. **Reject: `reason_id` is silently remapped** via the 7-entry `reasonMap` (section 4.2) before
   any DB write — values outside the map pass through unchanged.
9. **Reject: the `DIR_QUERY_WAITING` UPDATE has an explicit `FK_APPROVAL_REASON_ID IS NULL`
   guard** — an already-approved enquiry's waiting row will not be touched by a subsequent
   reject call (the `UPDATE` simply matches zero rows; the code does not check
   `RowsAffected` and proceeds to call the bounce-move SP regardless, meaning the bounce-move SP
   itself is the actual authority on whether the reject "sticks" — **[INFERRED: SP-side
   behavior when the corresponding waiting row was never updated is not visible from Go —
   confirm with DBA team]**). [`rejectEnquiryModel.go:279-311`](../../internal/pkg/models/rejectEnquiryModel/rejectEnquiryModel.go)
10. **Reject: a missing `Query Not Found` case also returns `204`→`200`**, same pattern as
    Approve. [`rejectEnquiry.go:77-79`](../../internal/api/controllers/rejectEnquiry.go)
11. **Finish: `query_id` must be present and numeric.**
    [`finishEnquiryModel.go:879-889`](../../internal/pkg/models/finishEnquiryModel/finishEnquiryModel.go)
12. **Finish: `query_destination` must be `"1"` or empty (defaults to `"1"`)** — any other value
    returns `"query_destination is wrong"` before any DB call.
    [`finishEnquiryModel.go:130-132,890-893`](../../internal/pkg/models/finishEnquiryModel/finishEnquiryModel.go)
13. **Finish: receiver GLID resolution failure (`sql.ErrNoRows` on `DIR_QUERY_MAP`) returns
    `200`/`"failure"`**, not a hard error code — treated as a graceful "nothing to do" outcome.
    [`finishEnquiryModel.go:165-172`](../../internal/pkg/models/finishEnquiryModel/finishEnquiryModel.go)
14. **Finish: the sharded target table is computed from the receiver GLID's last two digits**
    (`glid % 100`, zero-padded below 10) — `DIR_QUERY_00` through `DIR_QUERY_99`.
    [`finishEnquiryModel.go:174-175,409-418`](../../internal/pkg/models/finishEnquiryModel/finishEnquiryModel.go)
15. **Finish: the core SELECT only matches rows where `DIR_QUERY_MAIL_SEND IS NULL`** — this is
    the idempotency guard (section 4.6). A second Finish call for the same `QUERY_ID` after the
    first succeeded will find zero rows and return `"Data not found in query"` with code `203`
    (remapped to `200` by the controller).
    [`finishEnquiryModel.go:339,406`](../../internal/pkg/models/finishEnquiryModel/finishEnquiryModel.go)
16. **Finish: sender/receiver field enrichment falls back to `GLUSR_USR` only when the enquiry
    row's own sender fields are empty** (`COALESCE`-based in the big join query, and again via
    explicit `getValue()` Go-side fallback logic against a second `GLUSR_USR` fetch in
    `GetSenderDetailPG`) — meaning sender data can come from either the enquiry snapshot or a
    live user-profile lookup, whichever is populated.
    [`finishEnquiryModel.go:339,653-665`](../../internal/pkg/models/finishEnquiryModel/finishEnquiryModel.go)
17. **Finish: the communication-data build (`redisData` map — see section 9 naming caveat) is
    skipped entirely if `email_sms_comm == "1"`** — only the LMS push happens in that case.
    [`finishEnquiryModel.go:722-780`](../../internal/pkg/models/finishEnquiryModel/finishEnquiryModel.go)
18. **Finish: `getReceiverId` and `instantCommunication` both use different timeout tiers**
    (`local`=1000s/10s, `dev`/`stg`=10s for receiver lookup specifically, everything else 2s) —
    `dev`/`stg` getting a distinct 10s tier for the receiver-lookup query (but not the main
    select) is an asymmetry worth knowing when debugging environment-specific latency.
    [`finishEnquiryModel.go:294-304`](../../internal/pkg/models/finishEnquiryModel/finishEnquiryModel.go)

---

## 6. End-to-End Technical Flows

### Flow A — Manual Approve (happy path)

```
Admin panel / caller
    │
    ▼
[API] POST /enquiry/approveEnquiry  {query_id, emp_id, is_instant, approval_reason_id(ignored)}
    │  approveEnquiry.go: bind → ParameterValidation → ApproveEnquiryModel
    ▼
[MODEL] ApproveEnquiryModel()
    │  1. ApprovalReason forced to "6" (section 4.1)
    │  2. dbConn() — enquiry-write Postgres
    │  3. getQueryDetailPG(): SELECT receiver/sender/enqType/attemptId FROM DIR_QUERY_WAITING
    │     WHERE QUERY_ID=$1  → sql.ErrNoRows → 204/"Query Not Found" (early return)
    │  4. updateTablePG(flag=1): UPDATE DIR_QUERY_WAITING
    │     SET FK_APPROVAL_REASON_ID=6, FK_BOUNCE_REASON_ID=NULL, ACTION_BY_EMP_ID, ACTION_DATE=NOW()
    │  5. moveQueryPG(): SELECT * FROM SP_PUSH_ENQ_TO_DIR_QUERY_V3(query_id, emp_id, "6")
    │     → returns dirQueryId (the live enquiry's new ID)
    ▼
[QUEUE] PushIntoLMSQueue() → RabbitMQ R_MESSAGE_CENTER_BIZFEED_LMS<0-4>, insertion_type 3 or 7
    ▼
Client receives: {status:Success, CODE:200, query_id, dir_query_id, ...}
```

### Flow B — Manual Reject (happy path)

```
Admin panel / caller
    │
    ▼
[API] POST /enquiry/rejectEnquiry  {query_id, emp_id, reason_id, is_instant}
    │  rejectEnquiry.go: bind → ParameterValidation (reason_id remap, section 4.2) → RejectEnquiryModel
    ▼
[MODEL] RejectEnquiryModel()
    │  1. dbConn()
    │  2. getQueryDetailPG(): SELECT ... FROM DIR_QUERY_WAITING WHERE QUERY_ID=$1
    │     → sql.ErrNoRows → 204/"Query Not Found"
    │  3. updateTablePG(): UPDATE DIR_QUERY_WAITING SET FK_BOUNCE_REASON_ID=$1, ACTION_BY_EMP_ID, ACTION_DATE=NOW()
    │     WHERE QUERY_ID=$3 AND FK_APPROVAL_REASON_ID IS NULL
    │  4. moveQueryPG(): SELECT * FROM sp_push_enq_to_dir_bounced_new_v2(query_id) → bounceId
    ▼
[QUEUE] PushIntoLMSQueue() → RabbitMQ R_MESSAGE_CENTER_BIZFEED_LMS<0-4>, insertion_type 4 or 6
    ▼
Client receives: {status:Success, CODE:200, query_id, query_bounce_id, ...}
```

### Flow C — Automated FENQ-bot Approve/Reject (the dominant real-world path)

```
[CONSUME] enq_post_fenq.go on queue ENQUIRY_FENQ (populated by Save Enquiry's ENQ_DATA fan-out
           when a new enquiry lands in waiting state — see Save Enquiry KT, Flow B)
    │
    ▼
IN_FK_APPROVAL_REASON_ID == 2 (i.e. this message represents a genuinely-waiting enquiry)?
    │  YES
    ▼
time.Sleep(10 seconds)  ── settle buffer before re-check
    ▼
updateDirQueryWaiting() + isValidEnquiry() sanity checks
    ▼
valid == 1?
    │  YES
    ▼
isAbusiveKeywordFound(S_DESC) AND isAbusiveKeywordFound(PRODUCT_NAME)
    │  (external BI/abuse-detection HTTP call each, empty text short-circuits to "approve")
    │
    ├─ either returns "reject" → callRejectEnquiry(query_id, emp_id="17967", reason_id=13)
    │      → HTTP POST to THIS SAME rejectEnquiry API (Flow B, internally)
    │      → is_instant_rejected_approved=1, processing stops here (d.Ack, return)
    │
    └─ both return "approve" → callApproveEnquiry(query_id, emp_id="17967")
           → HTTP POST to THIS SAME approveEnquiry API (Flow A, internally)
           → response's query_id is parsed out and REPLACES mapString["QUERY_ID"]
             for the rest of this consumer's processing (approve can return a new/moved ID)
    ▼
startFenqProcess() — separate scoring logic computes a numeric bounce reason_id
   (duplicate-check / quality-scoring, not traced in this doc's scope)
    ▼
reason_id > 0 AND IN_FK_APPROVAL_REASON_ID == 2?
    │  YES → callRejectEnquiry(query_id, emp_id="17967", reason_id) — a SECOND, independent
    │         reject opportunity, this time with a system-computed reason rather than 13
    ▼
callBlRejectionService(), callTransferIsq()/callTransferAttachment() (if offer_id present),
pushToMessageCenter() (only if still unresolved) — all out of scope for this doc
```

[`enq_post_fenq.go:211-354,662-799`](../../../enq-consumers/src/Enquiry/enq_post_fenq.go)

**Key insight**: the vast majority of Approve/Reject invocations in production are almost
certainly **this automated path**, not the manual admin-panel path (Flow A/B) — the same two
APIs serve both a human-triggered and a bot-triggered caller, indistinguishable at the API
layer except by `emp_id == "17967"`.

### Flow D — Duplicate-key safety net (inside Approve)

```
[MODEL] ApproveEnquiryModel() → moveQueryPG() → SP_PUSH_ENQ_TO_DIR_QUERY_V3 fails
    │
    ▼
setQueryId(err) inspects err.Error() for "duplicate key value violates unique constraint"
    │  MATCH
    ▼
updateTablePG(flag=0) — resets DIR_QUERY_WAITING back to un-approved/un-bounced
    ▼
curlRejectEnquiry(query_id, "17967", "-94") — internal HTTP POST to /enquiry/rejectEnquiry
    │  (env-specific URL: dev-enq2.intermesh.net / enqphp.intermesh.net / enq2.intermesh.net)
    ▼
Success → id="-1" → ApproveEnquiryModel returns Success/200 "Duplicate Entry moved to Bounced Table"
Failure → id="-2" → ApproveEnquiryModel returns Failure/500 "Duplicate Entry Insertion failed to Bounced Table"
```

[`approveEnquiryModel.go:424-491`](../../internal/pkg/models/approveEnquiryModel/approveEnquiryModel.go)

### Flow E — Finish Enquiry (idempotent communication trigger)

```
Caller (admin panel, or automated — see Flow F)
    │
    ▼
[API] POST /enquiry/finishEnquiry  {query_id, query_destination(default 1), modid, email_sms_comm(default 0)}
    │  finishEnquiry.go: bind → ParameterValidation (query_destination must be "1"/"") → FinishEnquiryModel
    ▼
[MODEL] FinishEnquiryModel()
    │  1. dbConn()
    │  2. getReceiverId(): SELECT QUERY_RCV_GLUSR_USR_ID FROM DIR_QUERY_MAP WHERE QUERY_ID=$1
    │     → sql.ErrNoRows → 200/"failure" (nothing to finish)
    │  3. dirTableName = "DIR_QUERY_" + last-two-digits(receiverGlid)
    │  4. instantCommunication():
    │     big multi-join SELECT from dirTableName + IIL_MASTER_DATA(x5) + GL_CITY(x3) + GLUSR_USR
    │     WHERE QUERY_ID=$1 AND DIR_QUERY_MAIL_SEND IS NULL  ← idempotency guard
    │     → 0 rows → code 203 ("Data not found in query", i.e. already finished or never existed)
    │     → 1 row found and non-empty →
    │        ├─ ProcessingMailQueue(result):
    │        │    GetSenderDetailPG() (GLUSR_USR fallback fetch) + getEnquiryAttachment()
    │        │    (DIR_QUERY_ATTACHMENT), then 2 PARALLEL goroutines:
    │        │    ├─ (if email_sms_comm != "1") build comm-data map → pushEnquiryCommDataRabbit()
    │        │    │   → RabbitMQ queue "enquiry.comm", exchange "ENQUIRY.topic"
    │        │    └─ PushIntoLMSQueue() → RabbitMQ R_MESSAGE_CENTER_BIZFEED_LMS<0-4>, insertion_type "1"
    │        └─ updateDirQuery(): UPDATE dirTableName SET DIR_QUERY_MAIL_SEND = -7 or -9
    │           WHERE QUERY_ID=$2 AND query_rcv_glusr_usr_id=$3
    ▼
Client receives: {RESP:"success"/"failure", MESSAGE, CODE(stripped before send — see section 14)}
```

### Flow F — Automated Finish Enquiry trigger (from Save Enquiry's downstream)

```
[CONSUME] enq_preferred_channel.go on queue ENQ_DATA (Save Enquiry's primary fan-out —
           see Save Enquiry KT, Flow A)
    │
    ▼
Personalization_Queue_Name == "ENQ_PERSONALIZATION_OTHER"
    AND QueryDestinationId == "1"
    AND personalization payload's 4th q_data element == "leap@indiamart.com"
    │  ALL MATCH
    ▼
callFinishEnquiry(query_id, modid, "1") → HTTP POST to THIS SAME /enquiry/finishEnquiry API
    (env-resolved via utils.String("finishEnquiry", ""))
```

[`enq_preferred_channel.go:266-296,834-886`](../../../enq-consumers/src/Enquiry/enq_preferred_channel.go)

**Cross-check against Save Enquiry KT**: the Save Enquiry Technical Doc (section 7) already
documents this exact trigger — *"when `enq_preferred_channel.go` sees
`Personalization_Queue_Name == "ENQ_PERSONALIZATION_OTHER"` and `QueryDestinationId == "1"` and
the personalization payload's 4th q_data element equals `leap@indiamart.com`, it makes an
outbound HTTP call to the `finishEnquiry` API"*. This review confirms it is **the same endpoint**
documented here (`FinishEnquiryModel` in `finishEnquiryModel.go`) — **no discrepancy found**
between the two docs on this point.

---

## 7. Flow-wise DB & Table Usage

### Flow A — Manual Approve

| # | DB | Table / SP | Operation | Why |
|---|---|---|---|---|
| 1 | enquiry-write Postgres | `DIR_QUERY_WAITING` | SELECT | Fetch receiver/sender/enqType/attemptId for the LMS push payload and to confirm the row exists |
| 2 | enquiry-write Postgres | `DIR_QUERY_WAITING` | UPDATE | Set approval reason `6`, clear bounce reason, stamp acting employee + timestamp |
| 3 | enquiry-write Postgres | `SP_PUSH_ENQ_TO_DIR_QUERY_V3` | Stored-procedure CALL | Move the enquiry row into its live `DIR_QUERY_*` shard |
| 4 | RabbitMQ | `R_MESSAGE_CENTER_BIZFEED_LMS<0-4>` | Publish | LMS transaction record |

**Duplicate-conflict sub-path (Flow D) adds**: another `DIR_QUERY_WAITING` UPDATE (reset) +
1 outbound HTTP call to Reject Enquiry's own full DB chain (Flow B, below).

### Flow B — Manual Reject

| # | DB | Table / SP | Operation | Why |
|---|---|---|---|---|
| 1 | enquiry-write Postgres | `DIR_QUERY_WAITING` | SELECT | Same fetch as Approve |
| 2 | enquiry-write Postgres | `DIR_QUERY_WAITING` | UPDATE (guarded by `FK_APPROVAL_REASON_ID IS NULL`) | Set bounce reason, stamp employee + timestamp |
| 3 | enquiry-write Postgres | `sp_push_enq_to_dir_bounced_new_v2` | Stored-procedure CALL | Move the enquiry row into `DIR_QUERY_BOUNCED` |
| 4 | RabbitMQ | `R_MESSAGE_CENTER_BIZFEED_LMS<0-4>` | Publish | LMS transaction record |

### Flow C — Automated FENQ Bot

| # | DB / Service | Operation | Why |
|---|---|---|---|
| 1-2 | (consumer-side, `enqPgConn`) | `updateDirQueryWaiting()`, `isValidEnquiry()` | Pre-checks before deciding to re-scan (not traced in full in this doc's scope) |
| 3-4 | External HTTP (BI/abuse service) | Content re-scan (description, product-name) | Determine approve vs. reject |
| 5 | **Full Flow A or Flow B DB chain** (via internal HTTP call) | — | The bot doesn't touch `DIR_QUERY_WAITING`/`SP_PUSH_ENQ_TO_DIR_QUERY_V3`/`sp_push_enq_to_dir_bounced_new_v2` directly — it goes through the same live API, so all of Flow A's or Flow B's DB round-trips happen identically |
| 6 | (consumer-side) | `startFenqProcess()`, `update_fenq_reason_id_in_dir_queryPG()`, etc. | Independent scoring/bookkeeping — out of scope for this doc, but can trigger a **second** Reject call (Flow B again) if a bounce `reason_id > 0` is computed |

### Flow D — Duplicate-key Safety Net

| # | DB | Table / SP | Operation | Why |
|---|---|---|---|---|
| 1 | enquiry-write Postgres | `DIR_QUERY_WAITING` | UPDATE (flag=0 reset) | Undo the flag=1 update from step 2 of Flow A |
| 2 | (via internal HTTP call) | Full Flow B chain | — | The conflicting enquiry is rejected instead, with reason `-94` |

### Flow E — Finish Enquiry

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1 | enquiry Postgres | `DIR_QUERY_MAP` | SELECT | Resolve receiver GLID → determine which sharded table to query |
| 2 | enquiry Postgres | `DIR_QUERY_<NN>` (+ `IIL_MASTER_DATA`x5, `GL_CITY`x3, `GLUSR_USR` inline joins) | SELECT | Full enquiry detail fetch, guarded by `DIR_QUERY_MAIL_SEND IS NULL` |
| 3 | enquiry Postgres | `GLUSR_USR` | SELECT (`GetSenderDetailPG`) | Fallback sender enrichment (name/email/company/address/mobile/phone) |
| 4 | enquiry Postgres | `DIR_QUERY_ATTACHMENT` | SELECT | Attachment list for the LMS payload |
| 5 | RabbitMQ | `enquiry.comm` (exchange `ENQUIRY.topic`) | Publish (conditional, skipped if `email_sms_comm=="1"`) | Full communication-data packet for downstream email/SMS |
| 6 | RabbitMQ | `R_MESSAGE_CENTER_BIZFEED_LMS<0-4>` | Publish | LMS transaction record, `insertion_type="1"` |
| 7 | enquiry Postgres | `DIR_QUERY_<NN>` | UPDATE | Set `DIR_QUERY_MAIL_SEND = -7` or `-9` — idempotency flag |

**Total round-trips for a single Finish: 4 SELECTs + 1 UPDATE across enquiry Postgres, plus 2
RabbitMQ publishes (1 conditional)** — comparable in shape to Save Enquiry's Flow A footprint,
though against different tables.

---

## 8. RabbitMQ — Queues Touched

| Queue / Exchange | Publisher | Purpose |
|---|---|---|
| `R_MESSAGE_CENTER_BIZFEED_LMS<0-4>` (5-way sharded, random modulus), exchange `lms-gcp.topic` | Approve (`PushIntoLMSQueue`), Reject (`PushIntoLMSQueue`), Finish (`PushIntoLMSQueue`) — **three separate, near-identical implementations of the same LMS push exist**, one per model file, using different `insertion_type` conventions per endpoint (section 4.4) | Lead-Management transaction intake — same queue family Save Enquiry's `enq_pns_lms.go` consumer already listens on |
| `enquiry.comm`, exchange `ENQUIRY.topic` | Finish only (`pushEnquiryCommDataRabbit`) | Full communication-data packet — **downstream consumer of this specific queue not traced in this review [INFERRED — likely an email/SMS-sending service, confirm with team]** |
| — (internal HTTP, not a queue) | Approve's duplicate-key fallback → Reject; FENQ bot → Approve/Reject; preferred-channel consumer → Finish | These are **synchronous HTTP calls between services**, not RabbitMQ messages — flagged here because they are easy to mistake for queue-driven triggers when tracing "what calls Approve/Reject/Finish" |

No evidence of any of these three endpoints publishing to `ENQ_DATA`, `ENQUIRY_FENQ`, or
`ENQ_PERSONALIZATION_OTHER` (Save Enquiry's queues) — they are strictly downstream consumers of
that pipeline, not participants in it.

---

## 9. Kafka

**No Kafka usage found in any of the three controllers or models** (`approveEnquiryModel.go`,
`rejectEnquiryModel.go`, `finishEnquiryModel.go`) — confirmed via targeted grep for
`kafka`/`Kafka` across `enq-services-scm`, no matches. All three endpoints' own queue
interactions are RabbitMQ-only (section 8).

**Straight answer if asked "does Approve/Reject/Finish use Kafka":** no, none of the three do,
directly or via a queue push from within these models. (Their upstream trigger, Save Enquiry's
downstream consumers, does use Kafka for unrelated purposes — see Save Enquiry KT section 8 —
but that is not part of these three endpoints' own execution path.)

---

## 10. Redis

**No real Redis usage found in `approveEnquiryModel.go`, `rejectEnquiryModel.go`, or
`finishEnquiryModel.go`.** A repo-wide grep for `redis`/`Redis` across `enq-services-scm` only
hits `finishEnquiryModel.go:721` (`var redisData map[string]string`) — this is a **local Go
variable name**, not a Redis client call; it's a plain in-memory map used to build the
`enquiry.comm` RabbitMQ payload (section 6, Flow E). This is the exact same false-positive
pattern the Save Enquiry Technical Doc already flagged for `QASapprovedMailer/main.go`'s
`redisData` variable — confirmed consistent, not a contradiction.

All three endpoints hit Postgres directly on every request, with no caching layer.

---

## 11. Optimization Scope — DB/Response-Time Contributors

### High-impact

1. **Approve performs its `DIR_QUERY_WAITING` UPDATE (flag=1) before confirming the move-to-live
   SP will succeed.** If the SP call subsequently fails (timeout, unclassified error), the
   waiting row is left "approved but not moved" with no automatic rollback of that first UPDATE
   (only the duplicate-key branch explicitly resets it, section 5.5) — a non-duplicate-key
   failure leaves the row in an inconsistent intermediate state until a retry.
   [`approveEnquiryModel.go:112-130`](../../internal/pkg/models/approveEnquiryModel/approveEnquiryModel.go).
   **Suggestion**: consider wrapping steps 4-5 (waiting-update + move-SP) in a single DB
   transaction, or reordering so the waiting-update only commits after the move-SP succeeds.
2. **Three near-duplicate `PushIntoLMSQueue` implementations exist** (one per model file,
   `approveEnquiryModel.go`, `rejectEnquiryModel.go`, `finishEnquiryModel.go`) with slightly
   different signatures and `insertion_type` conventions — any future change to the LMS-push
   pattern (retry logic, timeout, queue-selection modulus) has to be applied in **three places**
   independently, a maintenance-cost risk more than a runtime-perf one.
3. **`instantCommunication`'s SELECT (Finish Enquiry) joins across 5 `IIL_MASTER_DATA` subquery
   joins + 3 `GL_CITY` joins + 1 `GLUSR_USR` join in a single query**, against a *sharded* table
   whose name is built from string concatenation
   (`"SELECT ... FROM " + tableName + " WHERE ..."`) — this is a wide, heavy read on every
   Finish call. **Suggestion**: verify each shard has appropriate indexes on `QUERY_ID` +
   `DIR_QUERY_MAIL_SEND`, since the guard clause (`DIR_QUERY_MAIL_SEND IS NULL`) is doing real
   filtering work here. [`finishEnquiryModel.go:339`](../../internal/pkg/models/finishEnquiryModel/finishEnquiryModel.go)

### Medium-impact

4. **No caching on any of the three endpoints' reads** — `DIR_QUERY_WAITING` lookups (Approve/
   Reject) and the multi-join Finish SELECT all hit Postgres live every time. Given the FENQ bot
   (Flow C) is likely the dominant caller and processes enquiries close together in time after
   Save Enquiry, there's no obvious hot-key reuse pattern here (unlike GST's rarely-changing
   verified records) — this is a lower-priority gap than GST's equivalent finding.
5. **`getLastTwoDigits` sharding logic is duplicated inline** rather than shared as a common
   helper referenced from a single place — [`finishEnquiryModel.go:409-419`](../../internal/pkg/models/finishEnquiryModel/finishEnquiryModel.go)
   is the only definition found in this review's scope, but if Save Enquiry or other endpoints
   independently compute the same shard-routing logic elsewhere, a drift risk exists if the
   sharding scheme (currently mod-100) ever changes.
6. **Approve's internal duplicate-key fallback makes a full HTTP round-trip to its own sibling
   endpoint** (`curlRejectEnquiry`, section 6 Flow D) rather than calling `RejectEnquiryModel`
   directly in-process — this adds an unnecessary network hop (with its own 2s/20s timeout tier)
   for a scenario that's already an error-recovery path, compounding the original failure's
   latency. [`approveEnquiryModel.go:448-491`](../../internal/pkg/models/approveEnquiryModel/approveEnquiryModel.go)

### Low-impact / good-practice already present

7. **Positive finding**: Finish Enquiry's comm-data build and LMS push run in **parallel
   goroutines** (`sync.WaitGroup`) inside `ProcessingMailQueue`, not sequentially.
   [`finishEnquiryModel.go:720-787`](../../internal/pkg/models/finishEnquiryModel/finishEnquiryModel.go)
8. **Positive finding**: Finish Enquiry's idempotency guard (`DIR_QUERY_MAIL_SEND IS NULL`) is
   enforced at the SQL `WHERE` clause level, not just application logic — a genuinely safe,
   race-condition-resistant pattern for "don't send this communication twice."

---

## 12. Cron Inventory

No cron job in `enq-services-scm/pkg/crons/` directly references `approveEnquiry`,
`rejectEnquiry`, or `finishEnquiry` by name — confirmed via targeted grep across that directory.
`EnquiryMailer/main.go` and `QASapprovedMailer/main.go` are present in the crons directory and
their published `SERVICENAME` values (`EnquiryMailerCron`, `QASapprovedMailerCron`) are
recognized by `enq_preferred_channel.go`'s BL-Transfer-ISQ gate (same gate documented in Save
Enquiry KT section 7) — but **neither cron's own source was found to call any of these three
endpoints directly in this review's scope**; their relationship to Approve/Reject/Finish is only
via that shared consumer-side `SERVICENAME` check, which is unrelated to this doc's three APIs.
**[INFERRED — if either mailer cron is later found to trigger Approve/Reject/Finish indirectly,
this section should be updated.]**

---

## 13. Edge Cases & Gotchas (technical POV)

1. **Approve and Reject both return HTTP `204`/`Success` on "Query Not Found," but the
   controller then remaps that specific code to `200` before sending** — a caller checking for
   HTTP `204` directly (rather than inspecting the response body) will never see it; it's always
   `200` on the wire. [`approveEnquiry.go:76-79`](../../internal/api/controllers/approveEnquiry.go),
   [`rejectEnquiry.go:77-79`](../../internal/api/controllers/rejectEnquiry.go)
2. **Finish Enquiry's controller uses a *different* logger type** (`LogFormatKibanaEnquiry` /
   `CentralizedKibanaLoggingEnquiry()`) than Approve/Reject (`LogFormatKibana` /
   `CentralizedKibanaLogging()`) — includes extra fields (`LMSQueueData`, `CommQueueData`,
   `ModId`, `QueryId`) not present in the other two. Anyone building cross-endpoint log
   dashboards needs to account for this schema difference.
   [`finishEnquiry.go:36,54,76,91`](../../internal/api/controllers/finishEnquiry.go)
3. **Finish Enquiry's `Response.Code` is explicitly set to `nil` before every JSON response**,
   with the actual HTTP status code computed separately (`code := helper.ConvertToInt(Response.Code)`
   captured **before** the nil-out) — meaning the JSON body's `CODE` field is always empty/omitted
   for successful and validation-error paths, unlike Approve/Reject which keep `CODE` in the body.
   [`finishEnquiry.go:58,72,99-101`](../../internal/api/controllers/finishEnquiry.go)
4. **The FENQ bot can call Reject *twice* for the same message** (section 6, Flow C) — once from
   the immediate abusive-content re-check (reason `13`), and again later from
   `startFenqProcess`'s scoring output (`reason_id`) if the first branch didn't already resolve
   it (`is_instant_rejected_approved` guards against literal double-firing within one message,
   but the two branches use different reason codes and different code paths, worth knowing when
   auditing why an enquiry has a particular bounce reason).
5. **Approve's response body is stripped of most fields right before sending**
   (`Response.QueryID`, `DbTime`, `EnqStatusCode`, `EnquqeTime`, `EnqueStatus`,
   `DIR_QUERY_BOUNCED_ID`, `DIR_QUERY_ID` are all explicitly blanked in the controller after
   logging) — meaning despite the model computing and returning a real `dir_query_id`, **the
   controller discards it before the client ever sees it** in the current code.
   [`approveEnquiry.go:68-75`](../../internal/api/controllers/approveEnquiry.go) — this looks like
   it may be unintentional (a debug-cleanup that went too far) since the model clearly computes
   these values for a reason. **Flag for team: confirm whether stripping `dir_query_id` from the
   response is intentional or a regression.**
6. **Reject's controller strips the same set of fields**, including `Response.QueryID` and
   `Response.BounceID` — same caveat as above.
   [`rejectEnquiry.go:70-76`](../../internal/api/controllers/rejectEnquiry.go)
7. **`ApprovalReason` being hardcoded to `"6"` means the `binding:"omitempty,numeric"` tag on
   `InputParam.ApprovalReason` is effectively dead validation** — any value (or lack of one) the
   client sends for `approval_reason_id` is discarded before it can matter.
8. **Reject's `reasonMap` (section 4.2) only covers 7 specific input values** — anything else
   (including the hardcoded system values `13` and `-94`) passes through unmapped directly into
   `FK_BOUNCE_REASON_ID`. A caller assuming *all* reason IDs go through this normalization would
   be wrong for system-triggered rejects.

---

## 14. Open Questions

1. **Exact business meaning of Approve's hardcoded reason `"6"`** (section 4.1) — no comment or
   enum in code.
2. **Exact business meaning of Reject's `reasonMap` values** (`1→80, 3→81, 7→82, 8→83, 9→84,
   10→85, 11→86`, section 4.2) — both sides of the mapping.
3. **Exact business meaning of hardcoded system reject reasons `-94`** (duplicate-key fallback)
   and `13` (FENQ bot's abusive-content reject) — section 4.3.
4. **Exact meaning of LMS `insertion_type` codes `1, 3, 4, 6, 7`** across the three endpoints —
   owned by the LMS team, not visible from this codebase.
5. **What table(s) do `SP_PUSH_ENQ_TO_DIR_QUERY_V3` and `sp_push_enq_to_dir_bounced_new_v2`
   actually write to** — both are black-box stored procedures from the Go code's perspective,
   same caveat as Save Enquiry's `sp_insert_iil_enquiry_v32` and GST's SP-based writes.
6. **Whether employee ID `17967` is a dedicated, actively-maintained "system bot" account** or a
   legacy/shared ID — section 4.7.
7. **Downstream consumer of the `enquiry.comm` RabbitMQ queue** (exchange `ENQUIRY.topic`) —
   not traced in this review; presumably an email/SMS-dispatch service outside the two repos in
   scope.
8. **Whether stripping `dir_query_id`/`query_id`/`query_bounce_id` from Approve's and Reject's
   final HTTP response (section 13, items 5-6) is intentional** — worth a direct confirmation
   with the team since it looks like it could be an accidental regression given the model layer
   clearly computes and threads these values through for a reason.
9. **Reject's `FK_APPROVAL_REASON_ID IS NULL` UPDATE guard (section 5.9) behavior when it
   matches zero rows** — the code doesn't check `RowsAffected` and proceeds to call the
   bounce-move SP regardless; whether the SP itself has an equivalent guard, or whether an
   already-approved enquiry could still get incorrectly moved to bounced via this endpoint, is
   not visible from the Go code.
10. **Live DB schema verification** (column types, nullability, indexes, constraints) for
    `DIR_QUERY_WAITING`, `DIR_QUERY_MAP`, `DIR_QUERY_<NN>` shards, and `DIR_QUERY_ATTACHMENT` —
    this doc only reflects what the Go SQL strings imply.

---

## See also

- [`Approve_Reject_Finish_Enquiry_Business_Doc.md`](./Approve_Reject_Finish_Enquiry_Business_Doc.md) — same flows, product/business perspective, bina code ke
- [`../Save Enquiry KT/Save_Enquiry_Technical_Doc.md`](../Save%20Enquiry%20KT/Save_Enquiry_Technical_Doc.md) — the upstream flow that creates the waiting-state enquiries these three APIs act on, and the source of both automated callers (`enq_post_fenq.go` via `ENQUIRY_FENQ`, `enq_preferred_channel.go` via `ENQ_DATA`)

# Call Enquiry — Technical Doc (Code-Level Deep Dive)

Yeh doc 4 closely-related call-based enquiry endpoints ka **technical implementation** cover
karta hai — routes, DB tables, queries, RabbitMQ, downstream Kafka, sab kuch code se verify
karke. Business/product perspective ke liye
[`Call_Enquiry_Business_Doc.md`](./Call_Enquiry_Business_Doc.md) dekho.

**Repos**: `enq-services-scm` (write API — all 4 controllers/models), `enq-consumers` (shared
downstream LMS consumer that also serves Save Enquiry / Approve-Reject-Finish Enquiry).

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file:line diya gaya
hai har jagah). Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — team se confirm karo]** likh diya hai, guess nahi kiya.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Identified-call controller (bind, ModID/GLID validation, rate-limit-429 handling) | enq-services-scm | [`callEnquiry.go`](../../internal/api/controllers/callEnquiry.go) |
| Identified-call model — insert/update `C2C_RECORDS`, `iil_call_mapping`, bsmapping rate-limit check, `findCallType` auto-context-detection, LMS push | enq-services-scm | [`callEnquiryModel.go`](../../internal/pkg/models/callEnquiryModel/callEnquiryModel.go) |
| Unidentified-call controller | enq-services-scm | [`CallEnquiryUnidentified.go`](../../internal/api/controllers/CallEnquiryUnidentified.go) |
| Unidentified-call model — insert/update `C2C_RECORDS_UNIDENTIFIED`, PNS-probable-number update, App-type update that internally chains into `callEnquiryModel.CallEnquiryModel` | enq-services-scm | [`CallEnquiryUnidentifiedModel.go`](../../internal/pkg/models/callEnquiryUnidentified/CallEnquiryUnidentifiedModel.go) |
| Unidentified-call **read/lookup** controller (despite `POST` route, this is a read-only fetch) | enq-services-scm | [`unidentifiedC2C.go`](../../internal/api/controllers/unidentifiedC2C.go) |
| Unidentified-call read model — by GA-cookie or by receiver GLID (last-5-min unclaimed calls) | enq-services-scm | [`UnidentifiedC2CModel.go`](../../internal/pkg/models/unidentifiedC2C/UnidentifiedC2CModel.go) |
| PNS-linking controller (state/timestamp validation) | enq-services-scm | [`linkC2CPNS.go`](../../internal/api/controllers/linkC2CPNS.go) |
| PNS-linking model — `iil_call_mapping` insert/update logic for pre-call vs post-call states, LMS push | enq-services-scm | [`linkC2CModel.go`](../../internal/pkg/models/linkC2CPNSModel/linkC2CModel.go) |
| Route registration | enq-services-scm | [`router.go:60-63,88-91,95`](../../internal/api/router/router.go) |
| Shared LMS consumer (same one Save Enquiry / Approve-Reject-Finish use) — republishes **every** consumed LMS message (including Call Enquiry's) to Kafka topic `ENQ_PNS_DATA` | enq-consumers | [`enq_pns_lms.go`](../../../enq-consumers/src/Enquiry/enq_pns_lms.go) (queue `R_MESSAGE_CENTER_BIZFEED_LMS<0-4>`) |

---

## 2. Routes (confirmed from router.go)

| Method | Path | Controller | Middleware position |
|---|---|---|---|
| GET | `/enquiry/callEnquiry` | `controllers.CallEnquiry` | **Before** `middleware.Validation` |
| GET | `/enquiry/callEnquiry/` | `controllers.CallEnquiry` | Before `middleware.Validation` |
| POST | `/enquiry/callEnquiry` | `controllers.CallEnquiry` | Before `middleware.Validation` |
| POST | `/enquiry/callEnquiry/` | `controllers.CallEnquiry` | Before `middleware.Validation` |
| POST | `/enquiry/unidentifiedC2C/` | `controllers.UnidentifiedC2C` | Before `middleware.Validation` |
| POST | `/enquiry/unidentifiedC2C` | `controllers.UnidentifiedC2C` | Before `middleware.Validation` |
| POST | `/enquiry/callEnquiryUnidentified/` | `controllers.CallEnquiryUnidentified` | Before `middleware.Validation` |
| POST | `/enquiry/callEnquiryUnidentified` | `controllers.CallEnquiryUnidentified` | Before `middleware.Validation` |
| POST | `/enquiry/linkC2CPNS` | `controllers.LinkC2CPNS` | **After** `middleware.Validation` (inside the `r.Use(middleware.Validation)` protected block) |

[`router.go:60-63,88-95`](../../internal/api/router/router.go)

**Meaningful asymmetry**: `LinkC2CPNS` is the **only one of these 4 endpoints** registered
after `r.Use(middleware.Validation)` (`router.go:92`) — it sits in the same protected block as
`leapSupplierList15Days`, `waEnqInsert`, `waEnqUpdate`. The other three (`CallEnquiry`,
`CallEnquiryUnidentified`, `UnidentifiedC2C`) are registered earlier in the file, **before**
that middleware is attached, so `middleware.Validation` does **not** run for them. What exactly
`middleware.Validation` checks was not traced in this review's scope — flagged in Open
Questions (section 15) since it's a concrete, code-verified behavioral difference between
`LinkC2CPNS` and its 3 siblings.

---

## 3. Data Model — Tables

> **Verification note**: yeh sab table/column names Go code ke andar embedded SQL strings se
> liye gaye hain (har row ke saamne file path diya hai). Live DB schema se cross-verify **nahi**
> kiya gaya hai — kisi migration/integration se pehle woh zaroor karo.

| Table | Purpose | Key columns / operations (code se) |
|---|---|---|
| `C2C_RECORDS` | Primary identified-call record — every `callEnquiry` insert/update lands here | `C2C_RECORD_ID` (PK, `RETURNING`), `C2C_MODID`, `C2C_CALLER_GLUSR_ID`, `C2C_RECEIVER_GLUSR_ID`, `CALL_DURATION`, `C2C_CALLER_CITY`, `MODREFID`, `QUERY_REF_ID`, `QUERY_REF_TYPE`, `FK_C2C_RECORD_UNIDENTIFIED_ID` — [`callEnquiryModel.go:164,242-276`](../../internal/pkg/models/callEnquiryModel/callEnquiryModel.go) |
| `iil_call_mapping` | Cross-reference/audit table linking a `C2C_RECORDS` row (or PNS records) to entry/last-modified timestamps — used by both `callEnquiry` and `linkC2CPNS` | `fk_c2c_record_id`, `fk_pns_call_records_active_id`, `fk_pns_call_record_id`, `entry_date`, `last_modified_date`, `iil_call_mapping_id` (PK) — [`callEnquiryModel.go:349`](../../internal/pkg/models/callEnquiryModel/callEnquiryModel.go), [`linkC2CModel.go:109,169,226,304,340,411`](../../internal/pkg/models/linkC2CPNSModel/linkC2CModel.go) |
| `C2C_RECORDS_UNIDENTIFIED` | Anonymous/no-GLID-yet call record — `callEnquiryUnidentified` insert/update target | `C2C_RECORD_UNIDENTIFIED_ID` (PK, `RETURNING`), `C2C_MODID`, `C2C_CALLER_GLUSR_ID`, `C2C_CALLER_NUMBER`, `C2C_RECEIVER_GLUSR_ID`, `C2C_BUYER_PROBABLE_NUMBER`, `C2C_BUYER_PROBABLE_GLUSR_ID`, `C2C_BUYER_GA_COOKIE` — [`CallEnquiryUnidentifiedModel.go:340,388,433`](../../internal/pkg/models/callEnquiryUnidentified/CallEnquiryUnidentifiedModel.go) |
| `c2c_records_unidentified` (lowercase in SQL string — same physical table as above, case-insensitive Postgres identifier) | Read target for the `unidentifiedC2C` lookup endpoint | `c2c_record_unidentified_id`, `c2c_buyer_probable_number`, `c2c_receiver_glusr_id`, `c2c_call_time`, `c2c_buyer_ga_cookie` — [`UnidentifiedC2CModel.go:114,157`](../../internal/pkg/models/unidentifiedC2C/UnidentifiedC2CModel.go) |
| `dir_query_<NN>` (sharded, `NN` = `glid % 100`, zero-padded below 10) | Read-only lookup used by `findCallType` (identified-call and unidentified-call models both have their own copy of this function) to auto-detect whether a call correlates to an existing enquiry | `date_r`, `query_id`, `query_rcv_glusr_usr_id`, `fk_glusr_usr_id` — [`callEnquiryModel.go:736-752`](../../internal/pkg/models/callEnquiryModel/callEnquiryModel.go), [`CallEnquiryUnidentifiedModel.go:154-170`](../../internal/pkg/models/callEnquiryUnidentified/CallEnquiryUnidentifiedModel.go) |
| `eto_lead_pur_hist` | Read-only lookup, same `findCallType` step — checks if a purchased-lead (BL) transaction exists between the caller/receiver pair | `eto_pur_date`, `fk_eto_ofr_id`, `fk_glusr_usr_id`, `eto_ofr_glusr_usr_id` — [`callEnquiryModel.go:748`](../../internal/pkg/models/callEnquiryModel/callEnquiryModel.go) |
| `c2c_records` (lowercase in SQL — same table as `C2C_RECORDS`) | Read-only 5-minute-window lookback used by `linkC2CPNS` to find a matching identified-call record for a caller/receiver/timestamp triple | `c2c_record_id`, `c2c_record_type`, `modrefid`, `modreftyp`, `c2c_number_type`, `C2C_CALL_TIME` — [`linkC2CModel.go:89,284,481,695`](../../internal/pkg/models/linkC2CPNSModel/linkC2CModel.go) |
| `pns_call_records` / `pns_call_records_active` | **Not queried by this repo's write path** — read by the downstream `enq_pns_lms.go` consumer (section 8) when it enriches an LMS message whose `insertion_type` is `3` (post-call) or `2` (pre-call) respectively | [`enq_pns_lms.go:224-248`](../../../enq-consumers/src/Enquiry/enq_pns_lms.go) |

**Why `callEnquiryUnidentified`'s App-type update doesn't have its own insert SQL for
`C2C_RECORDS`**: `updateCallIMOB()` doesn't `INSERT INTO C2C_RECORDS` directly — it builds a
`callEnquiryModel.InputParam` and calls `callEnquiry.CallEnquiryModel()` in-process
([`CallEnquiryUnidentifiedModel.go:476-478`](../../internal/pkg/models/callEnquiryUnidentified/CallEnquiryUnidentifiedModel.go)),
reusing the exact same insert path documented in section 3's `C2C_RECORDS` row, just with
`FK_C2C_RECORD_UNIDENTIFIED_ID` populated so the two tables stay linked.

---

## 4. "Decode This Magic Value" — Codes Traced From The Code

### 4.1 `CreateJson` / `InsertInto` — insert-vs-update routing flag

Both `callEnquiryModel.InputParam` and `callEnquiryUnidentified.InputParam` carry a
`CreateJson int` + `InsertInto string` pair that decide which DB path runs:

| `InsertInto` | Meaning |
|---|---|
| `"PI"` ("Post Insert", inferred from usage) | Route to the INSERT path (`insertCallPostgres` / `insertCallUnidentified`) |
| `"PU"` ("Post Update", inferred from usage) | Route to the UPDATE path (`updateCallPostgres` / `updateCallPNS` or `updateCallIMOB`) |

[`callEnquiryModel.go:128-156`](../../internal/pkg/models/callEnquiryModel/callEnquiryModel.go),
[`CallEnquiryUnidentifiedModel.go:111-133`](../../internal/pkg/models/callEnquiryUnidentified/CallEnquiryUnidentifiedModel.go).
**Exact expansion of "PI"/"PU" as acronyms — [INFERRED, no comment/enum in code — confirm with
team].** These same flags are also reused as a backup-file marker (`saveAsJSON`/`SaveAsJSON`)
when a DB write fails — the code re-sets `CreateJson=1` and `InsertInto` before persisting the
failed payload to disk, so a retried/replayed payload re-enters the same insert-vs-update
branch (section 6).

### 4.2 `GetCallerDetails` — `connect_type` / rate-limit `flag`

`GetCallerDetails()` calls an external "bsmapping" HTTP service, then branches on its
`connect_type` response:

| `connect_type` | Meaning | Resulting `callBack` |
|---|---|---|
| `2` | "Hard match" — bsmapping service confirms caller/receiver are a known pair | `2` |
| `0` (interpreted by fallthrough) or bsmapping error (`-1`) | "Soft match" / lookup failure | `1` |

[`callEnquiryModel.go:614-620`](../../internal/pkg/models/callEnquiryModel/callEnquiryModel.go)
— code comment: *"connect_type =2 hard match connect_type=1 soft match connect_type=0 no data
found ; for 1 and 0 run query and check"*. **`C2CRecordType`'s exact business meaning (what
"hard match" vs "soft match" implies downstream) — [INFERRED, only this comment found, confirm
with Users/Trust team].**

If `connect_type != 2` (i.e. not a hard match), a second query runs against `C2C_RECORDS`
counting **distinct receivers called by this caller today**; if that count is `>= 50`, `flag=1`
is returned, which the model turns into an HTTP `429`-coded rate-limit response.
[`callEnquiryModel.go:614-648`](../../internal/pkg/models/callEnquiryModel/callEnquiryModel.go)

### 4.3 `findCallType` — `QueryRefType` `W` vs `B`

Identical logic exists **independently in two files** (`callEnquiryModel.go` and
`CallEnquiryUnidentifiedModel.go`) — see section 6 for the duplication flag:

| Value | Meaning | Evidence |
|---|---|---|
| `"W"` | The call correlates to an existing **enquiry** (`dir_query_<NN>` match is more recent, or is the only match) | `getTypeData()` returns `("W", enq["query_id"])` — [`callEnquiryModel.go:809-828`](../../internal/pkg/models/callEnquiryModel/callEnquiryModel.go) |
| `"B"` | The call correlates to a purchased **Business Lead** (`eto_lead_pur_hist` match is more recent, or is the only match) | `getTypeData()` returns `("B", bl["fk_eto_ofr_id"])` |
| `""` (empty) | Neither an enquiry nor a BL-purchase match found | Both `eto_pur_date` and `date_r` empty → `("", "")` |

This only runs when `C2CPageType`/page-context matches a hardcoded allowlist of Message-Center/
Lead-Manager-style screen names **and** the ModID is `IMOB`/`ANDROID`/`IOS` — a plain
web/other-ModID call never triggers this auto-detection.
[`callEnquiryModel.go:727-730`](../../internal/pkg/models/callEnquiryModel/callEnquiryModel.go)

### 4.4 `linkC2CPNS` — `state` (`pre`/`post`) and `insertion_type`

| `state` | Meaning | LMS `insertion_type` set |
|---|---|---|
| `"pre"` | The call is still active/ringing on the PNS (virtual-number) side; linking uses `PNSActiveID` | `"2"` |
| `"post"` | The call has completed on the PNS side; linking uses `PNSID` | `"3"` |

[`linkC2CModel.go:49-55`](../../internal/pkg/models/linkC2CPNSModel/linkC2CModel.go). Both
values are also the exact `insertion_type` values the downstream `enq_pns_lms.go` consumer
switches on to decide which PNS table (`pns_call_records_active` vs `pns_call_records`) to
enrich from (section 8) — confirming `linkC2CPNS`'s `insertion_type` and the consumer's
`insertion_type` branching are the same contract.

### 4.5 `C2CNumberType` / `C2CLinkType` / `C2CRecordType`

These are free-form single-character fields (`validate:"truncate:1"` / `numeric:1`) accepted
as-is from the client and stored verbatim — **no enum or decode table for their values was
found anywhere in this codebase.** [`callEnquiryModel.go:52,43,69`](../../internal/pkg/models/callEnquiryModel/callEnquiryModel.go)
**[INFERRED — likely owned by a frontend/telephony contract outside this repo, confirm with
team before assuming any specific meaning].**

---

## 5. Business Rules & Validation (code se exhaustive list)

1. **`callEnquiry`: Receiver GLID, Caller GLID, and ModID are all mandatory; caller cannot
   equal receiver.** `GlidModidvalidation()` returns early on the first failing check, no DB
   round-trip. [`callEnquiryModel.go:830-841`](../../internal/pkg/models/callEnquiryModel/callEnquiryModel.go)
2. **`callEnquiry`: ModID must pass `helper.IsModIdValid` against a config-driven allowlist
   file, AND must not be `"MY"`** — `"MY"` is explicitly excluded even if the allowlist file
   contains it. A failing check returns model-code `429` with `Status="Success"` (an unusual
   pairing — see section 14).
   [`callEnquiry.go:66-78`](../../internal/api/controllers/callEnquiry.go)
3. **`callEnquiry`: every field goes through a reflection-based `Validate()` pass** identical
   in shape to the `truncate:N`/`numeric:N` pattern documented in Save Enquiry KT — silent
   truncation for strings, hard rejection for oversized/non-numeric values.
   [`callEnquiryModel.go:651-720`](../../internal/pkg/models/callEnquiryModel/callEnquiryModel.go)
4. **`callEnquiry`: the daily unique-receiver rate-limit (50) is only checked when
   `FK_C2C_RECORD_UNIDENTIFIED_ID` is empty** — i.e. calls arriving as a "convert an
   unidentified call to identified" (chained from `callEnquiryUnidentified`'s
   `updateCallIMOB`) skip this check entirely, since `FK_C2C_RECORD_UNIDENTIFIED_ID` is set in
   that path. [`callEnquiryModel.go:105-126`](../../internal/pkg/models/callEnquiryModel/callEnquiryModel.go)
5. **`callEnquiry`: `C2CRecordType` defaults from the bsmapping `callBack` value only if the
   client didn't already send one** (`if inputParam.C2CRecordType == ""`).
   [`callEnquiryModel.go:121-124`](../../internal/pkg/models/callEnquiryModel/callEnquiryModel.go)
6. **`callEnquiry`: insert-vs-update routing depends on `C2CRecordID` presence AND the
   `CreateJson`/`InsertInto` flag combination** (section 4.1) — a non-empty `C2CRecordID` with
   `InsertInto != "PI"` routes to UPDATE; otherwise INSERT.
   [`callEnquiryModel.go:127-156`](../../internal/pkg/models/callEnquiryModel/callEnquiryModel.go)
7. **`callEnquiry`: `C2C_CALL_TIME` is only parsed from client input for `SMSREAD`/`smsread`
   ModIDs or when `FK_C2C_RECORD_UNIDENTIFIED_ID` is set** — all other calls get `NOW()` at the
   DB layer, i.e. the server's insert-time, not any client-supplied call time.
   [`callEnquiryModel.go:206-223`](../../internal/pkg/models/callEnquiryModel/callEnquiryModel.go)
8. **`callEnquiry` insert/update SP-equivalent calls have a 2-second timeout in non-local
   environments** (100s/1000s locally) — `deadline exceeded`/`canceling statement` maps to
   `504`, anything else to `500`, and either failure triggers `saveAsJSON()` backup-file write.
   [`callEnquiryModel.go:166-171,278-286`](../../internal/pkg/models/callEnquiryModel/callEnquiryModel.go)
9. **`callEnquiryUnidentified`: `C2C_MODID` and `INSERT_INTO` are always mandatory;
   `INSERT_INTO` must be exactly `"PI"` or `"PU"`.** For `"PI"`: Receiver GLID and
   `C2C_BUYER_GA_COOKIE` are additionally mandatory. For `"PU"`: `C2C_RECORD_UNIDENTIFIED_ID`
   is mandatory, and then — if `ModID == "PNS"`, `C2C_BUYER_PROBABLE_NUMBER` is mandatory;
   otherwise, Caller GLID and Caller Number are mandatory.
   [`CallEnquiryUnidentifiedModel.go:527-558`](../../internal/pkg/models/callEnquiryUnidentified/CallEnquiryUnidentifiedModel.go)
10. **`callEnquiryUnidentified` "PU" update branches on `C2CBuyerProbableNumber` presence, not
    on `ModID` alone**: if a probable number was sent, `updateCallPNS()` runs (attach probable
    number/GLID only); otherwise `updateCallIMOB()` runs (attach real caller GLID/number **and**
    chain into `callEnquiry.CallEnquiryModel()` to create the actual identified `C2C_RECORDS`
    row). [`CallEnquiryUnidentifiedModel.go:105-118`](../../internal/pkg/models/callEnquiryUnidentified/CallEnquiryUnidentifiedModel.go)
11. **`unidentifiedC2C`: exactly one of `c2c_buyer_ga_cookie` / `c2c_receiver_glusr_id` must be
    supplied (not both, not neither), and `c2c_modid` is always mandatory.**
    [`UnidentifiedC2CModel.go:223-234`](../../internal/pkg/models/unidentifiedC2C/UnidentifiedC2CModel.go)
12. **`unidentifiedC2C`'s receiver-GLID lookup only returns calls from the last 5 minutes that
    have no probable number attached yet** (`c2c_call_time >= now() - interval '5 minutes' AND
    c2c_buyer_probable_number IS NULL`) — a hard filter for "still-unclaimed, still-recent"
    calls. [`UnidentifiedC2CModel.go:157`](../../internal/pkg/models/unidentifiedC2C/UnidentifiedC2CModel.go)
13. **`linkC2CPNS`: `callerid`, `receiverid`, `state` (alpha), and `timestamp` are all
    `binding:"required"`** at the Gin-tag level, checked before any custom logic.
    [`linkC2CModel.go:34-41`](../../internal/pkg/models/linkC2CPNSModel/linkC2CModel.go)
14. **`linkC2CPNS`: `timestamp` and `state` are further validated by dedicated helpers**
    (`helper.TimeValidation`, `helper.PNSStateValidation`, the latter applied to a
    lower-cased state) before the model is even called — a failing check returns HTTP `401`.
    [`linkC2CPNS.go:48-57`](../../internal/api/controllers/linkC2CPNS.go)
15. **`linkC2CPNS` pre-call requires `PNSActiveID`; post-call requires `PNSID`** — missing the
    relevant ID for the given state returns a `401` validation failure before any DB call.
    [`linkC2CModel.go:83-85,274-276`](../../internal/pkg/models/linkC2CPNSModel/linkC2CModel.go)
16. **`linkC2CPNS`'s matching/linking logic is a multi-branch decision tree** comparing
    `entry_date`/`last_modified_date` of an existing `iil_call_mapping` row against the
    incoming `timestamp` to decide insert-new vs. update-existing — the exact precedence rule
    (`fkPNSActiveID.String == "" && fkPNSID.String == ""` OR a date-ordering condition) is only
    evaluated for the pre-call branch; the post-call branch's equivalent decision uses a
    simpler `fkPNSID.String == ""` check. [`linkC2CModel.go:221,407`](../../internal/pkg/models/linkC2CPNSModel/linkC2CModel.go)
    **Exact business rationale for the date-ordering condition — [INFERRED, no comment found,
    confirm with team].**

---

## 6. Dead/Duplicated Code — Notable Findings

1. **`findCallType`/`getTypeData` are fully duplicated, near-byte-identical, in two files**:
   [`callEnquiryModel.go:722-828`](../../internal/pkg/models/callEnquiryModel/callEnquiryModel.go)
   and [`CallEnquiryUnidentifiedModel.go:141-246`](../../internal/pkg/models/callEnquiryUnidentified/CallEnquiryUnidentifiedModel.go)
   — same allowlists (`msitePage`, `androidPage`, `iosPage`), same sharding logic, same SQL.
   Any future fix (e.g. a new page-type, a sharding-scheme change) has to be applied in **two
   places**.
2. **A commented-out, superseded time-parsing block sits directly above the live
   equivalent** in `insertCallPostgres` — dead code left in place rather than removed, harmless
   but adds noise when reading the function.
   [`callEnquiryModel.go:224-237`](../../internal/pkg/models/callEnquiryModel/callEnquiryModel.go)
3. **`curlRequest()` in `CallEnquiryUnidentifiedModel.go` is fully implemented but its only
   call site is commented out** in `updateCallIMOB()` (`//C2CRecordID, curltime :=
   curlRequest(...)`) — superseded by the in-process `callEnquiry.CallEnquiryModel()` call on
   the line just above it. Genuinely dead code, not reachable from any live path.
   [`CallEnquiryUnidentifiedModel.go:490-491,497-525`](../../internal/pkg/models/callEnquiryUnidentified/CallEnquiryUnidentifiedModel.go)
4. **`linkC2CModel.go`'s pre-call and post-call branches each repeat the same
   lookup→insert-or-update `iil_call_mapping` pattern roughly 6 times** with only the target
   column (`fk_pns_call_records_active_id` vs `fk_pns_call_record_id`) and a couple of
   conditions differing — a single ~760-line function
   ([`linkC2CModel.go:43-802`](../../internal/pkg/models/linkC2CPNSModel/linkC2CModel.go)) that
   is highly repetitive rather than factored into a shared helper. This is a maintenance-cost
   flag, not a correctness bug — but any future change to the linking logic needs to be
   verified against every repeated block.

---

## 7. Concurrency Gotcha — Shared Package-Level Mutable State

`controllers.uniqueid` (string) and `controllers.ConnectionTime` (int) are declared as
**package-level variables** in
[`leap15DaysSupplierListController.go:20-23`](../../internal/api/controllers/leap15DaysSupplierListController.go)
(`var ( ConnectionTime int; uniqueid string )`), not scoped to any individual request. Three of
the four controllers in this doc write to them with plain assignment (`=`, not `:=`), meaning
they mutate this **shared** state rather than a request-local variable:

- `callEnquiry.go:25` — `uniqueid = persistence.Uniqid()`
- `linkC2CPNS.go:24,53` — `uniqueid = persistence.Uniqid()`, `ConnectionTime = ...`
- `unidentifiedC2C.go:33,43,68` — `ConnectionTime = ...` (multiple sites)

Since Gin handlers run concurrently (one goroutine per request), two simultaneous requests to
`callEnquiry`/`linkC2CPNS`/`unidentifiedC2C` can **race on these package-level variables** —
one request's `uniqueid`/`ConnectionTime` could theoretically be overwritten mid-flight by
another concurrent request before it's read (e.g. before it's placed into a log line or
response). This was **not observed failing** in this review (no test/repro was run — this is a
static-analysis finding), but it is a genuine data race by Go's memory model.
**[INFERRED severity — worth a `go test -race` run or targeted load test to confirm real-world
impact before treating this as urgent; flagged here because it is code-verified, not
speculative.]**

Note: `CallEnquiryUnidentified.go` and `unidentifiedC2C`'s own `uniqueid` usage instead use
`:=` (request-local), so this issue is specifically about `callEnquiry.go`, `linkC2CPNS.go`,
and `unidentifiedC2C.go`'s `ConnectionTime` handling.

---

## 8. RabbitMQ — Queues Touched By Call Enquiry

| Queue / Exchange | Publisher | Purpose |
|---|---|---|
| `R_MESSAGE_CENTER_BIZFEED_LMS<0-4>` (5-way sharded, random modulus), exchange `lms-gcp.topic` | `callEnquiryModel.PushIntoLMSQueue` (`insertion_type="1"`, `transaction_type="C"`) — [`callEnquiryModel.go:395-450`](../../internal/pkg/models/callEnquiryModel/callEnquiryModel.go); `linkC2CModel` (`insertion_type="2"`/`"3"` per pre/post state, `transaction_type="P"`) — [`linkC2CModel.go:769-801`](../../internal/pkg/models/linkC2CPNSModel/linkC2CModel.go) | Lead-Management transaction intake — **the same queue family** Save Enquiry's and Approve/Reject/Finish's `enq_pns_lms.go` consumer already listens on |

**Endpoints that do NOT publish to any queue**: `CallEnquiryUnidentified`'s direct
insert/update paths (`insertCallUnidentified`, `updateCallPNS`) have no `PushIntoLMSQueue`-style
call anywhere in `CallEnquiryUnidentifiedModel.go` — only `updateCallIMOB`'s in-process call
into `callEnquiry.CallEnquiryModel()` indirectly triggers an LMS push (as part of that
delegated call). `unidentifiedC2C` (the read/lookup endpoint) is pure SELECT — no queue
interaction at all.

**No evidence of these 4 endpoints publishing to `ENQ_DATA`, `ENQUIRY_FENQ`, or
`ENQ_PERSONALIZATION_OTHER`** (Save Enquiry's queues) — confirmed via full read of all 4 model
files; Call Enquiry group is a parallel, independent producer into the shared LMS queue family
only, not a participant in the Save-Enquiry/Approve-Reject-Finish pipeline.

---

## 9. Kafka

**No direct Kafka usage found in any of the 4 controllers or models** in `enq-services-scm`
(`callEnquiryModel.go`, `CallEnquiryUnidentifiedModel.go`, `UnidentifiedC2CModel.go`,
`linkC2CModel.go`) — confirmed via a repo-wide grep for `kafka`/`Kafka` across
`enq-services-scm`; no matches anywhere in this repo (consistent with what the Save Enquiry and
Approve/Reject/Finish Technical Docs already report for their own write paths).

**However**, every message these 4 endpoints push into `R_MESSAGE_CENTER_BIZFEED_LMS<0-4>` is
consumed by the **same** `enq_pns_lms.go` worker documented in Save Enquiry KT, and that worker
**unconditionally republishes every message it consumes to Kafka topic `ENQ_PNS_DATA`** via
`curlKafka()` — regardless of `insertion_type`, i.e. Call Enquiry's `insertion_type="1"`
messages go through this too, even though the two DB-enrichment branches
(`datapublishintoPNSKafka`) only handle `insertion_type` `"2"`/`"3"` (PNS pre/post); a Call
Enquiry message with `insertion_type="1"` falls through both `if`/`else if` branches with no
enrichment query run, and is still forwarded to Kafka with the enrichment fields defaulted to
empty strings. [`enq_pns_lms.go:188-290,335-360`](../../../enq-consumers/src/Enquiry/enq_pns_lms.go)

**Straight answer if asked "does Call Enquiry use Kafka":** the write API itself does not,
directly. Its downstream consumer (shared with Save Enquiry/Approve-Reject-Finish) always
forwards to Kafka topic `ENQ_PNS_DATA`, with real DB-enrichment only for PNS-originated
messages (`linkC2CPNS`'s `insertion_type` `2`/`3`) — `callEnquiry`'s own messages
(`insertion_type="1"`) pass through with empty enrichment fields.

---

## 10. Redis

**No Redis usage found in any of the 4 controllers or models** — confirmed via a targeted grep
for `redis`/`Redis` across `enq-services-scm`; the only hits in the whole repo are the
already-documented false-positives in `finishEnquiryModel.go` and the two mailer crons
(local Go variable names, not a Redis client — see Approve/Reject/Finish Technical Doc section
10). None of `callEnquiryModel.go`, `CallEnquiryUnidentifiedModel.go`,
`UnidentifiedC2CModel.go`, or `linkC2CModel.go` reference Redis in any form.

All 4 endpoints hit Postgres directly on every request, with no caching layer.

---

## 11. End-to-End Technical Flows

### Flow A — Identified call, insert path (happy path)

```
Client (WEB/IOS/ANDROID/IMOB/...)
    │
    ▼
[API] GET/POST /enquiry/callEnquiry(/)
    │  callEnquiry.go
    │  1. c.ShouldBindBodyWith(&inputParam, binding.JSON) — bind failure → 400 (HTTP 200 on wire)
    │  2. GlidModidvalidation() — Receiver/Caller GLID + ModID mandatory, caller != receiver
    │  3. helper.IsModIdValid(...) && ModID != "MY" — else 429/"Success" (HTTP 200 on wire)
    │  4. inputParam.Validate() — reflection-based truncate/numeric pass
    ▼
[MODEL] CallEnquiryModel()
    │  1. LogRawData() — raw backup log write
    │  2. dbConn()
    │  3. If FK_C2C_RECORD_UNIDENTIFIED_ID empty:
    │     GetCallerDetails() → bsmapping HTTP call (connect_type) → daily unique-receiver
    │     count query on C2C_RECORDS → flag==1 → 429 rate-limit response (early return)
    │  4. C2CRecordID empty/CreateJson!=1||InsertInto!="PU" → INSERT branch:
    │     findCallType() (conditional, ModID/page-type gated) → dir_query_<NN> + eto_lead_pur_hist
    │     lookups → insertCallPostgres(): INSERT INTO C2C_RECORDS ... RETURNING C2C_RECORD_ID
    │     → insert into iil_call_mapping (fk_c2c_record_id, entry_date, last_modified_date)
    ▼
[QUEUE] PushIntoLMSQueue() → RabbitMQ R_MESSAGE_CENTER_BIZFEED_LMS<0-4>, insertion_type "1"
    ▼
Client receives: {code, status, c2c_record_id, iil_call_mapping_id, UNIQUE_MSG_ID, response}

    (async, downstream)
    ▼
[CONSUME] enq_pns_lms.go on R_MESSAGE_CENTER_BIZFEED_LMS<n>
    │  insertion_type=="1" falls through both PNS-enrichment branches (no DB query run)
    ▼
    curlKafka() → Kafka topic ENQ_PNS_DATA (empty enrichment fields for this insertion_type)
```

### Flow B — Identified call, update path (duration/city arrives later)

```
Client
    │
    ▼
[API] POST /enquiry/callEnquiry  {c2c_record_id, call_duration, c2c_caller_city, ...,
                                    create_json:0 or (create_json:1, insert_into:"PU")}
    ▼
[MODEL] CallEnquiryModel() → C2CRecordID present + routing condition met → updateCallPostgres()
    │  UPDATE C2C_RECORDS SET CALL_DURATION=$1, C2C_CALLER_CITY=$2, C2C_CALLER_LOC_IDENTIFY_BY=$3
    │  WHERE C2C_RECORD_ID=$4 AND C2C_CALLER_GLUSR_ID=$5 AND C2C_RECEIVER_GLUSR_ID=$6
    │  → error → saveAsJSON() backup file, 500/504
    ▼
Client receives: {code:200, status:"success", c2c_record_id, response:"updation successful in DB"}
    (no queue push on the update path — PushIntoLMSQueue is only called from the insert branch)
```

### Flow C — Unidentified call (anonymous, no GLID yet)

```
Client (e.g. website widget, no login)
    │
    ▼
[API] POST /enquiry/callEnquiryUnidentified(/)  {c2c_modid, insert_into:"PI",
                                                    c2c_receiver_glusr_id, c2c_buyer_ga_cookie, ...}
    │  MandatoryValidation() → PI-branch checks (receiver GLID + GA cookie mandatory)
    │  inputParam.Validate() → truncate/numeric pass
    ▼
[MODEL] UpdateUnidetifiedCall() → insert_into != "PU" → findCallType() (conditional) →
    insertCallUnidentified(): INSERT INTO C2C_RECORDS_UNIDENTIFIED ... RETURNING
    C2C_RECORD_UNIDENTIFIED_ID
    ▼
Client receives: {code:200, c2c_record_unidentified_id, response:"Successfully Inserted in DB"}
    (no queue push — this endpoint never calls PushIntoLMSQueue directly)
```

### Flow D — Unidentified call becomes identified (App-type "claim")

```
Client
    │
    ▼
[API] POST /enquiry/callEnquiryUnidentified(/)  {insert_into:"PU", c2c_record_unidentified_id,
                                                    c2c_modid != "PNS", c2c_caller_glusr_id,
                                                    c2c_caller_number}
    ▼
[MODEL] UpdateUnidetifiedCall() → insert_into=="PU", no probable-number in payload → updateCallIMOB()
    │  UPDATE C2C_RECORDS_UNIDENTIFIED SET C2C_CALLER_GLUSR_ID=$1, C2C_CALLER_NUMBER=$2
    │  WHERE C2C_RECORD_UNIDENTIFIED_ID=$3
    │  on success →
    │     build callEnquiryModel.InputParam (createCallEnquiryParams), set
    │     FK_C2C_RECORD_UNIDENTIFIED_ID = this unidentified record's ID
    │     → callEnquiry.CallEnquiryModel() IN-PROCESS CALL → Flow A's insert branch runs in full
    │       (rate-limit check SKIPPED since FK_C2C_RECORD_UNIDENTIFIED_ID is now set — section 5.4)
    ▼
Client receives: {code:200, c2c_record_unidentified_id, c2c_record_id (from the chained call),
                   response:"Buyer Number & GLID updated successfully"}
```

### Flow E — Unidentified call becomes PNS-linked ("probable number" attach)

```
Client
    │
    ▼
[API] POST /enquiry/callEnquiryUnidentified(/)  {insert_into:"PU", c2c_record_unidentified_id,
                                                    c2c_modid:"PNS", c2c_buyer_probable_number}
    ▼
[MODEL] UpdateUnidetifiedCall() → insert_into=="PU", probable-number present → updateCallPNS()
    │  UPDATE C2C_RECORDS_UNIDENTIFIED SET C2C_BUYER_PROBABLE_NUMBER=$1,
    │  C2C_BUYER_PROBABLE_GLUSR_ID=$2, INTEREST_PDATE=CURRENT_TIMESTAMP
    │  WHERE C2C_RECORD_UNIDENTIFIED_ID=$3
    ▼
Client receives: {code:200, response:"Buyer Probable Number updated successfully"}
```

### Flow F — Unidentified-call lookup (read-only)

```
Client
    │
    ▼
[API] POST /enquiry/unidentifiedC2C(/)  {c2c_buyer_ga_cookie XOR c2c_receiver_glusr_id, c2c_modid}
    │  unidentifiedC2C.go: bind → ValidateInputParams (exactly-one-of check)
    ▼
[MODEL] FetchUnidentifiedCallRecords()
    │  ga_cookie present →
    │     SELECT c2c_record_unidentified_id, c2c_buyer_probable_number
    │     FROM c2c_records_unidentified WHERE c2c_buyer_ga_cookie=$1
    │  else (receiver glid present) →
    │     SELECT c2c_record_unidentified_id, c2c_receiver_glusr_id, c2c_call_time
    │     FROM c2c_records_unidentified
    │     WHERE c2c_receiver_glusr_id=$1 AND c2c_buyer_probable_number IS NULL
    │       AND c2c_call_time >= now() - interval '5 minutes'
    │     ORDER BY c2c_call_time DESC
    ▼
Client receives: {code:200, response: [...rows...]}
    (pure read — no writes, no queue interaction anywhere in this flow)
```

### Flow G — PNS pre-call linking

```
PNS system / client
    │
    ▼
[API] POST /enquiry/linkC2CPNS  {callerid, receiverid, state:"pre", fkpnsactiveid, timestamp}
    │  linkC2CPNS.go: bind → TimeValidation + PNSStateValidation → LinkC2CPNSModel()
    ▼
[MODEL] LinkC2CPNSModel()
    │  1. dbConn()
    │  2. state=="pre": PNSActiveID mandatory (else 401)
    │  3. SELECT c2c_record_id, c2c_record_type, modrefid, modreftyp, c2c_number_type
    │     FROM c2c_records WHERE caller=$1 AND receiver=$2
    │     AND call_time in [timestamp-5min, timestamp] ORDER BY call_time DESC LIMIT 1
    │     ├─ sql.ErrNoRows → INSERT into iil_call_mapping (fk_pns_call_records_active_id,
    │     │                   entry_date, last_modified_date)
    │     ├─ other error → INSERT (same, as a fail-safe fallback) + error logged
    │     └─ row found → SELECT existing iil_call_mapping row by fk_c2c_record_id
    │           → date-ordering decision tree (section 5.16) → INSERT new or UPDATE existing
    │             iil_call_mapping row's fk_pns_call_records_active_id
    ▼
[QUEUE] Insert_rabbitmq() → RabbitMQ R_MESSAGE_CENTER_BIZFEED_LMS<0-4>, insertion_type "2",
        transaction_type "P"
    ▼
Client receives: {code:200/203, status, response}
```

### Flow H — PNS post-call linking

```
PNS system / client
    │
    ▼
[API] POST /enquiry/linkC2CPNS  {callerid, receiverid, state:"post", fkpnsid, timestamp,
                                    fkpnsactiveid (optional)}
    ▼
[MODEL] LinkC2CPNSModel()
    │  state=="post": PNSID mandatory (else 401); insertion_type forced to "3"
    │  ├─ PNSActiveID empty → same c2c_records 5-min lookback as Flow G, then insert/update
    │  │    iil_call_mapping keyed on fk_pns_call_record_id
    │  └─ PNSActiveID present → look up iil_call_mapping by fk_pns_call_records_active_id
    │       (linking the pre-call and post-call PNS records together) → UPDATE
    │       fk_pns_call_record_id on that same row, then fill in c2c_record_id/record_type/
    │       product info for the LMS payload from either c2c_records (5-min lookback) or a
    │       direct c2c_record_id lookup
    ▼
[QUEUE] Insert_rabbitmq() → RabbitMQ R_MESSAGE_CENTER_BIZFEED_LMS<0-4>, insertion_type "3",
        transaction_type "P"
    ▼
Client receives: {code:200/203, status, response}
```

---

## 12. Flow-wise DB & Table Usage

### Flow A — Identified Call, Insert

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1 | (external HTTP) | bsmapping service | GET-with-form-body | Determine `connect_type` (hard/soft match) for the caller-receiver pair |
| 2 | enquiry-write Postgres | `C2C_RECORDS` | SELECT (`COUNT(DISTINCT c2c_receiver_glusr_id)`) | Daily unique-receiver rate-limit check, only when `connect_type != 2` |
| 3 | enquiry-write Postgres | `dir_query_<NN>` (x2, caller-shard + receiver-shard, `UNION`) | SELECT | Conditional (`findCallType`) — check for a recent matching enquiry |
| 4 | enquiry-write Postgres | `eto_lead_pur_hist` | SELECT | Conditional (`findCallType`) — check for a recent matching purchased-lead |
| 5 | enquiry-write Postgres | `C2C_RECORDS` | INSERT (`RETURNING C2C_RECORD_ID`) | The actual call record write |
| 6 | enquiry-write Postgres | `iil_call_mapping` | INSERT (`RETURNING iil_call_mapping_id`) | Cross-reference/audit row for this call |
| 7 | RabbitMQ | `R_MESSAGE_CENTER_BIZFEED_LMS<0-4>` | Publish | LMS transaction intake |

### Flow B — Identified Call, Update

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1 | enquiry-write Postgres | `C2C_RECORDS` | UPDATE | Attach duration/city/loc-identify-by to an existing call record |

### Flow C — Unidentified Call Insert

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1-2 | enquiry-write Postgres | `dir_query_<NN>`, `eto_lead_pur_hist` | SELECT | Same conditional `findCallType` as Flow A |
| 3 | enquiry-write Postgres | `C2C_RECORDS_UNIDENTIFIED` | INSERT (`RETURNING C2C_RECORD_UNIDENTIFIED_ID`) | Anonymous call record write |

### Flow D — Unidentified → Identified (App-type claim)

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1 | enquiry-write Postgres | `C2C_RECORDS_UNIDENTIFIED` | UPDATE | Attach caller GLID/number |
| 2-8 | *(entire Flow A DB+queue chain reused via the in-process `CallEnquiryModel()` call)* | — | — | Creates the actual identified `C2C_RECORDS` row, linked back via `FK_C2C_RECORD_UNIDENTIFIED_ID` |

### Flow E — Unidentified → PNS-probable (attach)

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1 | enquiry-write Postgres | `C2C_RECORDS_UNIDENTIFIED` | UPDATE | Attach probable number/GLID only, no chained call |

### Flow F — Unidentified-call Lookup

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1 | enquiry-read Postgres (`db.DatabaseR`) | `c2c_records_unidentified` | SELECT | By GA-cookie or by receiver-GLID + 5-minute-unclaimed filter |

### Flow G/H — PNS Linking (pre/post)

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1 | enquiry-write Postgres | `c2c_records` | SELECT (5-min lookback) | Find a matching identified-call record for this caller/receiver/timestamp |
| 2 | enquiry-write Postgres | `iil_call_mapping` | SELECT | Check for an existing cross-reference row to update vs. insert-fresh |
| 3 | enquiry-write Postgres | `iil_call_mapping` | INSERT or UPDATE | Write/refresh the PNS↔C2C link |
| 4 | RabbitMQ | `R_MESSAGE_CENTER_BIZFEED_LMS<0-4>` | Publish | LMS transaction intake, `insertion_type` `"2"`/`"3"` |
| 5 (downstream, `enq-consumers`) | Postgres (`enqdb_write`) | `pns_call_records_active` (pre) / `pns_call_records` (post) | SELECT | Enrichment before the Kafka `ENQ_PNS_DATA` publish (section 9) |

---

## 13. Optimization Scope — DB/Response-Time Contributors

### High-impact

1. **`callEnquiry`'s external bsmapping HTTP call is synchronous and in the critical path**
   for every insert-path call that doesn't already carry `FK_C2C_RECORD_UNIDENTIFIED_ID** —
   same category of risk as Save Enquiry's BAN-API dependency and GST's BI-API dependency: an
   external network hop directly inflates response time, with only a 2-second client timeout
   as a ceiling. [`callEnquiryModel.go:524-533`](../../internal/pkg/models/callEnquiryModel/callEnquiryModel.go)
2. **`findCallType`'s duplicated implementation (section 6.1) means any index/query
   optimization has to be applied twice** — a real risk of the two copies drifting out of sync
   over time (one gets optimized, the other doesn't).
3. **`linkC2CModel.go`'s repeated lookup→insert-or-update blocks (section 6.4) mean the same
   3-5 sequential DB round-trips happen in almost every branch** — no parallelism anywhere in
   this file's DB access, unlike Save Enquiry's/Finish Enquiry's parallel-goroutine patterns
   for independent lookups.

### Medium-impact

4. **No caching on any of the 4 endpoints' reads** — the `dir_query_<NN>`/`eto_lead_pur_hist`
   auto-context lookups (Flow A/C), the 5-minute `c2c_records` lookback (Flow G/H), and the
   unidentified-call lookup (Flow F) all hit Postgres live every time. The 5-minute-window
   nature of the PNS lookback in particular makes it a poor caching candidate (highly
   time-sensitive), but the enquiry/BL auto-detection lookups are less time-sensitive and could
   plausibly benefit from a short-TTL cache the same way Save Enquiry's sender/receiver lookups
   are flagged for in its own Technical Doc.
5. **`callEnquiry` opens a DB connection unconditionally via `dbConn()` before the
   rate-limit/insert-vs-update branching even begins**, similar to the "connections opened
   before guard-check" pattern flagged in Save Enquiry's Technical Doc.
   [`callEnquiryModel.go:91`](../../internal/pkg/models/callEnquiryModel/callEnquiryModel.go)

### Low-impact / good-practice already present

6. **Positive finding**: `callEnquiry`'s backup-on-failure pattern (`saveAsJSON`, triggered on
   both insert and update DB errors) mirrors Save Enquiry's resilience design — a failed write
   is preserved to disk rather than silently lost, even though this review found **no
   dedicated Call-Enquiry-specific replay cron** consuming these backup files (section 14,
   Open Questions) — worth confirming whether Save Enquiry's `QueryParse` cron also picks these
   up, or whether they're currently unconsumed.

---

## 14. Cron Inventory

No cron job in `enq-services-scm/pkg/crons/` (`EnquiryMailer`, `EnquiryParseManual`,
`QASapprovedMailer`, `QueryParse`, `setModIds.go`) references `callEnquiry`,
`callEnquiryUnidentified`, `unidentifiedC2C`, or `linkC2CPNS` by name or by their backup-file
naming pattern (`enquirycallEnquiry`) — confirmed via targeted review of the crons directory
listing. This means the `saveAsJSON`/`SaveAsJSON` backup files written on DB failure
(`callEnquiryModel.go:201,307`, `CallEnquiryUnidentifiedModel.go:373,417,464`) have **no
confirmed automated replay mechanism** analogous to Save Enquiry's `QueryParse` cron.
**[INFERRED — flagged as Open Question below; may be handled by a cron outside this repo's
`pkg/crons/` directory, or may genuinely be an unaddressed gap.]**

---

## 15. Edge Cases & Gotchas (technical POV)

1. **`callEnquiry`'s invalid-ModID rejection uses `Response.Code=429` paired with
   `Status="Success"`** — an unusual combination (a rate-limit-style HTTP code with a
   "success" status string) that's easy to misread as an actual success in log dashboards.
   [`callEnquiry.go:69`](../../internal/api/controllers/callEnquiry.go)
2. **Every early-rejection path across all 4 endpoints returns HTTP `200` on the wire**
   (400/401/429/504 model-codes all get `c.JSON(200, Response)`'d) — the real outcome is only
   visible via the `code`/`status` fields in the response body, not the HTTP status line. This
   matches the pattern already documented in Save Enquiry KT (HTTP `430`→`200`) and
   Approve/Reject/Finish KT (HTTP `204`→`200`) — a consistent house style across this whole
   enquiry domain, not unique to Call Enquiry.
3. **The package-level `uniqueid`/`ConnectionTime` race (section 7) affects 3 of these 4
   endpoints** — worth confirming with a race-detector run given these are high-traffic,
   concurrently-hit endpoints.
4. **`findCallType`'s duplication (section 6.1) means a fix applied to one copy is easy to
   forget applying to the other** — anyone debugging "why did the enquiry/BL auto-tag not
   work for an unidentified call" should check `CallEnquiryUnidentifiedModel.go`'s copy
   specifically, not just `callEnquiryModel.go`'s.
5. **`unidentifiedC2C` is registered as a `POST` route but is purely read-only** (no INSERT/
   UPDATE anywhere in `FetchUnidentifiedCallRecords`) — easy to misclassify as a write endpoint
   from the route method alone; the actual behavior is a lookup/list query.
6. **`callEnquiryUnidentified`'s App-type "claim" (Flow D) silently swallows a
   `CallEnquiryModel()` failure** — if the chained call returns a non-200/non-success result,
   the code only logs an error and still returns its own `200`/`"success"` response with an
   empty `C2CRecordID`, meaning the client has no direct signal that the identified-call side
   of the operation failed. [`CallEnquiryUnidentifiedModel.go:482-492`](../../internal/pkg/models/callEnquiryUnidentified/CallEnquiryUnidentifiedModel.go)
7. **`LinkC2CPNS` is the only one of these 4 routes behind `middleware.Validation`** (section
   2) — if that middleware enforces something like an auth/signature check, the other 3
   endpoints are, by definition, not subject to it. Worth confirming this asymmetry is
   intentional given all 4 are conceptually part of the same call-tracking feature group.
8. **Rate-limit check (50 unique receivers/day) only fires for genuinely first-time,
   non-chained calls** (section 5.4) — an unidentified-call-turned-identified via `updateCallIMOB`
   never goes through this check, meaning the rate-limit is not a hard universal cap on total
   identified `C2C_RECORDS` inserts for a caller, only on inserts arriving directly through
   `callEnquiry`'s own entrypoint.

---

## 16. Open Questions

1. **What does `middleware.Validation` actually check**, and why is `LinkC2CPNS` the only one
   of these 4 routes placed behind it (section 2, section 15.7)? Not traced in this review's
   scope.
2. **Exact meaning of `InsertInto` values `"PI"`/`"PU"`** (section 4.1) — acronym expansion not
   found anywhere in code or comments.
3. **Exact business meaning of bsmapping's `connect_type`/hard-match vs soft-match distinction**
   and what `C2CRecordType` values `1`/`2` (and any others) mean downstream (section 4.2) — no
   enum found in this repo; likely owned by the Users/Trust domain.
4. **Do the `saveAsJSON`/`SaveAsJSON` backup files written by these 4 endpoints on DB failure
   have any automated replay mechanism** (section 14)? No cron in this repo's `pkg/crons/`
   references them by name — confirm whether this is a genuine gap or handled elsewhere.
5. **Exact business rationale for the date-ordering precedence condition** in `linkC2CPNS`'s
   pre-call insert-vs-update decision (section 5.16) — no comment found explaining why this
   specific ordering matters.
6. **Whether `unidentifiedC2C`'s GA-cookie based lookup query has an index on
   `c2c_buyer_ga_cookie`** — not verifiable from Go code; worth confirming given this is a
   frequently-hit read path with no caching (section 13.4).
7. **Live DB schema verification** (column types, nullability, indexes, constraints) for
   `C2C_RECORDS`, `C2C_RECORDS_UNIDENTIFIED`, and `iil_call_mapping` — this doc only reflects
   what the Go SQL strings imply.
8. **Whether the package-level `uniqueid`/`ConnectionTime` race (section 7) has caused any
   observed production issue** (mismatched logging, cross-request ID leakage) — this review
   found the pattern via static analysis only, no runtime evidence either way.

---

## See also

- [`Call_Enquiry_Business_Doc.md`](./Call_Enquiry_Business_Doc.md) — same flows, product/
  business perspective, bina code ke
- [`../Save Enquiry KT/Save_Enquiry_Technical_Doc.md`](../Save%20Enquiry%20KT/Save_Enquiry_Technical_Doc.md) — text-enquiry equivalent; shares the same `R_MESSAGE_CENTER_BIZFEED_LMS<0-4>` queue family and the same `enq_pns_lms.go` downstream consumer documented here in section 8-9
- [`../Approve-Reject-Finish Enquiry KT/Approve_Reject_Finish_Enquiry_Technical_Doc.md`](../Approve-Reject-Finish%20Enquiry%20KT/Approve_Reject_Finish_Enquiry_Technical_Doc.md) — text-enquiry lifecycle resolution; no direct code-level interaction found with Call Enquiry group, but shares the same LMS consumer infrastructure

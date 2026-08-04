# Feedback — Technical Doc (Code-Level Deep Dive)

Yeh doc Feedback feature ka **technical implementation** cover karta hai — APIs, DB tables,
queries, RabbitMQ, consumers, sab kuch code se verify karke. Business/product perspective ke
liye [`Feedback_Business_Doc.md`](./Feedback_Business_Doc.md) dekho — dono docs same flows
cover karte hain, bas alag audience ke liye.

**Scope note**: 3 feedback-types, ek folder mein combined kyunki My-Feedback aur
IMSearch-Feedback ek hi controller/endpoint share karte hain; App-Rating-Feedback alag
endpoint hai lekin (jaisa iss deeper pass mein confirm hua, section 4/10) **actually usi
write-endpoint ko ek internal HTTP call se hit karta hai**, isliye teeno genuinely ek hi
domain hain, sirf entry-point alag hai. **Social Review** (external social-media reviews)
genuinely alag concept hai, dekho [`../Social Review KT/`](../Social%20Review%20KT/).

**Repos**: `service-api-go-production` (write: My Feedback + IMSearch Feedback),
`users-api-go-production` (App-Rating Feedback — apna khud ka HTTP endpoint, jo internally
write-repo ko hi call karta hai), `user-temp-consumers-production` (`USER_FEEDBACK` consumer
+ generic `USER_RELATED_SEND_MAIL` consumer).

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file:line diya
gaya hai har jagah). Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — team se confirm karo]** likh diya hai, guess nahi kiya.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| My Feedback + IMSearch Feedback write (controller) | write | [`UserFeedbackController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserFeedbackController.go) |
| My Feedback + IMSearch Feedback write (model/DB/RabbitMQ) | write | [`UserFeedbackModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFeedbackModel.go) — `FeedbackCheckIssueTypeID`, `FeedbackUpdateDB`, `ImSearchFeedbackUpdateDB` |
| Mandatory-field validation | write | `MandatoryFieldsFeedback` — [`UserUtilsMandatory.go:1068-1152`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go) |
| Length/type validation + column maps | write | `ValidateUserFeedback` (line 2147), `UserFeedbackMapMyFeedbacks` (line 115), `UserFeedbackMapImsearchFeedback` (line 233) — [`UsersValidationMaps.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |
| My Feedback consumer (mail + approvalPg sync) | consumers | [`USER_FEEDBACK.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_FEEDBACK.go) — `dbActionUserFeedback`, `InsertIntoPG`, `feedback_mail` |
| Generic mail-relay consumer (App-Rating's own email path) | consumers | [`USER_RELATED_SEND_MAIL.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RELATED_SEND_MAIL.go) — shared with an unrelated `PAY_NOW` flow, dispatches on `SERVICE_NAME` |
| App-Rating Feedback (controller, read-repo) | read | [`UserFeedbackController.go`](../internal/controllers/UsersControllers/UserFeedbackController.go) — `UserFeedback` |
| App-Rating Feedback (model — HTTP-loopback orchestration) | read | [`UserFeedbackModel.go`](../internal/models/users/UserFeedbackModel.go) — `UsersFeedback` |

---

## 2. Routes

| Method | Path/serviceName | Repo | Controller |
|---|---|---|---|
| POST | `/feedback` → `serviceName=FEEDBACK_SERVICE`, `FEEDBACK_FLAG=MY_FEEDBACKS` (default) or `IMSEARCH_FEEDBACK` | write | `UserFeedbackController` — [`router.go:169,337`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) |
| GET/POST | `feedback/*params` → `serviceName=USERFEEDBACK` | read | `UserFeedback` (App-Rating Feedback) — [`routerUsers.go:355-356,526-527`](../internal/api/users_router/routerUsers.go) |

**Discovery — the two routes are chained, not parallel**: the read-repo's `feedback/*params`
route does not write to any DB itself. Its model (`UsersFeedback`) makes an internal HTTP
call to the exact same write-repo `/feedback` endpoint above (see section 4, point 13) — so
every App-Rating submission is, underneath, also a `MY_FEEDBACKS` write on the same code path
as My-Feedback. This directly resolves Open Question #1 from the previous (shallower) pass of
this doc.

---

## 3. Data Model — Tables

> **Verification note**: yeh sab table/column names Go code ke andar embedded SQL strings se
> liye gaye hain (har row ke saamne file path diya hai). Live DB schema se pgAdmin pe
> cross-verify **nahi** kiya gaya hai.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `MY_FEEDBACKS` | **meshpg** (primary, API-side write) + **approvalPg** (consumer-side second copy, see §6) | General feedback (My Feedback **and** App-Rating Feedback — both land here) | `MY_FEEDBACKS_ID` (PK, `RETURNING`), `MY_FEEDBACKS_MOBILE`, `FK_GLUSR_ID`, `MY_FEEDBACKS_DESCRIPTION`, `MY_FEEDBACKS_EMAIL`, `MY_FEEDBACKS_DATE`, `MY_FEEDBACKS_COUNTRY`, `MY_FEEDBACKS_SOURCE`, `MY_FEEDBACKS_USR_TYPE`, `MY_FEEDBACKS_MODID`, `MY_FEEDBACKS_TITLE`, `MY_FEEDBACKS_RATINGS`, `MY_FEEDBACKS_APP_VER`, `MY_FEEDBACKS_APPROVAL_STATUS` (`'P'` hardcoded on insert), `MY_FEEDBACKS_ISSUE_IS_VAGUE`, `FK_FEEDBACK_ISSUE_TYPE_ID`, `MY_FEEDBACKS_EMP_REMARKS`, `MY_FEEDBACKS_TYPE_FLAG` (`'M'` = mail sent, consumer-side only) — [`UserFeedbackModel.go:183-217`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFeedbackModel.go) (write-side), [`USER_FEEDBACK.go:130,172,200`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_FEEDBACK.go) (consumer-side) |
| `MY_FEEDBACK_ISSUE_TYPE` | meshpg (lookup) | Valid issue-type-ID whitelist + display text | `FEEDBACK_ISSUE_TYPE_ID`, `FEEDBACK_ISSUE_TYPE_VAL` — [`UserFeedbackModel.go:20`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFeedbackModel.go), [`USER_FEEDBACK.go:283`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_FEEDBACK.go) |
| `IM_SEARCH_FEEDBACK` | meshpg | Search-quality feedback | `IM_SEARCH_ID` (PK, `RETURNING`), `IM_SEARCH_QUERY`, `IM_SEARCH_DATE`, `IM_SEARCH_PAGE_NO`, `IM_SEARCH_URL` (max 2048 chars, app-truncated), `IM_SEARCH_TOP_RESULTS`, `IM_SEARCH_PAGE_SCROLL`, `RATING`, `COMMENTS`, `COOKIE`, `IM_SEARCH_TOTAL_RESULTS`, `IM_SEARCH_MOBILE`, `FK_GLUSR_USR_ID`, `IM_SEARCH_FEEDBACK_PRD_POSITION`, `IM_SEARCH_QUERY_UPDATED`, `IM_SEARCH_PAGE_REFERRER` (max 2048), `IM_SEARCH_SUGGESTED_MCATS` (JSON), `IM_SEARCH_CATEGORY_ID`, `IM_SEARCH_MODID` — [`UserFeedbackModel.go:473-492`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFeedbackModel.go) |
| `glusr_usr` (read, mail-lookup) | meshPg (consumer side) | Consumer looks up the reporting user's email (`GLUSR_USR_EMAIL` / `GLUSR_USR_EMAIL2` fallback) before sending the internal notification mail | `GLUSR_USR_ID`, `GLUSR_USR_EMAIL`, `GLUSR_USR_EMAIL2` — [`USER_FEEDBACK.go:250-262`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_FEEDBACK.go) |
| `glusr_usr` (read, App-Rating side) | mesh_pg_user | App-Rating flow reads `glusr_usr_firstname` / `glusr_usr_email` when the caller didn't send `FIRSTNAME` or `email` inline | [`UserFeedbackModel.go:292-346,384-394`](../internal/models/users/UserFeedbackModel.go) |

**App-Rating Feedback owns no table directly** — confirmed in this pass: its actual
persistence is the `MY_FEEDBACKS` insert/update done by the write-repo's `FeedbackUpdateDB`,
reached via HTTP loopback (§4.13-14, §10 Flow C).

---

## 4. Business Rules & Validation (code se exhaustive list)

### My Feedback / IMSearch Feedback (shared write-controller)

1. **`FEEDBACK_FLAG` dispatches the flow**: defaults to `MY_FEEDBACKS` if absent;
   `IMSEARCH_FEEDBACK` routes to the search-feedback path; anything else → hard reject
   `"FEEDBACK_FLAG is not same as desired."`
   [`UserFeedbackController.go:81-90,296-298`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserFeedbackController.go)
2. **Top-level mandatory fields (`FEEDBACK` flag)**: `VALIDATION_KEY`, `FEEDBACK_FLAG`, `IP`
   all required before anything else runs — `"Please Enter Mandatory(VALIDATION_KEY/
   FEEDBACK_FLAG/IP) Fields"`.
   [`UserUtilsMandatory.go:1071-1090`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
3. **Gateway allowlist**: `GLADMIN`, `Weberp`, `MAPI`, `LEAP`, `M.INDIAMART.COM`, `DIR`,
   `FCP`, `BUYERS_FEEDBACK`, `SELLERMY`, `MY`, `EXPORT`, `EXPORTM`.
   [`UserFeedbackController.go:100-105`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserFeedbackController.go)
4. **My-Feedback mandatory-field state machine is genuinely conditional, not a flat list**
   (`MandatoryFieldsFeedback`, `MY_FEEDBACKS` branch):
   - `FEEDBACKS_ID` present but `my_feedbacks_only_date` empty → reject.
   - `my_feedbacks_only_date` present but `FEEDBACKS_ID` empty → reject.
   - Both present (**update mode**, `flagUpd=1`): `APPROVAL_STATUS`, if given, must be `A` or
     `D` — anything else rejected; and if `APPROVAL_STATUS` is empty, `WORKLOG_REMARK` must be
     given (or reject `"In case of Update either status or Work log has to be given"`).
   - Neither present (**insert mode**): `GLUSR_ID`, `FEEDBACKS_TITLE`, `FEEDBACKS_SOURCE`,
     `FEEDBACKS_USR_TYPE`, `FEEDBACKS_MODID` all mandatory.
   - Optional `FLAG` param, if given, must be exactly `"MY"` else `"Invalid FLAG value"`.
   [`UserUtilsMandatory.go:1091-1152`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
5. **My-Feedback insert-vs-update (at DB layer)**: driven by `APPROVAL_STATUS` presence AND
   `flagUpd` (derived in the controller from `FEEDBACKS_ID` + `my_feedbacks_only_date` both
   being present) — if `status==""` and `flagUpd!=1` → INSERT; else (feedback-ID present) →
   dynamic whitelist-driven UPDATE using `UserFeedbackMapMyFeedbacks` as the column allowlist.
   [`UserFeedbackModel.go:182-296`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFeedbackModel.go)
6. **Mobile-number normalization**: accepts either 10-digit or 12-digit (`91`-prefixed)
   mobile matching `components.Regex["MOBILE"]`; strips the `91` prefix before storage;
   anything else rejected `"FEEDBACKS_MOBILE is not correct"`.
   [`UserFeedbackController.go:140-155`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserFeedbackController.go)
7. **Email format check**: `FEEDBACKS_EMAIL`, if given, must match
   `components.Regex["EMAIL_RFC822"]` else rejected.
   [`UserFeedbackController.go:157-164`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserFeedbackController.go)
8. **Length/type validation** runs after the above, via `ValidateUserFeedback` →
   `LengthAndTypeValidations_v2` against `UserFeedbackMapMyFeedbacks` — notable limits:
   `FEEDBACKS_DESCRIPTION` ≤ 5000 chars, `FEEDBACKS_TITLE` ≤ 500, `FEEDBACKS_MOBILE` ≤ 40,
   `FEEDBACKS_EMAIL` ≤ 100, `WORKLOG_REMARK` ≤ 500, `EMP_REMARKS` ≤ 100.
   [`UsersValidationMaps.go:115-231,2147-2165`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go)
9. **"Vague" auto-classification**: if `ISSUE_IS_VAGUE` not explicitly given, it's derived —
   description length `< 6` chars → vague (`1`), else not (`0`).
   [`UserFeedbackModel.go:58-66`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFeedbackModel.go)
10. **`ISSUE_TYPE_ID` validated against `MY_FEEDBACK_ISSUE_TYPE`** via a pre-check
    `SELECT COUNT(1)` (`FeedbackCheckIssueTypeID`) — invalid/unrecognized/non-numeric IDs are
    silently nulled, not rejected. [`UserFeedbackModel.go:18-44`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFeedbackModel.go)
11. **On My-Feedback insert/update success, a RabbitMQ event fires** (`SERVICENAME=
    USER_FEEDBACK`) carrying the full feedback payload plus a `host` (lowercased environment)
    tag and `UNIQUE_LOGGING_ID`. [`UserFeedbackModel.go:298-346`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFeedbackModel.go)
12. **IMSearch-Feedback insert-vs-update**: driven purely by `feedback_id` presence — no
    feedback-ID → INSERT (requires `SEARCH_QUERY` and `SEARCH_TOTAL_RESULTS` non-empty, else
    rejected `"SEARCH_QUERY/SEARCH_TOTAL_RESULTS can not be empty."`); feedback-ID present →
    dynamic whitelist-driven UPDATE against `UserFeedbackMapImsearchFeedback`.
    `IM_SEARCH_SUGGESTED_MCATS` is JSON-marshaled before storage in both paths.
    [`UserFeedbackModel.go:352-626`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFeedbackModel.go)
13. **No RabbitMQ event on IMSearch-Feedback** — only My-Feedback pushes to the `USER_FEEDBACK`
    queue; IMSearch-Feedback is a pure DB write with no fan-out.
14. **`SEARCH_URL`/`SEARCH_PAGE_REFERRER` are app-truncated to 2048 chars** before validation,
    rather than rejecting over-length input. [`UserFeedbackController.go:246-259`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserFeedbackController.go)
15. **IMSearch-Feedback UPDATE runs with a much tighter timeout (80ms)** than every other
    query in this domain (which use 1s) — [`UserFeedbackModel.go:602`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFeedbackModel.go)
    vs. 1-second timeouts elsewhere in the same file (e.g. line 24, 223). This asymmetry is
    unexplained in code — **[INFERRED — likely deliberate, given IMSearch-Feedback is
    higher-volume/lower-criticality traffic, confirm intent with team]**.

### App-Rating Feedback (self-contained-looking, but actually a wrapper — read-repo)

16. **`glusrid`, if given, must be numeric** — non-numeric or explicitly-empty `glusrid`
    rejected with response-code family `{"7","","38"}` (empty) / `{"7","","37"}` (non-numeric).
    [`UserFeedbackController.go:53-66`](../internal/controllers/UsersControllers/UserFeedbackController.go)
17. **`app_rating` must be single-digit and numeric** — non-numeric rejected
    (`{"4","","109"}`), length > 1 rejected (`{"4","","110"}`).
    [`UserFeedbackController.go:71-80`](../internal/controllers/UsersControllers/UserFeedbackController.go)
18. **`app_version_no` capped at 10 characters** — longer values rejected (`{"4","","111"}`).
    [`UserFeedbackController.go:81-86`](../internal/controllers/UsersControllers/UserFeedbackController.go)
19. **App-version gating (routing-level)**: `ANDROID` requires version `> 12.4.1`, `IOS`
    requires version `> 12.0.0` (via `CompareVersions`); below that,
    `modidVersionCheck=false` is computed and passed down — it does **not** block the
    request, it only **changes the response-shape** (`RESPONSE{...}` wrapped/uppercased vs.
    flat lowercased keys) returned to old clients on every branch of `UsersFeedback`.
    [`UserFeedbackController.go:96-121`](../internal/controllers/UsersControllers/UserFeedbackController.go),
    confirmed by every `if modidVersionCheck { ... } else { ... }` branch in
    [`UserFeedbackModel.go`](../internal/models/users/UserFeedbackModel.go) (lines
    225-243, 259-278, 330-345, 452-468, 476-494, 501-518, 558-574).
20. **Unauthenticated-capable, on-the-fly account creation**: if `glusrid` is empty, a
    **first HTTP-loopback (`CurlReqWithRetry POST`, circuit-breaker wrapped via
    `config.CB_api.Execute`) hits `APIList["user_add"]`** (→ `service.intermesh.net/user/add`
    in prod, per `data/config.yaml:845`) with a `forNewUserCreation` payload — country-code
    `91` maps to `FK_GL_COUNTRY_ISO = "IN"`. 3 retries, 100ms connect/request timeout each.
    [`UserFeedbackModel.go:186-280`](../internal/models/users/UserFeedbackModel.go)
21. **A second HTTP-loopback (`CurlReqWithRetry POST`, also circuit-breaker wrapped) hits
    `APIList["feedbackwapi"]`** (→ `service.intermesh.net/feedback` in prod, per
    `data/config.yaml:837`) — **this is the exact same write-repo `/feedback` endpoint as
    My-Feedback** (§2). Since `FEEDBACK_FLAG` is never sent in this payload, the write-repo
    defaults it to `MY_FEEDBACKS`, meaning **every App-Rating submission is, underneath, a
    My-Feedback INSERT into `MY_FEEDBACKS`** — same validation, same RabbitMQ push, same
    consumer. [`UserFeedbackModel.go:397-419`](../internal/models/users/UserFeedbackModel.go)
22. **`FEEDBACKS_TITLE` is synthesized, not user-supplied**: always `"Feedback from " +
    modid` (e.g. `"Feedback from ANDROID"`). [`UserFeedbackModel.go:163`](../internal/models/users/UserFeedbackModel.go)
23. **Rating-based branch skips the follow-up email**: if `app_rating` is in the set
    `{"4","5","6","8"}`, the function returns success **without** pushing to
    `USER_RELATED_SEND_MAIL` — for any other rating value (including empty), a
    customer-facing email is queued (§5, §10 Flow C). Business meaning of exactly
    `{4,5,6,8}` (out of what scale, and why not 1/2/3/7/9/10) is **not documented anywhere
    in code** — **[INFERRED — likely "satisfied" ratings that don't need a follow-up
    outreach mail; confirm exact rating-scale semantics with app/product team]**.
    [`UserFeedbackModel.go:497-523`](../internal/models/users/UserFeedbackModel.go)
24. **IP is spoofed to a fixed India IP/country for old app versions**: if the caller's app
    version is below a second, *independent* threshold (`ANDROID < 12.3`, `IOS < 11.4`, a
    different cutoff than the response-shape gate in point 19), the request's `IP` and
    `IP_COUNTRY` are hardcoded to `"5.5.5.5"` / `"India"` before being forwarded to
    `user_add` — an undocumented legacy compatibility shim.
    [`UserFeedbackModel.go:136-155`](../internal/models/users/UserFeedbackModel.go)

---

## 5. RabbitMQ

| Queue / `SERVICENAME` | Publisher | Consumer | Purpose |
|---|---|---|---|
| `USER_FEEDBACK` | `UserFeedbackModel.go` (`FeedbackUpdateDB`) — **My-Feedback path only** (which, per §4.21, also includes every App-Rating submission since it lands here too), on insert/update success | [`USER_FEEDBACK.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_FEEDBACK.go) (`dbActionUserFeedback`) — connects to both `meshPg` and `approvalPg` | Sends an **internal** notification mail to the feedback/support team (not the submitting user) and syncs a second copy of the `MY_FEEDBACKS` row into `approvalPg` (§6) |
| `USER_RELATED_SEND_MAIL` with `SERVICE_NAME=USER_FEEDBACK` (payload key, not the RabbitMQ SERVICENAME) | `UserFeedbackModel.go` (`UsersFeedback`, App-Rating flow, read-repo) — **only when email is known and rating not in `{4,5,6,8}`** | [`USER_RELATED_SEND_MAIL.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RELATED_SEND_MAIL.go) (`PerformBackendOpeation` → `SendMail`) — a generic mail-relay consumer **shared with an unrelated `PAY_NOW` flow**, dispatching on the `SERVICE_NAME` payload key | Sends a **customer-facing** confirmation/follow-up mail to the App-Rating submitter (template `668` for app-modids, `669` otherwise) |

**Two structurally different mail paths in one domain** — a subtlety worth flagging: the
`USER_FEEDBACK` RabbitMQ queue's own consumer (`USER_FEEDBACK.go`) mails the *internal team*
(`feedback@indiamart.com`); the App-Rating flow's mail goes through a completely different
queue/consumer (`USER_RELATED_SEND_MAIL`) to mail the *end user*. Both can theoretically fire
for the same App-Rating submission (since it also triggers the `USER_FEEDBACK` queue via the
shared write-path), meaning a single app-rating submission can generate **two independent
emails to two different audiences**.

---

## 6. Kafka

**Koi Kafka usage nahi mila** in any of the three sub-flows across all three repos (grepped
`Kafka` in `UserFeedbackModel.go` both repos and `USER_FEEDBACK.go` — zero hits).

---

## 7. Redis

**Koi Redis usage nahi mila** in any of the three sub-flows (grepped `Redis`/`redis` across
all feedback-domain files in both write and read repos — zero hits). Every read/write in this
domain hits Postgres directly.

---

## 8. Cron Inventory

Grepped `feedback` (case-insensitive) across `crons/` in both `service-api-go-production` and
`user-temp-consumers-production`:

| Cron | Live hai? | Notes |
|---|---|---|
| `crons/recommend/mcatcrontest.go`, `crons/recommend/new_recom_csl_pg.go` | **Not this domain** | The word "feedback" appears only in a commented-out, dead variable/goroutine (`// var feedbackch chan Feedback`, `// go update_feedback(...)`) inside an unrelated product-recommendation cron — [`mcatcrontest.go:121,337`](../../service-api-go-production/service-api-go-production/crons/recommend/mcatcrontest.go). Not related to My-Feedback/IMSearch-Feedback/App-Rating-Feedback. |

**No dedicated cron exists for the Feedback domain covered by this doc** — confirmed by grep,
not assumed.

---

## 9. Full Flow Diagram (Lucid, icon-based)

**[Poora Feedback flowchart yahan dekho](https://lucid.app/lucidchart/0908bb38-1883-4f8e-86d6-8ef1e7461877/edit)**

---

## 10. End-to-End Technical Flows (step by step, code-level)

### Flow A — My Feedback submit (insert)

```
User (via GLADMIN/Weberp/MAPI/LEAP/M.INDIAMART.COM/DIR/FCP/BUYERS_FEEDBACK/SELLERMY/
      MY/EXPORT/EXPORTM)
    │
    ▼
[API — write]  POST /feedback  {FEEDBACK_FLAG: "MY_FEEDBACKS" (default), GLUSR_ID,
                 FEEDBACKS_TITLE, FEEDBACKS_SOURCE, FEEDBACKS_USR_TYPE, FEEDBACKS_MODID, ...}
    │  UserFeedbackController.go
    │  1. Top-level mandatory check (VALIDATION_KEY/FEEDBACK_FLAG/IP)
    │  2. Gateway allowlist check
    │  3. MY_FEEDBACKS mandatory-field state machine (insert branch)
    │  4. Mobile normalize + email format check
    │  5. Length/type validation (ValidateUserFeedback)
    ▼
[DB — meshpg]  SELECT COUNT(1) FROM MY_FEEDBACK_ISSUE_TYPE (ISSUE_TYPE_ID pre-check)
    ▼
[DB — meshpg]  INSERT INTO MY_FEEDBACKS ... RETURNING MY_FEEDBACKS_ID
    ▼
[RabbitMQ publish]  SERVICENAME=USER_FEEDBACK  {full payload + host}
    ▼
[CONSUME]  USER_FEEDBACK.go → dbActionUserFeedback
    │  if feedback_desc non-empty AND modid in {ANDROID, IOS, HELPIM}:
    ├─ getglusrdata()          → SELECT email from glusr_usr (meshPg)
    ├─ mail_attachment()       → SELECT issue-type text from MY_FEEDBACK_ISSUE_TYPE (meshPg)
    ├─ sendFeedbackMailToCC()  → internal mail to feedback@indiamart.com (soa.user@... in DEV)
    │                             template 2106 (2804 in DEV); rate-limited (429 on threshold)
    └─ updateDataInMESHPGFeedback() → UPDATE MY_FEEDBACKS SET MY_FEEDBACKS_TYPE_FLAG='M' (meshPg)
    │
    ▼
[DB — approvalPg]  InsertIntoPG() → SELECT COUNT(1) ... then INSERT/UPDATE MY_FEEDBACKS
                     (a SECOND, independent copy of the same feedback row)
```

### Flow B — My Feedback update (worklog/status change by internal team)

```
Internal team (GLADMIN etc.)
    │
    ▼
[API]  POST /feedback  {FEEDBACKS_ID, my_feedbacks_only_date, APPROVAL_STATUS: "A"|"D"
                          and/or WORKLOG_REMARK, ISSUE_TYPE_ID?}
    │  Mandatory-check requires FEEDBACKS_ID + my_feedbacks_only_date together (flagUpd=1)
    ▼
[DB — meshpg]  dynamic UPDATE MY_FEEDBACKS SET <whitelisted columns> WHERE MY_FEEDBACKS_ID=$n
    ▼
[RabbitMQ publish]  SERVICENAME=USER_FEEDBACK (same as Flow A — update also re-fires the queue)
    ▼
[CONSUME]  same dbActionUserFeedback path as Flow A
```

### Flow C — App-Rating Feedback (in-app rating popup, possibly unauthenticated)

```
User (app, glusrid may be empty)
    │
    ▼
[API — read-repo]  GET/POST feedback/*params  {glusrid?, mobile?, app_rating, modid,
                     app_version_no, msg?, email?, ...}
    │  UserFeedbackController.go (read) — glusrid/app_rating/app_version_no format checks
    │  computes modidVersionCheck (response-shape gate, §4.19)
    ▼
UsersFeedback()
    ├─ glusrid empty? → [HTTP POST loopback, circuit-breaker, 3 retries]
    │                    APIList["user_add"] → creates a new GLID on-the-fly
    │                    (fails here → early return, no feedback ever written)
    ├─ FIRSTNAME/email missing? → [DB — mesh_pg_user] SELECT glusr_usr_firstname/email
    ▼
    [HTTP POST loopback, circuit-breaker, 3 retries]  APIList["feedbackwapi"]
        → hits write-repo POST /feedback  {no FEEDBACK_FLAG → defaults to MY_FEEDBACKS}
        → this re-enters Flow A in full: mandatory-check, insert, RabbitMQ → USER_FEEDBACK
          consumer → internal team mail + approvalPg sync
    ▼
    app_rating in {"4","5","6","8"}?
    ├─ YES → return success immediately, NO customer-facing email
    └─ NO  → [RabbitMQ publish]  USER_RELATED_SEND_MAIL  {to: email, modid, firstname,
                                   SERVICE_NAME: "USER_FEEDBACK"}
             ▼
             [CONSUME]  USER_RELATED_SEND_MAIL.go → SendMail()
                 → customer-facing mail, template 668 (app-modids) / 669 (other modids)
    ▼
Response relayed back to caller (shape depends on modidVersionCheck)
```

### Flow D — IMSearch Feedback (insert or update)

```
User / search-widget caller
    │
    ▼
[API — write]  POST /feedback  {FEEDBACK_FLAG: "IMSEARCH_FEEDBACK", SEARCH_QUERY,
                 SEARCH_TOTAL_RESULTS, feedback_id?, ...}
    │  URL/referrer truncated to 2048 chars
    │  ValidateUserFeedback("IMSEARCH_FEEDBACK")
    ▼
feedback_id absent?
    ├─ YES → [DB — meshpg]  INSERT INTO IM_SEARCH_FEEDBACK ... RETURNING IM_SEARCH_ID
    │         (requires SEARCH_QUERY + SEARCH_TOTAL_RESULTS non-empty)
    └─ NO  → [DB — meshpg]  dynamic UPDATE IM_SEARCH_FEEDBACK ... (80ms timeout, tighter
              than My-Feedback's 1s)
    ▼
No RabbitMQ push — this is the only sub-flow in the domain with zero downstream fan-out.
```

---

## 11. Flow-wise DB & Table Usage — Kaun sa DB, Kaun sa Table, Kis Liye

### Flow A — My Feedback Insert

| # | DB (physical) | Table | Operation | Kya nikala/likha jaata hai, aur kyun |
|---|---|---|---|---|
| 1 | meshpg | `MY_FEEDBACK_ISSUE_TYPE` | SELECT (`COUNT(1)`) | Pre-check ki `ISSUE_TYPE_ID` valid hai ya nahi — invalid ho toh silently null kar diya jaata hai |
| 2 | meshpg | `MY_FEEDBACKS` | INSERT ... RETURNING | Primary feedback record likha jaata hai, naya `MY_FEEDBACKS_ID` mil jaata hai |
| 3 | meshPg (consumer side) | `glusr_usr` | SELECT | Reporting user ka email nikala jaata hai internal notification-mail ke liye |
| 4 | meshPg (consumer side) | `MY_FEEDBACK_ISSUE_TYPE` | SELECT | Issue-type ka display-text nikala jaata hai mail body ke liye |
| 5 | meshPg (consumer side) | `MY_FEEDBACKS` | UPDATE | `MY_FEEDBACKS_TYPE_FLAG='M'` set hota hai mail-sent-confirmation ke roop mein |
| 6 | **approvalPg** | `MY_FEEDBACKS` | SELECT (`COUNT(1)`) phir INSERT/UPDATE | Ek **doosri, independent copy** ka same feedback record approvalPg mein likha jaata hai |

**Total round-trips ek single My-Feedback insert ke liye: kam se kam 6**, do alag physical
databases (meshpg, approvalPg) pe, plus ek internal mail-send external call.

### Flow B — My Feedback Update

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `MY_FEEDBACKS` | dynamic UPDATE (whitelisted columns) | Status/worklog/issue-type update karta hai |
| 2-6 | *(same as Flow A steps 3-6, kyunki update bhi RabbitMQ re-fire karta hai)* | — | — | Same mail + approvalPg-sync chain dobara chalti hai har update pe bhi |

### Flow C — App-Rating Feedback

| # | DB / External | Table/Target | Operation | Kyun |
|---|---|---|---|---|
| 1 | External HTTP (`user_add` API) | — | POST, circuit-breaker + 3 retries | Sirf jab `glusrid` empty ho — naya user create karta hai |
| 2 | mesh_pg_user | `glusr_usr` | SELECT | `FIRSTNAME`/`email` nikalta hai agar caller ne nahi diya |
| 3 | External HTTP (`feedbackwapi` = write-repo `/feedback`) | — | POST, circuit-breaker + 3 retries | Yeh call khud **Flow A ke saare steps 1-6 trigger karta hai** — App-Rating ka apna alag DB-path nahi hai, yeh Flow A ko wrap karta hai |
| 4 | RabbitMQ | `USER_RELATED_SEND_MAIL` queue | Publish (conditional, rating-based) | Customer-facing follow-up mail queue karta hai |

**Business-critical insight**: App-Rating Feedback ka DB-cost = Flow A ka poora DB-cost
(6 round-trips) **plus** 2 extra HTTP-loopback network hops **plus** ek optional DB read for
firstname/email — yeh domain ka sabse expensive single sub-flow hai per-request.

### Flow D — IMSearch Feedback

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `IM_SEARCH_FEEDBACK` | INSERT ... RETURNING (ya UPDATE agar `feedback_id` diya gaya) | Search-quality feedback record likhta/update karta hai |

**Sabse cheap flow iss domain mein** — ek hi DB round-trip, koi RabbitMQ nahi, koi mail nahi.

---

## 12. Optimization Scope — DB/Response-Time Contribution

### High-impact

1. **App-Rating Feedback ka critical path mein 2 sequential external HTTP-loopback calls
   hain**, dono `CurlReqWithRetry` (3 retries) ke through — `user_add` (conditional, sirf
   unauthenticated case mein) aur `feedbackwapi` (hamesha). Dono synchronous hain aur agle
   step ko block karte hain. Worst case (naya user + retries) mein yeh 6 network attempts
   tak ban sakta hai ek hi request ke liye. **Suggestion**: agar `glusrid` zyadatar present
   rehta hai (logged-in-app-user ka common case), yeh cost avoid ho jaata hai — lekin
   unauthenticated/rating-popup ke liye yeh ek meaningful added-latency chain hai jise
   monitor karna chahiye.
2. **App-Rating Feedback ka "feedbackwapi" loopback dobara poora My-Feedback validation +
   insert + RabbitMQ chain trigger karta hai** — effectively domain ka sabse heavy flow
   (Flow A) har single app-rating submission ke liye implicitly chalta hai, plus consumer-side
   ka mail + approvalPg-sync overhead bhi. Agar App-Rating volume high ho, yeh My-Feedback
   consumer/queue pe indirect load daalta hai jo shayad team ko pata na ho.

### Medium-impact

3. **`FeedbackCheckIssueTypeID` ek sequential pre-check SELECT hai main insert se pehle** —
   same "extra round-trip before the real write" pattern jo baaki KT series (GST, Rating) mein
   bhi flag hua hai; issue-type list chhoti aur slow-changing honi chahiye (Redis-candidate,
   lekin domain mein abhi Redis kahin nahi hai).
4. **My-Feedback ek hi event ke liye do alag physical databases (meshpg + approvalPg) pe
   likhta hai**, ek synchronous API-side write aur ek asynchronous consumer-side write — agar
   approvalPg down ho ya slow ho, consumer retries/backlog ban sakta hai (RabbitMQ Nack pe
   requeue), lekin user ko already "SUCCESS" mil chuka hota hai meshpg-write ke baad hi.
   Eventual-consistency window hai jo team ko pata hona chahiye.
5. **IMSearch-Feedback ka UPDATE query timeout (80ms) baaki domain (1s) se bahut tight hai**
   [`UserFeedbackModel.go:602`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFeedbackModel.go)
   — agar meshpg latency kabhi thodi bhi spike kare, yeh specific update path baaki sabse
   pehle timeout hoga. **[INFERRED — likely deliberate given traffic volume, confirm]**.

### Low-impact / good-practice already present

6. **IMSearch-Feedback ka koi RabbitMQ fan-out nahi hai** — simplest/lowest-overhead sub-flow
   iss domain mein, appropriate given yeh high-volume/lower-per-record-criticality data hai.

---

## 13. Edge Cases & Gotchas (technical POV)

1. **Invalid `ISSUE_TYPE_ID` silently nulled, not rejected** — typo'd ID koi error nahi deta,
   feedback bas bina classification ke save ho jaata hai.
2. **My-Feedback aur IMSearch-Feedback ek controller share karte hain but diverge
   significantly insert/update-detection logic mein** (`flagUpd`-based vs pure
   `feedback_id`-presence) — casual reading mein confuse hona easy hai.
3. **App-Rating Feedback ka "own" data-model file (`internal/models/users/UserFeedbackModel.go`)
   koi table nahi likhta** — poori persistence ek HTTP call ke peeche chhupi hai. Is baar yeh
   HTTP call trace karke confirm ho gaya ki yeh My-Feedback ka hi write-path hai (§4.21) — pehle
   pass mein yeh sirf ek open question tha.
4. **Ek App-Rating submission do independent emails trigger kar sakta hai**: ek internal
   (`feedback@indiamart.com`, via `USER_FEEDBACK` consumer) aur ek customer-facing (via
   `USER_RELATED_SEND_MAIL` consumer, sirf non-{4,5,6,8} ratings ke liye) — dono alag queues,
   alag consumers, alag templates.
5. **`USER_RELATED_SEND_MAIL` consumer ek shared/generic consumer hai `PAY_NOW` flow ke
   saath** — agar iss consumer mein koi change karna ho Feedback ke liye, `PAY_NOW` templates/
   logic bhi impact ho sakti hai, dhyaan se dispatch-logic (`SERVICE_NAME` switch) padhna
   zaroori hai. [`USER_RELATED_SEND_MAIL.go:72-98`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RELATED_SEND_MAIL.go)
6. **Consumer-side `feedback_mail` internal notification mail hai rate-limited** — agar
   threshold exceed ho jaaye, error `"THRESHOLD LIMIT EXCEEDED"` specifically 429 status code
   ke saath logged hota hai; baaki mail failures generic 502. [`USER_FEEDBACK.go:227-234`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_FEEDBACK.go)
7. **`formatFeedbackDesc` ek structured mini-format parse karta hai**
   (`Reason[...]=...Sub[...]=...Comment=...`) — App/mobile clients feedback description ko
   ek encoded reason/sub-reason string ke roop mein bhejte hain, plain free-text nahi hamesha.
   Agar app-side format kabhi change ho, yeh parser silently degrade ho sakta hai (partial
   lines drop ho jaate hain agar bracket-matching fail ho). [`USER_FEEDBACK.go:383-448`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_FEEDBACK.go)
8. **Two independent, differently-thresholded "old app version" checks exist in the App-Rating
   flow** — one (§4.19) only changes response-shape, another (§4.24) spoofs IP/country. Easy
   to conflate the two when debugging old-client behavior.
9. **App-Rating's `FEEDBACKS_TITLE` is always machine-generated** (`"Feedback from " +
   modid`) — a caller cannot set a custom title through this sub-flow, unlike raw My-Feedback.

---

## 14. Open Questions

1. Exact business meaning of `app_rating` values `{"4","5","6","8"}` as the "skip follow-up
   mail" set (§4.23) — what scale is `app_rating` on, and why these specific four values?
2. Mail template `2106`/`2804` (internal team mail) and `668`/`669` (customer mail) —
   template content itself not reviewed in this pass; only IDs/trigger-conditions confirmed.
3. Why IMSearch-Feedback's UPDATE uses an 80ms timeout vs. 1s everywhere else in the same
   file (§4.15, §12.5) — deliberate or oversight?
4. Live DB schema verification (column types, nullability, indexes, constraints) for all
   tables in section 3 — this doc only reflects what Go SQL strings imply.
5. `RabbitMQ` retry/dead-letter behavior on `Nack` for both `USER_FEEDBACK` and
   `USER_RELATED_SEND_MAIL` consumers — not traced in this pass (queue-infra config, not
   domain code).

---

## See also

- [`Feedback_Business_Doc.md`](./Feedback_Business_Doc.md) — product perspective
- [`../Social Review KT/Social_Review_Technical_Doc.md`](../Social%20Review%20KT/Social_Review_Technical_Doc.md) —
  unrelated concept (external social-media reviews)
- [`../Rating Usefulness KT/Rating_Usefulness_Technical_Doc.md`](../Rating%20Usefulness%20KT/Rating_Usefulness_Technical_Doc.md),
  [`../Update With OTP KT/Update_With_OTP_Technical_Doc.md`](../Update%20With%20OTP%20KT/Update_With_OTP_Technical_Doc.md) —
  same HTTP-loopback architectural pattern used by App-Rating Feedback
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — depth/structure
  reference this doc was rebuilt against

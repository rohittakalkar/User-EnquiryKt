# Feedback — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`Feedback_Business_Doc.md`](./Feedback_Business_Doc.md)
dekho.

**Scope note**: 3 feedback-types, ek folder mein combined kyunki My-Feedback aur
IMSearch-Feedback ek hi controller/endpoint share karte hain; App-Rating-Feedback alag
endpoint hai but same domain-concept hai. **Social Review** (external social-media
reviews) genuinely alag concept hai, dekho
[`../Social Review KT/`](../Social%20Review%20KT/).

**Repos**: `service-api-go-production` (write: My Feedback + IMSearch Feedback),
`users-api-go-production` (App-Rating Feedback — a self-contained flow with its own
HTTP-loopback), `user-temp-consumers-production` (consumer for My Feedback).

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| My Feedback + IMSearch Feedback write | write | [`UserFeedbackController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserFeedbackController.go), [`UserFeedbackModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFeedbackModel.go) (`FeedbackUpdateDB`, `ImSearchFeedbackUpdateDB`, `FeedbackCheckIssueTypeID`) |
| Validation | write | `MandatoryFieldsFeedback` — [`UserUtilsMandatory.go:1068`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go), `ValidateUserFeedback`, `UserFeedbackMapMyFeedbacks`, `UserFeedbackMapImsearchFeedback` — [`UsersValidationMaps.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |
| My Feedback consumer | consumers | [`USER_FEEDBACK.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_FEEDBACK.go) (`dbActionUserFeedback`) |
| App-Rating Feedback (self-contained, unauth-capable) | read | [`UserFeedbackController.go`](../../users-api-go-production/internal/controllers/UsersControllers/UserFeedbackController.go) (`UserFeedback`), [`UserFeedbackModel.go`](../../users-api-go-production/internal/models/users/UserFeedbackModel.go) (`UsersFeedback`) |

---

## 2. Routes

| Method | serviceName | Repo | Controller |
|---|---|---|---|
| POST | `FEEDBACK_SERVICE` (`FEEDBACK_FLAG=MY_FEEDBACKS` default, or `IMSEARCH_FEEDBACK`) | write | `UserFeedbackController` |
| — | `USERFEEDBACK` | read | `UserFeedback` (App-Rating Feedback) |

---

## 3. Data Model — Tables

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `MY_FEEDBACKS` | meshpg | General feedback (My Feedback) | `MY_FEEDBACKS_ID` (PK, `RETURNING`), `MY_FEEDBACKS_MOBILE`, `FK_GLUSR_ID`, `MY_FEEDBACKS_DESCRIPTION`, `MY_FEEDBACKS_EMAIL`, `MY_FEEDBACKS_DATE`, `MY_FEEDBACKS_COUNTRY`, `MY_FEEDBACKS_SOURCE`, `MY_FEEDBACKS_USR_TYPE`, `MY_FEEDBACKS_MODID`, `MY_FEEDBACKS_TITLE`, `MY_FEEDBACKS_RATINGS`, `MY_FEEDBACKS_APP_VER`, `MY_FEEDBACKS_APPROVAL_STATUS` (`'P'` hardcoded on insert), `MY_FEEDBACKS_ISSUE_IS_VAGUE`, `FK_FEEDBACK_ISSUE_TYPE_ID`, `MY_FEEDBACKS_EMP_REMARKS` — [`UserFeedbackModel.go:183-217`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFeedbackModel.go) |
| `MY_FEEDBACK_ISSUE_TYPE` | meshpg (lookup) | Valid issue-type-ID whitelist | `FEEDBACK_ISSUE_TYPE_ID` — [`UserFeedbackModel.go:20`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFeedbackModel.go) |
| `IM_SEARCH_FEEDBACK` | meshpg | Search-quality feedback | `IM_SEARCH_ID` (PK, `RETURNING`), `IM_SEARCH_QUERY`, `IM_SEARCH_DATE`, `IM_SEARCH_PAGE_NO`, `IM_SEARCH_URL` (max 2048 chars, app-truncated), `IM_SEARCH_TOP_RESULTS`, `IM_SEARCH_PAGE_SCROLL`, `RATING`, `COMMENTS`, `COOKIE`, `IM_SEARCH_TOTAL_RESULTS`, `IM_SEARCH_MOBILE`, `FK_GLUSR_USR_ID`, `IM_SEARCH_FEEDBACK_PRD_POSITION`, `IM_SEARCH_QUERY_UPDATED`, `IM_SEARCH_PAGE_REFERRER` (max 2048), `IM_SEARCH_SUGGESTED_MCATS` (JSON), `IM_SEARCH_CATEGORY_ID`, `IM_SEARCH_MODID` — [`UserFeedbackModel.go:473-492`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFeedbackModel.go) |

**App-Rating Feedback**: no direct DB-write found in `UserFeedbackModel.go` (read-repo) —
it operates via **two HTTP-loopback calls** (see §4) rather than direct SQL, so no table
is owned by this sub-flow's own code.

---

## 4. Business Rules & Validation (code se)

### My Feedback / IMSearch Feedback (shared write-controller)

1. **`FEEDBACK_FLAG` dispatches the flow**: defaults to `MY_FEEDBACKS` if absent;
   `IMSEARCH_FEEDBACK` routes to the search-feedback path; anything else → hard reject
   `"FEEDBACK_FLAG is not same as desired."`
   [`UserFeedbackController.go:81-90,296-298`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserFeedbackController.go)
2. **Gateway allowlist**: `GLADMIN`, `Weberp`, `MAPI`, `LEAP`, `M.INDIAMART.COM`, `DIR`,
   `FCP`, `BUYERS_FEEDBACK`, `SELLERMY`, `MY`, `EXPORT`, `EXPORTM`.
3. **My-Feedback insert-vs-update**: driven by `APPROVAL_STATUS` presence AND `flagUpd`
   (itself derived from `FEEDBACKS_ID` + `my_feedbacks_only_date` both being present) —
   if `status==""` and `flagUpd!=1` → INSERT; else (feedback-ID present) → dynamic
   whitelist-driven UPDATE.
   [`UserFeedbackModel.go:182-296`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFeedbackModel.go)
4. **Mobile-number normalization**: accepts either 10-digit or 12-digit
   (`91`-prefixed) mobile, strips the `91` prefix before storage; anything else rejected.
   [`UserFeedbackController.go:140-155`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserFeedbackController.go)
5. **"Vague" auto-classification**: if `ISSUE_IS_VAGUE` not explicitly given, it's
   derived — description length `< 6` chars → vague (`1`), else not (`0`).
   [`UserFeedbackModel.go:58-66`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFeedbackModel.go)
6. **`ISSUE_TYPE_ID` validated against `MY_FEEDBACK_ISSUE_TYPE`** via a pre-check
   `SELECT COUNT(1)` — invalid/unrecognized IDs are silently nulled, not rejected.
7. **On My-Feedback insert/update success, a RabbitMQ event fires** (`SERVICENAME=
   USER_FEEDBACK`) with the full feedback payload plus a `host` (environment) tag.
8. **IMSearch-Feedback insert-vs-update**: driven purely by `feedback_id` presence — no
   feedback-ID → INSERT (requires `SEARCH_QUERY` and `SEARCH_TOTAL_RESULTS` non-empty,
   else rejected); feedback-ID present → dynamic whitelist-driven UPDATE.
   `SEARCH_SUGGESTED_MCATS` is JSON-marshaled before storage in both paths.
   [`UserFeedbackModel.go:471-621`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFeedbackModel.go)
9. **No RabbitMQ event on IMSearch-Feedback** — only My-Feedback pushes to the queue.
10. **`SEARCH_URL`/`SEARCH_PAGE_REFERRER` are app-truncated to 2048 chars** before
    validation, rather than rejecting over-length input.

### App-Rating Feedback (self-contained, read-repo)

11. **App-version gating**: `ANDROID` requires version `> 12.4.1`, `IOS` requires
    version `> 12.0.0` (via `CompareVersions`) — below that, `modidVersionCheck=false`
    is passed through (behavior on false not fully traced in this pass).
    [`UserFeedbackController.go:105-119`](../../users-api-go-production/internal/controllers/UsersControllers/UserFeedbackController.go)
12. **`app_rating` must be single-digit numeric** (length ≤ 1, numeric) — multi-digit or
    non-numeric ratings rejected before any processing.
13. **Unauthenticated-capable**: if no `glusrid` is supplied but a `mobile` number is,
    `forNewUserCreation` is populated and a **first HTTP-loopback (`CurlReqWithRetry
    POST`) creates a new user on-the-fly** — country-code `91` maps to `FK_GL_COUNTRY_ISO
    = "IN"`.
    [`UserFeedbackModel.go:30-48,201`](../../users-api-go-production/internal/models/users/UserFeedbackModel.go)
14. **A second HTTP-loopback (`CurlReqWithRetry POST`) submits the actual feedback**
    to another internal API — same HTTP-loopback architectural pattern seen in Rating
    Usefulness, Trust Verification, and Update With OTP KTs; this sub-flow does not
    write to any table directly in its own code.
    [`UserFeedbackModel.go:428`](../../users-api-go-production/internal/models/users/UserFeedbackModel.go)

---

## 5. RabbitMQ / Kafka / Redis

| Queue / `SERVICENAME` | Publisher | Consumer | Purpose |
|---|---|---|---|
| `USER_FEEDBACK` | `UserFeedbackModel.go` (`FeedbackUpdateDB`) — My-Feedback only, on insert/update success | [`USER_FEEDBACK.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_FEEDBACK.go) (`dbActionUserFeedback`) — connects to both `meshPg` and `approvalPg` | Downstream sync/processing of feedback events (email-notification code also referenced in `USER_RELATED_SEND_MAIL.go` per earlier grep) |

**Koi Kafka ya Redis usage nahi mila** in any of the three sub-flows.

---

## 6. End-to-End Technical Flow

### My Feedback / IMSearch Feedback
```
User (via GLADMIN/Weberp/MAPI/LEAP/M.INDIAMART.COM/DIR/FCP/BUYERS_FEEDBACK/SELLERMY/
      MY/EXPORT/EXPORTM)
    │
    ▼
[API — write]  POST serviceName=FEEDBACK_SERVICE  {FEEDBACK_FLAG, ...}
    │  UserFeedbackController.go — Gateway check
    ▼
FEEDBACK_FLAG dispatch
    │
    ├─ MY_FEEDBACKS ────────────────────────────┐
    │   mobile-normalize, email-validate         │
    │   ValidateUserFeedback()                   │
    │   FeedbackCheckIssueTypeID() (pre-check)    │
    │   FeedbackUpdateDB()                        │
    │       ├─ new → INSERT INTO MY_FEEDBACKS     │
    │       └─ existing → dynamic UPDATE           │
    │   [DB — meshpg]                              │
    │   on success → [RabbitMQ] USER_FEEDBACK      │
    │       ▼                                       │
    │   [CONSUME] dbActionUserFeedback (meshPg +    │
    │              approvalPg connections)          │
    │                                                │
    └─ IMSEARCH_FEEDBACK ───────────────────────────┘
        URL/referrer truncate (2048)
        ValidateUserFeedback()
        ImSearchFeedbackUpdateDB()
            ├─ no feedback_id → INSERT INTO IM_SEARCH_FEEDBACK
            └─ feedback_id given → dynamic UPDATE
        [DB — meshpg]  (no RabbitMQ push)
```

### App-Rating Feedback
```
User (app, possibly unauthenticated)
    │
    ▼
[API — read-repo]  serviceName=USERFEEDBACK  {glusrid?, mobile?, app_rating, modid,
                     app_version_no, ...}
    │  UserFeedbackController.go (read) — app_rating/app_version format checks
    ▼
UsersFeedback()
    │  app-version-gate check (Android >12.4.1 / iOS >12.0.0)
    ├─ glusrid absent + mobile present →
    │     [HTTP POST loopback]  create-new-user API (forNewUserCreation payload)
    ▼
    [HTTP POST loopback]  submit-feedback API (actual feedback data)
    ▼
Response relayed back to caller
```

---

## 7. Optimization Scope — DB Response-Time Contribution

### Medium impact

1. **`FeedbackCheckIssueTypeID` is a sequential pre-check SELECT before the main
   insert/update** — same "extra round-trip before the real write" pattern flagged
   repeatedly across this KT series (GST, Rating, ID Proof); could be folded into the
   main INSERT via a subquery/`EXISTS` check, or cached if the issue-type-list is small
   and slow-changing (Redis-candidate).
2. **App-Rating Feedback does up to 2 sequential HTTP-loopback calls** (create-user, then
   submit-feedback) — inherent to the on-the-fly-account-creation design; if `glusrid`
   is usually present, this cost is avoided, but for the unauthenticated path it's a
   meaningful added-latency chain (2 network round-trips, each with retry-logic).

### Low impact

3. **IMSearch-Feedback has no RabbitMQ fan-out** — simpler/lower-overhead than
   My-Feedback, appropriate given it's higher-volume/lower-criticality data.

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora Feedback flowchart yahan dekho](https://lucid.app/lucidchart/0908bb38-1883-4f8e-86d6-8ef1e7461877/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **Invalid `ISSUE_TYPE_ID` is silently nulled, not rejected** — a caller sending a
   typo'd issue-type-ID gets no error, the feedback just saves without a classification.
2. **My-Feedback and IMSearch-Feedback share one controller but diverge significantly
   in insert/update-detection logic** (`flagUpd`-based vs pure `feedback_id`-presence) —
   easy to conflate the two when reading the code casually.
3. **App-Rating Feedback's on-the-fly user-creation** means a single feedback-submission
   API call can have the side-effect of creating a brand-new supplier/buyer account —
   worth being aware of when analyzing account-creation-volume/sources.
4. **No table ownership for App-Rating Feedback in the read-repo's own code** — its
   actual persistence lives wherever the two loopback-target APIs write, not traceable
   further without reading those target-controllers.

---

## 10. Open Questions

1. What do the two HTTP-loopback target-APIs (`forNewUserCreation` and the feedback-
   submission API) actually map to — likely `UserAddModel`/`UserFeedbackController.go`
   in the write-repo, but not confirmed in this pass.
2. `modidVersionCheck=false` (old app-version) — what happens downstream when this is
   false? Not traced in this pass.
3. What does `USER_FEEDBACK.go`'s consumer actually do with the event (email? another
   DB sync?) — only partially traced (`approvalPg` connection present but its usage not
   read in full).
4. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`Feedback_Business_Doc.md`](./Feedback_Business_Doc.md) — product perspective
- [`../Social Review KT/Social_Review_Technical_Doc.md`](../Social%20Review%20KT/Social_Review_Technical_Doc.md) —
  unrelated concept (external social-media reviews)
- [`../Rating Usefulness KT/Rating_Usefulness_Technical_Doc.md`](../Rating%20Usefulness%20KT/Rating_Usefulness_Technical_Doc.md),
  [`../Update With OTP KT/Update_With_OTP_Technical_Doc.md`](../Update%20With%20OTP%20KT/Update_With_OTP_Technical_Doc.md) —
  same HTTP-loopback architectural pattern used by App-Rating Feedback

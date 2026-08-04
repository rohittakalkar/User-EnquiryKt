# Profile / Meta & Template (Company & PDP Content) — Technical Doc

Business/product perspective ke liye
[`Profile_Meta_Template_Business_Doc.md`](./Profile_Meta_Template_Business_Doc.md) dekho.

**Repos**: `users-api-go-production` (read), `service-api-go-production` (write),
`user-temp-consumers-production` (consumers — moderation + replica-sync).

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file path diya
gaya hai har jagah). Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — team se confirm karo]** likha hai, guess nahi kiya. Yeh doc pichle (shallow)
pass ko replace karta hai — is baar Template/Meta ka read-path aur moderation-consumer dono
fully traced hain, jo pehle sirf open-question the.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Profile write (API) | write | [`UserProfileController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserProfileController.go) |
| Profile write (DB logic) | write | [`UserProfileModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserProfileModel.go) (`UpsertProfile`, `InsertProfileImages`, `InsertProfileDocs`, `UpsertProfileImages`) |
| Profile read (API) | read | [`ProfileController.go`](../internal/controllers/UsersControllers/ProfileController.go) (`ActionProfileController`) |
| Profile read (DB logic) — **also serves Template read** | read | [`UserProfileModel.go`](../internal/models/users/UserProfileModel.go) (`ActionProfileDisplayModel`, `GetDisplayName`) |
| Profile read — **dead code, fully commented out** | read | [`actionProfile.go`](../internal/controllers/UsersControllers/actionProfile.go) — entire function body is a Go comment block (`// func ActionProfileController(c *gin.Context) {...}`), so it cannot even compile as active code. Already flagged dead in [`users_api_database_reference.md`](../API%20to%20Tables/users_api_database_reference.md); confirmed again here by direct read of the file. |
| Template write (API) | write | [`UserTemplateController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTemplateController.go) (`TemplateController`) |
| Template write (DB logic) | write | [`UserTemplateModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTemplateModel.go) (`UserTemplateMeshPg`, `setPcClntFsv`, `CheckFeatureTemplate`) |
| Meta write (API) | write | [`UserMetaController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserMetaController.go) |
| Meta write (DB logic) | write | [`UserMetaModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMetaModel.go) (`UpsertMeta`) |
| Content-moderation consumer — **shared by all three features** | consumers | [`USER_PROFILE_BANNED_DETECT.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PROFILE_BANNED_DETECT.go) — branches on `SERVICENAME` (`USER_PROFILE_SERVICE` / `TEMPLATE_SERVICE` / `META_SERVICE` / `USER_DETAIL_BANNED_SERVICE`) |
| Profile replica-sync consumer (writes to search DB) | consumers | [`USER_PROFILE_IMSDB.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PROFILE_IMSDB.go) |

**Read-side correction vs. previous doc pass**: the previous doc said "no dedicated
`GET /template` or `GET /meta` was found." Re-verifying this pass, that is **half true**:

- **Template**: `GET/POST /profile/*params` with query param `tmpl_content=1` IS the
  dedicated Template read path — it joins `pc_clnt_feature_section_value` with
  `pc_feature_section` and returns all section rows for the GLID.
  [`UserProfileModel.go:32-89`](../internal/models/users/UserProfileModel.go) (users-api repo).
  So Template *does* have a real read endpoint, it's just piggybacked on `/profile` via a
  flag rather than a separate path.
- **Meta**: still **no read path found** anywhere in `users-api-go-production` — no query
  anywhere in this repo selects from `PC_CLNT_PCAT`'s meta columns
  (`PC_CLNT_PCAT_META_TITLE`/`_DESCRIPTION`/`_KEYWORDS`). This remains an open question
  (section 15).

---

## 2. Routes (confirmed from router.go)

| Method | Path | Repo | Controller |
|---|---|---|---|
| GET/POST | `/profile/*params` | read | `ActionProfileController` — [`routerUsers.go:168-169,492-493,598-599`](../internal/api/users_router/routerUsers.go) |
| POST | `/profile` | write | `UserProfileController` — [`router.go:178,347`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) |
| POST | `/template` | write | `TemplateController` — [`router.go:177,346`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) |
| POST | `/meta` | write | `UserMetaController` — [`router.go:173,341`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) |

**Why routes appear 2-3 times**: `routerUsers.go`/`router.go` define multiple
`SetupXCluster(...)` functions (separate `gin.Engine` instances for different deployment
clusters/failover groups), and each registers the same route set independently — this is a
codebase-wide pattern, not specific to this feature, and not a routing bug.

---

## 3. Data Model — Tables

> **Verification note**: yeh sab table/column names Go code ke andar embedded SQL strings
> se liye gaye hain (har row ke saamne file:line diya hai). Live DB schema se cross-verify
> **nahi** kiya gaya hai.

### Profile

| Table | Physical DB | Purpose | Confirmed from |
|---|---|---|---|
| `GLUSR_USR_PROFILE` | meshpg (write) / `mesh_pg_user` (read) / searchPg (replica, via consumer) | **Core profile-section record** — one row per (GLID, `GLUSR_PROFILE_TYPE_ID`) for Media/Awards/ExtProfile/ExtProfile1/ExtProfile2/Infrastructure/Jobs/Media/News/Quality/Testimonial section types | [`UserProfileModel.go:121,198,234,268,495,530`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserProfileModel.go); read: [`UserProfileModel.go:132,440`](../internal/models/users/UserProfileModel.go) |
| `GLUSR_USR_CLNT_FEATURE_VALUE` | meshpg | Company display-name per "feature ref name" (e.g. `awards`, `testimonial`) — read via `GetDisplayName`, written via `flag=="CLNT"` branch or the dedupe-insert-if-absent branch | [`UserProfileModel.go:92,933,953`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserProfileModel.go), [`UserProfileModel.go:641`](../internal/models/users/UserProfileModel.go) (read) |
| `GLUSR_USR_PROFILE_IMAGE` | meshpg | Up to 5 images per profile-section, delete-then-bulk-insert pattern | [`UserProfileModel.go:1176,1271,1289`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserProfileModel.go) |
| `GLUSR_USR_PROFILE_DOC` | meshpg | Attached documents per profile-section, same delete-then-insert pattern as images | [`UserProfileModel.go:1379,1407`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserProfileModel.go) |

### Template

| Table | Physical DB | Purpose | Confirmed from |
|---|---|---|---|
| `PC_CLNT_FEATURE_SECTION_VALUE` | meshpg | **Individual template-section content** — Awards/Testimonials/Infrastructure/News/Custom/Franchise/tab-titles, one row per (GLID, `FK_PC_FEATURE_SECTION_ID`) | [`UserTemplateModel.go:1604,1651,1794,1914`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTemplateModel.go); read: [`UserProfileModel.go:33`](../internal/models/users/UserProfileModel.go) |
| `PC_FEATURE_SECTION` | meshpg | Master/lookup table of section-types (`PC_FEATURE_SECTION_ID`, `PC_FEATURE_SECTION_REF_NAME`) — used to validate submitted `fsid`+`refname` pairs | [`UserTemplateModel.go:19,1651`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTemplateModel.go) |
| `PC_CLNT_PCAT` | meshpg | **Category-level content** — Template writes name/file-name here for `category_section` type (`Type == "TemplateContent"`); shared with Meta (below) | [`UserTemplateModel.go:85`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTemplateModel.go) |
| `PC_CLIENT` | meshpg | Top-level client/profile record — Google Analytics code, page/rollup trackers, "updated by" audit fields (not section content itself) | [`UserTemplateModel.go:1500-1537`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTemplateModel.go) |

### Meta

| Table | Physical DB | Purpose | Confirmed from |
|---|---|---|---|
| `PC_CLNT_PCAT` | meshpg | **SEO/category metadata** — `PC_CLNT_PCAT_NAME`, `PC_CLNT_FLNAME`, `PC_CLNT_PCAT_META_TITLE`, `PC_CLNT_PCAT_META_DESCRIPTION`, `PC_CLNT_PCAT_META_KEYWORDS` | [`UserMetaModel.go:47-58`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMetaModel.go) |

**Confirmed shared-table finding**: `PC_CLNT_PCAT` is written by **both** Template
(`category_section`/`TemplateContent` path, generic name/file-name fields) and Meta
(SEO-specific meta fields) — same row, different columns. Both write paths filter with
`WHERE PC_CLNT_PCAT_GLUSR_ID = $x AND PC_CLNT_TOPLEVEL = 'F' AND PC_CLNT_PCAT_IS_ECOM = 0` (Meta)
or `... AND (PC_CLNT_PCAT_IS_ECOM = 0 OR PC_CLNT_PCAT_IS_ECOM IS NULL)` (Template) —
[`UserMetaModel.go:58`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMetaModel.go),
[`UserTemplateModel.go:85-86`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTemplateModel.go).
Note the two `IS_ECOM` predicates aren't identical (Meta requires exactly `0`, Template
also accepts `NULL`) — a small but real inconsistency between the two write paths on a
shared table.

---

## 4. "Decode This Magic Value" — Section/Profile Type IDs

### `GLUSR_PROFILE_TYPE_ID` (Profile sections)

Reconstructed from the `getProfileData()`/switch-statement mapping used by both the write
controller and the moderation consumer — **not from a documented enum**:

| ID | `PROFILE_TYPE` string | Evidence |
|---|---|---|
| 1 | `Awards` | [`USER_PROFILE_BANNED_DETECT.go:699`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PROFILE_BANNED_DETECT.go) |
| 2 | `Custom_Profile` | same, line 701 |
| 3 | `ExtProfile` | same, line 703 |
| 4 | `ExtProfile1` | same, line 705 |
| 5 | `ExtProfile2` | same, line 707 |
| 6 | `Infrastructure` | same, line 709 |
| 7 | `Jobs` | same, line 711 |
| 8 | `Media` | same, line 713 |
| 9 | `News` | same, line 715 |
| 10 | `Quality` | same, line 717 |
| 11 | `Testimonial` | same, line 719 |

Confirmed consistent with the read-side switch in
[`UserProfileModel.go:94-130`](../internal/models/users/UserProfileModel.go) (users-api repo).

### `FK_PC_FEATURE_SECTION_ID` (Template sections)

Reconstructed from `getTemplateData()`'s switch statement in the moderation consumer —
**[INFERRED — confirm exact business meaning with Template/product team]**, since these are
bare numeric IDs with no documented enum anywhere in code:

| ID | Section (inferred) | Evidence |
|---|---|---|
| `253` | Category/PDP section (`category_section`) | [`USER_PROFILE_BANNED_DETECT.go:618`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PROFILE_BANNED_DETECT.go); also hardcoded as `cat_fsid: "253"` at line 198 |
| `5` | "Profile" section (general description) | line 624 |
| `218` | "Profile1" section | line 633 |
| `226` | "Profile2" section | line 642 |
| `164` | News section | line 651 |
| `194` | Jobs/opportunities section | line 660 |
| `210` | Testimonial section | line 669 |
| `259` | Query-page top text | line 678 |
| `266` | Company tagline | line 685 |
| `260` | Template tab-5 title (hardcoded literal in write path, not from the switch above) | [`UserTemplateModel.go:685,691`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTemplateModel.go) |
| `261` | Template tab-4 title (same pattern) | [`UserTemplateModel.go:801,807`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTemplateModel.go) |

**[INFERRED — confirm with team]**: what "Tab-4"/"Tab-5" concretely correspond to on the
storefront UI is not derivable from code alone — the hardcoded history-comment strings
("Template Tab-5 Title Changed") are the only hint.

---

## 5. Business Rules & Validation (code se exhaustive list)

### Profile

1. **`PROFILE_IMGS`/`PROFILE_DOCS` explicitly exempted from input-formatting** — deleted
   from `inputParams` before `utils.FormatInputParams()` runs, then re-added — protects
   image/doc array structure from being mangled by generic string formatting.
   [`UserProfileController.go:43-54`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserProfileController.go)
2. **`P_NO` (page number) not in `{0,4,5,8}` triggers `AddtParamsMap` merge** — these four
   page-numbers take a simpler param set; all other page-numbers get extra merged params.
   [`UserProfileController.go:78-85`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserProfileController.go)
3. **Insert vs. update branch depends on whether `PROFILE_ID` is empty** — empty →
   `SELECT MAX(sortorder)+1` then `INSERT ... RETURNING`; present → fetch current row,
   diff every field against submitted value, build a dynamic `UPDATE` with only changed
   columns. [`UserProfileModel.go:232-268,469-884`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserProfileModel.go)
4. **Image inheritance chain when a new image isn't supplied**: if `MEDIA`/`IMAGE1` is
   empty on update, the code tries (a) the next image in the just-submitted `PROFILE_IMGS`
   batch, then (b) a live `UpsertProfileImages(..., "SELECT")` lookup against
   `GLUSR_USR_PROFILE_IMAGE`, before falling back to keeping the old value — an extra,
   conditional DB round-trip that only fires in this specific case (`CASE "3.1"`).
   [`UserProfileModel.go:555-600,661-704`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserProfileModel.go)
5. **Max 5 images per profile section** — `InsertProfileImages` hard-rejects (`500`,
   "Number of images to be inserted exceeded") if more than 5 are submitted.
   [`UserProfileModel.go:1166-1173`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserProfileModel.go)
6. **Images/docs are always delete-then-bulk-insert, never diffed** — every image/doc
   update deletes all existing rows for that `(GLUSR_ID, PROFILE_ID)` and re-inserts the
   full submitted batch inside one DB transaction (`BeginTx`/commit/rollback).
   [`UserProfileModel.go:1262-1330`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserProfileModel.go)
7. **Approval status auto-forced to Approved for owner-submitted content**: if
   `LAST_MODIFIED_BY == "O"` (owner), `STATUS` is force-set to `"A"` regardless of what was
   submitted. [`UserProfileModel.go:831-833`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserProfileModel.go)
8. **`GLUSR_USR_CLNT_FEATURE_VALUE` dedupe-insert only for specific gates**: the
   display-name row is inserted only if `gate` is `BUYERMY`/`MY`/`SELLERMY` **and** the
   `PROFILE_TYPE` is not `Media`/`ExtProfile1`/`ExtProfile2`, and only if a `COUNT(*)`
   check shows no existing row — a read-before-write dedupe pattern, not a DB-level unique
   constraint. [`UserProfileModel.go:930-1026`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserProfileModel.go)
9. **Read side hides rejected content from non-owners**: `GET /profile?type=...` filters
   out rows with `glusr_profile_approval_status == "R"` unless `modid == "MY"` (i.e. the
   supplier viewing their own profile can see their rejected content, buyers cannot).
   [`UserProfileModel.go:162,282`](../internal/models/users/UserProfileModel.go) (users-api repo)
10. **`HTVENDOR`/`HELLOTD` callers get an unfiltered-by-type read** — same table, but the
    query drops the `AND glusr_profile_type_id=$2` clause entirely, returning all
    profile-section rows for the GLID in one call.
    [`UserProfileModel.go:440`](../internal/models/users/UserProfileModel.go) (users-api repo)

### Template

11. **Large gateway allowlist (23 caller surfaces)** — `MY`, `BUYERMY`,
    `M.INDIAMART.COM`, `GLADMIN`, `TOLLFREE`, `Email Marketing`, `HTVENDOR`, `WEBERP`,
    `LEAP`, `IPC-Admin`, `NSD Sales`, `MAPI`, `ENQUIRY`, `FREE-WEBSITE`, `BL`, `TRADE`,
    `TENDER`, `PAYNOW`, `CREDIT ALLOCATION`, `OVP Process`, `SAMPARK Process`,
    `TOLLFREE Process`, `VENDOR CITY Pin Correction`, `Notification Server` — much broader
    than Meta's 2-entry allowlist (rule 15), suggesting Template content is written from
    many more internal/automated processes than Meta.
    [`UserTemplateController.go:157`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTemplateController.go)
12. **FSID/Refname cross-check (`CheckFeatureTemplate`)**: if both a `*fsid` and `*refname`
    key are present in the request, the code verifies `PC_FEATURE_SECTION_ID = fsid AND
    PC_FEATURE_SECTION_REF_NAME = refname` actually exist together in `PC_FEATURE_SECTION`
    — mismatch produces "Provided FSID and Refname are diffrent" (sic).
    [`UserTemplateController.go:196-231`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTemplateController.go),
    [`UserTemplateModel.go:15-55`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTemplateModel.go)
13. **`PC_CLNT_FEATURE_SECTION_VALUE` supports insert-vs-update**: new section → INSERT;
    existing → UPDATE, keyed on `(FK_GLUSR_USR_ID, FK_PC_FEATURE_SECTION_ID)`.
    [`UserTemplateModel.go:1794,1914`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTemplateModel.go)
14. **Tab-4/Tab-5 title updates are unconditional per-request side effects**: every
    `TemplateController` call checks `tt4_old_value != tt4_radio_hidden` and
    `tt5_old_value != tt5_radio_hidden` and, if changed, always fires an extra
    INSERT/UPDATE against hardcoded `FK_PC_FEATURE_SECTION_ID` `260`/`261` — independent of
    whatever section-type the rest of the request was actually about.
    [`UserTemplateModel.go:670-834`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTemplateModel.go)

### Meta

15. **Gateway allowlist is small — only `SELLERMY`/`BUYERMY`**.
    [`UserMetaController.go:49`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserMetaController.go)
16. **Only one content-branch exists: `category_section`** — updates
    `PC_CLNT_PCAT` name/meta-title/meta-description/meta-keywords, filtered to
    `PC_CLNT_TOPLEVEL='F' AND PC_CLNT_PCAT_IS_ECOM = 0`. No product-level (individual PDP)
    branch found anywhere in `UserMetaModel.go`.
    [`UserMetaModel.go:44-58`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMetaModel.go)
17. **Success depends on an exact string match**: `code=200` only if `output ==
    "UPDATE SUCCESS IN MESH PG"` — any drift in that literal string (e.g. a future refactor
    typo) silently becomes a `500` failure from the caller's perspective.
    [`UserMetaController.go:71`](../../service-api-go-production/service-api-go-controller/UserControllers/UserMetaController.go)
    (also verified directly in [`UserMetaModel.go:14,75`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMetaModel.go))
18. **Post-update SELECT result (`PC_CLNT_PCAT_ID` list) is computed but not consumed
    further** — `UpsertMeta` runs a second query fetching matching `PC_CLNT_PCAT_ID`s into
    `params["id_array"]`, but neither `UpsertMeta`'s return value nor the controller uses
    `id_array` afterward — this looks like a **dead round-trip** (see Optimization Scope,
    point 3). [`UserMetaModel.go:96-126`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMetaModel.go)

---

## 6. RabbitMQ

All three write paths publish to RabbitMQ via the shared `PushToQueue`/`serviceToQueueMap`
mechanism, and **all three route to the same downstream queue**:

| SERVICENAME | Routes to (queue) | Publisher | Evidence |
|---|---|---|---|
| `USER_PROFILE_SERVICE` | `USER_PROFILE_BANNED_DETECT` | `UserProfileModel.go` (`UpsertProfile`) | [`rabbitmq.go:45`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go) |
| `TEMPLATE_SERVICE` | `USER_PROFILE_BANNED_DETECT` | `UserTemplateModel.go` (`UserTemplateMeshPg`, `setPcClntFsv`) | [`rabbitmq.go:44`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go) |
| `META_SERVICE` | `USER_PROFILE_BANNED_DETECT` | `UserMetaModel.go` (`UpsertMeta`) | [`rabbitmq.go:41`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go) |

**Major finding this pass — the consumer is not a dead end.** The previous doc flagged
`META_SERVICE`'s routing to `USER_PROFILE_BANNED_DETECT` as unconfirmed/possibly-dead. It is
**not dead** — `USER_PROFILE_BANNED_DETECT.go` explicitly branches on all three
`SERVICENAME` values (plus a fourth, `USER_DETAIL_BANNED_SERVICE`, out of this doc's scope)
and does real work for each:

1. Extracts the free-text description field relevant to that service
   (`GLUSR_PROFILE_DESCRIPTION` / `PC_CLNT_FEATURE_SV_DESCRIPTION` /
   `PC_CLNT_PCAT_META_DESCRIPTION`).
2. Calls an external banned-content API (`utils.CallBannedApi`) with that text.
3. **If the API flags the content (`flag == "2"`)**: the consumer builds a synthetic
   "resubmit with description blanked out" request and calls the **write API itself** over
   HTTP (`utils.CallWapiService` against `wapi_profile_api` / `wapi_template_api` /
   `wapi_meta_api`), with `FROM_WORKER: "1"` so the resubmission skips re-publishing to
   RabbitMQ (avoiding an infinite loop).
4. **If the API doesn't flag it**: the consumer re-publishes the original message onward to
   `comp.sync.<glid%20>` (or `user.profile.<glid%20>` for Profile) for the standard
   downstream company-sync fan-out.

   [`USER_PROFILE_BANNED_DETECT.go:68-270`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PROFILE_BANNED_DETECT.go)

**Business meaning**: this is a **silent, automatic content-moderation loop** — a supplier
who enters PII/abusive text in a profile/template/meta description will have that text
auto-blanked by a background worker without an explicit rejection notice in the original
request/response cycle. See Business Doc section 5 and Edge Cases (section 12) below.

**Note on Template's `PC_CLNT_FEATURE_SECTION_VALUE` also appearing under
`USER_DETAIL_BANNED_SERVICE`**: the same table is checked again under a different
`SERVICENAME` branch for a `"Franchise"` sub-flow
([`USER_PROFILE_BANNED_DETECT.go:559-602`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PROFILE_BANNED_DETECT.go))
— out of this doc's direct scope (belongs to the Details/Franchise feature) but worth
knowing the table is touched from more than one consumer branch.

**Downstream exchange/queue mechanics**: all re-publish paths use exchange `USER.topic`
with routing keys `comp.sync.<glid % 20>` or `user.profile.<glid % 20>` (20-way shard,
standard cross-domain pattern, not specific to this feature).

---

## 7. Kafka

**No Kafka usage found** anywhere in the Profile/Template/Meta write, read, or consumer
code paths (grepped across all three repos' relevant files). Unlike GST's
`USER_GST_DETAILS_BULK`, this domain has no Kafka exception.

---

## 8. Redis

**No Redis usage found** anywhere in this feature group. Every read
(`GET /profile`, including the `tmpl_content=1` Template path) hits Postgres directly —
confirmed by grep across both the write-side model files and the read-side
`UserProfileModel.go` (users-api repo). See Optimization Scope (section 11) for the
caching-gap implication.

---

## 9. End-to-End Technical Flows

### Flow A — Profile section write (Awards/Testimonial/etc.)

```
Supplier (seller panel/app)
    │
    ▼
[API — write]  POST /profile  {P_NO, GLUSR_ID, PROFILE_TYPE, TITLE/DESCRIPTION, PROFILE_IMGS, PROFILE_DOCS, ...}
    │  UserProfileController.go
    │  1. PROFILE_IMGS/DOCS exempted from generic formatting
    │  2. Mandatory-params check (MandatoryParamsCheckProfile)
    │  3. P_NO not in {0,4,5,8} → merge AddtParamsMap
    ▼
[DB]  UpsertProfile() — insert (new PROFILE_ID) or diff-and-update (existing) on GLUSR_USR_PROFILE
    │  conditionally: image-inheritance chain (SELECT GLUSR_USR_PROFILE_IMAGE if no new image supplied)
    │  conditionally: dedupe-insert into GLUSR_USR_CLNT_FEATURE_VALUE (gate-gated)
    ▼
[DB, if PROFILE_IMGS present]  delete-then-bulk-insert GLUSR_USR_PROFILE_IMAGE (in a transaction)
    ▼
[DB, if PROFILE_DOCS present]  delete-then-bulk-insert GLUSR_USR_PROFILE_DOC
    ▼
[RabbitMQ]  SERVICENAME=USER_PROFILE_SERVICE → routes to USER_PROFILE_BANNED_DETECT
    ▼
[CONSUME]  USER_PROFILE_BANNED_DETECT.go
    │  extracts GLUSR_PROFILE_DESCRIPTION, calls banned-content API
    ├─ flagged → auto-blank description, resubmit via wapi_profile_api (FROM_WORKER=1, no re-publish)
    └─ clean   → re-publish to comp.sync.<glid%20> / user.profile.<glid%20> for downstream fan-out
```

### Flow B — Template section write

```
Supplier / internal caller (23-value gateway allowlist)
    │
    ▼
[API — write]  POST /template  {type: "TemplateContent", glusr_id, cat_fsid/fsid+refname, ...}
    │  TemplateController.go
    │  1. Mandatory-fields check (MandatoryFieldsTemplate)
    │  2. Gateway_v1 check against 23-entry allowlist
    │  3. LengthAndTypeValidations_v2 (UserTemplateMap)
    │  4. CheckFeatureTemplate() — if fsid+refname both given, verify they match in PC_FEATURE_SECTION
    ▼
[DB]  UserTemplateMeshPg() → dispatches per Type/section:
    │  - category_section (Type=="TemplateContent")  → SELECT PC_CLNT_PCAT_ID, then setPcClntFsv()
    │  - other feature sections                       → INSERT/UPDATE PC_CLNT_FEATURE_SECTION_VALUE
    │  - always (if changed)                           → tab4/tab5 title INSERT/UPDATE (hardcoded FSID 260/261)
    │  - top-level client fields (GA code, trackers)   → UPDATE PC_CLIENT
    ▼
[RabbitMQ]  SERVICENAME=TEMPLATE_SERVICE → routes to USER_PROFILE_BANNED_DETECT
    ▼
[CONSUME]  same moderation flow as Flow A, keyed on PC_CLNT_FEATURE_SV_DESCRIPTION /
           PC_CLNT_DESCRIPTION → auto-blank + resubmit via wapi_template_api, or fan-out to comp.sync.*
```

### Flow C — Meta (SEO/category) write

```
Supplier or SELLERMY/BUYERMY-gated caller
    │
    ▼
[API — write]  POST /meta  {category_section, category_cat_name, category_meta_title/desc/kwd, ...}
    │  UserMetaController.go
    │  1. MandatoryParamsCheckMeta
    │  2. Gateway_v1(["SELLERMY","BUYERMY"], ...)
    │  3. LengthAndTypeValidations_v3 (UserMetaMap)
    ▼
[DB]  UpsertMeta()
    │  1. UPDATE PC_CLNT_PCAT SET name/meta_title/meta_description/meta_keywords
    │     WHERE glusr_id=$x AND TOPLEVEL='F' AND IS_ECOM=0
    │  2. SELECT PC_CLNT_PCAT_ID FROM PC_CLNT_PCAT WHERE same filter (result appears unused downstream — see §5.18)
    ▼
[RabbitMQ]  SERVICENAME=META_SERVICE → routes to USER_PROFILE_BANNED_DETECT
    ▼
[CONSUME]  same moderation flow, keyed on PC_CLNT_PCAT_META_DESCRIPTION →
           auto-blank + resubmit via wapi_meta_api, or fan-out to comp.sync.*
```

### Flow D — Reads

```
Buyer / App / Internal caller
    │
    ▼
[API — read]  GET or POST /profile/*params  {glid, modid, type?, tmpl_content?}
    │  ActionProfileController.go → ActionProfileDisplayModel()
    ├─ tmpl_content=1        → JOIN pc_clnt_feature_section_value + pc_feature_section (Template read)
    ├─ type=<Awards|...>     → SELECT glusr_usr_profile WHERE type_id=X (+ GetDisplayName from GLUSR_USR_CLNT_FEATURE_VALUE)
    ├─ modid=HTVENDOR/HELLOTD → SELECT glusr_usr_profile, all types, unfiltered
    └─ none of the above     → "TYPE_NOT_FOUND"
```

**No Meta read path found** in this flow or anywhere else in `users-api-go-production` —
see Open Questions.

### Flow E — Profile replica-sync to search DB

```
[CONSUME]  comp.sync.<glid%20> (generic cross-domain fan-out, not Profile/Template/Meta-specific)
    │  USER_PROFILE_IMSDB.go — subscribes with its own dedicated searchPg connection
    ▼
[DB]  INSERT or UPDATE GLUSR_USR_PROFILE on searchPg (separate physical Postgres instance
      from meshpg) — keeps a search/read-replica copy of profile-section rows in sync
```

---

## 10. Flow-wise DB & Table Usage

### Flow A — Profile Write

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `GLUSR_USR_PROFILE` | SELECT (`MAX(sortorder)`, insert path) or SELECT (current row, update path) | Determine next sort order (insert) or diff against submitted values (update) |
| 2 | meshpg | `GLUSR_USR_PROFILE` | INSERT `RETURNING ...ID` or UPDATE | Persist the profile-section row |
| 3 | meshpg | `GLUSR_USR_PROFILE_IMAGE` | SELECT (conditional, image-inheritance fallback, `CASE 3.1`) | Look up an existing image to reuse if the caller didn't supply a new one |
| 4 | meshpg | `GLUSR_USR_CLNT_FEATURE_VALUE` | SELECT (`COUNT(*)` dedupe check, gate-gated) | Decide whether a display-name row needs inserting |
| 5 | meshpg | `GLUSR_USR_CLNT_FEATURE_VALUE` | INSERT `RETURNING ...ID` (only if count==0) | Create the display-name row |
| 6 | meshpg (transaction) | `GLUSR_USR_PROFILE_IMAGE` | DELETE then bulk INSERT `RETURNING ...ID`s | Replace all images for this profile-section in one atomic step |
| 7 | meshpg (transaction) | `GLUSR_USR_PROFILE_DOC` | DELETE then bulk INSERT | Replace all docs for this profile-section |

**Total round-trips for a single Profile write with images+docs: up to 7**, several of
which are conditional. This is comparable in shape to GST's multi-round-trip writes but
confined to a single physical DB (meshpg) rather than GST's 4-database fan-out.

### Flow B — Template Write

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `PC_FEATURE_SECTION` | SELECT (`CheckFeatureTemplate`, only if fsid+refname both submitted) | Validate the submitted fsid/refname pair actually correspond |
| 2 | meshpg | `PC_CLNT_PCAT` | SELECT `PC_CLNT_PCAT_ID` (only for `category_section`) | Resolve which category rows this GLID owns before updating |
| 3 | meshpg | `PC_CLNT_FEATURE_SECTION_VALUE` | INSERT or UPDATE | Persist the actual section content (Awards/News/Testimonial/etc., or category-section proxy row) |
| 4 | meshpg | `PC_CLNT_FEATURE_SECTION_VALUE` | INSERT or UPDATE (tab-5, hardcoded `FSID=260`, only if `tt5` value changed) | Persist a "Tab-5 title" side-write, independent of the main section being edited |
| 5 | meshpg | `PC_CLNT_FEATURE_SECTION_VALUE` | INSERT or UPDATE (tab-4, hardcoded `FSID=261`, only if `tt4` value changed) | Same pattern for "Tab-4 title" |
| 6 | meshpg | `PC_CLIENT` | UPDATE | Persist top-level client fields (analytics code, page/rollup trackers) if submitted |

**Up to 9 query-slots are tracked in the controller** (`QUERY1`...`QUERY9`), consistent
with this being the most complex controller in the group — confirmed, not just inferred,
from the actual branch structure above.

### Flow C — Meta Write

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `PC_CLNT_PCAT` | UPDATE (name, meta title/description/keywords) | The actual SEO-content write |
| 2 | meshpg | `PC_CLNT_PCAT` | SELECT `PC_CLNT_PCAT_ID` (same filter as #1) | Result stored in `params["id_array"]` but **not consumed further** — see §5.18 and Optimization Scope §11.3 |

### Flow D — Reads

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | `mesh_pg_user` (read replica) | `pc_clnt_feature_section_value` JOIN `pc_feature_section` | SELECT | `tmpl_content=1` — Template section read |
| 2 | `mesh_pg_user` | `glusr_usr_profile` | SELECT (typed or unfiltered) | `type=...` or `modid=HTVENDOR/HELLOTD` — Profile section read |
| 3 | `mesh_pg_user` | `GLUSR_USR_CLNT_FEATURE_VALUE` | SELECT (`GetDisplayName`) | Fetch display name for the section's `refName`, joined in-app not in-SQL |

### Flow E — Replica-Sync Consumer

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | searchPg (separate physical instance from meshpg) | `GLUSR_USR_PROFILE` | INSERT or UPDATE | Keeps a search-serving copy of profile-section rows in sync with the meshpg source of truth, driven off the generic `comp.sync.*` fan-out |

---

## 11. Optimization Scope — DB / Response-Time Contribution

### High-impact

1. **Template's unconditional tab-4/tab-5 side-writes run on every request that touches
   those fields, regardless of what section the caller actually meant to edit** — two
   extra INSERT/UPDATE round-trips (`Query7Exec`, `Query8Exec`) fire whenever
   `tt4_radio_hidden`/`tt5_radio_hidden` differ from their "old value" companions, which,
   given `paramstoformat` defaults empty strings for missing keys, can trigger more often
   than intended. **Suggestion**: confirm with the frontend/team whether these two fields
   are sent on every Template call (even ones unrelated to tab titles) — if so, this is a
   real 2-extra-round-trip tax on the whole endpoint.
   [`UserTemplateModel.go:656-834`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTemplateModel.go)
2. **Profile's image-inheritance fallback (`CASE 3.1`) adds a conditional extra SELECT**
   (`UpsertProfileImages(..., "SELECT")`) inside what is otherwise a single-query update
   path — this only fires when the caller updates a profile section without resending an
   image, but when it does fire it's a genuine sequential extra round-trip inside the
   critical write path.
   [`UserProfileModel.go:565-585,671-690`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserProfileModel.go)
3. **Meta's second query (`SELECT PC_CLNT_PCAT_ID`) appears to be dead weight** — its
   result (`params["id_array"]`) is set but never read again in `UpsertMeta` or by the
   calling controller. **Concrete fix**: if genuinely unused, remove this query entirely —
   it's a free 1-round-trip-per-request saving with essentially zero risk, the same
   "easiest fix in the doc" category as GST's `RETURNING` suggestion.
   [`UserMetaModel.go:96-126`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMetaModel.go)

### Medium-impact

4. **No caching anywhere on the read side** (section 8) — `GET /profile` (both the
   type-filtered and `tmpl_content=1` paths) hits Postgres live on every call. Profile/
   Template content changes infrequently relative to read volume (storefront pages are
   viewed far more often than edited) — a natural cache-aside candidate (same shape as
   GST's finding), especially for the `tmpl_content=1` join query which touches two tables.
5. **`PC_CLNT_PCAT` shared-write race between Template and Meta** — if both APIs are
   called concurrently for the same `(glusr_id, category_section)`, the two `UPDATE`s
   touch different columns of the same row but neither uses row versioning/locking beyond
   Postgres's default MVCC — a last-write-wins risk on interleaved concurrent edits is
   plausible, though not confirmed to have caused an actual incident.
   [`UserTemplateModel.go:85-130`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTemplateModel.go),
   [`UserMetaModel.go:47-58`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMetaModel.go)
6. **Images/docs are always full delete+re-insert, never diffed** (section 5.6) — for a
   supplier who edits one field on a profile section with 5 images attached, all 5 image
   rows are deleted and re-inserted even though none of the image data changed. This is
   correctness-safe but wastes writes; a diff-based upsert would reduce write volume on
   image-heavy profile sections.

### Low-impact / good-practice already present

7. **Delete+bulk-insert for images/docs runs inside an explicit SQL transaction**
   (`BeginTx`/commit/rollback) in `InsertProfileImages`/`InsertProfileDocs` — correctness
   is solid even though it isn't the most write-efficient pattern (point 6 above).
8. **Meta's design is otherwise the simplest of the three** — single content-branch,
   minimal validation, no obvious further optimization needed beyond removing the dead
   query (point 3).

---

## 12. Cron Inventory

**No cron job touches Profile, Template, or Meta** in any of the three repos. Checked
`service-api-go-production/crons/` (only GST/rating/mcat-related crons exist there) and
grepped `user-temp-consumers-production` for any file/function name matching
`*rofile*cron*`, `*emplate*cron*`, `*eta*cron*` — none found. This is a genuine "confirmed
absence," not an unchecked assumption.

---

## 13. Edge Cases & Gotchas (technical POV)

1. **`actionProfile.go` is not just "dead" — it is fully commented out Go source**,
   meaning it contributes zero behavior and cannot even accidentally be wired into a
   route; the only reason it's worth flagging at all is that a future refactor might
   uncomment it thinking it's the "real" implementation, when `ProfileController.go` is
   the live one.
2. **Template read is not where you'd expect it** — there is no `GET /template`; the read
   path is `GET /profile?tmpl_content=1`. Anyone debugging "why doesn't my Template edit
   show up" needs to know to look at `/profile` with that flag, not a `/template` GET that
   doesn't exist.
3. **Meta has no confirmed read path at all** — if a "my SEO title isn't showing" ticket
   comes in, there is currently no traced code path in `users-api-go-production` that
   serves it back; either it's read by a system outside these three repos, or it's a real
   gap (see Open Questions).
4. **Content-moderation is a silent auto-edit, not a rejection** — if the banned-content
   API flags a description, the system doesn't reject the original write; it lets it
   succeed, then quietly resubmits a version with the description blanked out
   asynchronously via the consumer. A supplier could see their own submission "succeed"
   and then later find their description empty, with no explicit error message tying the
   two together.
5. **`PC_CLNT_PCAT`'s `IS_ECOM` filter is not written identically by Template vs. Meta** —
   Meta requires exactly `0`; Template accepts `0` OR `NULL`. If any row has `IS_ECOM IS
   NULL`, Meta's write to that row would silently not match while Template's would. Worth
   confirming intentional vs. accidental.
6. **Hardcoded validation keys/section IDs baked into consumer code** —
   `VALIDATION_KEY: "e27d039e38ae7b3d439e8d1fe870fc68"` and section IDs like `253`/`260`/
   `261` are hardcoded literals inside `USER_PROFILE_BANNED_DETECT.go`'s auto-resubmit
   logic — if the write API's validation key or section-ID scheme ever changes, this
   consumer's silent-redaction path breaks without any obvious error surfacing to the
   original caller.
7. **Routes registered 2-3 times across cluster-specific router setups** — not a bug
   (see section 2), but can look confusing when grepping `router.go`/`routerUsers.go` and
   seeing the same line appear multiple times.

---

## 14. Open Questions — Team Ke Liye

1. **Meta read path**: confirm whether `PC_CLNT_PCAT`'s meta columns
   (`PC_CLNT_PCAT_META_TITLE`/`_DESCRIPTION`/`_KEYWORDS`) are actually read by some other
   service (search-indexing pipeline? a PHP/legacy front-end outside these Go repos?), or
   whether this is a genuine gap in the current microservices.
2. **Tab-4/Tab-5 semantics** (section 4) — what do these two hardcoded template-section IDs
   (`260`/`261`) concretely correspond to on the storefront UI? Only inferable from history
   comment strings.
3. **`tt4`/`tt5` fields on every Template call** (Optimization §11.1) — are these two fields
   sent by the frontend on every `/template` request regardless of which section is being
   edited, or only when the tab-title UI is actually touched? Determines whether this is a
   universal 2-round-trip tax or a rare one.
4. **Meta's unused `id_array` query** (§5.18, §11.3) — confirm it's truly unused before
   removing; there may be a downstream consumer of `params["id_array"]` this pass didn't
   trace (e.g. via a shared `params` map reference the RabbitMQ payload construction reads
   from — not confirmed either way in this pass).
5. **`PC_CLNT_PCAT` `IS_ECOM` filter inconsistency** (Edge Case 5) — intentional or
   accidental divergence between Template's and Meta's write filters?
6. **Live DB schema verification** — this doc reflects only what the Go SQL strings imply;
   no pgAdmin/schema cross-check was done.

---

## 15. See also

- [`Profile_Meta_Template_Business_Doc.md`](./Profile_Meta_Template_Business_Doc.md) — same
  flows, product/business perspective, bina code ke
- [`../users_domain_product_stories.md`](../users_domain_product_stories.md) — Story 2
  "Profile & Business Identity Management," including the shared moderation-layer note
- [`../API to Tables/users_api_database_reference.md`](../API%20to%20Tables/users_api_database_reference.md) —
  `actionProfile.go` dead-code finding, originally documented here
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — depth/structure
  reference this doc was rebuilt against

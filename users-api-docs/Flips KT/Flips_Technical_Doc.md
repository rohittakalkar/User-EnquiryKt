# Flips (Social-Commerce Video) — Technical Doc (Code-Level Deep Dive)

Yeh doc Flips feature ka **technical implementation** cover karta hai — APIs, DB tables,
queries, RabbitMQ/Kafka, consumer, sab kuch code se verify karke. Business/product
perspective ke liye [`Flips_Business_Doc.md`](./Flips_Business_Doc.md) dekho.

**Repos**: `users-api-go-production` (read), `service-api-go-production` (write),
`user-temp-consumers-production` (Kafka-driven sync consumer).

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file path diya
gaya hai har jagah, sab files iss pass mein **poore padhe gaye** hain). Jahan code se pura
confirm nahi ho paaya, wahan clearly **[INFERRED — team se confirm karo]** likh diya hai.

**Scope note**: Flips ke chhe alag pieces hain — Account **read**, Content **read**, Account
**write**, Content **write**, Content **validation**, aur ek Kafka-driven **sync consumer**.
Account aur Content write-side same physical tables (`GLUSR_FLIPS_MAP`, `GLUSR_FLIPS_DETAIL`,
`GLUSR_FLIPS_CONTENT_DETAIL`) share karte hain aur dono ek hi `FLIPS_WRITE_SERVICE`
fan-out event trigger karte hain — isliye ek hi combined doc mein hain.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Account read | read | [`FlipsAccountController.go`](../../users-api-go-production/internal/controllers/UsersControllers/FlipsAccountController.go), [`FlipsAccountModel.go`](../../users-api-go-production/internal/models/users/FlipsAccountModel.go) (`FlipsAccountDetail`) |
| Content read | read | [`FlipsContentController.go`](../../users-api-go-production/internal/controllers/UsersControllers/FlipsContentController.go), [`FlipsContentModel.go`](../../users-api-go-production/internal/models/users/FlipsContentModel.go) (`FlipsContentDetail`, `buildFlipsQuery`, `getPcItemDocVideos`, `mergePcItemDocVideos`) |
| Account write | write | [`UserFlipsAccountController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserFlipsAccountController.go), [`UserFlipsAccountModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFlipsAccountModel.go) (`UpdateFlipsToken`, `InsertFlipsAcct`, `UpdateFlipsAcct`, `SendFlipToComp`) |
| Content write | write | [`UserFlipsContentController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserFlipsContentController.go) (`processFlipsRequest`), [`UserFlipsContentModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFlipsContentModel.go) (`InsertFlipsContent`, `buildInsertNotExistsQuery`, `UpdateFlipsContent`, `buildFullUpdateQuery`) |
| Validation (field maps + content-array rules) | write | `FlipsAccountMap`, `FlipsContentMap`, `ValidationFlipsContent` — [`UsersValidationMaps.go:2546,2660,2686`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |
| Sync consumer (**Kafka-driven, calls back into write-API**) | consumers | [`USER_FLIPS_CONTENT_SYNC.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_FLIPS_CONTENT_SYNC.go) — function `FlipsContentSync` / `dbActionFlipsContentSync` |
| Consumer registration (queue-name → handler-function map) | consumers | [`IntializeMsgBroker.go:257-258`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go) |
| `SERVICENAME` → RabbitMQ routing-key map | write | [`rabbitmq.go:60,90`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go) |

---

## 2. Routes

Gin route-registration file (actual `POST /...` path strings) iss pass mein bhi directly
nahi mila — dono write-controllers aur dono read-controllers `FetchParams`/`serviceName`
pattern se hi confirm hue hain, jo iss codebase ka standard convention hai. Route-registration
file confirm karna **Open Question** mein flag kiya gaya hai (section 13).

| serviceName | Repo | Controller | Purpose |
|---|---|---|---|
| `USER_FLIPS_ACC_DETAILS` | read | `FlipsAccountDetail` (called via `UserFlipsAccount` wrapper — note controller function name `UserFlipsAccount` in read-repo is a naming collision with write-repo's `UserFlipsAccountController`, don't confuse the two) | Account-level read (`MAPPING_TYPE` 1=token, 2=meta) |
| — | read | `FlipsContentDetail` | Content read + merged catalog-video read |
| `USER_FLIPS_ACCOUNT` | write | `UserFlipsAccountController` | Account link/update (token or meta) |
| `USER_FLIPS_CONTENT` | write | `UserFlipsContentController` | Content insert/update |

---

## 3. Data Model — Tables

> **Verification note**: yeh sab table/column names Go code ke andar embedded SQL strings se
> liye gaye hain (har row ke saamne file path diya hai). Live DB schema se cross-verify
> **nahi** kiya gaya hai.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_FLIPS_MAP` | document_pg | Account-level vendor-token mapping, **1 row per user** (unique-key upsert) | `FK_GLUSR_USR_ID` (unique, `ON CONFLICT` key), `VENDOR_ID`, `VENDOR_ACCESS_TOKEN` — [`UserFlipsAccountModel.go:17`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFlipsAccountModel.go) |
| `GLUSR_FLIPS_DETAIL` | document_pg | Per-social-media-account detail record (Meta/Instagram/YouTube/Facebook linked accounts), disable-flag versioned | `FK_GLUSR_USR_ID`, `FK_GL_SOCIAL_MEDIA_ID`, `GLUSR_SOCIAL_ACCT_ID`, `GLUSR_FLIPS_DETAIL_ID`, `GLUSR_FLIPS_ACCT_DISABLE_FLAG`, `GLUSR_FLIPS_DETAIL_MOD_DATE`, `GLUSR_SOCIAL_ACCT_ACCESS_TOKEN`, `GLUSR_FLIPS_SIMILARITY_SCORE` — [`UserFlipsAccountModel.go:68,105,143`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFlipsAccountModel.go) |
| `GLUSR_FLIPS_CONTENT_DETAIL` | document_pg | Actual video-content records, per social-account (`FK_GLUSR_FLIPS_DETAIL_ID`) | `GLUSR_FLIPS_CONTENT_DETAIL_ID`, `FK_GLUSR_FLIPS_DETAIL_ID`, `FK_GLUSR_USR_ID`, `GLUSR_SOCIAL_CONTENT_VIDEO_ID`, `GLUSR_SOCIAL_CONTENT_DISABLE_FLAG`, `GLUSR_SOCIAL_CONTENT_LINK_URL` — [`UserFlipsContentModel.go:109-143`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFlipsContentModel.go) |
| `pc_item_doc` / `pc_item` | document_pg (read-side, [`FlipsContentModel.go:255-290`](../../users-api-go-production/internal/models/users/FlipsContentModel.go)) | **Product-catalog video documents** — separate from Flips tables entirely; these are videos uploaded via the product-catalog doc-upload flow, not via a Flips vendor-account | `pc_item_doc_id`, `fk_glusr_usr_id`, `pc_item_doc_path`, `pc_item_doc_type='VIDEO'`, `pc_item_status='A'`, `fk_doc_type_id`, `pc_item_doc_path_id` |

**Important**: teeno core Flips tables `document_pg` physical DB pe hain — baaki zyada-tar
write-paths iss codebase mein `meshpg` use karte hain (dekho GST/other KT docs); Flips iska
apvaad (exception) hai.

---

## 4. Decode This: `fk_doc_type_id` → Social Platform (product-catalog video merge)

Read-side `getPcItemDocVideos()` catalog-videos ko Flips-content ke saath ek hi response mein
merge karta hai (`mergePcItemDocVideos`, de-dupe on `CONTENT_VIDEO_ID`/`pc_item_doc_path_id`).
Is merge ke dauraan, `fk_doc_type_id` (catalog-doc table ka column) ko ek Flips-style
`SOCIAL_MEDIA_ID` + `CONTENT_MEDIA_PRODUCT_TYPE` mein map kiya jaata hai:

| `fk_doc_type_id` | Mapped `SOCIAL_MEDIA_ID` | Mapped `CONTENT_MEDIA_PRODUCT_TYPE` | Platform (inferred) |
|---|---|---|---|
| `1`, `2` | `"1"` (default/else-branch) | `"SHORTS"` | YouTube Shorts **[INFERRED — confirm exact platform-ID convention with team]** |
| `5`, `6` | `"2"` | `"REELS"` | Instagram Reels |
| `3`, `4` | `"3"` | `"FBVIDEOS"` | Facebook Videos |

[`FlipsContentModel.go:307-317`](../../users-api-go-production/internal/models/users/FlipsContentModel.go)
— yeh mapping sirf is one function ke andar hardcoded hai, koi shared enum/constant file mein
nahi. `SOCIAL_MEDIA_ID` values `1`/`2`/`3` ka exact business meaning (kaunsa numeric ID kaunse
platform ko represent karta hai `GL_SOCIAL_MEDIA` master table mein) is pass mein conclusively
confirm nahi hua — **Open Question**, section 13.

`CONTENT_DISP_STATUS` / `GLUSR_SOCIAL_CONTENT_DISABLE_FLAG` 4 values leta hai validation ke
mutabik (`0`,`1`,`2`,`3`) — [`UsersValidationMaps.go:2621`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go).
Sirf `0` (disabled) aur `1` (enabled) ka business meaning code-flow se confirm hota hai (sync
consumer `0`=disable, `1`=enable set karta hai — section 6). `2` aur `3` ke meanings iss pass
mein kahin explicitly handle/branched nahi hue — **[INFERRED/unconfirmed — team se poochho]**.

---

## 5. Business Rules & Validation (code se exhaustive list)

1. **`MAPPING_TYPE` do values leta hai** — `1` = Token-based auth (direct vendor-API token),
   `2` = Meta-based auth (Facebook/Instagram business-login); koi aur value reject hoti hai
   `"Invalid MAPPING_TYPE value."`.
   [`UserFlipsAccountController.go:80`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserFlipsAccountController.go)
2. **Mapping-type-specific mandatory fields**: `MAPPING_TYPE=1` ke liye `VENDOR_ID` +
   `VENDOR_TOKEN` mandatory; `MAPPING_TYPE=2` ke liye `SOCIAL_ACCT_STATUS` + `SOCIAL_ACCT_ID` +
   `SOCIAL_MEDIA_ID` mandatory. [`UserFlipsAccountController.go:86-97`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserFlipsAccountController.go)
3. **`SOCIAL_ACCT_STATUS` sirf `"0"`/`"1"` leta hai** (mapping-type 2 ke liye) —
   [`UserFlipsAccountController.go:98`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserFlipsAccountController.go)
4. **`SIMILARITY_SCORE` range-checked hai** — agar diya gaya ho, `1`-`99` ke beech ek integer
   hona chahiye (`<=0` ya `>=100` reject). [`UserFlipsAccountController.go:104`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserFlipsAccountController.go)
   — note: field-name string mein hi ek trailing space bug hai (`"SIMILARITY_SCORE "` with
   trailing space, line 42) jab map se read kiya jaata hai; iska matlab agar client field-name
   mein trailing space na bheje toh yeh always empty read hoga aur validation skip ho jaayegi
   — worth flagging as a likely bug.
5. **Account-write gateway allowlist**: `FLIPS` / `MAPI` / `IMOB` —
   [`UserFlipsAccountController.go:73`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserFlipsAccountController.go)
6. **Content-write gateway allowlist chhota hai**: sirf `FLIPS` / `IMOB` (`MAPI` missing) —
   [`UserFlipsContentController.go:49`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserFlipsContentController.go)
   — inconsistency confirmed by direct code comparison of the two `Gateway_v1` calls.
7. **`GLUSR_FLIPS_MAP` par `ON CONFLICT (FK_GLUSR_USR_ID) DO UPDATE`** — ek user ka sirf ek hi
   active vendor-token-mapping ho sakta hai; naya token submit karna purana silently
   overwrite kar deta hai, koi history nahi rakhi jaati. [`UserFlipsAccountModel.go:17`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFlipsAccountModel.go)
8. **`GLUSR_FLIPS_DETAIL` insert se pehle duplicate-account check hota hai** — `SOCIAL_ACCT_COUNT`
   query karti hai `FK_GLUSR_USR_ID` + `GLUSR_SOCIAL_ACCT_ID` + `FK_GL_SOCIAL_MEDIA_ID` ke
   against; match milne par insert reject hoti hai `"Social media account already exists."`.
   [`UserFlipsAccountModel.go:68-104`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFlipsAccountModel.go)
9. **Disable-flag CTE pattern (insert path)**: naya detail-row insert hote hi, ek CTE (`WITH
   inserted AS (...)`) baaki purane rows (same user+social-media, `ID <> naya ID`) ko
   `GLUSR_FLIPS_ACCT_DISABLE_FLAG = 0` set kar deta hai — matlab **ek user+platform ke liye
   sirf ek hi active/enabled detail-row honi chahiye**, purani automatically disable ho jaati
   hai. [`UserFlipsAccountModel.go:105`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFlipsAccountModel.go)
10. **Wahi CTE pattern update path pe bhi hai** (`acctStatus == "1"` ke case mein) —
    [`UserFlipsAccountModel.go:172-173`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFlipsAccountModel.go)
11. **Content-write `UPDATE_FLAG` I/U-based hai** (Insert/Update); mandatory:
    `USR_ID`, `FLAG`(`UPDATE_FLAG`), `FLIPS_ID`, `VALIDATION_KEY`, `CONTENT` key present.
    [`UserFlipsContentController.go:41-46`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserFlipsContentController.go)
12. **Insert-side de-dupe hai DB-level `WHERE NOT EXISTS`**: `(usr_id, flips_id, video_id)`
    triple already exist karta ho toh anti-join se silently filter ho jaata hai — agar **sab**
    rows dupe nikle, response `"No record to insert."` (500-coded business-error, not a hard
    failure). [`UserFlipsContentModel.go:36-146`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFlipsContentModel.go)
13. **Content update sirf non-null fields overwrite karta hai** — `COALESCE(v.colX, t.<col>)`
    pattern se, matlab agar payload mein koi field missing/null ho, DB ki existing value
    preserve hoti hai (partial update semantics). [`UserFlipsContentModel.go:289`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFlipsContentModel.go)
14. **`ValidationFlipsContent` per-content-item rules**: `flag=U` par `CONTENT_ID` mandatory
    hai, `flag=I` par **disallowed** hai; `CONTENT_VIDEO_ID` aur `CONTENT_DISP_STATUS` (values
    `0`/`1`/`2`/`3` only) har item mein mandatory hain; `CONTENT_COMMENTS_DETAIL` diya gaya ho
    toh JSON-compatible type ka hona chahiye. [`UsersValidationMaps.go:2546-2626`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go)
15. **Content-write batch-style query use karta hai** — `buildInsertNotExistsQuery` (insert)
    aur `buildFullUpdateQuery` (update) dono ek hi multi-row `VALUES (...)` statement banate
    hain poore `CONTENT` array ke liye, per-row loop nahi (good practice — GST/legacy-Rating
    jaisa per-row-insert issue yahan **nahi** hai). [`UserFlipsContentModel.go:55-146,252-302`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFlipsContentModel.go)
16. **`SendFlipToComp` ek "any-disabled-content" flag compute karta hai** — query check karti
    hai kya iss user ke kisi bhi enabled account (`disable_flag=1`) ka koi content
    `disable_flag IN (1,3)` hai; result `FLIP_FLAG` naam se `FLIPS_WRITE_SERVICE` packet mein
    fan-out hota hai. Exact downstream business-meaning iss flag ka conclusively trace nahi
    hua — **[INFERRED — confirm karo]**. [`UserFlipsAccountModel.go:209-254`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFlipsAccountModel.go)

---

## 6. RabbitMQ

| SERVICENAME | Publisher | Resolved RabbitMQ routing-key/queue | Purpose |
|---|---|---|---|
| `FLIPS_WRITE_SERVICE` | `SendFlipToComp()` — called from **both** `UserFlipsAccountController.go` (after account insert/update) and `UserFlipsContentController.go` (`processFlipsRequest`, after content insert/update) | `comp.sync.<modulus>` (per [`rabbitmq.go:60`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go)), exchange `USER.topic` ([`rabbitmq.go:90`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go)) | Standard generic "company-level data changed" fan-out queue that the wider codebase uses (same pattern documented for GST/other domains) — **not** Flips-specific, and **not** what triggers the Kafka sync-consumer below |

**Correction to prior doc version**: an earlier pass of this doc claimed the sync consumer
listens on this RabbitMQ queue and re-applies duplicate writes. That is **not** what the
current code does — see section 7.

---

## 7. Kafka — the Sync Consumer's Real Trigger

`USER_FLIPS_CONTENT_SYNC` is registered as a **Kafka** consumer, not RabbitMQ:
[`IntializeMsgBroker.go:92-104,257-258`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go)
— it calls `InitializeKafka(queueName, subTopic, consumerGroup, dbstruct)` where `subTopic`
and `consumerGroup` come from environment variables (`sub_topic`, `consumer_group`) set at
deployment — **the actual Kafka topic name is not in source code**
[`USER_FLIPS_CONTENT_SYNC.go:41-42`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_FLIPS_CONTENT_SYNC.go).

**What it actually does (re-verified against current source, corrects the prior doc
version)**: this consumer does **not** write `GLUSR_FLIPS_MAP`/`GLUSR_FLIPS_DETAIL`/
`GLUSR_FLIPS_CONTENT_DETAIL` directly. Instead:

1. Reads existing content rows for `(glusrId, flipsId)` from `glusr_flips_content_detail`
   (SELECT only) — [`readExistingFlipsContent`, USER_FLIPS_CONTENT_SYNC.go:156-171](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_FLIPS_CONTENT_SYNC.go)
2. Diffs against the incoming Kafka message's `CONTENT` array (keyed by `CONTENT_VIDEO_ID`),
   classifying every item into **disable** (in DB, not in fresh payload, currently enabled),
   **update** (in both), or **insert** (in fresh payload only) —
   [`classifyFlipsContent`, USER_FLIPS_CONTENT_SYNC.go:191-234](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_FLIPS_CONTENT_SYNC.go)
3. Batches disable+update together (`UPDATE_FLAG=U`) and insert separately (`UPDATE_FLAG=I`),
   chunked at 30 items per call, and **calls the `/flips/content` write-service HTTP API**
   (`utils.CallReadService(..., "flips_content_write_service", "USER_FLIPS_CONTENT_SYNC")`) —
   i.e. it re-enters the exact same `UserFlipsContentController` → `ValidationFlipsContent` →
   `InsertFlipsContent`/`UpdateFlipsContent` path documented in sections 5-6, rather than
   touching the DB itself. [`callFlipsWriteAPI`, USER_FLIPS_CONTENT_SYNC.go:272-320](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_FLIPS_CONTENT_SYNC.go)
4. On any batch failure, publishes the original message to Kafka fail-topic
   `FLIPS_CONTENT_SYNC_FAIL` via `kafkapublishapi` —
   [`publishToFailTopic`, USER_FLIPS_CONTENT_SYNC.go:340-352](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_FLIPS_CONTENT_SYNC.go)

**Business purpose (inferred)**: naming (`_SYNC`) aur ki yeh Kafka-driven hai (baaki
domain RabbitMQ-based hai), strongly suggest karta hai ki yeh ek **external/bulk content feed**
ka intake path hai — jaise koi upstream system (BI/integration team) periodically poore
GLID/FLIPS_ID ka fresh content-snapshot bhejta hai, aur yeh consumer usse diff karke sirf
delta apply karta hai write-API ke through (taaki write-side validation/business-rules bypass
na ho). **Exact upstream publisher/topic name team se confirm karo** — Open Question, section
13.

---

## 8. Redis

**Koi Redis usage nahi mila** Flips ke kisi bhi 6-piece mein (grep confirm kiya gaya
`FlipsAccountController.go`, `FlipsContentModel.go`, `UserFlipsContentModel.go` samet). Reads
har request pe seedhe `document_pg` hit karte hain, koi caching layer nahi hai.

---

## 9. Cron Inventory

Flips-specific koi cron nahi mila iss pass mein (`user-temp-consumers-production` repo mein
top-level `crons/` folder mein grep-search ke through "flips" ka koi match nahi mila). Sync
mechanism poori tarah event-driven Kafka-consumer hai (section 7), scheduled batch job nahi.

---

## 10. End-to-End Technical Flows (step-by-step, code-level)

### Flow A — Account linking (token-based, `MAPPING_TYPE=1`)

```
Supplier / integration
    │
    ▼
[API — write] POST serviceName=USER_FLIPS_ACCOUNT  {USR_ID, MAPPING_TYPE=1, VENDOR_ID,
                VENDOR_TOKEN, VALIDATION_KEY, unique_id}
    │  UserFlipsAccountController.go
    │  1. Mandatory-field checks, Gateway_v1(FLIPS/MAPI/IMOB)
    │  2. LengthAndTypeValidations_v3 against FlipsAccountMap
    ▼
[DB — document_pg]  UpdateFlipsToken()
    └─ INSERT INTO GLUSR_FLIPS_MAP (...) ON CONFLICT (FK_GLUSR_USR_ID) DO UPDATE ...
    ▼
Response {STATUS, CODE, MESSAGE}  — no comp-sync fan-out on this path (token-only updates
don't call SendFlipToComp; only account-detail insert/update do — see Flow B)
```

### Flow B — Account linking (Meta-based, `MAPPING_TYPE=2`, new account)

```
Supplier / integration
    │
    ▼
[API — write] POST serviceName=USER_FLIPS_ACCOUNT  {USR_ID, MAPPING_TYPE=2, SOCIAL_ACCT_ID,
                SOCIAL_MEDIA_ID, SOCIAL_ACCT_STATUS, VALIDATION_KEY, FLIPS_ID=""(empty→insert)}
    │  UserFlipsAccountController.go
    ▼
[DB — document_pg]  InsertFlipsAcct()
    ├─ SELECT COUNT(...) FROM GLUSR_FLIPS_DETAIL WHERE user+social-media+social-acct-id
    │       (duplicate-account check)
    ├─ if duplicate → REJECT "Social media account already exists."
    └─ if not → WITH inserted AS (INSERT INTO GLUSR_FLIPS_DETAIL ...)
                UPDATE GLUSR_FLIPS_DETAIL SET DISABLE_FLAG=0 WHERE ID <> inserted.ID
                (disables any other stale row for same user+platform)
    ▼
[RabbitMQ]  SendFlipToComp() → SERVICENAME=FLIPS_WRITE_SERVICE → comp.sync.<modulus>
    │  (packet: ACTION=UPDATE, TABLES=GLUSR_FLIPS_DETAIL, COLUMNS={FLIP_FLAG: computed})
    ▼
Response {STATUS, CODE, MESSAGE}
```

### Flow C — Account update (existing `FLIPS_ID` present)

```
[API — write] POST serviceName=USER_FLIPS_ACCOUNT {..., FLIPS_ID: <existing id>}
    │  UserFlipsAccountController.go
    ▼
[DB — document_pg]  UpdateFlipsAcct()
    └─ dynamic UPDATE GLUSR_FLIPS_DETAIL SET <only-fields-present> WHERE usr_id + flips_id
       if SOCIAL_ACCT_STATUS == "1" → wrapped in CTE that also disables sibling rows
       (same user+social-media, different detail-id)
    ▼
[RabbitMQ]  SendFlipToComp() → FLIPS_WRITE_SERVICE → comp.sync.<modulus>
```

### Flow D — Content posting (insert, `UPDATE_FLAG=I`)

```
Supplier / integration
    │
    ▼
[API — write] POST serviceName=USER_FLIPS_CONTENT {USR_ID, UPDATE_FLAG=I, FLIPS_ID,
                VALIDATION_KEY, CONTENT: [array of content items, no CONTENT_ID]}
    │  UserFlipsContentController.go — processFlipsRequest()
    │  1. Mandatory checks, Gateway_v1(FLIPS/IMOB — narrower than account-write)
    │  2. ValidationFlipsContent() — per-item CONTENT_VIDEO_ID/CONTENT_DISP_STATUS checks;
    │     rejects CONTENT_ID presence on insert
    ▼
[DB — document_pg]  InsertFlipsContent() → buildInsertNotExistsQuery()
    └─ single INSERT ... SELECT FROM (VALUES ...) WHERE NOT EXISTS (dedupe on
       usr_id+flips_id+video_id) — one round-trip for the whole batch
    ▼
[RabbitMQ]  if any row inserted → SendFlipToComp() → FLIPS_WRITE_SERVICE → comp.sync.<modulus>
```

### Flow E — Content update (`UPDATE_FLAG=U`)

```
[API — write] POST serviceName=USER_FLIPS_CONTENT {..., UPDATE_FLAG=U,
                CONTENT: [items, each with mandatory CONTENT_ID]}
    │  ValidationFlipsContent() — CONTENT_ID mandatory this time
    ▼
[DB — document_pg]  UpdateFlipsContent() → buildFullUpdateQuery()
    └─ single UPDATE ... FROM (VALUES ...) with COALESCE(v.col, t.col) per-column —
       one round-trip for the whole batch, partial-update semantics
    ▼
[RabbitMQ]  SendFlipToComp() → FLIPS_WRITE_SERVICE → comp.sync.<modulus>
```

### Flow F — Kafka-driven content sync (external feed → diff → write-API)

```
[External producer]  (topic name is deployment-config, not in source — Open Question)
    │
    ▼
[CONSUME — Kafka]  USER_FLIPS_CONTENT_SYNC.go — dbActionFlipsContentSync()
    │  message: {GLUSR_ID, FLIPS_ID, VALIDATION_KEY, CONTENT: [...], MODID, UNIQUE_LOGGING_ID}
    ▼
[DB read — document_pg]  SELECT existing glusr_flips_content_detail rows for (glid, flipsId)
    ▼
[In-memory diff]  classify each existing/fresh video_id → disable / update / insert
    ▼
[HTTP call → write-API]  flips_content_write_service
    ├─ batch(disable+update) → UPDATE_FLAG=U → same UserFlipsContentController path as Flow E
    └─ batch(insert)          → UPDATE_FLAG=I → same UserFlipsContentController path as Flow D
    │  (chunked at 30 items/call)
    ▼
On any batch failure → publish original message to Kafka topic FLIPS_CONTENT_SYNC_FAIL
```

### Flow G — Reads

```
Buyer / storefront-viewer, or supplier's own panel
    │
    ▼
[API — read]  FlipsAccountDetail()  — mapping_type=1 → single-row GLUSR_FLIPS_MAP lookup
              mapping_type=2 → array of GLUSR_FLIPS_DETAIL rows (optionally filtered by
              FK_GL_SOCIAL_MEDIA_ID unless MEDIA_ID="ALL")
    ▼
[API — read]  FlipsContentDetail()
    │  buildFlipsQuery() dynamically builds WHERE (glid | contentID | contentVideoID branch)
    ├─ [goroutine 1] main GLUSR_FLIPS_CONTENT_DETAIL JOIN GLUSR_FLIPS_DETAIL query
    └─ [goroutine 2, only if glid given] getPcItemDocVideos() — separate pc_item_doc query
       for product-catalog videos, fk_doc_type_id → SOCIAL_MEDIA_ID mapped (section 4)
    ▼
mergePcItemDocVideos() — merges both result sets, de-dupes on video-ID, both queries run
    in parallel goroutines (already a good practice — see Optimization Scope)
    ▼
Response — combined FLIPS_VIDEOS + PC_ITEM_DOC_VIDEOS video list
```

---

## 11. Flow-wise DB & Table Usage

### Flow A/C — Account write (token or Meta-update)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | document_pg | `GLUSR_FLIPS_MAP` | UPSERT (`ON CONFLICT DO UPDATE`) | Token path — sirf vendor-id/token likha jaata hai |
| 1 | document_pg | `GLUSR_FLIPS_DETAIL` | UPDATE (dynamic SET, + conditional disable-CTE) | Meta-update path — jo fields diye gaye unhi ko update karta hai |
| 2 | (comp.sync fan-out) | — | RabbitMQ publish | `SendFlipToComp` internally ek aur SELECT bhi karta hai (see row below) |

### Flow B — Account write (Meta insert, new account)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | document_pg | `GLUSR_FLIPS_DETAIL` | SELECT (`COUNT(CASE WHEN...)`) | Duplicate-account check, same social-acct-id ke against |
| 2 | document_pg | `GLUSR_FLIPS_DETAIL` | INSERT (CTE) + UPDATE (sibling disable) | Naya detail-row banata hai, purane sibling rows disable karta hai — ek hi statement mein |
| 3 | document_pg | `GLUSR_FLIPS_DETAIL` + `GLUSR_FLIPS_CONTENT_DETAIL` (JOIN) | SELECT (`SendFlipToComp` ke andar `EXISTS` check) | `FLIP_FLAG` compute karne ke liye — kya iss user ka koi enabled account/disabled-content combo hai |
| 4 | (comp.sync fan-out) | — | RabbitMQ publish | Downstream ko notify karta hai |

### Flow D/E — Content write (insert/update)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | document_pg | `GLUSR_FLIPS_CONTENT_DETAIL` | single batched INSERT...SELECT...WHERE NOT EXISTS (insert path) OR single batched UPDATE...FROM VALUES (update path) | Poore `CONTENT` array ke liye **ek hi round-trip** — batch pattern, per-row loop nahi |
| 2 | document_pg | `GLUSR_FLIPS_DETAIL` + `GLUSR_FLIPS_CONTENT_DETAIL` (JOIN) | SELECT (`SendFlipToComp`) | Same `FLIP_FLAG` computation jo account-write path bhi use karta hai |
| 3 | (comp.sync fan-out) | — | RabbitMQ publish | Downstream notify |

### Flow F — Kafka sync consumer

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | document_pg | `glusr_flips_content_detail` | SELECT | Existing rows padhta hai diff karne ke liye |
| 2+ | *(no direct write — HTTP calls back into Flow D/E)* | — | — | Consumer khud DB nahi likhta; write-API re-enter karta hai, isliye Flow D/E ka poora DB-cost bhi is flow ka hissa hai |

### Flow G — Reads

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | document_pg | `GLUSR_FLIPS_MAP` (mapping_type=1) OR `GLUSR_FLIPS_DETAIL` (mapping_type=2) | SELECT | Account read |
| 2 | document_pg | `GLUSR_FLIPS_CONTENT_DETAIL` JOIN `GLUSR_FLIPS_DETAIL` | SELECT | Content read (main query, parallel goroutine 1) |
| 3 | document_pg | `pc_item_doc` LEFT JOIN `pc_item` | SELECT (DISTINCT ON, only when `glid` present) | Catalog-video merge (parallel goroutine 2) |

**Total DB round-trips per content-write event including the fan-out check: 2** (batched
write + `SendFlipToComp` SELECT) — much leaner than GST's 9-10, because the batch-VALUES
pattern collapses what would otherwise be N round-trips into 1.

---

## 12. Optimization Scope — DB Response-Time Contribution

### Medium impact

1. **Kafka sync-consumer re-enters the write-API over HTTP instead of writing directly** —
   every synced batch pays the full write-API round-trip cost (network hop + validation +
   DB write) rather than a direct DB write. This is very likely intentional (keeps validation/
   business-rules single-sourced, avoids logic duplication — a real improvement over the
   "duplicate direct-write" pattern this doc previously (incorrectly) attributed to it), but it
   does mean the consumer's overall latency is bounded by the write-API's response time,
   compounded across up to `ceil(N/30)` batches for insert and update separately. Worth
   monitoring batch-count distribution if content volumes per sync grow large.
   [`sendFlipsBatchGroup`, USER_FLIPS_CONTENT_SYNC.go:254-269](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_FLIPS_CONTENT_SYNC.go)
2. **Content-write gateway allowlist inconsistency (`MAPI` missing)** — not a performance issue
   per se, but if `MAPI`-originated calls are expected to reach content-write and currently
   fail gateway validation, that's a silent-failure risk worth a quick fix regardless.

### Low impact

3. **Read-side already parallelizes the two independent queries** (`getPcItemDocVideos` +
   main Flips query) via goroutines and channels — this is a **positive finding**, not a gap;
   GST/legacy docs flag missing parallelism as an opportunity, Flips already has it here.
   [`FlipsContentModel.go:92-139`](../../users-api-go-production/internal/models/users/FlipsContentModel.go)
4. **Content insert/update already use single-round-trip batch `VALUES(...)` patterns**
   (`buildInsertNotExistsQuery`, `buildFullUpdateQuery`) — also a positive finding, this is the
   exact optimization GST's audit doc recommends elsewhere in the codebase, and Flips content-
   write already does it.
5. **No caching on any Flips read endpoint** (section 8) — same low-risk caching opportunity
   as other domains, worth considering only if read volume/latency ever becomes a concern;
   Flips account/content data changes relatively rarely per user.

---

## 13. Edge Cases & Gotchas (technical POV)

1. **Prior version of this doc's central finding was wrong** — it claimed the Kafka
   sync-consumer duplicates direct writes to `GLUSR_FLIPS_MAP`/`GLUSR_FLIPS_DETAIL`. Re-reading
   the current `USER_FLIPS_CONTENT_SYNC.go` in full shows it does a DB **read only**, then calls
   back into the write-API over HTTP — it does not touch `GLUSR_FLIPS_MAP` or `GLUSR_FLIPS_DETAIL`
   at all, only reads/writes `GLUSR_FLIPS_CONTENT_DETAIL` (indirectly, via the write API). If
   this doc is being compared against an older version, treat this version as authoritative
   (verified against the file as it exists today) and flag the discrepancy to the team as a
   possible recent refactor.
2. **`SIMILARITY_SCORE` field-name typo** (trailing space in the map lookup key, section 5.4)
   — likely means this validation branch never actually fires from real client requests unless
   the client also sends a trailing space, which would be unusual. Worth a code-level fix
   confirmation with the team.
3. **Gateway allowlist asymmetry** (`MAPI` allowed for account-write, not for content-write) —
   confirm with team whether intentional.
4. **`CONTENT_DISP_STATUS` values `2` and `3`** appear in validation's allowed-set but are not
   visibly branched-on anywhere else read in this pass (the sync-consumer only ever sets `0`
   or `1`) — likely meaningful states set/consumed elsewhere not covered by these 6 files.
5. **Route-registration file (`POST /...` path strings) not directly located** — all routing
   evidence in this doc is via `serviceName` constants and controller-signature conventions
   consistent with the rest of this codebase, not a router.go grep hit.
6. **`document_pg` is a separate physical DB** from most other write-paths (which mostly use
   `meshpg`) — no cross-DB joins/transactions are possible within this domain if Flips data
   ever needs combining with other user-data in a single query.
7. **Account-write's `UpdateFlipsToken` (token path) never calls `SendFlipToComp`** — only the
   Meta-detail insert/update paths (`InsertFlipsAcct`/`UpdateFlipsAcct`) and content
   insert/update trigger the `FLIPS_WRITE_SERVICE` fan-out. If downstream systems expect to be
   notified on every token refresh too, they currently are not.

---

## 14. Open Questions

1. `USER_FLIPS_CONTENT_SYNC`'s Kafka topic name and upstream publisher — set via `sub_topic`
   env var at deployment, not visible in source. Confirm with team which system publishes to
   it and on what trigger/frequency.
2. `SOCIAL_MEDIA_ID` numeric-value → platform mapping (`1`/`2`/`3` and beyond) — reconstructed
   here only from the `fk_doc_type_id` merge logic (section 4); the canonical `GL_SOCIAL_MEDIA`
   master-table values were not located in this pass.
3. `CONTENT_DISP_STATUS` values `2` and `3` — allowed by validation but not observed being set
   anywhere in the 6 files read for this doc; confirm their business meaning.
4. `FLIP_FLAG` (computed in `SendFlipToComp`) — exact downstream consumer/business-meaning of
   this boolean not traced past the RabbitMQ publish in this pass.
5. Actual Gin route path strings (`POST /flips/account`, `/flips/content` etc.) — not directly
   located; confirmed only via serviceName/controller convention.
6. `SIMILARITY_SCORE` trailing-space lookup (section 5.4, 13.2) — is this a live bug or
   intentional (e.g. client always sends it with a space)? Needs team confirmation before
   filing as a defect.
7. Live DB schema verification (column types, nullability, indexes, constraints) for all
   tables in section 3 — this doc only reflects what the Go SQL strings imply.

---

## See also

- [`Flips_Business_Doc.md`](./Flips_Business_Doc.md) — same flows, product/business
  perspective, bina code ke
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — depth/structure
  reference this doc follows
- [`../Social Review KT/Social_Review_Technical_Doc.md`](../Social%20Review%20KT/Social_Review_Technical_Doc.md) —
  similar social-content pattern in this codebase

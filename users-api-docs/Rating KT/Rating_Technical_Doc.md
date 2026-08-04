# Rating (Supplier Rating) — Technical Doc (Code-Level Deep Dive)

**Scope note**: yeh doc sirf **Rating** (buyer's 1-5 star + comment ka supplier ke against)
cover karta hai. **Rating Usefulness** (`USER_RATING_USEFULNESS` — "was this rating
helpful?" vote) alag concept hai, alag consumer/table set use karta hai, apna KT folder
already ban chuka hai —
[`../Rating Usefulness KT/Rating_Usefulness_Technical_Doc.md`](../Rating%20Usefulness%20KT/Rating_Usefulness_Technical_Doc.md).
Yeh doc us folder ko cross-reference karta hai jahan overlap hai (same controller/table
internally reused).

Business/product perspective ke liye [`Rating_Business_Doc.md`](./Rating_Business_Doc.md)
dekho.

**Repos**: `users-api-go-production` (read), `service-api-go-production` (write),
`user-temp-consumers-production` (9 consumers + 1 shared with archival), plus 2 standalone
crons in `service-api-go-production/crons/recommend/`.

**Methodology**: har claim neeche real source code se trace kiya gaya hai, exact file:line
citations ke saath. Jahan code se pura confirm nahi ho paaya, wahan
**[INFERRED — team se confirm karo]** likha hai ya Open Questions mein daala hai. Yeh pass
pichle doc se **caafi zyada deep** hai — read-controller ab pura confirm ho chuka hai, saare
9 consumer poori tarah re-read hue hain, aur RabbitMQ ka `serviceToQueueMap`/`exchangeSet`
poora trace hua hai (jo pichli baar sirf partial tha).

---

## 1. Rating Kahan-Kahan Hai — File Map

| Concern | Repo | File |
|---|---|---|
| Rating submit/update/reply (write) | write | [`UserSupplierRatingController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSupplierRatingController.go) (327 lines), [`UserSupplierRatingModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSupplierRatingModel.go) (865 lines — `InsertRating`, `UpdateRating`) |
| Rating read (profile page pe display) — **confirmed this pass** | read | [`SupplierRatingController.go`](../../internal/controllers/UsersControllers/SupplierRatingController.go) (`GetSupplierRating`, calls `users.SupplierRating`), [`SupplierRatingVersionsController.go`](../../internal/controllers/UsersControllers/SupplierRatingVersionsController.go) (`GetSupplierRatingVersions`, calls `users.SupplierRating_v1` — a newer variant), models: [`UserSupplierRatingModel.go`](../internal/models/users/UserSupplierRatingModel.go), `_v1`, `Old`, `New` variants |
| "Reasons" dropdown (Response/Quality/Delivery config) | read | [`GetInfluParametersController.go`](../../internal/controllers/UsersControllers/GetInfluParametersController.go) — function `ActionGetInfluParams`, reads a **static JSON file** (`ratinginfluparam`/`ratinginfluparam_gke` path), no DB at all |
| Content moderation (abuse/PII check) | consumer | [`USER_SUPPLIER_RATING_BANNED.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SUPPLIER_RATING_BANNED.go) — func `mainActionUserSuppRatingBanned` |
| Buyer-supplier matchmaking verification | consumer | [`USER_RATING_BS_MATCHMAKING.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_BS_MATCHMAKING.go) — func `dbActionUserRatingBSMatch` |
| Product/category (MCAT) enrichment | consumer | [`USER_RATING_ENRICHMENT.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_ENRICHMENT.go) — func `processMessageUserRatingEnrich` |
| Aggregate (star-count rollup + LMS-status log) | consumer | [`USER_RATING_AGGREGATE.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_AGGREGATE.go) — func `workerRatingAggregate` |
| Notification to supplier (push notification) | consumer | [`USER_RATING_NOTIFICATION.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_NOTIFICATION.go) — func `sendRatingNotification` |
| Notification-adjacent DB replication (approvalPg) | consumer | [`USER_RATING_NOTIFICATION_PG.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_NOTIFICATION_PG.go) — func `dbActionUserRatingNotifyPG` — **misleading name**, doesn't send notifications, replicates writes into `approvalPg` (see §5, §11) |
| Old-rating archival | consumer | [`USER_RATING_ARCHIVE.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_ARCHIVE.go) — func `dbActionUserRatingArchive` |
| Seller-risk signal (Kafka) | consumer | [`USER_RATING_SELLER_RISK.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_SELLER_RISK.go) — func `sellerRisk` |
| Sync to Lead-Management System (LMS), dedicated path | consumer | [`USER_SUPPLIER_RATING_TO_LMS.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SUPPLIER_RATING_TO_LMS.go) — func `dbActionSuppRatingToLms` |
| 91-day-old aggregate re-trigger cron — **not in prior doc** | cron | [`avg_rating.go`](../../service-api-go-production/service-api-go-production/crons/recommend/avg_rating.go) |
| Rating fraud/suspect-flag cron — **not in prior doc** | cron | [`rating_suspect_cron.go`](../../service-api-go-production/service-api-go-production/crons/recommend/rating_suspect_cron.go) |
| Queue-name/exchange resolution (all `SERVICENAME`s) | write | [`rabbitmq.go`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go) — `PushToQueue()`, `serviceToQueueMap`, `exchangeSet` (lines 20-91) |
| Consumer queue registration | consumer | [`router.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Router/router.go), [`IntializeMsgBroker.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go) |

---

## 2. Routes (confirmed)

| Method | Path | Repo | Controller |
|---|---|---|---|
| POST | `/supplierrating` | write | [`router.go:170,338`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) → `UserControllers.UserSupplierRatingController` (branches internally on `UPDATE_FLAG`: absent/not-`U` = insert, `U` = update/reply/admin-review) |
| GET/POST | `/supplierrating/*params` | read | [`routerUsers.go:130-131`](../../internal/api/users_router/routerUsers.go) → `UsersControllers.GetSupplierRating` → `users.SupplierRating()` |
| GET/POST | `/supplierrating_versions/*params` | read | [`routerUsers.go:134-135`](../../internal/api/users_router/routerUsers.go) → `UsersControllers.GetSupplierRatingVersions` → `users.SupplierRating_v1()` — a parallel/newer read variant, **structurally near-identical controller** to `GetSupplierRating` (same validation block, different model function) |
| GET/POST | `/getinfluparams/*params` | read | [`routerUsers.go:128-129`](../../internal/api/users_router/routerUsers.go) → `UsersControllers.ActionGetInfluParams` (static file, no DB) |

Note: `internal/api/router/router.go` (the other, legacy router variant in this repo) has
`/supplierrating` commented out at one registration point (line 101-102) but active at
another (line 130) — both ultimately point to the same controller.

---

## 3. Data Model — Tables

> **Verification note**: table/column names Go embedded SQL strings se liye gaye hain, live
> schema se cross-verify nahi kiya gaya.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_RATING` | meshpg | **Primary rating record** — har individual rating (star, comment, display-status) yahan store hoti hai | `GLUSR_RATING_ID`, `FK_GLUSR_BUYER_ID`, `FK_GLUSR_SUPPLIER_ID`, `GLUSR_RATING_VALUE`, `GLUSR_RATING_COMMENTS`, `GLUSR_RATING_DATE`, `FK_GLUSR_RATING_TYPE`, `FK_GLUSR_RATING_SOURCE`, `GLUSR_RATING_DISPLAY_STATUS` (default `-2` on insert, hardcoded — [`UserSupplierRatingModel.go:476`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSupplierRatingModel.go)), `FK_GLUSR_RATING_INFLU_PARAM_ID`, `GLUSR_RATING_MCAT_ID`, `is_rating_docs_available`, `influ_params_thumbsup_cnt`, `glusr_rating_pmcat_id`, `glusr_rating_cat_id`, `GLUSR_RATING_REVIEWED_STATUS`/`_BY`/`_DATE`, `GLUSR_RATING_REVIEW_COMMENT`, `GLUSR_RATE_COM_BUYDISP_STATUS`/`_SUPDISP_STATUS`, `FK_RATING_DISPOSITION_ID`, `RATING_SUSPECT_FLAG` (added for the fraud-cron flow, §9) — [`UserSupplierRatingModel.go:597`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSupplierRatingModel.go) (INSERT), lines 35-148 (UPDATE) |
| `glusr_rating_reply` | meshpg | Supplier's reply text to a rating — a **separate table**, not a column on `GLUSR_RATING` | `FK_GLUSR_RATING_ID`, `GLUSR_RATING_REPLY_DATE`, `GLUSR_RATING_RATEE_COMMENTS`, `GLUSR_RATING_REPLY_ID` (PK, RETURNING) — [`UserSupplierRatingModel.go:98`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSupplierRatingModel.go) |
| `GLUSR_RATING_DETAILS` | meshpg | Har influence-parameter (Response/Quality/Delivery) pe thumbs up/down ka individual vote | `FK_GLUSR_RATING_ID`, `GLUSR_RATING_VALUE` (1=up, 0=down), `FK_GLUSR_SUPP_ID`, `FK_GLUSR_RATING_INFLU_PARAM_ID` — [`UserSupplierRatingModel.go:635`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSupplierRatingModel.go) |
| `GLUSR_RATING_DOCS` | meshpg | Rating ke saath attach ki hui photos | `FK_GLUSR_RATING_ID`, `DOC_REVIEWED_STATUS`, `GLUSR_RATING_DOC_ID`, `DOC_PATH`, `IMG_DOC_500X500`, `IMG_DOC_125X125`, `FK_RATING_DISPOSITION_ID`, `DOC_REVIEW_COMMENT` — [`UserSupplierRatingModel.go:676`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSupplierRatingModel.go) |
| `GLUSR_RATING_AGGREGATE` | meshpg | Har supplier ka **rolled-up summary** — full-recompute-on-conflict pattern | `FK_GLUSR_ID`, `AVG_RATING` (via `fn_get_wt_avg_rating()` DB function), `ONE_STAR_CNT`...`FIVE_STAR_CNT`, `INFLU_PARAMS_CNT` (JSON via `json_agg`), `GLUSR_RATING_AGGREGATE_MOD_DATE` — [`USER_RATING_AGGREGATE.go:169-236`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_AGGREGATE.go) — filters `COALESCE(glusr_rating_display_status,0) > -1`, `INSERT ... ON CONFLICT (fk_glusr_id) DO UPDATE`, aur agar koi ratings hi nahi bachi toh row `DELETE` ho jaati hai (`NOT EXISTS insert_update`) |
| `GLUSR_RATING_LOG` | meshpg | Ek per-rating log jo LMS-matchmaking-status bhi capture karta hai — **only on INSERT action** | `GLUSR_RATING_ID`, `FK_GLUSR_SUPPLIER_ID`, `GLUSR_RATING_DATE`, `LMS_MATCHMAKING_STATUS_FLAG` — [`USER_RATING_AGGREGATE.go:242-263`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_AGGREGATE.go) (`dbActionRatingsLog`, runs in a **goroutine parallel to the aggregate recompute** — a genuine parallelization example already present in this codebase) |
| `glusr_rating_arch` | meshpg | Archive table — exact mirror schema of `glusr_rating`, old ratings moved here | Dynamic column list built at runtime from `glusr_rating`'s own columns (`INSERT INTO glusr_rating_arch (%s) VALUES (%s)`) — [`USER_RATING_ARCHIVE.go:69,105`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_ARCHIVE.go) |
| `GLCAT_MCAT_TO_MCAT`, `GLCAT_CAT_TO_MCAT` | meshpg (read-only, subquery) | Category-hierarchy lookup — `pmcat_id`/`cat_id` derive karne ke liye, **embedded subquery**, extra round-trip nahi | [`UserSupplierRatingModel.go:597`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSupplierRatingModel.go) (INSERT), [`UserSupplierRatingModel.go:81-82`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSupplierRatingModel.go) (UPDATE, when `GLUSR_RATING_MCAT_ID` changes) |
| `glusr_contactbook_mapping` | buyerprofilePg | Matchmaking check yahan se padhta hai — confirms buyer-supplier genuinely connected | `GLUSR_USR_ID1`, `GLUSR_USR_ID2`, `source_id` — query: `SELECT COUNT(1) ... WHERE GLUSR_USR_ID1=$1 AND GLUSR_USR_ID2=$2 AND coalesce(source_id,-1) in (-1,1,3,4)` — [`USER_RATING_BS_MATCHMAKING.go:87`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_BS_MATCHMAKING.go) |
| `GLUSR_USR` | meshpg | Buyer's name/company, read by the notification consumer to build the push-notification title | `GLUSR_USR_FIRSTNAME`, `_LASTNAME`, `_PH_COUNTRY`, `_PH_MOBILE`, `_COMPANYNAME` — [`USER_RATING_NOTIFICATION.go:142`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_NOTIFICATION.go) |
| `GLUSR_RATING_USEFULNESS`, `GLUSR_RATING` (`RATING_USEFULNESS_COUNT`/`RATING_ABUSE_COUNT` — corrected column names, see Rating Usefulness KT) | meshpg/approvalPg | Belongs to the **Rating Usefulness** feature, out of scope here — see [`../Rating Usefulness KT/Rating_Usefulness_Technical_Doc.md`](../Rating%20Usefulness%20KT/Rating_Usefulness_Technical_Doc.md). Noted here only because `USER_RATING_NOTIFICATION_PG.go` writes to both this table AND `GLUSR_RATING` depending on payload shape (§5) | — |
| `R_MESSAGE_CENTER_BIZFEED_LMS[0-4]` | (message-center system, external) | Not a DB table this domain owns — it's the **downstream LMS ingestion queue set** that `USER_SUPPLIER_RATING_TO_LMS` publishes into (sharded 0-4 by `supplier_id % 5`) | — [`USER_SUPPLIER_RATING_TO_LMS.go:84-96`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SUPPLIER_RATING_TO_LMS.go) |

---

## 4. Decode-This-Magic-Value — `GLUSR_RATING_DISPLAY_STATUS` and fraud/suspect flags

`GLUSR_RATING_DISPLAY_STATUS` is the single most important "hidden state machine" column in
this domain. Traced values, all from code (not from a lookup table — no such table found):

| Value | Meaning | Where set |
|---|---|---|
| `-2` | **Default on insert** — pending, never shown publicly until reviewed/passed all checks | [`UserSupplierRatingModel.go:476`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSupplierRatingModel.go) (hardcoded, request value ignored) |
| `-1` | **Disabled/hidden** — set explicitly by the matchmaking consumer when no genuine buyer-supplier connection is found (`count == 0`), and by the fraud/suspect cron for several suspect-flag branches | [`USER_RATING_BS_MATCHMAKING.go:105`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_BS_MATCHMAKING.go), [`rating_suspect_cron.go:264,272`](../../service-api-go-production/service-api-go-production/crons/recommend/rating_suspect_cron.go) |
| `0` | Also used as a "disabled/neutral" value in the aggregate's `COALESCE(...,0)` and in some suspect-cron branches (banned-product flag=8) | [`USER_RATING_AGGREGATE.go:201`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_AGGREGATE.go), [`rating_suspect_cron.go:270-277`](../../service-api-go-production/service-api-go-production/crons/recommend/rating_suspect_cron.go) |
| `> -1` (i.e. `0` or higher, presumably positive values used elsewhere for "visible/approved") | Counted into `GLUSR_RATING_AGGREGATE`'s star-counts and influence-param counts | [`USER_RATING_AGGREGATE.go:186,205`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_AGGREGATE.go) |
| Exact positive value that marks "fully approved/visible" | **[INFERRED — confirm with team]** — not observed as a literal write anywhere in this pass; likely set by an admin-review flow (`GLUSR_RATING_REVIEWED_STATUS='Y'`) or defaults once nothing disables it, but no single explicit `DISPLAY_STATUS = <positive>` write was found in the 3 repos searched | — |

### `RATING_SUSPECT_FLAG` (via `fn_get_rating_suspect_flag` DB function, driven by `rating_suspect_cron.go`)

This is a **second, separate fraud-detection layer**, distinct from the matchmaking check,
run entirely by a standalone cron (not a RabbitMQ consumer) — **not documented at all in the
prior pass**. The cron calls a Postgres function `fn_get_rating_suspect_flag(buyer_id,
supplier_id, rating_value, ip, rating_id, rating_date)` which returns `(out_suspect_flag,
out_rating_to_disable)`. The Go code then interprets flag values via a **priority map**:

| Flag | Priority (lower = higher priority) | Action taken | Where |
|---|---|---|---|
| `8` | 0 (highest) | Banned-product — `DISPLAY_STATUS = 0` | [`rating_suspect_cron.go:270-277`](../../service-api-go-production/service-api-go-production/crons/recommend/rating_suspect_cron.go) |
| `7` | 1 | `DISPLAY_STATUS = -1` | same, case `1,2,5,6,7` block |
| `1` | 2 | `DISPLAY_STATUS = -1` (also drives `HandleFlag1Or2Flow` — fraud-graph "shortest path" check via `fraud.indiamart.com`) | [`rating_suspect_cron.go:122-137, 262-269`](../../service-api-go-production/service-api-go-production/crons/recommend/rating_suspect_cron.go) |
| `2` | 3 | `DISPLAY_STATUS = -1` — set when `HandleFlag1Or2Flow` returns code `200` (graph-connection confirmed) | same |
| `5` | 4 | `DISPLAY_STATUS = -1` | same |
| `6` | 5 (lowest of the "disable" set) | `DISPLAY_STATUS = -1` | same |
| `0` / any other | — | `DISPLAY_STATUS = 0`, no special action | default branch |

**Meaning of flags 1/2/5/6/7 individually**: **[INFERRED — confirm with team]** — the cron
only encodes *priority ordering* and the *resulting display-status action*, it never
comments what each numeric flag semantically represents (e.g. is `5` "same-IP multiple
ratings", is `6` "circular rating ring"?). This entire semantic mapping is opaque from code
alone and should be confirmed with whoever owns `fn_get_rating_suspect_flag` (a DB-side
function, not in this Go codebase).

The cron additionally does a **buyer-pair cross-check**: for a supplier with multiple same-day
ratings, it calls the fraud-graph API (`HandleFlag1Or2Flow`) pairwise across all buyers who
rated that supplier that day, and if **3 or more pairwise checks succeed**, marks the
associated ratings suspect (flag `1`) — otherwise flag `0`
([`rating_suspect_cron.go:169-222`](../../service-api-go-production/service-api-go-production/crons/recommend/rating_suspect_cron.go)). This is a **ring-detection heuristic** — buyers who all know each other rating the same supplier the same day.

---

## 5. Business Rules & Validation (code se)

1. **`RATING_VAL` sirf `1`-`5` accept hota hai** (skipped when `CALLEDFROM != ""`, i.e. internal
   consumer calls bypass this) — [`UserSupplierRatingController.go:125`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSupplierRatingController.go)
2. **`BUYER_ID != SUPPLIER_ID`** (same `CALLEDFROM==""` exemption) —
   [`UserSupplierRatingController.go:127`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSupplierRatingController.go)
3. **`DISPLAY_STATUS` hamesha `-2` se shuru hoti hai insert pe**, request value ignored — a
   server-side enforced default — [`UserSupplierRatingModel.go:476`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSupplierRatingModel.go)
4. **Existing-relationship detection**: insert se pehle `SELECT COUNT(*) FROM GLUSR_RATING
   WHERE fk_glusr_supplier_id=$1 AND fk_glusr_buyer_id=$2` — result sets `lms_flag` (`I`=new,
   `U`=existing) jo `TO_LMS` sub-packet mein `insertion_type` field banta hai —
   [`UserSupplierRatingModel.go:497-520`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSupplierRatingModel.go)
5. **Influence-params encoding**: request array/map/string/nil kisi bhi format mein aa sakta
   hai, normalized ek `{"0": thumbsDownIds, "1": thumbsUpIds}` shape mein. Agar dono khali hon
   aur ek single `GLUSR_RATING_INFLU_PARAM_ID` diya ho: star `< 3` = thumbs-down auto-classify,
   `>= 3` = thumbs-up — [`UserSupplierRatingModel.go:522-591`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSupplierRatingModel.go)
6. **`influ_params_thumbsup_cnt` precomputed per-rating** — aggregate-consumer ko har baar
   `GLUSR_RATING_DETAILS` join na karna pade.
7. **Rating photos optional**, count sirf non-empty `IMG_ID` waali entries ki hoti hai —
   [`UserSupplierRatingModel.go:681-685`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSupplierRatingModel.go)
8. **Update path (`UPDATE_FLAG=U`) mein 3 alag mandatory-field sets hain, exact branching
   confirmed this pass** (line-numbers below refer to
   [`UserSupplierRatingController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSupplierRatingController.go)):
   - `IS_ADMIN=="1"` + `UPDATE_FLAG=="U"` → requires `RATING_ID/VALIDATION_KEY/UPDATEDBY/UPDATEDUSING/IP/IP_COUNTRY` (line 119)
   - `UPDATE_FLAG=="U"` + `CALLEDFROM==""` (non-admin) → requires additionally `SUPPLIER_ID` (line 121)
   - `UPDATE_FLAG!="U"` + `CALLEDFROM==""` (fresh insert) → requires `BUYER_ID/SUPPLIER_ID/UPDATEDBY/VALIDATION_KEY/UPDATEDUSING/IP/IP_COUNTRY/RATING_SOURCE/RATING_VAL` (line 123)
   - `CALLEDFROM=="RATING_USEFULNESS_WRITE"` + `UPDATE_FLAG=="U"` → requires `RATING_ID/VALIDATION_KEY/UPDATEDBY/HELPFUL_COUNT/ABUSE_COUNT/IP/IP_COUNTRY` (line 129) — confirms the controller is genuinely **reused by the Rating Usefulness feature** (see [`../Rating Usefulness KT/`](../Rating%20Usefulness%20KT/))
9. **`UpdateRating`'s internal branching — fully re-traced this pass** (was an Open Question
   before; resolved now):
   - `IS_ADMIN` present → builds admin-review columns (reviewed status, display status,
     disposition, suspect flag), conditionally updates `GLUSR_RATING_DOCS` reviewed-status per
     image, then pushes **`NEW_COMPANY`** then **`SUPPLIER_RATING_NEW`** in sequence (both,
     always, when `IS_ADMIN` — [`UserSupplierRatingModel.go:307-357`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSupplierRatingModel.go))
   - Not admin, `SUPPLIER_COMMENTS` present → this is the **supplier-reply flow** — inserts
     into `glusr_rating_reply`, pushes `SUPPLIER_FEEDBACK` with `TO_LMS` sub-packet
     (`insertion_type: "U"`) (lines 358-393)
   - Not admin, no `SUPPLIER_COMMENTS`, `CALLEDFROM=="RATING_USEFULNESS_WRITE"` → pushes
     `SUPPLIER_RATING_NEW` only, keyed by `RATING_ID` not `SUPPLIER_ID` (lines 410-422)
   - Not admin, no `SUPPLIER_COMMENTS`, generic update → pushes `SUPPLIER_RATING_NEW`, then
     (unless `FROM_WORKER=="1"`, i.e. this call itself came from the banned-content worker,
     preventing an infinite consumer loop) also pushes **`USER_RATING_BANNED`** (lines 423-450)
10. **Admin review path can also carry `RATING_SUSPECT_FLAG`** — this is how the fraud/suspect
    cron (§4, §9 crons) writes its verdict back through the same write API, `IS_ADMIN=1`,
    `CALLEDFROM` absent — [`rating_suspect_cron.go:295-317`](../../service-api-go-production/service-api-go-production/crons/recommend/rating_suspect_cron.go)
11. **`FROM_WORKER` flag** on the banned-content consumer's own callback prevents it from
    re-triggering itself — [`UserSupplierRatingModel.go:437`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSupplierRatingModel.go), [`USER_SUPPLIER_RATING_BANNED.go:420`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SUPPLIER_RATING_BANNED.go)
12. **Archival keeps only the latest rating per buyer-supplier pair** — query does
    `ORDER BY glusr_rating_id DESC OFFSET 1` (i.e. everything except the newest), moves each
    matched row into `glusr_rating_arch`, deletes from `glusr_rating` and
    `GLUSR_RATING_LOG`, then re-triggers an aggregate recompute for the supplier —
    [`USER_RATING_ARCHIVE.go:69,105,113,129,143`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_ARCHIVE.go)

---

## 5b. Content-Moderation Pipeline Detail (`USER_SUPPLIER_RATING_BANNED`)

This consumer's logic is significantly richer than "call one API" — it's a **multi-stage,
API-call-minimizing cascade**, fully re-traced this pass (file:
[`USER_SUPPLIER_RATING_BANNED.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SUPPLIER_RATING_BANNED.go)):

1. **CBS API check on supplier reply** (if `UPDATE_FLAG=="U"` and `GLUSR_RATING_SUPPLIER_COMMENTS` present) — one call, sets `SUPP_COMM_DISPLAY=-1` if banned keyword found (lines 100-114)
2. **CBS API check on combined product+mcat+buyer-comment string** — one call combining all three fields, avoids 3 separate calls when possible (lines 116-206). If banned-keyword hits are found, does **local regex matching** (word-boundary + space-evasion detection, e.g. `"d i a z e p a m"` matched against `diazepam`) to figure out *which* of the three fields actually contains the banned word, setting `IS_SUSPECTED_PRODUCT` / `BUYER_COMM_DISPLAY=-1` accordingly (lines 144-205)
3. **Fallback**: only if the combined check found *something* banned but couldn't attribute it to a specific field via regex, falls back to 3 individual CBS calls (product name, mcat name, buyer comment) — lines 208-252
4. **PII API check** on the buyer's comment text — detects emails/phones, and if found, publishes to a **separate "suspected queue"** (`utils.PublishToSuspectedQueue`, `SUSPECT_TYPE=2947`) in addition to hiding the comment (lines 263-306)
5. **ML banned-keyword-prediction API** — only called if CBS found nothing AND comment isn't empty (an optimization: skip an extra API call when CBS already confirmed a ban) — lines 278-293
6. Final: if anything was flagged, calls back into `UserSupplierRatingModel.UpdateRating` (via `rating_write_service`) with `FROM_WORKER=1` to actually hide the comment/reply

**Business logic embedded in code, not obviously discoverable elsewhere**: the CBS API call
always passes `modid: "SUPPLIER_FEEDBACK"` and `no_subcat_flag: 1` — [`USER_SUPPLIER_RATING_BANNED.go:367`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SUPPLIER_RATING_BANNED.go).

---

## 6. RabbitMQ — Queues Used

**This pass fully traced `PushToQueue()`'s `serviceToQueueMap`/`exchangeSet`
([`rabbitmq.go:20-91`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go)) — this resolves the prior doc's two biggest Open Questions** (exact routing
key per `SERVICENAME`, and whether `SUPPLIER_FEEDBACK` fans out via topic-routing).

| `SERVICENAME` (as pushed) | Routing key / queue name | Exchange | Publisher(s) | Consumer(s) (bound, inferred from routing-key pattern — actual bindings are infra config, not in this repo) |
|---|---|---|---|---|
| `SUPPLIER_FEEDBACK` | `user.supplierrating.<glid % 20>` | `USER.topic` | `InsertRating` (new rating), `UpdateRating` (supplier-reply branch) | **Topic-exchange fan-out** — all of `USER_SUPPLIER_RATING_BANNED`, `USER_RATING_ENRICHMENT`, `USER_RATING_AGGREGATE`, `USER_RATING_NOTIFICATION`, `USER_RATING_SELLER_RISK`, `USER_SUPPLIER_RATING_TO_LMS` are registered against `SUPPLIER_FEEDBACK` as their `serviceName` (see `UserRatingEnrich`, `UserSuppRatingBanned`, `UserRatingSellerRisk` all set `serviceName = "SUPPLIER_FEEDBACK"`) — meaning this **single publish fans out to (up to) 6 independent consumers** via topic-exchange binding patterns on `user.supplierrating.*` |
| `USER_RATING_BANNED` | `USER_SUPPLIER_RATING_BANNED` (direct queue name, no exchange) | *(none — direct)* | `UpdateRating` (generic-update branch, unless `FROM_WORKER=="1"`) | `USER_SUPPLIER_RATING_BANNED` consumer — re-runs moderation on an **update** (e.g. supplier's edited reply) |
| `NEW_COMPANY` | `user.rating_aggregate` | `USER.topic` | `UpdateRating` (`IS_ADMIN` branch, rabbit1), `avg_rating.go` cron (91-day retrigger) | `USER_RATING_AGGREGATE` consumer — **confirms this is in fact how admin-review/cron-triggered aggregate recomputes happen** (the prior doc marked this `[INFERRED]`; now confirmed via the shared `queueName := "user.rating_aggregate"` literal in the cron) |
| `SUPPLIER_RATING_NEW` | `user.ratingnotification.*` | `USER.topic` | `UpdateRating` (`IS_ADMIN` branch rabbit2, `RATING_USEFULNESS_WRITE` branch rabbit4, generic-update branch rabbit5) | `USER_RATING_NOTIFICATION` / `USER_RATING_NOTIFICATION_PG` (both registered against a `SUPPLIER_RATING_NEW`-pattern queue — `NOTIFICATION_PG`'s own `serviceName = "SUPPLIER_RATING_NEW"` confirms this literally) |
| `USER_RATING_BS_MATCHMAKING` | *(registered directly as a named queue, no `serviceToQueueMap` entry found — likely published by an upstream buyer-seller-connection event, not from this domain's own write path)* | — | **[INFERRED]** not the rating-write API itself — see cross-reference to [`../Matchmaking KT/Matchmaking_Technical_Doc.md`](../Matchmaking%20KT/Matchmaking_Technical_Doc.md) which documents the disable-payload from the matchmaking side | `USER_RATING_BS_MATCHMAKING` consumer |

**Key correction vs. prior doc**: `SUPPLIER_FEEDBACK` is **not** "a generic comp.sync-style
fan-out with no single dedicated consumer" as previously stated — it is a **topic-exchange
routing key that up to 6 rating-domain consumers all bind against**, confirmed by every one
of those consumer files literally setting `serviceName = "SUPPLIER_FEEDBACK"` at
initialization (this is how they self-identify for Kibana logging even though they consume
from differently-named queues bound to the same routing-key pattern).

**Confirmed staleness point (still true, re-verified)**: `USER_RATING_BS_MATCHMAKING`, when
disabling a rating (`count==0`), calls the write API with `CALLEDFROM="MATCHMAKING_CONSUMER"`
+ `IS_ADMIN=1`. This routes through `UpdateRating`'s `IS_ADMIN` branch, which **always**
pushes `NEW_COMPANY` (→ `USER_RATING_AGGREGATE`) — meaning **the aggregate staleness gap
claimed in the prior doc is actually NOT present**: disabling via matchmaking *does* trigger
an aggregate recompute, because it goes through the same `IS_ADMIN` path as any other admin
review. This is a **correction to the prior doc**, not a confirmation of its claim — see
§12 Edge Cases.

---

## 7. Kafka

**Confirmed usage found this pass** — the prior doc's "no Kafka" claim was incomplete.
[`USER_RATING_SELLER_RISK.go:101`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_SELLER_RISK.go) calls `utils.CallKafkaService(valueMap, "user_seller_risk")`, publishing to
topic `user_seller_risk_topic` = `"risk_score_prod"` ([`config.yaml:332`](../../user-temp-consumers-production/user-temp-consumers-production/data/config.yaml)), payload `{rater_glid, glid, event_name: "glusr_rating", rating,
comment, event_timestamp}`. This consumer discards `UPDATE_FLAG` and `MATCHMAKING_CONSUMER`/`MCAT_CONSUMER`-sourced messages (fresh inserts only) before publishing.

No other Kafka usage found in the write path or other 8 consumers.

---

## 8. Redis

**Koi Redis usage nahi mila** — confirmed again this pass across all rating write/read files
and all 9 consumer files (`grep -li redis` returned zero hits). `GET /supplierrating` and
`/supplierrating_versions` har baar live DB hit karte hain. High-read endpoint (supplier
profile page), no caching layer — see §11 Optimization Scope.

---

## 9. Cron Inventory — **not present in prior doc, both confirmed this pass**

| Cron | File | Trigger | What it does |
|---|---|---|---|
| 91-Days-Old Avg Rating Recalculation | [`avg_rating.go`](../../service-api-go-production/service-api-go-production/crons/recommend/avg_rating.go) | Scheduled (external cron scheduler, not in-code — likely daily, [INFERRED]) | `SELECT DISTINCT FK_GLUSR_SUPPLIER_ID, FK_GLUSR_BUYER_ID FROM GLUSR_RATING WHERE date(GLUSR_RATING_DATE) = date(now()) - 91` — for every such pair, pushes a `NEW_COMPANY` message directly (bypassing the write-API's `PushToQueue` wrapper's modulus logic by constructing `queue_name` itself) to force an aggregate recompute for ratings turning 91 days old. Purpose **[INFERRED — confirm with team]**: likely a weighted-average-decay mechanic where `fn_get_wt_avg_rating()` weighs recent ratings more, so this forces a periodic recompute even with no new activity |
| Rating Suspect/Fraud Flag Cron | [`rating_suspect_cron.go`](../../service-api-go-production/service-api-go-production/crons/recommend/rating_suspect_cron.go) | Scheduled, queries `GLUSR_RATING WHERE glusr_rating_display_status = -2 AND date(glusr_rating_date) >= current_date - 1` (i.e. yesterday+today's still-pending ratings) | Runs `fn_get_rating_suspect_flag()` DB function per rating, cross-checks buyer-pairs via an external fraud-graph API (`fraud.indiamart.com/path/?mode=shortestPath`), and writes back `DISPLAY_STATUS`/`RATING_SUSPECT_FLAG` via the same `/supplierrating` write API with `IS_ADMIN=1`. See §4 for full flag semantics. This is the **primary fraud-detection mechanism for Rating** — separate from, and in addition to, the real-time matchmaking check |

---

## 10. End-to-End Technical Flows

### Flow A — Naya rating submit (insert)

```
Buyer
    │
    ▼
[API — write]  POST /supplierrating (no UPDATE_FLAG, or UPDATE_FLAG != "U")
    │  UserSupplierRatingController.go
    │  1. Mandatory-field validation (BUYER_ID/SUPPLIER_ID/RATING_VAL/RATING_SOURCE/...)
    │  2. RATING_VAL 1-5 check, BUYER_ID != SUPPLIER_ID check
    │  3. Gateway/modid validation (allowed_modids whitelist)
    │  4. ValidationSupplierRating() — field-level validation
    ▼
[DB #1]  SELECT COUNT(*) FROM GLUSR_RATING (existing-relationship check → lms_flag I/U)
    ▼
[DB #2]  INSERT INTO GLUSR_RATING (DISPLAY_STATUS forced to -2)
    │  — pmcat_id/cat_id subqueries embedded (no extra round-trip)
    │  RETURNING GLUSR_RATING_ID
    ▼
[DB #3 — conditional]  INSERT INTO GLUSR_RATING_DETAILS (thumbs up/down per influence param)
[DB #4 — conditional]  INSERT INTO GLUSR_RATING_DOCS (photos)
    ▼
[RabbitMQ publish]  SERVICENAME=SUPPLIER_FEEDBACK → routing key user.supplierrating.<mod20>
    │  carries full GLUSR_RATING row + embedded TO_LMS sub-packet
    ▼
[Topic-exchange fan-out, async — buyer doesn't wait]
    ├─ USER_SUPPLIER_RATING_BANNED   → CBS + PII + ML moderation cascade (§5b)
    ├─ USER_RATING_BS_MATCHMAKING    → verify real buyer-supplier connection (glusr_contactbook_mapping)
    ├─ USER_RATING_ENRICHMENT        → product/category context (LMS API)
    ├─ USER_RATING_AGGREGATE         → recompute supplier's star-summary (+ GLUSR_RATING_LOG in parallel goroutine)
    ├─ USER_RATING_NOTIFICATION      → push notification to supplier
    └─ USER_SUPPLIER_RATING_TO_LMS   → dedicated Lead-Management sync (R_MESSAGE_CENTER_BIZFEED_LMS[0-4])
```

### Flow B — Admin/system review path (`IS_ADMIN=1`, `UPDATE_FLAG=U`)

```
[API — write]  POST /supplierrating (UPDATE_FLAG="U", IS_ADMIN="1")
    │  Callers: GLADMIN UI, USER_RATING_BS_MATCHMAKING consumer (CALLEDFROM=MATCHMAKING_CONSUMER),
    │  USER_RATING_ENRICHMENT consumer (CALLEDFROM=MCAT_CONSUMER), rating_suspect_cron.go
    ▼
UserModels.UpdateRating()  — IS_ADMIN branch
    │  UPDATE GLUSR_RATING SET (review status/date, display_status, disposition, suspect flag, ...)
    │  Conditionally UPDATE GLUSR_RATING_DOCS per-image reviewed-status
    ▼
[RabbitMQ]  NEW_COMPANY  → USER_RATING_AGGREGATE (recompute)
[RabbitMQ]  SUPPLIER_RATING_NEW → USER_RATING_NOTIFICATION(_PG)
```

### Flow C — Supplier reply

```
Supplier
    │
    ▼
[API]  POST /supplierrating (UPDATE_FLAG="U", SUPPLIER_COMMENTS present, no IS_ADMIN)
    │  UpdateRating() — SUPPLIER_COMMENTS branch
    ▼
[DB]  INSERT INTO glusr_rating_reply (RETURNING GLUSR_RATING_REPLY_ID)
[DB]  UPDATE GLUSR_RATING SET GLUSR_RATING_REPLY_DATE=current_timestamp, ...
    ▼
[RabbitMQ]  SUPPLIER_FEEDBACK (with TO_LMS sub-packet, insertion_type="U")
    │  → same topic fan-out as Flow A (re-triggers moderation on the reply text too)
```

### Flow D — Archival + re-aggregate

```
[Trigger: any event that reaches USER_RATING_ARCHIVE — publisher not directly traced this
 pass, likely fired alongside INSERT/UPDATE events for the same pair — Open Question]
    │
    ▼
USER_RATING_ARCHIVE consumer
    │  SELECT ... FROM glusr_rating WHERE buyer=X AND supplier=Y ORDER BY id DESC OFFSET 1
    │  (i.e. everything except the newest rating for this pair)
    ▼
    for each row: INSERT INTO glusr_rating_arch, DELETE FROM glusr_rating, DELETE FROM GLUSR_RATING_LOG
    ▼
    dbActionRatingAggregate(supplier_id)  — direct in-process call, re-triggers aggregate recompute
```

### Flow E — Read (profile page)

```
Buyer / App
    │
    ▼
[API — read]  GET /supplierrating or /supplierrating_versions
    │  GetSupplierRating / GetSupplierRatingVersions
    │  Heavy inline validation (token, MODID, GLUSR ID format/length, sort_type enum 1-9,
    │  comp_flag, mcat_id+product_name required together for sort_type=8)
    ▼
[DB]  users.SupplierRating() / SupplierRating_v1() — joins GLUSR_RATING + GLUSR_RATING_AGGREGATE
    │  (+ GLUSR_USR for showroom/custtype) — no caching, live query every time
    ▼
Response: star average, per-star counts, individual rating list, influence-param %
```

### Flow F — Fraud/suspect cron (daily batch)

```
[Cron scheduler — external, not in-repo]
    │
    ▼
rating_suspect_cron.go: SELECT ... WHERE display_status = -2 AND date >= current_date - 1
    │
    ▼
For each pending rating:
    │  fn_get_rating_suspect_flag(buyer, supplier, rating_val, ip, rating_id, date) — DB function
    │  → (suspect_flag, rating_to_disable[])
    ▼
[conditional] HandleFlag1Or2Flow() — external fraud-graph "shortest path" API call(s),
    │  pairwise across same-day buyers of the same supplier (ring-detection heuristic)
    ▼
TriggerRatingUpdate() → POST /supplierrating (IS_ADMIN=1, DISPLAY_STATUS, RATING_SUSPECT_FLAG)
    │  → same Flow B admin-review path → NEW_COMPANY + SUPPLIER_RATING_NEW pushed
```

---

## 11. Flow-wise DB & Table Usage Matrix

The single most useful section of this doc (per the GST-doc convention) — every DB
round-trip, per flow, with the reason.

### Flow A — New rating insert

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1 | meshpg | `GLUSR_RATING` | SELECT COUNT(*) | Detect new vs. existing buyer-supplier relationship (drives `lms_flag`) |
| 2 | meshpg | `GLUSR_RATING` (+ embedded subqueries on `GLCAT_MCAT_TO_MCAT`, `GLCAT_CAT_TO_MCAT`) | INSERT ... RETURNING | Create the rating row, derive pmcat/cat inline |
| 3 | meshpg | `GLUSR_RATING_DETAILS` | INSERT (batched, multi-row) | Store thumbs up/down per influence param, conditional on presence |
| 4 | meshpg | `GLUSR_RATING_DOCS` | INSERT ... RETURNING (batched) | Store photo metadata, conditional on presence |
| 5 (async, `USER_SUPPLIER_RATING_BANNED`) | — | (external CBS/PII/ML APIs) | HTTP calls | Content moderation, no DB write unless something is flagged |
| 5b (conditional write-back) | meshpg | `GLUSR_RATING` | UPDATE (via write-API, `FROM_WORKER=1`) | Hide comment/reply if moderation flagged something |
| 6 (async, `USER_RATING_BS_MATCHMAKING`) | buyerprofilePg | `glusr_contactbook_mapping` | SELECT COUNT(1) | Verify genuine buyer-supplier connection |
| 6b (conditional write-back) | meshpg | `GLUSR_RATING` | UPDATE (via write-API, `IS_ADMIN=1`) | Disable rating if no connection found — cascades into Flow B |
| 7 (async, `USER_RATING_ENRICHMENT`) | — | (LMS read API) | HTTP call | Fetch product/category/transaction context |
| 7b (conditional write-back) | meshpg | `GLUSR_RATING` | UPDATE (via write-API, `CALLEDFROM=MCAT_CONSUMER`) | Persist enriched mcat/product/connection-date fields |
| 8 (async, `USER_RATING_AGGREGATE`) | meshpg | `GLUSR_RATING_DETAILS`, `GLUSR_RATING` (via join in CTE) | SELECT (full scan per supplier) | Compute star-counts + influence-param counts |
| 9 | meshpg | `GLUSR_RATING_AGGREGATE` | INSERT ... ON CONFLICT DO UPDATE / DELETE | Persist rolled-up summary |
| 10 (parallel goroutine) | meshpg | `GLUSR_RATING_LOG` | INSERT | Log LMS-matchmaking-status for this rating (INSERT action only) |
| 11 (async, `USER_RATING_NOTIFICATION`) | meshpg | `GLUSR_USR` | SELECT | Fetch buyer's name/company for notification text |
| 12 (async, `USER_RATING_SELLER_RISK`) | — | (Kafka publish `risk_score_prod`) | — | Feed seller risk-scoring pipeline, no DB |
| 13 (async, `USER_SUPPLIER_RATING_TO_LMS`) | meshpgLB | `GLUSR_RATING`, `GLUSR_RATING_ARCH` (via `getRecentRatingId`) | SELECT (union query, checks both live + archived) | Determine `insertion_type` (I/U) for the dedicated LMS-sync packet |
| 14 (async, `USER_SUPPLIER_RATING_TO_LMS`) | — | (publish to `R_MESSAGE_CENTER_BIZFEED_LMS[0-4]` + `lms-gcp.topic`) | PubAPI | Deliver the dedicated LMS packet |

### Flow B — Admin review

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1 | meshpg | `GLUSR_RATING` | UPDATE | Review status, display status, disposition, suspect flag |
| 2 (conditional) | meshpg | `GLUSR_RATING_DOCS` | UPDATE (batched via `ExecuteQueryBlock`) | Per-image reviewed status/comment/disposition |
| 3 (async) | meshpg | (see Flow A #8-9) | — | `NEW_COMPANY` → aggregate recompute |
| 4 (async) | approvalPg + meshPg | `GLUSR_RATING` (via `USER_RATING_NOTIFICATION_PG`) | UPDATE (both DBs) | Replicate admin-review fields into `approvalPg` |

### Flow C — Supplier reply

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1 | meshpg | `glusr_rating_reply` | INSERT ... RETURNING | Store reply text |
| 2 | meshpg | `GLUSR_RATING` | UPDATE (`GLUSR_RATING_REPLY_DATE`) | Mark reply timestamp |
| 3 (async) | — | (same as Flow A #5-14, re-triggered) | — | Moderation re-runs on reply text too |

### Flow D — Archival

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1 | meshpg | `glusr_rating` | SELECT ... ORDER BY id DESC OFFSET 1 | Fetch all but the newest rating for the pair |
| 2 (per row) | meshpg | `glusr_rating_arch` | INSERT (dynamic column list) | Move the row to archive |
| 3 (per row) | meshpg | `glusr_rating` | DELETE | Remove from live table |
| 4 (per row) | meshpg | `GLUSR_RATING_LOG` | DELETE | Clean up the log row too |
| 5 | meshpg | (Flow A #8-9) | — | Re-trigger aggregate recompute after archival |

### Flow E — Read

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1 | meshpg | `GLUSR_RATING` + `GLUSR_RATING_AGGREGATE` (+ `GLUSR_USR`) | SELECT (joined) | Build the profile-page rating response — **no caching, every request hits DB live** |

### Flow F — Suspect cron

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1 | meshpg | `GLUSR_RATING` | SELECT (pending ratings, last 2 days) | Batch source for fraud scan |
| 2 (per row) | meshpg | `fn_get_rating_suspect_flag()` (DB function) | SELECT (function call) | Compute suspect verdict |
| 3 (conditional, per row) | meshpg | `glusr_rating` | SELECT (same-day ratings for supplier) | Ring-detection buyer-pair cross-check |
| 4 (per verdict) | — | (writes back via `/supplierrating` write API — see Flow B) | — | Apply the verdict |

---

## 12. Optimization Scope — DB/Response-Time Contribution

Ranked High/Medium/Low, code-cited.

### High-impact

1. **`InsertRating` runs up to 4 sequential DB queries in one HTTP request** (existing-check →
   main insert → influence-details insert → docs insert), all on the same `meshpg`
   connection, **sequentially not parallel** — but `GLUSR_RATING_DETAILS` insert and
   `GLUSR_RATING_DOCS` insert are mutually independent once `RATING_ID` is known.
   **Concrete fix**: run query6 (influ-details) and query7 (docs) as goroutines — same
   pattern already proven in `USER_RATING_AGGREGATE.go`'s `dbActionRatingsLog` (§3), which
   *does* run in a parallel goroutine alongside the main aggregate query. This is a
   demonstrated-safe pattern in this exact codebase, not a theoretical suggestion.
   [`UserSupplierRatingModel.go:634-755`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSupplierRatingModel.go)
2. **`GLUSR_RATING_AGGREGATE` full-recompute-on-every-trigger** — a supplier-wide scan across
   `GLUSR_RATING` + `GLUSR_RATING_DETAILS` fires on every insert, every admin action, every
   archival, and the 91-day cron. For high-rating-volume suppliers this doesn't scale;
   an incremental increment/decrement-counter approach (like GST tech doc's suggested
   `INSERT...RETURNING` pattern) would avoid the full-table scan.
   [`USER_RATING_AGGREGATE.go:169-236`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_AGGREGATE.go)
3. **Content-moderation consumer can make up to 5 sequential external API calls** per rating
   (CBS-reply, CBS-combined, up to 3 CBS-fallback calls, PII, ML) — all synchronous,
   sequential, each with its own timeout risk. The combined-check-first design (§5b) is a
   good mitigation already in place, but the fallback path (product/mcat/comment individually)
   can still stack up to 5 total HTTP round-trips for one rating.
   [`USER_SUPPLIER_RATING_BANNED.go:100-252`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SUPPLIER_RATING_BANNED.go)
4. **`USER_SUPPLIER_RATING_TO_LMS`'s `getRecentRatingId` runs a UNION query across both live
   `GLUSR_RATING` and archived `GLUSR_RATING_ARCH`** for every single rating message — this is
   an extra DB round-trip on the *consumer* side (in addition to the main write path), and
   the UNION-across-two-tables shape suggests missing covering indexes if either table grows
   large. [`USER_SUPPLIER_RATING_TO_LMS.go:112-141`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SUPPLIER_RATING_TO_LMS.go)

### Medium-impact

5. **`GET /supplierrating` / `/supplierrating_versions` have no caching** — supplier profile
   page (high-traffic, low-change-frequency data) is a natural cache-aside candidate. Short
   TTL (2-5 min) would meaningfully reduce load for high-traffic suppliers.
6. **Two near-duplicate read controllers** (`GetSupplierRating` and `GetSupplierRatingVersions`)
   have almost byte-identical validation blocks (confirmed this pass — the two files are
   ~95% identical code, differing only in which model function is called and the
   `kibana_msg["SERVICE_NAME"]` value). This is a maintenance-cost/optimization concern —
   any validation fix must be applied twice, and any future bug fix risks drifting between
   the two. Worth refactoring into a shared helper.
7. **Category-hierarchy subqueries** (`GLCAT_MCAT_TO_MCAT`, `GLCAT_CAT_TO_MCAT`) are already
   smartly embedded inside the insert/update query (no extra round-trip) — but if these
   tables lack indexes on `FK_CHILD_MCAT_ID`/`FK_GLCAT_MCAT_ID`, every rating insert/mcat-update
   pays a scan cost. Worth an index-verification pass.
8. **Rating-suspect cron does an external fraud-graph HTTP call pairwise across all same-day
   buyers of a supplier** (`HandleFlag1Or2Flow`, O(n²) in buyer count for that
   supplier-day) — for a supplier receiving many ratings on the same day, this could spike
   external-API call volume quadratically. [`rating_suspect_cron.go:169-186`](../../service-api-go-production/service-api-go-production/crons/recommend/rating_suspect_cron.go)

### Low-impact / good practice already present

9. **`influ_params_thumbsup_cnt` precomputed column** — aggregate-consumer avoids a
   `GLUSR_RATING_DETAILS` join where possible.
10. **`dbActionRatingsLog` already runs in a parallel goroutine** alongside the main aggregate
    query inside `workerRatingAggregate` — a genuine, already-implemented example of the
    parallelization pattern recommended in point 1 above. Worth pointing to as the internal
    reference implementation when doing that refactor.

---

## 13. Edge Cases & Gotchas (technical POV)

1. **`USER_RATING_NOTIFICATION_PG`'s name is misleading** — it does not send notifications
   (that's `USER_RATING_NOTIFICATION`, no `_PG` suffix); it replicates rating-related writes
   into `approvalPg` (and sometimes `meshPg` too) for several different sub-payload shapes
   (admin review, reply, mark-useful, helpful-count update, fresh insert). Anyone searching
   for "where does the notification actually send" by file name alone will be misled.
   [`USER_RATING_NOTIFICATION_PG.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_NOTIFICATION_PG.go)
2. **Aggregate staleness claim from the prior doc is corrected, not confirmed** — matchmaking
   disable *does* reach `USER_RATING_AGGREGATE` via the `IS_ADMIN` → `NEW_COMPANY` path (§6).
   If staleness is still observed in production, the cause is elsewhere (e.g. the async
   fan-out ordering — nothing prevents `USER_RATING_AGGREGATE`'s own SUPPLIER_FEEDBACK-triggered
   recompute from running *before* the matchmaking check disables the rating, since both are
   independent consumers off the same publish with no ordering guarantee across queues).
3. **Two LMS-sync paths still exist and are architecturally different, not just redundant**:
   the inline `TO_LMS` packet (Flow A, carried inside the `SUPPLIER_FEEDBACK` message) feeds
   an "enquiry-style" transaction record; `USER_SUPPLIER_RATING_TO_LMS` is a **separate,
   dedicated consumer** that does its own `insertion_type` lookup (via a live+archive UNION
   query) and publishes into a **sharded queue set** (`R_MESSAGE_CENTER_BIZFEED_LMS[0-4]`) —
   these look like they serve genuinely different LMS ingestion endpoints, not simple
   duplication. Still worth confirming with the LMS team which one is authoritative.
4. **`RATING_SUSPECT_FLAG`'s five/six numeric codes (1,2,5,6,7,8) have no in-repo semantic
   documentation** — the priority-ordering and resulting action are fully traced (§4), but
   *what each flag number means* lives inside `fn_get_rating_suspect_flag()`, a Postgres
   function not in this codebase.
5. **`CALLEDFROM == "RATING_USEFULNESS_WRITE"` confirms cross-feature reuse** of this exact
   controller by Rating Usefulness — see
   [`../Rating Usefulness KT/Rating_Usefulness_Technical_Doc.md`](../Rating%20Usefulness%20KT/Rating_Usefulness_Technical_Doc.md) §4 for the other side of this shared code path.
6. **`GetSupplierRating` and `GetSupplierRatingVersions` are near-duplicate controllers**
   (§12 point 6) — a bug fixed in one may silently persist in the other.
7. **The fraud-graph API call (`fraud.indiamart.com`) and the two hardcoded JWT-style `AK`
   tokens in `rating_suspect_cron.go` and `USER_SUPPLIER_RATING_BANNED.go`'s PII-API call are
   committed directly in source** (not read from a secrets store in this code path) — a
   security-hygiene note worth flagging, though out of scope to "fix" in a KT doc.

---

## 14. Open Questions

1. What is the exact positive `GLUSR_RATING_DISPLAY_STATUS` value(s) that mean "fully visible"?
   No literal write was found in this pass — likely set implicitly (absence of any disabling
   consumer) or via an admin-approval path not covered by the 3 repos searched.
2. Semantic meaning of `RATING_SUSPECT_FLAG` values `1, 2, 5, 6, 7, 8` individually (only the
   priority-ordering and resulting display-status action are known from Go code) — lives in
   `fn_get_rating_suspect_flag()`, a DB-side function.
3. What publishes to `USER_RATING_ARCHIVE`'s queue? Not traced in this pass or the prior one —
   likely tied to insert/update events for the same buyer-supplier pair, but no `serviceToQueueMap`
   entry or direct `PushToQueue` call targeting it was found in the write-API repo.
4. What publishes `USER_RATING_BS_MATCHMAKING`'s queue in the first place (the initial trigger,
   before the consumer calls back into `UpdateRating`)? **[INFERRED]** likely the rating-insert
   fan-out itself (topic exchange, §6) — the consumer's own `serviceName = "USER_DETAIL_SERVICE"`
   (not `SUPPLIER_FEEDBACK`) is a discrepancy worth clarifying, since every other
   `SUPPLIER_FEEDBACK`-fan-out consumer sets `serviceName = "SUPPLIER_FEEDBACK"` but this one
   doesn't — possibly this consumer is bound to a *different* routing key/queue than the other
   five, not the same `user.supplierrating.*` pattern. Needs infra-side (RabbitMQ bindings)
   confirmation, not just code.
5. Actual RabbitMQ **binding** configuration (which queues bind to `user.supplierrating.*`,
   `user.rating_aggregate`, `user.ratingnotification.*`) is infra config, not in either repo —
   the routing-key *destination strings* are confirmed from code, but the physical queue-to-
   binding wiring itself is not.
6. Schedule/trigger mechanism for both crons (`avg_rating.go`, `rating_suspect_cron.go`) —
   not in-repo (external cron scheduler config).
7. Live DB schema verification — this doc reflects only what Go SQL strings imply.

---

## See also

- [`Rating_Business_Doc.md`](./Rating_Business_Doc.md) — same flows, product perspective
- [`../Rating Usefulness KT/Rating_Usefulness_Technical_Doc.md`](../Rating%20Usefulness%20KT/Rating_Usefulness_Technical_Doc.md) — the "helpful?" vote feature, which reuses `UserSupplierRatingController` internally (§5 point 8, §13 point 5)
- [`../Matchmaking KT/Matchmaking_Technical_Doc.md`](../Matchmaking%20KT/Matchmaking_Technical_Doc.md) — documents `USER_RATING_BS_MATCHMAKING`'s disable-payload from the matchmaking side; cross-check before assuming either doc alone is complete
- [`../ratings_reviews_product_overview.md`](../ratings_reviews_product_overview.md) — bigger Ratings & Reviews story (Social Reviews samet)
- [`../full_read_write_picture.md`](../full_read_write_picture.md) — supplier-rating ka poora worked-example trace
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md), [`../Privacy Setting KT/Privacy_Setting_Technical_Doc.md`](../Privacy%20Setting%20KT/Privacy_Setting_Technical_Doc.md) — similar-shape domain docs, useful comparison for the parallelization/caching optimization patterns referenced in §12

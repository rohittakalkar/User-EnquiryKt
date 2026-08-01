# Rating (Supplier Rating) — Technical Doc (Code-Level Deep Dive)

**Scope note**: yeh doc sirf **Rating** (buyer's 1-5 star + comment ka supplier ke against)
cover karta hai. **Rating Usefulness** (`USER_RATING_USEFULNESS` / `RATING_HELPFUL` — "was
this rating helpful?" vote) alag concept hai, alag file/table/consumer set use karta hai,
aur uska apna KT folder baad mein banega.

Business/product perspective ke liye [`Rating_Business_Doc.md`](./Rating_Business_Doc.md)
dekho.

**Repos**: `users-api-go-production` (read), `service-api-go-production` (write),
`user-temp-consumers-production` (consumers).

**Methodology**: har claim neeche real source code se trace kiya gaya hai. Jahan code se
pura confirm nahi ho paaya, wahan **[INFERRED — team se confirm karo]** likha hai.

---

## 1. Rating Kahan-Kahan Hai — File Map

| Concern | Repo | File |
|---|---|---|
| Rating submit/update/reply | write | [`UserSupplierRatingController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSupplierRatingController.go), [`UserSupplierRatingModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSupplierRatingModel.go) (functions `InsertRating`, `UpdateRating`) |
| Rating read (profile page pe display) | read | [`UserSupplierRatingModel.go`](../internal/models/users/UserSupplierRatingModel.go) (+ `_v1`, `Old`, `New` variants), controller not directly grep'd in this pass but confirmed route below |
| "Reasons" dropdown (Response/Quality/Delivery config) | read | [`GetInfluParametersController.go`](../internal/controllers/UsersControllers/GetInfluParametersController.go) (static config, no DB) |
| Content moderation (abuse/PII check) | consumers | [`USER_SUPPLIER_RATING_BANNED.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SUPPLIER_RATING_BANNED.go) |
| Buyer-supplier matchmaking verification | consumers | [`USER_RATING_BS_MATCHMAKING.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_BS_MATCHMAKING.go) |
| Product/category enrichment | consumers | `USER_RATING_ENRICHMENT` worker (file `USER_RATING_ENRICHMENT.go`, referenced in router as `workers.UserRatingEnrich`) |
| Aggregate (star-count/influence-param rollup) | consumers | [`USER_RATING_AGGREGATE.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_AGGREGATE.go) |
| Notification to supplier | consumers | `USER_RATING_NOTIFICATION` / `USER_RATING_NOTIFICATION_PG` workers |
| Old-rating archival | consumers | `USER_RATING_ARCHIVE` worker |
| Seller-risk signal | consumers | `USER_RATING_SELLER_RISK` worker |
| Sync to Lead-Management System (LMS) | consumers | `USER_SUPPLIER_RATING_TO_LMS` worker |

---

## 2. Routes (confirmed)

| Method | Path | Repo | Controller |
|---|---|---|---|
| POST | `/supplierrating` | write | `UserSupplierRatingController` (branches internally on `UPDATE_FLAG`: absent/not-`U` = insert, `U` = update/reply) |
| GET | `/supplierrating` | read | `SupplierRatingController` (+ `SupplierRatingVersionsController`, a newer/faster variant) |
| GET | `/getinfluparams` | read | `GetInfluParametersController` (static, no DB) |

---

## 3. Data Model — Tables

> **Verification note**: table/column names Go embedded SQL se liye gaye hain, live schema
> se cross-verify nahi kiya gaya.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_RATING` | meshpg | **Primary rating record** — har individual rating (star, comment, display-status) yahan store hoti hai | `GLUSR_RATING_ID`, `FK_GLUSR_BUYER_ID`, `FK_GLUSR_SUPPLIER_ID`, `GLUSR_RATING_VALUE`, `GLUSR_RATING_COMMENTS`, `GLUSR_RATING_DATE`, `FK_GLUSR_RATING_TYPE`, `FK_GLUSR_RATING_SOURCE`, `GLUSR_RATING_DISPLAY_STATUS` (default `-2` on insert), `FK_GLUSR_RATING_INFLU_PARAM_ID`, `GLUSR_RATING_MCAT_ID`, `is_rating_docs_available`, `influ_params_thumbsup_cnt`, `glusr_rating_pmcat_id`, `glusr_rating_cat_id` — [`UserSupplierRatingModel.go:597`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSupplierRatingModel.go) |
| `GLUSR_RATING_DETAILS` | meshpg | Har influence-parameter (Response/Quality/Delivery) pe thumbs up/down ka individual vote | `FK_GLUSR_RATING_ID`, `GLUSR_RATING_VALUE` (1=up, 0=down), `FK_GLUSR_SUPP_ID`, `FK_GLUSR_RATING_INFLU_PARAM_ID` — [`UserSupplierRatingModel.go:635`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSupplierRatingModel.go) |
| `GLUSR_RATING_DOCS` | meshpg | Rating ke saath attach ki hui photos | `FK_GLUSR_RATING_ID`, `DOC_REVIEWED_STATUS`, `GLUSR_RATING_DOC_ID`, `DOC_PATH`, `IMG_DOC_500X500`, `IMG_DOC_125X125` — [`UserSupplierRatingModel.go:676`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSupplierRatingModel.go) |
| `GLUSR_RATING_AGGREGATE` | meshpg | Har supplier ka **rolled-up summary** — full-recompute pattern (dekho GST tech doc jaisa hi) | `FK_GLUSR_ID`, `AVG_RATING`, `ONE_STAR_CNT`...`FIVE_STAR_CNT`, `INFLU_PARAMS_CNT` (JSON), `GLUSR_RATING_AGGREGATE_MOD_DATE` — [`USER_RATING_AGGREGATE.go:167-240`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_AGGREGATE.go) *(already fully documented in a previous conversation — full recompute on every trigger, filters `display_status > -1`)* |
| `GLUSR_RATING_LOG` | meshpg | Ek per-rating log jo LMS-matchmaking-status bhi capture karta hai | `GLUSR_RATING_ID`, `FK_GLUSR_SUPPLIER_ID`, `GLUSR_RATING_DATE`, `LMS_MATCHMAKING_STATUS_FLAG` — [`USER_RATING_AGGREGATE.go:255-262`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_AGGREGATE.go) (`dbActionRatingsLog`, sirf `ACTION=="INSERT"` ke liye trigger hota hai, LMS API call karta hai) |
| `GLCAT_MCAT_TO_MCAT`, `GLCAT_CAT_TO_MCAT` | meshpg (read-only, subquery) | Category-hierarchy lookup — rating insert ke waqt hi `pmcat_id`/`cat_id` derive karne ke liye, **subquery ke andar hi** (extra round-trip nahi) | Referenced inline in the `GLUSR_RATING` INSERT statement — [`UserSupplierRatingModel.go:597`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSupplierRatingModel.go) |
| `GLUSR_CONTACTBOOK_MAPPING` | meshpg/buyerprofilepg | Matchmaking check yahan se padhta/likhta hai — confirm karta hai buyer-supplier genuinely connected the | Dekho [`buyer_seller_discovery_matching_read_write_picture.md`](../buyer_seller_discovery_matching_read_write_picture.md) — matchmaking check yahi table use karta hai |

---

## 4. Business Rules & Validation (code se)

1. **`RATING_VAL` sirf `1`-`5` accept hota hai** — koi aur value reject.
   [`UserSupplierRatingController.go:125`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSupplierRatingController.go)
2. **`BUYER_ID != SUPPLIER_ID`** — khud ko rate nahi kar sakte.
   [`UserSupplierRatingController.go:127`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSupplierRatingController.go)
3. **`DISPLAY_STATUS` hamesha `-2` se shuru hoti hai insert pe**, request mein jo bhi value
   di ho usse ignore kar diya jaata hai — yeh ek server-side enforced default hai, client
   isse override nahi kar sakta.
   [`UserSupplierRatingModel.go:476`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSupplierRatingModel.go)
4. **Existing-relationship detection**: insert se pehle ek `COUNT(*)` query check karti hai
   ki yeh buyer-supplier pair pehle bhi rate ho chuka hai ya nahi — result LMS ko bheje jaane
   wale packet mein `insertion_type` (`I`=naya, `U`=existing) set karta hai.
   [`UserSupplierRatingModel.go:497-520`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSupplierRatingModel.go)
5. **Influence-params ka encoding thoda complex hai**: request array/map/string/nil kisi bhi
   format mein aa sakta hai, sabko ek normalized `{"0": thumbsDownIds, "1": thumbsUpIds}`
   shape mein convert kiya jaata hai. Agar dono khali hon aur ek single
   `GLUSR_RATING_INFLU_PARAM_ID` diya ho, toh star-rating ke basis pe (< 3 = thumbs-down,
   >= 3 = thumbs-up) auto-classify hota hai.
   [`UserSupplierRatingModel.go:522-591`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSupplierRatingModel.go)
6. **`influ_params_thumbsup_cnt` column ek precomputed summary hai per-rating** — kitne
   thumbs-up diye gaye, taaki `GLUSR_RATING_AGGREGATE` consumer ko har baar
   `GLUSR_RATING_DETAILS` join na karna pade.
7. **Rating photos optional hain, aur unki count `GLUSR_RATING_IMGS` se `IMG_ID` present hone
   par hi count hoti hai** — empty/missing `IMG_ID` waali entries silently skip hoti hain.
   [`UserSupplierRatingModel.go:681-685`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSupplierRatingModel.go)
8. **Update path (`UPDATE_FLAG=U`) alag mandatory-fields validation follow karta hai** —
   `RATING_ID`, `VALIDATION_KEY`, `UPDATEDBY`, `UPDATEDUSING`, `IP`, `IP_COUNTRY` chahiye,
   `BUYER_ID`/`RATING_VAL`/`RATING_SOURCE` nahi (kyunki yeh already-existing record pe action
   hai, naya data nahi).
   [`UserSupplierRatingController.go:119-124`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSupplierRatingController.go)
9. **`CALLEDFROM == "RATING_USEFULNESS_WRITE"` ek special branch hai** — yeh confirm karta
   hai ki `UserSupplierRatingController` **Rating Usefulness feature se bhi internally
   reused hota hai** (`HELPFUL_COUNT`/`ABUSE_COUNT` fields), lekin yeh alag scope hai (dekho
   header note) — yahan sirf iska existence flag kiya ja raha hai.
   [`UserSupplierRatingController.go:129`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSupplierRatingController.go)

---

## 5. RabbitMQ — Queues Used

| Queue / `SERVICENAME` | Publisher | Consumer | Purpose |
|---|---|---|---|
| `SUPPLIER_FEEDBACK` → routes to `user.supplierrating.<glid%20>` | `UserSupplierRatingController.go` (set at request-start) + `InsertRating` (final push) | *(generic comp.sync-style fan-out — no single dedicated consumer name found; likely feeds the moderation/enrichment chain below via further internal routing)* | Primary "a new rating was inserted" event — carries the full `GLUSR_RATING` row **plus an embedded `TO_LMS` sub-packet** for Lead-Management sync, all in one message |
| `USER_RATING_BANNED` | `UserSupplierRatingModel.go` (`UpdateRating` path, conditional) | [`USER_SUPPLIER_RATING_BANNED.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SUPPLIER_RATING_BANNED.go) | Content-moderation trigger (abuse/PII scan via CBS+ML) |
| `SUPPLIER_RATING_NEW` (seen twice in `UpdateRating`, different branches) | `UserSupplierRatingModel.go` | *(routes to `user.ratingnotification.*` per the write-API's `serviceToQueueMap`)* | Notification-adjacent event |
| `NEW_COMPANY` | `UserSupplierRatingModel.go` (`UpdateRating`) | *(routes to `user.rating_aggregate` per `serviceToQueueMap`)* | **[INFERRED]** likely a trigger that also feeds `USER_RATING_AGGREGATE` — naming suggests a "new company relationship" event, exact condition not fully traced in this pass |
| `USER_RATING_AGGREGATE` | Various (rating lifecycle events) | [`USER_RATING_AGGREGATE.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_AGGREGATE.go) | Full recompute of the supplier's star-count/influence-param summary (see §3, `GLUSR_RATING_AGGREGATE`) — **note from a prior investigation in this project**: `USER_RATING_BS_MATCHMAKING` does NOT push here when it disables a rating, which is a known staleness gap (see [`full_read_write_picture.md`](../full_read_write_picture.md)) |
| `USER_RATING_BS_MATCHMAKING` | *(publisher not re-traced in this pass — see prior finding)* | [`USER_RATING_BS_MATCHMAKING.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_BS_MATCHMAKING.go) | Verifies buyer-supplier connection is real; sets `display_status = -1` if not found |
| `USER_RATING_ENRICHMENT` | *(not re-traced)* | `USER_RATING_ENRICHMENT` worker | Product/category context enrichment (pulls from LMS) |
| `USER_RATING_NOTIFICATION` / `_NOTIFICATION_PG` | *(not re-traced)* | Respective workers | Notifies supplier of the new rating |
| `USER_RATING_ARCHIVE` | *(not re-traced)* | `USER_RATING_ARCHIVE` worker | Archives older ratings for the same buyer-supplier pair |
| `USER_RATING_SELLER_RISK` | *(not re-traced)* | `USER_RATING_SELLER_RISK` worker | Feeds seller risk-scoring |
| `USER_SUPPLIER_RATING_TO_LMS` | *(not re-traced — separate from the inline `TO_LMS` packet in `SUPPLIER_FEEDBACK`)* | `USER_SUPPLIER_RATING_TO_LMS` worker | A second, dedicated LMS-sync path — relationship to the inline `TO_LMS` packet (§ above) not fully clarified in this pass, worth confirming if these are redundant or serve different LMS endpoints |

**Overall pattern**: is domain mein bhi RabbitMQ hi primary messaging hai, jaisa GST aur
Privacy Setting domains mein tha.

---

## 6. Kafka

**Rating (star-rating) domain mein koi direct Kafka usage nahi mila** insert/update write
path mein. (Note: iss se related, lekin scope se bahar, `USER_BS_MATCHMAKING` — matchmaking
ka ek **broader**, non-rating-specific version — Kafka-driven hai per
`buyer_seller_discovery_matching_read_write_picture.md`; lekin rating-specific
`USER_RATING_BS_MATCHMAKING` consumer khud RabbitMQ pe hai, confirmed via router
registration.)

---

## 7. Redis

**Koi Redis usage nahi mila** rating write ya read path mein iss pass mein. `GET
/supplierrating` har baar live DB hit karta hai (`GLUSR_RATING` + `GLUSR_RATING_AGGREGATE`
join). Yeh ek high-read endpoint hai (supplier profile page baar-baar load hoti hai) — dekho
§9 Optimization Scope.

---

## 8. End-to-End Technical Flows

### Flow A — Naya rating submit (insert)

```
Buyer
    │
    ▼
[API — write]  POST /supplierrating (no UPDATE_FLAG, or UPDATE_FLAG != "U")
    │  UserSupplierRatingController.go
    │  1. Mandatory-field validation (BUYER_ID/SUPPLIER_ID/RATING_VAL/RATING_SOURCE/...)
    │  2. RATING_VAL 1-5 check, BUYER_ID != SUPPLIER_ID check
    │  3. Gateway/modid validation
    │  4. ValidationSupplierRating() — field-level validation
    ▼
[DB]  SELECT COUNT(*) FROM GLUSR_RATING (existing-relationship check → lms_flag I/U)
    ▼
[DB]  INSERT INTO GLUSR_RATING (DISPLAY_STATUS forced to -2)
    │  — subqueries embedded for pmcat_id/cat_id (no extra round-trip)
    │  RETURNING GLUSR_RATING_ID
    ▼
[DB — conditional]  INSERT INTO GLUSR_RATING_DETAILS (thumbs up/down per influence param)
[DB — conditional]  INSERT INTO GLUSR_RATING_DOCS (photos)
    ▼
[RabbitMQ publish]  SERVICENAME=SUPPLIER_FEEDBACK
    │  carries full GLUSR_RATING row + embedded TO_LMS sub-packet
    ▼
[CONSUME chain, async — buyer doesn't wait]
    ├─ USER_SUPPLIER_RATING_BANNED  → content moderation (CBS+ML+PII)
    ├─ USER_RATING_BS_MATCHMAKING   → verify real buyer-supplier connection
    ├─ USER_RATING_ENRICHMENT       → product/category context (LMS)
    ├─ USER_RATING_AGGREGATE        → recompute supplier's star-summary
    ├─ USER_RATING_NOTIFICATION     → notify supplier
    └─ USER_SUPPLIER_RATING_TO_LMS  → Lead-Management sync (dedicated path)
```

### Flow B — Update / admin-action path

```
[API — write]  POST /supplierrating (UPDATE_FLAG="U")
    │  Different mandatory-fields set (RATING_ID/UPDATEDBY/VALIDATION_KEY/...)
    ▼
UserModels.UpdateRating()
    │  UPDATE GLUSR_RATING SET ...
    │  Conditionally pushes ONE OF: NEW_COMPANY / SUPPLIER_RATING_NEW / USER_RATING_BANNED
    │  (exact branching condition not fully re-traced in this pass — see §5)
```

### Flow C — Supplier reply

```
Supplier
    │
    ▼
[API]  POST /supplierrating (UPDATE_FLAG="U", SUPPLIER_COMMENTS present)
    │  Same UpdateRating() path as Flow B — reply is just another update
```

### Flow D — Archival

```
[Trigger: some rating-lifecycle event for a buyer-supplier pair]
    │
    ▼
USER_RATING_ARCHIVE consumer
    │  (exact query not re-traced in this pass)
    ▼
Older ratings for the same pair get archived — only latest stays "live"
```

### Flow E — Read (profile page)

```
Buyer / App
    │
    ▼
[API — read]  GET /supplierrating
    │  SupplierRatingController / SupplierRatingVersionsController
    ▼
[DB]  JOIN GLUSR_RATING + GLUSR_RATING_AGGREGATE (+ GLUSR_USR for showroom/custtype)
    │  (already documented in full_read_write_picture.md — this is the same
    │   query family investigated in the earlier "influential param" ticket)
    ▼
Response: star average, per-star counts, individual rating list, influence-param %
```

---

## 9. Optimization Scope — DB Response-Time Contribution

Rating domain mein bhi DB hi biggest response-time contributor hai — is baar mainly **query
count per request** ki wajah se, na ki multi-DB fan-out (GST jitna cross-database nahi hai).

### High-impact

1. **`InsertRating` ek hi HTTP request mein sequentially up to 4 queries chalata hai**
   (existing-check → main insert → influence-details insert → docs insert), sab ek hi
   `meshpg` connection pe, **par sequential hain, parallel nahi** — jabki
   `GLUSR_RATING_DETAILS` insert aur `GLUSR_RATING_DOCS` insert ek doosre pe depend nahi
   karte (dono `RATING_ID` chahiye jo query5 se aata hai, lekin uske baad yeh do independent
   hain). **Concrete fix**: query6 (influ-details) aur query7 (docs) ko goroutines mein
   parallel chalao — GST/Privacy-Setting docs mein establish kiya hua pattern yahan bhi
   directly copy ho sakta hai.
2. **`GLUSR_RATING_AGGREGATE` full-recompute-on-every-trigger pattern** (already documented
   in a prior conversation, [`full_read_write_picture.md`](../full_read_write_picture.md))
   — poore supplier ke saare ratings ka full table scan hota hai har single naye rating pe.
   High-rating-volume suppliers ke liye yeh scale nahi karega achi tarah — GST tech doc mein
   suggested `INSERT...RETURNING`-jaisa incremental-update approach yahan bhi relevant hai
   (increment/decrement counters instead of full rebuild).
3. **`USER_RATING_BS_MATCHMAKING` disable karne pe `USER_RATING_AGGREGATE` ko notify nahi
   karta** (confirmed in a prior investigation this session) — matlab ek disabled/fake rating
   silently stale aggregate chhod sakta hai jab tak koi aur unrelated rating event trigger
   na ho. Yeh sirf ek correctness issue nahi, ek **response-accuracy/optimization dono** hai
   — stale data serve karna bhi ek "wasted correct-looking but wrong" response hai.

### Medium-impact

4. **`GET /supplierrating` pe koi caching nahi hai** — supplier profile page (high-traffic,
   low-change-frequency data — ratings don't change every second) is a natural
   cache-aside candidate, jaisa GST doc mein suggest kiya tha GST-read endpoints ke liye.
   Short TTL (2-5 min) bhi meaningful load reduce kar sakta hai high-traffic suppliers ke
   liye.
5. **Category-hierarchy subqueries** (`GLCAT_MCAT_TO_MCAT`, `GLCAT_CAT_TO_MCAT`) already
   smartly embedded hain insert-query ke andar (ek positive pattern — extra round-trip nahi
   liya), lekin agar yeh tables bade hain aur properly indexed nahi hain
   `FK_CHILD_MCAT_ID`/`FK_GLCAT_MCAT_ID` pe, yeh insert ko slow kar sakta hai — index
   verification worth karna.

### Low-impact / good practice already present

6. **`influ_params_thumbsup_cnt` precomputed column** (§4, point 6) already ek achi
   optimization hai — aggregate-consumer ko har rating ke liye `GLUSR_RATING_DETAILS` join
   nahi karna padta, precomputed count use kar sakta hai jahan possible ho.

---

## 10. Full Flow Diagrams (Lucid, icon-based)

**[Poora Rating flowchart yahan dekho](https://lucid.app/lucidchart/d2cb7aa3-79ae-4702-9c25-082d3eaa3150/edit)**

| Page | Content |
|---|---|
| **0. Superset — All Flows** | Ek page pe poora Rating domain |
| **A. New Rating Insert** | Section 8 Flow A — 4 sequential DB queries, RabbitMQ, 6-way consumer fan-out |
| **B/C. Update & Reply** | Section 8 Flow B/C — conditional queue push branches |
| **D. Archival** | Section 8 Flow D |
| **E. Rating Read** | Section 8 Flow E — join pattern for profile-page display |

---

## 11. Edge Cases & Gotchas (technical POV)

1. **Update-path queue-push branching** (§5, §8 Flow B) — `NEW_COMPANY` vs
   `SUPPLIER_RATING_NEW` vs `USER_RATING_BANNED`, exact condition-per-branch nahi fully
   re-traced iss pass mein. Agar kisi specific update scenario mein wrong consumer trigger
   ho raha lage, yeh function line-by-line dobara padhna padega.
2. **Two LMS-sync paths exist** — inline `TO_LMS` packet (Flow A ke andar) aur dedicated
   `USER_SUPPLIER_RATING_TO_LMS` consumer — dono redundant hain ya alag LMS endpoints serve
   karte hain, confirm karo.
3. **Aggregate staleness gap** (§9, point 3) — already known issue is codebase mein, is doc
   mein cross-referenced.
4. **`CALLEDFROM == "RATING_USEFULNESS_WRITE"`** confirm karta hai yeh hi controller Rating
   Usefulness feature se bhi internally reuse hota hai — jab uska KT folder banega, iska
   cross-reference zaroor add karna.

---

## 12. Open Questions

1. `NEW_COMPANY` / `SUPPLIER_RATING_NEW` queue-push ka exact trigger-condition kya hai
   `UpdateRating()` ke andar? Iss pass mein sirf existence confirm hui, condition nahi.
2. `SUPPLIER_FEEDBACK` → `user.supplierrating.<glid%20>` se aage konsa consumer ismein se
   moderation-chain trigger karta hai? Exact routing-to-first-consumer link iss pass mein
   trace nahi hua.
3. Inline `TO_LMS` packet vs dedicated `USER_SUPPLIER_RATING_TO_LMS` consumer — redundant
   hain ya alag purpose?
4. `USER_RATING_ENRICHMENT`, `USER_RATING_NOTIFICATION`, `USER_RATING_ARCHIVE`,
   `USER_RATING_SELLER_RISK` workers ka full code iss pass mein re-read nahi hua (pehle ki
   conversation mein summary-level cover hue the) — agar deep technical detail chahiye in
   par, ek follow-up pass zaroori hoga.
5. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`Rating_Business_Doc.md`](./Rating_Business_Doc.md) — same flows, product perspective
- [`../ratings_reviews_product_overview.md`](../ratings_reviews_product_overview.md) —
  bigger Ratings & Reviews story (Social Reviews samet)
- [`../full_read_write_picture.md`](../full_read_write_picture.md) — supplier-rating ka
  poora worked-example trace, incl. why `GLUSR_RATING_AGGREGATE` can go stale
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md),
  [`../Privacy Setting KT/Privacy_Setting_Technical_Doc.md`](../Privacy%20Setting%20KT/Privacy_Setting_Technical_Doc.md) —
  similar-shape domain docs, useful comparison for the parallelization/caching optimization
  patterns referenced in §9

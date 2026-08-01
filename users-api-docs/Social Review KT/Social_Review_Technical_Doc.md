# Social Review — Technical Doc (Code-Level Deep Dive)

**Scope note**: yeh doc sirf **Social Reviews** (Google/Facebook-linked external reviews)
cover karta hai — **Rating** (native supplier star-rating) alag system hai, dekho
[`../Rating KT/Rating_Technical_Doc.md`](../Rating%20KT/Rating_Technical_Doc.md).

Business/product perspective ke liye
[`Social_Review_Business_Doc.md`](./Social_Review_Business_Doc.md) dekho.

**Repos**: `service-api-go-production` (write), `users-api-go-production` (read).

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Account link/update/deactivate | write | [`SocialReviewAccountController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/SocialReviewAccountController.go), [`SocialReviewAccountModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/SocialReviewAccountModel.go) |
| Content (individual reviews) sync | write | [`SocialReviewContentController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/SocialReviewContentController.go), [`SocialReviewContentModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/SocialReviewContentModel.go) |
| Read (display) | read | [`SocialReviewReadModel.go`](../internal/models/users/SocialReviewReadModel.go), controller `GetSocialReviews` (file not individually located in this pass, route confirmed) |

**Koi consumer/queue file nahi mila** iss feature ke liye — poori tarah synchronous,
request/response-based hai (Blocking, TrustSeal jaisa hi simple architecture).

---

## 2. Routes (confirmed)

| Method | Path | Repo | Controller |
|---|---|---|---|
| POST | `/socialreviews/account` | write | `SocialReviewAccountController` |
| POST | `/socialreviews/content` | write | `SocialReviewContentController` |
| GET | `/socialreviews/*params` | read | `GetSocialReviews` |

---

## 3. Data Model — Tables

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `glusr_social_review` | meshpg (inferred, same connection pattern as domain) | **Account-level record** — ek row per (supplier, social-platform, location) combination | `fk_glusr_usr_id`, `fk_gl_social_media_id`, `social_location_id`, `social_account_id`, `total_rating_count`, `social_average_rating`, `is_organic`, `social_refresh_token`, `last_update_date` — unique constraint on `(fk_glusr_usr_id, fk_gl_social_media_id, social_location_id)` — [`SocialReviewAccountModel.go:38,46`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/SocialReviewAccountModel.go) |
| `glusr_social_review_detail` | meshpg | **Individual review content** — ek row per actual review | `fk_glusr_social_review_id` (FK to `glusr_social_review`), `social_platform_review_id`, star-rating, display-status, reply fields — unique constraint on `(fk_glusr_social_review_id, social_platform_review_id)` — [`SocialReviewContentModel.go:96,102`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/SocialReviewContentModel.go) |

**Relationship**: `glusr_social_review_detail.fk_glusr_social_review_id` →
`glusr_social_review` — matlab content hamesha ek existing account-record se linked hona
chahiye (foreign-key dependent), Trust Seal Company/TrustSeal jaisa hi parent-child pattern.

---

## 4. Business Rules & Validation (code se)

1. **Account API `FLAG` param se Insert/Update distinguish karta hai** (`"I"` for Insert,
   `"U"` for Update) — explicit action-flag pattern, GST/Rating domains mein bhi common.
   [`SocialReviewAccountController.go:45`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/SocialReviewAccountController.go)
2. **`GLUSR_ID` numeric-hona mandatory hai**, dono controllers mein independently validate
   hota hai.
   [`SocialReviewAccountController.go:68-70`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/SocialReviewAccountController.go)
3. **Account-write ek genuine `UPSERT` hai** (`ON CONFLICT (fk_glusr_usr_id,
   fk_gl_social_media_id, social_location_id) DO UPDATE`) — Blocking-module jaisa hi clean,
   single-round-trip pattern.
   [`SocialReviewAccountModel.go:38,46`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/SocialReviewAccountModel.go)
4. **"Deactivate" ek alag, explicit UPDATE query hai** (soft-delete, not row-delete) — Business
   Doc §4 point 4 ka technical-confirmation.
   [`SocialReviewAccountModel.go:79`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/SocialReviewAccountModel.go)
5. **Content-sync ek `DETAILS` array leta hai, max 20 items per request** (`maxBatchSize =
   20`) — bade content-syncs client-side chunk karne padenge.
   [`SocialReviewContentController.go:13`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/SocialReviewContentController.go)
6. **Har `DETAILS` item independently validate hota hai**, ek array-index-specific error
   message ke saath (`"Item 3 Error: ..."`) — partial-failure ka pata chalta hai kaunsa item
   fail hua, poori request generic-fail nahi hoti.
   [`SocialReviewContentModel.go:13-31`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/SocialReviewContentModel.go)
7. **`STAR_RATING` aur `REVIEW_DISPLAY_STATUS` mandatory hain per-item**, saath 4 date-fields
   (`REVIEW_INSERT_DATE`, `REVIEW_UPDATE_DATE`, `REPLY_INSERT_DATE`, `REPLY_UPDATE_DATE`) —
   matlab supplier-reply ka apna alag insert/update timestamp bhi track hota hai, review
   khud se independently.
   [`SocialReviewContentModel.go:29-40`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/SocialReviewContentModel.go)
8. **Content-write bhi UPSERT hai** (`ON CONFLICT (fk_glusr_social_review_id,
   social_platform_review_id) DO UPDATE`) — same platform-review-ID dobara sync ho toh
   update, duplicate insert nahi.
   [`SocialReviewContentModel.go:96,102`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/SocialReviewContentModel.go)
9. **Read-model mein ek `ROW_NUMBER() OVER (...)` windowing query mili** — likely "latest N
   reviews per platform" jaisa pagination/dedup pattern, exact partition-logic iss pass mein
   pura decode nahi hui.
   [`SocialReviewReadModel.go:198`](../internal/models/users/SocialReviewReadModel.go)

---

## 5. RabbitMQ / Kafka / Redis

**Koi RabbitMQ, Kafka, ya Redis usage nahi mila** iss poore feature mein — purely synchronous
DB writes/reads, koi async fan-out ya moderation-queue nahi (jaisa Business Doc mein bhi
predicted tha: "no content-moderation" kyunki content already-moderated external source se
aata hai).

---

## 6. End-to-End Technical Flows

### Flow A — Account link/update

```
Supplier (or an external sync job on their behalf)
    │
    ▼
[API — write]  POST /socialreviews/account  {GLUSR_ID, FLAG, SOCIAL_MEDIA_ID, ...}
    │  SocialReviewAccountController.go
    │  Mandatory-field + numeric-GLUSR_ID validation
    ▼
[DB]  INSERT INTO glusr_social_review ... ON CONFLICT (...) DO UPDATE
    │  (or, for deactivation: dedicated UPDATE query)
    ▼
Response
```

### Flow B — Content sync (batch)

```
External sync job (outside these 3 repos — likely a Google/Facebook API integration)
    │
    ▼
[API — write]  POST /socialreviews/content  {GLUSR_ID, SOCIAL_MEDIA_ID, DETAILS: [up to 20 items]}
    │  SocialReviewContentController.go
    │  Per-item validation (STAR_RATING, REVIEW_DISPLAY_STATUS mandatory, date fields checked)
    ▼
[DB]  For each item: INSERT INTO glusr_social_review_detail ... ON CONFLICT (...) DO UPDATE
    ▼
Response
```

### Flow C — Read (profile display)

```
Buyer / App
    │
    ▼
[API — read]  GET /socialreviews
    │  GetSocialReviews → SocialReviewReadModel.go
    ▼
[DB]  SELECT (multiple query variants — single-platform, all-platforms, paginated with
      ROW_NUMBER() windowing)
    ▼
Response: reviews list + account-level summary (average rating, total count)
```

---

## 7. Optimization Scope — DB Response-Time Contribution

### Medium-impact

1. **Content-sync batch-of-20 loop likely runs sequential UPSERTs, ek query per item** —
   agar `DETAILS` array poora 20 items ka ho, yeh 20 sequential round-trips ho sakte hain
   (exact loop-implementation iss pass mein poori tarah trace nahi hui). Postgres ka
   multi-row `INSERT ... VALUES (...), (...), ... ON CONFLICT DO UPDATE` single-statement
   se yeh 20 round-trips ek mein convert ho sakte hain — GST-domain ke `GST_TO_HSN_MAPPING`
   batch-insert se bhi is pattern ka comparison ho sakta hai.
2. **Read-side `ROW_NUMBER()` windowing query** — agar underlying `glusr_social_review_detail`
   table bahut badi ho (popular suppliers, hundreds of reviews per platform), window-function
   ka cost badh sakta hai bina proper indexing ke `fk_glusr_social_review_id` pe.

### Low-impact / good practice already present

3. **Dono write-paths already clean UPSERT-based hain** (single-round-trip per record) —
   Blocking-module jaisa hi simple, efficient design.
4. **Per-item validation with index-specific errors** (§4, point 6) — achi UX/debugging
   practice, batch-failure ko granular banati hai.

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora Social Review flowchart yahan dekho](https://lucid.app/lucidchart/e2ec37e2-2eb9-4c8f-9a4d-c1d07910889c/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **Content-sync loop ka exact per-item-vs-batch query-pattern confirm nahi hua** (§7,
   point 1) — worth verify karna performance-tuning se pehle.
2. **`ROW_NUMBER()` windowing query ka indexing-status unknown** (§7, point 2).
3. **External sync-job (jo Google/Facebook se actual data laata hai) in teeno repos ke bahar
   hai** — agar "meri reviews out-of-date hain" jaisa ticket aaye, root-cause investigation
   in repos ke bahar ke system tak jaani padegi.

---

## 10. Open Questions

1. Content-sync batch (`DETAILS` array) DB mein kaise likhi jaati hai — single multi-row
   statement ya per-item loop?
2. `ROW_NUMBER()` windowing query ka exact partition/order-by logic aur uska indexing-status?
3. External Google/Facebook sync-job (in repos ke bahar) kitni frequently chalta hai?
4. `is_organic` flag ka exact business-usage — kahin filter/display-logic mein consult hota
   hai kya?
5. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`Social_Review_Business_Doc.md`](./Social_Review_Business_Doc.md) — product perspective
- [`../Rating KT/Rating_Technical_Doc.md`](../Rating%20KT/Rating_Technical_Doc.md) — related
  but independent native-rating system
- [`../ratings_reviews_product_overview.md`](../ratings_reviews_product_overview.md) —
  original combined product-overview (Rating + Social Review together)

# Social Review — Technical Doc (Code-Level Deep Dive)

**Scope note**: yeh doc sirf **Social Reviews** (Google/Facebook-linked external reviews)
cover karta hai — **Rating** (native supplier star-rating) alag system hai, dekho
[`../Rating KT/Rating_Technical_Doc.md`](../Rating%20KT/Rating_Technical_Doc.md); **Social
Contacts** (supplier ke social-media profile-links, jaise Instagram handle) bhi alag system
hai, dekho [`../Social Contacts KT/Social_Contacts_Technical_Doc.md`](../Social%20Contacts%20KT/Social_Contacts_Technical_Doc.md)
— dono naam se milte-julte hain lekin code mein bilkul alag files/tables hain.

Business/product perspective ke liye
[`Social_Review_Business_Doc.md`](./Social_Review_Business_Doc.md) dekho.

**Repos**: `service-api-go-production` (write), `users-api-go-production` (read). Is
review mein `user-temp-consumers-production` bhi explicitly grep kiya gaya — koi Social
Review-specific consumer/cron nahi mila (dekho section 5 aur 12).

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file:line diya
gaya hai). Jahan code se pura confirm nahi ho paaya, wahan **[INFERRED — confirm with
team]** likha hai.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Account link/update | write | [`SocialReviewAccountController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/SocialReviewAccountController.go), [`SocialReviewAccountModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/SocialReviewAccountModel.go) |
| Content (individual reviews) sync | write | [`SocialReviewContentController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/SocialReviewContentController.go), [`SocialReviewContentModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/SocialReviewContentModel.go) |
| Validation field-maps (both APIs) | write | [`UsersValidationMaps.go:2781-2819`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) — `SocialReviewAccountMap`, `SocialReviewContentMap` |
| Read (display) — **previously unlocated controller, found in this pass** | read | [`SocialReviewReadController.go`](../internal/controllers/UsersControllers/SocialReviewReadController.go) (function `GetSocialReviews`), [`SocialReviewReadModel.go`](../internal/models/users/SocialReviewReadModel.go) |

**Related but distinct** (do not confuse): `UserSocialContactModel.go` /
`UserSocialContactsController.go` (write) and `USER_SOCIAL_CONTACT.go` (consumer) belong to
the separate **Social Contacts** feature (supplier's social-media profile-link handles), not
Social Review. Confirmed by grepping `SocialReview`/`social_review` across all three repos —
these files never appear together.
[`user_consumers_reference.md` style repo-wide grep, run in this pass]

**No consumer, no cron file for Social Review found** in `user-temp-consumers-production` —
grepping the whole repo (`grep -ril "SOCIAL_REVIEW\|SocialReview\|social_review"`) returns
zero matches there. This actively re-verifies (not just repeats) the earlier doc's claim —
see section 5 for the full RabbitMQ/Kafka/Redis re-check.

---

## 2. Routes (confirmed from router files)

| Method | Path | Repo | Controller | Evidence |
|---|---|---|---|---|
| POST | `/socialreviews/account` | write | `SocialReviewAccountController` | [`router.go:287`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) |
| POST | `/socialreviews/content` | write | `SocialReviewContentController` | [`router.go:288`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) |
| GET | `/socialreviews/*params` | read | `UsersControllers.GetSocialReviews` | [`routerUsers.go:359`](../internal/api/users_router/routerUsers.go) |

---

## 3. Data Model — Tables

> **Verification note**: yeh sab table/column names Go code ke andar embedded SQL strings se
> liye gaye hain (har row ke saamne file:line diya hai). Live DB schema se pgAdmin pe
> cross-verify **nahi** kiya gaya hai.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `glusr_social_review` | meshpg (write side, `config.GetPGDbConnection("meshpg")`) / `mesh_pg` (read side, `config.GetPG("mesh_pg")`) — same logical Postgres, naming-convention difference across the two repos, consistent with how other domains in this codebase name the same pool differently per repo | **Account-level record** — ek row per (supplier, social-platform, location) combination | `glusr_social_review_id` (PK), `fk_glusr_usr_id`, `fk_gl_social_media_id`, `social_location_id`, `social_account_id`, `social_company_name/email/phone/address/website`, `total_rating_count`, `social_average_rating`, `social_data_source_is_organic`, `social_refresh_token` (write-only, never selected on read side — [`SocialReviewReadModel.go:26-27`](../internal/models/users/SocialReviewReadModel.go)), `social_review_display_status`, `modid`, `insert_date`, `last_update_date` — unique constraint (per code comment, not confirmed live) on `(fk_glusr_usr_id, fk_gl_social_media_id, social_location_id)` — [`SocialReviewAccountModel.go:20-24,46`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/SocialReviewAccountModel.go) |
| `glusr_social_review_detail` | meshpg | **Individual review content** — ek row per actual review | `glusr_social_review_detail_id` (PK), `fk_glusr_social_review_id` (FK to `glusr_social_review`), `fk_glusr_usr_id`, `social_platform_review_id`, `social_reviewer_name`, `social_star_rating`, `social_review_comment`, `social_review_insert_date`, `social_review_update_date`, `social_review_display_status`, `social_reply_comment`, `social_reply_insert_date`, `social_reply_update_date`, `social_reply_display_status` — unique constraint (per code comment) on `(fk_glusr_social_review_id, social_platform_review_id)` — [`SocialReviewContentModel.go:53-57,96,102`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/SocialReviewContentModel.go) |

**Relationship**: `glusr_social_review_detail.fk_glusr_social_review_id` →
`glusr_social_review.glusr_social_review_id` — content always attaches to a specific
location-row, and specifically to whichever location-row is currently
`social_review_display_status = 1` (the "active" one) for that GLUSR + platform combination
— see section 4, rule 3.

---

## 4. Decode-This-Value Section — Key Flags

| Field | Values seen in code | Meaning | Evidence |
|---|---|---|---|
| `FLAG` (Account API only) | `"I"` = Insert, `"U"` = Update | Explicit action-flag deciding `InsertSocialReviewAccount` vs `UpdateSocialReviewAccount` | [`SocialReviewAccountController.go:45,80-96,121-124`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/SocialReviewAccountController.go) |
| `social_review_display_status` (on `glusr_social_review`, i.e. account/location row) | `1` = active/displayed location for that platform; `0` = deactivated (superseded by a newer location connect) | Only one row per `(GLUSR_ID, SOCIAL_MEDIA_ID)` is ever `1` at a time — enforced transactionally, see section 6 Flow A | [`SocialReviewAccountModel.go:37,57,79-84`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/SocialReviewAccountModel.go) |
| `social_review_display_status` (on `glusr_social_review_detail`, i.e. content row) | Numeric, mandatory per-item on content sync (`REVIEW_DISPLAY_STATUS`) | Per-review display toggle — exact value semantics (e.g. is `0` "hidden by supplier," "flagged," something else) **[INFERRED — confirm with team]**; only confirmed fact is it's read filtered as `= 1` on the read side ([`SocialReviewReadModel.go:105,130,203`](../internal/models/users/SocialReviewReadModel.go)) |
| `social_data_source_is_organic` (`IS_ORGANIC`) | Numeric (0/1 presumed) | Business Doc's "organic vs boosted" flag; only ever stored/passed through, no branching logic on its value found anywhere in the three repos — **[INFERRED — confirm with team re: any downstream consumer of this flag; none found in these repos]** | [`SocialReviewAccountModel.go:41,55`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/SocialReviewAccountModel.go) |
| Gateway modid check `Gateway_v1([]string{"FLIPS", "MAPI"}, ...)` | Restricts both write APIs to callers presenting a `FLIPS` or `MAPI` gateway key | Standard shared-gateway validation pattern used across many domains in this codebase, not Social-Review-specific | [`SocialReviewAccountController.go:97`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/SocialReviewAccountController.go), [`SocialReviewContentController.go:87`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/SocialReviewContentController.go) |

---

## 5. RabbitMQ / Kafka / Redis — Explicitly Re-Verified

The prior shallow doc claimed "purely synchronous, no consumer/queue." This pass re-checked
that claim directly rather than repeating it:

**RabbitMQ**: grepped `PushToQueue|PubAPI|Requeue|rabbitmq|amqp` across all four
Social-Review source files (both controllers, both write models, the read model, the read
controller) — **zero matches**. No queue publish anywhere in this feature's write or read
path.

**Kafka**: grepped `kafka` (case-insensitive) across the same files, plus the entire
`user-temp-consumers-production` repo for `SocialReview`/`social_review`/`SOCIAL_REVIEW` —
**zero matches on both counts**. Unlike the GST domain (which has one Kafka exception,
`USER_GST_DETAILS_BULK`), Social Review has no async intake path of any kind in these three
repos.

**Redis**: grepped `redis` (case-insensitive) across the same four files — **zero matches**.
No caching layer on the read side either (`GetSocialReviews` hits Postgres directly every
call, [`SocialReviewReadModel.go:77`](../internal/models/users/SocialReviewReadModel.go)).

**Conclusion — claim holds up**: this feature is genuinely, fully synchronous
request/response across all three repos. There is no scheduled/async external-sync job
implemented inside `users-api-go-production`, `service-api-go-production`, or
`user-temp-consumers-production` — whatever mechanism actually pulls fresh reviews from
Google/Facebook (the Business Doc's "external integration/sync system") lives entirely
outside these repos and calls the two write APIs (`/socialreviews/account`,
`/socialreviews/content`) as an ordinary API client, the same way a supplier's own
seller-panel session would. This is a genuine open question (section 15) — its calling
frequency/cadence is invisible from this codebase.

---

## 6. End-to-End Technical Flows

### Flow A — Account link/update (insert path, `FLAG=I`)

```
Supplier (or an external sync job on their behalf)
    │
    ▼
[API — write]  POST /socialreviews/account  {GLUSR_ID, FLAG=I, VALIDATION_KEY,
                SOCIAL_MEDIA_ID, SOCIAL_ACCOUNT_ID, SOCIAL_LOCATION_ID,
                SOCIAL_REFRESH_TOKEN, TOTAL_RATING_COUNT, SOCIAL_AVERAGE_RATING,
                IS_ORGANIC, ...}
    │  SocialReviewAccountController.go
    │  Mandatory-field + numeric-GLUSR_ID/SOCIAL_MEDIA_ID checks, FLAG=I-specific
    │  mandatory fields, Gateway_v1([FLIPS,MAPI]) check, LengthAndTypeValidations_v3
    ▼
[DB — meshpg, single transaction, 2s timeout]  InsertSocialReviewAccount()
    │  1. INSERT INTO glusr_social_review ... ON CONFLICT (fk_glusr_usr_id,
    │     fk_gl_social_media_id, social_location_id) DO UPDATE ... RETURNING id
    │     (sets social_review_display_status = 1 for this location)
    │  2. UPDATE glusr_social_review SET social_review_display_status = 0
    │     WHERE fk_glusr_usr_id=$1 AND fk_gl_social_media_id=$2 AND id != $3
    │     (deactivates every OTHER location row for same GLUSR+platform)
    │  3. COMMIT (both steps atomic — either both happen or neither)
    ▼
Response {STATUS, CODE, MESSAGE, MASTER_ID}
```

### Flow B — Account update (`FLAG=U`)

```
Supplier/sync job
    │
    ▼
[API — write]  POST /socialreviews/account  {GLUSR_ID, FLAG=U, SOCIAL_LOCATION_ID, ...}
    │  SocialReviewAccountController.go
    ▼
[DB]  UpdateSocialReviewAccount() — dynamically builds
      UPDATE glusr_social_review SET <only-provided-columns>
      WHERE fk_glusr_usr_id=$n AND fk_gl_social_media_id=$n AND social_location_id=$n
      (sorted-key column iteration for deterministic query-plan caching)
    ▼
Response — ROWS_AFFECTED=0 if no matching row ("not found," not an error)
```

### Flow C — Content sync (batch, up to 20 items)

```
External sync job (outside these 3 repos)
    │
    ▼
[API — write]  POST /socialreviews/content  {GLUSR_ID, SOCIAL_MEDIA_ID,
                DETAILS: [up to 20 items]}
    │  SocialReviewContentController.go
    │  Per-item validation via ValidateSocialReviewContent(): type/length map,
    │  mandatory STAR_RATING + REVIEW_DISPLAY_STATUS, 4 date-field format checks
    │  — first failing item aborts the WHOLE batch with an index-specific message
    ▼
[DB — meshpg, single transaction, 10s timeout]  InsertSocialReviewContent()
    │  1. If SOCIAL_REVIEW_ID (master id) not given: SELECT glusr_social_review_id
    │     FROM glusr_social_review WHERE fk_glusr_usr_id=$1 AND fk_gl_social_media_id=$2
    │     AND social_review_display_status=1  (resolve the currently-ACTIVE location)
    │     -> fails the whole request if no active account exists ("Create Account first")
    │  2. PrepareContext once, then loop: ExecContext per item —
    │     INSERT INTO glusr_social_review_detail ... ON CONFLICT
    │     (fk_glusr_social_review_id, social_platform_review_id) DO UPDATE
    │  3. COMMIT — all-or-nothing across the whole batch
    ▼
Response {STATUS, CODE, MESSAGE}  (row count reported via ROW_CNT misc field)
```

### Flow D — Read (profile display)

```
Buyer / App
    │
    ▼
[API — read]  GET /socialreviews?GLUSR_ID=...&SOCIAL_MEDIA_ID=(optional)&LIMIT=&OFFSET=
    │  SocialReviewReadController.go → GetSocialReviews()
    │  MODID presence/validity check, numeric GLUSR_ID/SOCIAL_MEDIA_ID check,
    │  limit/offset clamped via ClampLimit/ClampOffset (default 20, max 100)
    ▼
[DB — mesh_pg]
    ├─ If SOCIAL_MEDIA_ID given → getSinglePlatformReviews():
    │     SELECT master account (display_status=1) for that platform
    │     + SELECT reviews (display_status=1) ORDER BY insert_date DESC
    │       LIMIT (limit+1) OFFSET offset  (fetch one extra row to compute HAS_MORE)
    │
    └─ If SOCIAL_MEDIA_ID omitted → getAllPlatformReviews():
          SELECT all active master accounts for GLUSR_ID
          + one query with ROW_NUMBER() OVER (PARTITION BY fk_glusr_social_review_id
            ORDER BY social_review_insert_date DESC) to grab top (limit+1) reviews
            PER platform in a single round-trip, then grouped in Go by master ID
    ▼
Response: {DATA: [{ACCOUNT: {...}, REVIEWS: [...], HAS_MORE: bool}, ...]}
(social_refresh_token is never selected/returned on this path)
```

---

## 7. Flow-wise DB & Table Usage Matrix

### Flow A — Account Insert (`FLAG=I`)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `glusr_social_review` | INSERT ... ON CONFLICT DO UPDATE ... RETURNING id | Upsert the connected location, mark it active |
| 2 | meshpg | `glusr_social_review` | UPDATE (deactivate siblings) | Enforce "only one active location per platform" rule |
| 3 | meshpg | (implicit) COMMIT | — | Both writes are one atomic transaction |

### Flow B — Account Update (`FLAG=U`)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `glusr_social_review` | UPDATE (dynamic column list) | Patch only the fields the caller sent, keyed on (GLUSR_ID, SOCIAL_MEDIA_ID, SOCIAL_LOCATION_ID) |

### Flow C — Content Sync (batch of up to 20)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `glusr_social_review` | SELECT (only if `SOCIAL_REVIEW_ID` not supplied) | Resolve which location-row is currently "active" for this GLUSR+platform, so reviews attach to the right master record |
| 2..N | meshpg | `glusr_social_review_detail` | INSERT ... ON CONFLICT DO UPDATE, once per DETAILS item (prepared statement reused, not re-parsed) | Upsert each review; duplicate platform-review-ID updates in place |
| N+1 | meshpg | (implicit) COMMIT | — | All items succeed or the whole batch rolls back |

### Flow D — Read, single platform (`SOCIAL_MEDIA_ID` given)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | mesh_pg | `glusr_social_review` | SELECT (WHERE display_status=1) | Fetch the one active account/location record for this platform |
| 2 | mesh_pg | `glusr_social_review_detail` | SELECT ... LIMIT (limit+1) OFFSET offset | Paginated reviews for that master, +1 row fetched to compute HAS_MORE without a separate COUNT query |

### Flow D — Read, all platforms (`SOCIAL_MEDIA_ID` omitted)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | mesh_pg | `glusr_social_review` | SELECT (WHERE display_status=1, all platforms) | Fetch every active platform-account for this supplier |
| 2 | mesh_pg | `glusr_social_review_detail` | SELECT with `ROW_NUMBER() OVER (PARTITION BY fk_glusr_social_review_id ...)` subquery | Grab top (limit+1) reviews PER platform in one single round-trip instead of one query per platform — a deliberate batching optimization already in place |

**Total round-trips per event**: Account-Insert = 2 (1 txn), Account-Update = 1,
Content-Sync = 1 + N (1 txn, N ≤ 20), Read-single-platform = 2, Read-all-platforms = 2
(regardless of how many platforms the supplier has linked — the windowing query keeps this
flat, which is notably better than the naive N+1 pattern GST's `UserGSTHSNMappingController`
falls into for its 5-query chain).

---

## 8. Optimization Scope — DB Response-Time Contribution

### Medium-impact

1. **Content-sync batch is a per-item loop with N sequential `ExecContext` calls inside one
   transaction** (up to 20 round-trips to the same connection for one API call), rather than
   a single multi-row `INSERT ... VALUES (...), (...), ... ON CONFLICT DO UPDATE` statement.
   The loop does reuse a single prepared statement (`stmt.PrepareContext` once,
   `stmt.ExecContext` N times — [`SocialReviewContentModel.go:117-144`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/SocialReviewContentModel.go)),
   which avoids re-parsing SQL each time, but each `Exec` is still a separate network
   round-trip to Postgres. A single multi-row upsert would collapse this to one round-trip.
   This directly resolves the prior doc's Open Question #1 ("is it per-item or batch?") —
   it is confirmed per-item, with the prepared-statement optimization already applied.

2. **Read-side `ROW_NUMBER()` windowing query** on `getAllPlatformReviews`
   ([`SocialReviewReadModel.go:198`](../internal/models/users/SocialReviewReadModel.go)) —
   if a popular supplier accumulates hundreds of reviews per platform across many platforms,
   this window-function's cost scales with total row count scanned before partitioning.
   Whether `fk_glusr_social_review_id` (the partition key) is indexed is not confirmed from
   code — **[INFERRED — confirm with team / DBA before assuming this is fast at scale]**.

3. **No caching layer on any of the three read paths** (single-platform, all-platforms,
   or the account-only lookup) — every `GET /socialreviews` call hits `mesh_pg` live. Given
   that reviews and account summaries change relatively infrequently per supplier (bounded by
   how often the external sync job calls the write APIs — likely not more than daily per
   section 5's Open Questions), this is a reasonable Redis cache-aside candidate, following
   the same pattern flagged as a gap in the GST domain
   ([`GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) section 12, point 4).

### Low-impact / good practice already present

4. **Both write paths are already clean, single-transaction UPSERTs** — Account-Insert (2
   round-trips, transactional) and per-item Content-Sync (prepared-statement reuse) are
   simple and efficient relative to GST's multi-database fan-out.
2. **`getAllPlatformReviews` already avoids the N+1 trap** — a naive implementation would
   run one query per linked platform; this code fetches all platforms' top-N reviews in a
   single windowed query (section 7). This is a positive finding worth preserving in any
   refactor.
5. **Per-item validation with index-specific errors** (`"Item 3 Error: ..."`) is good
   debugging/UX practice — batch failures are traceable to the exact offending item.

---

## 9. Cron Inventory

**None found.** Grepped `SocialReview|social_review|SOCIAL_REVIEW` across the entirety of
`user-temp-consumers-production` (where GST's daily cron and all its consumers live) —
zero matches. Also checked for any `cron`-named file under
`service-api-go-production/.../crons/` referencing social-review tables — none found. This
was checked, not assumed: the absence is a confirmed finding, not an oversight.

---

## 10. Edge Cases & Gotchas (technical POV)

1. **The "one active location per platform" deactivation (Flow A step 2) runs inside the
   same transaction as the insert** — if that UPDATE ever failed independently of the
   INSERT, the whole transaction rolls back including the new insert, so there's no
   window where two locations end up simultaneously active. Good; but it also means a
   large number of prior locations for a heavy multi-branch supplier all get touched
   (an UPDATE with no row-count cap) on every single new-location connect — worth watching
   if any supplier has an unusually large number of historical locations.
2. **Content sync's active-location lookup (Flow C step 1) is a live SELECT on every
   request that doesn't pass `SOCIAL_REVIEW_ID` explicitly** — if the external sync job
   never passes this optional field, every content-sync call pays this extra round-trip.
   Whether the sync job actually supplies it is unknown from this codebase —
   **[INFERRED — confirm with team]**.
3. **`ROW_NUMBER()` windowing query's indexing status is unconfirmed** (section 8, point 2).
4. **External sync-job (jo Google/Facebook se actual data laata hai) is entirely outside
   these three repos** — if a "my reviews are stale" ticket comes in, root-cause
   investigation has to go outside this codebase entirely; there is no cron, consumer, or
   scheduled job here to inspect.
5. **`social_refresh_token` hygiene**: it's stored plaintext in `glusr_social_review`
   (no encryption function visible in the INSERT/UPDATE code) and is explicitly excluded
   from the read model with a code comment flagging it as write-only for security reasons
   ([`SocialReviewReadModel.go:26-27`](../internal/models/users/SocialReviewReadModel.go))
   — worth confirming with a security review whether at-rest encryption exists at the DB
   layer, since it isn't visible at the application layer.

---

## 11. Open Questions

1. External Google/Facebook sync-job (outside these three repos) — what triggers it, how
   frequently does it call `/socialreviews/account` and `/socialreviews/content`, and does
   it always pass `SOCIAL_REVIEW_ID` on content-sync calls (section 10, point 2)?
2. Exact semantics of `social_review_display_status` on the *content* (detail) table beyond
   "must equal 1 to be shown" — is `0` supplier-hidden, platform-flagged, or something else
   (section 4)?
3. Is `social_data_source_is_organic` consulted anywhere downstream (filtering, display
   badges, analytics) outside these three repos? No consuming logic found here.
4. Indexing status of `fk_glusr_social_review_id` on `glusr_social_review_detail` for the
   `ROW_NUMBER()` windowing query (section 8, point 2).
5. Whether `social_refresh_token` is encrypted at rest at the DB layer (section 10, point 5).
6. Live DB schema verification — this doc only reflects Go SQL strings, not a checked live
   schema (per section 3's verification note).

---

## See also

- [`Social_Review_Business_Doc.md`](./Social_Review_Business_Doc.md) — product perspective
- [`../Rating KT/Rating_Technical_Doc.md`](../Rating%20KT/Rating_Technical_Doc.md) — related
  but independent native-rating system
- [`../Social Contacts KT/Social_Contacts_Technical_Doc.md`](../Social%20Contacts%20KT/Social_Contacts_Technical_Doc.md) — related but independent social-media contact-links system
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — referenced above for
  comparable optimization patterns (caching gap, N+1 avoidance)
- [`../ratings_reviews_product_overview.md`](../ratings_reviews_product_overview.md) —
  original combined product-overview (Rating + Social Review together)

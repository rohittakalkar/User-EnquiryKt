# Bizfeed & Recommendation — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`Bizfeed_And_Recommendation_Business_Doc.md`](./Bizfeed_And_Recommendation_Business_Doc.md)
dekho — dono docs same flows cover karte hain, bas alag audience ke liye.

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file path diya
gaya hai har jagah). Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — team se confirm karo]** likh diya hai, guess nahi kiya.

**Scope decision (verified via code)**:
- **User Activity + Catalog View + Buyer Activity** → same write-API
  (`POST /user/addbuyeractivity`), same Cassandra table, same downstream Kafka-consumer —
  genuinely one pipeline, documented together.
- **Bizfeed CV Hide/Unhide** → a separate controller (`POST /user/bizfeed_hide_unhide`), but
  writes directly to the pipeline's own output-table (`glusr_usr_biz_feeds`) — kept in this
  same folder as a sibling action on the same data, not a fully independent feature.
- **Recommendation (Action Items)** → a separate dashboard-aggregator (`GET/POST
  /seller_recommendation/*params`) that pulls Bizfeed-data (among 4 other services) —
  included here since it's the direct downstream consumer of Bizfeed output.
- **Latitude & Longitude Details** → confirmed unrelated (different table
  `GLUSR_GEO_ADDT_CONTACT`, different write-path) — already documented at
  [`../Location Update KT/`](../Location%20Update%20KT/), not duplicated here.
- **Last Seen** → grepped `last_seen`/`LastSeen`/`LAST_SEEN` (case-insensitive) across all 3
  repos — **zero matches**. Not implemented anywhere in this codebase.

**Repos** (all under `C:\IM API Repos\local\users-api-go-production\`):
`service-api-go-production` (write: Buyer Activity, Bizfeed Hide/Unhide),
`user-temp-consumers-production` (Kafka consumer: Catalog View/Bizfeed generation),
`users-api-go-production` (read: Recommendation/Action-Items aggregator).

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Buyer/User Activity write | write | [`UserBuyerActivityController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserBuyerActivityController.go), [`UserBuyerActivityModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserBuyerActivityModel.go) (`UpsertBuyerActivity`) |
| Validation (mandatory) | write | `MandatoryParamsCheckBuyerActivity` — [`UserUtilsMandatory.go:1997`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go) |
| Validation (type/length map) | write | `BuyerActivityMap` — [`UsersValidationMaps.go:810`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |
| Catalog View / Bizfeed generation (consumer, Kafka-driven) | consumers | [`USER_BUSINESS_FEEDS.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go) (`UserBusinessFeeds` = connection bootstrap, `workerUserBusinessFeeds` = per-message handler, `pgInsert` = write, `CheckUserBlocked` = block-gate, `checkSellerClientMeeting` = Redis gate) |
| Column-mapping for consumer insert | consumers | `UserBizFeedMap` — [`map_index.go:820`](../../user-temp-consumers-production/user-temp-consumers-production/pkg/utils/map_index.go) |
| Bizfeed Hide/Unhide write | write | [`UserBizfeedHideUnhideController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserBizfeedHideUnhideController.go) (`UserBizHideUnhideController`), [`UserBizHideUnhideModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserBizHideUnhideModel.go) (`UpsertBizhideunhide`) |
| Validation | write | `MandatoryParamsCheckBizHideUnhide` — [`UserUtilsMandatory.go:784`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go) |
| Recommendation/Action-Items (read, aggregator) | read | [`SellerRecommendationController.go`](../internal/controllers/UsersControllers/SellerRecommendationController.go) (`ActionSellerRecommendation`), [`UserSellerRecommendationModel.go`](../internal/models/users/UserSellerRecommendationModel.go) (`FetchActionItems`, `CallApi`/`CallApi2` HTTP-loopback helpers) |

---

## 2. Routes (confirmed from router.go)

| Method | Path | serviceName | Repo | Controller |
|---|---|---|---|---|
| POST | `/user/addbuyeractivity` | `BUYER_ACTIVITY_SERVICE` | write | `UserBuyerActivityController` — [`router.go:345`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) |
| POST | `/user/bizfeed_hide_unhide` | `BIZFEED_TOGGLE_SERVICE` | write | `UserBizHideUnhideController` — [`router.go:152,249,320`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) (registered 3x, across different route-group versions) |
| GET/POST | `/seller_recommendation/*params` | `SELLER_RECOMMENDATION` | read | `ActionSellerRecommendation` — [`routerUsers.go:457-458,772-773`](../internal/api/users_router/routerUsers.go) (registered twice, across route-group versions) |
| — (Kafka-consumed) | queues `USER_BUSINESS_FEED` and `CATALOG_VIEW_BACKFILLING` | `CATALOG_VIEW_SERVICE` | consumers | `workerUserBusinessFeeds` — both queue-names map to the **same handler function** ([`IntializeMsgBroker.go:248,253`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go)) — `USER_BUSINESS_FEED` is real-time, `CATALOG_VIEW_BACKFILLING` is presumably a backfill/replay path for the same processing (name-based inference) |

---

## 3. Data Model — Tables

> **Verification note**: yeh sab table/column names Go code ke andar embedded SQL/CQL
> strings se liye gaye hain (har row ke saamne file path diya hai). Live DB schema se
> cross-verify **nahi** kiya gaya hai — kisi migration/integration se pehle woh zaroor karo.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `clickstream.im_buyer_activity_attr_v2` | csl-cassandra | Raw activity-event log — covers User Activity AND Catalog-View-triggering-events, differentiated by `activity_type_id` | `glusr_id`, `datetime`, `activity_type_id`, `fk_activity_id`, `sno` (server-generated `uuid()`), `coordinate_latitude`/`_longitude`/`_accuracy` (event-location-metadata, NOT the Lat/Long feature's table), `fk_display_title`, `ga_utma_cookie`, `glcat_mcat_id`/`_name`, `group_id`/`_name`, `identified_data_flag`, `insertion_time`, `keyword`, `nob_type`, `product_disp_id`, `referer`, `remote_ip`, `request_url`, `seller_glusr_id`, `state_id`/`_name`, `subcat_id`/`_name`, `url_weight`, `location_pref_city_ids`/`_names`, `mcat_ids`/`_names` (32 columns total) — [`UserBuyerActivityModel.go:63-65`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserBuyerActivityModel.go) |
| `glusr_usr_biz_feeds` | cslpg | Processed, supplier-facing feed-items (Bizfeed) generated from activity-events by the Kafka consumer | `fk_glusr_usr_id` (supplier/catalog-owner), `biz_feed_activity_glusr_id` (the buyer), `biz_feed_activity_city_id`/`_state_id`/`_country_iso`, `biz_feed_date`, `biz_feed_activity_id`, `biz_feed_domain`, `biz_feed_display_title`, `biz_feed_modid`, `biz_feed_modref_id`/`_name`/`_type` (modref_id always inserted `nil` — see Edge Cases #3), `biz_feed_mcat_id`, `biz_feed_referer`, `biz_feed_remote_ip`, `biz_feed_request_url`, `biz_feed_is_hidden` — [`USER_BUSINESS_FEEDS.go:266`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go) (insert), [`UserBizHideUnhideModel.go:38`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserBizHideUnhideModel.go) (update) |
| `glusr_usr` | meshPg | General user/company table — read (not written) in this domain, twice: once by the consumer to fetch `FK_GL_CITY_ID`/`FK_GL_STATE_ID` for the buyer, once (with a much larger blacklist/fraud-check query) as part of the block-gate | [`USER_BUSINESS_FEEDS.go:120-121,364-398`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go) |
| `pc_item` | meshPg | Product-catalog item table — read only inside the block-gate's `out_of_stock` sub-check, when a `product_disp_id` is present on the message | [`USER_BUSINESS_FEEDS.go:401-411`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go) |
| `query_approval_blacklist` | meshPg | Blacklist table (mobile/email/email-domain) — read only, inside the block-gate | [`USER_BUSINESS_FEEDS.go:322-347`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go) |
| `employee` | meshPg | Internal-employee table — read only, inside the block-gate, to detect if the buyer's mobile belongs to an internal employee (excluding IB/MANTHAN designations) | [`USER_BUSINESS_FEEDS.go:355-362`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go) |

**Note on physical-DB split**: raw-activity lives in **Cassandra** (high-write-volume
clickstream-style data), while the processed/derived Bizfeed and all its supporting
lookup-tables live in **Postgres** (`cslpg`/`meshPg`) — a classic write-optimized-ingest vs
read-optimized-serving split.

---

## 4. Decoding Magic Values

### `activity_type_id` / `nob_type` (Buyer Activity write)

**Not found as a documented enum or reference-table anywhere in the 3 repos covered by this
KT series.** `MandatoryParamsCheckBuyerActivity` only validates that `activity_type_id` and
`nob_type` are numeric strings ([`UserUtilsMandatory.go:2007-2027`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go),
[`UsersValidationMaps.go:813,826`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go)) — it never branches on a specific
value. The consumer (`workerUserBusinessFeeds`) similarly passes `activity_type_id` /
`fk_activity_id` through opaquely as `biz_feed_activity_id`, without any value-based
branching. **[INFERRED — confirm with team]**: given the doc's earlier scope-note
("User Activity ka general-type ya Catalog View-specific"), `activity_type_id` almost
certainly distinguishes different clickstream-event categories (search vs product-view vs
category-browse), but the exact value→meaning mapping could not be located in code.

### `nob_type` field

Type `number`, length 1 ([`UsersValidationMaps.go:826`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go)) — likely "Nature of Business"
type code, single-digit. **[INFERRED — confirm with team]**: no enum found in these 3 repos.

### `visibility_value` (Bizfeed Hide/Unhide)

This one **is** fully traceable and unambiguous, unlike the two above:

| Value | Meaning | Evidence |
|---|---|---|
| `"1"` | Hide the feed-item (`biz_feed_is_hidden = 1`) | Mandatory-check only accepts `0` or `1` — [`UserUtilsMandatory.go:817-819`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go) |
| `"0"` | **Translated to SQL `NULL`** before the UPDATE — "clear/un-hide," not "set to false" | [`UserBizHideUnhideModel.go:13-21`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserBizHideUnhideModel.go) |
| anything else | Rejected by mandatory-check as `INVALID_VALUE` | [`UserUtilsMandatory.go:808-822`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go) |

### `is_free` (Recommendation aggregator)

`params["is_free"] == "Y"` skips the `blcredit_service` HTTP call entirely and re-ranks the
"Enrichment" category items to fill the priority slots vacated by BLCredit/Bizfeed/Unread
being deprioritized — see [`UserSellerRecommendationModel.go:32-36,107-118`](../internal/models/users/UserSellerRecommendationModel.go). Any other value (including
absent) is treated as "not free," all 5 services are called.

### `cust_type` (Recommendation aggregator)

Values `"13"`, `"21"`, `"27"` specifically trigger population of the `sugg_act_pns_defaulter`
recommendation item ([`UserSellerRecommendationModel.go:296-299`](../internal/models/users/UserSellerRecommendationModel.go)) — **[INFERRED — confirm
with team]**: these look like PNS (Paid Non-Serious / Package-related) customer-type codes,
but no enum/reference for `cust_type` was found in these 3 repos to confirm the exact
business meaning of 13/21/27.

### `disable_reason` (Recommendation aggregator)

If `strings.ToLower(disable_reason)` equals `"fraud complaint"` or `"fraud_complaint"` — OR
if `cust_type` isn't in `components.CustTypeBizMap` — the Bizfeed/"catalog views"
recommendation item is force-emptied regardless of what the `bizfeed_service` call actually
returned ([`UserSellerRecommendationModel.go:305-309`](../internal/models/users/UserSellerRecommendationModel.go)). `components.CustTypeBizMap`'s contents
weren't traced in this pass (defined outside the 3 files read directly) — **[INFERRED —
confirm with team]**: likely a customer-type allowlist for which Bizfeed recommendations make
sense to show at all.

---

## 5. Business Rules & Validation (code se)

### Buyer/User Activity write

1. **Gateway allowlist**: `GLADMIN`, `LEAP` — narrow, since this is mostly a
   machine-to-machine clickstream-ingestion endpoint. `VALIDATION_KEY` is actually
   **optional** here — if absent, `gate=""` and the request still proceeds, unlike almost
   every other controller in this codebase which hard-rejects on a missing/invalid
   validation-key. [`UserBuyerActivityController.go:44-51`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserBuyerActivityController.go)
2. **Mandatory**: `glusr_id`, `VALIDATION_KEY`, `activity_type_id`, `fk_activity_id` (all
   4 checked together in one combined error message), then `datetime` (must be exactly
   14 numeric characters — `YYYYMMDDHHMMSS`), then `activity_type_id`/`fk_activity_id` must
   both be numeric. [`UserUtilsMandatory.go:1997-2033`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
3. **Every column in `BuyerActivityMap` gets a type-appropriate default if absent from the
   request** (`0`/`0.0` for numbers, `[]` for list-types, `""` for strings) — the Cassandra
   insert always supplies all 32 columns, never partial.
   [`UserBuyerActivityModel.go:36-61`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserBuyerActivityModel.go)
4. **`list<int>`/`list<string>`-typed fields are JSON/comma-parsed from the request** into
   actual Cassandra list-collections (`location_pref_city_ids`/`_names`,
   `mcat_ids_list`/`mcat_names_list` — note the request-key differs from the DB-column-name
   for the mcat fields: `mcat_ids_list` request-key maps to `mcat_ids` DB-column).
   [`UsersValidationMaps.go:839-840`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go)
5. **Insert uses parameterized Cassandra CQL with a server-generated `uuid()`** for the `sno`
   column — no client-supplied primary-key. `insertion_time` is always server-stamped
   (`current_time`), never client-supplied, regardless of what the request sends.
   [`UserBuyerActivityModel.go:62-65`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserBuyerActivityModel.go)

### Catalog View / Bizfeed generation (consumer)

6. **Self-activity is silently dropped**: the message is only processed if
   `glusr_id != "0"/""` AND `glusr_id != catalog_owner_glusr_id` AND
   `catalog_owner_glusr_id != "0"/""` — i.e. a supplier viewing their own catalog never
   generates a Bizfeed entry (makes sense: "buyer viewed your product" wouldn't be
   meaningful if buyer==seller). [`USER_BUSINESS_FEEDS.go:91`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go)
7. **Block-gate (`CheckUserBlocked`) runs before any Bizfeed row is created** — a single
   large query against `glusr_usr` (joined subquery-style with `query_approval_blacklist`,
   `pc_item`, `employee`) computes 8 boolean-ish flags; if the buyer is disabled
   (`GLUSR_USR_APPROV != 'A'`), blacklisted by mobile/email/email-domain, has an invalid
   mobile (country=IN and mobile starts with `1`), the specific product is out-of-stock
   (`PC_ITEM_STATUS_APPROVAL = 9`), or the buyer is an internal employee — the message is
   dropped entirely, no Bizfeed row is written.
   [`USER_BUSINESS_FEEDS.go:314-439`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go)
8. **Redis-based "seller-client-meeting" bypass**: `checkSellerClientMeeting` does a plain
   Redis `EXISTS` check on the buyer's GLID as the key — if that key exists in
   `bizfeedRedis`, the message is dropped (no Bizfeed row written), logged as "User bypassed
   due to Meeting log." **[INFERRED — confirm with team]**: the key's meaning (buyer had an
   in-person/offline sales-meeting with this seller recently?) and who *writes* this Redis
   key is not visible in these 3 repos — it's read-only from this consumer's perspective.
   [`USER_BUSINESS_FEEDS.go:106-115,441-450`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go)
9. **City/State enrichment via a live DB lookup, with silent-fallback**: `FK_GL_CITY_ID`/
   `FK_GL_STATE_ID` for the buyer are fetched from `glusr_usr` by GLID; if the query errors
   (e.g. buyer not found), both default to `0` rather than failing the message.
   [`USER_BUSINESS_FEEDS.go:116-125`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go)
10. **Field-mapping is driven entirely by `UserBizFeedMap`** — the incoming Kafka message is
    a generic `map[string]interface{}`, and only keys present in `UserBizFeedMap` (with a
    matching declared `Type`: `int`/`bigint`/`float`/`double`/`string`/`text`/`date`, and
    `Length` of `""` or `"list"`) get copied into the insert-map — anything else in the
    message is silently ignored. [`USER_BUSINESS_FEEDS.go:126-174`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go)
11. **Hard length-truncation before insert** (not rejection): `referer` > 2500 chars,
    `remote_ip` > 250 chars, `request_url` > 500 chars, `product_name` > 250 chars are all
    silently truncated (not error'd), and the truncated-column-names are recorded in the
    Kibana log (`COLS_TRIMMED`) for observability. [`USER_BUSINESS_FEEDS.go:181-200`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go)
12. **GLID length hard-rejection**: if either `glusr_id` or `catalog_owner_glusr_id` exceeds
    10 digits as a string, the message is dropped entirely (logged as failure, no retry).
    [`USER_BUSINESS_FEEDS.go:202-215`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go)
13. **Insert retries on generic failure, but NOT on the two specific errors that are
    guaranteed to keep failing**: `pgInsert` retries up to 2 more times (500ms, 1000ms
    backoff, capped/reset at 10 minutes) UNLESS the Postgres error is `"numeric field
    overflow"` or `"pq: value too long"` — those two fail immediately without retry, since
    retrying identical bad data would just fail again. [`USER_BUSINESS_FEEDS.go:268-297`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go)
14. **`biz_feed_modref_id` is always inserted as `nil`** regardless of what's in the message
    (`dbparams` hardcodes a literal `nil` at that position) — `biz_feed_modref_name` is the
    one actually populated from `product_name`. [`USER_BUSINESS_FEEDS.go:260-266`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go)

### Bizfeed Hide/Unhide write

15. **Gateway allowlist**: `GLADMIN`, `SELLERMY`, `BUYERMY`, `MAPI`.
    [`UserBizfeedHideUnhideController.go:53`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserBizfeedHideUnhideController.go)
16. **Mandatory + type validation, all in one function**: `glusrid`/`buyerid` must be numeric
    strings if present, `visibility_value` must be exactly `"0"` or `"1"`, `datetime` must
    parse as `YYYYMMDD` (note: 8-char date format here, vs 14-char `YYYYMMDDHHMMSS` in Buyer
    Activity — the two related endpoints use different datetime granularities). All 5 fields
    (glusrid, buyerid, visibility_value, datetime, VALIDATION_KEY) are mandatory.
    [`UserUtilsMandatory.go:784-846`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
17. **`visibility_value="0"` is translated to `NULL`** before the update (see section 4) —
    i.e. `0` means "clear the hidden-flag," not "explicitly set to not-hidden."
18. **Match-key for the update is a 3-part composite**: `(fk_glusr_usr_id,
    biz_feed_activity_glusr_id, biz_feed_date::date formatted as yyyymmdd)` — a supplier can
    only hide/unhide a feed-item for a specific buyer-on-a-specific-day, not by a direct
    row-ID. If multiple Bizfeed rows exist for the same (supplier, buyer, day) triple — which
    is entirely possible since the consumer inserts one row per qualifying activity-event —
    **all of them get updated by a single hide/unhide call**, since the UPDATE has no
    row-level uniqueness constraint in its WHERE clause.
    [`UserBizHideUnhideModel.go:38-40`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserBizHideUnhideModel.go)

### Recommendation (Action Items)

19. **Mandatory-param validation is generic**: `token`, `glusr_id`, `modid` via
    `utils.CheckValidity`, then `is_free`, `cust_type`, `logtime` must all be present, and
    `cust_type` must be numeric — [`SellerRecommendationController.go:46,57-66`](../internal/controllers/UsersControllers/SellerRecommendationController.go).
20. **`FetchActionItems` is a parallel HTTP-loopback fan-out**: fires 4 or 5 concurrent
    goroutine-calls (`CallApi`/`CallApi2`) to `otherdetails_service`, `productlisting_service`,
    `unread_msg` (via `CallApi`, a POST-style call), `bizfeed_service`, and conditionally
    `blcredit_service` (skipped if `is_free=="Y"`) — same HTTP-loopback pattern seen across
    Rating Usefulness, Trust Verification, Update With OTP, App-Rating Feedback, but here used
    for **read-aggregation** rather than a write-delegation.
    [`UserSellerRecommendationModel.go:18-50`](../internal/models/users/UserSellerRecommendationModel.go)
21. **Every response-slot is defaulted, never left absent**: after the `wg.Wait()`, every one
    of the 11 `sugg_act_*` keys is force-normalized to `{"datacount": 0}` if its `datafield`
    is still an empty slice (i.e. never populated, whether due to a service error or a
    business-rule suppression) — so the response shape is always complete regardless of
    partial upstream failures. [`UserSellerRecommendationModel.go:301-371`](../internal/models/users/UserSellerRecommendationModel.go)
22. **Each source is tagged with a `priority` and `type`** (e.g. `PayIMMap`: priority 1, type
    "Monetization"; `BLCreditMap`: priority 2, type "Engagement"; GST/PAN/Product/CustType:
    "Enrichment") — and **priorities re-shuffle when `is_free=="Y"`** (GST jumps from
    priority 5 to priority 2, etc. — see section 4) — the combined response is presented to
    the supplier in a priority-ranked, categorized "action items" list, and free-tier
    suppliers see a different priority order than paid ones.
    [`UserSellerRecommendationModel.go:52-118`](../internal/models/users/UserSellerRecommendationModel.go)
23. **`sugg_act_pwim` (Pay-IM/Monetization item) is never actually populated by any of the 5
    HTTP calls** — `PayIMMap["datafield"]` stays `[]string{}` through the entire function and
    always gets force-normalized to `{"datacount": 0}` at the end. **[INFERRED — confirm with
    team]**: either this is dead/placeholder code for a not-yet-wired data source, or PayIM
    status is intentionally always reported as "0" from this endpoint (e.g. computed
    elsewhere). [`UserSellerRecommendationModel.go:52-55,311-314`](../internal/models/users/UserSellerRecommendationModel.go)

---

## 6. RabbitMQ

**No RabbitMQ found in this domain** — Buyer Activity, Bizfeed generation, Bizfeed
Hide/Unhide, and Recommendation are all either direct synchronous DB writes/reads, or a
Kafka-consumed pipeline. This is unlike most other features in this KT series, and is
consistent with Buyer Activity's high-throughput-clickstream nature (Kafka scales better for
this workload-shape than RabbitMQ's per-message ack/routing overhead). If someone asks "does
Bizfeed use RabbitMQ," the accurate answer is: no, this entire domain is RabbitMQ-free.

---

## 7. Kafka

| Queue name(s) | Publisher | Consumer | Purpose |
|---|---|---|---|
| `USER_BUSINESS_FEED` | **Not conclusively traced in these 3 repos** — the write-path (`UpsertBuyerActivity`) only writes directly to Cassandra; no code in these repos calls a Kafka producer for this topic. **[INFERRED — confirm with team]**: given a sibling queue `GLUSR_USR_DEBEZIUM_SYNC` exists in the same router registry ([`IntializeMsgBroker.go:252`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go)), a Debezium/CDC-style bridge off Cassandra is architecturally plausible, but not confirmed | `workerUserBusinessFeeds` — [`USER_BUSINESS_FEEDS.go:58`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go) | Real-time transform of raw activity into `glusr_usr_biz_feeds` rows |
| `CATALOG_VIEW_BACKFILLING` | Not traced — same handler as above | `workerUserBusinessFeeds` (identical handler, [`IntializeMsgBroker.go:253`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go)) | **[INFERRED]** name strongly suggests a backfill/replay path re-using the exact same business logic — e.g. to regenerate Bizfeed rows for a historical date-range or recover from an outage |

Both queues route through `InitializeKafka` with env-configured `sub_topic` /
`consumer_group` ([`USER_BUSINESS_FEEDS.go:27-51`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go)), and both are registered with a
concurrency of 10 and 5 goroutines respectively ([`IntializeMsgBroker.go:240,243`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go)).

**No other Kafka touchpoint exists in this domain** — Bizfeed Hide/Unhide and the
Recommendation aggregator are both purely synchronous (DB write / HTTP fan-out
respectively), no queue involvement at all.

---

## 8. Redis

**One confirmed touchpoint**: `bizfeedRedis` — connected inside `UserBusinessFeeds`
([`USER_BUSINESS_FEEDS.go:41`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go)), used exclusively by `checkSellerClientMeeting`
which does a single `EXISTS <buyer_glusr_id>` check — if the key exists, the incoming
activity-event is treated as a "meeting bypass" and no Bizfeed row is generated for it
(section 5, rule 8). This is a **read-only** usage from this consumer's perspective — nothing
in these 3 repos writes to this Redis instance, so the key's producer/TTL/exact semantics are
**[INFERRED — confirm with team]**.

**No Redis usage found** in Buyer Activity write, Bizfeed Hide/Unhide write, or the
Recommendation aggregator — all three hit Postgres/Cassandra or make live HTTP calls on
every request, no caching layer.

---

## 9. End-to-End Technical Flows

### Flow A — Buyer generates activity → Bizfeed entry appears for the supplier

```
Buyer (browsing/searching on IndiaMART)
    │
    ▼
[API — write]  POST /user/addbuyeractivity  serviceName=BUYER_ACTIVITY_SERVICE
                {glusr_id, VALIDATION_KEY, datetime(14-char), activity_type_id,
                 fk_activity_id, keyword, glcat_mcat_id, seller_glusr_id,
                 coordinate_latitude/longitude (event-metadata), mcat_ids_list, ...}
    │  UserBuyerActivityController.go — Gateway(GLADMIN/LEAP, VALIDATION_KEY optional)
    │  MandatoryParamsCheckBuyerActivity() → LengthAndTypeValidations_v3(BuyerActivityMap)
    ▼
UpsertBuyerActivity()
    │  every BuyerActivityMap column defaulted if absent, list-fields JSON-parsed
    │  INSERT INTO clickstream.im_buyer_activity_attr_v2 (32 columns, server uuid() + insertion_time)
    ▼
[DB — csl-cassandra]
    │
    ▼  (Kafka publish-point NOT traced in these 3 repos — see Open Questions)
[Kafka topic]  queue USER_BUSINESS_FEED
    ▼
[CONSUME]  workerUserBusinessFeeds()
    │  1. Skip if glusr_id==catalog_owner_glusr_id (self-view) or either is "0"/""
    │  2. CheckUserBlocked() — glusr_usr + query_approval_blacklist + pc_item + employee
    │       → drop message if buyer disabled/blacklisted/invalid-mobile/OOS-product/employee
    │  3. checkSellerClientMeeting() — Redis EXISTS on buyer GLID → drop if bypassed
    │  4. SELECT city/state from glusr_usr (silent fallback to 0/0 on error)
    │  5. Map message fields via UserBizFeedMap (type-driven cast/parse)
    │  6. Truncate oversized referer/remote_ip/request_url/product_name (log COLS_TRIMMED)
    │  7. Reject if glusr_id or catalog_owner_glusr_id > 10 digits
    ▼
pgInsert()
    │  INSERT INTO glusr_usr_biz_feeds (17 columns; biz_feed_modref_id always NULL)
    │  retries up to 2x (500ms/1000ms backoff) UNLESS "numeric field overflow" /
    │  "pq: value too long" (those fail immediately, no retry)
    ▼
[DB — cslpg]  glusr_usr_biz_feeds  ← supplier's Bizfeed now shows this entry
```

### Flow B — Supplier hides/unhides a Bizfeed entry

```
Supplier (viewing/managing feed in seller panel/app)
    │
    ▼
[API — write]  POST /user/bizfeed_hide_unhide  serviceName=BIZFEED_TOGGLE_SERVICE
                {glusrid, buyerid, datetime(8-char yyyymmdd), visibility_value(0|1),
                 VALIDATION_KEY}
    │  UserBizHideUnhideController.go
    │  InputJsonCheck → MandatoryParamsCheckBizHideUnhide() (numeric/format checks)
    │  Gateway(GLADMIN/SELLERMY/BUYERMY/MAPI)
    ▼
UpsertBizhideunhide()
    │  visibility_value=="0" → translated to SQL NULL
    │  UPDATE glusr_usr_biz_feeds SET biz_feed_is_hidden=$1
    │  WHERE (fk_glusr_usr_id, biz_feed_activity_glusr_id, date) match
    │  (composite match — can update MULTIPLE rows if >1 activity for same buyer+day)
    ▼
[DB — cslpg]  glusr_usr_biz_feeds updated
```

### Flow C — Supplier views the Recommendation dashboard

```
Supplier (viewing Recommendation/Action-Items dashboard)
    │
    ▼
[API — read]  GET/POST /seller_recommendation/*params  serviceName=SELLER_RECOMMENDATION
                {token, glusr_id, modid, is_free, cust_type, logtime}
    │  ActionSellerRecommendation() — CheckValidity(token/glusr_id/modid),
    │  mandatory-check(is_free/cust_type/logtime), cust_type must be numeric
    ▼
FetchActionItems() — parallel HTTP-loopback fan-out (4 or 5 goroutines, sync.WaitGroup)
    ├─ otherdetails_service   (GET-style, CallApi2)  → GST + PAN data
    ├─ productlisting_service (GET-style, CallApi2)  → photo/price/rejection/ISQ product counts
    ├─ unread_msg             (POST-style, CallApi)  → unread-enquiries count
    ├─ bizfeed_service        (GET-style, CallApi2)  → reads glusr_usr_biz_feeds via its own service
    └─ blcredit_service       (GET-style, CallApi2, SKIPPED if is_free=="Y") → lapsed-credit count
    │
    │  each result mapped into one of 11 sugg_act_* keys, tagged priority+type
    │  (priorities re-order for is_free=="Y" — see section 4)
    │  disable_reason=="fraud complaint"/"fraud_complaint" or cust_type not in
    │  components.CustTypeBizMap → force-empty the Bizfeed recommendation item
    │  every still-empty datafield normalized to {"datacount": 0}
    ▼
Combined, priority-ranked "action items" JSON response (always 11 keys present)
```

---

## 10. Flow-wise DB & Table Usage — Kaun sa DB, Kaun sa Table, Kis Liye

### Flow A — Buyer Activity → Bizfeed Generation

| # | DB (physical) | Table | Operation | Kya nikala/likha jaata hai, aur kyun |
|---|---|---|---|---|
| 1 | csl-cassandra | `clickstream.im_buyer_activity_attr_v2` | INSERT | Raw clickstream event ka permanent, append-only record — 32 columns, server-generated `sno`/`insertion_time` |
| 2 | meshPg (consumer side) | `glusr_usr` | SELECT | Buyer ke saare fraud/block-signals ek query mein nikalta hai — blacklist joins, employee-check, disabled-check (`CheckUserBlocked`) |
| 3 | meshPg | `pc_item` | SELECT (conditional, sirf jab `product_disp_id` present ho) | Out-of-stock check — agar product already `PC_ITEM_STATUS_APPROVAL=9` hai, block-gate isse bhi factor karta hai |
| 4 | meshPg | `query_approval_blacklist` | SELECT (3x, subqueries within `CheckUserBlocked`'s single query) | Mobile/alt-mobile/email/email-domain blacklist check |
| 5 | meshPg | `employee` | SELECT (subquery) | Buyer ka mobile kisi internal-employee ka toh nahi (IB/MANTHAN designation exempt) |
| 6 | bizfeedRedis | (no table, key=buyer GLID) | `EXISTS` | "Meeting bypass" check — agar key exist karta hai, message drop ho jaata hai |
| 7 | meshPg | `glusr_usr` (again, separate query) | SELECT `FK_GL_CITY_ID`, `FK_GL_STATE_ID` | Buyer ka city/state Bizfeed row mein enrich karne ke liye (fails silently to 0/0) |
| 8 | cslPg | `glusr_usr_biz_feeds` | INSERT (with up to 2 retries on generic failure) | Final Bizfeed-entry likhi jaati hai — yeh feature ka actual output/serving-row hai |

**Total DB/Redis round-trips per qualifying activity-event: at least 7** (one Cassandra
write on the upstream side, then 6+ reads/1 write on the consumer side) — and that's *before*
counting the fact that a single Kafka message can fail this many gates and never reach the
final INSERT at all (self-view, blocked, meeting-bypass, oversized-GLID).

### Flow B — Bizfeed Hide/Unhide

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | cslpg | `glusr_usr_biz_feeds` | UPDATE (single statement, composite WHERE) | Sirf ek write — is domain ka sabse cheap/fast flow, but can silently affect multiple rows (section 5, rule 18) |

### Flow C — Recommendation/Action-Items

| # | DB / Service | Table (agar direct visible ho) | Operation | Kyun |
|---|---|---|---|---|
| 1-5 | 5 separate downstream HTTP services (`otherdetails_service`, `productlisting_service`, `unread_msg`, `bizfeed_service`, `blcredit_service`) | Not visible from this aggregator — each service owns its own DB access | GET/POST HTTP-loopback (parallel) | This endpoint itself does **zero direct DB access** — it's a pure fan-out/aggregation layer. All actual DB reads happen inside the 5 downstream services (`bizfeed_service` presumably reads `glusr_usr_biz_feeds`, but that's outside these 3 repos' visible code) |

**Business-critical insight**: because Flow C has no DB access of its own, its response time
is **entirely bounded by its slowest downstream HTTP dependency** — there is no DB
optimization possible at this layer; any speedup has to happen either in the 5 downstream
services or in this layer's fan-out/timeout strategy (see Optimization Scope).

---

## 11. Optimization Scope — DB / Response-Time Contribution

### High-impact

1. **Flow A's block-gate (`CheckUserBlocked`) runs one large multi-subquery SELECT against
   `glusr_usr` PLUS a second separate `glusr_usr` SELECT for city/state, PLUS a Redis EXISTS
   check — all sequentially, all before the actual Bizfeed INSERT even starts.** For a
   high-throughput clickstream consumer (concurrency 10, per `GoroutinesConsumers`), this is
   4-5 sequential round-trips per message before any useful write happens. **Suggestion**: the
   city/state `glusr_usr` lookup (step 7 in Flow A) could be folded into the same query as the
   block-gate's `glusr_usr` read (step 2) — both are keyed by GLID and both hit the same table
   — cutting one full round-trip per message.
   [`USER_BUSINESS_FEEDS.go:96,120-121`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go)

2. **Recommendation-aggregator's overall latency is bounded by its slowest of 4-5 parallel
   dependencies** (`wg.Wait()` blocks until all complete) — since this layer has zero DB
   access of its own, a single slow downstream service (e.g. `productlisting_service` running
   a heavy catalog-completeness query) directly becomes this endpoint's p99. **Suggestion**: a
   per-call timeout with partial-response fallback (already effectively half-built, since
   every `sugg_act_*` key IS defaulted to `{"datacount": 0}` on `err_response` — but that
   default only triggers on a hard error, not a slow-but-eventually-successful call) would
   improve perceived performance without changing correctness.
   [`UserSellerRecommendationModel.go:39-50,286`](../internal/models/users/UserSellerRecommendationModel.go)

### Medium impact

3. **`pgInsert`'s retry-with-sleep runs on the same goroutine that's processing the Kafka
   message** — a 500ms+1000ms retry sequence (up to 1.5s+ of blocking sleep) happens inline,
   holding that consumer-goroutine slot the whole time. With only 10 goroutines configured for
   `USER_BUSINESS_FEED` ([`IntializeMsgBroker.go:240`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go)), a burst of
   transient Postgres errors could meaningfully throttle overall consumer throughput.
   [`USER_BUSINESS_FEEDS.go:279-293`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go)

4. **`fmt.Println(input_params)` in `UpsertBuyerActivity`** writes every single insert's full
   parameter list to stdout on every request — on a high-volume clickstream endpoint, this is
   meaningful, easily-avoidable I/O overhead (and a minor data-hygiene concern, since it's an
   unstructured stdout dump rather than the structured Kibana logging used everywhere else in
   this codebase). [`UserBuyerActivityModel.go:66`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserBuyerActivityModel.go)

### Low-impact / good-practice already present

5. **Recommendation-aggregator already uses parallel goroutines + a buffered result-channel**,
   not sequential HTTP calls — already optimized for the "fan out, wait for all" pattern.

6. **Cassandra for raw-clickstream ingest, Postgres for processed-serving-data** — an
   appropriate storage-split for this workload-shape (high-write-throughput vs
   query-friendly-reads), same pattern GST uses for its own high-volume paths.

7. **The block-gate's error-tolerant design (`if err != nil { return dataMap, false, err }`
   but callers still log-and-continue rather than crash) means a transient `glusr_usr` query
   failure doesn't halt the whole consumer** — a reasonable resilience tradeoff, though it
   does mean a DB blip could let a should-have-been-blocked buyer's activity through (see Edge
   Cases #2).

---

## 12. Cron Inventory

**No cron found for this domain** — searched `service-api-go-production/service-api-go-
production/crons/` and the `users-api-go-production` cmd/crons trees for anything referencing
`biz`, `activity`, `recommendation`, or `catalog_view` — nothing matched. Unlike GST (which
has a nightly BigQuery-driven re-verification job), Bizfeed & Recommendation is a purely
event-driven (Kafka) + on-demand-read (Recommendation) domain with no scheduled batch job.

---

## 13. Edge Cases & Gotchas (technical POV)

1. **Buyer Activity write doesn't hard-require a valid `VALIDATION_KEY`** — unlike almost
   every other endpoint in this codebase; if `VALIDATION_KEY` is absent, `gate` is just
   empty-string and the request proceeds anyway. Worth confirming this is intentional for a
   high-volume clickstream-ingest endpoint (trusted internal callers only) rather than an
   oversight. [`UserBuyerActivityController.go:46-51`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserBuyerActivityController.go)
2. **Block-gate failure is fail-open, not fail-closed**: if `CheckUserBlocked`'s query errors
   out (e.g. DB blip), `isBlock` stays `false` and the message proceeds to generate a Bizfeed
   entry anyway — the error is only logged (`Messagelog["BlockError"]`), not enforced.
   [`USER_BUSINESS_FEEDS.go:96-104`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go)
3. **Hide/Unhide's composite match-key (user+buyer+date) means multiple feed-items on the
   same day for the same buyer can't be independently hidden** — if a supplier had two
   distinct interactions with the same buyer on the same day (e.g. buyer viewed 2 different
   products), hiding one hides both, since there's no item-level unique-ID in the WHERE
   clause — `biz_feed_modref_id` is always `NULL` (never populated at insert-time, per section
   5 rule 14), so it couldn't be used for row-level targeting even if the API wanted to.
4. **Buyer Activity's `datetime` is 14-char (`YYYYMMDDHHMMSS`) but Bizfeed Hide/Unhide's
   `datetime` is 8-char (`YYYYMMDD`)** — two endpoints in the same domain use different
   granularity for a field with the same name; easy to get wrong integrating against both.
5. **`workerUserBusinessFeeds` drops messages silently in 4 different ways** (self-view,
   blocked-buyer, meeting-bypass, oversized-GLID) with only a Kibana log entry, no retry/DLQ
   for these specific drops (as opposed to the DB-insert-failure path, which does retry) — if
   a supplier reports "buyer clearly viewed my product but it's not in my Bizfeed," any of
   these 4 silent-drop paths is a plausible first thing to check.
6. **Kafka-topic's producer isn't visible in these 3 repos** — the write-path
   (`UpsertBuyerActivity`) only writes to Cassandra directly; how that becomes a
   `USER_BUSINESS_FEED` Kafka message isn't traceable from this codebase alone (see Open
   Questions).
7. **`sugg_act_pwim` (PayIM) is structurally present in every Recommendation response but
   never actually populated by live data** (section 5, rule 23) — if a consumer of this API
   expects a real PayIM signal there, they'll always see `{"datacount": 0}`.
8. **Two nearly-identical route registrations for the same 3 endpoints** (`bizfeed_hide_unhide`
   registered 3x across route-group versions, `seller_recommendation` registered 2x) — if a
   behavior-change is needed, check whether it needs applying at multiple router
   registration-points, not just one.

---

## 14. Open Questions

1. **What actually publishes to the `USER_BUSINESS_FEED` / `CATALOG_VIEW_BACKFILLING` Kafka
   topics?** `UpsertBuyerActivity` only writes to Cassandra directly in these 3 repos — no
   Kafka producer call was found for this topic. A Debezium/CDC-style bridge is architecturally
   plausible (a sibling `GLUSR_USR_DEBEZIUM_SYNC` queue exists in the same consumer registry)
   but not confirmed.
2. **What writes the `bizfeedRedis` "seller-client-meeting" key** that `checkSellerClientMeeting`
   reads? Not found in these 3 repos — this consumer only reads it.
3. **`activity_type_id`, `fk_activity_id`, and `nob_type`'s full value-mappings** — no
   enum/reference table found in this codebase for any of the three.
4. **`components.CustTypeBizMap`'s exact contents** — referenced by the Recommendation
   aggregator to gate the Bizfeed recommendation item, but its definition wasn't read in this
   pass (lives in a `components` package outside the 3 files directly traced).
5. **What does `CATALOG_VIEW_BACKFILLING` actually backfill, and how/when is it triggered?**
   Same handler function as the real-time queue, but the trigger-mechanism (manual replay?
   scheduled?) wasn't found.
6. **Is `sugg_act_pwim` (PayIM) intentionally always empty from this endpoint**, or is this an
   unfinished/dead integration point?
7. Live DB schema verification — is doc ne sirf Go SQL/CQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`Bizfeed_And_Recommendation_Business_Doc.md`](./Bizfeed_And_Recommendation_Business_Doc.md) — product perspective
- [`../Location Update KT/Location_Update_Technical_Doc.md`](../Location%20Update%20KT/Location_Update_Technical_Doc.md) —
  unrelated Lat/Long feature (different table, different write-path)
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — depth/structure reference this doc was rebuilt against

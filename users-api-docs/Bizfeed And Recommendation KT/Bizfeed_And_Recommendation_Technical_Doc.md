# Bizfeed & Recommendation — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`Bizfeed_And_Recommendation_Business_Doc.md`](./Bizfeed_And_Recommendation_Business_Doc.md)
dekho.

**Scope decision (verified via code)**:
- **User Activity + Catalog View + Buyer Activity** → same write-API, same Cassandra
  table, same downstream Kafka-consumer — genuinely one pipeline, documented together.
- **Bizfeed CV Hide/Unhide** → a separate controller, but writes directly to the
  pipeline's own output-table (`glusr_usr_biz_feeds`) — kept in this same folder as a
  sibling action on the same data, not a fully independent feature.
- **Recommendation (Action Items)** → a separate dashboard-aggregator that pulls
  Bizfeed-data (among other services) — included here since it's the direct consumer of
  Bizfeed output.
- **Latitude & Longitude Details** → confirmed unrelated (different table
  `GLUSR_GEO_ADDT_CONTACT`, different write-path) — already documented at
  [`../Location Update KT/`](../Location%20Update%20KT/), not duplicated here.
- **Last Seen** → grepped `last_seen`/`LastSeen`/`LAST_SEEN` (case-insensitive) across
  all 3 repos — **zero matches**. Not implemented anywhere in this codebase.

**Repos**: `service-api-go-production` (write: Buyer Activity, Bizfeed Hide/Unhide),
`user-temp-consumers-production` (Kafka consumer: Catalog View/Bizfeed generation),
`users-api-go-production` (read: Recommendation/Action-Items aggregator).

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Buyer/User Activity write | write | [`UserBuyerActivityController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserBuyerActivityController.go), [`UserBuyerActivityModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserBuyerActivityModel.go) (`UpsertBuyerActivity`) |
| Validation | write | `MandatoryParamsCheckBuyerActivity` — [`UserUtilsMandatory.go:1997`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go), `BuyerActivityMap` |
| Catalog View / Bizfeed generation (consumer) | consumers | [`USER_BUSINESS_FEEDS.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go) (`UserBusinessFeeds`, `workerUserBusinessFeeds`) — Kafka-driven |
| Bizfeed Hide/Unhide write | write | [`UserBizfeedHideUnhideController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserBizfeedHideUnhideController.go) (`UserBizHideUnhideController`), [`UserBizHideUnhideModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserBizHideUnhideModel.go) (`UpsertBizhideunhide`) |
| Validation | write | `MandatoryParamsCheckBizHideUnhide` — [`UserUtilsMandatory.go:784`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go) |
| Recommendation/Action-Items (read, aggregator) | read | [`UserSellerRecommendationModel.go`](../../users-api-go-production/internal/models/users/UserSellerRecommendationModel.go) (`FetchActionItems`, `CallApi`/`CallApi2` HTTP-loopback helpers) |

---

## 2. Routes

| Method | serviceName | Repo | Controller |
|---|---|---|---|
| POST | `BUYER_ACTIVITY_SERVICE` | write | `UserBuyerActivityController` |
| POST | `BIZFEED_TOGGLE_SERVICE` | write | `UserBizHideUnhideController` |
| — | (Kafka-consumed, `CATALOG_VIEW_SERVICE`) | consumers | `workerUserBusinessFeeds` |
| — | (Recommendation/Action-Items service) | read | `FetchActionItems` |

---

## 3. Data Model — Tables

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `clickstream.im_buyer_activity_attr_v2` | csl-cassandra | Raw activity-event log — covers User Activity AND Catalog-View-triggering-events, differentiated by `activity_type_id` | `glusr_id`, `datetime`, `activity_type_id`, `fk_activity_id`, `sno` (UUID), `coordinate_latitude`/`_longitude`/`_accuracy` (event-location-metadata, NOT the same as the Lat/Long feature), `fk_display_title`, `glcat_mcat_id`/`_name`, `group_id`/`_name`, `keyword`, `nob_type`, `product_disp_id`, `referer`, `remote_ip`, `request_url`, `seller_glusr_id`, `state_id`/`_name`, `subcat_id`/`_name`, `location_pref_city_ids`/`_names`, `mcat_ids`/`_names` — [`UserBuyerActivityModel.go:63-65`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserBuyerActivityModel.go) |
| `glusr_usr_biz_feeds` | cslpg | Processed, supplier-facing feed-items (Bizfeed) generated from activity-events | `fk_glusr_usr_id`, `biz_feed_activity_glusr_id` (the buyer), `biz_feed_date`, `biz_feed_is_hidden` — [`UserBizHideUnhideModel.go:38`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserBizHideUnhideModel.go) |

**Note on physical-DB split**: raw-activity lives in **Cassandra** (high-write-volume
clickstream-style data), while the processed/derived Bizfeed lives in **Postgres**
(`cslpg`) — a classic write-optimized-ingest vs read-optimized-serving split.

---

## 4. Business Rules & Validation (code se)

### Buyer/User Activity write

1. **Gateway allowlist**: `GLADMIN`, `LEAP` — narrow, since this is mostly a
   machine-to-machine clickstream-ingestion endpoint (`VALIDATION_KEY` is actually
   optional here — if absent, `gate=""` and the request still proceeds, unlike almost
   every other controller in this codebase which hard-rejects on missing/invalid
   validation-key).
   [`UserBuyerActivityController.go:46-51`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserBuyerActivityController.go)
2. **Mandatory**: `glusr_id`, `datetime`, `activity_type_id` (per
   `MandatoryParamsCheckBuyerActivity`).
3. **Every column in `BuyerActivityMap` gets a type-appropriate default if absent from
   the request** (`0` for numbers, `[]` for list-types, `""` for strings) — the
   Cassandra insert always supplies all 32 columns, never partial.
   [`UserBuyerActivityModel.go:36-61`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserBuyerActivityModel.go)
4. **`list<int>`/`list<string>`-typed fields are JSON/comma-parsed from the request**
   into actual Cassandra list-collections.
5. **Insert uses parameterized Cassandra CQL with a server-generated `uuid()`** for the
   `sno` column — no client-supplied primary-key.

### Catalog View / Bizfeed generation (consumer)

6. **Kafka-consumer, `serviceName="CATALOG_VIEW_SERVICE"`**, connects to `cslPg`,
   `meshPg`, AND a Redis instance (`bizfeedRedis`) — three separate data-stores touched
   per message, confirming this is the actual "generate the feed-item" processing-stage
   downstream of the raw-activity-write.
   [`USER_BUSINESS_FEEDS.go:23-56`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go)
7. **Full per-message transformation-logic wasn't traced beyond the connection-setup in
   this pass** — `workerUserBusinessFeeds`'s body (message-processing, Redis-usage
   pattern, exact `glusr_usr_biz_feeds` insert-shape) is flagged as an Open Question for
   deeper follow-up if needed.

### Bizfeed Hide/Unhide write

8. **Gateway allowlist**: `GLADMIN`, `SELLERMY`, `BUYERMY`, `MAPI`.
9. **`visibility_value="0"` is translated to `NULL`** before the update — i.e., `0`
   means "don't set a hidden-value" (clear it), any other value is passed through
   as-is (presumably `1` for "hidden").
   [`UserBizHideUnhideModel.go:13-21`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserBizHideUnhideModel.go)
10. **Match-key for the update is a 3-part composite**: `(fk_glusr_usr_id,
    biz_feed_activity_glusr_id, biz_feed_date::date formatted as yyyymmdd)` — a
    supplier can only hide/unhide a feed-item for a specific buyer-on-a-specific-day,
    not by a direct row-ID.

### Recommendation (Action Items)

11. **`FetchActionItems` is a parallel HTTP-loopback-fan-out**: fires up to 5 concurrent
    goroutine-calls (`CallApi`/`CallApi2`) to `otherdetails_service`,
    `productlisting_service`, `unread_msg`, `bizfeed_service`, and conditionally
    `blcredit_service` (skipped if `is_free=="Y"`) — same HTTP-loopback pattern seen
    across Rating Usefulness, Trust Verification, Update With OTP, App-Rating Feedback,
    but here used for **read-aggregation** rather than a write-delegation.
    [`UserSellerRecommendationModel.go:18-50`](../../users-api-go-production/internal/models/users/UserSellerRecommendationModel.go)
12. **Each source is tagged with a `priority` and `type`** (e.g. `PayIMMap`: priority 1,
    type "Monetization"; `BLCreditMap`: priority 2, type "Engagement") — suggesting the
    combined response is presented to the supplier in a priority-ranked,
    categorized "action items" list.

---

## 5. RabbitMQ / Kafka / Redis

| Channel | Publisher | Consumer | Purpose |
|---|---|---|---|
| Kafka topic (env-configured `sub_topic`) | Not traced in this pass — likely published either directly from the Cassandra-write-path or via a separate CDC/streaming-bridge not visible in these 3 repos | [`USER_BUSINESS_FEEDS.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go) (`workerUserBusinessFeeds`) | Transforms raw activity into `glusr_usr_biz_feeds` entries |
| `bizfeedRedis` | consumer-internal | consumer-internal | Caching/dedup layer for feed-generation (exact usage not traced) |

**No RabbitMQ found in this specific pipeline** — unlike most other features in this KT
series, Buyer Activity → Bizfeed uses **Kafka**, consistent with its
high-throughput-clickstream nature.

---

## 6. End-to-End Technical Flow

```
Buyer (browsing/searching)
    │
    ▼
[API — write]  POST serviceName=BUYER_ACTIVITY_SERVICE
                {glusr_id, datetime, activity_type_id, keyword, glcat_mcat_id,
                 seller_glusr_id, coordinate_latitude/longitude (event-metadata), ...}
    │  UserBuyerActivityController.go — Gateway(GLADMIN/LEAP, optional)
    │  MandatoryParamsCheckBuyerActivity() → LengthAndTypeValidations_v3()
    ▼
UpsertBuyerActivity()
    │  INSERT INTO clickstream.im_buyer_activity_attr_v2 (32 columns, defaults filled)
    ▼
[DB — csl-cassandra]
    │
    ▼  (event flows to Kafka — exact publish-point not traced)
[Kafka topic]
    ▼
[CONSUME]  workerUserBusinessFeeds()  — cslPg + meshPg + bizfeedRedis
    │  generates/updates glusr_usr_biz_feeds rows
    ▼
[DB — cslpg]  glusr_usr_biz_feeds

Supplier (viewing/managing feed)
    │
    ▼
[API — write]  POST serviceName=BIZFEED_TOGGLE_SERVICE  {glusrid, buyerid, datetime,
                visibility_value}
    │  UserBizHideUnhideController.go — Gateway(GLADMIN/SELLERMY/BUYERMY/MAPI)
    ▼
UpsertBizhideunhide()
    │  UPDATE glusr_usr_biz_feeds SET biz_feed_is_hidden=$1
    │  WHERE (fk_glusr_usr_id, biz_feed_activity_glusr_id, date) match
    ▼
[DB — cslpg]

Supplier (viewing Recommendation dashboard)
    │
    ▼
[API — read]  Recommendation/Action-Items service
    │  FetchActionItems() — parallel HTTP-loopback fan-out
    ├─ otherdetails_service
    ├─ productlisting_service
    ├─ unread_msg
    ├─ bizfeed_service      ← reads glusr_usr_biz_feeds (via its own service)
    └─ blcredit_service (conditional, skipped for free-tier)
    ▼
Combined, priority-ranked "action items" response
```

---

## 7. Optimization Scope — DB Response-Time Contribution

### Low impact / good practice already present

1. **Cassandra for raw-clickstream, Postgres for processed-serving-data** — an
   appropriate storage-split for this workload-shape (high-write-throughput vs
   query-friendly-reads).
2. **Recommendation-aggregator uses parallel goroutines**, not sequential HTTP-calls —
   already optimized for the common "fan out, wait for all" pattern.

### Medium impact

3. **Recommendation-aggregator's overall latency is bounded by its slowest
   dependency** (5 parallel calls, but the response presumably waits for all via
   `wg.Wait()`) — if `bizfeed_service` or any other dependency is slow, the entire
   dashboard-load suffers; a timeout/partial-response strategy (show what's ready,
   backfill the rest) could improve perceived-performance if not already present
   (not confirmed either way in this pass).

### Not confirmed — needs follow-up

4. **The exact Cassandra→Kafka publish-mechanism wasn't traced** — if it involves
   polling/CDC rather than a direct application-level publish, that could be a
   meaningful latency/consistency factor between "activity happened" and "feed-item
   appears," but this wasn't visible in the 3 repos covered by this KT series.

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora Bizfeed & Recommendation flowchart yahan dekho](https://lucid.app/lucidchart/92805de5-d226-4026-9a9d-7397d3af27df/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **Buyer Activity write doesn't hard-require a valid `VALIDATION_KEY`** — unlike
   almost every other endpoint in this codebase; if `VALIDATION_KEY` is absent, `gate`
   is just empty-string and the request proceeds anyway. Worth confirming this is
   intentional for a high-volume clickstream-ingest endpoint (trusted internal callers
   only) rather than an oversight.
2. **Hide/Unhide's composite match-key (user+buyer+date) means multiple feed-items on
   the same day for the same buyer can't be independently hidden** — if a supplier had
   two distinct interactions with the same buyer on the same day, hiding one would hide
   both (or neither), since there's no item-level unique-ID in the WHERE clause.
3. **`workerUserBusinessFeeds`'s actual transformation-logic wasn't traced in this
   pass** — the exact rules for what raw-activity becomes a Bizfeed-entry (and how
   Redis is used) would need a deeper follow-up read.
4. **Kafka-topic's producer isn't visible in these 3 repos** — the write-path
   (`UpsertBuyerActivity`) only writes to Cassandra directly; how that becomes a Kafka
   message for `CATALOG_VIEW_SERVICE` to consume isn't traceable from this codebase
   alone (likely a Cassandra-CDC/Debezium-style bridge, or a separate publisher not in
   scope).

---

## 10. Open Questions

1. What triggers the Kafka message that `workerUserBusinessFeeds` consumes — direct
   publish from the write-API, or a CDC-style bridge off Cassandra?
2. What is `bizfeedRedis` used for exactly inside the consumer (cache, dedup,
   rate-limiting)?
3. Does the Recommendation-aggregator have a timeout/partial-response strategy if one
   of its 5 parallel dependencies is slow or fails?
4. `activity_type_id`'s full value-mapping (which values mean "Catalog View" vs general
   "User Activity" vs other types) — not found as an enum/reference in this codebase.
5. Live DB schema verification — is doc ne sirf Go SQL/CQL strings jo imply karti hain
   wahi reflect kiya hai.

---

## See also

- [`Bizfeed_And_Recommendation_Business_Doc.md`](./Bizfeed_And_Recommendation_Business_Doc.md) — product perspective
- [`../Location Update KT/Location_Update_Technical_Doc.md`](../Location%20Update%20KT/Location_Update_Technical_Doc.md) —
  unrelated Lat/Long feature (different table, different write-path)

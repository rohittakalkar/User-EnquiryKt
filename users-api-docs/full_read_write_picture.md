# Full Read + Write Picture — users domain across 3 repos

Three sibling repos under `c:\IM API Repos\users-api-go-production\` together implement the
users domain:

| Repo | Role | Docs |
|---|---|---|
| `users-api-go-production` | **Read API** — every controller is read-only (SELECT/stored-function calls) | [`users_api_database_reference.md`](./users_api_database_reference.md) |
| `service-api-go-production` | **Write API** — INSERT/UPDATE/DELETE for the users domain | [`service_api_write_reference.md`](./service_api_write_reference.md) |
| `user-temp-consumers-production` | **Async consumers** — one deployed instance per RabbitMQ/Kafka queue, processing what the write API publishes | [`user_consumers_reference.md`](./user_consumers_reference.md) |

This came together from a single observation while auditing the read API: **every write
endpoint we found delegated its actual persistence out** — to an external "WAPI" HTTP call
or a RabbitMQ publish. That observation turned out to be structural, not incidental: this
is a genuine CQRS-flavored split, where reads, writes, and async side-effects live in three
separately deployed services.

---

## Architecture at a glance

```text
                         ┌─────────────────────────────┐
   Client (app/web) ──▶  │  users-api-go-production      │  ──▶ Postgres/Cassandra/BigQuery
                         │  (READ API, 50 controllers)   │      (read-only queries)
                         └─────────────────────────────┘
                                      │
                                      │ writes go to a DIFFERENT service
                                      ▼
                         ┌─────────────────────────────┐
   Client (app/web) ──▶  │  service-api-go-production    │  ──▶ Postgres/Cassandra
                         │  (WRITE API, 49 controllers)  │      (INSERT/UPDATE/DELETE)
                         └─────────────────────────────┘
                                      │
                                      │ utils.PushToQueue(SERVICENAME, data)
                                      ▼
                         ┌─────────────────────────────┐
                         │        RabbitMQ / Kafka        │
                         └─────────────────────────────┘
                                      │
                          one deployed instance per queue
                                      ▼
                         ┌─────────────────────────────┐
                         │ user-temp-consumers-production │ ──▶ Postgres/Cassandra/Redis
                         │ (76 workers, internal/Workers/)│      (replica sync, aggregates,
                         └─────────────────────────────┘       notifications, moderation)
                                      │
                     often republishes / calls WAPI again ──▶ back into service-api or
                                                                 other consumers (fan-out)
```

Reads never see the write API's fresh data through a shared connection — they see whatever
the consumers have synced into the tables the read API queries. This is why the read-side
`users_api_database_reference.md` found tables like `glusr_rating_aggregate` that are
**maintained entirely by a consumer**, not by any write endpoint directly.

---

## Worked example: `supplierrating` end to end

This is the exact endpoint we set out to explore hands-on in pgAdmin. Here's the full loop,
now that all three repos are documented.

### 1. Write — a buyer submits a rating

`POST /supplierrating` on **service-api-go-production**
([`UserSupplierRatingController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSupplierRatingController.go) → [`UserSupplierRatingModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSupplierRatingModel.go), DB `meshpg`):

- INSERT `GLUSR_RATING` — the main rating row. **`GLUSR_RATING_DISPLAY_STATUS` is hardcoded
  to `-2`** (pending review) — a brand-new rating is never immediately visible.
- INSERT `GLUSR_RATING_DETAILS` — one row per influence-parameter thumbs up/down the buyer
  selected.
- INSERT `GLUSR_RATING_DOCS` — one row per uploaded review image.
- Publishes to RabbitMQ: `SERVICENAME=SUPPLIER_FEEDBACK` (→ an LMS enquiry payload).

### 2. Moderation — content gets checked before it can go live

`USER_SUPPLIER_RATING_BANNED` queue → [**UserSuppRatingBanned**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SUPPLIER_RATING_BANNED.go) consumer
(`user-temp-consumers-production`):

- Calls CBS banned-keyword API, then an ML banned-keyword fallback, then a PII-detection API
  against the rating comment.
- If anything is flagged: calls `rating_write_service` (a WAPI endpoint, back into the write
  API) to set `SUPP_COMM_DISPLAY`/`BUYER_COMM_DISPLAY`/`IS_SUSPECTED_PRODUCT`, suppressing
  the offending content.
- If PII was found: also publishes to a "suspected" queue for separate review.

### 3. Review/approval — an admin or automated flow updates display status

Eventually something calls `POST /supplierrating` again with `UPDATE_FLAG=U` (either an
admin via `IS_ADMIN=1`, or an automated consumer like [`UserRatingBSMatch`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_BS_MATCHMAKING.go) after a
matchmaking check). This is where `GLUSR_RATING_DISPLAY_STATUS` actually flips to visible
(or stays suppressed). Two RabbitMQ pushes fire on this path:

- `SERVICENAME=NEW_COMPANY` → queue `user.rating_aggregate`
- `SERVICENAME=SUPPLIER_RATING_NEW` → queue `user.ratingnotification.*`

**Important nuance found during the trace**: the actual `QueueFuncMap` key that's wired to
the aggregate-maintaining worker is `USER_RATING_AGGREGATE`, and the notification worker is
bound to `USER_RATING_NOTIFICATION`/`USER_RATING_NOTIFICATION_PG` — these are the functional
targets of the `NEW_COMPANY`/`SUPPLIER_RATING_NEW` service names via the `SERVICENAME →
queue` mapping in [`pkg/utils/rabbitmq.go`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go) (not read in this pass), so the naming doesn't
match 1:1 between the write API's `SERVICENAME` strings and the consumer's queue name — this
is normal for this codebase (see the `USER_LATLONG_VERIFICATION` naming mismatch below too)
but worth knowing if you're ever tracing a flow by grepping for matching strings across
repos.

### 4. Aggregate maintenance — this is what fills `glusr_rating_aggregate`

`USER_RATING_AGGREGATE` queue → [**UserRatingAggregate**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_AGGREGATE.go) consumer:

- Runs a **live re-aggregation query** over `glusr_rating` (filtered
  `glusr_rating_display_status > -1` — so still-suppressed ratings from step 2 are excluded)
  joined with `glusr_rating_details`, computing one/two/three/four/five-star counts, average
  rating (via `fn_get_wt_avg_rating`), and influence-param counts.
- **Upserts** `glusr_rating_aggregate` (`ON CONFLICT (fk_glusr_id) DO UPDATE`) — or **deletes**
  the row entirely if the supplier now has zero eligible ratings.
- Also conditionally logs to `GLUSR_RATING_LOG`.
- Republishes to `COMP_RATING_QUEUE` for company-side aggregation.
- **This worker is purely event-driven — there is no cron/scheduled job that populates
  `glusr_rating_aggregate`.** It only runs when a message arrives on this specific queue.

`USER_RATING_ARCHIVE` (a separate consumer that trims old ratings per buyer/supplier pair
down to the most recent one) **independently re-triggers this same aggregate recomputation**
after archiving — so there are two distinct code paths that can update
`glusr_rating_aggregate`, not one.

### 5. Notification — the supplier finds out

`USER_RATING_NOTIFICATION` queue → [**UserRatingNotify**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_NOTIFICATION.go) consumer:

- Fetches the buyer's name/company/mobile from `GLUSR_USR` (read-only).
- Calls `notification_api` to push an app/SMS notification to the **supplier**: "X has
  reviewed you," with a deep link and click-to-call.
- Writes nothing to any database — pure notification dispatch.
- Skips entirely if this is an admin action, a supplier reply (not a new rating), or was
  triggered by the matchmaking-check consumer (loop-guard via `CALLEDFROM` tag).

### 6. Read — the buyer/supplier or any caller reads the result back

`GET /wservce/users/supplierrating/*params` on **users-api-go-production**
([`SupplierRatingController`](../internal/controllers/UsersControllers/SupplierRatingController.go) → [`UserSupplierRatingModel.go`](../internal/models/users/UserSupplierRatingModel.go), same `mesh` DB):

- Reads `glusr_rating`, `glusr_rating_details`, `glusr_rating_aggregate`, `glusr_usr`,
  `glusr_rating_docs` — everything steps 1-4 wrote.
- Uses the "View More"-style pagination we documented earlier (client sends a running
  count, server computes OFFSET/LIMIT from it).

### Why `glusr_rating_aggregate` was empty on the dev DB

We found this during the pgAdmin exploration: `glusr_rating` had 34 rows, but
`glusr_rating_aggregate` had 0. Now it's explained — `UserRatingAggregate` only fires from a
live RabbitMQ message. If those 34 rows were seeded directly into Postgres (a data dump,
manual insert, or migration) rather than created via `POST /supplierrating` on the real
write API, **no message was ever published**, so the aggregate table was never populated.
This isn't a bug in either service; it's a property of purely event-driven aggregation with
no reconciliation/backfill job for this specific table (contrast with
[`McatDetailsBackfillingScript`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/McatDetailsBackfillingScript.go)/[`gst_tact_cron_sync`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/gst_tact_cron_sync.go)/[`TEST_DB_SCRIPT`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/TEST_DB_SCRIPT.go), which *are* one-off
backfill scripts for other tables — no equivalent exists for `glusr_rating_aggregate`).

**Practical implication for the hands-on session**: if we want to see a realistic
`glusr_rating_aggregate` row in pgAdmin, we have two options — (a) submit a rating through
the real write API (`POST /supplierrating` on `service-api-go-production`) and let the
consumer chain populate it naturally, or (b) manually run the same aggregation query
`UserRatingAggregate` uses, directly in pgAdmin's Query Tool, as a one-off exercise.

---

## Other confirmed read → write → consume loops

Besides supplier rating, we can now trace these end-to-end too:

| Domain | Read (users-api) | Write (service-api) | Consumer(s) |
|---|---|---|---|
| Lat/long verification | — | `LatLongVerifyController` enqueues `LAT_LONG_VERIFICATION` (RabbitMQ) | [`UserLatLongVerification`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_LATLONG_VERIFICATION.go) (bound to queue key `USER_GST_LAST_MODIFIED_DL` — a naming mismatch worth knowing) |
| Feedback / low-rating alert | [`UserFeedback`](../internal/controllers/UsersControllers/UserFeedbackController.go) read | [`UserFeedbackController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserFeedbackController.go) writes `MY_FEEDBACKS`, pushes `USER_FEEDBACK` | [`UserRelatedSendMail`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RELATED_SEND_MAIL.go) sends the actual email |
| User creation | [`UserDetailController`](../internal/controllers/UsersControllers/UserDetailController.go) reads `glusr_usr` | [`UserAddController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserAddController.go) writes `GLUSR_USR` (mesh+auth, non-atomic) | [`UserCentralizedQueue`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_CENTRALIZED_QUEUE.go)/[`UserUpsertAdditional`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_ADDITIONAL.go)/[`UserUpsertConsumers`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_CONSUMERS.go) fan out to ~15 replica DBs, blacklist checks, welcome flows |
| Privacy settings | [`SettingController`](../internal/controllers/UsersControllers/SettingController.go) reads `GLUSR_USR_PRIVACY_SETTING` | [`UserDetailsController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDetailsController.go) (`type=PrivSetting`) writes it, pushes `USER_PRIVACY_SETTING` | [`UserPrivSettImsdb`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_IMSDB.go)/[`UserPrivSettAlertPg`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_ALERTPG.go) sync replicas |
| GST/company registration | [`UserOtherDetailController`](../internal/controllers/UsersControllers/UserOtherDetailController.go) reads `GLUSR_USR_COMP_REGISTRATIONS` | `UserDetailsController` (`type=CompRgst`) writes it | [`UserGstDetails`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_GST_DETAILS.go)/[`UserGstLastModified`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_GST_LAST_MODIFIED.go)/[`GstLastModifiedBuyerProfile`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_GST_LAST_MODIFIED_BUYERPROFILE.go)/[`GstLastModifiedTrust`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_GST_LAST_MODIFIED_TRUSTPG.go) sync 4+ replica DBs and trigger BI verification |
| Trust seal | [`ActionTrustSealController`](../internal/controllers/UsersControllers/TrustSealController.go)/[`TrustsealDetailController`](../internal/controllers/UsersControllers/TrustsealDetailController.go) read `TRUSTSEAL`/`TRUSTSEAL_COMPANY` | [`UserTrustsealController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTrustsealController.go) writes them | (no dedicated consumer found in this audit — writes appear synchronous only) |
| Visiting card | [`ActionVcardDisplay`](../internal/controllers/UsersControllers/VcardDisplayControllers.go) reads `STS_COMP_VISIT_CARD` | [`VisitingCardController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/VisitingCardController.go) writes it | [`UserVisitingCardApproval`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VISITINGCARD_APPROVAL.go) (the only worker using an explicit SQL transaction) |

---

## Cross-repo naming/consistency issues found

Worth knowing before tracing any *other* flow the same way:

1. **Queue name ≠ file/function name, sometimes.** [`USER_LATLONG_VERIFICATION.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_LATLONG_VERIFICATION.go)'s function
   is actually bound to `USER_GST_LAST_MODIFIED_DL` in `QueueFuncMap`. Always grep
   [`internal/Router/router.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Router/router.go)'s `QueueFuncMap`, never assume the filename is the queue name.
2. **`SERVICENAME` (write API) and queue key (consumer) aren't always the same string** —
   they're related through a `SERVICENAME → queue` lookup table in
   [`pkg/utils/rabbitmq.go`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go) (write API side) that we haven't fully mapped in this audit.
   When tracing a new flow, confirm the actual queue by reading that lookup, not just by
   matching strings.
3. **Symmetric dead code across repos**: [`EcomSubscriptionDetailsController`](../internal/controllers/UsersControllers/EcomSubscriptionDetailsController.go) (read) and
   [`UserEcomSubscriptionController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserEcomSubscriptionController.go) (write) are both hardcoded to return "decommissioned" —
   confirms the feature was retired cleanly on both sides, not a half-finished removal.
4. **Multiple consumers can write the same logical target from different trigger paths** —
   `glusr_rating_aggregate` is written by both `UserRatingAggregate` (direct) and
   [`UserRatingArchive`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_ARCHIVE.go) (indirectly, via the same recomputation). `GLUSR_USR_LOGO` is
   synced by three near-identical workers targeting three different Postgres instances
   ([`USER_COMPSYNC_LOGO_IMSDB`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_COMPSYNC_LOGO_IMSDB.go), [`USER_COMP_SYNC_LOGO`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_COMP_SYNC_LOGO.go), [`USER_LOGO_LMSPG`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_LOGO_LMSPG.go)) — same for
   negative-mcat sync. If you're debugging a replica lag issue, check all of them, not just
   the one with the most obvious name.

---

## Hands-on next step (pgAdmin)

Given the trace above, a good hands-on sequence in pgAdmin against the `mesh` database on
`35.200.136.127` would be:

1. Run the read-side query from `SupplierRatingController` (already in
   `users_api_database_reference.md`) to see the current empty-aggregate state.
2. Manually run the `UserRatingAggregate` consumer's aggregation query for one
   `fk_glusr_supplier_id` present in the 34 `glusr_rating` rows, and compare the result
   against what the read API would return once that row exists.
3. Optionally, insert a test row shaped like `UserSupplierRatingController`'s INSERT (same
   columns, on a non-critical dev supplier ID) to see the full loop's *data shape*, though
   actually triggering the RabbitMQ chain would require calling the real write API, not just
   SQL.

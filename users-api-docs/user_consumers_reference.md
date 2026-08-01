# user-temp-consumers-production — Async Consumer Reference (Users Domain)

Documents every worker in
`c:\IM API Repos\users-api-go-production\user-temp-consumers-production\user-temp-consumers-production\internal\Workers\`
(76 files). This is the third leg of the users-domain architecture: reads happen in
`users-api-go-production`, writes happen in `service-api-go-production`, and this repo
processes the RabbitMQ/Kafka messages those writes emit — updating replica databases,
sending notifications, running content moderation, and (critically) maintaining derived/
aggregate tables like `glusr_rating_aggregate`.

**How it's deployed**: `cmd/main.go` reads env var `CONSUMERNAME`, and
`internal/Router/router.go`'s `QueueFuncMap` (a `map[string]func(string)`) looks up which
worker function handles that name. **Each deployed consumer instance handles exactly one
queue** — the queue name is the map key.

**Companion docs**: [`users_api_database_reference.md`](./users_api_database_reference.md)
(reads), [`service_api_write_reference.md`](./service_api_write_reference.md) (writes),
[`full_read_write_picture.md`](./full_read_write_picture.md) (unified picture + the
`supplierrating` worked example).

---

## Cross-cutting patterns

- **Not every file is a real queue consumer.** Three are effectively standalone scripts
  wired into `QueueFuncMap` for deployment convenience but don't consume live messages:
  `McatDetailsBackfillingScript.go` (not in `QueueFuncMap` at all — pure standalone),
  `SYNC_SCRIPT.go` (reads a static `data/glid47K.csv`, sleeps 500 minutes), `gst_tact_cron_sync.go`
  and `TEST_DB_SCRIPT.go` (both read local CSV files and sleep for hours). `IntializeMsgBroker.go`
  isn't a worker at all — it's the shared RabbitMQ/Kafka consume-loop infrastructure every
  other worker uses.
- **Delegation vs. direct write**: many workers don't write to a database directly — they
  call a downstream "WAPI" HTTP service (`utils.CallWapiService`) which performs the actual
  write in a different service, or they republish to another queue. Grep for "Tables
  written: none" below to find pure orchestration/relay workers.
- **Fan-out hubs**: `USER_CENTRALIZED_QUEUE` and `USER_UPSERT_ADDITIONAL` are the two
  biggest "central dispatcher" workers — they receive a generic GLUSR_USR change event and
  branch into dozens of conditional sub-processes (banned-content check, replica sync,
  blacklist, notifications, etc.), each independently retried on failure.
- **Content moderation gates**: `USER_PROFILE_BANNED_DETECT`, `USER_ATTR_BANNED_CONTENT_CHECK`,
  `USER_CENTRALIZED_QUEUE_BANNED`, and `USER_SUPPLIER_RATING_BANNED` all call banned-keyword/
  PII-detection APIs before letting content propagate, scrubbing flagged fields via WAPI
  calls back into the write API.
- **Self-healing resync pattern**: several replica-sync workers (`USER_ADDT_CONTACT_TRUSTPG`,
  `USER_BANK_DETAILS_TRUSTPG`, `USER_UPSERT_CONSUMERS`) fall back to a full row resync from
  `meshPg`/`meshPgLB` whenever an UPDATE affects 0 rows — self-healing against out-of-order
  or missing replica rows.

---

## Workers

### Debezium / CDC sync
- [**DebeziumSyncScript_GlusrUsr**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/DebeziumSyncScript_GlusrUsr.go) — queue `GLUSR_USR_DEBEZIUM_SYNC` (Kafka). Delete+insert full-row replace of `GLUSR_USR` into one of 11 destination DBs (Postgres or Oracle), selected via env `db_name`.
- [**UserCassUpdate**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_CASS_UPDATE.go) — queue `USER_CASS_UPDATE` (Kafka). Despite the name, writes GLUSR_USR fields via WAPI call, not direct Cassandra; publishes audit trail to Kafka `glusr_cass_update_log`.
- [**UserUpsertCsl**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_CSL.go) — queues `USER_UPSERT_CSL`/`_BULK`. Writes `clickstream.GLUSR_USR` in **Cassandra** (config key `csl`) — the CSL/clickstream denormalized copy.
- [**UserUpsertConsumers**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_CONSUMERS.go) — shared handler for `USER_UPSERT_ALERTPG`/`_TRUSTPG`/`_ENQUIRYPG`/`_LMSPG_FEDRATED`/`_BLDISPLAY` (+bulk variants) and several more wired via `dbNameString`. Writes `glusr_usr` into whichever DB the queue name targets; self-heals via `MeshSync` fallback.

### Company/logo/profile sync (3 near-duplicate targets)
- [**UserCompsyncLogoImsdb**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_COMPSYNC_LOGO_IMSDB.go) (`USER_COMPSYNC_LOGO_IMSDB` → searchPg), [**UserCompSyncLogo**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_COMP_SYNC_LOGO.go)
  (`USER_COMP_SYNC_LOGO` → meshPg), [**UserLogoLmsPg**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_LOGO_LMSPG.go) (`USER_LOGO_LMSPG` → lmsPg) — all three
  sync `GLUSR_USR_LOGO` with identical upsert logic against three different Postgres
  instances. The IMSDB and COMP_SYNC variants requeue to a `_FAIL` queue on error; the
  LMSPG variant just Nacks.
- [**UserProfileImsdb**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PROFILE_IMSDB.go) — `USER_PROFILE_IMSDB` → searchPg, whitelisted-column INSERT/UPDATE
  on `GLUSR_USR_PROFILE`.
- [**UserCompregImsdb**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_COMPREG_IMSDB.go) — `USER_COMPREG_IMSDB` → searchPg, syncs IEC code on
  `GLUSR_USR_COMP_REGISTRATIONS`.

### GST / company registration
- [**UserGstDetails**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_GST_DETAILS.go) — `USER_GST_DETAILS`. Complex multi-branch: bank verification, GST
  detail update (`GLUSR_GST_DETAILS` on meshPg+trustPg), other-company-attribute
  verification, all fanning into BI matchmaking APIs and `comp.sync.<glid%20>` republish.
- [**UserGstDetailsBulk**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_GST_DETAILS_BULK.go) — `USER_GST_DETAILS_BULK` (Kafka). Same logic as above, bulk-sourced.
- [**UserGstLastModified**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_GST_LAST_MODIFIED.go) — `USER_GST_LAST_MODIFIED`. Updates `GLUSR_USR_COMP_REGISTRATIONS`
  (mainPg) + `GLUSR_GST_DETAILS` (mesh/trust) + RTF master upsert; sends GST-change email;
  three sub-processes run concurrently.
- [**GstLastModifiedBuyerProfile**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_GST_LAST_MODIFIED_BUYERPROFILE.go) — `USER_GST_LAST_MODIFIED_BUYERPROFILE` → syncs GST/PAN/IEC
  onto the BuyerProfile Postgres replica.
- [**GstLastModifiedTrust**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_GST_LAST_MODIFIED_TRUSTPG.go) — `USER_COMPREG_TRUSTPG` → writes `GLUSR_USR_COMP_REGISTRATIONS`
  + `GLUSR_USR_UDYAM_DETAILS` on trustPg.
- [**GstHsnHitory**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/GST_HSN_HISTORY.go) — `GST_HSN_HISTORY` → bulk INSERT into `GL_ALL_MASTER_HISTORY` (audit trail
  for HSN code add/remove, fed by `UserGSTHSNMappingController`'s writes).
- [**gst_tact_cron_sync**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/gst_tact_cron_sync.go) (`GSTScript`) — not a real consumer; CSV-driven reconciliation script
  that republishes to `USER_GST_DETAILS` for records needing re-verification.

### Ratings — the chain behind `supplierrating` (see capstone doc for full trace)
- [**UserRatingNotifyPG**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_NOTIFICATION_PG.go) (`USER_RATING_NOTIFICATION_PG`) — **the actual origin writer**:
  INSERTs new `GLUSR_RATING` rows (dual-write to approvalPg + meshPg/buyerprofilePg), plus
  `GLUSR_RATING_DETAILS`, `GLUSR_RATING_DOCS`, `GLUSR_RATING_REPLY`,
  `GLUSR_RATING_USEFULNESS`. Fed by `rating_write_service` WAPI calls from the write API and
  from several other consumers below (loop-guarded via `CALLEDFROM` tags).
- [**UserRatingAggregate**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_AGGREGATE.go) (`USER_RATING_AGGREGATE`) — **re-derives and upserts
  `glusr_rating_aggregate`** from a live re-aggregation query over `glusr_rating` +
  `glusr_rating_details`, keyed by `fk_glusr_supplier_id`. Purely event-driven — no
  cron/polling. Also republishes to `COMP_RATING_QUEUE`.
- [**UserRatingArchive**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_ARCHIVE.go) (`USER_RATING_ARCHIVE`) — archives old ratings per buyer/supplier
  pair (keeps only the latest) into `glusr_rating_arch`, deletes from `glusr_rating`, and
  **independently re-triggers the same aggregate recomputation** as `UserRatingAggregate`.
- [**UserRatingBSMatch**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_BS_MATCHMAKING.go) (`USER_RATING_BS_MATCHMAKING`) — checks buyer/supplier connection
  status; if unmatched, calls `rating_write_service` to hide the rating (`DISPLAY_STATUS=-1`).
- [**UserRatingEnrich**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_ENRICHMENT.go) (`USER_RATING_ENRICHMENT`) — fetches mcat/product context from
  `latest_lms_api`, writes back via `rating_write_service`.
- [**UserRatingNotify**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_NOTIFICATION.go) (`USER_RATING_NOTIFICATION`) — **no DB write** — pure push-notification
  dispatcher, tells the supplier "X has reviewed you" via `notification_api`.
- [**UserRatingSellerRisk**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_SELLER_RISK.go) (`USER_RATING_SELLER_RISK`) — forwards new-rating events to a
  Kafka seller-risk-scoring topic, no DB write.
- [**UserRatingUsefulness**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_USEFULNESS.go) (`USER_RATING_USEFULNESS`) — thin forwarder to
  `rating_write_service` for helpful/abuse count updates.
- [**UserSuppRatingBanned**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SUPPLIER_RATING_BANNED.go) (`USER_SUPPLIER_RATING_BANNED`) — content moderation: CBS/ML
  banned-keyword + PII checks; if flagged, calls `rating_write_service` to suppress the
  comment display.
- [**SuppRatingToLms**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SUPPLIER_RATING_TO_LMS.go) (`USER_SUPPLIER_RATING_TO_LMS`) — shards ratings out to LMS via
  RabbitMQ `R_MESSAGE_CENTER_BIZFEED_LMS{0-4}`, determining insert-vs-update semantics by
  checking for a recent (<5 min) prior rating between the same pair.

### PNS (phone number service)
- [**UserPnsAuthPg**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PNS_AUTHPG.go) (`USER_PNS_AUTH_PG` → authPg), [**UserPnsImsdb**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PNS_IMSDB.go) (`USER_PNS_IMSDB` →
  searchPg) — near-duplicate `GL_GSM_MASTER` sync against two DBs.
- [**UserPnsDeletionReason**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PNS_DELETION_REASON.go) (`USER_PNS_DELETION_REASON`) — INSERT/DELETE on
  `PNS_DELETION_REASON`.
- [**UserVerifyBankDetails**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VERIFY_BANK_DETAILS.go) (`USER_VERIFY_BANK_DETAILS`) — no DB write; transforms and
  forwards to RabbitMQ `PennyDrop` (bank-verification trigger service).

### Privacy settings
- [**UserPrivSettImsdb**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_IMSDB.go) (`USER_PRIVACYSETTING_IMSDB` → searchPg) — INSERT/UPDATE/DELETE on
  `GLUSR_USR_PRIVACY_SETTING`.
- [**UserPrivSettAlertPg**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_ALERTPG.go) (`USER_PRIVACYSETTING_ALERTPG`) — updates `GLUSR_USR`'s privacy-ID
  CSV list on authPg + `GLUSR_EMAIL_UNSUBSCRIBERS` on alertPg.

### Verification / attributes
- [**UserApprovalAttribute**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_APPROVAL_ATTRIBUTE.go) (`USER_APPROVAL_ATTRIBUTE`) — DELETE+INSERT
  `iil_user_approval_pending` + audit INSERT `GL_ATTRIBUTE_AUDIT_REPORT`; auto GST
  verification/rejection via external matchmaking API.
- [**UserBulkVerification**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BULK_VERIFICATION.go) (`USER_VERIFICATION_BULK_HISTORY`) — writes
  `IIL_VERIFICATION_DETAILS` (MERGE/upsert) + sharded `GL_HISTORY<NN>` audit rows + updates
  `GLUSR_USR` verification-date columns on authPg; fans out to 3 further queues.
- [**UserVerificationTrustPg**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VERIFICATION_TRUSTPG.go) (`USER_VERIFICATION_TRUSTPG`) — no direct DB write; forwards to
  `user_trust_attr_veri_details` WAPI.
- [**UserUpsertVerification**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_VERIFICATION.go) (`USER_UPSERT_VERIFICATION`) — no direct write; calls
  `utils.Verificaton` (WAPI unverify call) for banned/manual-cron-sourced attribute changes.
- [**UserDetailApacheCass**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_DETAIL_APACHE_CASS.go) (`USER_DETAIL_APACHE_CASS`) — no direct DB access despite the
  name; orchestrates unverify + PNS-mapping WAPI calls for additional-contact changes.

### Content moderation / banned-content pipeline
- [**UserCentralizedQueue**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_CENTRALIZED_QUEUE.go) (`USER_CENTRALIZED_QUEUE`) — the central fan-out hub for generic
  GLUSR_USR changes; no DB write itself, routes to `USER_CENTRALIZED_BANNED` (if
  banned-keyword risk) and sharded `user.upsert.<glid%20>` queues.
- [**UserCentralizedQueueBanned**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_CENTRALIZED_QUEUE_BANNED.go) (`USER_CENTRALIZED_BANNED`) — calls Banned API per column,
  scrubs flagged values via WAPI.
- [**UserCentralizedQueueFail**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_CENTRALIZED_QUEUE_FAIL.go) (`USER_CENTRALIZED_QUEUE_FAIL`) — re-checks banned content on
  retry, republishes to sharded queue.
- [**UserAttrBanned**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_ATTR_BANNED_CONTENT_CHECK.go) (`USER_ATTR_BANNED_CONTENT_CHECK`) — banned-keyword checks on address/
  name/company fields; blanks vague company names for FREE sellers.
- [**UserProfileBannedDetect**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PROFILE_BANNED_DETECT.go) (`USER_PROFILE_BANNED_DETECT`) — moderation gate sitting in
  front of profile/template/meta/detail sync queues; scrubs via WAPI or republishes clean
  content onward.
- [**UserChatLmsBlklist**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_CHAT_BLACKLIST_DATA.go) (`USER_CHAT_BLACKLIST_DATA`, Kafka) — matches mobile/email against
  `GL_BLACKLIST_VALUES`, inserts Gladmin blacklist record via WAPI.

### Notifications / mail / SMS
- [**User2MinSmsDelay**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_2MIN_SMS_MAIN.go) (`USER_2MIN_SMS_MAIN`) — sends delayed registration SMS if mobile
  still unverified after 2 min. No DB write.
- [**UserFeedback**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_FEEDBACK.go) (`USER_FEEDBACK`) — INSERT/UPDATE `MY_FEEDBACKS` (approvalPg); sends
  feedback email when non-blank description on mobile modids.
- [**UserBankDetails**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BANK_DETAILS.go) (`USER_BANK_DETAILS`) — sends bank-change email/SMS; no DB write.
- [**UserMailFreqAlert**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_MAIL_FREQUENCY_ALERT.go) (`USER_MAIL_FREQUENCY_ALERT`) — INSERT/UPDATE
  `ETO_BL_ALERT_LIMIT` (alertPg). **Flagged bug**: `defer conn.Close()`s the shared DB
  connection on every message — risk of breaking subsequent messages if the connection is
  reused from a pool.
- [**UserRelatedSendMail**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RELATED_SEND_MAIL.go) (`USER_RELATED_SEND_MAIL`) — **this is the consumer for the
  read-side `UserFeedbackController`'s low-rating notification** (also handles `PAY_NOW`
  emails). No DB write — pure mail dispatch via `utils.UserMail_V2`.

### Matchmaking (buyer-supplier connections)
- [**UserBSMatchMakingKafka**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_KAFKA.go) (`USER_BS_MATCHMAKING`, Kafka) and [**UserBsMatchLMS**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_LMS.go)
  (`USER_BS_MATCHMAKING_LMS`) — both call stored proc `sp_insert_contacts(...)` on
  `GLUSR_CONTACTBOOK_MAPPING` (buyerprofilePg); near-identical logic, different transport.

### Location / geo
- [**UserLatLongVerification**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_LATLONG_VERIFICATION.go) — bound to queue key `USER_GST_LAST_MODIFIED_DL` (naming
  mismatch — the file/function name suggests `LAT_LONG_VERIFICATION`, but that's not the
  actual `QueueFuncMap` key). **This is the consumer side of
  `LatLongVerifyController`'s RabbitMQ enqueue** from the read-side repo. Delegates all
  writes to `utils.LocalizationProcess` (cslPg + meshPg).
- [**UserUpsertCityOthers**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_CITY_OTHERS.go) (`USER_UPSERT_CITY_OTHERS`) — resolves free-text city to a city
  ID via an external LatLong API, applies via WAPI call. No direct DB write.
- [**GlusrUsrShowroomUrl**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/GLUSR_USR_SHOWROOM_URL.go) (`GLUSR_USR_SHOWROOM_URL`) — Cassandra sync of
  `glusr_usr_free_alias`/`_free_url`/`_paid_url` (3 parallel goroutines, insert/update/delete
  diffing).
- [**UserMktpUrl**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_MKTP_URL.go) (`USER_MKTP_URL`) — INSERT/UPDATE `glusr_mktp_url` (buyerprofilePg).
- [**UserSocialContact**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SOCIAL_CONTACT.go) (`USER_SOCIAL_CONTACT`) — UPSERT `GLUSR_USR_SOCIAL_CONTACT` (meshPg).
- [**UserSearchBrwseIntrst**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SEARCH_BROWSE_INTEREST.go) (`USER_SEARCH_BROWSE_INTEREST`) — UPSERT
  `GLUSR_USR_INTEREST_ACTIVITIES` (buyerprofilePg), retries on Postgres deadlock.
- [**UserBuyerDetails**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUYER_DETAILS.go) (`USER_BUYER_DETAILS`) — UPSERT `IIL_BUYER_PROFILE_DETAILS`
  (buyerprofilePg).
- [**UserBusinessFeeds**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go) (`USER_BUSINESS_FEED`, Kafka) — INSERT `glusr_usr_biz_feeds` (cslPg),
  with a Redis-backed "seller-client meeting" skip check.
- [**UserPersonalizationActivity**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PERSONALIZATION_ACTIVITY.go) (`USER_PERSONALIZATION_ACTIVITY`, Kafka) — writes to 4
  Cassandra clickstream tables; backfills anonymous GA-cookie sessions once GLID is known.

### Negative mcat (3 near-duplicate DB targets, same pattern as logo sync)
- [**UserNegMcatImbLPg**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_NEG_MCAT_IMBLPG.go) (`USER_NEG_MCAT_IMBLPG` → alertPg, manual update-then-insert upsert),
  [**UserNegMcatImsdb**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_NEG_MCAT_IMSDB.go) (`USER_NEG_MCAT_IMSDB` → searchPg), [**UserNegMcatMLBL**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_NEG_MCAT_MLBL.go)
  (`USER_NEG_MCAT_MLBL` — pure forwarder to `ETO_REJECTION_MASTER_QUEUE`, no DB at all).

### Bank / financial
- [**UserBankDetailsTrustPg**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BANK_DETAILS_TRUSTPG.go) (`USER_BANK_DETAILS_TRUSTPG`) — INSERT/UPDATE
  `glusr_bank_details` on trustPg, self-heals from meshPg on 0-row UPDATE.
- [**UserUpsertAdditionalGcpIn**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_ADDITIONAL_GCP_IN.go) (`USER_UPSERT_ADDITIONAL_GCP_IN`) — dispatches to
  `utils.Glusr_vbb` (verified-buyer-badge, alertPg) and `utils.Instafinance` (financial flag,
  buyerprofilePg).

### The two "central dispatcher" workers
- [**UserUpsertAdditional**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_ADDITIONAL.go) (`USER_UPSERT_ADDITIONAL`) — 1664-line orchestrator triggered off
  any GLUSR update; dozens of conditional sub-processes (blacklist, big-buyer flag, social
  info, bounce-email, parent/child fraud-disable cascade, WAPI disable sync, PNS
  removal/update, disposable-domain detection, locality resolution, city-change Kafka
  publish) each independently requeue on failure.
- [**SyncScript**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/SYNC_SCRIPT.go) (`SYNC_SCRIPT`) — CSV-driven variant of the centralized-queue banned-content
  re-check logic; not a real consumer (reads a static file).

### Misc / one-off
- [**LegalStatusComputation**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/LEGAL_STATUS_COMPUTATION.go) (`LEGAL_STATUS_COMPUTATION`) — computes legal status code from
  GST 6th-character + company name, applies via WAPI. No direct DB write.
- [**CustomHistory**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_HISTORY.go) (`USER_HISTORY`, Kafka) — thin proxy into stored proc
  `pck_gl_history_sp_add_custom_history`.
- [**UserSellOnImCalling**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SELLONIM_CALLING.go) (`USER_SELLONIM_CALLING`) — 5-stage eligibility gate (verification
  log, FCP priority, dup email, dup GST, dup allocation) before INSERT into
  `iil_supp_verification`.
- [**UserSuspectTagging**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SUSPECT_TAGGING.go) (`USER_SUSPECT_TAGGING`, Kafka) — UPSERT `GLUSR_SUSPECT_TAGGING`
  (buyerprofilePg).
- [**UserVisitingCardApproval**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VISITINGCARD_APPROVAL.go) (`USER_VISITINGCARD_APPROVAL`) — the only worker using an
  explicit SQL transaction (`tx.Begin`/`Commit`/`Rollback`) for a delete+insert pair on
  `IIL_visit_card_APPR_PENDING` + audit INSERT.
- [**FlipsContentSync**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_FLIPS_CONTENT_SYNC.go) (`USER_FLIPS_CONTENT_SYNC`, Kafka) — diffs incoming content against
  `glusr_flips_content_detail`, batches INSERT/UPDATE/DISABLE calls (30 at a time) to
  `flips_content_write_service`.

### Not real consumers (documented for completeness)
- [**McatDetailsBackfillingScript**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/McatDetailsBackfillingScript.go) — standalone one-off backfill script, not in `QueueFuncMap`.
- [**TestDBScript**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/TEST_DB_SCRIPT.go) (`TEST_DB_SCRIPT`/`DB_SCRIPT`) — CSV-driven mobile-number
  de-duplication/reassignment migration script, sleeps 10 hours between runs.
- **gst_tact_cron_sync** (see GST section above) — CSV-driven reconciliation, sleeps 1 hour.
- [**IntializeMsgBroker**](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go) — shared infrastructure (RabbitMQ/Kafka consume loops, dispatch
  tables, graceful shutdown), not a queue handler itself.

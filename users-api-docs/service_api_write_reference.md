# service-api-go-production — Write API Reference (Users Domain)

This is the write-side counterpart to
[`users_api_database_reference.md`](./users_api_database_reference.md) (which documents
`users-api-go-production`, a **read-only** API). Every controller below lives in
`c:\IM API Repos\users-api-go-production\service-api-go-production\service-api-go-production\internal\controllers\UserControllers\`
and performs actual INSERT/UPDATE/DELETE writes for the users domain, backed by models in
`internal/models/UserModels/`.

**Companion docs**: [`users_api_database_reference.md`](./users_api_database_reference.md) (reads),
[`user_consumers_reference.md`](./user_consumers_reference.md) (async consumers),
[`full_read_write_picture.md`](./full_read_write_picture.md) (the unified 3-repo picture and
the worked `supplierrating` example).

---

## Cross-cutting patterns across all 49 controllers

- **Config keys / databases**: `meshpg` (Postgres "mesh" DB — used by ~35 of 49 controllers),
  `authpg` (Postgres, auth-side mirror of GLUSR_USR), `trustpg`, `approvalpg`, `document_pg`,
  `cslpg`, `buyerprofilepg` — all Postgres. `UserBuyerActivityController` is the only
  controller in this set writing to **Cassandra** (`csl-cassandra`).
- **Side-effect mechanism**: `utils.PushToQueue(serviceName, data)` looks up a
  `SERVICENAME → queue/exchange` mapping (`pkg/utils/rabbitmq.go`) and POSTs to a RabbitMQ
  HTTP publish gateway. This is how nearly every successful write notifies downstream
  consumers (see the consumer reference doc for what picks these up).
- **No consistent transaction usage**: only a handful of controllers use real SQL
  transactions (`SocialReviewAccountController`, `SocialReviewContentController`,
  `UserAddController`'s two-DB writes are explicitly *not* transactional together). Most
  writes are single/sequential statements — partial failure is possible in multi-step flows.
- **Dead/decommissioned controllers found**: `UserBlacklistController` (hardcoded
  "deprecated" response, model code unreachable) and `UserEcomSubscriptionController`
  (hardcoded "decommissioned" response) — both mirror dead endpoints already found on the
  read side (`EcomSubscriptionDetailsController`), suggesting these features were retired
  together.
- **Upsert-via-application-logic is common**: many controllers do UPDATE-then-check-rows-
  affected-then-INSERT (`UserImageController`, `UserGSMUpdateController`'s pre-check
  pattern) rather than native `ON CONFLICT`, even where an `ON CONFLICT` upsert would be
  simpler — inconsistent style across the codebase.

---

## Controllers

### [BlacklistValuesController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/BlacklistValuesController.go)
- **Route**: `POST /user/blacklistvalues` | **Write**: UPDATE `GL_BLACKLIST_VALUES` (keyed on `FK_ORIGNAL_GLUSR_ID` or `UPPER(TRIM(GL_BLACKLIST_VALUE))`) | **DB**: meshpg | **Side effects**: none

### [BsMatchMakingController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/BsMatchMakingController.go)
- **Route**: `POST /user/bsmatchmaking` | **Write**: calls stored proc `sp_insert_contacts(...)` (inserts into a contacts table server-side) | **DB**: buyerprofilepg | **Side effects**: Kafka publish `soa-user-bs_matchmaking-csl_glid_logs`

### [IdProofController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/IdProofController.go)
- **Route**: `POST /user/idproof` | **Write**: mixed INSERT/UPDATE on `GLUSR_USR_IDENTITY_PROOFS` (SELECT-then-branch, not native upsert), keyed `FK_GLUSR_USR_ID + FK_IIL_IDENTITY_PROOF_ID` | **DB**: meshpg | **Side effects**: none

### [PnsTxnController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/PnsTxnController.go) (`UserPnsAllocationController`)
- **Route**: `POST /pns` | **Write**: multi-step saga on `GL_GSM_MASTER` (insert/update/random-allocate/bulk-convert by `ACTION` 1-5) | **DB**: meshpg | **Side effects**: HTTP self-call to `user_update_api` (propagates to GLUSR_USR), RabbitMQ `PNS_DELETION` on deletion actions, error mail on failure. Most complex controller in the batch — a saga, not an atomic transaction.

### [PopupDetailsController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/PopupDetailsController.go)
- **Route**: `POST /popupdetails` | **Write**: UPSERT `TRACK_POPUP_MDC` (`ON CONFLICT (fk_glusr_usr_id) DO UPDATE`) | **DB**: meshpg | **Side effects**: none

### [SocialReviewAccountController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/SocialReviewAccountController.go)
- **Route**: `POST /socialreviews/account` | **Write**: UPSERT `glusr_social_review` (`ON CONFLICT (fk_glusr_usr_id, fk_gl_social_media_id, social_location_id) DO UPDATE`), wrapped in an explicit **transaction** that also deactivates sibling location rows | **DB**: meshpg | **Side effects**: none. Requires a pre-existing unique index (deployment prerequisite, not auto-created).

### [SocialReviewContentController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/SocialReviewContentController.go)
- **Route**: `POST /socialreviews/content` | **Write**: batch UPSERT `glusr_social_review_detail` (`ON CONFLICT (fk_glusr_social_review_id, social_platform_review_id) DO UPDATE`) via a **prepared statement inside a transaction** — all-or-nothing for up to 20 items | **DB**: meshpg | **Side effects**: none. Requires a pre-existing "master account" row in `glusr_social_review`.

### [UpdateWithOtpController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UpdateWithOtpController.go)
- **Route**: `POST /user/updatewithotp` | **Write**: none directly — two-factor OTP gate (checks `glusr_usr_otp` on **authpg**, read-only) that delegates the actual `GLUSR_USR` update to a downstream HTTP call | **Side effects**: RabbitMQ `GLUSR_USR` UPDATE message published immediately after OTP1 succeeds (independent of OTP2 outcome); HTTP POST to `user_update_api` after both OTPs succeed.

### [UserAddController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserAddController.go) — the primary "create user" endpoint
- **Route**: `POST /user/add` | **Write**: INSERT `GLUSR_USR` on **meshpg** (`ON CONFLICT DO NOTHING RETURNING ...`), then a separate INSERT `GLUSR_USR` on **authpg** — **not atomic together** | **Side effects**: RabbitMQ INSERT message on mesh-insert success; error mail + `USER_ADDUP_AUTH_FAILQUEUE` publish on authPG-insert failure; on duplicate detection (mobile/email/PNS already exists) makes an **internal HTTP self-call** to merge fields into the existing user instead of inserting. Extremely branchy duplicate-resolution logic.

### [UserAddLatLongController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserAddLatLongController.go)
- **Route**: `POST /user/addlatlong` (also `/instafinance`, same controller) | **Write**: plain INSERT `GLUSR_USR_ADDRESS_LAT_LONG` (always a new history row) | **DB**: cslpg | **Side effects**: none

### [UserAddMechantToVendorController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserAddMechantToVendorController.go)
- **Route**: `POST /user/addmerchanttovendor` | **Write**: INSERT or UPDATE `Glusr_payment_vendor_account` by `action` param | **DB**: meshpg | **Side effects**: none

### [UserAddPaymentTxnDetailsController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserAddPaymentTxnDetailsController.go)
- **Route**: `POST /user/addpaymenttxndetails` | **Write**: INSERT or UPDATE `Glusr_international_txn_detail` by `action` | **DB**: meshpg | **Side effects**: none

### [UserAddPaymentVendorController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserAddPaymentVendorController.go)
- **Route**: `POST /user/addpaymentvendor` | **Write**: INSERT or UPDATE `payment_vendor_master` by `action` | **DB**: meshpg | **Side effects**: none

### [UserBizfeedHideUnhideController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserBizfeedHideUnhideController.go)
- **Route**: `POST /user/bizfeed_hide_unhide` | **Write**: UPDATE `glusr_usr_biz_feeds` (composite key incl. date-formatted match) | **DB**: cslpg | **Side effects**: none

### [UserBlacklistController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserBlacklistController.go) — **dead code**
- **Route**: `POST /user/blacklist` | **Write**: **NONE** — controller unconditionally returns `"This API is deprecated now"` without touching the DB, despite a fully-implemented model function (`UserBlacklistUpdateintoDB`, keyed on `QUERY_APPROVAL_BLACKLIST`) sitting unused behind it.

### [UserBlockUnblockController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserBlockUnblockController.go)
- **Route**: `POST /user/user_block_unblock` | **Write**: UPSERT `USER_BLOCKED_STATUS` (`ON CONFLICT (user_glid, blocked_glid) DO UPDATE`) | **DB**: meshpg | **Side effects**: none

### [UserBuyerActivityController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserBuyerActivityController.go) — only Cassandra writer in this repo
- **Route**: `POST /user/addbuyeractivity` | **Write**: INSERT `clickstream.im_buyer_activity_attr_v2` (fresh row every call, `uuid()` key) | **DB**: `csl-cassandra` | **Side effects**: none

### [UserDetailsController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDetailsController.go) — large multi-purpose endpoint
- **Route**: `POST /details` | **Write**: dispatched by `type` param across ~9 sub-domains — `CompRgst`→`GLUSR_USR_COMP_REGISTRATIONS`, `BankDetails`→`GLUSR_BANK_DETAILS` (+ prime-account clearing UPDATE), `FactSheet`→`GLUSR_USR_FACT_SHEET`, `Franchise`→`PC_CLNT_FEATURE_SECTION_VALUE`, `OtherDetails`→`GLUSR_OTH_REM_DETAIL`, plus `ContactDetails`/`LocDetails`/`LocImage`/`PrivSetting`/`EmpDetails` (four other legacy types explicitly deprecated/rejected) | **DB**: meshpg | **Side effects**: RabbitMQ `USER_PRIVACY_SETTING`/`MAIL_FREQUENCY_ALERT` pushes; HTTP call to `permmaster_api` for GST-override screen-permission checks; disposable-email-domain check; RTF (right-to-forget) field nulling.

### [UserDispositionController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDispositionController.go)
- **Route**: `POST /user/dispositon` | **Write**: plain INSERT `GLUSR_DISPOSITIONS` | **DB**: meshpg | **Side effects**: none

### [UserEcomMappingController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserEcomMappingController.go) (`EcomMappingController`)
- **Route**: `POST /ecom_mapping` | **Write**: UPSERT `GLUSR_ECOM_MAPP` (`ON CONFLICT (ECOM_STORE_USR_REF_ID, FK_ECOM_STORE_MASTER_ID) DO UPDATE`) | **DB**: meshpg | **Side effects**: none. No gateway/validation-key check (unusual for this codebase).

### [UserEcomSubscriptionController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserEcomSubscriptionController.go) — **dead code**
- **Route**: `POST /ecom_subscription_details` | **Write**: **NONE** — hardcoded "decommissioned" response; backing model (`EcomSubscriberModel.go`, handles `ECOM_PLAN`/`ECOM_SUBSCRIBER_DETAILS`/`ECOM_SUBSCRIBER_BILLING_DETAILS`) is fully implemented but orphaned.

### [UserFeedbackController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserFeedbackController.go)
- **Route**: `POST /feedback` | **Write**: dispatched by `FEEDBACK_FLAG` — default → INSERT/UPDATE `MY_FEEDBACKS`; `IMSEARCH_FEEDBACK` → INSERT/UPDATE `IM_SEARCH_FEEDBACK` | **DB**: meshpg | **Side effects**: RabbitMQ `USER_FEEDBACK` push on the `MY_FEEDBACKS` path only.

### [UserFlipsAccountController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserFlipsAccountController.go)
- **Route**: `POST /flips/account` | **Write**: UPSERT `GLUSR_FLIPS_MAP` (token) + INSERT/UPDATE `GLUSR_FLIPS_DETAIL` (meta), dispatched by `MAPPING_TYPE` | **DB**: document_pg | **Side effects**: `SendFlipToComp` → RabbitMQ `FLIPS_WRITE_SERVICE`

### [UserFlipsContentController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserFlipsContentController.go)
- **Route**: `POST /flips/content` | **Write**: batched anti-join INSERT or batched multi-row UPDATE on `GLUSR_FLIPS_CONTENT_DETAIL` | **DB**: document_pg | **Side effects**: same `SendFlipToComp` RabbitMQ push as UserFlipsAccountController

### [UserGSMUpdateController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserGSMUpdateController.go)
- **Route**: `POST /gsm_update` | **Write**: UPDATE `GL_GSM_MASTER.FLAG_IS_AVAILABLE` after two read-only pre-checks (GSM exists+available, not already assigned) | **DB**: meshpg | **Side effects**: none

### [UserGSTHSNMappingController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserGSTHSNMappingController.go)
- **Route**: `POST /gsthsnmapping` | **Write**: INSERT (add) + DELETE (remove) on `GST_TO_HSN_MAPPING` in the same request | **DB**: meshpg | **Side effects**: two RabbitMQ pushes — `GST_TO_HSN_HISTORY` and per-affected-user `GST_TO_HSN_MAPPING_SERVICE`

### [UserImageController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserImageController.go)
- **Route**: `POST /user/image` | **Write**: UPDATE-then-INSERT-fallback on `GLUSR_USR_IMAGE` | **DB**: meshpg | **Side effects**: none

### [UserInstaFinanceController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserInstaFinanceController.go)
- **Route**: `POST /instafinance` | **Write**: UPDATE `COMP_MASTER_FINANCIALS.COMP_MASTER_FINANCIAL_ENABLED` by `COMPANY_CIN` | **DB**: meshpg | **Side effects**: RabbitMQ `INSTAFINANCE` push on success. **Bug flag**: hardcodes `env := "dev"` instead of reading `config.GetEnv()`, so NewRelic tracing is effectively always off for this endpoint.

### [UserKwplController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserKwplController.go)
- **Route**: `POST /kwpl` | **Write**: INSERT/UPDATE/DISABLE/SEARCH_UPDATE on `PL_KWRD`, dispatched by `ACTION` | **DB**: meshpg | **Side effects**: none

### [UserLocationsUpdateController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserLocationsUpdateController.go)
- **Route**: `POST /userlocations/update` | **Write**: INSERT-only (batched, up to worker-pool-of-4) `GLUSR_GEO_ADDT_CONTACT` (`ON CONFLICT DO NOTHING` on lat/long+user) | **DB**: cslpg | **Side effects**: none. Rejects non-insert `flag` values entirely.

### [UserLogoController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserLogoController.go)
- **Route**: `POST /user/logo` | **Write**: INSERT/UPDATE `GLUSR_USR_LOGO`, dispatched by `APPROVAL_ID` presence + `FLAG`/`STATUS` (approve/reject/reset flows) | **DB**: meshpg | **Side effects**: RabbitMQ `COMPANY_LOGO_SERVICE` push on success; unique-constraint violations remapped to a friendly "already exists" success message.

### [UserMarketPlaceUrlController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserMarketPlaceUrlController.go) (`MarketPlaceUrlController`)
- **Route**: `POST /market_place_url` | **Write**: INSERT or UPDATE `GLUSR_MKTP_URL` by `op_flag` | **DB**: meshpg | **Side effects**: none

### [UserMetaController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserMetaController.go)
- **Route**: `POST /meta` | **Write**: UPDATE `PC_CLNT_PCAT` (category/meta fields) | **DB**: meshpg | **Side effects**: RabbitMQ `META_SERVICE` push → feeds `USER_PROFILE_BANNED_DETECT` queue for content moderation

### [UserMsgIntegrationController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserMsgIntegrationController.go)
- **Route**: `POST /user/msg_integration` | **Write**: UPSERT `Glusr_msg_platform_integration` (`ON CONFLICT (fk_glusr_usr_id, fk_msg_platform_id)`) + INSERT `glusr_msg_platform_log` (status-change/scam-spam log) | **DB**: meshpg | **Side effects**: RabbitMQ `MSG_PLATFORM_INTEGRATION` push

### [UserNegMcatController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserNegMcatController.go)
- **Route**: `POST /negative/mcat` | **Write**: INSERT (add) or soft-delete UPDATE (`IS_DELETED=1`) on `NEGATIVE_MCAT_FOR_PRODUCTS` | **DB**: meshpg | **Side effects**: RabbitMQ `USER_NEGATIVE_MCAT` push

### [UserPnsSettingController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserPnsSettingController.go)
- **Route**: `POST /pnssetting` | **Write**: INSERT (`ON CONFLICT DO NOTHING`) / UPDATE / DELETE on `IIL_PNS_SETTING` + stored-proc history call on delete | **DB**: meshpg | **Side effects**: none directly (only DB writes + history proc)

### [UserProfileController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserProfileController.go)
- **Route**: `POST /profile` | **Write**: complex — UPDATE `GLUSR_USR_CLNT_FEATURE_VALUE`, batch sort-order UPDATE + soft-delete UPDATE + INSERT/UPDATE on `GLUSR_USR_PROFILE`, dispatched by profile type | **DB**: meshpg | **Side effects**: RabbitMQ `USER_PROFILE_SERVICE` push → feeds `USER_PROFILE_BANNED_DETECT`

### [UserRatingUsefulnessController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserRatingUsefulnessController.go)
- **Route**: `POST /rating_usefulness` | **Write**: plain INSERT `GLUSR_RATING_USEFULNESS` (duplicate caught as "already exists") | **DB**: meshpg | **Side effects**: RabbitMQ `USER_RATING_USEFULNESS` push with recomputed helpful/abuse counts

### [UserSellonimLogController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSellonimLogController.go)
- **Route**: `POST /user/sellonimlog` | **Write**: INSERT or UPDATE `sellonim_log` | **DB**: **approvalpg** (note: distinct from meshpg) | **Side effects**: RabbitMQ `USER_SELLONIM_LOG` only when `CUSTTYPE_ID == "39"`

### [UserSocialContactsController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSocialContactsController.go)
- **Route**: `POST /user/addsocialcontact` | **Write**: INSERT or UPDATE `glusr_usr_social_contact` | **DB**: meshpg | **Side effects**: none

### [UserSuppVerifyLogController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSuppVerifyLogController.go) (`SuppVerifyLog`)
- **Route**: `POST /supp_verify_log` | **Write**: INSERT / UPDATE / DELETE on `IIL_SUPP_VERIFICATION_LOG` (+ history stored proc on delete) | **DB**: meshpg | **Side effects**: none

### [UserSupplierRatingController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSupplierRatingController.go) — the write side of our hands-on target
See [`full_read_write_picture.md`](./full_read_write_picture.md) for the complete traced
flow. Summary:
- **Route**: `POST /supplierrating` | **DB**: meshpg (same physical `mesh` DB the read-side
  `SupplierRatingController` reads from)
- **INSERT path**: writes `GLUSR_RATING` (main row, display status forced to `-2` pending
  review), `GLUSR_RATING_DETAILS` (one row per influence-parameter thumbs up/down), and
  `GLUSR_RATING_DOCS` (one row per uploaded image).
- **UPDATE path** (`UPDATE_FLAG=U`): updates `GLUSR_RATING` review/display fields, INSERTs
  `glusr_rating_reply` (supplier reply), UPDATEs `GLUSR_RATING_DOCS` review status.
- **Side effects** (up to 7 distinct RabbitMQ pushes depending on branch):
  `SUPPLIER_FEEDBACK` (→ LMS enquiry), `NEW_COMPANY`→`user.rating_aggregate`,
  `SUPPLIER_RATING_NEW`→`user.ratingnotification.*`, `USER_RATING_BANNED`.
- **Validation**: `BUYER_ID == SUPPLIER_ID` rejected; `RATING_VAL` must be 1-5.

### [UserTemplateController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTemplateController.go)
- **Route**: `POST /template` | **Write**: UPDATE `PC_CLNT_PCAT` + INSERT/UPDATE `PC_CLNT_FEATURE_SECTION_VALUE` across many profile-tab sections (largest model file in the batch, ~2000 lines, not fully traced) | **DB**: meshpg | **Side effects**: RabbitMQ `TEMPLATE_SERVICE` push

### [UserTncAcceptanceController.go.go](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTncAcceptanceController.go.go) (literal double-`.go` filename)
- **Route**: `POST /user/tncacceptance` | **Write**: plain INSERT `IIL_TERMS_COND_ACCEPTANCE` | **DB**: meshpg | **Side effects**: RabbitMQ `TNC_ACCEPTANCE`→`SESSION_UPDATE`

### [UserTrustsealController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTrustsealController.go)
- **Route**: `POST /user/trustseal` | **Write**: INSERT/UPDATE `TRUSTSEAL`, UPDATE `TRUSTSEAL_COMPANY`, UPDATE-then-INSERT-fallback `TRUSTSEAL_HISTORY`, dispatched by `TYPE`/`ACTION` | **DB**: meshpg | **Side effects**: none observed. Gateway restricted to `WEBERP` only.

### [UserUpdateController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserUpdateController.go) — largest/most complex controller in the whole write API
- **Route**: `POST /user/update` | **Write**: dynamic UPDATE `GLUSR_USR` (~40+ possible fields: email, mobile, address, GST, listing/approval status) | **DB**: meshpg (+ HTTP call to sync authpg-side record) | **Side effects**: RabbitMQ via `GLUSR_UPDATE_SERVICE`→`USER_CENTRALIZED_QUEUE` (the central fan-out hub — see consumer doc). Runs `FN_CHECK_DUP_GLUSR` DB function for duplicate-value checks before allowing the update; "Right to be forgotten" flow nulls PII and appends `-autodeleted@indiamart.com` to email. Model file is 1500+ lines — not fully traced.

### [UserVerificationController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserVerificationController.go)
- **Route**: `POST /user/verification` | **Write**: idempotent UPSERT `IIL_VERIFICATION_DETAILS` (`ON CONFLICT DO NOTHING`) + dynamic writes to whichever table owns the verified attribute (resolved at runtime via `gl_attribute.gl_attribute_table_name`) + history table insert | **DB**: meshpg (+ authpg via a secondary call site) | **Side effects**: implied RabbitMQ call inside `updateMeshPg` (not fully traced). Re-reads live attribute values before verifying and rejects mismatches.

### [UserVerificationDetailsController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserVerificationDetailsController.go)
- **Route**: `POST /user/userverificationdetails` | **Write**: calls stored function `fn_insert_glusr_attribute_verification(...)` once per primary + secondary attribute — no raw SQL | **DB**: **trustpg** (distinct from meshpg) | **Side effects**: none observed

### [VisitingCardController](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/VisitingCardController.go)
- **Route**: `POST /visiting/card` | **Write**: UPSERT `STS_COMP_NO_VISIT_CARD` (`ON CONFLICT (FK_GLUSR_USR_ID) DO UPDATE`, for the "no card" flag) or UPDATE `STS_COMP_VISIT_CARD` (enrich/approve/design flows, dispatched by `TYPE`/`UPDATE_FLAG`) | **DB**: meshpg | **Side effects**: implied RabbitMQ `VISITING_CARD_SERVICE`→`USER_VISITINGCARD_APPROVAL` (queue map confirms, exact call site not reached in read window)

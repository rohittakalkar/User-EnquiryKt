# Users Domain — Product Stories & Features

A full product-perspective inventory of everything the users domain does, across all three
repos (`users-api-go-production` read API, `service-api-go-production` write API,
`user-temp-consumers-production` async consumers). Organized as **stories** (major product
capabilities) broken into **features** (specific sub-capabilities), each mapped to the APIs
that implement it.

This is a product-level index. For exact tables/columns/queries, see:
[`users_api_database_reference.md`](./users_api_database_reference.md) (reads),
[`service_api_write_reference.md`](./service_api_write_reference.md) (writes),
[`user_consumers_reference.md`](./user_consumers_reference.md) (async consumers),
[`full_read_write_picture.md`](./full_read_write_picture.md) (architecture + worked example),
[`ratings_reviews_product_overview.md`](./ratings_reviews_product_overview.md) (Story 5,
already documented in full detail).

---

## Index of stories

1. Account Creation & Onboarding
2. Profile & Business Identity Management
3. Location Intelligence
4. Trust, Verification & Compliance
5. Ratings & Reviews *(see dedicated doc)*
6. Buyer-Seller Discovery & Matching
7. Notifications, Alerts & Preferences
8. Ecommerce & Social-Commerce Integrations
9. Payments & Financial Services
10. Seller Growth & Recommendations
11. Internal Operations / Call-Center & Employee Tools

---

## 1. Account Creation & Onboarding

**Business purpose**: get a new buyer or supplier registered on the platform, with
duplicate-account protection so the same person/business doesn't end up with multiple
overlapping records.

| Feature | APIs | Repo |
|---|---|---|
| Create a new user account | `POST /user/add` — [`UserAddController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserAddController.go) | service-api (write) |
| Duplicate detection & merge (same mobile/email/PNS already exists) | built into `UserAddController` — converts an insert into an HTTP-delegated field-merge update on the existing record | service-api (write) |
| Fan-out to replica databases on account creation | [`UserCentralizedQueue`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_CENTRALIZED_QUEUE.go), [`UserUpsertAdditional`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_ADDITIONAL.go), [`UserUpsertConsumers`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_CONSUMERS.go) (syncs to ~15 databases: alert, trust, enquiry, LMS, search, image, etc.) | consumers |
| Terms & conditions acceptance | `POST /user/tncacceptance` — [`UserTncAcceptanceController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTncAcceptanceController.go.go) | service-api (write) |
| OTP-secured account update (e.g. changing a verified mobile/email) | `POST /user/updatewithotp` — [`UpdateWithOtpController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UpdateWithOtpController.go) | service-api (write) |
| Blacklist/fraud screen at signup | [`UserCentralizedQueueBanned`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_CENTRALIZED_QUEUE_BANNED.go), blacklist checks inside `UserUpsertAdditional` | consumers |

---

## 2. Profile & Business Identity Management

**Business purpose**: let a user (mostly suppliers) build out and maintain their business
profile — contact details, company branding, storefront content — the information buyers
see and use to decide whether to trust and contact them.

| Feature | APIs | Repo |
|---|---|---|
| View core contact/profile details | `GET /detail`, `GET /minidetail` — [`UserDetailController`](../internal/controllers/UsersControllers/UserDetailController.go), [`MiniDetailController`](../internal/controllers/UsersControllers/MiniDetailController.go) | users-api (read) |
| Edit contact/profile details | `POST /user/update` — [`UserUpdateController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserUpdateController.go) (largest controller in the write API, ~40+ editable fields) | service-api (write) |
| Multi-purpose detail editor (company reg, bank, fact sheet, franchise, location, other-remarks) | `POST /details` — [`UserDetailsController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDetailsController.go) (dispatched by `type` param across 9 sub-domains) | service-api (write) |
| Company logo / branding images | `GET /user/logo` reads via `otherdetail`; `POST /user/logo`, `POST /user/image` — [`UserLogoController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserLogoController.go), [`UserImageController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserImageController.go) | both |
| Storefront profile sections (awards, testimonials, infrastructure, news, custom sections) | `GET /profile` — [`ProfileController`](../internal/controllers/UsersControllers/ProfileController.go); `POST /profile` — [`UserProfileController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserProfileController.go); template-tab editor `POST /template` — [`UserTemplateController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTemplateController.go) | both |
| SEO metadata for category/storefront pages | `POST /meta` — [`UserMetaController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserMetaController.go) | service-api (write) |
| Social media profile links (LinkedIn, Twitter, Facebook, Instagram, WhatsApp toggle) | `POST /user/addsocialcontact` — [`UserSocialContactsController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSocialContactsController.go) | service-api (write) |
| Digital visiting card | `GET /vcard/display` — [`ActionVcardDisplay`](../internal/controllers/UsersControllers/VcardDisplayControllers.go); `POST /visiting/card` — [`VisitingCardController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/VisitingCardController.go) + [`UserVisitingCardApproval`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VISITINGCARD_APPROVAL.go) consumer (approval workflow) | all three |
| Content moderation on profile/template/meta edits | [`USER_PROFILE_BANNED_DETECT`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PROFILE_BANNED_DETECT.go), [`USER_ATTR_BANNED_CONTENT_CHECK`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_ATTR_BANNED_CONTENT_CHECK.go) (banned-keyword + PII scan before content propagates) | consumers |
| "Other detail" reads spanning company reg / bank / fact sheet / franchise / location | `GET /otherdetail` — [`UserOtherDetailController`](../internal/controllers/UsersControllers/UserOtherDetailController.go) | users-api (read) |

---

## 3. Location Intelligence

**Business purpose**: know where a business is physically located, both for display (to
buyers looking for nearby suppliers) and for verification (does the claimed address match
reality).

| Feature | APIs | Repo |
|---|---|---|
| Reverse-geocode lat/long to a structured address | `GET /latlongtoaddress/addressfields` — [`LatLongController`](../internal/controllers/UsersControllers/LatLongController.go) (read); `LatLongController` write helpers via DB-first or Google/MMI fallback | users-api (read) |
| Capture a business's address/coordinates | `POST /userlocations/update` — [`UserLocationsUpdateController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserLocationsUpdateController.go) (batch-capable, up to 4 concurrent) | service-api (write) |
| Verify submitted location against stored location | `POST /verification/latlongverify` — [`LatLongVerifyController`](../internal/controllers/UsersControllers/LatLongVerifyController.go) → [`UserLatLongVerification`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_LATLONG_VERIFICATION.go) consumer (bound to queue key `USER_GST_LAST_MODIFIED_DL` — a naming mismatch worth remembering) | users-api (write-adjacent) + consumers |
| Resolve free-text city names when structured city ID is missing | [`USER_UPSERT_CITY_OTHERS`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_CITY_OTHERS.go) consumer (calls an external LatLong API, then writes back via WAPI) | consumers |
| Showroom/alias URL sync tied to city/country changes | [`GLUSR_USR_SHOWROOM_URL`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/GLUSR_USR_SHOWROOM_URL.go) consumer | consumers |

---

## 4. Trust, Verification & Compliance

**Business purpose**: build buyer confidence in a supplier by verifying their identity,
business registration, and contact details — and screen out fraudulent or non-compliant
accounts. This is the single largest, most cross-cutting story in the whole domain.

| Feature | APIs | Repo |
|---|---|---|
| Trust seal badge display | `GET /trustseal`, `GET /trustsealdetail`, `GET /tsdetails`, `GET /tscompdetails`, `GET /tsmiscdetails` | users-api (read) |
| Trust seal management (approve/assign) | `POST /user/trustseal` — [`UserTrustsealController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTrustsealController.go) (`WEBERP`-only gateway) | service-api (write) |
| ID proof upload/lookup | `GET /idproof` — [`IdProofController`](../internal/controllers/UsersControllers/IdProofController.go) (read); `POST /user/idproof` — [`IdProofController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/IdProofController.go) (write) | both |
| Per-attribute verification (mobile, email, GST, PAN, bank, Aadhaar, address...) | `GET /getverificationdetails`, `GET /verifieddetail` (read); `POST /user/verification`, `POST /user/userverificationdetails`, bulk via `USER_VERIFICATION_BULK_HISTORY` (write) | both + consumers |
| GST / company registration & tax classification | `GET /otherdetail` (type=CompRgst); `POST /details` (type=CompRgst); `POST /gsthsnmapping` — HSN code mapping; sync fan-out via [`UserGstDetails`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_GST_DETAILS.go), [`UserGstLastModified`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_GST_LAST_MODIFIED.go), [`GstLastModifiedBuyerProfile`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_GST_LAST_MODIFIED_BUYERPROFILE.go), [`GstLastModifiedTrust`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_GST_LAST_MODIFIED_TRUSTPG.go) consumers | all three |
| Legal status computation from GST | [`LEGAL_STATUS_COMPUTATION`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/LEGAL_STATUS_COMPUTATION.go) consumer (derives legal entity type from GST's 6th character + company name) | consumers |
| Fraud/blacklist screening | `GET /readmlfraud`, `GET /blacklistvalues` (read); [`UserBlacklistController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserBlacklistController.go) (write — **currently dead/deprecated**) | users-api (read) |
| Supplier verification call tracking | `GET /supp_verify_log`; `POST /supp_verify_log` — [`UserSuppVerifyLogController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSuppVerifyLogController.go); eligibility pipeline in [`UserSellOnImCalling`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SELLONIM_CALLING.go) consumer | all three |
| Blacklist attribute disposition (call-outcome tagging on GST/attribute checks) | `GET /attrdispositions` — [`AttrDispositionController`](../internal/controllers/UsersControllers/AttrDispositionController.go); `POST /user/dispositon` — [`UserDispositionController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDispositionController.go) | both |
| Bank account verification (penny-drop) | write via [`UserDetailsController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDetailsController.go) (type=BankDetails); [`USER_VERIFY_BANK_DETAILS`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VERIFY_BANK_DETAILS.go) consumer → RabbitMQ `PennyDrop` | service-api + consumers |

---

## 5. Ratings & Reviews

See [`ratings_reviews_product_overview.md`](./ratings_reviews_product_overview.md) for the
full story/feature breakdown — it covers Supplier Ratings (submission, moderation,
matchmaking check, enrichment, aggregation, notification, replies, "helpful" voting,
archival) and Social Reviews (Google/Facebook account linking and content sync) in detail.

---

## 6. Buyer-Seller Discovery & Matching

**Business purpose**: help suppliers evaluate incoming buyer leads (is this a real,
trustworthy buyer worth responding to?) and help the platform understand buyer intent for
better matching/recommendations.

| Feature | APIs | Repo |
|---|---|---|
| Full buyer profile for a supplier viewing a lead | `GET /buyerprofile` — [`BuyerProfileController`](../internal/controllers/UsersControllers/BuyerProfileController.go) (heavily PII-masked) | users-api (read) |
| "Know your buyer/user" identity search (support/internal tool) | `GET /know_your_user`, `GET /knowyourbuyer` — [`KnowYourUserController`](../internal/controllers/UsersControllers/KnowYourUserController.go), [`KnowyourbuyerController`](../internal/controllers/UsersControllers/KnowyourbuyerController.go) | users-api (read) |
| Recently browsed/enquired products for a buyer | `GET /getbrowsedproducts` — `ActionBrowsedProducts` | users-api (read) |
| Buyer product-of-interest inference | `GET /userinterestdetails` — `ActionUserInterestDetailsController` (stored function over search/browse history) | users-api (read) |
| Category exclusion (supplier opts out of certain product categories) | `GET /negativemcat`, `POST /negative/mcat` — read/write pair | both |
| Buyer blocking (supplier blocks an abusive buyer) | `POST /user/user_block_unblock` — [`UserBlockUnblockController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserBlockUnblockController.go) | service-api (write) |
| Buyer-supplier connection/matchmaking tracking | `POST /user/bsmatchmaking` — [`BsMatchMakingController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/BsMatchMakingController.go); [`UserBSMatchMakingKafka`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_KAFKA.go), [`UserBsMatchLMS`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_LMS.go) consumers (write `GLUSR_CONTACTBOOK_MAPPING` via stored proc) | service-api + consumers |
| Buyer activity/clickstream tracking (page views, searches) | `POST /user/addbuyeractivity` — [`UserBuyerActivityController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserBuyerActivityController.go) (Cassandra); [`UserPersonalizationActivity`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PERSONALIZATION_ACTIVITY.go), [`UserBusinessFeeds`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go), [`UserSearchBrwseIntrst`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SEARCH_BROWSE_INTEREST.go) consumers | service-api + consumers |

---

## 7. Notifications, Alerts & Preferences

**Business purpose**: give users control over how and when they're contacted, and route
messages/notifications through the right channel.

| Feature | APIs | Repo |
|---|---|---|
| Privacy/notification settings (what's visible, what alerts fire) | `GET /setting` (+`_v1`) — [`SettingController`](../internal/controllers/UsersControllers/SettingController.go), [`SettingController_v1`](../internal/controllers/UsersControllers/SettingController_v1.go); `POST /details` (type=PrivSetting) | both |
| Phone-number-service (PNS) off-hours / call routing preferences | `GET /pnssetting`, `POST /pnssetting` — [`PnsSettingControllers`](../internal/controllers/UsersControllers/PnsSettingControllers.go) (read/write pair) | both |
| Messaging-platform integration (e.g. WhatsApp Business linkage) | `GET /msgintegration`, `POST /user/msg_integration` | both |
| Email frequency/alert threshold management | [`USER_MAIL_FREQUENCY_ALERT`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_MAIL_FREQUENCY_ALERT.go) consumer | consumers |
| In-app feedback capture | `GET/POST /feedback` — [`UserFeedbackController`](../internal/controllers/UsersControllers/UserFeedbackController.go) (both repos) → [`UserRelatedSendMail`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RELATED_SEND_MAIL.go) consumer sends the actual notification | all three |
| In-app popup interest tracking (e.g. "interested in upgrading?") | `POST /popupdetails` — [`PopupDetailsController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/PopupDetailsController.go) | service-api (write) |

---

## 8. Ecommerce & Social-Commerce Integrations

**Business purpose**: let suppliers connect external storefronts and social-commerce
content (e.g. short-form video/"Flips") to their IndiaMART presence.

| Feature | APIs | Repo |
|---|---|---|
| Ecommerce store account mapping | `GET /ecom_mapping`, `POST /ecom_mapping` — [`EcomMappingController`](../internal/controllers/UsersControllers/EcomMappingController.go) (read/write pair) | both |
| Ecommerce subscription management | `GET/POST /ecom_subscription_details` — **decommissioned on both sides** ([`EcomSubscriptionDetailsController`](../internal/controllers/UsersControllers/EcomSubscriptionDetailsController.go) / [`UserEcomSubscriptionController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserEcomSubscriptionController.go) both hardcoded to return "decommissioned") | both (dead) |
| Flips (social-commerce video) account linkage | `GET/POST /flips/account` — [`FlipsAccountController`](../internal/controllers/UsersControllers/FlipsAccountController.go) (read/write pair) | both |
| Flips video content sync | `GET/POST /flips/content` — [`FlipsContentController`](../internal/controllers/UsersControllers/FlipsContentController.go); [`FlipsContentSync`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_FLIPS_CONTENT_SYNC.go) consumer (diffs and batches to `flips_content_write_service`) | all three |
| Marketplace URL tracking (Facebook/Instagram/Google shop links) | `GET /mrkturl` — [`MrktUrlController`](../internal/controllers/UsersControllers/MrktUrlController.go); `POST /market_place_url` — [`UserMarketPlaceUrlController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserMarketPlaceUrlController.go); [`UserMktpUrl`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_MKTP_URL.go) consumer | all three |

---

## 9. Payments & Financial Services

**Business purpose**: fintech-adjacent features layered on top of the core user record —
payment vendor linkage, transaction logging, and financing eligibility.

| Feature | APIs | Repo |
|---|---|---|
| Payment vendor account linkage | `GET /getmerchanttovendordetails` (read); `POST /user/addmerchanttovendor`, `POST /user/addpaymentvendor` — [`UserAddMechantToVendorController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserAddMechantToVendorController.go), [`UserAddPaymentVendorController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserAddPaymentVendorController.go) | both |
| International payment transaction logging | `POST /user/addpaymenttxndetails` — [`UserAddPaymentTxnDetailsController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserAddPaymentTxnDetailsController.go) | service-api (write) |
| InstaFinance eligibility flag (financing product) | `POST /instafinance` — [`UserInstaFinanceController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserInstaFinanceController.go) (note: has a flagged bug — hardcodes `env="dev"`, disabling APM tracing in prod) | service-api (write) |
| Bank details & penny-drop verification | `POST /details` (type=BankDetails); [`USER_VERIFY_BANK_DETAILS`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VERIFY_BANK_DETAILS.go) consumer → `PennyDrop` queue | service-api + consumers |

---

## 10. Seller Growth & Recommendations

**Business purpose**: drive supplier engagement and revenue by surfacing suggested actions,
managing paid visibility features, and tracking service subscriptions.

| Feature | APIs | Repo |
|---|---|---|
| Suggested-actions feed for sellers | `GET /seller_recommendation` — [`SellerRecommendationController`](../internal/controllers/UsersControllers/SellerRecommendationController.go) (aggregates from 4-5 downstream services, no direct DB) | users-api (read) |
| Paid listing keywords (PL/KWPL) | `POST /kwpl` — [`UserKwplController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserKwplController.go) (insert/disable/update/search-update by ACTION) | service-api (write) |
| Statutory/compliance detail aggregator | `GET /statutory_details` — [`StatutoryDetailsController`](../internal/controllers/UsersControllers/StatutoryDetailsController.go) (fan-out to GST/user-detail/other-detail services) | users-api (read) |
| Paid service subscription status & NPS survey eligibility | `GET /services` — [`ServiceController`](../internal/controllers/UsersControllers/ServiceController.go) | users-api (read) |

---

## 11. Internal Operations / Call-Center & Employee Tools

**Business purpose**: support IndiaMART's internal telecalling, verification, and audit
teams — not customer-facing, but essential to keeping the trust/verification pipeline (Story
4) running.

| Feature | APIs | Repo |
|---|---|---|
| Verification call disposition logging (agent records outcome of a verification call) | `GET /attrdispositions` — [`AttrDispositionController`](../internal/controllers/UsersControllers/AttrDispositionController.go); `POST /user/dispositon` — [`UserDispositionController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDispositionController.go) | both |
| Phone-number allocation for telecalling operations | `POST /pns` — [`PnsTxnController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/PnsTxnController.go) (`UserPnsAllocationController`, the most complex saga in the write API — 5 action types) | service-api (write) |
| "Sell on IndiaMART" telecalling log | `POST /user/sellonimlog` — [`UserSellonimLogController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSellonimLogController.go) | service-api (write) |
| Change-history / audit trail viewer | `GET /history` — [`HistoryController`](../internal/controllers/UsersControllers/HistoryController.go) (8+ distinct query shapes across Postgres + BigQuery — the single most complex read endpoint in the domain) | users-api (read) |
| Rating history / behavioral signal for GST teams | `GET /bs_attr_log` — [`BSAttrLogController`](../internal/controllers/UsersControllers/BSAttrLogController.go) (BigQuery-backed) | users-api (read) |
| Buyer-seller mapping lookup (support tool) | `GET /bsmapping` — [`BSMappingController`](../internal/controllers/UsersControllers/BSMappingController.go) | users-api (read) |

---

## Cross-story infrastructure (not a "story" itself, but underpins all of them)

- **[`USER_CENTRALIZED_QUEUE`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_CENTRALIZED_QUEUE.go)** and **[`USER_UPSERT_ADDITIONAL`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_ADDITIONAL.go)** — the two central fan-out
  consumers that nearly every "write a field on GLUSR_USR" flow eventually routes through,
  regardless of which story triggered it.
- **[`USER_PROFILE_BANNED_DETECT`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PROFILE_BANNED_DETECT.go)**, **[`USER_ATTR_BANNED_CONTENT_CHECK`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_ATTR_BANNED_CONTENT_CHECK.go)**,
  **[`USER_CENTRALIZED_QUEUE_BANNED`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_CENTRALIZED_QUEUE_BANNED.go)** — the shared content-moderation layer, reused across
  Stories 2, 4, and 5.
- **Replica database sync** — dozens of near-duplicate consumers (logo sync × 3, negative-mcat
  sync × 3, GST sync × 4) keep `GLUSR_USR` and related tables consistent across ~15 physically
  separate Postgres/Cassandra databases. This is plumbing, not a product story, but explains
  why so many consumer files exist per feature.

---

## How to use this doc

Each row above is a starting point — for the exact request params, SQL, and side effects
behind any single feature, follow the linked API name into
[`users_api_database_reference.md`](./users_api_database_reference.md),
[`service_api_write_reference.md`](./service_api_write_reference.md), or
[`user_consumers_reference.md`](./user_consumers_reference.md). For a fully worked
end-to-end example of a story tying all three repos together, see the supplier-rating trace
in [`full_read_write_picture.md`](./full_read_write_picture.md).

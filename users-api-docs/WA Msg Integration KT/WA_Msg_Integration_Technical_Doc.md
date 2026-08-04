# WhatsApp Message Integration — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye
[`WA_Msg_Integration_Business_Doc.md`](./WA_Msg_Integration_Business_Doc.md) dekho. Yeh doc
GST_Technical_Doc.md jaisi depth follow karta hai — har claim file:line ke saath, jahan
conclusively confirm nahi ho paaya wahan **[INFERRED — confirm with team]** ya Open Questions
mein flag kiya hai.

**Repos**: `service-api-go-production` (write, integration record),
`users-api-go-production` (read, integration record + outbound-nudge utility + its actual
trigger site).

**Scope**: do genuinely alag components —
(A) WhatsApp Business Platform (BSP) integration record (`Glusr_msg_platform_integration`),
(B) outbound "app-install nudge" WhatsApp message utility (`whatsapp_handler.go`), triggered
from a completely different subsystem (`SendMsg`/`AppTrack_SendMsg`) that also sends SMS.
A third, genuinely separate feature — the WhatsApp-toggle on a supplier's public
social-contact links (`IS_WHATSAPP_ACTIVE` on `GLUSR_USR_SOCIAL_CONTACT`) — is **not** covered
here; see [`../Social Contacts KT/Social_Contacts_Technical_Doc.md`](../Social%20Contacts%20KT/Social_Contacts_Technical_Doc.md)
for that (re-confirmed tangential, see section 1 note below).

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Integration write (insert/update) | write | [`UserMsgIntegrationController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserMsgIntegrationController.go), [`UserMsgIntegrationModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMsgIntegrationModel.go) (`UpsertMsgintegration`, `check_glid`, `checkStatus`) |
| Integration read | read | [`MsgIntegrationController.go`](../internal/controllers/UsersControllers/MsgIntegrationController.go) (`ActionMsgIntegration`), [`UserMsgIntegrationModel.go`](../internal/models/users/UserMsgIntegrationModel.go) (`GetMsgIntegration`, status/enum maps) |
| Outbound nudge-message utility | read repo, shared util | [`whatsapp_handler.go`](../pkg/utils/whatsapp_handler.go) (`SendWhatsAppAppInstall`, `checkWhatsAppFrequency`, `sendToCentralizedWhatsApp`) |
| **Nudge trigger site — NEW finding, resolves old doc's Open Question #3** | read repo | [`SendMsgModel.go`](../internal/models/apps/SendMsgModel.go) (`triggerWhatsApp` at line 730, called from `SendMsg_model` at 4 call-sites: lines 320, 361, 442, 483), controller [`SendMsg.go`](../internal/controllers/AppsControllers/SendMsg.go) |
| Related — WhatsApp toggle on social-contact links | write | `UserSocialContactModel.go`, consumer [`USER_SOCIAL_CONTACT.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SOCIAL_CONTACT.go) — **confirmed still a separate feature** after reading the deeper [Social Contacts Technical Doc](../Social%20Contacts%20KT/Social_Contacts_Technical_Doc.md): that doc's `IS_WHATSAPP_ACTIVE` is a boolean "does this supplier want WhatsApp shown as a public contact-method" flag on `GLUSR_USR_SOCIAL_CONTACT`, written via 3 independent paths (sync API, dedicated consumer, generic `USER_UPSERT_ADDITIONAL` dispatcher) — none of those 3 paths touch `Glusr_msg_platform_integration` or `whatsapp_handler.go`, and neither of this doc's tables appear anywhere in the Social Contacts doc. Genuinely independent systems sharing only the word "WhatsApp." |

---

## 2. Routes

| Method | Path | Repo | Controller |
|---|---|---|---|
| POST | serviceName `MSG_PLATFORM_INTEGRATION` (direct API call, gateway `GLADMIN`/`LMS`) | write | `UserMsgIntegrationController` — [`UserMsgIntegrationController.go:16`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserMsgIntegrationController.go) |
| GET/POST | `msgintegration/*params` | read | `ActionMsgIntegration` — registered twice in [`routerUsers.go:390-391,528-529`](../internal/api/users_router/routerUsers.go) (two router groups use it; a third registration at lines 727-728 is commented out) |
| GET/POST | `sendmsg/*params` | read (unrelated router group, `apps_router`) | `AppsControllers.SendMsg` — [`routerApps.go:98-99`](../internal/api/apps_router/routerApps.go), also mounted on a failover group at line 146. This is **not** a Msg-Integration route at all — it's the generic SMS/WhatsApp "app install nudge" sender, which is where `whatsapp_handler.go` actually gets invoked from (section 6, Flow C). |

`whatsapp_handler.go`'s own functions expose no HTTP route — it's an internal package called
from `SendMsgModel.go`.

---

## 3. Data Model — Tables

> **Verification note**: table/column names Go code ke andar embedded SQL strings se liye
> gaye hain (file:line har row ke saamne diya hai). Live DB schema se cross-verify **nahi**
> kiya gaya hai.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `Glusr_msg_platform_integration` | meshpg | **Primary integration record** — ek row per (supplier, messaging-platform) pair, `ON CONFLICT (fk_glusr_usr_id, fk_msg_platform_id)` unique constraint | `fk_glusr_usr_id`, `fk_msg_platform_id`, `msg_platform_phone_number_id`, `msg_platform_mobile_number`, `msg_platform_user_token`, `msg_platform_business_id`, `service_start_date`/`service_end_date`, `is_service_enabled`, `Is_outbound_service_enabled`, `outbound_service_start_date`/`_end_date`, `onboarded_date`, `fk_msg_platform_status_id`, `fk_msg_platform_bsp_id`, `messaging_tier`, `msg_display_name_verf_status`, `fb_business_mgr_verf_status`, `msg_quality_rating`, `fb_login_profile_id`, `fb_login_username`, `msg_platform_dp_logo_url`, `glusr_bsp_account_details`, `msg_platform_vendor_business_id`/`_vendor_project_id`, `fb_kyc_verification_status`, `fb_kyc_failure_reason`, `msg_platform_last_onboarded_date`, `fb_mgr_id`, `is_coexistence_enabled`, `meta_access_token`, `onboarding_flow`, `past_msg_sync_flag`/`_date`, `last_modified_date`, `glusr_msg_platform_updated_by_id`/`_name`/`_screen` — [`UserMsgIntegrationModel.go:136`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMsgIntegrationModel.go), read side [`UserMsgIntegrationModel.go:33,128`](../internal/models/users/UserMsgIntegrationModel.go) |
| `glusr_msg_platform_log` | meshpg | **Audit/status-change log** — status transitions aur scam/spam flags record hote hain, two independent INSERT call-sites into the same table (section 4) | `fk_glusr_usr_id`, `glusr_msg_platform_log_type` (statusflag `0`/`1`, or scam/spam code `3`/`4`), `glusr_msg_platform_log_insert_date`, `glusr_msg_platform_log_comment`, `fk_msg_platform_status_id` — [`UserMsgIntegrationModel.go:132`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMsgIntegrationModel.go) |
| `iil_whatsapp_msg_events` | notification_pg_master (a different physical DB than the integration table — see Edge Cases #2) | Outbound-nudge send-history — read-only from these 3 repos' perspective, used purely for the 24-hour frequency cap | `glusr_mobile_number`, `process_name`, `msg_attempt_date`, `msg_failure_reason` — [`whatsapp_handler.go:102-109`](../pkg/utils/whatsapp_handler.go). **No INSERT into this table was found anywhere in `users-api-go-production`, `service-api-go-production`, or `user-temp-consumers-production`** — something outside these 3 repos (likely the external WhatsApp-sending vendor's callback/webhook, or the `dev-cnotify.indiamart.com/whatsapp` centralized-notify service itself) must be writing it. Flagged in Open Questions. |
| `sms_install_app` | notification_pg_master | Sibling frequency-tracking table used by the **SMS** side of the same `SendMsg_model` function (`IsSmsSentAttempted`) — not WhatsApp-specific, included here only because it lives in the same function and is easy to confuse with `iil_whatsapp_msg_events` | `mobile_number`, `entry_date`, `service_call_status` — [`SendMsgModel.go:659`](../internal/models/apps/SendMsgModel.go) |

**Naming decode**: `bsp` = Business Solution Provider (external WhatsApp messaging vendor —
concretely enumerated as `AISENSY`/`META`/`VALUEFIRST` in `MsgPlatformBspMap`, section 4).
`fb_*` columns = Facebook/Meta Business Manager verification fields. `is_coexistence_enabled`
= WhatsApp's "coexistence" feature (business app + Cloud API used together). `onboarding_flow`
= which onboarding path was used (`BSP` / `TECH PARTNER` / `SPLBV` / `OTHER`, per
`OnboardingFlowMap`, section 4).

---

## 4. Decode-the-Magic-Values — Status & Enum Maps

This section resolves the prior shallow doc's biggest gap (Open Question #1: "what do
`fk_msg_platform_status_id` ranges 12-14 / 15-19 mean?"). The **read-side** model
(`GetMsgIntegration`) contains explicit Go maps that decode every numeric status/enum column
the write-side only stores as raw integers — these maps are the authoritative, code-confirmed
source:

### `fk_msg_platform_status_id` → `MsgPlatformSubStatusMap` [`UserMsgIntegrationModel.go:338-354`](../internal/models/users/UserMsgIntegrationModel.go)

| ID | Sub-status | Parent bucket (`parent_msg_platform_status`, computed at [`UserMsgIntegrationModel.go:260-271`](../internal/models/users/UserMsgIntegrationModel.go)) |
|---|---|---|
| 7 | `PROCESS_INCOMPLETE` | `NOT_ONBOARDED` |
| 8 | `MISSING_TOKEN` | `NOT_ONBOARDED` |
| 9 | `FAILED` | `NOT_ONBOARDED` |
| 10 | `IN_PROGRESS` | `NOT_ONBOARDED` |
| 11 | `LIVE` | `ACTIVE` |
| 12 | `REINSTATED` | `ACTIVE` |
| 13 | `UNLOCKED` | `ACTIVE` |
| 14 | `REONBOARDED` | `ACTIVE` |
| 15 | `LOCKED` | `PARTIALLY_ACTIVE` |
| 16 | `DISABLE` | `INACTIVE` |
| 17 | `PHONE_NUM_REMOVE` | `INACTIVE` |
| 18 | `PARTNER_REMOVE` | `INACTIVE` |
| 19 | `DELETED` | `INACTIVE` |
| 20 | `DEACTIVATED_ACCOUNT` | `DOWNLOADED` |
| 21 | `OTHER` | `OTHERS` (fallback) |

**This directly explains the write-side's numeric-range check** (`UserMsgIntegrationModel.go:364-371`,
section 5 point 5): `substatusid` `12`-`14` (`REINSTATED`/`UNLOCKED`/`REONBOARDED` — i.e. moving
*back into* the `ACTIVE` bucket from a locked/disabled state) sets `statusflag="1"`; `15`-`19`
(`LOCKED` through `DELETED` — i.e. moving *out of* active) sets `statusflag="0"`. Note the write
path's range **excludes** `11` (`LIVE`, the "already active, first-time-live" state) — a fresh
first-time activation to `LIVE` does not itself generate a `glusr_msg_platform_log` row via this
range-check; only *transitions through* the reinstated/locked states do.

### Other enum maps (all in [`UserMsgIntegrationModel.go`](../internal/models/users/UserMsgIntegrationModel.go), read side)

| Column | Map name | Values |
|---|---|---|
| `msg_display_name_verf_status` | `MsgDispVerfStatusMap` | `1`=AVAILABLE_WITHOUT_REVIEW, `2`=PENDING_REVIEW, `3`=APPROVED, `4`=REJECTED, `5`=DEFERRED, `6`=OTHER |
| `fb_business_mgr_verf_status` | `FbVerfStatusMap` | `1`=VERIFIED, `2`=NOT_VERIFIED, `3`=PENDING_NEED_MORE_INFO, `4`=PENDING_SUBMISSION, `5`=RESTRICTED, `6`=REJECTED, `7`=OTHER |
| `msg_quality_rating` | `MsgQltyRatingMap` | `1`=UNKNOWN, `2`=ONBOARDING, `3`=UPGRADE, `4`=DOWNGRADE, `5`=FLAGGED, `6`=UNFLAGGED, `7`=GREEN, `8`=YELLOW, `9`=RED, `10`=OTHER |
| `messaging_tier` | `MsgTierLimitMap` | `1`=TIER_NOT_SET, `2`=TIER_50, `3`=TIER_250, `4`=TIER_1K, `5`=TIER_10K, `6`=TIER_100K, `7`=TIER_UNLIMITED, `8`=OTHER, `9`=TIER_2K |
| `fk_msg_platform_bsp_id` | `MsgPlatformBspMap` | `1`=AISENSY, `2`=META, `3`=VALUEFIRST, `4`=OTHER |
| `fb_kyc_verification_status` | `MsgKycStatusMap` | `1`=PENDING, `2`=APPROVED, `3`=FAILED, `4`=REVOKED, `5`=DISCARDED, `6`=OTHER, `7`=UPLOADED |
| `onboarding_flow` | `OnboardingFlowMap` | `1`=BSP, `2`=TECH PARTNER, `3`=SPLBV, `4`=OTHER |
| `isScamSpam` input (`"1"`/`"2"`) | inline, write-side | `"1"`→ internal code `3`, comment `"SCAM"`; `"2"`→ internal code `4`, comment `"SPAM"` — [`UserMsgIntegrationModel.go:103-113`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMsgIntegrationModel.go) |

`MakeJsonMap` (the function that renders these) returns **every** key in the map with `"1"` if
it matches the stored value and `"0"` otherwise — i.e. the API response is a one-hot object, not
a single string field. Any client reading `msg_platform_status`/`msg_disp_verf_status`/etc. must
scan for the `"1"` key. [`UserMsgIntegrationModel.go:326-336`](../internal/models/users/UserMsgIntegrationModel.go)

---

## 5. Business Rules & Validation (code se)

### Integration record (write)
1. **Gateway allowlist chhota hai — `GLADMIN`/`LMS` sirf** — matlab yeh feature primarily
   internal/admin-driven hai, supplier khud seedha nahi call karta.
   [`UserMsgIntegrationController.go:59`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserMsgIntegrationController.go)
2. **`action` param (`"i"` ya `"u"`) explicitly insert-vs-update decide karta hai** — koi
   auto-detect nahi.
   [`UserMsgIntegrationModel.go:114,133,156`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMsgIntegrationModel.go)
3. **Insert path ek explicit `ON CONFLICT (fk_glusr_usr_id, fk_msg_platform_id) DO UPDATE`
   bhi carry karta hai** — `"i"` (insert) ke saath bhi, agar record already exist karta hai,
   silently update ho jaata hai.
   [`UserMsgIntegrationModel.go:136`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMsgIntegrationModel.go)
4. **Update path (`"u"`) ek pre-check karta hai pehle** (`check_glid` — `SELECT COUNT(*) ...
   msg_platform_integration_id = $2`, exact PK match required, not just GLID) — agar record
   nahi milta, `"No Data Available For Update"` return hota hai bina kisi write ke.
   [`UserMsgIntegrationModel.go:159,422-454`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMsgIntegrationModel.go)
5. **Update path (`"u"`) dynamic query-building karta hai** — sirf woh fields update hoti
   hain jo request mein present hain (bada `switch` statement, 25+ possible fields).
   [`UserMsgIntegrationModel.go:164-320`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMsgIntegrationModel.go)
6. **Status-range-based logging** (decoded in full in section 4): `substatusid` `12`-`14`
   → `statusflag="1"`, `15`-`19` → `statusflag="0"`, aur sirf tab log likha jaata hai jab
   status **actually change** ho raha ho (`checkStatus()` pre-check — `!isStatusChange`).
   [`UserMsgIntegrationModel.go:343,364-381,457-490`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMsgIntegrationModel.go)
7. **Scam/Spam flag alag numeric codes use karta hai**: `"1"` → `3` ("SCAM"), `"2"` → `4`
   ("SPAM") — yeh bhi `glusr_msg_platform_log` mein likhe jaate hain, ek **independent**
   `ExecuteQueryRows` call ke saath (Query4), status-change logging (Query3) se alag —
   matlab ek hi request mein dono trigger ho sakte hain (2 separate log-INSERTs).
   [`UserMsgIntegrationModel.go:103-113,384-401`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMsgIntegrationModel.go)
8. **`msg_platform_status`/`msg_platform_substatus` key dono ek hi column pe map hote hain**
   (`fk_msg_platform_status_id`) — caller in dono mein se koi bhi key bhej sakta hai same
   effect ke liye.
   [`UserMsgIntegrationModel.go:227`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMsgIntegrationModel.go)

### Outbound nudge (`whatsapp_handler.go`, section 6 Flow C mein exact trigger)
9. **Mobile number auto-normalize hota hai `+91` prefix ke saath** agar missing ho.
   [`whatsapp_handler.go:49-51`](../pkg/utils/whatsapp_handler.go)
10. **24-hour frequency cap ek DB query se enforce hota hai**, `iil_whatsapp_msg_events`
    table ke against, per (`mobile`, `process_name=APP_INSTALL_WHATSAPP`) pair, sirf
    `msg_failure_reason IS NULL` (i.e. successful-attempt) rows count hote hain — a failed
    prior attempt does **not** count against the cap, so a failure can be retried sooner than
    24 hours. Query pe khud ek **200ms timeout** hai (`context.WithTimeout`), independent
    of the outer HTTP timeout.
    [`whatsapp_handler.go:100-122`](../pkg/utils/whatsapp_handler.go)
11. **`ProcessAppInstall` (`"APP_INSTALL_WHATSAPP"`) is the only process constant defined** —
    the function signature/design supports other process-types sharing the same frequency
    mechanism, but only this one exists today.
    [`whatsapp_handler.go:18`](../pkg/utils/whatsapp_handler.go)
12. **Message-content source par depend karta hai** — `source == "WHATSAPP_9696"` ho toh
    pre-approved content-template (`contentTemplateId="app_install_wa"`) use hota hai, warna
    (default, includes `WHATSAPP_8181`) ek free-text message (`freeFlowMultiText`, hardcoded
    "Chat with Sellers in Real time on the app...") bheja jaata hai. `source == "WHATSAPP_8181"`
    alag se `wa_type="1"` set karta hai (warna default `"2"`).
    [`whatsapp_handler.go:63-77`](../pkg/utils/whatsapp_handler.go)
13. **`SendWhatsAppAppInstall` itself is only ever reached if `SendMsg_model`'s much larger
    gating logic allows it** — see section 6 Flow C for the exact, multi-layered trigger
    condition (this is the single most important new finding of this pass).
14. **HTTP client reusable/pooled hai** (`var httpClient = &http.Client{Timeout: 5 * time.Second}`).
    [`whatsapp_handler.go:26-28`](../pkg/utils/whatsapp_handler.go)
15. **External vendor timeout differs by environment**: dev-cnotify URL (fallback default) →
    60s timeout; any other (presumably prod) configured URL → 2s timeout — a much stricter
    prod SLA than dev.
    [`whatsapp_handler.go:127-144`](../pkg/utils/whatsapp_handler.go)

---

## 6. End-to-End Technical Flows

### Flow A — WhatsApp Business integration link/update

```
Admin/LMS (GLADMIN or LMS caller)
    │
    ▼
[API — write]  serviceName=MSG_PLATFORM_INTEGRATION  {action: "i" or "u", glusr_id, platform_id, ...}
    │  UserMsgIntegrationController.go
    │  1. Mandatory-param check (MandatoryParamscheckMsgIntegration)
    │  2. Gateway validation (GLADMIN/LMS only)
    ▼
UpsertMsgintegration()
    │
    ├─ action="i" → INSERT ... ON CONFLICT (fk_glusr_usr_id, fk_msg_platform_id) DO UPDATE
    │
    └─ action="u" → check_glid() SELECT COUNT (record exists for this exact msg_pid + glid?)
          │
          └─ dynamic UPDATE (only fields present in request), RETURNING msg_platform_integration_id
    ▼
[Conditional]  checkStatus() pre-check → if status actually changed AND substatusid in 12-14 or 15-19
    │            → INSERT glusr_msg_platform_log  (Query3)
    │
[Conditional]  if isScamSpam present → another, independent INSERT glusr_msg_platform_log (Query4)
    ▼
[RabbitMQ]  SERVICENAME=MSG_PLATFORM_INTEGRATION → route "comp.sync.<modulus>" (generic fan-out,
             confirmed via serviceToQueueMap in rabbitmq.go — no dedicated consumer found)
             fires only if output == "UPDATE SUCCESS"
```

### Flow B — Read integration status

```
Caller
    │
    ▼
[API — read]  GET/POST msgintegration/*params  {token, glusrid, modid, req_keys}
    │  ActionMsgIntegration.go → GetMsgIntegration()
    ▼
[DB]  SELECT [selected columns or *] FROM Glusr_msg_platform_integration
      WHERE fk_glusr_usr_id=$1 [ORDER BY last_modified_date DESC] LIMIT 1  (mesh_pg_user connection)
    │  decode numeric status/enum columns via the maps in section 4
    │  compute derived `active`/`is_ob_serv_active` flags from service_start/end_date windows
    ▼
Response: current integration status, verification/KYC/quality-rating fields (one-hot encoded)
```

### Flow C — Outbound "App Install" WhatsApp nudge — **exact trigger, traced end-to-end**

This was the prior doc's single biggest open question ("who calls `SendWhatsAppAppInstall`?").
It is reached only through the **generic `sendmsg` API** (also used for SMS), via multiple
layers of gating:

```
Caller (internal system — enq/click/PNS/etc. trigger, or app/web flow requesting an
"app install" nudge)
    │
    ▼
[API]  GET/POST sendmsg/*params  {token, mobile OR gluser, source, subsource, modid, ...}
    │  SendMsg.go — controller-level gates:
    │  1. token must equal a hardcoded value ("imobile@15061981")            — Auth Failed otherwise
    │  2. mobile OR gluser required
    │  3. source required, must be one of an explicit allowlist (includes WHATSAPP_9696/WHATSAPP_8181,
    │     plus many SMS sources: MissedCall, enq, click, PTT, AppVerify, ...)
    │  4. source=="enq"/"dev-enq" → modid required, restricted to {FCP,MDC,DIR,CTL,ETO,IMOB,TDW}
    ▼
SendMsg_model()
    │  5. source must be exactly "WHATSAPP_8181" or "WHATSAPP_9696" — ANY other source value
    │     short-circuits here with "App install SMS disabled" (this function doubles as the
    │     generic SMS sender for other sources, but the WhatsApp branch requires one of these two)
    │  6. mobile required (or resolved via gluser)
    │  7. if source != "AppVerify": IsEntryGLUSRMobDev() checked — if GLID already present in
    │     mobile-device table → service_call_status=8 (skips WhatsApp entirely, see step 9)
    │  8. if mobile not in a small hardcoded test-mobile allowlist (mob_map):
    │     IsSmsSentAttempted() checked against sms_install_app (this is the SMS-side frequency
    │     check, table-distinct from WhatsApp's own iil_whatsapp_msg_events check in step 10) —
    │     if already attempted → service_call_status=1, call_service=0 (skips WhatsApp)
    │  9. call_service must be nonzero (i.e. not already skipped by steps 7/8) to even reach the
    │     WhatsApp/SMS branch at all
    ▼
Inside the call_service branch (duplicated 2x for a X-Forwarded-Server dev/stg check that,
per code, is functionally a no-op — see Edge Cases #3):
    │  10. if source == WHATSAPP_9696 or WHATSAPP_8181 AND service_call_status NOT IN {5, 8}
    │      (5 = mobile format invalid; 8 = already in mob-dev table) →
    ▼
triggerWhatsApp() → SendWhatsAppAppInstall()
    │  11. Normalize mobile (+91 prefix)
    │  12. checkWhatsAppFrequency() — DB-backed 24hr check against iil_whatsapp_msg_events,
    │      scoped to process=APP_INSTALL_WHATSAPP, only counting non-failed prior attempts
    │
    ├─ Not allowed (sent within 24hrs) → ErrWhatsAppFrequencyCapped, delivery_status =
    │   "WhatsApp not sent as already sent within 24 hrs"
    │
    └─ Allowed →
          Determine wa_type + content (template if WHATSAPP_9696, else free-text)
          │
          ▼
          [HTTP POST]  apiURL = config APIList["appwhatsapp"] (falls back to
          http://dev-cnotify.indiamart.com/whatsapp if unset), pooled httpClient,
          timeout 60s (dev fallback URL) or 2s (configured/prod URL), VendorID="78"
          │
          ├─ non-200 or vendor RESPONSE=="FAILURE" → error, Kibana log (AppTrack_SendMsg, FAILURE)
          └─ success → "Whatsapp Triggered"
    ▼
Note: if the source is NOT WHATSAPP_9696/8181 (i.e. any other allowed source), this exact same
code branch instead calls utils.NewAppTrackMobileRabbit(input, db) — the SMS path. WhatsApp and
SMS share one dispatch point, gated purely by `source` string value.
```

**Business meaning [INFERRED — confirm with team]**: `source=WHATSAPP_9696`/`WHATSAPP_8181`
almost certainly correspond to two different upstream short-code/campaign identifiers (9696 and
8181 are IndiaMART's known SMS/missed-call short-codes) — i.e. this is the same "nudge a user to
install the app" flow that already exists for SMS, with a WhatsApp variant selected purely by
which `source` value the caller passes.

---

## 7. RabbitMQ / Kafka / Redis

| System | Usage |
|---|---|
| **RabbitMQ** | Integration-write path publishes `SERVICENAME=MSG_PLATFORM_INTEGRATION` (`PushToQueue`), only when `output == "UPDATE SUCCESS"`. Confirmed in [`rabbitmq.go:26,69`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go): `MSG_PLATFORM_INTEGRATION` maps to route `"comp.sync.<modulus>"` (generic company-sync fan-out, exchange `USER.topic`) — same generic mechanism used by GST/HSN and other domains, **not** a dedicated queue. No dedicated consumer for this SERVICENAME was found in `user-temp-consumers-production` (grepped `msg_platform`, `MSG_PLATFORM_INTEGRATION`, `WHATSAPP` — only an unrelated generic `"WHATSAPP": "WHATSAPP"` map-index constant in [`map_index.go:24082`](../../user-temp-consumers-production/user-temp-consumers-production/pkg/utils/map_index.go), not connected to this feature). The outbound-nudge side (`whatsapp_handler.go`) publishes nothing to RabbitMQ — it's a synchronous HTTP call. |
| **Kafka** | Koi usage nahi mila — grepped across all 3 repos for `msg_platform`/`WHATSAPP`/`MSG_PLATFORM_INTEGRATION` alongside Kafka-specific helpers (`InitializeKafka`, `sub_topic`); no matches. |
| **Redis** | Koi usage nahi mila directly. `checkWhatsAppFrequency` (§5 pt. 10) takes a `*sql.DB`, not a Redis client — the 24-hour frequency window is DB-backed, not Redis-TTL-backed, despite being a textbook Redis use-case. See Optimization Scope §9. |

---

## 8. Flow-wise DB & Table Usage

### Flow A — Integration write

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `Glusr_msg_platform_integration` | INSERT...ON CONFLICT DO UPDATE (action=i) OR SELECT (`check_glid`) then UPDATE (action=u) | The core upsert of the integration record; action=u costs 2 round-trips (existence-check then update), action=i costs 1 |
| 2 | meshpg | `Glusr_msg_platform_integration` | SELECT (`checkStatus`) | Pre-check whether the incoming substatus differs from current — decides whether to log (avoids duplicate log rows on a no-op status resubmit) |
| 3 | meshpg | `glusr_msg_platform_log` | INSERT (conditional — status changed AND in 12-14/15-19 range) | Status-change audit trail |
| 4 | meshpg | `glusr_msg_platform_log` | INSERT (conditional — independent of #3, fires if `isScamSpam` present) | Scam/Spam audit trail — can co-occur with #3 in the same request, 2 separate INSERTs |

### Flow B — Integration read

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | mesh_pg_user (read-API connection) | `Glusr_msg_platform_integration` | SELECT (either `*` or a dynamically-built column subset based on `req_keys`) | Single round-trip read; column selection is app-side optimization to avoid over-fetching |

### Flow C — Outbound nudge

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | (unspecified, "db" param — whichever connection `SendMsg_model` opened, `notification_pg_master`) | `GLUSR_MOBILE_DEVICE`-equivalent (via `IsEntryGLUSRMobDev`, exact table not traced in this pass) | SELECT | Determines whether this GLID is already a known app-installed device — if so, skip WhatsApp/SMS entirely |
| 2 | notification_pg_master | `sms_install_app` | SELECT | SMS-side frequency check (only relevant for the *SMS* branch of the same function; still executes even for WHATSAPP sources per code path, since it's evaluated before the source-branch) |
| 3 | notification_pg_master | `iil_whatsapp_msg_events` | SELECT (COUNT, `checkWhatsAppFrequency`, 200ms timeout) | WhatsApp-specific 24-hour frequency cap |
| 4 | *(external, not a DB)* | — | HTTP POST to vendor API | Actual message send — outside the 4 round-trips above, this is a network call, not a DB call |

**Total for a successful nudge send: at least 3 DB round-trips + 1 external HTTP call**, before
any actual message leaves IndiaMART's infrastructure.

---

## 9. Optimization Scope — DB Response-Time Contribution

### High-impact

1. **Frequency-check ek classic Redis-TTL use-case hai, lekin DB se implement hai** (§7) —
   `checkWhatsAppFrequency` is a DB query on every outbound-send attempt, with its own 200ms
   timeout (implying the team is already worried about it being slow). If nudge volume is
   high, Redis (`SETNX`/TTL pattern) would collapse this to a single fast key-lookup —
   [`utils_programming_guide.md`](../utils_programming_guide.md) documents Redis infra already
   available in this codebase, just not wired here.
2. **Insert-path's `ON CONFLICT DO UPDATE` covers a 38-parameter statement** — Postgres-level
   efficient (single round-trip), but a wide row like this under concurrent writes is worth
   monitoring for lock contention if integration-write volume grows.

### Medium-impact

3. **Update-path costs 3 sequential SELECT/UPDATE round-trips** (`check_glid` SELECT →
   `checkStatus` SELECT → dynamic UPDATE) before any log-INSERT — GST's "verify-then-write"
   pattern repeated here, not combined into a single `UPDATE ... WHERE EXISTS` /
   `RETURNING`-based approach the way the insert-path already does.
4. **Flow C's gating logic runs a DB SELECT (`IsEntryGLUSRMobDev`) and a second DB SELECT
   (`IsSmsSentAttempted` against `sms_install_app`) even when the request is a WhatsApp
   source** — the SMS-frequency check appears to run unconditionally before the
   source-branch decides WhatsApp vs SMS, meaning a WhatsApp-only caller still pays for an
   SMS-table query it will never use the result of. Worth confirming with the team whether
   this is intentional (shared spam-protection) or an avoidable extra round-trip.
   [`SendMsgModel.go:170-198`](../internal/models/apps/SendMsgModel.go)
5. **Status-change and scam/spam log-inserts are two independent round-trips that could
   sometimes combine** — if both conditions are true in the same request, that's 2 separate
   `glusr_msg_platform_log` INSERTs where a single multi-row INSERT could work.

### Low-impact / good practice already present

6. **Pooled HTTP client, 5s hard timeout, plus environment-aware inner timeout (2s prod / 60s
   dev)** (§5 pt. 15) — mature practice, tighter than most other domains reviewed.
7. **Partial-update pattern (only changed fields)** on the integration-write update path —
   avoids unnecessary full-row overwrites.
8. **Frequency check's own 200ms context timeout, independent of the outer request timeout** —
   defensive design that prevents a slow frequency-check query from blowing the whole nudge
   budget.

---

## 10. Cron Inventory

Grepped `service-api-go-production/crons` and all of `user-temp-consumers-production` for any
reference to `msg_platform`, `MSG_PLATFORM_INTEGRATION`, or `WHATSAPP` — **no cron/standalone
binary touches either table in this feature.** (The unrelated `gst_tact_veri_cron.go` and other
domain crons do not reference `Glusr_msg_platform_integration`, `glusr_msg_platform_log`, or
`iil_whatsapp_msg_events`.)

---

## 11. Edge Cases & Gotchas (technical POV)

1. **The write-side status-range check (12-14/15-19) is now fully decoded** (§4) — no longer
   an open question, but worth remembering the asymmetry: `LIVE` (11, first-time activation)
   does **not** itself generate a status-change log row via this specific range-check; only
   *transitions through* reinstated/locked/deleted states do.
2. **`iil_whatsapp_msg_events` lives in a different physical DB (`notification_pg_master`)
   than `Glusr_msg_platform_integration` (`meshpg`)** — these are not the same integration
   record. Do not confuse "has this supplier's BSP integration status changed" (meshpg) with
   "has an app-install nudge been sent to this mobile number" (notification_pg_master) — two
   fully independent tables, in two different databases, tracking two different concepts that
   both happen to be called "WhatsApp."
3. **The `X-Forwarded-Server` dev/stg branch in `SendMsgModel.go` is functionally a no-op**:
   both the `dev-mapi`/`stg-mapi` branch (lines 316-354) and the else/production branch
   (lines 355-395+) contain byte-identical `triggerWhatsApp`/`NewAppTrackMobileRabbit` logic —
   the only difference is dead commented-out Resque code in the `else` arm of `num >= 10`.
   Additionally `num = mob_int % 10` with `num<=0 → num=0` means `num` is always in `[0,9]`,
   so the `num < 10` condition is **always true** — the `num >= 10` "queue overflow" branches
   on both sides are unreachable dead code as written. Confirm with the team whether this was
   intentional simplification or a latent bug from an incomplete Resque→Rabbit migration.
   [`SendMsgModel.go:312-395`](../internal/models/apps/SendMsgModel.go)
4. **Nothing in these 3 repos writes `iil_whatsapp_msg_events`** — if the 24-hour cap ever
   appears to not be working (duplicate nudges), the write-side of that table is outside this
   codebase's visibility; check the vendor callback / `dev-cnotify.indiamart.com/whatsapp`
   service itself.
5. **`whatsapp_handler.go` living in the *read* repo (`users-api-go-production`) despite being
   an outbound-sender is explained by its actual call site**: it's invoked from
   `SendMsgModel.go`, which lives under `internal/models/apps` in the same repo, behind the
   `sendmsg` route in `apps_router` — a **generic notification-dispatch endpoint** (handles
   both SMS and WhatsApp app-install nudges), not the WA-Msg-Integration read endpoint at all.
   In other words: `users-api-go-production` isn't purely a "read" repo — it also hosts this
   one write-adjacent (send-side-effect) notification-dispatch surface, which happens to share
   a `pkg/utils` package with the true read endpoints. This is a structural quirk worth
   flagging to the team if repo-boundary conventions are ever tightened.
6. **The scam/spam and status-change log paths both key off `substatusid`/`params["substatus"]`
   independently** — `params["substatus"]` (used in the Query3 log-comment) is a different key
   from `params["msg_platform_substatus"]`/`params["msg_platform_status"]` (used to derive
   `substatusid` itself) — worth double-checking caller payloads set both consistently.
   [`UserMsgIntegrationModel.go:374`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMsgIntegrationModel.go)

---

## 12. Open Questions

1. Who/what writes `iil_whatsapp_msg_events`? Not found in any of the 3 repos (§3, §11 pt. 4)
   — likely the vendor-side webhook or the centralized notify service itself.
2. What exactly does `IsEntryGLUSRMobDev` query (§8 Flow C #1) — table name not traced in this
   pass, only the function's role (checking prior app-install) is confirmed.
3. Is the `num < 10` / dead `num >= 10` branch (§11 pt. 3) intentional, or an artifact of an
   incomplete PHP-Resque-to-RabbitMQ migration (the commented-out code literally references
   `Resque::enqueue` and a Redis backend IP)? If intentional, why keep the dead branch at all?
4. What are `source=WHATSAPP_9696` and `source=WHATSAPP_8181` in business terms — confirmed
   short-code/campaign identifiers, or something else? [INFERRED in §6 based on IndiaMART's
   known SMS short-codes, not confirmed with the team.]
5. Does any consumer actually process the generic `comp.sync.*` fan-out for
   `MSG_PLATFORM_INTEGRATION` events specifically, or is it purely informational/unused
   downstream?
6. Live DB schema verification — this doc reflects only what Go SQL strings imply, not a
   pgAdmin/schema cross-check.

---

## See also

- [`WA_Msg_Integration_Business_Doc.md`](./WA_Msg_Integration_Business_Doc.md) — product
  perspective
- [`../Social Contacts KT/Social_Contacts_Technical_Doc.md`](../Social%20Contacts%20KT/Social_Contacts_Technical_Doc.md) —
  the separate `IS_WHATSAPP_ACTIVE` social-contact-toggle feature, re-confirmed unrelated to
  this doc's scope
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — depth/structure
  reference this doc follows, also the source of the `comp.sync.*` generic-fan-out pattern
  reused by `MSG_PLATFORM_INTEGRATION`
- [`../users_domain_product_stories.md`](../users_domain_product_stories.md) —
  "Notifications, Alerts & Preferences" story
- [`../utils_programming_guide.md`](../utils_programming_guide.md) — shared Redis/RabbitMQ
  patterns referenced in §9's optimization suggestion

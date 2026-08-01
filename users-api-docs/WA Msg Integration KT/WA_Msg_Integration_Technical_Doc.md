# WhatsApp Message Integration — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye
[`WA_Msg_Integration_Business_Doc.md`](./WA_Msg_Integration_Business_Doc.md) dekho.

**Repos**: `service-api-go-production` (write, integration record), `users-api-go-production`
(read, integration record + outbound-message utility).

**Scope**: do genuinely alag components — (A) WhatsApp Business Platform integration record,
(B) outbound nudge-message sending utility. Dono §-wise clearly separate rakhe gaye hain.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Integration write (insert/update) | write | [`UserMsgIntegrationController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserMsgIntegrationController.go), [`UserMsgIntegrationModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMsgIntegrationModel.go) (`UpsertMsgintegration`) |
| Integration read | read | [`MsgIntegrationController.go`](../internal/controllers/UsersControllers/MsgIntegrationController.go) (`ActionMsgIntegration`), [`UserMsgIntegrationModel.go`](../internal/models/users/UserMsgIntegrationModel.go) (`GetMsgIntegration`) |
| Outbound nudge-message utility | read repo, shared util | [`whatsapp_handler.go`](../pkg/utils/whatsapp_handler.go) (`SendWhatsAppAppInstall`) |
| Related — WhatsApp toggle on social-contact links | write | `UserSocialContactModel.go`, consumer [`USER_SOCIAL_CONTACT.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SOCIAL_CONTACT.go) — **separate feature, only tangentially related** (a supplier's public WhatsApp-contact-toggle on their profile page, not the BSP integration) |

---

## 2. Routes

| Method | Path | Repo | Controller |
|---|---|---|---|
| POST | `/user/msg_integration` | write | `UserMsgIntegrationController` |
| GET/POST | `/msgintegration/*params` | read | `ActionMsgIntegration` |

`whatsapp_handler.go` ke functions (§ below) koi apna HTTP route expose nahi karte — yeh
ek internal Go package hai jo doosre controllers/consumers apne andar se call karte hain.

---

## 3. Data Model — Tables

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `Glusr_msg_platform_integration` | meshpg | **Primary integration record** — ek row per (supplier, messaging-platform) pair | `fk_glusr_usr_id`, `fk_msg_platform_id`, `msg_platform_phone_number_id`, `msg_platform_mobile_number`, `msg_platform_user_token`, `msg_platform_business_id`, `service_start_date`/`service_end_date`, `is_service_enabled`, `Is_outbound_service_enabled`, `fk_msg_platform_status_id`, `fk_msg_platform_bsp_id`, `messaging_tier`, `msg_display_name_verf_status`, `fb_business_mgr_verf_status`, `msg_quality_rating`, `fb_login_profile_id`, `fb_kyc_verification_status`, `fb_kyc_failure_reason`, `is_coexistence_enabled`, `meta_access_token`, `onboarding_flow`, `past_msg_sync_flag`/`_date` — [`UserMsgIntegrationModel.go:136`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMsgIntegrationModel.go) |
| `glusr_msg_platform_log` | meshpg | **Audit/status-change log** — status transitions aur scam/spam flags record hote hain | `fk_glusr_usr_id`, `glusr_msg_platform_log_type`, `glusr_msg_platform_log_insert_date`, `glusr_msg_platform_log_comment`, `fk_msg_platform_status_id` — [`UserMsgIntegrationModel.go:132`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMsgIntegrationModel.go) |

**Naming decode**: `bsp` = Business Solution Provider (external WhatsApp messaging vendor),
`fb_*` columns = Facebook/Meta Business Manager verification fields, `coexist_flag`/
`is_coexistence_enabled` = WhatsApp's "coexistence" feature (business app + API dono ek saath
use karna), `onboard_flw`/`onboarding_flow` = kaunse onboarding path se aaya (self-serve vs
assisted, likely).

`whatsapp_handler.go` ke `SendWhatsAppAppInstall` function ka **apna koi persistent table
nahi mila** iss pass mein — sirf ek "frequency check" function
(`checkWhatsAppFrequency`) hai jo kisi table/cache se pichle-send ka record padhta hai
(exact table/mechanism iss pass mein trace nahi hua, dekho Open Questions).

---

## 4. Business Rules & Validation (code se)

### Integration record (write)
1. **Gateway allowlist chhota hai — `GLADMIN`/`LMS` sirf** — matlab yeh feature
   primarily internal/admin-driven hai, supplier khud seedha nahi call karta (kam se kam
   iss specific write-path se).
   [`UserMsgIntegrationController.go:59`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserMsgIntegrationController.go)
2. **`action` param (`"i"` ya `"u"`) explicitly insert-vs-update decide karta hai** — koi
   auto-detect nahi, caller ko batana padta hai kaunsa operation chahiye.
   [`UserMsgIntegrationModel.go:114,133,156`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMsgIntegrationModel.go)
3. **Insert path ek explicit `ON CONFLICT (fk_glusr_usr_id, fk_msg_platform_id) DO UPDATE`
   bhi carry karta hai** — matlab `"i"` (insert) flag ke saath bhi, agar record already
   exist karta hai, silently update ho jaata hai (upsert-jaisa behavior, chahe caller ne
   explicitly "insert" bola ho).
   [`UserMsgIntegrationModel.go:136`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMsgIntegrationModel.go)
4. **Update path (`"u"`) dynamic query-building karta hai** — sirf woh fields update hoti
   hain jo request mein present hain (bada `switch` statement, 25+ possible fields), baaki
   untouched rehte hain. Yeh partial-update pattern hai, poori row overwrite nahi.
   [`UserMsgIntegrationModel.go:164-320`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMsgIntegrationModel.go)
5. **Status-range-based logging**: `substatusid` `12`-`14` ke beech ho toh `statusflag="1"`,
   `15`-`19` ke beech ho toh `statusflag="0"` — ye specific numeric ranges kis business-state
   ko represent karte hain, exact mapping iss pass mein decode nahi hui (likely
   "verified"-jaisi states vs "rejected/suspended"-jaisi states) — sirf tab logging hoti hai
   jab status **actually change** ho raha ho (`!isStatusChange` check).
   [`UserMsgIntegrationModel.go:363-381`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMsgIntegrationModel.go)
6. **Scam/Spam flag alag numeric codes use karta hai `isScamSpam` input se**:
   `"1"` → internally `3` (comment: "SCAM"), `"2"` → internally `4` (comment: "SPAM") — yeh
   bhi `glusr_msg_platform_log` mein likhe jaate hain, alag se DB round-trip ke saath.
   [`UserMsgIntegrationModel.go:103-113,384-401`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMsgIntegrationModel.go)

### Outbound nudge (whatsapp_handler.go)
7. **Mobile number auto-normalize hota hai `+91` prefix ke saath** agar missing ho.
   [`whatsapp_handler.go:49`](../pkg/utils/whatsapp_handler.go)
8. **24-hour frequency cap har `process` type ke liye alag hai** (`ProcessAppInstall` ek
   constant hai — matlab future mein doosre process-types bhi isi pattern se add ho sakte
   hain, har ek apna khud ka 24-hour window).
   [`whatsapp_handler.go:22,54`](../pkg/utils/whatsapp_handler.go)
9. **Message-content source par depend karta hai** — `source == "WHATSAPP_9696"` ho toh ek
   pre-approved content-template use hota hai (`contentTemplateId`), warna ek free-text
   message (`freeFlowMultiText`) bheja jaata hai. `source == "WHATSAPP_8181"` alag `wa_type`
   (`"1"` vs default `"2"`) trigger karta hai.
   [`whatsapp_handler.go:63-77`](../pkg/utils/whatsapp_handler.go)
10. **HTTP client reusable/pooled hai** (`var httpClient = &http.Client{Timeout: 5 * time.Second}`)
    — ek achi practice, har call pe naya client nahi banta.
    [`whatsapp_handler.go:26-28`](../pkg/utils/whatsapp_handler.go)

---

## 5. RabbitMQ / Kafka / Redis

| System | Usage |
|---|---|
| **RabbitMQ** | Integration-write path `SERVICENAME=MSG_PLATFORM_INTEGRATION` publish karta hai (`PushToQueue`), sirf jab `output == "UPDATE SUCCESS"` ho. Per domain-wide `serviceToQueueMap` (confirmed earlier this session), `MSG_PLATFORM_INTEGRATION` → `comp.sync.<glid%20>` route hoti hai — generic company-sync fan-out, koi dedicated consumer identify nahi hua iss pass mein. [`UserMsgIntegrationModel.go:405-418`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMsgIntegrationModel.go) |
| **Kafka** | Koi usage nahi mila |
| **Redis** | Koi usage nahi mila directly, lekin `whatsapp_handler.go` ka `checkWhatsAppFrequency` function ek DB (`*sql.DB`) parameter leta hai — matlab frequency-tracking **DB-backed hai, Redis nahi**, jabki yeh conceptually ek classic Redis-TTL use-case hai (24-hour rate-limit). Worth flag karna optimization ke liye (§7). |

---

## 6. End-to-End Technical Flows

### Flow A — WhatsApp Business integration link/update

```
Admin/LMS (GLADMIN or LMS caller)
    │
    ▼
[API — write]  POST /user/msg_integration  {action: "i" or "u", glusr_id, platform_id, ...}
    │  UserMsgIntegrationController.go
    │  1. Mandatory-param check
    │  2. Gateway validation (GLADMIN/LMS only)
    ▼
UpsertMsgintegration()
    │
    ├─ action="i" → INSERT ... ON CONFLICT DO UPDATE  (Glusr_msg_platform_integration)
    │
    └─ action="u" → check_glid() SELECT COUNT (record exists?)
          │
          └─ dynamic UPDATE (only fields present in request)
    ▼
[Conditional]  checkStatus() → if status actually changed AND in specific ranges (12-14 or 15-19)
    │            → INSERT glusr_msg_platform_log
    │
[Conditional]  if isScamSpam present → another INSERT glusr_msg_platform_log
    ▼
[RabbitMQ]  SERVICENAME=MSG_PLATFORM_INTEGRATION → comp.sync.<glid%20> (generic fan-out)
```

### Flow B — Read integration status

```
Caller
    │
    ▼
[API — read]  GET /msgintegration
    │  ActionMsgIntegration.go → GetMsgIntegration()
    ▼
[DB]  SELECT from Glusr_msg_platform_integration (mesh_pg_user connection)
    ▼
Response: current integration status, verification/KYC/quality-rating fields
```

### Flow C — Outbound nudge message (independent of Flow A/B)

```
Internal trigger (e.g. app-engagement job — publisher not traced in this pass)
    │
    ▼
SendWhatsAppAppInstall(ctx, mobile, gluserid, source, modid, uniqueid, db)
    │  whatsapp_handler.go
    │  1. Normalize mobile (+91 prefix)
    │  2. checkWhatsAppFrequency() — DB-backed 24hr check for ProcessAppInstall
    │
    ├─ Not allowed (sent within 24hrs) → ErrWhatsAppFrequencyCapped, no send
    │
    └─ Allowed →
          Determine wa_type (source-based) + content (template vs free-text)
          │
          ▼
          [HTTP POST]  external vendor API (VendorID="78")
          │  pooled httpClient, 5s timeout
          ▼
          Message sent, response parsed
```

---

## 7. Optimization Scope — DB Response-Time Contribution

### High-impact

1. **Frequency-check ek classic Redis-TTL use-case hai, lekin DB se implement hai**
   (§5) — `checkWhatsAppFrequency` ek DB query hai har outbound-send attempt se pehle. Agar
   yeh feature high-volume hai (bahut saare users ko nudges bhejna), Redis
   (`SETNX`/TTL pattern) se yeh check DB-round-trip se kaafi tez ho sakta hai —
   [`utils_programming_guide.md`](../utils_programming_guide.md) mein already documented
   Redis infra reuse ho sakta hai.
2. **Insert-path ka `ON CONFLICT DO UPDATE` on a 38-column table** (§4, point 3) — yeh
   Postgres-level efficient hai (single round-trip), lekin itni badi row ek single statement
   mein maintain karna, agar concurrent writes bade ho, lock-contention ka risk le sakta hai
   — monitoring layak agar write-volume high ho.

### Medium-impact

3. **Update-path mein do sequential queries hain** (`check_glid` SELECT, phir dynamic
   UPDATE) — GST/Rating domains jaisa hi "verify-then-write" pattern, single round-trip mein
   convert nahi kiya gaya (jaise `UPDATE ... WHERE EXISTS` ya `INSERT ... ON CONFLICT`
   pattern se combine ho sakta tha, jaisa insert-path mein already hai).
4. **Status-change aur scam/spam dono independently `glusr_msg_platform_log` mein insert
   kar sakte hain ek hi request mein** (§4, points 5-6) — do alag INSERT statements, agar
   dono condition true ho, do round-trips. Combine karna possible hai agar dono ek saath
   trigger hote hain.

### Low-impact / good practice already present

5. **Pooled HTTP client** (§4, point 10) — achi practice hai already.
6. **Partial-update pattern (only changed fields)** (§4, point 4) — achi practice, poori
   row overwrite nahi karta unnecessarily.

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora WA Msg Integration flowchart yahan dekho](https://lucid.app/lucidchart/6fa94f0b-46d6-49c5-8b0b-bc4b083774a8/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **Status-range numeric codes (`12`-`14`, `15`-`19`) ka exact business-meaning decode
   nahi hua** (§4, point 5) — agar debugging karni pade, in ranges ka actual mapping
   (verified/rejected/suspended/etc.) confirm karo `fk_msg_platform_status_id` master-table
   se.
2. **Frequency-check DB-backed hai, Redis nahi** (§7, point 1) — scalability-risk agar
   volume badhe.
3. **`checkWhatsAppFrequency`/frequency-storage ka exact table iss pass mein trace nahi
   hua** — sirf function-signature confirm hui (leta hai `*sql.DB`).
4. **`SendWhatsAppAppInstall` ka publisher (kaun ise call karta hai) iss pass mein identify
   nahi hua** — koi controller/consumer call-site trace nahi kiya gaya.

---

## 10. Open Questions

1. `fk_msg_platform_status_id` ranges (`12`-`14`, `15`-`19`) ka exact business-meaning kya
   hai?
2. `checkWhatsAppFrequency` kis table/mechanism se pichla-send-record padhta hai?
3. `SendWhatsAppAppInstall` ko kaun-kaun call karta hai (kaunse triggers/jobs)?
4. `MSG_PLATFORM_INTEGRATION` RabbitMQ event ko koi dedicated consumer process karta hai, ya
   sirf generic `comp.sync.*` fan-out tak simit hai?
5. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`WA_Msg_Integration_Business_Doc.md`](./WA_Msg_Integration_Business_Doc.md) — product
  perspective
- [`../users_domain_product_stories.md`](../users_domain_product_stories.md) —
  "Notifications, Alerts & Preferences" story
- [`../utils_programming_guide.md`](../utils_programming_guide.md) — shared Redis/RabbitMQ
  patterns referenced in §7's optimization suggestion

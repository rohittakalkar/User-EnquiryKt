# Popup Details — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`Popup_Details_Business_Doc.md`](./Popup_Details_Business_Doc.md)
dekho — dono docs same flow cover karte hain, bas alag audience ke liye.

**Repos involved**: `service-api-go-production` (write only, confirmed by exhaustive grep across
all three repos — dekho section 1). **Koi read-controller, RabbitMQ, Kafka, Redis, ya consumer
nahi mila** teeno repos (`users-api-go-production`, `service-api-go-production`,
`user-temp-consumers-production`) mein — yeh is poori KT series ka sabse simplest/smallest
feature hai, sirf ek write-only upsert-endpoint.

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file path + line
number diya gaya hai). Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — confirm with team]** likha hai, guess nahi kiya. Is baar ke pass mein purani
(shallow) doc ke saare claims re-verify kiye gaye, plus explicitly grep kiya gaya
(`grep -rli "popup"`) teeno repos mein yeh confirm karne ke liye ki koi read-endpoint ya
consumer miss toh nahi hua.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write (controller) | write | [`PopupDetailsController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/PopupDetailsController.go) |
| Write (model/query) | write | [`PopupDetailsModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PopupDetailsModel.go) (`PopupDetailsModel`) |
| Route registration (x2, do alag Gin engines mein) | write | [`router.go:144,312`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) |
| JWT/AK-validation whitelist entry | write | [`requestValidation.go:28`](../../service-api-go-production/service-api-go-production/pkg/middleware/requestValidation.go) |
| Shared gateway-allowlist helper (`Gateway_v1`) | write | [`globalfunctions.go:595-633`](../../service-api-go-production/service-api-go-production/pkg/utils/globalfunctions.go) |

**Verification of "no read side" claim**: `grep -rli "popup" --include=*.go` chalaya gaya
teeno repos ke against. Jitni bhi files match hui (`UserVcardDisplayModel.go`,
`user-temp-consumers-production/pkg/utils/additional.go`, `MailContent.go`, `map_index.go`),
un sab mein "popup" word generic UI/marketing context mein hai (jaise `mvPopup=1` UTM
query-param email-links mein, ya `"LOGIN WITH EMAIL POPUP"` — ek log-source string), **koi bhi
`TRACK_POPUP_MDC` table ya is feature se related nahi hai**. `grep -rli "TRACK_POPUP"` sirf
`PopupDetailsModel.go` aur doc files mein match hua — koi aur consumer/read-path table ko touch
nahi karta.
[Confirmed via grep, is session mein re-run kiya gaya]

---

## 2. Routes

| Method | Path | serviceName | Repo | Controller |
|---|---|---|---|---|
| POST | `/popupdetails` | `POPUP_DETAILS` | write | `PopupDetailsController` |

**Naya finding is pass mein**: route `/popupdetails` **do baar** register hoti hai, do alag
Gin-engine setup functions mein — `SetupUsersCluster2` aur `SetupUsersFailoverCluster2`
[`router.go:144`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go)
aur
[`router.go:312`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go).
Yeh koi bug nahi hai — baaki saare write-endpoints bhi isi tarah dono cluster-setups mein
register hote hain (primary cluster + failover cluster), same controller-function reference
karte hue. Dono routes `middleware.Validation` ke peeche hain.

---

## 3. Data Model — Table

> **Verification note**: table/column names Go code ke andar embedded SQL string se liye gaye
> hain ([`PopupDetailsModel.go:42`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PopupDetailsModel.go)).
> Live DB schema se pgAdmin pe cross-verify **nahi** kiya gaya hai.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `TRACK_POPUP_MDC` | meshpg (via `config.GetPGDbConnection("meshpg")`, [`PopupDetailsModel.go:19`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PopupDetailsModel.go)) | Latest popup-interest-status per supplier | `fk_glusr_usr_id` (unique — `ON CONFLICT` target), `is_interested` (0/1), `last_entry_date` — [`PopupDetailsModel.go:42`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PopupDetailsModel.go) |

**Note**: table-name suffix `MDC` ka exact matlab code se resolve nahi hua (koi comment,
constant, ya enum documentation nahi mila). Dekho Open Questions.

---

## 4. Decode This: `is_interested` aur Gateway-Resolution Values

Is feature mein koi multi-valued "magic status code" nahi hai jaisa GST ke
`FK_GST_VERIFICATION_SRC_ID` (1-5) hai — sirf ek boolean-jaisa flag hai. Phir bhi, exhaustive
trace neeche hai taaki koi ambiguity na rahe:

| Value | Meaning | Evidence |
|---|---|---|
| `is_interested = "0"` | Supplier **not interested** hai (ya default, agar key hi missing ho) | Controller default `isInterested := "0"` [`PopupDetailsController.go:83`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/PopupDetailsController.go); Model bhi missing-key ko `"0"` default karta hai [`PopupDetailsModel.go:39`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PopupDetailsModel.go) |
| `is_interested = "1"` | Supplier **interested** hai | Sirf yeh do values allowed hain — validation check [`PopupDetailsController.go:109`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/PopupDetailsController.go) |
| Koi aur value (`"2"`, `"yes"`, khaali, etc.) | Reject | `!(isInterested == "0" || isInterested == "1")` → `"Invalid value for is_interested"` [`PopupDetailsController.go:109-110`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/PopupDetailsController.go) |

Yeh field literally boolean hai, koi hidden semantic states nahi — is section mein koi
**[INFERRED]** flag lagane layak cheez nahi mili.

---

## 5. Business Rules & Validation (code se exhaustive)

1. **`glusridval` (GLID) mandatory aur numeric hona chahiye** — pehle empty-check
   (`"Glid should not be blank"`), phir `utils.IsNumeric(glusrid)` check
   (`"Invalid value for glid"`).
   [`PopupDetailsController.go:105-108`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/PopupDetailsController.go)
2. **`is_interested` sirf `"0"` ya `"1"` ho sakta hai** — numeric + exact-match dono check hote
   hain, warna `"Invalid value for is_interested"`.
   [`PopupDetailsController.go:109-110`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/PopupDetailsController.go)
3. **In-app validations DB-call se pehle hote hain** — sab upar wale checks DB-connection open
   hone se pehle chalte hain, isliye invalid requests kabhi DB connection consume nahi karte.
   [`PopupDetailsController.go:101-111`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/PopupDetailsController.go)
4. **Gateway allowlist — 30 entries, is series ka sabse bada allowlist ab tak dekha gaya**:
   `MY`, `M.INDIAMART.COM`, `GLADMIN`, `TOLLFREE`, `Email Marketing`, `HTVENDOR`, `WEBERP`,
   `LEAP`, `IPC-Admin`, `NSD Sales`, `MAPI`, `ENQUIRY`, `FREE-WEBSITE`, `BL`, `TRADE`, `TENDER`,
   `PAYNOW`, `CREDIT ALLOCATION`, `OVP Process`, `SAMPARK Process`, `TOLLFREE Process`,
   `VENDOR CITY Pin Correction`, `Notification Server`, `search`, `FCP`, `PNS`, `Merp`, `IMOB`,
   `SELLERMY`, `PAYWIM`, `BUYERS_FEEDBACK`.
   [`PopupDetailsController.go:15-47`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/PopupDetailsController.go)
5. **`Gateway_v1` resolution logic** — `VALIDATION_KEY` pehle `components.ValidationHashMap`
   ke against reverse-looked-up hota hai (key→value match) resolved-caller-name nikalne ke
   liye, phir yeh naam upar ki 30-entry allowlist ke against check hota hai. Do special-case
   aliasing rules bhi hain jo is codebase-wide hain, is feature ke liye specific nahi:
   `MY`/`BUYERMY` aur `BUYERS_FEEDBACK`/`FULFILLMENT_FEEDBACK` ek doosre ke substitute maane
   jaate hain agar dono allowlist mein se koi ek present ho.
   [`globalfunctions.go:595-633`](../../service-api-go-production/service-api-go-production/pkg/utils/globalfunctions.go)
6. **Double-layer gateway control (naya finding)** — `/popupdetails` do independent gates ke
   peeche hai: (a) outer `middleware.Validation` (JWT/AK based), jiske `whitelist_service` mein
   `/popupdetails` explicitly listed hai — matlab yeh route JWT-AK check se **exempt** hai
   [`requestValidation.go:28,471`](../../service-api-go-production/service-api-go-production/pkg/middleware/requestValidation.go);
   aur (b) andar controller ke apna `Gateway_v1` 30-entry allowlist check. Yaani JWT-level
   security skip hoti hai is endpoint ke liye, lekin ek alag, controller-level shared-secret
   check (`VALIDATION_KEY`) uski jagah leta hai.
7. **Single upsert statement**: `INSERT ... ON CONFLICT (fk_glusr_usr_id) DO UPDATE SET
   is_interested=$2, last_entry_date=CURRENT_TIMESTAMP` — ek hi query first-time aur
   repeat-submission dono handle karti hai, explicit insert-vs-update branching ki zaroorat
   nahi (best-practice pattern, consistent with Social Contact's async-path aur Flips'
   `GLUSR_FLIPS_MAP`).
   [`PopupDetailsModel.go:42`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PopupDetailsModel.go)
8. **No history preserved** — har submission purane `is_interested` value ko overwrite karta
   hai; sirf current/latest status hi kabhi bhi queryable hai is table se.
9. **Query timeout 1 second hard-coded hai** — `queryTimeOut := 1 * time.Second`
   [`PopupDetailsModel.go:43`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PopupDetailsModel.go);
   agar `meshpg` slow ho, request 1s baad hi timeout error return kar dega.
10. **Response format success/failure sirf ek exact string-match se decide hota hai** — controller
    check karta hai `pgOutput == "INSERT SUCCESS"` HTTP code 200/500 decide karne ke liye
    [`PopupDetailsController.go:131`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/PopupDetailsController.go);
    agar model ka success-string kabhi typo/change ho jaaye, yeh silently failure-code return
    karega bina obvious wajah ke — ek fragile string-coupling.

---

## 6. RabbitMQ

**Koi RabbitMQ usage nahi mila.** `PopupDetailsController.go` aur `PopupDetailsModel.go` mein
koi `PushToQueue`/`PubAPI`/`Requeue` call nahi hai. `grep -rli "TRACK_POPUP\|PopupDetails"`
teeno repos ke consumers-code mein bhi kuch nahi laata (dekho section 1) — koi consumer isse
listen nahi karta.

## 7. Kafka

**Nahi.** Koi Kafka reference (`InitializeKafka`, `sub_topic`, `consumer_group`) is feature ke
kisi file mein nahi hai — grep se confirmed.

## 8. Redis

**Nahi.** Koi Redis call (`RedisGet`/`RedisSet`) is feature ke controller ya model mein nahi
hai — grep se confirmed. Read-path hi nahi hai is feature ka, isliye caching ka sawal hi nahi
uthta.

---

## 9. End-to-End Technical Flow

```
Supplier (via any of ~30 allowed internal-apps/systems)
    │
    ▼
[Gin Router]  POST /popupdetails  (serviceName internally = POPUP_DETAILS)
    │  Registered twice: SetupUsersCluster2 (router.go:144) + SetupUsersFailoverCluster2
    │  (router.go:312) — same controller, primary + failover cluster
    ▼
[middleware.Validation]
    │  /popupdetails whitelist_service mein hai → JWT/AK-token validation SKIP hoti hai
    │  (requestValidation.go:28, 471)
    ▼
[Controller]  PopupDetailsController.go
    │  In-app checks: glusridval present+numeric, is_interested in {0,1}
    │  Gateway_v1(popupDetailsHash, VALIDATION_KEY, "k") — 30-entry allowlist check
    │  (yeh hi actual auth-gate hai, kyunki JWT-check skip hua)
    ▼
PopupDetailsModel()
    │  dbConn := GetPGDbConnection("meshpg")
    │  INSERT INTO TRACK_POPUP_MDC (fk_glusr_usr_id, is_interested, last_entry_date)
    │  VALUES ($1,$2,CURRENT_TIMESTAMP)
    │  ON CONFLICT (fk_glusr_usr_id) DO UPDATE SET is_interested=$2, last_entry_date=NOW()
    │  (query timeout: 1 second, hard-coded)
    ▼
[DB — meshpg]  TRACK_POPUP_MDC  (single round-trip)
    ▼
Response {"STATUS":"SUCCESS","CODE":"200","MESSAGE":"INSERT SUCCESS","SERVICE_NAME":"POPUP_DETAILS"}
    (ya STATUS=FAILURE/CODE=500 agar pgOutput != "INSERT SUCCESS")
    ▼
[Logging]  KibanaLogging_v1(...) — execution-time metrics + PG_OUTPUT + gateway + glid log hote hain
```

---

## 10. Flow-wise DB & Table Usage

Is feature mein sirf **ek** flow hai (upsert), aur usmein sirf **ek** DB round-trip.

| # | DB (physical) | Table | Operation | Kya ho raha hai, aur kyun |
|---|---|---|---|---|
| 1 | meshpg (`GetPGDbConnection("meshpg")`) | `TRACK_POPUP_MDC` | `INSERT ... ON CONFLICT DO UPDATE` (single statement) | Supplier ka latest interest-status upsert hota hai — first-time insert ho ya repeat-update, dono same query se handle hote hain |

**Total DB round-trips per request: 1** — is poori KT series mein confirmed sabse simple/cheap
DB-usage pattern (compare karo GST ke Flow A ke 9-10 round-trips se, section 11 GST_Technical_Doc.md mein).

---

## 11. Optimization Scope — DB Response-Time Contribution

### Low impact / good practice already present

1. **Single `ON CONFLICT` upsert-statement** — already sabse efficient pattern is kism ke
   latest-status-only write ke liye; round-trip-reduction ka koi scope nahi hai yahan.
2. **In-app validation DB-connection khulne se pehle** — invalid `is_interested`/`glusridval`
   values DB ko touch kiye bina hi reject ho jaati hain, wasted connections avoid hote hain.
3. **1-second query timeout already set hai** — ek slow/hung query poori request ko indefinitely
   block nahi kar sakti; yeh achi defensive practice hai.
4. **No async/consumer complexity** — sabse simplest, lowest-risk architecture is poori KT
   series mein: ek hi synchronous write, koi fan-out, koi retry-queue, koi cross-DB
   consistency-problem nahi.

### Watch-points (low-risk, but worth noting)

5. **Fragile string-based success detection** (section 5.10) — agar kabhi
   `PopupDetailsModel`'s success-string `"INSERT SUCCESS"` refactor mein badal jaaye, controller
   silently `CODE=500` return karega bina koi runtime-error ke. Concrete low-effort fix: model
   se ek boolean/error return karo string-compare ki jagah.
6. **Har request pe naya DB connection resolve hota hai** (`GetPGDbConnection("meshpg")` per
   call) — agar yeh internally connection-pool se milta hai (jaisa GST domain mein hai,
   `SetMaxOpenConns`), toh koi issue nahi; agar nahi, high-traffic internal-callers (30 allowed
   systems) ke saath connection-churn ban sakta hai. **[INFERRED — `GetPGDbConnection` ka
   internal pooling-behavior is review mein trace nahi kiya gaya, `config` package se confirm
   karo]**.

---

## 12. Cron Inventory

Grep kiya gaya `user-temp-consumers-production` mein kisi bhi cron/scheduled-binary ke liye jo
`TRACK_POPUP_MDC` ya `PopupDetails` reference karta ho — **koi nahi mila**. Yeh feature ke saath
koi cron associated nahi hai.

---

## 13. Edge Cases & Gotchas (technical POV)

1. **No history** — agar kabhi business-need aaye yeh dekhne ki *kab* supplier ne apna
   interest-status multiple baar badla, yeh table isse answer nahi kar sakti; sirf latest
   value survive karti hai.
2. **Table-name `MDC` suffix unexplained** — koi bhi future debugging/dashboard work jo is
   table ko reference kare, pehle "MDC" ka matlab confirm kare.
3. **JWT validation skip + separate gateway-check** (section 5.6) — is route ke liye security
   model baaki JWT-protected routes se fundamentally different hai. Agar koi future engineer
   assume kare ki har route JWT-protected hai, yeh galat hoga is specific endpoint ke liye.
4. **Route duplicate-registered do cluster-setups mein** (section 2) — normal pattern hai is
   codebase mein (baaki endpoints bhi aise hi hain), lekin agar kabhi failover-cluster ka
   `PopupDetailsController` reference accidentally divergent ho jaaye primary se, dono clusters
   silently different behavior de sakte hain.
5. **Hard 1-second query timeout** — agar meshpg kabhi thoda slow ho (jaise heavy load ke
   waqt), yeh timeout baaki queries se bahut tight hai; monitor karo agar timeout-errors kabhi
   spike karein.

---

## 14. Open Questions

1. `TRACK_POPUP_MDC` — "MDC" ka exact matlab kya hai, aur konsa specific popup/campaign yeh
   track karta hai? Code se resolve nahi ho paya.
2. Konsa screen/app supplier ko actually yeh popup dikhata hai — client-side/frontend concern
   hai, is codebase se traceable nahi.
3. Kya is data ka koi downstream consumer hai (reporting, targeting, marketing dashboard) in
   teen repos ke bahar? Is review mein nahi mila.
4. `GetPGDbConnection("meshpg")` ka internal connection-pooling behavior — per-request naya
   connection banta hai ya pool se milta hai? (section 11.6)
5. Live DB schema verification — yeh doc sirf Go SQL string jo imply karti hai wahi reflect
   karta hai (`fk_glusr_usr_id`, `is_interested`, `last_entry_date`); koi live pgAdmin
   cross-check nahi kiya gaya.

---

## See also

- [`Popup_Details_Business_Doc.md`](./Popup_Details_Business_Doc.md) — product perspective
- [`../utils_programming_guide.md`](../utils_programming_guide.md) — shared `Gateway_v1`,
  `ExecuteQuery`, `PushToQueue`/Kafka/Redis helpers jo iss doc mein reference hue hain
- [`../user_consumers_reference.md`](../user_consumers_reference.md) — poori consumer
  inventory (confirms koi Popup-related consumer nahi hai)

# Buyer Blocking — Technical Doc (Code-Level Deep Dive)

**Scope note**: yeh doc sirf **Blocking** cover karta hai — **Matchmaking** alag module hai,
dekho [`../Matchmaking KT/Matchmaking_Technical_Doc.md`](../Matchmaking%20KT/Matchmaking_Technical_Doc.md).

Business/product perspective ke liye
[`Blocking_Business_Doc.md`](./Blocking_Business_Doc.md) dekho.

**Repos**: `service-api-go-production` (write), `users-api-go-production` (read, indirectly
via Buyer Profile). `user-temp-consumers-production` grep kiya gaya — koi block/unblock
related consumer nahi mila (dekho section 5).

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file path diya
gaya hai har jagah). Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — team se confirm karo]** likh diya hai, guess nahi kiya.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Block/Unblock submit (controller) | write | [`UserBlockUnblockController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserBlockUnblockController.go) |
| Block/Unblock submit (DB upsert) | write | [`UserBlockUnblockModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserBlockUnblockModel.go) |
| Mandatory-field validation | write | [`UserUtilsMandatory.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go) (`MandatoryParamsCheckUserBlockUnblock`, lines 848-910) |
| Route registration (primary + failover router) | write | [`router.go`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) — `SetupUsers...` (line 146) aur `SetupUsersFailoverCluster2` (line 314), dono mein identical registration |
| Read usage — enforcement point (buyer-profile, embedded read) | read | [`UserBuyerProfileModel.go`](../internal/models/users/UserBuyerProfileModel.go) — `checkMatchMaking()` (line 535-570) LEFT JOINs `USER_BLOCKED_STATUS` with the matchmaking table (`glusr_contactbook_mapping`) in one query, aur block-status **contact-info ko actively hide karta hai** (dekho section 4.5 — yeh ek naya finding hai, purane doc mein sirf "display" bola gaya tha) |

**Re-verified claim from previous doc pass**: is module ka poora write-side footprint sach
mein sirf 2 files hai (controller + model), plus 1 shared validation function. Read-side
1 file mein consumed hota hai. Grep se confirm hua ki `USER_BLOCKED_STATUS` / block-unblock
logic in dono repos combined sirf inhi 3 code-files mein reference hoti hai (docs ko chhodke).
Yeh abhi bhi **sabse chhota, purely synchronous** feature hai poore KT-series mein — GST
(15+ files, 6 consumers) ya Rating jaise multi-consumer domains ke muqable.

---

## 2. Routes

| Method | Path | Repo | Controller |
|---|---|---|---|
| POST | `/user/user_block_unblock` | write | `UserBlockUnblockController` |

Route do jagah register hoti hai — `SetupUsersFailoverCluster` (primary) aur
`SetupUsersFailoverCluster2` (failover) — [`router.go:146,314`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go).
Yeh ek standard pattern hai iss repo ki har route ke liye (failover-cluster redundancy),
block/unblock-specific nahi hai.

Koi dedicated GET endpoint nahi hai — block-status sirf `GET /buyerprofile` ke andar
embedded milta hai (ek JOIN ke through, section 7 dekho).

---

## 3. Data Model — Table

> **Verification note**: yeh table/column names Go code ke andar embedded SQL strings se
> liye gaye hain. Live DB schema se pgAdmin pe cross-verify **nahi** kiya gaya hai.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `USER_BLOCKED_STATUS` | meshpg | **Primary aur sirf record** — ek row per (actor, target) pair | `user_glid` (actor — jisne block/unblock action liya), `blocked_glid` (target — jisko block/unblock kiya gaya), `block_status` (1=blocked, 0=unblocked), `blocking_date`, `unblocking_date`, `mod_id` (gateway/caller identity), `user_ip` — [`UserBlockUnblockModel.go:34`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserBlockUnblockModel.go) |

**Primary key**: `(user_glid, blocked_glid)` — confirmed via `ON CONFLICT(user_glid,
blocked_glid) DO UPDATE` clause, [`UserBlockUnblockModel.go:34`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserBlockUnblockModel.go).

**Important semantic note (naya finding is pass mein)**: table/column naming (`user_glid` =
actor, `blocked_glid` = target) **generic hai — koi enforced constraint nahi hai ki `user_glid`
hamesha "supplier" hi ho**. Controller/model layer mein koi role-check nahi hai jo confirm
kare ki caller supplier hai ya buyer. Section 7 (read-side integration) mein dikhta hai ki
`USER_BLOCKED_STATUS` actually **buyer → supplier direction** mein bhi consult hoti hai
(buyer ne supplier ko block kiya), na ki sirf "supplier blocks buyer" — dekho section 4.5 aur
Open Questions #1.

---

## 4. Business Rules & Validation (code se, exhaustive)

1. **`user_glid` aur `blocked_glid` dono numeric hone chahiye** — agar non-numeric string
   aaye, silently `"INVALID_VALUE"` mark ho jaata hai jo aage reject hota hai
   (`" INVALID USER_GLID/BLOCKED_GLID."`).
   [`UserUtilsMandatory.go:850-871,902-903`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
2. **`blocked_status` sirf `1` ya `0` ho sakta hai** — koi aur numeric value (jaise `2`)
   `"INVALID_VALUE"` mark hoti hai, error: `" INVALID BLOCKED_STATUS"`.
   [`UserUtilsMandatory.go:872-885,904-905`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
3. **5 mandatory fields**: `user_glid`, `blocked_glid`, `blocked_status`, `VALIDATION_KEY`,
   `user_ip` — koi bhi khaali ho toh combined error message
   `"Please Enter Mandatory(USER_GLID/BLOCKED_GLID/BLOCKED_STATUS/VALIDATION KEY/IP) Fields"`.
   [`UserUtilsMandatory.go:900-901`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
4. **`user_ip` bhi validate hoti hai** `utils.IPValidation()` se — agar IP format invalid ho,
   iski error mandatory-fields check se bhi pehle priority leti hai.
   [`UserUtilsMandatory.go:897-899`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
5. **Gateway allowlist bada hai** (10 callers: `GLADMIN`, `SELLERMY`, `MY`, `MY1`, `IMOB`,
   `PNS`, `ANDROID`, `IOS`, `ANDWEB`, `IOSWEB`) — matlab yeh feature bahut saare
   client-surfaces se accessible hai (web, app, admin panel sab), koi role-restriction nahi
   (e.g. sirf supplier ya sirf buyer app se hi allowed ho, aisa koi filter code mein nahi hai).
   [`UserBlockUnblockController.go:54`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserBlockUnblockController.go)
6. **`MY1` gateway ko response mein `MY` mein normalize kiya jaata hai** (sirf logging/response
   ke liye, DB write mein already-resolved `gate` value use hoti hai jo abhi `MY1` hi thi jab
   `UpsertBlockunblock` call hua) — ek chhota caller-identity-mapping quirk.
   [`UserBlockUnblockController.go:58-62`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserBlockUnblockController.go)
7. **`blocking_date`/`unblocking_date` column dynamically choose hoti hai** — `blocked_status
   == "1"` ho toh sirf `blocking_date` set/update hoti hai, `== "0"` ho toh sirf
   `unblocking_date` — matlab jab tum unblock karte ho, `blocking_date` **wahi purani reh
   jaati hai** (overwrite nahi hoti), aur vice versa. Dono columns independently apna-apna
   last-action-timestamp maintain karte hain.
   [`UserBlockUnblockModel.go:28-35`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserBlockUnblockModel.go)
8. **Ek single `UPSERT` (`INSERT ... ON CONFLICT DO UPDATE`) poora operation handle karta
   hai** — pehli baar block karna ho ya dobara block/unblock, sab isi ek query se ho jaata
   hai. Koi separate insert-vs-update branching logic nahi hai application-code mein — DB
   khud decide karta hai.
   [`UserBlockUnblockModel.go:34-35`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserBlockUnblockModel.go)
9. **Query timeout hardcoded 1 second** — `ExecuteQuery(meshpgconn, sql, input_params, 1*time.Second)`.
   [`UserBlockUnblockModel.go:44`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserBlockUnblockModel.go)
10. **Response string-matching se success/fail decide hota hai**, koi typed error/status code
    nahi — controller literal string `"UPDATE SUCCESS"` ke against compare karta hai HTTP
    `200`/`500` decide karne ke liye.
    [`UserBlockUnblockController.go:72-78`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserBlockUnblockController.go)

### 4.5 — Enforcement point (naya finding, purane doc mein missing tha)

Purana doc claim karta tha ki `UserBuyerProfileModel.go` block-status ko sirf **display**
karta hai, enforcement kahin nahi milta. **Yeh deeper trace mein galat nikla.** Enforcement
actually yahin, isi file mein milta hai:

`checkMatchMaking()` [`UserBuyerProfileModel.go:535-570`](../internal/models/users/UserBuyerProfileModel.go)
ek query chalata hai:
```sql
SELECT g.source_id, COALESCE(u.block_status,0) AS block_status
FROM (SELECT GLUSR_USR_ID1,GLUSR_USR_ID2,COALESCE(source_id,0) AS source_id
      FROM glusr_contactbook_mapping
      WHERE GLUSR_USR_ID1 = $1 AND GLUSR_USR_ID2 = $2) g
LEFT JOIN USER_BLOCKED_STATUS u ON u.blocked_glid = GLUSR_USR_ID1 AND u.user_glid = GLUSR_USR_ID2;
```
yahan `$1 = sup_glid` (viewing supplier), `$2 = buy_glid` (buyer whose profile is being
viewed) — [`UserBuyerProfileModel.go:70`](../internal/models/users/UserBuyerProfileModel.go).
Iska matlab join condition resolve hoti hai `u.blocked_glid = sup_glid AND u.user_glid =
buy_glid` — yaani yeh row check karta hai **"kya buyer (`user_glid`) ne supplier
(`blocked_glid`) ko block kiya hai"**, ulta nahi.

`checkMatchMaking` ka return-value order hai `(valid int, match_typ int, block string, ...)`
[`UserBuyerProfileModel.go:535`](../internal/models/users/UserBuyerProfileModel.go), lekin
caller mein assignment hai:
```go
is_valid, match_typ, valid, response, log_map, q1_time = checkMatchMaking(...)
```
[`UserBuyerProfileModel.go:70`](../internal/models/users/UserBuyerProfileModel.go) —
positional assignment ki wajah se caller ka **`valid` (string) variable actually `block`
value hold karta hai**, naam se confuse mat hona. Yeh `valid` string phir
`getBuyerProfileData()` mein pass hoti hai, aur wahan:
```go
if valid == "1" {
    FinalResultMap["contacts_mobile1"] = nil
    FinalResultMap["contacts_email1"] = nil
    FinalResultMap["glusr_usr_im_gsm"] = nil
    FinalResultMap["contacts_mobile2"] = nil
    FinalResultMap["contacts_landline"] = nil
    FinalResultMap["contacts_email2"] = nil
    FinalResultMap["contact_number"] = nil
    FinalResultMap["glusr_usr_ph_country"] = nil
    FinalResultMap["glusr_usr_ph_area"] = nil
    FinalResultMap["is_email_verified"] = nil
    FinalResultMap["is_alternate_mobile_verified"] = nil
    FinalResultMap["is_alternate_email_verified"] = nil
    FinalResultMap["verification_status"] = nil
}
```
[`UserBuyerProfileModel.go:452-466`](../internal/models/users/UserBuyerProfileModel.go) — sab
contact/verification fields poori tarah `nil` kar diye jaate hain.

**Confirmed enforcement (revised from previous doc)**: jab **buyer ne is supplier ko block
kiya ho**, `GET /buyerprofile` response mein buyer ke saare contact-details aur
verification-status fields blank aa jaate hain us supplier ke liye. Yeh ek real, active
enforcement point hai — sirf "display" nahi. Jo cheez confirm nahi ho paayi: kya "supplier
blocks buyer" direction (i.e. `UserBlockUnblockController` ka typical documented use-case)
kahin bhi enforce hoti hai — is query ka join condition sirf buyer→supplier direction check
karta hai. **[INFERRED — team se confirm karo: kya "supplier blocks buyer" ka koi alag
enforcement point hai jo iss review mein nahi mila, ya table symmetric use hoti hai dono
directions ke liye alag-alag features mein?]**

---

## 5. RabbitMQ — Confirmed Absent

**Grep se confirmed: koi RabbitMQ publish/consume iss module mein nahi hai.**
`UserBlockUnblockController.go` aur `UserBlockUnblockModel.go` mein koi `PushToQueue`/
`PubAPI`/`Requeue` call nahi hai — sirf `config`, `utils`, `time`, `newrelic` imports hain.
`user-temp-consumers-production` repo poora grep kiya gaya `USER_BLOCKED_STATUS` /
`block_status` / `blocked_glid` / `BlockUnblock` ke liye — **zero matches**. Koi dedicated
consumer iss feature ke liye exist nahi karta.

## 6. Kafka — Confirmed Absent

**Koi Kafka usage nahi mila.** Same grep-scope (dono repos) — koi `InitializeKafka` ya
Kafka-related import iss module ke kisi bhi file mein nahi hai.

## 7. Redis — Confirmed Absent

**Koi Redis usage nahi mila.** Na write-side (`UserBlockUnblockModel.go`), na read-side
(`checkMatchMaking`/`getBuyerProfileData` — `USER_BLOCKED_STATUS` read seedha live Postgres
se hoti hai, har baar).

---

## 8. End-to-End Technical Flows

### Flow A — Block ya Unblock submit

```
Supplier/Buyer (jo bhi caller ho — koi role-check nahi hai)
    │
    ▼
[API — write]  POST /user/user_block_unblock
    {user_glid, blocked_glid, blocked_status, VALIDATION_KEY, user_ip}
    │  UserBlockUnblockController.go
    │  1. FetchRequestParams + FormatInputParams
    │  2. MandatoryParamsCheckUserBlockUnblock() — numeric checks, blocked_status ∈ {0,1},
    │     5 mandatory fields, IP format
    │  3. Gateway_v1() — 10-caller allowlist validate
    ▼
[DB — single upsert]  meshpg
    INSERT INTO USER_BLOCKED_STATUS(user_glid,blocked_glid,block_status,
        blocking_date OR unblocking_date, mod_id, user_ip)
    VALUES(...)
    ON CONFLICT(user_glid, blocked_glid) DO UPDATE
    SET block_status=EXCLUDED.block_status, <date-col>=EXCLUDED.<date-col>,
        mod_id=EXCLUDED.mod_id, user_ip=EXCLUDED.user_ip
    (1-second query timeout)
    ▼
Response: "UPDATE SUCCESS" (HTTP 200) ya failure message (HTTP 500)
    │  Kibana logging: USER_GLID, OUTPUT, timestamps, query-exec-time, gate
```

**Poori pipeline ek hi HTTP request ke andar complete ho jaati hai** — koi asynchronous
step nahi, koi eventual-consistency concern nahi, koi email/notification.

### Flow B — Block-status enforcement (Read-side, buyer profile)

```
Supplier
    │
    ▼
[API — read]  GET /buyerprofile  (k=encrypted buyer-identifier)
    │  UserBuyerProfileModel.go → GetBuyerProfileModel()
    │  1. Decrypt `k` → buyerglid; validate against supplier_glusrid
    ▼
[DB]  checkMatchMaking(sup_glid, buy_glid)
    SELECT g.source_id, COALESCE(u.block_status,0) AS block_status
    FROM glusr_contactbook_mapping g
    LEFT JOIN USER_BLOCKED_STATUS u
        ON u.blocked_glid = GLUSR_USR_ID1 AND u.user_glid = GLUSR_USR_ID2
    WHERE GLUSR_USR_ID1=sup_glid AND GLUSR_USR_ID2=buy_glid
    │  — matchmaking source_id AUR block_status (buyer→supplier direction) dono
    │    ek hi query mein milte hain
    ▼
[Application logic]  getBuyerProfileData()
    │  agar block_status == "1" (buyer blocked this supplier):
    │      → contacts_mobile1/2, email1/2, landline, contact_number, ph_country/area,
    │        is_email_verified, is_alternate_*_verified, verification_status
    │        SAB nil kar diye jaate hain response mein
    ▼
Response: buyer profile — contact fields present ya blank, block ke basis pe
```

**Yeh Flow A se completely decoupled hai (koi shared code path nahi, alag repo, alag DB
call)** — Flow A sirf ek row likhta hai, Flow B independently usi row ko baad mein padhta hai
jab bhi koi supplier us buyer ka profile khole.

---

## 9. Flow-wise DB & Table Usage — Kaun sa DB, Kaun sa Table, Kis Liye

### Flow A — Block/Unblock Submit (1 write, 1 DB)

| # | DB (physical) | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `USER_BLOCKED_STATUS` | UPSERT (`INSERT ... ON CONFLICT DO UPDATE`) | Block ya unblock action ka single-row record — status, timestamp (block ya unblock date), caller identity (`mod_id`), `user_ip` |

**Total DB round-trips: 1.** Yeh poore KT-series mein sabse chhota write-path hai — GST Flow
A ke 9-10 round-trips ke muqable, Blocking sirf 1 hai.

### Flow B — Buyer Profile Read (block-status enforcement) — table sirf ek chhota hissa hai ek bade multi-query read ka

| # | DB (physical) | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | buyer_profile_pg | `glusr_contactbook_mapping` + `USER_BLOCKED_STATUS` (LEFT JOIN, ek hi query) | SELECT | Matchmaking `source_id` aur block-status dono ek round-trip mein nikale jaate hain — `checkMatchMaking()` |
| 2-4 | buyer_profile_pg | `GLUSR_USR` + ~8 more joined tables (registrations, logo, rating, trustseal, activity-stats, social-contact, ext), 3 parallel goroutine queries | SELECT | Poora buyer-profile data — block-status khud iski koi alag query nahi leta, sirf step 1 ka result (`valid` string) yahan pass ho ke fields blank karta hai |

**Note**: `USER_BLOCKED_STATUS` ka `GET /buyerprofile` mein sirf 1 round-trip contribution hai
(step 1 ka hissa) — baaki saara query-load poore buyer-profile feature ka hai, blocking ka
nahi. Isliye blocking ka marginal DB-cost is read pe negligible hai.

---

## 10. Optimization Scope — DB Response-Time Contribution

Yeh sabse chhota domain hai poore KT-series mein — optimization scope bhi sabse chhota hai,
kyunki architecture already minimal hai. Concrete, code-cited findings neeche:

### Medium-impact

1. **Koi index-hint ya explicit index confirmation nahi mila `USER_BLOCKED_STATUS` pe** — join
   condition `u.blocked_glid = GLUSR_USR_ID1 AND u.user_glid = GLUSR_USR_ID2` do columns pe
   equality hai. Primary key already `(user_glid, blocked_glid)` hai (`ON CONFLICT` clause se
   confirmed, section 3), toh agar PK composite-index `(user_glid, blocked_glid)` order mein
   hai, yeh join efficient honi chahiye — lekin **join column order query mein REVERSE hai**
   (`blocked_glid` pehle compare hota hai `user_glid` se, phir `user_glid` compare hota hai
   `blocked_glid` se) [`UserBuyerProfileModel.go:547`](../internal/models/users/UserBuyerProfileModel.go).
   Agar underlying index sirf `(user_glid, blocked_glid)` order mein hai (as PK likely is),
   Postgres planner still isse use kar sakta hai equality predicates ke liye (order
   irrelevant hai equality ke liye), lekin **live schema/EXPLAIN se confirm karna chahiye**
   ki koi sequential scan toh nahi ho raha is table pe.
2. **1-second hardcoded query timeout** (section 4.9) har environment (dev/staging/prod) ke
   liye same hai — agar meshpg kabhi slow ho, koi env-specific tuning nahi hai. Low-risk
   given yeh ek single-row upsert hai, lekin worth noting agar consistently timeouts dikhein
   monitoring mein.
3. **`checkMatchMaking()` har `GET /buyerprofile` call pe live query karta hai, koi caching
   nahi** — block-status frequently change nahi hoti (yeh ek manual, infrequent supplier/buyer
   action hai), toh yeh ek theoretical caching candidate hai agar `GET /buyerprofile` high
   QPS pe ho — lekin yeh sirf 1 query hai jo already ek bade multi-query read ka chhota hissa
   hai (section 9), isliye standalone optimization ka ROI low hai jab tak poora buyer-profile
   read khud optimize na ho raha ho.

### Low-impact / good-practice already present

4. **Single-query upsert already optimal hai write-side pe** — koi extra round-trip, koi
   redundant SELECT-before-INSERT pattern (jo GST/Rating domains mein dikha tha). Yeh sabse
   clean write-path hai teeno KT-series mein.
5. **`GET /buyerprofile`'s LEFT JOIN pattern efficient hai** — matchmaking aur blocking
   status ek hi query mein milte hain, do alag API calls ki zaroorat nahi.

**Overall**: iss module ke liye koi urgent optimization recommend nahi ki jaa rahi hai — yeh
already ek achi minimal, single-round-trip write design hai; read-side ka enforcement bhi ek
hi extra join se ho jaata hai, standalone koi bada cost-center nahi hai.

---

## 11. Cron Inventory

**Koi cron iss feature ko touch nahi karta.** `service-api-go-production/crons/` directory
grep ki gayi block/blocked/blocking keywords ke liye — **zero matches**. GST jaisa koi daily
re-verification/reconciliation job Blocking ke liye exist nahi karta — yeh consistent hai iss
feature ke purely-manual, purely-synchronous nature ke saath (koi background
verification/re-processing chahiye hi nahi, kyunki state khud hi authoritative hai jo
supplier/buyer ne set kiya).

---

## 12. Edge Cases & Gotchas (technical POV)

1. **`USER_BLOCKED_STATUS` naming asymmetric hai apni actual usage se** — column naam
   generic hain (`user_glid`=actor, `blocked_glid`=target), aur koi role-enforcement nahi hai
   controller/model level pe. Section 4.5 mein confirmed enforcement path **buyer→supplier**
   direction check karta hai. Agar koi assume kare "yeh table sirf supplier-blocks-buyer ke
   liye hai," woh galat ho sakta hai — same table dono directions ke liye reusable hai jo bhi
   caller `user_glid`/`blocked_glid` ke roop mein khud ko/target ko pass kare.
2. **`blocked_status` positional-variable confusion** (section 4.5) — `checkMatchMaking()` ka
   3rd return value naam `block` hai lekin caller mein ek variable naam `valid` mein assign
   hota hai jo khud function ke andar bhi `valid` naam se use hota hai — do alag "valid"
   concepts (contactbook-match-valid vs. block-status) same identifier "valid" share karte
   hain do alag scopes mein. Future debugging mein confusing ho sakta hai.
3. **`blocking_date`/`unblocking_date` independently persist hote hain** (section 4.7) — agar
   koi query sirf latest action ka timestamp chahta ho, `GREATEST(blocking_date,
   unblocking_date)` jaisa logic chahiye hoga, single column read kaafi nahi hoga.
4. **Route registered do jagah identically** (`SetupUsersFailoverCluster` aur
   `SetupUsersFailoverCluster2`, section 2) — yeh poore repo ka standard pattern hai
   (failover redundancy), Blocking-specific nahi, lekin agar future mein route logic change
   karna ho, dono jagah sync mein rakhna padega.
5. **"Supplier blocks buyer" ka koi confirmed enforcement point nahi mila** — sirf
   buyer→supplier direction ka enforcement `GetBuyerProfileModel` mein mila (section 4.5).
   Agar business-side yeh expect karti hai ki supplier bhi ek buyer ko block kare aur uska
   koi visible/enforced effect ho (jaise lead-delivery filter), **woh enforcement point iss
   review mein nahi mila** — dekho Open Questions.

---

## 13. Open Questions

1. **"Supplier blocks buyer" direction ka enforcement kahan hota hai?** Confirmed enforcement
   (section 4.5, 12.5) sirf buyer→supplier direction check karta hai
   (`GET /buyerprofile` mein). Kya supplier-side blocking (jo business doc mein primary
   use-case describe hoti hai) kisi lead-delivery/routing system mein consult hoti hai jo iss
   codebase review ke scope se bahar hai (jaise enq-consumers ya kisi alag lead-matching
   service mein)? **[INFERRED — team se confirm karo]**
2. Kya blocking ka koi upper-limit hai (ek user kitne dusre users ko block kar sakta hai)?
   Code mein koi aisi limit nahi mili.
3. `USER_BLOCKED_STATUS` table pe indexes/constraints ka live schema verification (section 3,
   10.1) — yeh doc sirf Go SQL strings jo imply karti hain wahi reflect karta hai.
4. Kya `USER_BLOCKED_STATUS` kisi aur feature/API mein bhi consult hoti hai jo iss review ke
   file-map (section 1) mein cover nahi hui — jaise kisi admin-panel report ya analytics
   pipeline mein? Grep ne sirf 3 code-files match kiye, lekin agar naming inconsistent ho
   kisi third location mein, woh miss ho sakta hai.

---

## See also

- [`Blocking_Business_Doc.md`](./Blocking_Business_Doc.md) — product perspective
- [`../Matchmaking KT/Matchmaking_Technical_Doc.md`](../Matchmaking%20KT/Matchmaking_Technical_Doc.md) —
  related independent module, same read-query intersection point (`UserBuyerProfileModel.go`,
  `checkMatchMaking()`)
- [`../buyer_seller_discovery_matching_read_write_picture.md`](../buyer_seller_discovery_matching_read_write_picture.md) —
  original architecture trace this doc builds on

# Buyer-Supplier Matchmaking — Technical Doc (Code-Level Deep Dive)

**Scope note**: yeh doc sirf **Matchmaking** cover karta hai — **Blocking** alag module hai,
dekho [`../Blocking KT/Blocking_Technical_Doc.md`](../Blocking%20KT/Blocking_Technical_Doc.md).

Business/product perspective ke liye
[`Matchmaking_Business_Doc.md`](./Matchmaking_Business_Doc.md) dekho.

**Repos**: `service-api-go-production` (write), `user-temp-consumers-production`
(consumers).

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Matchmaking record submit | write | [`BsMatchMakingController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/BsMatchMakingController.go), [`BsMatchMakingModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/BsMatchMakingModel.go) |
| Kafka consume + DB write (mirror copy) | consumers | [`USER_BS_MATCHMAKING_KAFKA.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_KAFKA.go) |
| LMS sync consumer | consumers | `USER_BS_MATCHMAKING_LMS.go` *(registered as `UserBsMatchLMS` in router — content not re-read in this pass)* |
| Consumed by (downstream) | consumers | [`USER_RATING_BS_MATCHMAKING.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_BS_MATCHMAKING.go) — Rating pipeline's anti-fraud check reads this data |
| Read usage (buyer profile) | read | [`UserBuyerProfileModel.go`](../internal/models/users/UserBuyerProfileModel.go) — LEFT JOINs matchmaking table with blocking table in one query |

---

## 2. Routes

| Method | Path | Repo | Controller |
|---|---|---|---|
| POST | `/user/bsmatchmaking` | write | `BsMatchMakingController` |

Koi dedicated GET endpoint nahi hai — matchmaking status sirf doosre reads (jaise
`GET /buyerprofile`) ke andar embedded milta hai.

---

## 3. Data Model — Tables

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_CONTACTBOOK_MAPPING` | buyerprofilepg (write, synchronous) + meshPg (consumer, async mirror) | **Primary matchmaking record** — ek row per buyer-supplier connection | Written via stored procedure `sp_insert_contacts(glid_1, glid_2, contact_type, s_id)` — [`BsMatchMakingModel.go:54`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/BsMatchMakingModel.go); consumer-side columns: `GLUSR_USR_ID1`, `GLUSR_USR_ID2`, `SOURCE_ID`, `IS_DUAL_SIDE_ADDED` — [`USER_BS_MATCHMAKING_KAFKA.go:75-79`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_KAFKA.go) |

**Important architecture note (already documented earlier this session)**: yeh table **do
alag physical databases** mein independently likhi jaati hai — ek baar synchronously
(`buyerprofilepg`, API request ke andar hi, Step 1) aur ek baar asynchronously (`meshPg`, via
Kafka consumer, Step 2). Same stored procedure, do alag connections — ek replication pattern.

---

## 4. Business Rules & Validation (code se)

1. **Mandatory fields**: `buyer_id`, `supplier_id`, `VALIDATION_KEY` — inke bina
   `"Please Enter Mandatory(buyer_id / supplier_id / VALIDATION_KEY) Fields"`.
   [`BsMatchMakingController.go:78`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/BsMatchMakingController.go)
2. **Gateway validation sirf `"LEADS"` allowlist ke against hoti hai** — ek chhota,
   focused caller-list (baaki controllers ke bade allowlists ke muqable).
   [`BsMatchMakingController.go:81`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/BsMatchMakingController.go)
3. **`insertion_type` (contact_type) default `"2"` hai** agar nahi diya gaya — matlab is
   parameter ka exact business-meaning (1 vs 2 vs koi aur) iss pass mein trace nahi hua.
   [`BsMatchMakingModel.go:40-46`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/BsMatchMakingModel.go)
4. **DB write pehle hoti hai, Kafka publish uske baad — dono ek hi API request mein**:
   `sp_insert_contacts` seedha `buyerprofilepg` pe synchronously chalta hai (Step 3 se
   pehle hi record ban chuka hota hai), phir Kafka topic
   `soa-user-bs_matchmaking-csl_glid_logs` pe ek notification jaati hai
   (`SERVICENAME: "USER_BS_MATCHMAKING"`). Kafka publish fail bhi ho jaaye, API response
   fir bhi `"INSERT SUCCESS"` deta hai — kyunki primary write already ho chuka tha.
   [`BsMatchMakingModel.go:71-92`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/BsMatchMakingModel.go)
5. **Consumer-side dobara validation hoti hai** — Kafka message ke `GLUSR_USR_ID1`/
   `GLUSR_USR_ID2` khaali ya `> 10` characters ho toh reject, chahe API-side validation
   pass ho chuki ho (defensive, cross-layer trust nahi karta).
   [`USER_BS_MATCHMAKING_KAFKA.go:80`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_KAFKA.go)
6. **`SOURCE_ID == "3"` ya `"4"` special-cased hai consumer mein** — inn source IDs ke liye
   DB write hi skip ho jaata hai, sirf ek "already inserted by WRITE SERVICE" success-log
   likha jaata hai. Matlab kuch specific sources ke liye consumer sirf logging karta hai,
   duplicate-write avoid karne ke liye.
   [`USER_BS_MATCHMAKING_KAFKA.go:83-85`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_KAFKA.go)
7. **Consumer-side DB write mein retry logic hai** — deadlock ya failure pe, 500ms se
   badhte interval ke saath (max 10 min tak) retry karta hai, exponential backoff jaisa.
   [`USER_BS_MATCHMAKING_KAFKA.go:136-149`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_KAFKA.go)

---

## 5. Kafka — Primary Messaging (yeh domain RabbitMQ nahi, Kafka use karta hai)

| Topic/Queue | Publisher | Consumer | Purpose |
|---|---|---|---|
| `soa-user-bs_matchmaking-csl_glid_logs` | `BsMatchMakingModel.go` (`utils.CallKafkaService`) | [`USER_BS_MATCHMAKING_KAFKA.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_KAFKA.go) (`InitializeKafka`-driven) | Mesh-DB replica write trigger karta hai |

**Important distinction from GST/Rating/Privacy-Setting domains**: yeh feature **primarily
Kafka pe hai, RabbitMQ pe nahi** — GST/Rating/Privacy-Setting sab mostly RabbitMQ-driven the
(Kafka sirf 1 exception tha unme). Matchmaking domain isse ulat pattern follow karta hai.
`CallKafkaService` internally ek HTTP proxy call hai (Kafka REST proxy pattern), native
socket-level Kafka connection nahi — same jaisa GST domain mein tha.

---

## 6. RabbitMQ

**Iss specific write-flow (`BsMatchMakingController`) mein koi RabbitMQ usage nahi mila.**
Sirf Kafka. (LMS-sync consumer `USER_BS_MATCHMAKING_LMS` ka exact messaging-mechanism iss
pass mein re-verify nahi hua.)

---

## 7. Redis

**Koi Redis usage nahi mila** iss module mein.

---

## 8. End-to-End Technical Flow

```
Contact-event (enquiry/order/conversation — outside this module's scope)
    │
    ▼
[API — write]  POST /user/bsmatchmaking
    │  BsMatchMakingController.go
    │  1. Mandatory-field check (buyer_id/supplier_id/VALIDATION_KEY)
    │  2. Gateway check ("LEADS" allowlist only)
    │  3. Length/type validation (BSMatchMakingMap)
    ▼
[DB — synchronous]  sp_insert_contacts() on buyerprofilepg
    │  (primary write happens HERE, inside the request)
    ▼
[Kafka publish]  topic: soa-user-bs_matchmaking-csl_glid_logs
    │  (fire-and-forget — API already returns "INSERT SUCCESS" regardless)
    ▼
──────────────────────────────  <- API response sent to caller by here
[Kafka consume, consumer repo]
    │  USER_BS_MATCHMAKING_KAFKA.go
    │  re-validates message (defensive)
    ▼
[DB — again, different DB]  sp_insert_contacts() on meshPg (with retry logic)
    │  — same stored proc, mirror copy for the mesh-side systems
    ▼
Later consumed by:
    ├─ USER_RATING_BS_MATCHMAKING (Rating anti-fraud check)
    └─ USER_BS_MATCHMAKING_LMS (Lead-Management sync)
```

---

## 9. Optimization Scope — DB Response-Time Contribution

Yeh ek chhota, focused domain hai — sirf ek table, ek write-path. DB response-time contribution
seedha aur predictable hai yahan.

### High-impact

1. **Primary DB write ek stored-procedure call hai, single round-trip** — already efficient
   hai, koi obvious multi-query overhead nahi jaisa GST/Rating domains mein tha.
2. **Retry-on-deadlock logic consumer mein hai** (§4, point 7), lekin **exponential backoff
   ki upper-bound 10 minute tak hai** — agar deadlocks frequent hon, ek single message
   consumer ko lambe samay tak block kar sakta hai (agar concurrency-per-message low ho).
   Worth monitoring: kitni baar yeh retry-path actually trigger hoti hai production mein.

### Medium-impact

3. **`GLUSR_USR_ID1`/`GLUSR_USR_ID2` length validation `> 10` characters pe hardcoded hai** —
   agar GLID format kabhi badle (jaise bade IDs), yeh silently valid records reject karna
   shuru kar dega. Low-probability lekin worth ek comment/constant-based check banane ka.

### Low-impact / good practice already present

4. **Consumer defensive-validation** (§4, point 5) — achi practice hai, GST/Rating domains
   mein bhi yehi pattern dekha gaya.
5. **`SOURCE_ID` 3/4 ke liye duplicate-write-skip** (§4, point 6) — smart optimization,
   unnecessary DB writes avoid karta hai jab primary write already kisi aur path se ho chuka
   ho.

---

## 10. Full Flow Diagram (Lucid, icon-based)

**[Poora Matchmaking flowchart yahan dekho](https://lucid.app/lucidchart/658a4d17-bf95-447f-84ce-02f08ea86cf6/edit)**

---

## 11. Edge Cases & Gotchas (technical POV)

1. **`insertion_type`/`contact_type` ka exact business-meaning trace nahi hua** (§4, point
   3) — sirf default value (`"2"`) confirmed hui, semantics nahi.
2. **`USER_BS_MATCHMAKING_LMS` consumer ka poora code iss pass mein re-read nahi hua** —
   file registered hai router mein, lekin deep-dive Rating/GST domains jitni nahi hui.
3. **Dual-database write ka failure-mismatch scenario**: agar `buyerprofilepg` write
   succeed kare lekin Kafka publish fail ho jaaye (ya consumer crash ho jaaye retry-window
   khatam hone se pehle), `meshPg` wali copy kabhi ban hi nahi sakti — matlab
   `buyerprofilepg` aur `meshPg` ke beech ek silent drift ban sakta hai. Koi reconciliation
   job iss review mein nahi mila (GST ke `gst_tact_veri_cron`-jaisa safety-net yahan nahi
   dikha).

---

## 12. Open Questions

1. `insertion_type`/`contact_type` field ka exact business-meaning kya hai (1 vs 2 vs
   others)?
2. `USER_BS_MATCHMAKING_LMS` consumer ka full technical trace abhi pending hai.
3. Kya `buyerprofilepg`↔`meshPg` drift ke liye koi reconciliation/backfill process hai
   (jaisa GST domain mein tha)? Iss pass mein nahi mila.
4. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`Matchmaking_Business_Doc.md`](./Matchmaking_Business_Doc.md) — product perspective
- [`../Blocking KT/Blocking_Technical_Doc.md`](../Blocking%20KT/Blocking_Technical_Doc.md) —
  related independent module, same read-query intersection point (`UserBuyerProfileModel.go`)
- [`../Rating KT/Rating_Technical_Doc.md`](../Rating%20KT/Rating_Technical_Doc.md) — where
  `USER_RATING_BS_MATCHMAKING` consumes this data for anti-fraud checks
- [`../buyer_seller_discovery_matching_read_write_picture.md`](../buyer_seller_discovery_matching_read_write_picture.md) —
  original architecture trace this doc builds on

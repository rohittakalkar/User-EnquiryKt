# Rating Usefulness — Technical Doc (Code-Level Deep Dive)

**Scope note**: yeh doc sirf **Rating Usefulness** ("helpful?" vote) cover karta hai. Yeh
**Rating** (1-5 star) se alag hai — dekho
[`../Rating KT/Rating_Technical_Doc.md`](../Rating%20KT/Rating_Technical_Doc.md).

Business/product perspective ke liye
[`Rating_Usefulness_Business_Doc.md`](./Rating_Usefulness_Business_Doc.md) dekho.

**Repos**: `service-api-go-production` (write), `user-temp-consumers-production`
(consumer).

**Methodology**: har claim source code se trace kiya gaya hai (file path diya gaya hai).

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Vote submit | write | [`UserRatingUsefulnessController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserRatingUsefulnessController.go), [`UserRatingUsefulnessModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserRatingUsefulnessModel.go) (function `RatingUsefulInsert`) |
| Count sync back to the rating record | consumers | [`USER_RATING_USEFULNESS.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_USEFULNESS.go) |
| Rating record jispe count update hota hai | write | [`UserSupplierRatingController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSupplierRatingController.go) — **isi controller ko internally reuse kiya jaata hai**, dekho §4 |

Koi dedicated read-side controller nahi mila iss feature ke liye — helpful/abuse counts
`GET /supplierrating` response ke andar hi aate hain (`HELPFUL_COUNT`/`ABUSE_COUNT` fields
jo `UserSupplierRatingController` handle karta hai), separate GET endpoint nahi hai.

---

## 2. Routes

| Method | Path | Repo | Controller |
|---|---|---|---|
| POST | `/rating_usefulness` | write | `UserRatingUsefulnessController` |

---

## 3. Data Model — Tables

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_RATING_USEFULNESS` | meshpg | **Primary vote record** — ek row per (buyer, rating) pair | `FK_GLUSR_RATING_ID`, `FK_GLUSR_USR_ID`, `IS_RATING_USEFUL` (`1`=helpful, `-1`=abuse), `FK_IIL_SCREEN_ID` (source_id), `ENTRY_DATE` — [`UserRatingUsefulnessModel.go:38`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserRatingUsefulnessModel.go) |
| `GLUSR_RATING` | meshpg | **Denormalized counts yahan bhi store hote hain** — `HELPFUL_COUNT`/`ABUSE_COUNT` columns is table ke `UPDATE` se update hote hain (via `UserSupplierRatingController`'s update path) | Dekho [`../Rating KT/Rating_Technical_Doc.md`](../Rating%20KT/Rating_Technical_Doc.md) §3 for full `GLUSR_RATING` schema |

---

## 4. Business Rules & Validation (code se)

1. **`IS_HELPFUL` sirf `"1"` ya `"-1"` accept hota hai** — koi aur value:
   `"IS_HELPFUL can be either 1 or -1"`.
   [`UserRatingUsefulnessController.go:92-104`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserRatingUsefulnessController.go)
2. **Gateway validation ek alag allowed-caller list use karta hai** (`RatingUsefulHash`) —
   yeh list `RATING_INFLU_PARAMS`-jaisi hi hai jo `UserRatingUsefulnessController.go` mein
   top-level defined hai, structurally Rating-submit ke `allowed_modids` jaisi hi (dono
   controllers mein alag-alag defined, duplicate list maintenance — dekho §9).
3. **Duplicate-vote handling database-constraint-level pe hai, application-logic pe nahi**:
   INSERT fail hone par (`strings.Contains(output, "execution failed")`), code assume karta
   hai yeh ek duplicate-key violation hai aur bas `"Record already exists"` return karta hai
   — **specific error-type check nahi hai**, koi bhi INSERT-failure isi tarah handle hoti
   hai. [`UserRatingUsefulnessModel.go:91-93,122-124`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserRatingUsefulnessModel.go)
4. **Count-sync sirf successful naye insert pe trigger hoti hai** — agar record already
   exists (duplicate vote), RabbitMQ push hi nahi hota, matlab koi unnecessary downstream
   call nahi jaati duplicate votes ke liye.

---

## 5. RabbitMQ — Queue Used

| Queue / `SERVICENAME` | Publisher | Consumer | Purpose |
|---|---|---|---|
| `USER_RATING_USEFULNESS` | `UserRatingUsefulnessModel.go` (`RatingUsefulInsert`, sirf naye insert pe) | [`USER_RATING_USEFULNESS.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_USEFULNESS.go) | Naye helpful/abuse counts ko wapas `GLUSR_RATING` record mein sync karne ke liye trigger |

Sirf **ek** queue hai iss poore feature mein — GST/Rating (star) domains ke multi-queue
fan-out ke muqable, yeh ek bahut chhota, linear pipeline hai.

---

## 6. Kafka

**Koi Kafka usage nahi mila.** `USER_RATING_USEFULNESS.go` `InitializeRabbitMq` use karta
hai.

---

## 7. Redis

**Koi Redis usage nahi mila.** Helpful/abuse counts har baar live query se aate hain (jab
bhi `GET /supplierrating` call hoti hai, jo `GLUSR_RATING` table se directly padhta hai).

---

## 8. End-to-End Technical Flow — Sabse Interesting Architecture Pattern

```
Buyer (rating padh raha, "helpful" ya "abuse" click karta hai)
    │
    ▼
[API — write]  POST /rating_usefulness
    │  UserRatingUsefulnessController.go
    │  1. Mandatory fields: GLUSR_ID/RATING_ID/SOURCE_ID/VALIDATION_KEY/IS_HELPFUL/IP/IP_COUNTRY
    │  2. IS_HELPFUL must be "1" or "-1"
    │  3. Gateway validation (RatingUsefulHash allowlist)
    ▼
[DB]  INSERT INTO GLUSR_RATING_USEFULNESS (rating_id, buyer_id, is_helpful, source_id)
    │
    ├─ FAIL (likely duplicate-key) → "Record already exists", stop here, no queue push
    │
    └─ SUCCESS →
          [DB]  SELECT COUNT(helpful), COUNT(abuse) FROM GLUSR_RATING_USEFULNESS
                WHERE FK_GLUSR_RATING_ID = $1
          │
          ▼
          [RabbitMQ publish]  SERVICENAME=USER_RATING_USEFULNESS
          │  carries: RATING_ID, HELPFUL_COUNT, ABUSE_COUNT, IP, IP_COUNTRY
          ▼
          [CONSUME]  USER_RATING_USEFULNESS.go
          │  — YEH CONSUMER DB MEIN SEEDHA WRITE NAHI KARTA
          │  — iski jagah, yeh WAPI (write API) ko wapas ek HTTP call karta hai
          │    (utils.CallWapiService, config key "rating_write_service")
          ▼
          [HTTP POST — internal loopback call]  service-api ke apne /supplierrating endpoint ko
          │  payload: {RATING_ID, HELPFUL_COUNT, ABUSE_COUNT, CALLEDFROM: "RATING_USEFULNESS_WRITE",
          │            UPDATE_FLAG: "U", VALIDATION_KEY: <hardcoded>, UPDATEDBY: "Internal Process (WAPI)"}
          ▼
          [Re-enters]  UserSupplierRatingController.go (Rating domain ka controller!)
          │  CALLEDFROM=="RATING_USEFULNESS_WRITE" branch validate karta hai
          │  (RATING_ID/UPDATEDBY/VALIDATION_KEY/HELPFUL_COUNT/ABUSE_COUNT/IP/IP_COUNTRY mandatory)
          ▼
          UserModels.UpdateRating() → UPDATE GLUSR_RATING SET HELPFUL_COUNT=..., ABUSE_COUNT=...
```

**Architecture observation**: yeh ek genuinely unusual pattern hai — normal consumers seedha
DB likhte hain, lekin `USER_RATING_USEFULNESS.go` **khud ke write-API ko ek internal HTTP
call** karta hai, database ko directly touch kiye bina. Yeh design decision ka fayda: rating
record ka **saara write-logic (validation, history, downstream triggers) ek hi jagah
centralized rehta hai** (`UserSupplierRatingController`/`UpdateRating`), duplicate na ho.
Nuksan: ek extra network-hop (HTTP call) DB-write ke bajaye, aur agar `UserSupplierRatingController`
kabhi apna validation-rules change kare, yeh silently is feature ko bhi affect kar sakta hai.

---

## 9. Optimization Scope — DB Response-Time Contribution

Yeh sabse chhota/simple domain hai teeno KT folders mein (GST, Privacy Setting, Rating ke
comparison mein) — DB response-time yahan utna bada factor nahi hai jitna network-hop
architecture.

### High-impact

1. **Consumer → HTTP-loopback-call → dobara controller-validation → phir DB write** — yeh
   ek **naya network round-trip** introduce karta hai jo seedha DB write se avoid ho sakta
   tha. Agar `rating_write_service` (jo `/supplierrating` ko point karta hai) kabhi slow ho
   ya down ho, `USER_RATING_USEFULNESS` consumer bhi backlog ban jaayega — **DB se zyada
   yeh internal-HTTP-dependency response-time ka risk hai iss specific feature mein**
   (GST ke external-BI-API risk jaisa hi pattern, bas internal system ke saath).
2. **Duplicate-vote check ek failed-INSERT pe depend karta hai** (§4, point 3) — matlab
   database ko ek failed write-attempt (with rollback overhead) bhardna padta hai har baar
   koi dobara vote kare, instead of pehle ek cheap `SELECT EXISTS` query se check karke.
   **Concrete fix**: pehle `SELECT EXISTS(...)` karo, agar already exists toh INSERT try hi
   mat karo — yeh failed-transaction overhead avoid karega.

### Medium-impact

3. **Count-recompute (`SELECT COUNT... GROUP conditionally`) full-table-scan-per-rating
   pattern hai**, chhota scale pe fine hai (per-rating votes typically kam hain), lekin agar
   koi rating bahut popular ho jaaye (bahut saare votes), yeh query thodi costlier ho sakti
   hai — abhi ke liye low-priority, monitor karne layak.

### Low-impact / good practice already present

4. **Count-sync sirf successful-new-vote pe trigger hoti hai** (§4, point 4) — duplicate
   votes ke liye koi unnecessary RabbitMQ push ya HTTP call nahi jaati, yeh achi filtering
   hai already in place.

---

## 10. Full Flow Diagram (Lucid, icon-based)

**[Poora Rating Usefulness flowchart yahan dekho](https://lucid.app/lucidchart/8915bd59-ff23-4121-9c94-daf73b381c9e/edit)**

*(Yeh ek chhota, single-flow domain hai — GST/Rating jaisa multi-page superset zaroori nahi
laga; ek hi page pe poora loop dikhaya gaya hai, including the unusual HTTP-loopback step.)*

---

## 11. Edge Cases & Gotchas (technical POV)

1. **HTTP-loopback dependency** (§8, §9 point 1) — agar `rating_write_service` config-key
   kabhi galat URL point kare ya down ho, helpful/abuse counts silently stale reh jaayenge
   (vote table mein record ban jaayega, lekin `GLUSR_RATING.HELPFUL_COUNT` update nahi
   hoga) — ek subtle data-inconsistency risk.
2. **Self-vote block missing** (Business Doc §5.3) — code mein koi explicit check nahi mila
   jo original-rating-dene-wale buyer ko apni khud ki rating vote karne se roke.
3. **Generic failed-INSERT-as-duplicate assumption** (§4, point 3) — agar INSERT kisi aur
   reason se fail ho (jaise connection timeout), user ko galat "Record already exists"
   message milega, actual error message nahi.

---

## 12. Open Questions

1. `RatingUsefulHash` aur Rating-submit ke `allowed_modids` — yeh do alag lists hain ya
   inhe ek shared config se maintain karna chahiye (duplication risk)?
2. Self-vote (apni khud ki rating ko helpful/abuse mark karna) explicitly blocked hai kahin
   aur, ya genuinely allowed hai?
3. `rating_write_service` config-key kahan define hai, aur uske liye koi retry/circuit-breaker
   hai agar `/supplierrating` call fail ho?
4. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`Rating_Usefulness_Business_Doc.md`](./Rating_Usefulness_Business_Doc.md) — product
  perspective
- [`../Rating KT/Rating_Technical_Doc.md`](../Rating%20KT/Rating_Technical_Doc.md) — Rating
  (star-rating) domain, jispe yeh feature depend karta hai (`UpdateRating`,
  `CALLEDFROM=="RATING_USEFULNESS_WRITE"` branch)
- [`../ratings_reviews_product_overview.md`](../ratings_reviews_product_overview.md) —
  bigger Ratings & Reviews story

# TrustSeal (Seal, Company, Misc & Trade Affiliations) — Technical Doc

Business/product perspective ke liye
[`TrustSeal_Business_Doc.md`](./TrustSeal_Business_Doc.md) dekho.

**Repos**: `service-api-go-production` (write), `users-api-go-production` (read).

**Scope decision (confirmed via code investigation)**: TrustSeal, TrustSeal Company, TS Misc
Details, TS Comp Details — sab ek hi table-family hai (`FK_TRUSTSEAL_ID` se joined), ek hi
write-controller (`UserTrustsealController`) se manage hoti hai. Isliye **ek hi folder** mein
rakha gaya — alag folders banana same write-flow ko 4 baar duplicate karta.
**Trust Verification** (generic attribute-verification) genuinely alag system hai, dekho
[`../Trust Verification KT/`](../Trust%20Verification%20KT/Trust_Verification_Technical_Doc.md).

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| TrustSeal write (single controller, TYPE-dispatched) | write | [`UserTrustsealController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTrustsealController.go), [`UserTrustsealModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTrustsealModel.go) |
| TrustSeal read (basic) | read | [`TrustSealController.go`](../internal/controllers/UsersControllers/TrustSealController.go) (`ActionTrustSealController`), [`UserTrustSealModel.go`](../internal/models/users/UserTrustSealModel.go) |
| TrustSeal detail read | read | [`TrustsealDetailController.go`](../internal/controllers/UsersControllers/TrustsealDetailController.go) (`TrustSealDetail`), [`UserTrustsealDetailModel.go`](../internal/models/users/UserTrustsealDetailModel.go) |
| TrustSeal Company details read | read | [`TSCompDetailsController.go`](../internal/controllers/UsersControllers/TSCompDetailsController.go), [`UserTSCompDetailsModel.go`](../internal/models/users/UserTSCompDetailsModel.go) |
| TrustSeal Misc details (Trade Affiliations/Safety Cert/Chamber) read | read | [`TSMiscDetailsController.go`](../internal/controllers/UsersControllers/TSMiscDetailsController.go) (`ActionTsDetails` — note: function name doesn't match file name, verify), [`UserTSMiscDetailsModel.go`](../internal/models/users/UserTSMiscDetailsModel.go) |

---

## 2. Routes (confirmed)

| Method | Path | Repo | Controller |
|---|---|---|---|
| GET/POST | `/trustseal/*params` | read | `ActionTrustSealController` |
| GET/POST | `/trustsealdetail/*params` | read | `TrustSealDetail` |
| GET/POST | `/tsdetails/*params` | read | `ActionTsDetails` |
| GET/POST | `/tscompdetails/*params` | read | `TSCompDetails` |
| GET/POST | `/tsmiscdetails/*params` | read | `TSMiscDetails` |
| POST | `/user/trustseal` | write | `UserTrustsealController` (sirf `WEBERP` gateway) |

---

## 3. Data Model — Tables

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `TRUSTSEAL` | meshpg | **Master record** — ek row per assigned seal | `TRUSTSEAL_ID`, `TRUSTSEAL_CODE`, `FK_COMPANYID`, `TRUSTSEAL_GLUSR_USR_ID`, `TRUSTSEAL_CREATIONDATE`, `TRUSTSEAL_STARTDATE`, `TRUSTSEAL_ENDDATE`, `TRUSTSEAL_APPROV` (status flag — `'M'`/`'A'` values seen, exact meaning not decoded), `TRUSTSEAL_MODIFICATIONDATE` — [`UserTsDetailsModel.go:41-52`](../internal/models/users/UserTsDetailsModel.go), [`UserTrustsealModel.go:86,94`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTrustsealModel.go) |
| `TRUSTSEAL_COMPANY` | meshpg | Company(s) linked to a TrustSeal — supports a "primary" company flag | `TRUSTSEAL_COMPANY_ID`, `FK_TRUSTSEAL_ID`, `FK_COMPANY_ID`, `TRUSTSEAL_COMPANY_ISPRIMARY`, `FK_CUST_TO_SERV_ID` (links to a customer-to-service/subscription record), `TRUSTSEAL_COMPANY_GLUSR_USR_ID`, `TRUSTSEAL_COMPANY_TSCODE` — [`UserTSCompDetailsModel.go:83-108`](../internal/models/users/UserTSCompDetailsModel.go) |
| `TRUSTSEAL_HISTORY` | meshpg | Audit-trail — free-text history, **prepended** (not appended) to existing text on every update | `FK_TRUSTSEAL_ID`, `TRUSTSEAL_HISTORY_TEXT` — [`UserTrustsealModel.go:109,125,145,245,266`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTrustsealModel.go) |
| `TS_TRADE_AFFILIATIONS` | meshpg | Trade-affiliation memberships linked to a TrustSeal | `TS_TRADE_AFFILIATIONS_ID`, `FK_TRUSTSEAL_ID`, `FK_TRADE_AFFILIATIONS_ID`, `TS_TRADE_AFFILIATE_VALIDUPTO`, `TS_TRADE_AFFILIATE_IMGPATH` — [`UserTSMiscDetailsModel.go:119-129`](../internal/models/users/UserTSMiscDetailsModel.go) |
| `TRADE_AFFILIATIONS`, `SAFETYCERT`, `CHAMBER` | meshpg | Master/lookup tables for the misc-detail types | Referenced by name-lookup (`WHERE UPPER(...)=...`) — [`UserTSMiscDetailsModel.go:107,112,119`](../internal/models/users/UserTSMiscDetailsModel.go) |

---

## 4. Business Rules & Validation (code se)

1. **Sirf `WEBERP` gateway allowed hai** — sabse tight-gated write-path in this whole
   domain-set (GST allows GLADMIN, Privacy-Setting allows multiple app-surfaces, TrustSeal
   sirf ek internal system).
   [`UserTrustsealController.go:43`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTrustsealController.go)
2. **`TYPE` param request ko dispatch karta hai** — same controller multiple sub-entities
   handle karta hai (seal khud, company-link, misc). Exact TYPE-values (`TS`, etc.) code
   comments se partially confirm hue (`"TYPE(TS) and ACTION(...)"` error string), poori list
   decode nahi hui.
   [`UserTrustsealController.go:46`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTrustsealController.go), [`UserTrustsealModel.go:185`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTrustsealModel.go)
3. **History-text prepend pattern, not append**: `TRUSTSEAL_HISTORY_TEXT=COALESCE($1,'') ||
   COALESCE(TRUSTSEAL_HISTORY_TEXT,'')` — naya text purane text ke **pehle** jud़ta hai,
   matlab latest-change hamesha text ke shuru mein milega, reverse-chronological.
   [`UserTrustsealModel.go:109,245`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTrustsealModel.go)
4. **Company-level end-date update ek subquery se apna `TRUSTSEAL_ID` khud resolve karta
   hai**, `WHERE TRUSTSEAL_APPROV IN ('M','A')` filter ke saath — matlab sirf specific
   approval-states waale seals hi is path se update ho sakte hain.
   [`UserTrustsealModel.go:119`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTrustsealModel.go)
5. **History-insert-vs-update dono support karta hai** — agar history-row already exist
   nahi karti (`UPDATE` 0 rows affect kare), phir `INSERT INTO TRUSTSEAL_HISTORY` fallback
   hota hai.
   [`UserTrustsealModel.go:245-266`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTrustsealModel.go)
6. **Misc-details (Trade Affiliations) do read-modes support karte hain**: linked-affiliations
   (already jud़i hui, `FK_TRUSTSEAL_ID` se) vs available-affiliations (jo abhi tak link
   nahi hui, `NOT IN` subquery se), — ek "add new affiliation" UI ke liye typical dropdown
   pattern.
   [`UserTSMiscDetailsModel.go:126,129`](../internal/models/users/UserTSMiscDetailsModel.go)

---

## 5. RabbitMQ / Kafka / Redis

**Iss pass mein koi RabbitMQ/Kafka/Redis usage nahi mila** TrustSeal write-path mein — sab
kuch synchronous DB writes hain, ek hi request ke andar. Yeh Blocking-module jaisa hi simple,
purely-synchronous architecture hai (dekho [`../Blocking KT/`](../Blocking%20KT/Blocking_Technical_Doc.md)).

---

## 6. End-to-End Technical Flow

```
WEBERP (internal system, sirf isse allowed)
    │
    ▼
[API — write]  POST /user/trustseal  {TYPE, ACTION, ...}
    │  UserTrustsealController.go
    │  Gateway check: WEBERP only
    ▼
Dispatch on TYPE
    │
    ├─ TrustSeal itself → INSERT/UPDATE TRUSTSEAL (approval-status, end-date, etc.)
    ├─ Company-link     → UPDATE TRUSTSEAL_COMPANY (FK_CUST_TO_SERV_ID linkage)
    └─ (misc/trade-affiliation writes — not fully traced in this pass)
    │
    ▼
[DB]  UPDATE/INSERT TRUSTSEAL_HISTORY (prepend audit-text)
    ▼
Response
```

```
Buyer / App (reads)
    │
    ├─ GET /trustseal          → TRUSTSEAL master fields
    ├─ GET /trustsealdetail    → extended detail view
    ├─ GET /tscompdetails      → TRUSTSEAL_COMPANY (linked companies, primary flag)
    └─ GET /tsmiscdetails      → TS_TRADE_AFFILIATIONS + SAFETYCERT/CHAMBER lookups
```

---

## 7. Optimization Scope — DB Response-Time Contribution

### Medium-impact

1. **History-write pattern (prepend, `COALESCE(...) || COALESCE(...)`) grows unbounded** —
   ek TrustSeal jitna purana/frequently-updated hoga, `TRUSTSEAL_HISTORY_TEXT` utna bada
   single text-blob banta jaayega. Har naya update **poora purana text bhi dobara likhta
   hai** (read-modify-write ek hi column pe) — agar history bahut lambi ho jaaye, yeh
   write-cost badhata jaayega. **Concrete fix**: ek proper append-only history-table
   (multiple rows, ek row per change) row-growth ki jagah column-growth se better scale
   karega.
2. **4 alag read-controllers, koi combined/single response nahi** — agar ek UI ko seal +
   company + misc-details teeno chahiye ek page pe, woh 3-4 separate API calls karega. Ek
   combined "TrustSeal full profile" endpoint response-time (network round-trips) kam kar
   sakta hai, agar yeh actual usage-pattern hai frontend mein.

### Low-impact

3. **Sabhi writes synchronous, single-DB hain** — koi multi-DB fan-out ya async complexity
   nahi (GST/Rating domains ke ulat) — yeh already simple/low-risk hai.

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora TrustSeal flowchart yahan dekho](https://lucid.app/lucidchart/75a9d573-926c-454d-8f78-d61c7ac5ee94/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **`TRUSTSEAL_APPROV` values (`'M'`, `'A'`) ka exact meaning decode nahi hua** — likely
   "Manual"/"Auto" ya "Marked"/"Approved" — confirm karo master-table/enum se.
2. **`TYPE` param ki poori list decode nahi hui** — sirf `"TS"` ek confirmed value hai error
   messages se; TrustSeal Company aur Misc-details ke write-paths ke exact TYPE-values trace
   nahi hue.
3. **History-table unbounded-growth risk** (§7, point 1).
4. **`TSMiscDetailsController` ka function name (`ActionTsDetails`) file-naam se match nahi
   karta** — thoda confusing navigate karne mein, worth flag karna.

---

## 10. Open Questions

1. `TRUSTSEAL_APPROV` ke exact status-values aur unka business-meaning?
2. TrustSeal ↔ GST-verification ke beech koi automatic trigger hai (jaise "GST verify hone
   pe TrustSeal-eligibility flag set ho"), ya poori tarah manual/WEBERP-driven hai?
3. `FK_CUST_TO_SERV_ID` (customer-to-service link) — kaunsi paid-service iske through
   TrustSeal ko link karti hai?
4. Company-level aur Misc-level (Trade Affiliation) writes ka poora `TYPE`-dispatch trace
   iss pass mein complete nahi hua.
5. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`TrustSeal_Business_Doc.md`](./TrustSeal_Business_Doc.md) — product perspective
- [`../Trust Verification KT/Trust_Verification_Technical_Doc.md`](../Trust%20Verification%20KT/Trust_Verification_Technical_Doc.md) —
  related, separate generic attribute-verification system
- [`../trust_verification_compliance_read_write_picture.md`](../trust_verification_compliance_read_write_picture.md) —
  bigger Trust & Compliance technical trace

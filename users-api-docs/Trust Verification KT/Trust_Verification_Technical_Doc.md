# Trust Details & Verification — Technical Doc

Business/product perspective ke liye
[`Trust_Verification_Business_Doc.md`](./Trust_Verification_Business_Doc.md) dekho.

**Repos**: `service-api-go-production` (write), `users-api-go-production` (read),
`user-temp-consumers-production` (loopback caller).

**Scope decision**: yeh generic per-attribute verification system hai (GST, bank, mobile,
email — koi bhi attribute). [`../TrustSeal KT/`](../TrustSeal%20KT/TrustSeal_Technical_Doc.md)
se **alag** rakha gaya kyunki koi direct DB-level link (`TRUSTSEAL` table se) nahi mila —
sirf conceptual/business-level relation hai.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Verification write (single attribute) | write | [`UserVerificationController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserVerificationController.go) *(file existence confirmed via route, controller body not re-read in this pass)* |
| Verification details write | write | `UserVerificationDetailsController.go`, [`UserVerificationDetailsModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go) |
| Verification read (attribute-level) | read | [`GetVerificationDetailModel.go`](../internal/models/users/GetVerificationDetailModel.go) — stored functions `fn_get_glusr_attr_verify`, `fn_get_glusr_attributes` |
| Verification read (verified-only view) | read | `VerifiedDetailController.go` *(route confirmed, body not re-read in this pass)* |
| Loopback verification helper (consumer-side) | consumers | [`USER_APPROVAL_ATTRIBUTE.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_APPROVAL_ATTRIBUTE.go) (`VerifyAttr` function) |
| Bulk verification consumer | consumers | `USER_BULK_VERIFICATION.go` (registered as `UserBulkVerification` — already referenced in [`GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) §3 for `USER_VERIFICATION_BULK_HISTORY`) |
| Trust-DB sync consumers | consumers | `USER_VERIFICATION_TRUSTPG.go` / `..._FAIL` (already referenced in GST Technical Doc §5, RabbitMQ table) |

---

## 2. Routes (confirmed)

| Method | Path | Repo | Controller |
|---|---|---|---|
| POST | `/user/verification` | write | `UserVerificationController` |
| POST | `/user/userverificationdetails` | write | `UserVerificationDetailsController` |
| GET/POST | `/getverificationdetails/*params` | read | `GetVerificationDetails` |
| GET/POST | `/verifieddetail/*params` | read | `VerifiedDetailController` |

---

## 3. Data Model — Tables/Functions

| Table/Function | Physical DB | Purpose | Confirmed from |
|---|---|---|---|
| `fn_get_glusr_attr_verify(glid)` | mesh_pg_user (stored function) | Ek supplier ke saare verified-attributes ka status return karta hai | [`GetVerificationDetailModel.go:23`](../internal/models/users/GetVerificationDetailModel.go) |
| `fn_get_glusr_attributes(glid)` | mesh_pg_user (stored function) | Ek supplier ke saare attributes (verified ya nahi, dono) return karta hai | [`GetVerificationDetailModel.go:25`](../internal/models/users/GetVerificationDetailModel.go) |
| `iil_verification_details` | BigQuery + likely Postgres mirror | Verification-tracking table — **already documented in [`GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) §3**, GST TACT cron isse join karta hai. Confirms yeh table domain-wide hai, GST-specific nahi. | Cross-referenced from GST KT research |
| Attribute-ID reference system | — | Har attribute-type ka apna numeric ID — confirmed examples: `2106` = GST, `390` = Bank account (`PRIMARY_ATTR_ID`/`ATTRIBUTE_ID` fields seen across GST aur yeh dono domains) | [`GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md), [`USER_APPROVAL_ATTRIBUTE.go:538`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_APPROVAL_ATTRIBUTE.go) |

**Important honest note**: is pass mein `UserVerificationController.go` aur
`UserVerificationDetailsModel.go` ke andar ka actual `INSERT`/`UPDATE` SQL fully trace nahi
hua — sirf loopback-caller (`VerifyAttr`) aur read-side stored-functions confirm hue. Iska
matlab yeh doc **read-path aur cross-domain-integration** pe zyada confident hai, write-side
ke exact table/columns pe utna nahi — dekho Open Questions.

---

## 4. The HTTP-Loopback Pattern (jaisa Rating Usefulness mein tha)

`VerifyAttr()` function — jo GST consumer (`USER_GST_DETAILS.go`) aur doosre consumers use
karte hain kisi attribute ko verify karne ke liye — **seedha DB nahi likhta**. Iski jagah:

```go
// user-temp-consumers-production/internal/Workers/USER_APPROVAL_ATTRIBUTE.go:535-560
func VerifyAttr(glid string, attributeID string, attributeval string,
                 updatedbyScreen string, refId string, reason interface{}) (string, error) {
    inputData := map[string]interface{}{
        "GLUSR_USR_ID":       glid,
        "ATTRIBUTE_ID":       attributeID,
        "ATTRIBUTE_VALUE":    attributeval,
        "VERIFIED_BY_NAME":   "WAPI",
        "action_flag":        "SP_VERIFY_ATTRIBUTE",   // suggests a stored proc "sp_verify_attribute"
        ...
    }
    status, _, err := utils.CallWapiService(inputData, "user_verification", utils.GetConsumerName(), "application/json")
    return status, err
}
```

**Yeh `POST /user/verification` (write-API) ko ek internal HTTP call karta hai** — bilkul
wahi pattern jo [`Rating_Usefulness_Technical_Doc.md`](../Rating%20Usefulness%20KT/Rating_Usefulness_Technical_Doc.md)
§8 mein documented tha. Matlab **poore domain ka single source of truth for "verify this
attribute" logic `UserVerificationController` hai** — chahe request seedha API se aaye ya
kisi consumer ke through loopback se.

`action_flag: "SP_VERIFY_ATTRIBUTE"` naam se strongly suggest karta hai ki underlying write
ek Postgres stored-procedure (`sp_verify_attribute`) hai, direct table-write nahi — GST domain
ke `sp_insert_contacts` (Matchmaking) jaisa hi pattern.

---

## 5. Business Rules & Validation (confirmed)

1. **Har attribute-verification request `ATTRIBUTE_ID` + `ATTRIBUTE_VALUE` pair leti hai** —
   generic, kisi bhi attribute-type ke liye reusable shape.
2. **`VERIFIED_BY_NAME`/`VERIFIED_BY_SCREEN`/`VERIFIED_BY_AGENCY` sab track hote hain** —
   kisne verify kiya (system vs agent), kis screen se, kis agency ke through (`"ONLINE"` seen
   as a value) — audit-friendly design.
   [`USER_APPROVAL_ATTRIBUTE.go:536-554`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_APPROVAL_ATTRIBUTE.go)
3. **Hardcoded `VALIDATION_KEY` loopback-calls mein use hoti hai** (`"e27d039e38ae7b3d439e8d1fe870fc68"`)
   — same hardcoded key jo GST domain ke internal loopback-calls mein bhi dekhi gayi thi
   (`USER_GST_DETAILS.go` mein similar pattern) — matlab yeh ek shared, internal-only
   validation-key hai jo consumer-to-API loopback calls ke liye reserved hai.
4. **Bulk-verification alag consumer hai** (`USER_VERIFICATION_BULK_HISTORY` →
   `UserBulkVerification`) — single-attribute verification se alag scaling-path, already
   GST KT doc mein documented.

---

## 6. RabbitMQ / Kafka / Redis

| System | Usage |
|---|---|
| **RabbitMQ** | `USER_VERIFICATION_TRUSTPG` / `_FAIL` queues — trust-DB replica-sync (already documented cross-domain in GST Technical Doc §5) |
| **RabbitMQ** | `USER_VERIFICATION_BULK_HISTORY` — bulk-verification intake queue |
| **Kafka** | Koi direct usage iss pass mein nahi mila |
| **Redis** | Koi usage nahi mila |

---

## 7. End-to-End Technical Flow

```
Trigger source (varies):
    ├─ Internal consumer (e.g. GST verification completing) → calls VerifyAttr() loopback
    ├─ Call-center agent tool → directly calls POST /user/verification
    └─ Bulk job → USER_VERIFICATION_BULK_HISTORY queue
    │
    ▼
[API — write]  POST /user/verification  {GLUSR_USR_ID, ATTRIBUTE_ID, ATTRIBUTE_VALUE, ...}
    │  UserVerificationController.go (body not fully re-traced in this pass)
    ▼
[DB — likely stored procedure]  sp_verify_attribute (inferred from action_flag naming)
    ▼
[RabbitMQ]  USER_VERIFICATION_TRUSTPG → trust-DB replica sync
```

```
Caller (any internal tool)
    │
    ▼
[API — read]  GET /getverificationdetails
    │  GetVerificationDetailModel.go
    ▼
[DB — stored function]  fn_get_glusr_attr_verify(glid)  or  fn_get_glusr_attributes(glid)
    ▼
Response: attribute-by-attribute verification status
```

---

## 8. Optimization Scope — DB Response-Time Contribution

### Medium-impact

1. **Read-side stored functions ka internal query-cost unknown hai** — `fn_get_glusr_attr_verify`
   aur `fn_get_glusr_attributes` dono ek hi glid ke liye similar-lagta data return karte
   hain (dono "attributes ki list" jaisa). Agar dono internally same base-tables scan karte
   hain, ek combined function (ya caching layer) response-time reduce kar sakta hai —
   confirm karna baaki hai ki yeh genuinely redundant hai ya alag purpose serve karte hain.
2. **HTTP-loopback pattern (§4) ek network-hop introduce karta hai** — Rating Usefulness
   domain mein jaisa establish kiya, yeh DB-latency se zyada **inter-service dependency**
   ka risk hai — agar `user_verification` service slow ho, saare loopback-callers (GST,
   doosre consumers) backlog ban sakte hain.

### Low-impact

3. **Koi caching nahi hai verification-reads pe** — GST/Rating domains jaisa hi gap, isi
   pattern se fix ho sakta hai agar verification-status frequently re-read hoti hai.

---

## 9. Full Flow Diagram (Lucid, icon-based)

**[Poora Trust Verification flowchart yahan dekho](https://lucid.app/lucidchart/980cca18-5709-4f5d-892c-f2565c6dd65a/edit)**

---

## 10. Edge Cases & Gotchas (technical POV)

1. **Write-side (`UserVerificationController`) ka poora code iss pass mein re-read nahi
   hua** — sirf loopback-caller (`VerifyAttr`) aur read-side confirm hue. Agar write-logic
   ka deep debugging chahiye, yeh file abhi bhi first-time-read hai.
2. **`fn_get_glusr_attr_verify` vs `fn_get_glusr_attributes` overlap-risk** (§8, point 1) —
   GST KT doc mein bhi aisa hi ek overlap-finding tha (`fn_sp_suspected_fraud_detection` vs
   `ML_FRAUD_SUSPECTED_USERS`) — is codebase mein "do similar-naam-wale functions/tables"
   ek recurring pattern lag raha hai, worth ek broader audit.
3. **Hardcoded shared `VALIDATION_KEY`** (§5, point 3) across multiple domains' loopback
   calls — agar yeh key kabhi rotate karni pade, multiple consumers simultaneously break ho
   sakte hain.

---

## 11. Open Questions

1. `UserVerificationController.go` aur `UserVerificationDetailsModel.go` ka poora
   write-path (exact tables/columns) — iss pass mein trace nahi hua.
2. `fn_get_glusr_attr_verify` vs `fn_get_glusr_attributes` — genuinely alag data return
   karte hain ya overlap?
3. `sp_verify_attribute` (inferred stored-proc name) — kya yeh sach mein exist karta hai,
   aur kya likhta hai?
4. Attribute-ID reference-list (2106=GST, 390=Bank, waghera) — koi master/enum table hai
   jahan yeh poori list documented hai?
5. Live DB schema verification — is doc ne sirf Go SQL strings/function-names jo imply
   karti hain wahi reflect kiya hai.

---

## See also

- [`Trust_Verification_Business_Doc.md`](./Trust_Verification_Business_Doc.md) — product
  perspective
- [`../TrustSeal KT/TrustSeal_Technical_Doc.md`](../TrustSeal%20KT/TrustSeal_Technical_Doc.md) —
  related, separate premium trust-badge system
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — GST verification as
  a concrete example of this generic system; `iil_verification_details` table cross-reference
- [`../Rating Usefulness KT/Rating_Usefulness_Technical_Doc.md`](../Rating%20Usefulness%20KT/Rating_Usefulness_Technical_Doc.md) —
  same HTTP-loopback architecture pattern (§4), useful comparison

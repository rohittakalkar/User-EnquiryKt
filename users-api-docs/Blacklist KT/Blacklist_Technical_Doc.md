# User Blacklist & Blacklist Values — Technical Doc (Code-Level Deep Dive)

**Scope note**: is doc mein teen alag data-sources cover ho rahe hain jo "blacklist" naam
share karte hain — dono docs (Business + Technical) mein inhe clearly §-wise separate rakha
gaya hai, confuse mat hona.

Business/product perspective ke liye [`Blacklist_Business_Doc.md`](./Blacklist_Business_Doc.md)
dekho.

**Repos**: `service-api-go-production` (write), `users-api-go-production` (read),
`user-temp-consumers-production` (consumer).

**Methodology**: har claim source code se trace kiya gaya hai (file path diya gaya hai).

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Manual blacklist-value update (write) | write | [`BlacklistValuesController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/BlacklistValuesController.go), [`BlacklistValuesModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/BlacklistValuesModel.go) |
| Manual blacklist-value read (by attribute) + fraud-detection (by glid) | read | [`BlacklistValuesController.go`](../internal/controllers/UsersControllers/BlacklistValuesController.go) (function `GetBlacklistvalues`), [`UserBlacklistValuesModel.go`](../internal/models/users/UserBlacklistValuesModel.go) |
| ML fraud-suspect read | read | [`ReadMLFraudController.go`](../internal/controllers/UsersControllers/ReadMLFraudController.go), [`UserReadMLFraudModel.go`](../internal/models/users/UserReadMLFraudModel.go) |
| Domain-level blacklist inline check | consumers | [`USER_BUSINESS_FEEDS.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go) (`CheckUserBlocked` function) |
| Chat-derived blacklist signal ingestion | consumers | [`USER_CHAT_BLACKLIST_DATA.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_CHAT_BLACKLIST_DATA.go) |
| **Dead/deprecated write API** | write | [`UserBlacklistController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserBlacklistController.go) — hardcoded `output = "This API is deprecated now"`, poori business-logic commented out |

---

## 2. Routes

| Method | Path | Repo | Controller | Status |
|---|---|---|---|---|
| POST | `/user/blacklistvalues` | write | `BlacklistValuesController` | **Live** |
| GET/POST | `/blacklistvalues/*params` | read | `GetBlacklistvalues` | **Live** |
| GET/POST | `/readmlfraud/*params` | read | `ReadMLFraudController` | **Live** |
| POST | *(unclear, route commented/legacy)* | write | `UserBlacklistController` | **Dead** — returns hardcoded deprecation message regardless of input |

---

## 3. Data Model — Tables (3 separate sources)

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GL_BLACKLIST_VALUES` | meshpg | **Manual blacklist** — admin-curated list of flagged values (mobile/email/etc.) | `GL_BLACKLIST_VALUE`, `GL_BLACKLIST_STATUS`, `GL_BLACKLIST_UPDATE_DATE`, `GL_BLACKLIST_UPDATED_BY`, `FK_ORIGNAL_GLUSR_ID`, `COMMENTS`, `FK_ATTRIBUTE_ID` — [`BlacklistValuesModel.go:19,28`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/BlacklistValuesModel.go) |
| `gl_attribute` | meshpg (mesh_pg_user, read) | Master table — attribute-type names (jaise "mobile", "email") jo `GL_BLACKLIST_VALUES.FK_ATTRIBUTE_ID` se join hoti hai | `GL_ATTRIBUTE_ID`, `GL_ATTRIBUTE_COLUMN_NAME` — [`UserBlacklistValuesModel.go:84`](../internal/models/users/UserBlacklistValuesModel.go) |
| `ML_FRAUD_SUSPECTED_USERS` | meshpg (mesh_pg_user, read) | **ML-model-driven** suspected-fraud accounts | `FK_GLUSR_USR_ID`, `SUSPECT_DATE`, `SUSPECT_COUNTER`, `SUSPECT_STATUS`, `SUSPECT_PROBABILITY`, `GLUSR_USR_CUSTTYPE_ID` — [`UserReadMLFraudModel.go:50`](../internal/models/users/UserReadMLFraudModel.go) |
| `ML_FRAUD_SUSPECT_EXCEPTIONS` | meshpg (mesh_pg_user, read) | Manual exceptions/exemptions ke liye — LEFT JOIN se `ML_FRAUD_SUSPECTED_USERS` ke saath | `FK_FRAUD_SUSPECT_EXCEPTION_ID`, `FRAUD_SUSPECT_EXCEPTION`, `EXCEPTION_DATE`, `EXCEPTION_ADDED_BY` — [`UserReadMLFraudModel.go:50`](../internal/models/users/UserReadMLFraudModel.go) |
| `QUERY_APPROVAL_BLACKLIST` | meshPg (consumer-side) | **Domain-level real-time check** — email/mobile domain-blacklist, inline-consulted by other features | `IS_DOMAIN`, `BLACKLIST_EMAIL` — [`USER_BUSINESS_FEEDS.go:318-329`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go) (already documented as part of `CheckUserBlocked` in [`utils_programming_guide.md`](../utils_programming_guide.md)-adjacent research) |
| `fn_sp_suspected_fraud_detection(glid)` | meshpg (mesh_pg_user), stored function | Ek stored-procedure-driven fraud check, called by the **glid-mode** of `GET /blacklistvalues` (not table-based, function returns JSON array) | Called via [`UserBlacklistValuesModel.go:47`](../internal/models/users/UserBlacklistValuesModel.go) — **overlap-risk with `ML_FRAUD_SUSPECTED_USERS`**, dekho §7 |

---

## 4. Business Rules & Validation (code se)

1. **`BlacklistValuesController` (write) sirf `GLADMIN`/`MAPI` callers accept karta hai** —
   sabse tight allowlist teeno KT-series domains ke muqable.
   [`BlacklistValuesController.go:104`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/BlacklistValuesController.go)
2. **Do alag mandatory-field paths hain**: normal update (`BLACKLIST_VALUE`/`UPDATEDBY`/
   `ORIGNAL_GLUSR_ID`/`COMMENTS` mandatory) vs whitelist-override (`WHITELISTBYGLID=="1"`
   ke saath — sirf `ORIGNAL_GLUSR_ID`/`COMMENTS`/`BLACKLIST_STATUS` mandatory).
   [`BlacklistValuesController.go:111-122`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/BlacklistValuesController.go)
3. **Query dynamically badalti hai based on `WHITELISTBYGLID`**: agar whitelist-override
   hai, `WHERE FK_ORIGNAL_GLUSR_ID=$3` se match hota hai (poore GLID ka record); warna
   `WHERE UPPER(TRIM(GL_BLACKLIST_VALUE))=$5` se (specific value match, case-insensitive,
   trimmed).
   [`BlacklistValuesModel.go:18-30`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/BlacklistValuesModel.go)
4. **`BLACKLIST_VALUE` uppercase-normalize hoti hai save hone se pehle**, lekin read-side
   lookup (`attribute_value` mode) **lowercase**-normalize karta hai comparison ke liye —
   dono jagah case-insensitivity ka intent same hai, implementation-detail alag
   (`UPPER(TRIM())` vs `lower()`), worth confirming consistency.
5. **`GET /blacklistvalues` do independent modes support karta hai** (same endpoint,
   different query param):
   - `glid` diya → `fn_sp_suspected_fraud_detection($1)` stored function call — JSON-array
     result parse hota hai
   - `attribute_value` diya (comma-separated) → direct `GL_BLACKLIST_VALUES` lookup, case-
     insensitive, multiple values ek saath
   [`UserBlacklistValuesModel.go:46-94`](../internal/models/users/UserBlacklistValuesModel.go)
6. **`UserBlacklistController` (dead) 100% hardcoded response deta hai** — chahe koi bhi
   input aaye, `"This API is deprecated now"`, code `500`. Poori validation/DB-write logic
   comment kiya hua hai, remove nahi kiya gaya.
   [`UserBlacklistController.go:139,169`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserBlacklistController.go)
7. **`QUERY_APPROVAL_BLACKLIST` check specifically `IS_DOMAIN=2` filter use karta hai** —
   matlab yeh table multi-purpose hai (`IS_DOMAIN` ek type-discriminator lagta hai), aur
   Business-Feed consumer sirf ek specific "domain" ke records consult karta hai.

---

## 5. Kafka / RabbitMQ / Redis

| System | Usage |
|---|---|
| **RabbitMQ** | Iss review mein koi RabbitMQ usage nahi mila blacklist-write/read paths mein |
| **Kafka** | [`USER_CHAT_BLACKLIST_DATA.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_CHAT_BLACKLIST_DATA.go) **Kafka-driven hai** (topic likely `lms_email_mob` per code comment) — chat/LMS system se blacklist-relevant signals (jaise `SENDER_GLID`/`RECIEVER_GLID`) ingest karta hai. Iska exact target-table (`GL_BLACKLIST_VALUES` ya `QUERY_APPROVAL_BLACKLIST`) iss pass mein pura trace nahi hua — **[INFERRED, confirm karo]** yeh possibly wahi jagah hai jahan se naye blacklist-values create hote hain (Business Doc §3.1 ka open question). |
| **Redis** | Koi usage nahi mila |

---

## 6. End-to-End Technical Flows

### Flow A — Admin blacklist-value update

```
Admin (GLADMIN/MAPI)
    │
    ▼
[API — write]  POST /user/blacklistvalues
    │  BlacklistValuesController.go
    │  1. Gateway validation (GLADMIN/MAPI only)
    │  2. Mandatory-field check (branches: normal vs WHITELISTBYGLID)
    ▼
[DB]  UPDATE GL_BLACKLIST_VALUES
    │  WHERE match on either GL_BLACKLIST_VALUE (normal) or FK_ORIGNAL_GLUSR_ID (whitelist)
    ▼
Response: "Data Updated Successfully" (or "No Record Exist" if UPDATE affected 0 rows)
```

**Important**: koi INSERT path nahi hai yahan — agar value pehle se table mein nahi hai,
`RowsAffected() == 0` → `"No Record Exist"`. Naya value kaise banta hai, iska answer likely
Flow C (chat-derived ingestion) mein hai, confirmed nahi.

### Flow B — Read blacklist-values / fraud-check

```
Caller (internal tool/admin)
    │
    ▼
[API — read]  GET /blacklistvalues
    │
    ├─ glid param diya →
    │     [DB — stored function]  fn_sp_suspected_fraud_detection(glid)
    │     └─ JSON array parse karke response banti hai
    │
    └─ attribute_value param diya →
          [DB]  SELECT b.*, a.GL_ATTRIBUTE_COLUMN_NAME
                FROM GL_BLACKLIST_VALUES b JOIN gl_attribute a ...
                WHERE lower(GL_BLACKLIST_VALUE) IN (...)
```

### Flow C — Chat-derived blacklist signal ingestion (Kafka)

```
Chat/LMS system (external trigger, topic "lms_email_mob")
    │
    ▼
[Kafka consume]  USER_CHAT_BLACKLIST_DATA.go
    │  message: SENDER_GLID, RECIEVER_GLID, ...
    ▼
[DB write — exact table not re-confirmed in this pass]
    │  likely feeds GL_BLACKLIST_VALUES or QUERY_APPROVAL_BLACKLIST
```

### Flow D — Domain-level real-time check (inline, no dedicated API)

```
Some other feature's pipeline (e.g. USER_BUSINESS_FEEDS consumer)
    │
    ▼
CheckUserBlocked() function call (inline, synchronous within that feature's own flow)
    │
    ▼
[DB]  SELECT ... FROM QUERY_APPROVAL_BLACKLIST WHERE IS_DOMAIN=2 AND BLACKLIST_EMAIL=...
    │
    ▼
Boolean result used to decide whether to proceed with that OTHER feature's business logic
```

### Flow E — ML Fraud Suspect read

```
Caller
    │
    ▼
[API — read]  GET /readmlfraud
    │  ReadMLFraudController.go → UserReadMLFraudModel.go
    ▼
[DB]  SELECT ... FROM ML_FRAUD_SUSPECTED_USERS
      LEFT OUTER JOIN ML_FRAUD_SUSPECT_EXCEPTIONS ON ...
      WHERE FK_GLUSR_USR_ID = ...
    ▼
Response: suspect status, probability score, exception details (if any)
```

---

## 7. Optimization Scope — aur ek Architecture Overlap Risk

### High-impact / correctness risk

1. **Do alag fraud-detection paths overlap kar sakte hain**: `GET /blacklistvalues?glid=...`
   ek stored function (`fn_sp_suspected_fraud_detection`) call karta hai, jabki
   `GET /readmlfraud` seedha `ML_FRAUD_SUSPECTED_USERS` table read karta hai. Dono naam se
   "fraud detection for a glid" jaisa lagte hain — **agar dono same underlying data
   represent karte hain, yeh duplicate maintenance/two-sources-of-truth risk hai**; agar
   alag data hain, naming se confuse karna easy hai. Team se confirm karna zaroori hai.

### Medium-impact

2. **Case-normalization inconsistency** (§4, point 4) — write-side `UPPER()`, read-side
   `lower()`. Functionally equivalent hai (dono case-insensitive compare karte hain), lekin
   agar kabhi koi third path raw-case pe compare kare, silent mismatch ban sakta hai.
3. **`GL_BLACKLIST_VALUES` mein koi INSERT path nahi mila application-code mein** — agar
   Kafka-consumer (Flow C) bhi isme write nahi karta (unconfirmed), toh yeh table
   **kaise grow karti hai** yeh genuinely unclear hai iss review se. Worth ek dedicated
   follow-up investigation.

### Low-impact

4. **Dead code cleanup opportunity**: `UserBlacklistController.go` poori tarah commented-out
   logic ke saath dead hai — safe hai remove karna (ya kam se kam route registration hataana,
   confirm karo pehle ki koi route ismein point nahi karta).

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora Blacklist flowchart yahan dekho](https://lucid.app/lucidchart/ca227b3c-49a1-4ba8-a71d-3d2e71a9202d/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **`ML_FRAUD_SUSPECTED_USERS` vs `fn_sp_suspected_fraud_detection` overlap** (§7, point 1)
   — sabse important open question iss domain mein.
2. **`GL_BLACKLIST_VALUES` growth-path unclear** (§7, point 3).
3. **`USER_CHAT_BLACKLIST_DATA` consumer ka target-table confirm nahi hua** (§5) — agar
   debugging karni pade "naya blacklist entry kahan se aaya," yahan se shuru karo.
4. **Dead `UserBlacklistController`** — agar koi purana client/integration abhi bhi isse
   call kar raha hai, unhe hamesha `"This API is deprecated now"` milega — silent failure
   nahi, explicit message hai, so kam risk, lekin phir bhi flag karne layak.

---

## 10. Open Questions

1. `fn_sp_suspected_fraud_detection` stored function `ML_FRAUD_SUSPECTED_USERS` se hi data
   leta hai, ya ek alag/derived source se? (§7, point 1)
2. `GL_BLACKLIST_VALUES` mein naye rows kis process se aate hain — `USER_CHAT_BLACKLIST_DATA`
   consumer se, ya koi aur (jaise ek offline/manual DB-script) tarika hai?
3. `QUERY_APPROVAL_BLACKLIST` ka `IS_DOMAIN` column kitne alag "domains"/types support
   karta hai, aur kaun-kaun se features iss table ko consult karte hain (`USER_BUSINESS_FEEDS`
   ke alawa)?
4. `UserBlacklistController` (dead) — kya isse safely remove kiya jaa sakta hai, ya kahin
   abhi bhi reference hai (routing table mein bhi check karna)?
5. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`Blacklist_Business_Doc.md`](./Blacklist_Business_Doc.md) — product perspective
- [`../trust_verification_compliance_read_write_picture.md`](../trust_verification_compliance_read_write_picture.md) —
  bigger Trust & Compliance technical trace, jahan `UserBlacklistController`
  dead-code-finding pehle bhi flag hui thi
- [`../utils_programming_guide.md`](../utils_programming_guide.md) — Kafka/consumer patterns
  jo `USER_CHAT_BLACKLIST_DATA` follow karta hai

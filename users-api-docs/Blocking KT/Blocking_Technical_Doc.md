# Buyer Blocking — Technical Doc (Code-Level Deep Dive)

**Scope note**: yeh doc sirf **Blocking** cover karta hai — **Matchmaking** alag module hai,
dekho [`../Matchmaking KT/Matchmaking_Technical_Doc.md`](../Matchmaking%20KT/Matchmaking_Technical_Doc.md).

Business/product perspective ke liye
[`Blocking_Business_Doc.md`](./Blocking_Business_Doc.md) dekho.

**Repos**: `service-api-go-production` (write), `users-api-go-production` (read, indirectly
via Buyer Profile).

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Block/Unblock submit | write | [`UserBlockUnblockController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserBlockUnblockController.go), [`UserBlockUnblockModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserBlockUnblockModel.go) |
| Mandatory-field validation | write | [`UserUtilsMandatory.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go) (`MandatoryParamsCheckUserBlockUnblock`) |
| Read usage (embedded in buyer profile) | read | [`UserBuyerProfileModel.go`](../internal/models/users/UserBuyerProfileModel.go) — LEFT JOINs this table with the matchmaking table in one query |

**Yeh module ka poora footprint bas 3 files mein hai** — GST/Rating/Privacy-Setting jaise
multi-consumer, multi-queue domains ke muqable, yeh **sabse chhota, purely synchronous**
feature hai teeno KT-series mein.

---

## 2. Routes

| Method | Path | Repo | Controller |
|---|---|---|---|
| POST | `/user/user_block_unblock` | write | `UserBlockUnblockController` |

Koi dedicated GET endpoint nahi hai — block-status sirf `GET /buyerprofile` ke andar
embedded milta hai (ek JOIN ke through).

---

## 3. Data Model — Table

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `USER_BLOCKED_STATUS` | meshpg | **Primary aur sirf record** — ek row per (supplier, blocked-buyer) pair | `user_glid`, `blocked_glid`, `block_status` (1=blocked, 0=unblocked), `blocking_date`, `unblocking_date`, `mod_id`, `user_ip` — [`UserBlockUnblockModel.go:34`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserBlockUnblockModel.go) |

**Primary key**: `(user_glid, blocked_glid)` — confirmed via `ON CONFLICT(user_glid,
blocked_glid) DO UPDATE` clause.

---

## 4. Business Rules & Validation (code se)

1. **`user_glid` aur `blocked_glid` dono numeric hone chahiye** — agar non-numeric string
   aaye, silently `"INVALID_VALUE"` mark ho jaata hai (jo aage validation mein reject hota
   hai).
   [`UserUtilsMandatory.go:848-868`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
2. **Gateway allowlist bada hai** (10 callers: `GLADMIN`, `SELLERMY`, `MY`, `MY1`, `IMOB`,
   `PNS`, `ANDROID`, `IOS`, `ANDWEB`, `IOSWEB`) — matlab yeh feature bahut saare
   client-surfaces se accessible hai (web, app, admin panel sab).
   [`UserBlockUnblockController.go:54`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserBlockUnblockController.go)
3. **`MY1` gateway ko response mein `MY` mein normalize kiya jaata hai** — ek chhota
   caller-identity-mapping quirk.
   [`UserBlockUnblockController.go:60-62`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserBlockUnblockController.go)
4. **`blocking_date`/`unblocking_date` column dynamically choose hoti hai** — `block_status
   == "1"` ho toh `blocking_date` set hoti hai, `== "0"` ho toh `unblocking_date` — dono
   columns ek saath maintain hote hain taaki dono actions ka apna-apna timestamp-history
   rahe, chahe current status kuch bhi ho.
   [`UserBlockUnblockModel.go:29-33`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserBlockUnblockModel.go)
5. **Ek single `UPSERT` (`INSERT ... ON CONFLICT DO UPDATE`) poora operation handle karta
   hai** — pehli baar block karna ho ya dobara block/unblock, sab isi ek query se ho jaata
   hai. Koi separate insert-vs-update branching logic nahi hai application-code mein — DB
   khud decide karta hai.
   [`UserBlockUnblockModel.go:34-35`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserBlockUnblockModel.go)

---

## 5. RabbitMQ / Kafka

**Koi messaging (RabbitMQ ya Kafka) nahi mila iss module mein.** Yeh poori tarah synchronous
hai — request aati hai, ek DB upsert hota hai, response chala jaata hai. Koi async
side-effect, koi downstream consumer, koi fan-out nahi.

---

## 6. Redis

**Koi Redis usage nahi mila.**

---

## 7. End-to-End Technical Flow

```
Supplier
    │
    ▼
[API — write]  POST /user/user_block_unblock  {user_glid, blocked_glid, blocked_status}
    │  UserBlockUnblockController.go
    │  1. Mandatory-field + numeric-format validation
    │  2. Gateway validation (10-caller allowlist)
    ▼
[DB — single upsert]  meshpg
    INSERT INTO USER_BLOCKED_STATUS(...)
    VALUES (...)
    ON CONFLICT(user_glid, blocked_glid) DO UPDATE
    SET block_status=..., blocking_date/unblocking_date=..., mod_id=..., user_ip=...
    ▼
Response: "UPDATE SUCCESS" ya failure message
```

**Poori pipeline ek hi HTTP request ke andar complete ho jaati hai** — koi asynchronous
step nahi, koi eventual-consistency concern nahi. Yeh iss module ki sabse badi
architectural simplicity hai baaki KT-series domains ke muqable.

### Read-side integration

```
Supplier
    │
    ▼
[API — read]  GET /buyerprofile
    │  UserBuyerProfileModel.go
    ▼
[DB]  SELECT ... FROM glusr_contactbook_mapping g
      LEFT JOIN USER_BLOCKED_STATUS u ON u.blocked_glid = ... AND u.user_glid = ...
    │  — matchmaking aur blocking dono ek hi query mein combine hote hain
    ▼
Response: buyer profile + connection-status + block-status, sab ek saath
```

---

## 8. Optimization Scope — DB Response-Time Contribution

Yeh sabse chhota domain hai — optimization scope bhi sabse chhota hai, kyunki architecture
already minimal hai.

### Observations (no major issues found)

1. **Single-query upsert already optimal hai** — koi extra round-trip, koi redundant
   SELECT-before-INSERT pattern nahi (jo GST/Rating domains mein dikha tha). Yeh sabse
   clean write-path hai teeno KT-series mein.
2. **`GET /buyerprofile`'s LEFT JOIN pattern efficient hai** — matchmaking aur blocking
   status ek hi query mein milte hain, do alag API calls ki zaroorat nahi. Yeh acha design
   hai — GST/Rating domains ke "no caching anywhere" gap iss specific join-query pe bhi
   apply hota hai (agar `GET /buyerprofile` high-traffic hai), lekin query khud efficient
   hai.
3. **Koi caching nahi hai**, lekin diye gaye chhote scope (ek table, single-key upsert) ke
   sath, yeh low-priority hai — is table pe likely low write-volume, low-latency-sensitivity
   hogi compared to GST/Rating jaise high-traffic domains.

**Overall**: iss module ke liye koi urgent optimization recommend nahi ki jaa rahi — yeh
already ek achi minimal, single-round-trip design hai.

---

## 9. Full Flow Diagram (Lucid, icon-based)

**[Poora Blocking flowchart yahan dekho](https://lucid.app/lucidchart/afce4cfe-cc6f-4c4e-a2d1-10d8f182ef78/edit)**

---

## 10. Edge Cases & Gotchas (technical POV)

1. **Enforcement point trace nahi hui** (Business Doc §5.1) — `USER_BLOCKED_STATUS` table
   khud sirf ek record hai; kaunsa system actual lead-delivery ke waqt isse consult karta
   hai (agar karta hai), iss review mein nahi mila. `UserBuyerProfileModel.go` sirf isse
   **display** ke liye read karta hai, enforcement ke liye nahi.
2. **`MY1`→`MY` normalization** (§4, point 3) — chhota quirk, agar kabhi debugging mein
   `modid` mismatch dikhe, yaad rakhna.

---

## 11. Open Questions

1. `USER_BLOCKED_STATUS.block_status` ko kaunsa system actively "enforce" karta hai (lead
   routing/delivery ke waqt)? Iss review ke scope se bahar tha.
2. Kya blocking ka koi upper-limit hai (ek supplier kitne buyers block kar sakta hai)? Code
   mein koi such limit nahi mila.
3. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`Blocking_Business_Doc.md`](./Blocking_Business_Doc.md) — product perspective
- [`../Matchmaking KT/Matchmaking_Technical_Doc.md`](../Matchmaking%20KT/Matchmaking_Technical_Doc.md) —
  related independent module, same read-query intersection point (`UserBuyerProfileModel.go`)
- [`../buyer_seller_discovery_matching_read_write_picture.md`](../buyer_seller_discovery_matching_read_write_picture.md) —
  original architecture trace this doc builds on

# Market Place URL — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`MarketPlaceUrl_Business_Doc.md`](./MarketPlaceUrl_Business_Doc.md)
dekho.

**Repos**: `service-api-go-production` (write), `users-api-go-production` (read).
**Koi RabbitMQ/Kafka/consumer nahi mila** — purely synchronous read/write, jaisa Blocking/
Social-Review/Image modules.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write | write | [`UserMarketPlaceUrlController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserMarketPlaceUrlController.go), [`UserMarketPlaceUrlModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMarketPlaceUrlModel.go) (`MktpInsertintoDB`, `MktpUpdateintoDB`, `GetUrlSeq`) |
| Write — validation/mandatory | write | `MandatoryFieldsMktpUrl` — [`UserUtilsMandatory.go:1202`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go); `ValidationMktpUrl` — [`UsersValidationMaps.go:2167`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |
| Read | read | [`MrktUrlController.go`](../../users-api-go-production/internal/controllers/UsersControllers/MrktUrlController.go), [`UserMrktlUrlModel.go`](../../users-api-go-production/internal/models/users/UserMrktlUrlModel.go) (`ActionMrktURLModel`) |

---

## 2. Routes

| Method | serviceName | Repo | Controller |
|---|---|---|---|
| POST | `MARKET_PLACE_URL_SERVICE` | write | `MarketPlaceUrlController` |
| — | `MARKET_PLACE_URL` | read | `ActionMarkUrl` |

---

## 3. Data Model — Table

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_MKTP_URL` | **write**: meshpg; **read**: mesh_pg_user (naming/physical-instance mismatch — Open Question) | Supplier ke marketplace-presence URLs, per-domain history-trail | `GLUSR_MKTP_URL_ID` (PK, `RETURNING` on insert), `FK_GLUSR_USR_ID`, `GLUSR_MKTP_URL`, `GLUSR_MKTP_URL_SEQ`, `GLUSR_MKTP_URL_DOMAIN` (`GOOGLE`/`FACEBOOK`/`INSTAGRAM`), `GLUSR_MKTP_URL_UPDATED_BY`/`_UPDATED_DATE`, `GLUSR_MKTP_URL_HISTUPDBY`/`_HISTUPDBY_ID`/`_HISTUPDBY_URL`/`_HISTUPD_AGENCY`/`_HISTUPD_SCREEN`, `GLUSR_MKTP_URL_HIST_IP`/`_HIST_IP_COUNTRY`, `GLUSR_MKTP_URL_HIST_COMMENTS`, `GLUSR_MKTP_URL_ADD_DATE`, `GLUSR_MKTP_URL_ENRCH_DATE`/`_ENRCH_BY`/`_ENRCH_FLAG`, `GLUSR_MKTP_URL_COMMENTS` — [`UserMarketPlaceUrlModel.go:120-125,271-285`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMarketPlaceUrlModel.go) |

---

## 4. Business Rules & Validation (code se)

1. **`GetUrlSeq(domainName)` ek hardcoded domain→sequence mapping hai** — `GOOGLE`=1,
   `FACEBOOK`=2, `INSTAGRAM`=3, koi aur domain=0 (unrecognized).
   [`UserMarketPlaceUrlModel.go:12-27`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMarketPlaceUrlModel.go)
2. **Mandatory fields**: `USR_ID`, `DOMAIN_NAME`, `MOD_ID`, `IP_COUNTRY`, `UPDATESCREEN`,
   `UPDATEDBY`, `VALIDATION_KEY`, `OP_FLAG` — sab required; `OP_FLAG` sirf `I`/`U`.
   `MKTP_URL` sirf `OP_FLAG="I"` (insert) case mein mandatory hai, update mein nahi
   (matlab update sirf metadata/history ke liye bhi ho sakta hai, URL value change kiye
   bina). [`UserUtilsMandatory.go:1202-1265`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
3. **Gateway allowlist**: `GLADMIN`, `Weberp`, `SELLERMY`, `MY`.
   [`UserUtilsMandatory.go:1268`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
4. **Update-query ke do variants hain, `MKTP_URL_ID` presence pe depend karta hai**:
   - Agar `MKTP_URL_ID` present hai → sirf metadata/enrichment-columns update hote hain
     (`ENRCH_*`, history-fields), URL value/seq nahi.
   - Agar absent hai → poora record update hota hai (`GLUSR_MKTP_URL` value bhi),
     enrichment-fields explicitly `NULL` reset ho jaate hain.
   [`UserMarketPlaceUrlModel.go:269-289`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMarketPlaceUrlModel.go)
5. **Update `WHERE FK_GLUSR_USR_ID = $N AND GLUSR_MKTP_URL_DOMAIN = $M`** — matlab per
   user+domain, latest-row update hoti hai (assuming ek hi row per domain maintain ho
   raha hai by convention, koi explicit unique-constraint code mein nahi dikha).
6. **Read-side dedup query `ROW_NUMBER() OVER (PARTITION BY glusr_mktp_url_domain ORDER
   BY GLUSR_MKTP_URL_UPDATED_DATE DESC) rn ... WHERE rn=1`** — matlab read-model khud
   latest-per-domain nikaalta hai, implying insert/history se multiple rows-per-domain
   accumulate ho sakti hain (table = append-friendly, read = dedup-on-query).
   [`UserMrktlUrlModel.go:27`](../../users-api-go-production/internal/models/users/UserMrktlUrlModel.go)

---

## 5. RabbitMQ / Kafka / Redis

**Koi RabbitMQ, Kafka, ya Redis usage nahi mila** — purely synchronous, single-table
insert/update aur read.

---

## 6. End-to-End Technical Flow

### Write
```
Supplier / Admin-tool
    │
    ▼
[API — write]  POST serviceName=MARKET_PLACE_URL_SERVICE
                {USR_ID, DOMAIN_NAME, OP_FLAG(I/U), MKTP_URL, MOD_ID, VALIDATION_KEY, ...}
    │  MarketPlaceUrlController.go
    │  GetUrlSeq(domain) → MKTP_URL_SEQ resolve
    │  MandatoryFieldsMktpUrl() — gateway check (GLADMIN/Weberp/SELLERMY/MY)
    │  ValidationMktpUrl()
    ▼
[DB — meshpg]
    ├─ OP_FLAG=I → MktpInsertintoDB() → INSERT INTO GLUSR_MKTP_URL (...) RETURNING ID
    └─ OP_FLAG=U → MktpUpdateintoDB()
          ├─ MKTP_URL_ID present  → metadata/enrichment-only UPDATE
          └─ MKTP_URL_ID absent   → full UPDATE (URL value + reset enrichment fields)
    ▼
Response {INSERT SUCCESS / UPDATE SUCCESS}
```

### Read
```
Buyer / Profile-viewer
    │
    ▼
[API — read]  serviceName=MARKET_PLACE_URL  {glusrid, modid}
    │  MrktUrlController.go — ActionMrktURLModel()
    ▼
[DB — mesh_pg_user]
    SELECT glusr_mktp_url, glusr_mktp_url_domain FROM (
        SELECT ..., ROW_NUMBER() OVER (PARTITION BY domain ORDER BY updated_date DESC) rn
        FROM glusr_mktp_url WHERE fk_glusr_usr_id=$1
        AND domain IN ('FACEBOOK','INSTAGRAM','GOOGLE')
    ) x1 WHERE rn=1
    ▼
Response {GOOGLE: url, FACEBOOK: url, INSTAGRAM: url}
```

---

## 7. Optimization Scope — DB Response-Time Contribution

### Medium impact

1. **Read query ek subquery + windowing function use karti hai per-request** — agar yeh
   high-frequency read hai (buyer-profile-load ka hissa), aur data slow-changing hai
   (supplier apna marketplace-URL baar-baar change nahi karta), **Redis-caching** iska
   strong candidate hai — GST/Rating docs mein bhi yeh pattern suggest hua hai.
2. **Read aur write alag physical-DB-instance names use kar rahe hain** (`meshpg` vs
   `mesh_pg_user`) — agar yeh actually alag replicas hain, replication-lag ke wajah se
   "maine update kiya, purana dikh raha hai" issue ho sakta hai — confirm karna zaroori
   hai ki yeh same underlying DB ke alag alias hain ya genuinely alag instances.

### Low impact

3. **History-append pattern (`GLUSR_MKTP_URL` mein multiple rows per domain accumulate ho
   sakte hain)** — agar table unbounded-grow karti hai, periodic archival/cleanup consider
   karo, jaisa TrustSeal's `TRUSTSEAL_HISTORY` ke liye already flag hua tha.

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora Market Place URL flowchart yahan dekho](https://lucid.app/lucidchart/4900e5d4-01cd-4bdc-92bd-07dfc576c1a5/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **Read `mesh_pg_user` vs write `meshpg`** — agar yeh do alag physical databases hain
   (not just aliases), turant-baad-write-read consistency guarantee nahi hai.
2. **Koi unique-constraint explicitly code mein nahi dikha `(FK_GLUSR_USR_ID,
   GLUSR_MKTP_URL_DOMAIN)` pe** — agar do concurrent inserts same user+domain ke liye aa
   jaayein, duplicate rows ban sakte hain (read-side dedup query isko silently handle kar
   leti hai, lekin root-cause fix nahi hai).
3. **`MKTP_URL_ID`-presence-based branching (point 4, §4) ka exact caller-flow trace nahi
   hua** — kaun/kab `MKTP_URL_ID` bhejta hai (jaise koi enrichment-cron/admin-tool) confirm
   nahi hua is pass mein.

---

## 10. Open Questions

1. `meshpg` (write) aur `mesh_pg_user` (read) same physical DB hain ya replicas — replication-
   lag ka risk hai kya?
2. `GLUSR_MKTP_URL_ID`-present update-path (enrichment-only) kaun trigger karta hai — koi
   dedicated enrichment-cron/tool iss pass mein nahi mila.
3. `(FK_GLUSR_USR_ID, GLUSR_MKTP_URL_DOMAIN)` pe DB-level unique-constraint hai kya?
4. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`MarketPlaceUrl_Business_Doc.md`](./MarketPlaceUrl_Business_Doc.md) — product perspective
- [`../TrustSeal KT/TrustSeal_Technical_Doc.md`](../TrustSeal%20KT/TrustSeal_Technical_Doc.md) —
  similar history-append-pattern precedent (`TRUSTSEAL_HISTORY`)

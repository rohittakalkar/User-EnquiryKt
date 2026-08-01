# Flips (Social-Commerce Video) — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`Flips_Business_Doc.md`](./Flips_Business_Doc.md) dekho.

**Repos**: `users-api-go-production` (read), `service-api-go-production` (write),
`user-temp-consumers-production` (Kafka-driven sync consumer).

**Scope note**: Flips **Account** (vendor-account/token mapping) aur Flips **Content**
(actual video-content) — dono ek hi combined folder mein, kyunki dono write-controllers
same tables (`GLUSR_FLIPS_MAP`, `GLUSR_FLIPS_DETAIL`) share karte hain aur ek hi
`FLIPS_WRITE_SERVICE` event/queue ka hissa hain.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Account read | read | [`FlipsAccountController.go`](../../users-api-go-production/internal/controllers/UsersControllers/FlipsAccountController.go), [`FlipsAccountModel.go`](../../users-api-go-production/internal/models/users/FlipsAccountModel.go) (`FlipsAccountDetail`) |
| Content read | read | [`FlipsContentController.go`](../../users-api-go-production/internal/controllers/UsersControllers/FlipsContentController.go), [`FlipsContentModel.go`](../../users-api-go-production/internal/models/users/FlipsContentModel.go) (`FlipsContentDetail`, `buildFlipsQuery`, `getPcItemDocVideos`) |
| Account write | write | [`UserFlipsAccountController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserFlipsAccountController.go), [`UserFlipsAccountModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFlipsAccountModel.go) |
| Content write | write | [`UserFlipsContentController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserFlipsContentController.go), [`UserFlipsContentModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFlipsContentModel.go) (`InsertFlipsContent`, `UpdateFlipsContent`, `buildFullUpdateQuery`) |
| Validation | write | `ValidationFlipsContent` — [`UsersValidationMaps.go:2546`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |
| Sync consumer | consumers | [`USER_FLIPS_CONTENT_SYNC.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_FLIPS_CONTENT_SYNC.go) — Kafka-driven |

---

## 2. Routes

| Method | Path/serviceName | Repo | Controller |
|---|---|---|---|
| — | `USER_FLIPS_ACC_DETAILS` (read, account) | read | `FlipsAccountDetail` |
| — | (read, content) | read | `FlipsContentDetail` |
| POST | serviceName `USER_FLIPS_ACCOUNT` | write | `UserFlipsAccount` |
| POST | serviceName `USER_FLIPS_CONTENT` | write | `UserFlipsContent` |

Route-registration file (gin-router path strings) is pass mein directly nahi mila; dono
write-controllers `FetchParams`/`serviceName` pattern se hi confirm hue (standard is codebase
ka pattern hai).

---

## 3. Data Model — Tables

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_FLIPS_MAP` | document_pg | Account-level vendor-token mapping, 1 row per user | `FK_GLUSR_USR_ID` (unique, upsert-key), `VENDOR_ID`, `VENDOR_ACCESS_TOKEN` — [`UserFlipsAccountModel.go:17`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFlipsAccountModel.go) |
| `GLUSR_FLIPS_DETAIL` | document_pg | Per-social-media-account detail record, disable-flag versioning | `FK_GLUSR_USR_ID`, `FK_GL_SOCIAL_MEDIA_ID`, `GLUSR_SOCIAL_ACCT_ID`, `GLUSR_FLIPS_DETAIL_ID`, `GLUSR_FLIPS_ACCT_DISABLE_FLAG`, `GLUSR_FLIPS_DETAIL_MOD_DATE` — [`UserFlipsAccountModel.go:68,143,173`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFlipsAccountModel.go) |
| `GLUSR_FLIPS_CONTENT_DETAIL` | document_pg | Actual video-content records | Content-ID, video-ID, disable-flag — [`UserFlipsContentModel.go:109,296`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFlipsContentModel.go) |

**Important**: teeno tables `document_pg` physical DB pe hain — baaki zyada-tar write-paths
is codebase mein `meshpg` use karte hain; Flips iska apvaad (exception) hai.

---

## 4. Business Rules & Validation (code se)

1. **`MAPPING_TYPE` do values leta hai** — `1` = Token-based auth, `2` = Meta-based
   (Facebook/Instagram) auth; invalid hone par request reject hoti hai.
   [`UserFlipsAccountController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserFlipsAccountController.go)
2. **Account-write gateway allowlist**: `FLIPS`/`MAPI`/`IMOB`.
3. **Content-write gateway allowlist chhota hai**: `FLIPS`/`IMOB` (MAPI missing yahan) —
   [`UserFlipsContentController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserFlipsContentController.go)
   `processFlipsRequest` function ke andar `Gateway_v1([]string{"FLIPS","IMOB"}, ...)`.
4. **Content-write `FLAG` I/U-based hai** (Insert/Update) — jaise Logo — mandatory fields:
   `glusrid`, `flag`, `flipsId`, `validationKey`, `CONTENT` key present hona chahiye.
5. **`GLUSR_FLIPS_MAP` par `ON CONFLICT (FK_GLUSR_USR_ID) DO UPDATE`** — ek user ka sirf
   ek hi active vendor-token-mapping hota hai, purana silently overwrite hota hai.
   [`UserFlipsAccountModel.go:17`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFlipsAccountModel.go)
6. **`GLUSR_FLIPS_DETAIL` par duplicate-account check hota hai** — `SOCIAL_ACCT_COUNT`
   query similarity ke against `FK_GL_SOCIAL_MEDIA_ID` ke sath check karti hai.
   [`UserFlipsAccountModel.go:68`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFlipsAccountModel.go)
7. **Disable-flag CTE pattern**: naya/updated detail-row insert/update hote hi, ek `WITH
   updated AS (...)` query baaki purane rows (`GLUSR_FLIPS_DETAIL_ID <> $N` same
   user+social-media ke) ko `GLUSR_FLIPS_ACCT_DISABLE_FLAG = 0`-reset kar deti hai — matlab
   **ek user+social-media-platform ke liye sirf ek active detail-row honi chahiye**, purani
   automatically disable ho jaati hai.
   [`UserFlipsAccountModel.go:173`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFlipsAccountModel.go)
8. **Content-write ek batch-style `buildFullUpdateQuery` helper use karta hai** — ek hi
   `UPDATE ... FROM (VALUES ...)`-jaisa statement, multi-row payload ke liye — per-row loop
   nahi (good practice, Social Review ke content-batch jaisa hi).
   [`UserFlipsContentModel.go:296`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserFlipsContentModel.go)

---

## 5. RabbitMQ / Kafka / Redis

| Queue / `SERVICENAME` | Publisher | Consumer(s) | Purpose |
|---|---|---|---|
| `FLIPS_WRITE_SERVICE` (per domain-wide `serviceToQueueMap`) | `UserFlipsAccountModel.go:232` (account writes) | [`USER_FLIPS_CONTENT_SYNC.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_FLIPS_CONTENT_SYNC.go) (Kafka-driven, function `dbActionFlipsContentSync`) | Downstream sync — same tables re-applied |

**Critical finding — dual-write to same tables**: `USER_FLIPS_CONTENT_SYNC.go` mein
consumer ke andar **hu-bahu wahi** `INSERT INTO GLUSR_FLIPS_MAP (...) ON CONFLICT
(FK_GLUSR_USR_ID) DO UPDATE ...` aur `UPDATE GLUSR_FLIPS_DETAIL SET ...` (including same
disable-flag CTE logic) queries mili — matlab write-API **aur** yeh consumer **dono** ek
hi `GLUSR_FLIPS_MAP`/`GLUSR_FLIPS_DETAIL` tables ko independently likhte hain. Naam
"CONTENT_SYNC" hai, lekin actual behavior account-level tables bhi touch karta hai — naam se
thoda misleading. Iss pass mein `GLUSR_FLIPS_CONTENT_DETAIL` (actual content table) ka koi
direct write consumer-side nahi mila — sirf account/map-level duplication confirm hui.

**Koi Redis usage nahi mila** iss feature mein.

---

## 6. End-to-End Technical Flow

### Account linking
```
Supplier / integration
    │
    ▼
[API — write]  serviceName=USER_FLIPS_ACCOUNT  {GLUSR_ID, MAPPING_TYPE(1/2), VENDOR_ID,
                FLIPS_ID, SOCIAL_ACCT_STATUS, VENDOR_TOKEN, SOCIAL_ACCT_ID,
                SOCIAL_MEDIA_ID, SIMILARITY_SCORE}
    │  UserFlipsAccountController.go — Gateway (FLIPS/MAPI/IMOB)
    ▼
[DB — document_pg]
    ├─ INSERT INTO GLUSR_FLIPS_MAP (...) ON CONFLICT (FK_GLUSR_USR_ID) DO UPDATE ...
    ├─ SELECT COUNT(...) FROM GLUSR_FLIPS_DETAIL WHERE ... (duplicate/similarity check)
    ├─ UPDATE/INSERT GLUSR_FLIPS_DETAIL (...)
    └─ WITH updated AS (...) — disable older detail-rows for same user+social-media
    ▼
[RabbitMQ]  SERVICENAME=FLIPS_WRITE_SERVICE
    ▼
[CONSUME — Kafka]  USER_FLIPS_CONTENT_SYNC.go (dbActionFlipsContentSync)
    └─ re-applies same GLUSR_FLIPS_MAP / GLUSR_FLIPS_DETAIL writes (documentPg conn)
```

### Content posting
```
Supplier / integration
    │
    ▼
[API — write]  serviceName=USER_FLIPS_CONTENT  {glusrid, flag(I/U), flipsId,
                validationKey, CONTENT}
    │  UserFlipsContentController.go — processFlipsRequest() — Gateway (FLIPS/IMOB)
    │  ValidationFlipsContent()
    ▼
[DB — document_pg]
    ├─ flag=I → InsertFlipsContent() → INSERT INTO GLUSR_FLIPS_CONTENT_DETAIL (...)
    └─ flag=U → UpdateFlipsContent() → buildFullUpdateQuery() batch-UPDATE
    ▼
Response {ROW_CNT / UPDATE_STATUS}
```

### Read side
```
Buyer / storefront-viewer
    │
    ▼
[API — read]  FlipsAccountDetail() / FlipsContentDetail()
    │  buildFlipsQuery() dynamically builds WHERE (glid, mediaID, contentID, disableFlag,
    │  contentVideoID)
    ├─ getPcItemDocVideos() — separate DISTINCT ON query for product-catalog videos
    └─ mergePcItemDocVideos() — merges catalog-videos with Flips-content, de-dupes on
       videoID
    ▼
Response — combined video list
```

---

## 7. Optimization Scope — DB Response-Time Contribution

### Medium impact

1. **Dual-write duplication (write-API + consumer both write `GLUSR_FLIPS_MAP`/
   `GLUSR_FLIPS_DETAIL`)** — is redundant hai agar dono paths hamesha same-data likh rahe
   hain; agar consumer sirf edge-case-reconciliation ke liye hai (jaisa naam "SYNC" suggest
   karta hai), toh iska trigger-condition/idempotency clearly document/verify karna chahiye
   taaki race-condition (stale overwrite by consumer after a newer direct-write) na ho.

### Low impact

2. **Read-side `getPcItemDocVideos` + Flips-content query alag-alag chalti hain, phir
   app-code mein merge hoti hai** — agar dono ek hi response ke liye zaroori hain har baar,
   ek combined query (UNION ya JOIN) round-trip kam kar sakti hai.
3. **`buildFullUpdateQuery` batch-pattern already achi practice hai** — GST/Rating jaisa
   per-row-loop-insert issue yahan nahi hai.

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora Flips flowchart yahan dekho](https://lucid.app/lucidchart/4822f706-62bb-4048-b626-ef367e129b00/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **Consumer re-applies account-writes** — agar consumer delay se process ho aur beech mein
   koi naya direct-write aa jaaye, consumer ka stale-data se overwrite ho sakta hai (no
   timestamp/version-check dikha in queries traced so far).
2. **Content-write gateway allowlist mein `MAPI` missing hai** (sirf `FLIPS`/`IMOB`) jabki
   account-write mein `MAPI` allowed hai — inconsistency, confirm karo intentional hai ya
   nahi.
3. **`document_pg` ek alag physical DB hai** baaki zyada-tar write-paths se — cross-DB joins/
   transactions is domain ke andar possible nahi honge agar kabhi Flips ko dusre user-data
   se combine karna pade.

---

## 10. Open Questions

1. `USER_FLIPS_CONTENT_SYNC.go` consumer kis trigger/purpose se account-level tables bhi
   likhta hai — poora backfill/reconciliation-logic (loop-source, kis Kafka-topic se
   consume karta hai) iss pass mein poora trace nahi hua.
2. Consumer `GLUSR_FLIPS_CONTENT_DETAIL` (actual content) ko bhi kabhi likhta hai ya sirf
   account/map-tables tak simit hai — is pass mein content-table ka write consumer-side nahi
   mila, confirm karna baaki hai.
3. `ValidationFlipsContent`'s poora body (mandatory-field/type-checks) detail mein nahi
   padha gaya.
4. Route-registration (actual `POST /...` path strings) gin-router file mein directly
   confirm nahi hui — serviceName-based hi trace hui.
5. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`Flips_Business_Doc.md`](./Flips_Business_Doc.md) — product perspective
- [`../Social Review KT/Social_Review_Technical_Doc.md`](../Social%20Review%20KT/Social_Review_Technical_Doc.md) —
  similar social-content pattern, purely synchronous (no consumer duplication issue there)

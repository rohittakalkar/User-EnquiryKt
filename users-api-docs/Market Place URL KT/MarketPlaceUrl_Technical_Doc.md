# Market Place URL — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`MarketPlaceUrl_Business_Doc.md`](./MarketPlaceUrl_Business_Doc.md)
dekho — dono docs same flows cover karte hain, bas alag audience ke liye.

**Repos**: `service-api-go-production` (write), `users-api-go-production` (read),
`user-temp-consumers-production` (**yeh doc jo naya discover hua hai** — ek live RabbitMQ
consumer hai isi domain ke liye, purani shallow doc mein miss ho gaya tha).

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file path diya
gaya hai). Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — confirm with team]** likh diya hai, guess nahi kiya.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write API (controller) | write | [`UserMarketPlaceUrlController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserMarketPlaceUrlController.go) |
| Write API (model — insert/update/seq) | write | [`UserMarketPlaceUrlModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMarketPlaceUrlModel.go) — `GetUrlSeq`, `MktpInsertintoDB`, `MktpUpdateintoDB` |
| Write — mandatory-field check | write | `MandatoryFieldsMktpUrl` — [`UserUtilsMandatory.go:1202-1274`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go) |
| Write — length/type validation | write | `ValidationMktpUrl` + `MrktpUrlMap` — [`UsersValidationMaps.go:2167`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go), field map at [`UsersValidationMaps.go:326-347`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |
| Read API (dedicated marketplace-url endpoint) | read | [`MrktUrlController.go`](../../internal/controllers/UsersControllers/MrktUrlController.go), [`UserMrktlUrlModel.go`](../../internal/models/users/UserMrktlUrlModel.go) — `ActionMrktURLModel` |
| Read (embedded, second read-site) | read | [`UserBuyerProfileModel.go:145-155`](../../internal/models/users/UserBuyerProfileModel.go) — `GET /buyerprofile` embeds `glusr_mktp_url` rows as `SOCIAL_PROFILES` inside the buyer-profile aggregate response |
| **RabbitMQ consumer — newly found, not in old doc** | consumers | [`USER_MKTP_URL.go`](../../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_MKTP_URL.go) — writes into `buyerprofilePg.glusr_mktp_url`, which is exactly what `GET /buyerprofile` reads (section 6, Flow C) |
| Queue-name mapping (write-repo side) | write | `PushToQueue`'s `serviceToQueueMap` — [`rabbitmq.go:25`](../../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go): `"MARKET_PLACE_URL_SERVICE": "USER_MKTP_URL"` |
| Route whitelist entry | write | [`requestValidation.go:28`](../../../service-api-go-production/service-api-go-production/pkg/middleware/requestValidation.go) — `/market_place_url` is in `whitelist_service` |

---

## 2. Routes

| Method | Path | Repo | Controller | Evidence |
|---|---|---|---|---|
| POST | `/market_place_url` | write | `UserControllers.MarketPlaceUrlController` | [`router.go:149,317`](../../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) |
| GET, POST | `/wservce/users/mrkturl/*params` | read | `UsersControllers.ActionMarkUrl` | [`routerUsers.go:268-269,508-509,648-649`](../../internal/api/users_router/routerUsers.go) (registered 3× across environment/version blocks) |
| GET, POST | `/wservce/users/buyerprofile/*params`, `/wservce/buyleads/BuyerProfile*params` | read | `UsersControllers.ActionBuyerProfile` → `GetBuyerProfileModel` (embeds marketplace URLs as `SOCIAL_PROFILES`) | [`routerUsers.go:214-215,232-233,498-499,556-557`](../../internal/api/users_router/routerUsers.go), controller at [`BuyerProfileController.go:14,70`](../../internal/controllers/UsersControllers/BuyerProfileController.go) |
| serviceName inside `/wservce/users/mrkturl/` request | `MARKET_PLACE_URL` (used for logging/Kibana, not a separate route) | read | — | [`MrktUrlController.go:31`](../../internal/controllers/UsersControllers/MrktUrlController.go) |

Gateway allowlist for the **write** endpoint (checked via `Gateway_v1`): `GLADMIN`, `Weberp`,
`SELLERMY`, `MY` — [`UserUtilsMandatory.go:1268`](../../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go).

---

## 3. Data Model — Table

> **Verification note**: table/column names yahan Go code ke andar embedded SQL strings se
> liye gaye hain (har row ke saamne file:line diya hai). Live DB schema se cross-verify
> **nahi** kiya gaya hai — kisi migration/integration se pehle woh zaroor karo.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_MKTP_URL` | **write**: `meshpg` — [`UserMarketPlaceUrlController.go:96`](../../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserMarketPlaceUrlController.go); **read (dedicated endpoint)**: `mesh_pg_user` — [`UserMrktlUrlModel.go:17`](../../internal/models/users/UserMrktlUrlModel.go); **read (buyer-profile embed)**: `buyer_profile_pg` — [`UserBuyerProfileModel.go:51`](../../internal/models/users/UserBuyerProfileModel.go); **consumer-write replica**: `buyerprofilePg` — [`USER_MKTP_URL.go:27`](../../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_MKTP_URL.go) | Supplier ke marketplace-presence URLs, per-domain history-trail | `GLUSR_MKTP_URL_ID` (PK, `RETURNING` on insert), `FK_GLUSR_USR_ID`, `GLUSR_MKTP_URL`, `GLUSR_MKTP_URL_SEQ`, `GLUSR_MKTP_URL_DOMAIN` (`GOOGLE`/`FACEBOOK`/`INSTAGRAM`), `GLUSR_MKTP_URL_UPDATED_BY`/`_UPDATED_DATE`, `GLUSR_MKTP_URL_HISTUPDBY`/`_HISTUPDBY_ID`/`_HISTUPDBY_URL`/`_HISTUPD_AGENCY`/`_HISTUPD_SCREEN`, `GLUSR_MKTP_URL_HIST_IP`/`_HIST_IP_COUNTRY`, `GLUSR_MKTP_URL_HIST_COMMENTS`, `GLUSR_MKTP_URL_ADD_DATE`, `GLUSR_MKTP_URL_UPD_DATE`, `GLUSR_MKTP_URL_ENRCH_DATE`/`_ENRCH_BY`/`_ENRCH_FLAG`, `GLUSR_MKTP_URL_COMMENTS` — [`UserMarketPlaceUrlModel.go:120-125,271-289`](../../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMarketPlaceUrlModel.go) |

**4 distinct DB-connection aliases point at what is presumably the same logical data**
(`meshpg`, `mesh_pg_user`, `buyer_profile_pg`, `buyerprofilePg`) — whether these are aliases
of the same physical instance, or genuinely separate replicas with lag risk, is
**[INFERRED — confirm with team]**. This matters directly: the `USER_MKTP_URL` consumer
(section 5) explicitly writes into `buyerprofilePg`, which is a **separate physical sync
target** from the `meshpg` the write-API itself writes to — i.e. the buyer-profile copy of
this data is asynchronous/eventually-consistent by design, not just an alias quirk.

---

## 4. Business Rules & Validation (code se)

1. **`GetUrlSeq(domainName)` ek hardcoded domain→sequence mapping hai** — `GOOGLE`=1,
   `FACEBOOK`=2, `INSTAGRAM`=3, koi aur domain=0 (unrecognized), lekin **0 bhi silently
   accept ho jaata hai** — koi explicit reject nahi hota agar domain in teeno mein na ho,
   yeh sirf `GLUSR_MKTP_URL_SEQ` column mein `0` store kar deta hai.
   [`UserMarketPlaceUrlModel.go:12-27`](../../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMarketPlaceUrlModel.go),
   called from [`UserMarketPlaceUrlController.go:74-80`](../../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserMarketPlaceUrlController.go)
2. **Mandatory fields**: `USR_ID`, `DOMAIN_NAME`, `MOD_ID`, `IP_COUNTRY`, `UPDATESCREEN`,
   `UPDATEDBY`, `VALIDATION_KEY`, `OP_FLAG` — sab required, warna
   `"Please Enter Mandatory(USR_ID/DOMAIN_NAME/UPDATEDBY/VALIDATION KEY/IP_COUNTRY/UpdateScreen/MOD_ID/OP_FLAG) Fields"`.
   [`UserUtilsMandatory.go:1202-1247`](../../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
3. **`MKTP_URL` sirf `OP_FLAG="I"` (insert) case mein mandatory hai**, update mein nahi —
   matlab update sirf metadata/history ke liye bhi ho sakta hai, URL value change kiye bina.
   Agar insert case mein khaali ho: `"MKTP_URL can't be empty"`.
   [`UserUtilsMandatory.go:1249-1257`](../../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
4. **`OP_FLAG` sirf `I` ya `U` accept karta hai**, warna `"Invalid Flag value"`.
   [`UserUtilsMandatory.go:1259-1263`](../../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
5. **Gateway allowlist**: `GLADMIN`, `Weberp`, `SELLERMY`, `MY` — koi aur caller
   `VALIDATION_KEY` se resolve nahi hoga aur `"Input gateway not validated"` milega.
   [`UserUtilsMandatory.go:1267-1274`](../../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
6. **Length/type validation ek shared map (`MrktpUrlMap`) se hota hai**: e.g. `MKTP_URL`
   max 500 chars, `DOMAIN_NAME` max 15 chars, `IP` max 100 chars, `HIST_COMMENTS`/
   `URL_COMMENTS` max 1000 chars, `UPDATEDBY_URL` max 255 chars, `HISTUPDBY` max 60 chars,
   `ENRICH_FLAG` numeric max 1 digit. `USR_ID`/`UPDATEDBY`/`UPDATEDBY_ID`/`MKTP_URL_ID` sab
   numeric, max 10 digits. [`UsersValidationMaps.go:326-347`](../../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go),
   validated via `ValidationMktpUrl` → `LengthAndTypeValidations_v2` — [`UsersValidationMaps.go:2167-2174`](../../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go)
7. **Update-query ke do variants hain, `MKTP_URL_ID` presence pe depend karta hai**:
   - Agar `MKTP_URL_ID` present hai (non-empty string) → sirf metadata/enrichment-columns
     update hote hain (`_UPDATED_BY`, history-fields, `ENRCH_*`, `SEQ`, `COMMENTS`) —
     **URL value nahi touch hoti**.
   - Agar absent hai → poora record update hota hai (`GLUSR_MKTP_URL` value bhi), aur
     enrichment-fields explicitly `NULL` reset ho jaate hain (`ENRCH_FLAG`/`ENRCH_BY`/
     `ENRCH_DATE = NULL`).
   [`UserMarketPlaceUrlModel.go:267-289`](../../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMarketPlaceUrlModel.go).
   Kaun/kab `MKTP_URL_ID` bhejta hai (enrichment-only path) — **[INFERRED — confirm with
   team]**, koi dedicated caller iss review mein trace nahi hua.
8. **Update `WHERE FK_GLUSR_USR_ID = $N AND GLUSR_MKTP_URL_DOMAIN = $M`** — matlab per
   user+domain, matching row(s) update hoti hain. Koi explicit unique-constraint code mein
   nahi dikha `(FK_GLUSR_USR_ID, GLUSR_MKTP_URL_DOMAIN)` pe — agar multiple rows match kar
   jaayein toh Postgres sab match hone waali rows ko update kar dega (silent multi-row
   update risk, section 9).
9. **Insert response check**: agar `RETURNING GLUSR_MKTP_URL_ID` se `0`/empty aaye, output
   `"INSERT FAILURE - NO mrkt_url_id RETURNED"` set hota hai, lekin HTTP-level yeh abhi bhi
   ek "output string" hai jo CODE/STATUS decide karta hai neeche (point 10).
   [`UserMarketPlaceUrlModel.go:147-151`](../../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMarketPlaceUrlModel.go)
10. **Success/failure sirf exact string-match se decide hota hai**: `output ==
    "INSERT SUCCESS" || output == "UPDATE SUCCESS"` → HTTP 200/`SUCCESSFUL`, warna 500/
    `FAILED`. [`UserMarketPlaceUrlController.go:143-149`](../../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserMarketPlaceUrlController.go)
11. **Read-side dedup query**: `ROW_NUMBER() OVER (PARTITION BY glusr_mktp_url_domain
    ORDER BY GLUSR_MKTP_URL_UPDATED_DATE DESC) rn ... WHERE rn=1`, sirf `GOOGLE`/
    `FACEBOOK`/`INSTAGRAM` domains ke liye filtered — matlab read-model khud latest-per-
    domain nikaalta hai, implying table = append-friendly (history), read = dedup-on-query.
    [`UserMrktlUrlModel.go:27`](../../internal/models/users/UserMrktlUrlModel.go)
12. **Buyer-profile embed query alag hai — sirf `LIMIT 1`, koi per-domain window-function
    nahi**: `SELECT ... FROM glusr_mktp_url WHERE fk_glusr_usr_id = $1 ... LIMIT 1` wrapped
    in `array_to_json(array_agg(...))` — yeh sirf **1 row** deta hai (poora array nahi,
    `LIMIT 1` outer subquery pe hai), jabki dedicated `/mrkturl` endpoint teeno domains ka
    latest row alag-alag deta hai. **Behavioral inconsistency between the two read paths**
    — [`UserBuyerProfileModel.go:145-155`](../../internal/models/users/UserBuyerProfileModel.go)
    vs [`UserMrktlUrlModel.go:27`](../../internal/models/users/UserMrktlUrlModel.go).
    Business impact aur exact intent **[INFERRED — confirm with team]**, dekho section 9.

---

## 5. RabbitMQ — Live Consumer Found (Correction to Prior Doc)

**Purani shallow doc ka claim "Koi RabbitMQ/Kafka/consumer nahi mila" galat nikla** — ek
live RabbitMQ consumer maujood hai iss domain ke liye:

| Queue / `SERVICENAME` | Publisher | Consumer | Purpose |
|---|---|---|---|
| `USER_MKTP_URL` (queue), mapped from `SERVICENAME=MARKET_PLACE_URL_SERVICE` in the shared `PushToQueue` map | **Not found in `UserMarketPlaceUrlController.go` / `UserMarketPlaceUrlModel.go`** — see gap below | [`USER_MKTP_URL.go`](../../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_MKTP_URL.go) (`UserMktpUrl` → `workerMktpUrl` → `dbActionMktpUrl`), registered in [`Router/router.go:43`](../../../user-temp-consumers-production/user-temp-consumers-production/internal/Router/router.go) and [`IntializeMsgBroker.go:196`](../../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go) | INSERT/UPDATE `GLUSR_MKTP_URL` fields into `buyerprofilePg` — a **downstream replica** used specifically so `GET /buyerprofile` can read marketplace URLs without hitting `meshpg` |

**Evidence the write-API doesn't currently publish this**: `UserMarketPlaceUrlController.go`
hardcodes `rabbitTime := float64(0)` ([line 134](../../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserMarketPlaceUrlController.go)) and neither it nor `UserMarketPlaceUrlModel.go`
calls `utils.PushToQueue(...)` anywhere (`grep -rn "PushToQueue(" UserMarketPlaceUrlController.go UserMarketPlaceUrlModel.go` → no matches). So:

- The queue-name mapping **exists** in the shared `serviceToQueueMap`
  ([`rabbitmq.go:25`](../../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go)).
- The **consumer is live and functional** (`USER_MKTP_URL.go` fully implemented, handles
  both `INSERT` and `UPDATE` actions, has panic-recovery + email-alert + Kibana logging).
- But **no publisher was found** in the current write controller/model in this codebase pass.

**[INFERRED — confirm with team]**: either (a) this is vestigial wiring from an earlier
version of the controller that did publish and was later removed/refactored out, (b) some
other caller not covered by this file map publishes to it (e.g. a legacy PHP path, an admin
tool, or a different Go service not in these three repos), or (c) `buyerprofilePg`'s copy of
`glusr_mktp_url` is now stale/unmaintained by this specific mechanism. This is the single
most important open question in this doc — flag to BI/Users-Platform team before assuming
buyer-profile marketplace-URL data is being kept in sync.

No other queue/`SERVICENAME` reference related to marketplace-URL was found:
`grep -rn "MARKET_PLACE_URL_SERVICE"` across both write and read repos surfaces only the
controller's own `serviceName` string, the shared `PrepareResponse`/`PrepareKibanaResponse`
switch cases (generic response-formatting helpers, not publish calls), and the
`rabbitmq.go` map entry above.

---

## 6. Kafka

**Koi Kafka usage nahi mila** iss domain mein — `grep -rni "kafka"` marketplace-url files
aur unke immediate call-chain (`UserMarketPlaceUrlController.go`, `UserMarketPlaceUrlModel.go`,
`MrktUrlController.go`, `UserMrktlUrlModel.go`, `USER_MKTP_URL.go`) mein koi match nahi
deta.

---

## 7. Redis

**Koi Redis usage nahi mila** — `grep -n "Redis\|redis"` upar diye gaye sabhi files mein
koi match nahi deta. Dono read paths (`/mrkturl`, marketplace-URL portion of
`/buyerprofile`) har request pe live Postgres hit karte hain.

---

## 8. End-to-End Technical Flows

### Flow A — Write (Insert/Update)
```
Supplier / Admin-tool (GLADMIN, Weberp, SELLERMY, MY)
    │
    ▼
[API — write]  POST /market_place_url
                {USR_ID, DOMAIN_NAME, OP_FLAG(I/U), MKTP_URL, MOD_ID, VALIDATION_KEY, ...}
    │  UserMarketPlaceUrlController.go
    │  1. GetUrlSeq(domain) → MKTP_URL_SEQ resolve (0 if unrecognized domain — not rejected)
    │  2. InputJsonCheck — basic JSON shape validation
    │  3. MandatoryFieldsMktpUrl() — required fields + gateway check
    │  4. ValidationMktpUrl() — length/type validation (MrktpUrlMap)
    ▼
[DB — meshpg]
    ├─ OP_FLAG=I → MktpInsertintoDB() → INSERT INTO GLUSR_MKTP_URL (...) RETURNING ID
    └─ OP_FLAG=U → MktpUpdateintoDB()
          ├─ MKTP_URL_ID present  → metadata/enrichment-only UPDATE
          └─ MKTP_URL_ID absent   → full UPDATE (URL value + reset enrichment fields)
    ▼
Response {STATUS, CODE, MESSAGE: "INSERT SUCCESS"/"UPDATE SUCCESS"/error string, RESPONSE_DATA}
    (no RabbitMQ publish observed in this code path — section 5)
```

### Flow B — Dedicated Read (`/mrkturl`)
```
Buyer / Profile-viewer
    │
    ▼
[API — read]  GET or POST /wservce/users/mrkturl/*params  {glusrid, modid}
    │  MrktUrlController.go → ActionMrktURLModel()
    ▼
[DB — mesh_pg_user]
    SELECT glusr_mktp_url, glusr_mktp_url_domain FROM (
        SELECT ..., ROW_NUMBER() OVER (PARTITION BY domain ORDER BY updated_date DESC) rn
        FROM glusr_mktp_url WHERE fk_glusr_usr_id=$1
        AND domain IN ('FACEBOOK','INSTAGRAM','GOOGLE')
    ) x1 WHERE rn=1
    ▼
Response {GOOGLE: url, FACEBOOK: url, INSTAGRAM: url}  -- up to 3 rows, one per domain
```

### Flow C — Embedded Read (`/buyerprofile`) + async replica-sync via consumer
```
Buyer-facing profile viewer
    │
    ▼
[API — read]  GET/POST /buyerprofile (or /buyleads/BuyerProfile)
    │  BuyerProfileController.go → GetBuyerProfileModel()
    ▼
[DB — buyer_profile_pg]
    SELECT glusr_mktp_url.glusr_mktp_url AS URL, glusr_mktp_url.glusr_mktp_url_domain AS domain
    FROM glusr_mktp_url WHERE fk_glusr_usr_id = $1 LIMIT 1     -- note: LIMIT 1, only 1 row
    → embedded as SOCIAL_PROFILES inside the larger buyer-profile JSON response
    │
    │  (this table on buyer_profile_pg is populated/kept current by:)
    ▼
[RabbitMQ CONSUME]  USER_MKTP_URL.go — workerMktpUrl()
    │  expects JSON: {ACTION: INSERT|UPDATE, GLUSR_ID, GLUSR_MKTP_URL_DOMAIN,
    │                  GLUSR_MKTP_URL_UPDATED_DATE, GLUSR_MKTP_URL, GLUSR_MKTP_URL_ID, ...}
    ▼
[DB — buyerprofilePg]
    ACTION=INSERT → INSERT INTO glusr_mktp_url(...)
    ACTION=UPDATE (MKTP_URL_ID present)  → UPDATE ... SET glusr_mktp_url_updated_date=$1
                                             WHERE fk_glusr_usr_id=$2 AND domain=$3
    ACTION=UPDATE (MKTP_URL_ID absent)   → UPDATE ... SET url=$1, updated_date=$2
                                             WHERE fk_glusr_usr_id=$4 AND domain=$5
    │  on panic: emails "PANIC IN USER_MKTP_URL(GCP-IN) CONSUMER"
    │  on failure: Nack + Kibana log; on success: Ack + Kibana log
```
**Publisher of the message this consumer expects was not found** — see section 5 for the
gap and open question.

---

## 9. Flow-wise DB & Table Usage

### Flow A — Write

| # | DB (physical) | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `GLUSR_MKTP_URL` | INSERT (`OP_FLAG=I`) or UPDATE (`OP_FLAG=U`) | Primary write of supplier's marketplace URL + history/enrichment metadata |

**Total DB round-trips: 1** — this is a simple, single-table, single-DB write. No
lock-checks, no cross-table reads before writing (unlike GST's multi-DB fan-out).

### Flow B — Dedicated Read

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | mesh_pg_user | `GLUSR_MKTP_URL` (windowed subquery) | SELECT | Latest URL per domain (Google/Facebook/Instagram) for one supplier |

### Flow C — Buyer-Profile Embedded Read

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | buyer_profile_pg | `GLUSR_MKTP_URL` (correlated subquery, `LIMIT 1`) | SELECT | One marketplace-URL row embedded into the larger buyer-profile aggregate response (this query is one of many sub-selects in a large `GetBuyerProfileModel` query — not isolated cost) |
| 2 | buyerprofilePg (async, via consumer) | `glusr_mktp_url` | INSERT/UPDATE | Keeps the buyer-profile-side copy in sync with the meshpg source-of-truth — **but see section 5: no confirmed publisher in current code** |

**Total round-trips for a full write→buyer-profile-visible cycle: 1 (write) + 1 (consumer
write, if/when triggered) + 1 (buyer-profile read)** — spread across 3 distinct DB aliases
(`meshpg`, `buyerprofilePg`, `buyer_profile_pg`).

---

## 10. Optimization Scope — DB / Response-Time Contributors

### High-impact

1. **Confirm/fix the missing publisher (section 5) before optimizing anything else.** If
   `buyerprofilePg.glusr_mktp_url` is not actually being kept in sync (because no code
   publishes to `USER_MKTP_URL`), then `GET /buyerprofile`'s `SOCIAL_PROFILES` field is
   either stale, permanently empty for post-cutover writes, or backed by some
   out-of-repo mechanism — this is a correctness risk, not just a performance one.

### Medium-impact

2. **No caching on either read path** (section 7) — `GET /mrkturl` and the marketplace-URL
   portion of `GET /buyerprofile` both hit live Postgres every request. Marketplace URLs
   change infrequently per-supplier, making this a natural Redis cache-aside candidate
   (same pattern suggested in GST/TrustSeal docs) — worth evaluating if `/buyerprofile`
   (a high-traffic buyer-facing endpoint) shows this subquery as a meaningful cost
   contributor in profiling.
3. **Two read paths diverge in shape and behavior** (section 4, point 12) — `/mrkturl`
   returns up to 3 rows (one per domain, latest via `ROW_NUMBER()`), `/buyerprofile`'s embed
   returns only 1 row (`LIMIT 1`, no domain filter beyond the correlated subquery scope).
   If callers assume both return "all 3 domains," `/buyerprofile` will silently
   under-deliver — worth confirming intended behavior with the team.

### Low-impact / structural notes

4. **History-append pattern**: `GLUSR_MKTP_URL` is written via INSERT (new rows) and
   UPDATE (existing rows) without any DELETE observed in the reviewed code — combined with
   the dedup-on-read pattern (`ROW_NUMBER()` in `/mrkturl`), this strongly suggests the
   table accumulates historical rows over time, similar to TrustSeal's `TRUSTSEAL_HISTORY`
   pattern flagged in that KT doc. No explicit archival/cleanup job was found in this
   review (section 12 confirms no cron either).
5. **`GetUrlSeq` silently accepting unrecognized domains as `0`** (section 4, point 1) is
   not a performance issue but a data-quality one — worth flagging alongside the
   optimization list since it affects the `GLUSR_MKTP_URL_SEQ` column's reliability as a
   sort/lookup key.

---

## 11. Cron Inventory

**No cron/scheduler referencing marketplace-URL was found.**
`grep -rli "mktp\|mrkturl\|market_place_url"` against `service-api-go-production`'s cron
directories and against `user-temp-consumers-production` returned no matches beyond the
files already covered in the file map (section 1) and the consumer in section 5. If a
reconciliation/backfill job for `buyerprofilePg` exists, it was not found in these three
repos — **[INFERRED — confirm with team]**.

---

## 12. Edge Cases & Gotchas (technical POV)

1. **`buyerprofilePg` replica-sync publisher not found** (section 5) — the single most
   important gap in this doc. Confirm with team before assuming `/buyerprofile`'s
   marketplace-URL data reflects recent writes.
2. **4 different DB-connection aliases touch what's presumably one logical table**
   (`meshpg`, `mesh_pg_user`, `buyer_profile_pg`, `buyerprofilePg`, section 3) — if any pair
   of these are genuinely separate physical databases (not just aliases of one instance),
   there is a real replication-lag / read-your-writes risk for suppliers who write then
   immediately re-check via `/mrkturl` or `/buyerprofile`.
3. **No unique constraint observed** on `(FK_GLUSR_USR_ID, GLUSR_MKTP_URL_DOMAIN)` in the
   reviewed SQL — concurrent inserts for the same user+domain could create duplicate rows;
   both read paths' dedup logic (windowed `ROW_NUMBER()` / `LIMIT 1`) papers over this but
   doesn't fix the root cause.
4. **`MKTP_URL_ID`-presence branching** (section 4, point 7) changes update behavior
   significantly (metadata-only vs. full-value update) but the caller that supplies
   `MKTP_URL_ID` for the metadata-only path was not identified in this review.
5. **Unrecognized domain names are silently accepted with `SEQ=0`**, not rejected —
   contrasts with the business doc's plain-language framing that "only 3 platforms are
   supported"; in practice the write API doesn't enforce that at the domain-name level,
   only the sequence-number lookup degrades gracefully.
6. **`/buyerprofile`'s embedded marketplace-URL sub-select uses `LIMIT 1` with no domain
   filter and no explicit `ORDER BY`** inside the correlated subquery — which single row
   comes back when a user has URLs for multiple domains is **[INFERRED — confirm with
   team]**, likely Postgres' unspecified row order for an unordered `LIMIT 1`, i.e.
   effectively arbitrary/non-deterministic without an `ORDER BY`.

---

## 13. Open Questions

1. Who/what publishes to `SERVICENAME=MARKET_PLACE_URL_SERVICE` / queue `USER_MKTP_URL`
   (section 5)? The mapping and the consumer both exist and are wired up, but no call site
   was found in `UserMarketPlaceUrlController.go` / `UserMarketPlaceUrlModel.go`.
2. Are `meshpg`, `mesh_pg_user`, `buyer_profile_pg`, and `buyerprofilePg` aliases of the
   same physical Postgres instance, or genuinely separate databases/replicas (section 3)?
3. Who/what sends a request with `MKTP_URL_ID` populated, triggering the metadata-only
   update path (section 4, point 7)?
4. Is the `/buyerprofile` embed's lack of `ORDER BY` inside its `LIMIT 1` subquery
   (section 12, point 6) intentional, or a latent bug that returns a non-deterministic
   domain when a supplier has more than one marketplace URL on file?
5. Is there a reconciliation/backfill job anywhere (possibly outside these three repos)
   that keeps `buyerprofilePg` in sync if the RabbitMQ path (section 5) turns out to be
   unused?
6. Live DB schema verification (column types, nullability, indexes, constraints,
   particularly whether `(FK_GLUSR_USR_ID, GLUSR_MKTP_URL_DOMAIN)` has a unique index) —
   this doc only reflects what the Go SQL strings imply.

---

## See also

- [`MarketPlaceUrl_Business_Doc.md`](./MarketPlaceUrl_Business_Doc.md) — product perspective
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — depth/structure
  reference this doc was rebuilt against
- [`../TrustSeal KT/TrustSeal_Technical_Doc.md`](../TrustSeal%20KT/TrustSeal_Technical_Doc.md) —
  similar history-append-pattern precedent (`TRUSTSEAL_HISTORY`)
- [`../user_consumers_reference.md`](../user_consumers_reference.md) — full consumer
  inventory, worth cross-checking `USER_MKTP_URL` against

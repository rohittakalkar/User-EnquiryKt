# Buyer-Supplier Matchmaking — Technical Doc (Code-Level Deep Dive)

**Scope note**: yeh doc sirf **Matchmaking** cover karta hai — **Blocking** alag module hai,
dekho [`../Blocking KT/Blocking_Technical_Doc.md`](../Blocking%20KT/Blocking_Technical_Doc.md).

Business/product perspective ke liye
[`Matchmaking_Business_Doc.md`](./Matchmaking_Business_Doc.md) dekho.

**Repos**: `service-api-go-production` (write), `user-temp-consumers-production` (3 consumers).

**Verification note**: table/column names is doc mein Go code ke andar likhi SQL
strings se liye gaye hain — ek live, checked-out DB schema se cross-verify nahi kiya gaya hai.
Jahan bhi field ka business-meaning code se seedha clear nahi tha, use
**[INFERRED — confirm with team]** maaka gaya hai ya §12 Open Questions mein daala gaya hai.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Matchmaking record submit (sync write) | write | [`BsMatchMakingController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/BsMatchMakingController.go), [`BsMatchMakingModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/BsMatchMakingModel.go) |
| Kafka consume + DB write (async "mirror" write) | consumers | [`USER_BS_MATCHMAKING_KAFKA.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_KAFKA.go) |
| LMS-originated write (RabbitMQ, independent producer) | consumers | [`USER_BS_MATCHMAKING_LMS.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_LMS.go) — queues `USER_BS_MATCHMAKING_LMS` / `USER_BS_MATCHMAKING_LMS_FAIL`, both routed to the same handler ([`router.go:42,107`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Router/router.go)) |
| Consumed by (downstream anti-fraud check) | consumers | [`USER_RATING_BS_MATCHMAKING.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_BS_MATCHMAKING.go) — Rating pipeline's CDC-driven anti-fraud check; queues `USER_RATING_BS_MATCHMAKING` / `USER_RATING_BS_MATCHMAKING_FAIL` ([`router.go:49,108`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Router/router.go)) |
| Read usage (buyer profile, this repo) | read | [`UserBuyerProfileModel.go:539-549`](../../internal/models/users/UserBuyerProfileModel.go) — `LEFT JOIN` of `glusr_contactbook_mapping` with `USER_BLOCKED_STATUS` in one query |
| Queue-name → handler registry | consumers | [`IntializeMsgBroker.go:198,235,249,363,368`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go), [`router.go:42,49,78,107,108`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Router/router.go) |

---

## 2. Routes

| Method | Path | Repo | Controller |
|---|---|---|---|
| POST | `/user/bsmatchmaking` | write | `BsMatchMakingController` — [`BsMatchMakingController.go:15`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/BsMatchMakingController.go) |

Koi dedicated GET endpoint nahi hai — matchmaking status sirf doosre reads (jaise
`GET /buyerprofile`, via `UserBuyerProfileModel.go`) ke andar embedded milta hai.

---

## 3. Data Model — Tables

| Table | Physical DB (connection alias) | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_CONTACTBOOK_MAPPING` (write side) / `glusr_contactbook_mapping` (read/consumer side, lowercase in SQL strings) | connection alias `buyerprofilepg` in both repos — **same physical instance in prod**: service-api-go's `buyerprofilepg` (prod host `10.90.140.5`, db `meshpg`) [`config.yml:542-547`](../../service-api-go-production/service-api-go-production/data/config.yml) and consumer's `buyerprofilePg` (prod host `10.90.140.5`, db `meshpg`) [`config.yaml:154-159`](../../user-temp-consumers-production/user-temp-consumers-production/data/config.yaml) resolve to the identical host/db/user | Single, canonical matchmaking table — a distinct `meshPg` connection alias also exists in the consumer's config (host `35.200.136.127`, db `mesh`) but **is not used by any of the three matchmaking workers** — confirmed by grep, none of `USER_BS_MATCHMAKING_KAFKA.go`, `USER_BS_MATCHMAKING_LMS.go`, `USER_RATING_BS_MATCHMAKING.go` reference `meshPg` | `GLUSR_USR_ID1` = **supplier_id** (confirmed 3x: write-side param order [`BsMatchMakingModel.go:30-33`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/BsMatchMakingModel.go), read-side `s_glid` in [`UserBuyerProfileModel.go:545`](../../internal/models/users/UserBuyerProfileModel.go), Rating consumer's `supplierID` in [`USER_RATING_BS_MATCHMAKING.go:87`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_BS_MATCHMAKING.go)); `GLUSR_USR_ID2` = **buyer_id**; `SOURCE_ID`; `IS_DUAL_SIDE_ADDED`/`contact_type` (3rd positional param of `sp_insert_contacts`, named `cotact_type` — sic, typo present in the SQL string itself, [`BsMatchMakingModel.go:54`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/BsMatchMakingModel.go)) |

**Correction to prior pass**: the earlier doc described the Kafka-consumer write as going to a
*separate* physical "meshPg mirror" DB from the synchronous write. Re-checking both repos'
prod config confirms the connection alias used by the async consumers (`buyerprofilePg`) and
the alias used by the synchronous write (`buyerprofilepg`) point to the **same host, same
db name (`meshpg`), same user** in prod. So the "second write" (§8) is a **duplicate write to
the same physical database**, not a cross-database mirror — see Optimization Scope §9 point 1
for the cost implication of this.

---

## 4. Decode-the-magic-values

| Field / value | Where used | Meaning (from code) |
|---|---|---|
| `SOURCE_ID` (unset/null) | Rating consumer treats as `-1` via `COALESCE(source_id,-1)` [`USER_RATING_BS_MATCHMAKING.go:87`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_BS_MATCHMAKING.go) | Default/organic contact source |
| `SOURCE_ID = "1"` | Kafka consumer's dual-side flip condition [`USER_BS_MATCHMAKING_KAFKA.go:112`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_KAFKA.go); also whitelisted in Rating anti-fraud's `source_id in (-1,1,3,4)` check | **[INFERRED — confirm with team]** likely "LMS/lead" source — exact business label not found in code |
| `SOURCE_ID = "3"` or `"4"` | Kafka consumer: DB write is **skipped entirely**, only a "already inserted by WRITE SERVICE" success-log is written [`USER_BS_MATCHMAKING_KAFKA.go:83-85`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_KAFKA.go) | These sources are treated as "already persisted by a different write path" — the Kafka consumer only logs, doesn't re-insert. Also whitelisted in Rating's `source_id in (-1,1,3,4)` check, so records from these sources DO count for the anti-fraud check even though this consumer doesn't insert them itself. **[INFERRED — confirm with team]** exact identity of sources 3/4 |
| `insertion_type` / `contact_type` (write API param, default `"2"`) | [`BsMatchMakingModel.go:40-46`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/BsMatchMakingModel.go) | Passed as 3rd positional arg to `sp_insert_contacts` (the `cotact_type` slot). **Naming oddity found in this pass**: the same value is separately stuffed into the Kafka payload under the key `"is_dual_side_added"` [`BsMatchMakingModel.go:37,42`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/BsMatchMakingModel.go) — i.e. on the sync-write side, "contact_type" and "is_dual_side_added" are literally the same variable (`isDual`), which reads as a naming/data mislabel rather than two independent concepts. Flagged in §11 Gotchas. |
| `is_dual_side_added` (Kafka consumer's own recompute) | [`USER_BS_MATCHMAKING_KAFKA.go:112-116`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_KAFKA.go): if `SOURCE_ID=="1"` AND incoming `IS_DUAL_SIDE_ADDED=="1"` → write `"2"`, else write `"1"` | The Kafka consumer does **not** trust the producer's `is_dual_side_added` value as-is; it re-derives it. So the value actually persisted to `meshpg` via the async path can differ from what the sync write intended. |
| `is_dual_side` (LMS consumer's own recompute) | [`USER_BS_MATCHMAKING_LMS.go:104-108`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_LMS.go): if `1` → `2`, else → `1` (unconditional flip, no `SOURCE_ID` gate unlike the Kafka consumer) | Same "flip 1↔2" idea as the Kafka consumer but **simpler/unconditional** — a third, independently-coded variant of the same business rule. |
| `CALLEDFROM = "MATCHMAKING_CONSUMER"` / `"MCAT_CONSUMER"` | Rating consumer's own loop-guard [`USER_RATING_BS_MATCHMAKING.go:69-71`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_BS_MATCHMAKING.go); same guard string reused in [`USER_RATING_SELLER_RISK.go:67-70`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_SELLER_RISK.go) and [`USER_RATING_ENRICHMENT.go:69`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_ENRICHMENT.go) | A shared CDC-loop-prevention convention across the Rating pipeline — messages the anti-fraud consumer itself generated (via its own write-back, see §8) are tagged this way so the CDC pipeline doesn't re-process its own writes. |
| `GLUSR_RATING_DISPLAY_STATUS = -1`, `GLUSR_RATING_REVIEWED_BY = -999` | Written by the anti-fraud consumer when no matchmaking record is found [`USER_RATING_BS_MATCHMAKING.go:105-106`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_BS_MATCHMAKING.go) | `-1` = rating hidden/disabled display status; `-999` = a synthetic "system" reviewer ID (not a real admin), consistent with the comment text `"Rating Disabled : ... By: -999"` |

---

## 5. Business Rules & Validation (code se)

1. **Mandatory fields**: `buyer_id`, `supplier_id`, `VALIDATION_KEY` — inke bina
   `"Please Enter Mandatory(buyer_id / supplier_id / VALIDATION_KEY) Fields"`.
   [`BsMatchMakingController.go:78-79`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/BsMatchMakingController.go)
2. **Gateway validation sirf `"LEADS"` allowlist ke against hoti hai** — ek chhota,
   focused caller-list.
   [`BsMatchMakingController.go:81`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/BsMatchMakingController.go)
3. **`insertion_type`/`contact_type` default `"2"`** agar nahi diya gaya, aur wahi value
   (confusingly) Kafka payload mein `is_dual_side_added` ke naam se bhi bheji jaati hai — dekho
   §4 magic-values table.
   [`BsMatchMakingModel.go:40-46`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/BsMatchMakingModel.go)
4. **Insert order is fixed: `glid_1=supplier_id`, `glid_2=buyer_id`** — confirmed consistently
   across write, read, aur Rating-consumer code (§3).
   [`BsMatchMakingModel.go:30-33`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/BsMatchMakingModel.go)
5. **DB write pehle hoti hai, Kafka publish uske baad — dono ek hi API request mein**:
   `sp_insert_contacts` seedha `buyerprofilepg` pe synchronously chalta hai, phir Kafka topic
   `soa-user-bs_matchmaking-csl_glid_logs` pe ek notification jaati hai
   (`rservice: "USER_BS_MATCHMAKING"`). Kafka publish fail bhi ho jaaye, API response fir bhi
   `"INSERT SUCCESS"` deta hai — primary write already ho chuka hota hai; Kafka failure sirf
   `extraParamsForOutput["KAFKA_OUTPUT"]` mein logged hoti hai, response body mein nahi.
   [`BsMatchMakingModel.go:71-93`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/BsMatchMakingModel.go)
6. **Kafka-consumer-side dobara validation hoti hai** — `GLUSR_USR_ID1`/`GLUSR_USR_ID2` khaali
   ya `> 10` characters ho toh reject-and-log, chahe API-side validation pass ho chuki ho.
   [`USER_BS_MATCHMAKING_KAFKA.go:80-81`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_KAFKA.go)
7. **`SOURCE_ID == "3"` ya `"4"` par Kafka-consumer DB write skip karta hai** — sirf ek
   "already inserted by WRITE SERVICE" success-log likha jaata hai (§4).
   [`USER_BS_MATCHMAKING_KAFKA.go:83-85`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_KAFKA.go)
8. **Kafka-consumer write mein retry logic hai** — deadlock/failure pe, 500ms se badhte
   interval ke saath (upar-bound-reset 10 min pe) retry karta hai, ek global `sync.Mutex`
   (`m`) ke andar — matlab is retry-loop ke dauraan **doosre saare matchmaking messages bhi
   is consumer instance mein block ho jaate hain** kyunki mutex shared hai
   `pgInsertUserBSMatchMakingKafka` aur `pgInsertLMS` dono ke beech (same package-level `var m
   sync.Mutex` [`USER_BS_MATCHMAKING_LMS.go:17`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_LMS.go)).
   [`USER_BS_MATCHMAKING_KAFKA.go:130-150`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_KAFKA.go)
9. **`numeric field overflow` errors retry nahi hote** — turant fail hoke return, koi
   retry-loop nahi (baaki errors ke uलट).
   [`USER_BS_MATCHMAKING_KAFKA.go:126-129`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_KAFKA.go)
10. **LMS consumer ek independent write-entry-point hai, sirf mirror nahi** — yeh RabbitMQ
    queue `USER_BS_MATCHMAKING_LMS` pe seedha listen karta hai, apne field-set
    (`glusr_id1`, `glusr_id2`, `source_id`, `is_dual_side_added`) ko strict type-check
    (`reflect.TypeOf` — string for glids, float64 for source_id/is_dual_side_added) karta hai,
    aur missing/invalid ho toh message **fail-queue (`USER_BS_MATCHMAKING_LMS_FAIL`) mein
    Ack(false) ke saath drop kar deta hai** (i.e. does NOT nack/requeue on validation
    failure — only genuine DB errors get `Nack`).
    [`USER_BS_MATCHMAKING_LMS.go:63-89`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_LMS.go)
11. **LMS consumer ka dual-side flip unconditional hai** (`1→2`, else `→1`) — Kafka
    consumer ke `SOURCE_ID`-gated logic se alag, ek teesra independent implementation of the
    same "dual side" idea (§4).
    [`USER_BS_MATCHMAKING_LMS.go:104-108`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_LMS.go)
12. **Rating anti-fraud consumer sirf CDC-events ko process karta hai jinke `COLUMNS` block
    mein `FK_GLUSR_SUPPLIER_ID`/`FK_GLUSR_BUYER_ID` present aur non-empty hon** — warna
    Ack-and-drop (no retry).
    [`USER_RATING_BS_MATCHMAKING.go:63-82`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_BS_MATCHMAKING.go)
13. **Rating anti-fraud consumer apne khud ke writes ko ignore karta hai** (loop-guard via
    `CALLEDFROM` — §4).
    [`USER_RATING_BS_MATCHMAKING.go:69-71`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_BS_MATCHMAKING.go)
14. **Anti-fraud check query**: `SELECT COUNT(1) FROM glusr_contactbook_mapping WHERE
    GLUSR_USR_ID1=$supplier AND GLUSR_USR_ID2=$buyer AND coalesce(source_id,-1) IN
    (-1,1,3,4)` — agar `count == 0`, rating ko disable kar diya jaata hai (`DISPLAY_STATUS
    = -1`, `REVIEWED_BY = -999`, review-comment mein current date). Yeh `rating_write_service`
    W-API ko call karke hota hai, `CALLEDFROM: "MATCHMAKING_CONSUMER"` tag ke saath (loop-guard
    ke liye).
    [`USER_RATING_BS_MATCHMAKING.go:87-132`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_BS_MATCHMAKING.go)
15. **Agar matchmaking record already exist karta hai (`count > 0`), koi action nahi liya
    jaata** — bas success-log likha jaata hai ("Buyer and Supplier are already connected"),
    rating untouched rehti hai.
    [`USER_RATING_BS_MATCHMAKING.go:133-137`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_BS_MATCHMAKING.go)

---

## 6. Kafka

| Topic | Publisher | Consumer | Purpose |
|---|---|---|---|
| `soa-user-bs_matchmaking-csl_glid_logs` | `BsMatchMakingModel.go` (`utils.CallKafkaService`, `rservice: "USER_BS_MATCHMAKING"`) [`BsMatchMakingModel.go:71-81`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/BsMatchMakingModel.go) | [`USER_BS_MATCHMAKING_KAFKA.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_KAFKA.go) (registered `"USER_BS_MATCHMAKING": workerUserBSMatchMakingKafka` — [`IntializeMsgBroker.go:249`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go)) | Duplicate/async write of the same row into the same physical `meshpg` DB (see §3 correction) |

`CallKafkaService` internally ek HTTP proxy call hai (Kafka REST proxy pattern), native
socket-level Kafka connection nahi.

**Only one Kafka topic exists in this domain** — the LMS and Rating flows are RabbitMQ, not
Kafka (correction from prior pass — see §7).

---

## 7. RabbitMQ

**Correction to prior pass**: the earlier doc stated "no RabbitMQ usage" for this domain. On
full read of `USER_BS_MATCHMAKING_LMS.go` and `USER_RATING_BS_MATCHMAKING.go`, **both use
RabbitMQ** (`github.com/streadway/amqp`, `InitializeRabbitMq`/`InitializeRabbitMqv1`) — the
domain is actually Kafka **and** RabbitMQ, not Kafka-only.

| Queue | Direction | Handler | Purpose |
|---|---|---|---|
| `USER_BS_MATCHMAKING_LMS` | consume | `workerUserBsMatchLMS` [`USER_BS_MATCHMAKING_LMS.go:32`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_LMS.go) | Independent write path — an external producer (**[INFERRED — confirm with team]** presumed to be the Lead-Management System) pushes buyer-supplier contact events directly here; consumer validates and writes to `glusr_contactbook_mapping` via `sp_insert_contacts` |
| `USER_BS_MATCHMAKING_LMS_FAIL` | consume | same handler, `workerUserBsMatchLMS` (registered separately — [`router.go:107`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Router/router.go)) | Dead-letter/retry queue for the above — messages `Nack`'d on genuine DB error land here; messages `Ack`'d-and-dropped on validation failure do NOT reach this queue (they're just logged and discarded) |
| `USER_RATING_BS_MATCHMAKING` | consume | `dbActionUserRatingBSMatch` [`USER_RATING_BS_MATCHMAKING.go:25`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_BS_MATCHMAKING.go) | Rating-pipeline CDC feed — consumes rating-change events, checks matchmaking existence, disables fake ratings (§5 point 14) |
| `USER_RATING_BS_MATCHMAKING_FAIL` | consume | same handler (registered separately — [`router.go:108`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Router/router.go)) | Dead-letter/retry queue for the anti-fraud consumer |

No producer of `USER_RATING_BS_MATCHMAKING` was found inside this codebase — it appears to be
fed by the Rating pipeline's own CDC/replication mechanism (out of scope of this doc; see
[`../Rating KT/Rating_Technical_Doc.md`](../Rating%20KT/Rating_Technical_Doc.md)).

---

## 8. Redis

**Koi Redis usage nahi mila** in `BsMatchMakingController.go`, `BsMatchMakingModel.go`,
`USER_BS_MATCHMAKING_KAFKA.go`, `USER_BS_MATCHMAKING_LMS.go`, ya
`USER_RATING_BS_MATCHMAKING.go` — confirmed via grep for `redis`/`Redis` (case-insensitive)
against each file individually; no matches.

---

## 9. Cron Inventory

Grep for `matchmaking`/`MATCHMAKING` (case-insensitive) across the entire
`user-temp-consumers-production` repo returns only: the four consumer worker files
themselves, their router/registry entries, `config.yaml`, and two unrelated Rating-pipeline
files (`USER_RATING_SELLER_RISK.go`, `USER_RATING_ENRICHMENT.go`) that reuse the
`CALLEDFROM = "MATCHMAKING_CONSUMER"` loop-guard string convention (§4) but are not
matchmaking-table consumers themselves. **No dedicated cron job for matchmaking found** — no
`gst_tact_veri_cron`-style scheduled job exists in this domain for backfill/reconciliation.

---

## 10. End-to-End Technical Flows

### Flow A — Sync write + async mirror write (triggered by internal system, e.g. leads)

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
[DB — synchronous]  sp_insert_contacts(glid_1=supplier_id, glid_2=buyer_id, contact_type, source_id)
    │  on connection alias "buyerprofilepg" (prod host 10.90.140.5, db meshpg)
    ▼
[Kafka publish]  topic: soa-user-bs_matchmaking-csl_glid_logs
    │  (fire-and-forget — API already returns "INSERT SUCCESS" regardless of Kafka outcome)
    ▼
──────────────────────────────  <- API response sent to caller by here
[Kafka consume, consumer repo]
    │  USER_BS_MATCHMAKING_KAFKA.go — re-validates message (glid length ≤ 10)
    │  SOURCE_ID 3/4 → skip DB write (already-inserted assumption)
    │  else → recompute is_dual_side_added, retry-on-deadlock (shared mutex with LMS path)
    ▼
[DB — again, SAME physical DB alias "buyerprofilePg" → same host/db as above]
    sp_insert_contacts(...)  — a duplicate write, not a cross-DB mirror (see §3 correction)
```

### Flow B — LMS-originated write (independent RabbitMQ producer)

```
[External producer, presumably LMS] → publishes to RabbitMQ queue USER_BS_MATCHMAKING_LMS
    │
    ▼
[Consumer]  USER_BS_MATCHMAKING_LMS.go — workerUserBsMatchLMS
    │  1. Type-check glusr_id1/glusr_id2 (must be string), source_id/is_dual_side_added (must be float64)
    │  2. Presence + length (≤10) checks — failures are Ack'd-and-dropped, NOT requeued
    │  3. Unconditional dual-side flip (1→2, else→1)
    ▼
[DB]  sp_insert_contacts(glid_1, glid_2, is_dual_side, source_id) on "buyerprofilePg"
    │  on genuine DB error → Nack(false,false) → routed to USER_BS_MATCHMAKING_LMS_FAIL
    ▼
Same table (glusr_contactbook_mapping) as Flow A — independent of the write API entirely
```

### Flow C — Downstream anti-fraud consult (Rating pipeline)

```
[Rating pipeline CDC event] → RabbitMQ queue USER_RATING_BS_MATCHMAKING
    │
    ▼
[Consumer]  USER_RATING_BS_MATCHMAKING.go — dbActionUserRatingBSMatch
    │  1. Loop-guard: skip if CALLEDFROM already MATCHMAKING_CONSUMER/MCAT_CONSUMER
    │  2. Require FK_GLUSR_SUPPLIER_ID / FK_GLUSR_BUYER_ID present & non-empty
    ▼
[DB — read]  SELECT COUNT(1) FROM glusr_contactbook_mapping
             WHERE GLUSR_USR_ID1=supplier AND GLUSR_USR_ID2=buyer
             AND coalesce(source_id,-1) IN (-1,1,3,4)
    │
    ├── count > 0 → "already connected", no action, Ack
    │
    └── count == 0 → DISPLAY_STATUS=-1, REVIEWED_BY=-999,
                      comment="Rating Disabled : Buyer and Supplier Matchmaking Not Found ..."
                      → callWriteService() → rating_write_service W-API (CALLEDFROM=MATCHMAKING_CONSUMER)
```

### Flow D — Buyer profile read (this repo)

```
GET buyer profile
    │
    ▼
UserBuyerProfileModel.go:539-549
    SELECT g.source_id, COALESCE(u.block_status,0) AS block_status
    FROM ( SELECT ... FROM glusr_contactbook_mapping WHERE GLUSR_USR_ID1=$1 AND GLUSR_USR_ID2=$2 ) g
    LEFT JOIN USER_BLOCKED_STATUS u ON u.blocked_glid=GLUSR_USR_ID1 AND u.user_glid=GLUSR_USR_ID2
    │  single round-trip, both "connected?" and "blocked?" signals returned together
    ▼
See also: ../Blocking KT/Blocking_Technical_Doc.md — same model file, enforcement logic there
```

---

## 11. Flow-wise DB & Table Usage Matrix

### Flow A — Sync write + Kafka async mirror

| # | DB (alias) | Table | Operation | Why |
|---|---|---|---|---|
| A1 | `buyerprofilepg` (service-api-go) | `GLUSR_CONTACTBOOK_MAPPING` (via `sp_insert_contacts`) | INSERT (stored proc) | Primary, synchronous record of the connection — API blocks on this |
| A2 | Kafka topic `soa-user-bs_matchmaking-csl_glid_logs` | n/a | Publish | Async fan-out trigger for the mirror write |
| A3 | `buyerprofilePg` (consumers, same physical DB as A1 — §3) | `GLUSR_CONTACTBOOK_MAPPING` (via `sp_insert_contacts`) | INSERT (stored proc, retried on deadlock) | Duplicate write of the same row — redundant with A1 in prod (see Optimization Scope) |

### Flow B — LMS write

| # | DB (alias) | Table | Operation | Why |
|---|---|---|---|---|
| B1 | `buyerprofilePg` (consumers) | `GLUSR_CONTACTBOOK_MAPPING` (via `sp_insert_contacts`) | INSERT (stored proc, retried on deadlock) | Independent record of a contact-event sourced from LMS, not from the write API |

### Flow C — Rating anti-fraud check

| # | DB (alias) | Table | Operation | Why |
|---|---|---|---|---|
| C1 | `buyerprofilePg` (consumers) | `glusr_contactbook_mapping` | SELECT COUNT(1) | Determine whether the rated buyer-supplier pair has a genuine matchmaking record |
| C2 | `rating_write_service` (external W-API, not a direct DB call from this consumer) | rating table (out of scope — see Rating KT doc) | UPDATE (via API) | Disable/hide the rating if no matchmaking record found |

### Flow D — Buyer profile read

| # | DB (alias) | Table | Operation | Why |
|---|---|---|---|---|
| D1 | (this repo's) buyer-profile DB connection | `glusr_contactbook_mapping` LEFT JOIN `USER_BLOCKED_STATUS` | SELECT | Combine "connected?" and "blocked?" signals in a single round-trip for the supplier viewing a buyer's profile |

---

## 12. Optimization Scope — DB / Response-Time Contributors

### High-impact

1. **The async "mirror" write (Flow A, step A3) writes to the same physical database as the
   synchronous write (A1)** — confirmed by prod config comparison (§3). This means every
   matchmaking event costs **two INSERTs into the same table on the same DB instance**
   (one sync, one async via Kafka round-trip), not a genuine cross-DB replication. If this
   duplication is not intentional (e.g. for a different downstream system's benefit that
   isn't evident from code), it's pure extra write load. **[INFERRED — confirm with team]**
   why this duplicate write exists — could not be found documented in code/comments.
2. **Kafka-consumer's deadlock retry holds a package-level `sync.Mutex` (`m`) shared with the
   LMS consumer's insert path** [`USER_BS_MATCHMAKING_KAFKA.go:130`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_KAFKA.go),
   [`USER_BS_MATCHMAKING_LMS.go:17`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_LMS.go) —
   a slow retry-loop in one path (up to effectively unbounded retries, since the 10-minute
   check only resets the backoff counter rather than giving up) can stall the other path's
   inserts too, since both hold the same mutex during their respective retry loops.

### Medium-impact

3. **`GLUSR_USR_ID1`/`GLUSR_USR_ID2` length validation is hardcoded to `> 10` characters** in
   three separate places (write model implicitly via `BSMatchMakingMap`, Kafka consumer
   [`USER_BS_MATCHMAKING_KAFKA.go:80`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_KAFKA.go),
   LMS consumer [`USER_BS_MATCHMAKING_LMS.go:75`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BS_MATCHMAKING_LMS.go)) —
   if GLID format ever changes, all three need updating; a shared constant would remove this
   drift risk.
4. **Three independent implementations of the "dual-side flip" rule** (write model, Kafka
   consumer, LMS consumer — §4/§5) with subtly different conditions (gated vs unconditional) —
   maintenance risk more than a runtime cost, but worth consolidating.

### Low-impact / good practice already present

5. **Anti-fraud check is a single indexed-looking `COUNT(1)` with an equality predicate on
   both GLID columns** — cheap, well-scoped read.
6. **Buyer-profile read (Flow D) combines matchmaking + blocking in one query** — avoids a
   second round-trip.
7. **`numeric field overflow` short-circuits without retrying** (§5 point 9) — avoids wasting
   retry cycles on an error that will never resolve itself.

---

## 13. Edge Cases & Gotchas (technical POV)

1. **Duplicate-write assumption needs confirmation** — §12 point 1; the "mirror to a
   different DB" narrative from the prior doc pass does not hold up against prod config; it
   looks like a genuine duplicate write to the same table.
2. **`contact_type`/`is_dual_side_added` naming collision on the sync-write side** (§4) — the
   same variable feeds both a stored-proc positional param labelled `contact_type` and a Kafka
   payload field labelled `is_dual_side_added`; whether these are meant to be the same concept
   or this is a latent bug is unclear from code alone.
3. **Three different dual-side-flip implementations can persist different values for the same
   logical event** depending on which path (sync, Kafka-async, LMS) ultimately performs the
   write that "wins" — no guarantee code paths converge on the same interpretation.
4. **LMS consumer drops invalid messages permanently (`Ack` on validation failure)** — unlike
   DB-error cases (which `Nack` to the fail queue), a malformed LMS message is logged and lost,
   no retry path.
5. **No reconciliation/backfill cron found** for drift between the sync write and the async
   mirror write, unlike GST domain's `gst_tact_veri_cron` safety net.
6. **`SOURCE_ID` 3/4 special-casing in the Kafka consumer assumes some other write path already
   inserted the row** — if that assumption is ever wrong (e.g. that other path changes
   behavior), matchmaking records for these sources would silently never get written via this
   consumer.

---

## 14. Open Questions

1. Why does the async Kafka-consumer path duplicate the synchronous write into the *same*
   physical database (§3, §12 point 1) — was this originally meant to be a genuinely different
   DB (e.g. the `meshPg` alias that exists but is unused by these workers), and did a config
   change collapse them onto the same instance without the code being revisited?
2. `insertion_type`/`contact_type`/`is_dual_side_added` — is this genuinely one concept wearing
   two names, or a bug where two different concepts got merged into one variable
   (`BsMatchMakingModel.go:37,42`)?
3. Exact business identity of `SOURCE_ID` values `1`, `3`, `4` (and the "LEADS" allowlist that
   gates the write API) — **[INFERRED — confirm with team]**.
4. Who is the actual producer of the `USER_BS_MATCHMAKING_LMS` RabbitMQ queue — confirmed to be
   an external/independent producer from code shape, but the producer's own repo was not part
   of this KT scope.
5. Who produces `USER_RATING_BS_MATCHMAKING` — appears to be the Rating pipeline's own CDC
   mechanism (Debezium-style), but the exact producer was not traced in this pass.
6. Live DB schema verification — this doc reflects only what the Go SQL strings imply.

---

## See also

- [`Matchmaking_Business_Doc.md`](./Matchmaking_Business_Doc.md) — product perspective
- [`../Blocking KT/Blocking_Technical_Doc.md`](../Blocking%20KT/Blocking_Technical_Doc.md) —
  related independent module, same read-query intersection point (`UserBuyerProfileModel.go`)
- [`../Rating KT/Rating_Technical_Doc.md`](../Rating%20KT/Rating_Technical_Doc.md) — where
  `USER_RATING_BS_MATCHMAKING` consumes this data for anti-fraud checks
- [`../buyer_seller_discovery_matching_read_write_picture.md`](../buyer_seller_discovery_matching_read_write_picture.md) —
  original architecture trace this doc builds on

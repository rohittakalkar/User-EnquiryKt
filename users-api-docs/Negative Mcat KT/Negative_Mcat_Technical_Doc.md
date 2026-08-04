# Negative Mcat — Technical Doc (Code-Level Deep Dive)

Yeh doc Negative Mcat feature ka **technical implementation** cover karta hai — APIs, DB
tables, queries, RabbitMQ, consumers, sab kuch code se verify karke. Business/product
perspective ke liye [`Negative_Mcat_Business_Doc.md`](./Negative_Mcat_Business_Doc.md) dekho —
dono docs same flows cover karte hain, bas alag audience ke liye.

**Repos**: `service-api-go-production` (write), `users-api-go-production` (read),
`user-temp-consumers-production` (3-way fan-out consumers). Koi dedicated cron nahi mila iss
feature ke liye (section 12).

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file path + line
number diya gaya hai har jagah). Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — team se confirm karo]** likh diya hai, guess nahi kiya.

---

## 1. Negative Mcat Kahan-Kahan Hai — File Map

| Concern | Repo | File |
|---|---|---|
| Write API (insert/soft-delete) | write | [`UserNegMcatController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserNegMcatController.go) |
| Write DB logic | write | [`UserNegMcatModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserNegMcatModel.go) (`NegMcatUpdateintoDB`) |
| Write — mandatory-field validation | write | [`UserUtilsMandatory.go:1390-1419`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go) (`MandatoryFieldsNegMcat`) |
| Write — type/length validation | write | [`UsersValidationMaps.go:2222-2224`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) (`ValidationNegMcat`) |
| Write — column-name/type map | write | [`UsersValidationMaps.go:375-396`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) (`UserNegMcatMap`) |
| Read API | read | [`NegativeMcatController.go`](../../users-api-go-production/internal/controllers/UsersControllers/NegativeMcatController.go) (`NegativeMcat`) |
| Read DB logic | read | [`UserNegativeMcatModel.go`](../../users-api-go-production/internal/models/users/UserNegativeMcatModel.go) (`GetNegativeMcat`, `getNegativeMcatforGlusr`, `getNegativeMcatforItems`) |
| Fan-out — search/directory replica | consumers | [`USER_NEG_MCAT_IMSDB.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_NEG_MCAT_IMSDB.go) (`dbActionUserNegMcatImsdb`, writes `searchPg`) |
| Fan-out — alert/blacklist DB | consumers | [`USER_NEG_MCAT_IMBLPG.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_NEG_MCAT_IMBLPG.go) (`dbActionUserNegMcatIMBLPg`, writes `alertPg`) |
| Fan-out — forward to another queue | consumers | [`USER_NEG_MCAT_MLBL.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_NEG_MCAT_MLBL.go) (`dbActionUserNegMcatMLBL`) — **koi local DB write nahi**, seedha `PubAPI` se `ETO_REJECTION_MASTER_QUEUE` pe republish karta hai |
| Consumer registry (queue-name → handler-function map) | consumers | [`IntializeMsgBroker.go:31,46,60`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go) |
| Route registration (write) | write | [`router.go:155,323`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) |
| Route registration (read) | read | [`routerUsers.go:301-302,510-511,670-671`](../internal/api/users_router/routerUsers.go) — teen jagah registered hai (environment/version-specific router blocks lagte hain) |

---

## 2. Routes

| Method | Path/serviceName | Repo | Controller |
|---|---|---|---|
| POST | `/negative/mcat` (serviceName `NEGATIVE_MCAT_SERVICE`) | write | [`UserNegMcatController`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserNegMcatController.go) — [`router.go:155,323`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) |
| GET/POST | `negativemcat/*params` (serviceName `NEGATIVE_MCAT`) | read | [`NegativeMcat`](../internal/controllers/UsersControllers/NegativeMcatController.go) — [`routerUsers.go:301-302`](../internal/api/users_router/routerUsers.go) etc. |

---

## 3. Data Model — Tables

> **Verification note**: yeh sab table/column names Go code ke andar embedded SQL strings se
> liye gaye hain (har row ke saamne file path diya hai). Live DB schema se pgAdmin pe
> cross-verify **nahi** kiya gaya hai — kisi migration/integration se pehle woh zaroor karo.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `NEGATIVE_MCAT_FOR_PRODUCTS` | meshpg (write) / mesh_pg_user (read) / **searchPg** (fan-out replica) | **Primary supplier/product-level mcat-exclusion record** | `GLCAT_MCAT_NEGATIVE_ID` (PK, `RETURNING` on insert), `FK_GLUSR_USR_ID`, `FK_PC_ITEM_ID`, `FK_GLCAT_MCAT_ID`, `EMPID`, `IS_DELETED`, `NEGATIVE_MCAT_UPDATED_DATE`, `ADDED_DATE` — [`UserNegMcatModel.go:102-207`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserNegMcatModel.go), [`USER_NEG_MCAT_IMSDB.go:75-79`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_NEG_MCAT_IMSDB.go) |
| `GLCAT_MCAT` | mesh_pg_user (read) | Mcat master table — human-readable `glcat_mcat_name` join ke liye | `GLCAT_MCAT_ID`, `GLCAT_MCAT_NAME` — [`UserNegativeMcatModel.go:142,200`](../internal/models/users/UserNegativeMcatModel.go) |
| **`NEGATIVE_MCAT_FOR_PRODUCTS_CURRENT`** | **alertPg** | Alert/blacklist system ka **apna alag table**, primary table se **naam alag hai** aur granularity bhi alag hai — sirf `(FK_GLUSR_USR_ID, FK_GLCAT_MCAT_MCAT_ID)` pe track karta hai, `item_id` iss table mein hai hi nahi | `FK_GLUSR_USR_ID`, `FK_GLCAT_MCAT_MCAT_ID`, `NEGATIVE_MCAT_IS_DELETED`, `ADDED_DATE`, `NEGATIVE_MCAT_DELETED_DATE`, `NEGATIVE_MCAT_ADDED_BY` — [`USER_NEG_MCAT_IMBLPG.go:70-84`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_NEG_MCAT_IMBLPG.go) |

**Important finding**: original doc ne sirf "writes alertPg" bola tha — actual table naam alag
hai (`NEGATIVE_MCAT_FOR_PRODUCTS_CURRENT`, not `NEGATIVE_MCAT_FOR_PRODUCTS`), **aur** iska
matching-key bhi alag hai — sirf glusr+mcat, item-level granularity iss replica mein collapse
ho jaati hai. Yeh do independently-notable facts hain, dono section 11 ke DB matrix mein
detail se cover hue hain.

---

## 4. Business Rules & Validation (code se exhaustive list)

1. **Gateway allowlist bahut bada hai** — `MY`, `GLADMIN`, `TOLLFREE`, `Email Marketing`,
   `HTVENDOR`, `Weberp`, `LEAP`, `IPC-Admin`, `NSD Sales`, `MAPI`, `ENQUIRY`, `FREE-WEBSITE`,
   `M.INDIAMART.COM`, `TRADE`, `BL`, `TENDER`, `PAYNOW`, `CREDIT ALLOCATION`, `OVP Process`,
   `SAMPARK Process`, `TOLLFREE Process`, `VENDOR CITY Pin Correction`, `Notification
   Server`, `PCAT-ADMIN`, `TOLEXO`, `IMOB`, `Merp` — 26 internal callers allowed hain, matlab
   yeh ek widely-used internal-tool feature hai.
   [`UserNegMcatController.go:99-101`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserNegMcatController.go)
2. **Mandatory fields sirf tab strictly enforce hote hain jab `action_flag` `I` ya `D` ho**:
   `item_id`, `UPDATED_BY`, `action_flag`, `mcat_id`, `GLUSR_USR_ID`. Agar `action_flag`
   in dono ke alawa kuch aur ho, yeh check bypass ho jaata hai lekin aage `action_flag`
   validation (rule 3) usse catch kar leti hai.
   [`UserUtilsMandatory.go:1390-1417`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
3. **`action_flag` sirf `I` ya `D` accept karta hai**, koi aur value ho toh explicit reject:
   `"MCAT_FLAG is not appropriate"`.
   [`UserUtilsMandatory.go:1415-1417`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
4. **`item_id` optional at request level, `-1` default ho jaata hai agar absent/empty**
   (matlab supplier-level/account-wide negative-mcat) — controller **do jagah** yeh default
   apply karta hai (once for validation, once before DB call), aur agar `item_id` explicitly
   empty-string diya jaaye toh alag ek "ITEM_NOTPRESENT" error path bhi hai read-side pe.
   [`UserNegMcatController.go:71-76,139-142`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserNegMcatController.go)
5. **`empid` presence `UPDATED_BY` resolution decide karta hai**: `empid == "-1"` (jo default
   set hota hai agar absent) → `UPDATED_BY = "User"`; warna `Employee_mesh_pg()` DB lookup se
   naam resolve hota hai aur `"<Name> (<empid>)"` format mein store hota hai — agar lookup
   khaali return kare toh caller-supplied `UPDATEDBY` fallback use hota hai.
   [`UserNegMcatController.go:93-95,144-168`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserNegMcatController.go)
6. **Insert (`action_flag=I`)**: `IS_DELETED='0'` hardcoded, `NEGATIVE_MCAT_UPDATED_DATE` aur
   HTTP-supplied `added_date` dono `CURRENT_TIMESTAMP` se overwrite ho jaate hain query mein —
   matlab `inputParams["added_date"]` jo controller format karke bhejta hai (`YYYYMMDDHHMMSS`
   string), woh actually query mein use hi nahi hota, `CURRENT_TIMESTAMP` literal hardcoded
   hai iske jagah. **[Minor code-hygiene finding]**: yeh formatted value ek dead computation
   hai.
   [`UserNegMcatModel.go:96-97,114-115,133`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserNegMcatModel.go)
7. **Delete (`action_flag` != `I`, effectively `D`)**: soft-delete via **UPDATE**, `IS_DELETED
   = 1`, matched on `(fk_glusr_usr_id, fk_pc_item_id, fk_glcat_mcat_id)` triple — **primary
   key se nahi**, matlab record physically retain hota hai, delete match ho toh purana row hi
   flip hota hai.
   [`UserNegMcatModel.go:160-206`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserNegMcatModel.go)
8. **Insert/update ke liye column-list dynamically build hoti hai** `UserNegMcatMap`
   whitelist ke against — iske through extra optional fields (`IP`, `IP_COUNTRY`, `MODULE`,
   `UPDATE_URL`, `HIST_COMMENT`, `COMMENTS`, `UPDATEDBY_AGENCY`, `UPDATEDUSING`) bhi
   dynamically insert/update ho sakte hain agar request mein present hon.
   [`UsersValidationMaps.go:375-396`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go)
9. **`HIST_COMMENT`/`COMMENTS` defaulting**: agar `HIST_COMMENT` absent ho, controller khaali
   string set kar deta hai; agar `COMMENTS` absent ho, `HIST_COMMENT` ki value copy ho jaati
   hai.
   [`UserNegMcatController.go:183-189`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserNegMcatController.go)
10. **Read-side sirf `IS_DELETED=0` filter karta hai**, koi de-dup nahi, `GLCAT_MCAT` join
    karke `mcat_name` attach karta hai.
    [`UserNegativeMcatModel.go:142,200`](../internal/models/users/UserNegativeMcatModel.go)
11. **[BUG-shaped finding] Read-side comma-separated `item_id` list effectively kaam nahi
    karta.** `GetNegativeMcat()` mein ek explicit branch hai jab `item_id` mein comma ho
    (line 98: `strings.Contains(itemid, ",") != false`), jo `getNegativeMcatforItems()` ko
    same tarah call karta hai jaise single-item case mein. Lekin `getNegativeMcatforItems()`
    ke andar khud ek guard hai (line 196: `strings.Contains(itemid, ",") == false`) jo sirf
    tab query banata hai jab comma **na** ho — comma-wale case mein function silently
    `itemmcats=[]`, `itemmcatstr=""`, `err=nil` return kar deta hai, **bina IN(...) query
    banaye**. Matlab agar koi caller `item_id=101,102,103` bheje, response hamesha empty
    aayega, code ka comma-branch hone ke bawajood — yeh ek dead/broken code path hai jo
    multi-item lookup intend karta hai lekin implement nahi karta.
    [`UserNegativeMcatModel.go:98-109,189-250`](../internal/models/users/UserNegativeMcatModel.go)

---

## 5. RabbitMQ

Negative Mcat ki poori messaging **RabbitMQ-based** hai, `PushToQueue` (publish) aur
`PubAPI` (re-publish/forward) helpers ke through.

| Queue / `SERVICENAME` | Publisher | Consumer(s) | Purpose |
|---|---|---|---|
| `USER_NEGATIVE_MCAT` → routes to `user.neg.*` on `USER.topic` exchange (`serviceToQueueMap`) | [`UserNegMcatModel.go:239,257`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserNegMcatModel.go) (`NegMcatUpdateintoDB`) — **sirf INSERT/UPDATE success pe fire hota hai** ([`UserNegMcatModel.go:209`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserNegMcatModel.go)) | 3-way fan-out (neeche) | Insert/delete event ko downstream replicas + ek external pipeline tak pahunchana |
| — | — | [`USER_NEG_MCAT_IMSDB.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_NEG_MCAT_IMSDB.go) → writes `searchPg` | Search/directory replica ko naya negative-mcat state sync karta hai |
| — | — | [`USER_NEG_MCAT_IMBLPG.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_NEG_MCAT_IMBLPG.go) → writes `alertPg` (`NEGATIVE_MCAT_FOR_PRODUCTS_CURRENT`, glusr+mcat granularity — section 3) | Alert/blacklist system apna copy sync karta hai |
| — | — | [`USER_NEG_MCAT_MLBL.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_NEG_MCAT_MLBL.go) → **no DB write**, re-publishes via `utils.PubAPI` into `ETO_REJECTION_MASTER_QUEUE` ([`USER_NEG_MCAT_MLBL.go:56-60`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_NEG_MCAT_MLBL.go)) | Same negative-mcat event ek **doosri pipeline** ko forward karta hai — "ETO" ka full-meaning ya downstream consumer iss teen-repo review se trace nahi ho paya (section 13, Open Questions) |

**Registry confirmation**: teeno consumer functions `IntializeMsgBroker.go` mein alag-alag
queue-name key ke saath registered hain — `USER_NEG_MCAT_IMSDB` (line 31), `USER_NEG_MCAT_IMBLPG`
(line 46), `USER_NEG_MCAT_MLBL` (line 60) — matlab yeh teeno independent RabbitMQ queues hain
(3 alag consumer processes/bindings), sab ek hi upstream `USER_NEGATIVE_MCAT` publish se feed
hote hain (via routing-key fan-out on `user.neg.*` / `USER.topic`, exact per-queue binding
config in queue-binding files, iss code review ke scope se bahar).

---

## 6. Kafka

**Koi Kafka usage nahi mila** negative-mcat ke kisi bhi write/read/consumer file mein —
explicitly grep kiya gaya `UserNegMcatController.go`, `UserNegMcatModel.go`,
`UserNegativeMcatModel.go`, aur teeno consumer files mein, zero matches. Poora domain
RabbitMQ-only hai.

---

## 7. Redis

**Koi Redis usage nahi mila** iss feature ke kisi bhi file mein (same grep sweep, section 6).
Read endpoint (`GET negativemcat/*params`) har request pe seedha Postgres (`mesh_pg_user`)
hit karta hai — koi caching layer nahi hai (dekho section 10, Optimization Scope).

---

## 8. End-to-End Technical Flows

### Flow A — Naya negative-mcat insert (`action_flag=I`)

```
Internal-tool/Admin (GLADMIN/WebERP/Tollfree/PCAT-Admin/...)
    │
    ▼
[API — write]  POST /negative/mcat  {GLUSR_USR_ID, item_id?, mcat_id, action_flag=I, empid?, VALIDATION_KEY}
    │  UserNegMcatController.go
    │  1. Gateway allowlist check (26 services)
    │  2. item_id default "-1" agar absent
    │  3. empid=="-1"? UPDATED_BY="User" : Employee_mesh_pg() lookup [conditional DB round-trip]
    │  4. MandatoryFieldsNegMcat() → ValidationNegMcat() (length/type/numeric checks)
    ▼
NegMcatUpdateintoDB()
    │  INSERT INTO NEGATIVE_MCAT_FOR_PRODUCTS (...) VALUES (...) RETURNING GLCAT_MCAT_NEGATIVE_ID
    ▼
[DB — meshpg, synchronous]
    │  success pe:
    ▼
[RabbitMQ publish]  SERVICENAME=USER_NEGATIVE_MCAT → user.neg.* (USER.topic)
    ▼
[CONSUME — 3-way parallel fan-out, dekho section 5]
    ├─ USER_NEG_MCAT_IMSDB   → INSERT INTO NEGATIVE_MCAT_FOR_PRODUCTS (searchPg), explicit PK = MCAT_NEG_ID
    ├─ USER_NEG_MCAT_IMBLPG  → UPDATE NEGATIVE_MCAT_FOR_PRODUCTS_CURRENT (alertPg, glusr+mcat match);
    │                           agar 0 rows affected → conditional INSERT (upsert-by-hand pattern)
    └─ USER_NEG_MCAT_MLBL    → forwards via PubAPI to ETO_REJECTION_MASTER_QUEUE (no local DB write)
```

### Flow B — Existing negative-mcat soft-delete (`action_flag=D`)

```
Internal-tool/Admin
    │
    ▼
[API — write]  POST /negative/mcat  {GLUSR_USR_ID, item_id, mcat_id, action_flag=D, empid?, VALIDATION_KEY}
    │  same gateway/mandatory/validation steps as Flow A
    ▼
NegMcatUpdateintoDB()
    │  UPDATE NEGATIVE_MCAT_FOR_PRODUCTS SET IS_DELETED=1, ... WHERE (glusr,item,mcat) match
    ▼
[DB — meshpg]
    │  success pe: same RabbitMQ publish + 3-way fan-out as Flow A, with ACTION="D"
    ├─ USER_NEG_MCAT_IMSDB   → UPDATE (same triple match, searchPg)
    ├─ USER_NEG_MCAT_IMBLPG  → UPDATE NEGATIVE_MCAT_FOR_PRODUCTS_CURRENT SET IS_DELETED=1 (glusr+mcat match, alertPg);
    │                           conditional INSERT agar 0 rows affected
    └─ USER_NEG_MCAT_MLBL    → forwards via PubAPI to ETO_REJECTION_MASTER_QUEUE
```

### Flow C — Negative-mcat read

```
Any caller (glusrid required, item_id optional/comma-list-in-theory)
    │
    ▼
[API — read]  GET/POST negativemcat/*params  {glusrid, item_id?}
    │  NegativeMcatController.go → NegativeMcat()
    │  glusrid mandatory-check; item_id empty-string explicit-check
    ▼
GetNegativeMcat()
    ├─ getNegativeMcatforGlusr()  → SELECT ... JOIN GLCAT_MCAT WHERE fk_glusr_usr_id=$1 AND is_deleted=0
    └─ getNegativeMcatforItems()  → SELECT ... JOIN GLCAT_MCAT WHERE fk_pc_item_id=$1 AND is_deleted=0
                                     (sirf agar item_id present AND single value — comma-list
                                      case silently no-ops, section 4 rule 11)
    ▼
Response {Negative_mcats_for_gluser, Negative_mcats_for_items}
```

---

## 9. Flow-wise DB & Table Usage — Kaun sa DB, Kaun sa Table, Kis Liye

### Flow A/B — Insert/Delete (write, 1-2 DB round-trips + 3-way async fan-out)

| # | DB (physical) | Table | Operation | Kya nikala/likha jaata hai, aur kyun |
|---|---|---|---|---|
| 1 | meshpg | *(Employee master, exact table via `Employee_mesh_pg`)* | SELECT | **Conditional** — sirf jab `empid != "-1"`. Human-readable employee-name resolve karta hai `UPDATED_BY` field ke liye |
| 2 | meshpg | `NEGATIVE_MCAT_FOR_PRODUCTS` | INSERT (`action_flag=I`, `RETURNING GLCAT_MCAT_NEGATIVE_ID`) ya UPDATE (`action_flag=D`, `IS_DELETED=1`) | Primary write — supplier/product ke against ek mcat ko exclude/un-exclude mark karta hai |

**Async fan-out (RabbitMQ consume, alag processes)**:

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 3 | **searchPg** | `NEGATIVE_MCAT_FOR_PRODUCTS` (same name, alag physical DB) | INSERT (explicit PK = `MCAT_NEG_ID` jo primary insert ne return kiya tha) ya UPDATE | Search/directory-facing replica ko sync rakhta hai, taaki listing-exclusion search results mein bhi reflect ho |
| 4 | **alertPg** | `NEGATIVE_MCAT_FOR_PRODUCTS_CURRENT` (naam alag, granularity alag — section 3) | UPDATE (glusr+mcat match) → agar 0 rows affected, conditional INSERT | Alert/blacklist system apna khud ka snapshot maintain karta hai, item-level granularity ke bina |
| 5 | *(none — pure message relay)* | — | `PubAPI` re-publish → `ETO_REJECTION_MASTER_QUEUE` | Same event ko ek alag rejection-processing pipeline mein forward karta hai; iss pipeline ka apna DB/logic in teen repos ke bahar hai |

**Total DB round-trips ek single write event ke liye: kam se kam 4** (1 conditional employee
lookup + 1 primary write + 2 replica writes), plus 1 pure message-forward jiska apna DB cost
in repos ke bahar hai.

### Flow C — Read

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | mesh_pg_user | `NEGATIVE_MCAT_FOR_PRODUCTS` JOIN `GLCAT_MCAT` | SELECT, `WHERE fk_glusr_usr_id=$1 AND is_deleted=0` | Supplier-level (account-wide) negative-mcats fetch karta hai, mcat-name ke saath |
| 2 | mesh_pg_user | `NEGATIVE_MCAT_FOR_PRODUCTS` JOIN `GLCAT_MCAT` | SELECT (conditional, agar single non-comma `item_id` diya ho), `WHERE fk_pc_item_id=$1 AND is_deleted=0` | Item-specific negative-mcats fetch karta hai |

---

## 10. Optimization Scope — DB Response-Time Contribution

### Medium-impact

1. **Read-query timeout bahut tight hai — 100ms.** `getNegativeMcatforGlusr` aur
   `getNegativeMcatforItems` dono `GetDataSqlAndReturnArrayContext(..., 100*time.Millisecond)`
   use karte hain — agar DB thoda bhi slow ho (connection contention, index scan cost),
   query timeout hit karegi aur poora request fail hoga. Yeh ek unusually aggressive
   application-level timeout hai compared to typical multi-second timeouts elsewhere.
   [`UserNegativeMcatModel.go:151,207`](../internal/models/users/UserNegativeMcatModel.go)
2. **`Employee_mesh_pg` lookup ek extra sequential DB round-trip hai** jab bhi `empid`
   present ho, main insert/update se pehle — same pattern already flagged in Logo/GST KT
   docs; Redis-cache for employee-name-resolution ek repeatable cross-module suggestion hai.
   [`UserNegMcatController.go:145-156`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserNegMcatController.go)
3. **Koi caching read endpoint pe nahi hai** (section 7) — har `negativemcat` GET call live
   Postgres hit karta hai. Negative-mcat records low-churn hote hain (admin-driven, supplier
   khud nahi badalta), isliye yeh ek reasonable caching candidate hai agar read-volume kabhi
   significant ho jaaye.

### Low impact

4. **3-way consumer fan-out** — standard replication-cost, koi obvious inefficiency nahi;
   `MLBL` forward-only consumer architecturally interesting hai lekin DB-performance
   concern nahi.
5. **Insert/update column-building via `range` over a map + whitelist-lookup
   (`UserNegMcatMap`)** — per-request map-iteration overhead negligible hai iss scale pe.
6. **`added_date` computation dead code hai** (section 4, rule 6) — `time.Now().Format(...)`
   call hoti hai har insert pe, lekin query usse discard karke `CURRENT_TIMESTAMP` literal
   use karti hai. Negligible CPU cost, but worth cleaning up for code clarity.

### Logging/observability finding (not a DB-performance issue, but worth flagging)

7. **`QUERY_EXEC_TIME` metric hamesha `~0` report hota hai read-side pe.** `GetNegativeMcat()`
   mein `query_exec_start_time` aur `query_exec_end_time` declare hote hain
   (`var query_exec_start_time, query_exec_end_time time.Time`) lekin **kabhi assign nahi
   hote** — dono zero-value `time.Time{}` reh jaate hain jab tak `result["QUERY_EXEC_TIME"] =
   query_exec_end_time.Sub(query_exec_start_time)...` compute hota hai. Matlab is field ka
   logged/Kibana value hamesha ek fixed (zero-diff) number hoga, actual query-time nahi —
   agar koi is metric ko dashboard/alerting ke liye use kar raha hai, woh currently meaningless
   data dekh raha hai.
   [`UserNegativeMcatModel.go:52,120,130`](../internal/models/users/UserNegativeMcatModel.go)

---

## 11. Full Flow Diagram (Lucid, icon-based)

**[Poora Negative Mcat flowchart yahan dekho](https://lucid.app/lucidchart/3f3741ae-a07d-47dd-b9f7-59b7d39d7573/edit)**

---

## 12. Cron Inventory

**Koi cron nahi mila** iss feature ke liye — `user-temp-consumers-production` repo ke crons
directory aur shell-script wrappers dono explicitly grep kiye gaye `NEG_MCAT`/`NegMcat` ke
liye, zero matches. Negative Mcat poori tarah request-driven (write API) + async-consumer-driven
(3-way fan-out) hai, koi scheduled/batch job nahi.

---

## 13. Edge Cases & Gotchas (technical POV)

1. **Do "Negative Mcat tables" alag naam aur alag granularity ke saath hain** — primary
   (`NEGATIVE_MCAT_FOR_PRODUCTS`, glusr+item+mcat level) vs alert-replica
   (`NEGATIVE_MCAT_FOR_PRODUCTS_CURRENT`, sirf glusr+mcat level, item_id iss table mein hai hi
   nahi). Agar do alag `item_id` ke against same glusr+mcat combination ke liye insert/delete
   ho, alertPg replica in dono ko ek hi row treat karega — item-level granularity yahan collapse
   ho jaati hai. Team ko confirm karna chahiye ki yeh intentional design hai.
2. **Comma-separated `item_id` read-query effectively broken hai** (section 4, rule 11) —
   code mein branch hai jo iska intent dikhata hai, lekin implementation missing hai.
3. **Soft-delete's `WHERE` clause matches on (glusr, item, mcat), primary key se nahi** — agar
   duplicate active rows kabhi exist karein same triple ke liye (koi unique-constraint code se
   confirm nahi hui), ek single delete-call multiple rows affect kar sakta hai.
4. **`USER_NEG_MCAT_MLBL`'s downstream `ETO_REJECTION_MASTER_QUEUE` pipeline in teen repos ke
   bahar hai** — us queue ka consumer, aur "ETO" ka actual meaning, iss codebase se trace nahi
   ho paya.
5. **Read-side `QUERY_EXEC_TIME` metric meaningless hai** (section 10, point 7) — agar koi
   iss metric pe alert/dashboard bana raha hai, woh galat signal dekh raha hoga.
6. **Read-query 100ms timeout** (section 10, point 1) — production mein agar DB thoda bhi
   slow ho, yeh timeout normal load ke under bhi trip ho sakta hai.

---

## 14. Open Questions

Yeh cheezein sirf code se conclusively answer nahi ho paayin, finalize karne se pehle confirm
karo:

1. `ETO_REJECTION_MASTER_QUEUE` (fed by `USER_NEG_MCAT_MLBL`) — kaun sa system isse consume
   karta hai, aur "ETO" kis business-process ko represent karta hai?
2. Composite index `(fk_glusr_usr_id, fk_pc_item_id, fk_glcat_mcat_id)` `NEGATIVE_MCAT_FOR_PRODUCTS`
   pe exist karta hai kya — confirm karo duplicate-prevention aur delete-performance dono ke
   liye.
3. `NEGATIVE_MCAT_FOR_PRODUCTS_CURRENT` (alertPg) ki item-level granularity na hona —
   intentional design hai ya ek historical simplification jo kabhi fix nahi hui?
4. Comma-separated multi-`item_id` read support — kya yeh kabhi kaam karta tha (regression)
   ya shuru se hi incomplete tha?
5. 100ms read-query timeout — kya yeh deliberately tight rakha gaya hai (fail-fast pattern) ya
   ek default value jo kabhi tune nahi hui?
6. Har table (section 3) ka live DB schema verification (column types, nullability, indexes,
   constraints) — yeh doc sirf Go SQL strings jo imply karti hain wahi reflect karta hai.

---

## See also

- [`Negative_Mcat_Business_Doc.md`](./Negative_Mcat_Business_Doc.md) — same flows, product/business
  perspective, bina code ke
- [`../Logo KT/Logo_Technical_Doc.md`](../Logo%20KT/Logo_Technical_Doc.md) — similar
  `Employee_mesh_pg` name-resolution pattern precedent
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — depth/structure
  reference doc iss rewrite ke liye

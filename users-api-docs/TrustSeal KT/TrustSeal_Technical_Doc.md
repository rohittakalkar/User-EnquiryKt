# TrustSeal (Seal, Company, Misc & Trade Affiliations) — Technical Doc

Business/product perspective ke liye [`TrustSeal_Business_Doc.md`](./TrustSeal_Business_Doc.md)
dekho — dono docs same flows cover karte hain, alag audience ke liye.

**Repos**: `service-api-go-production` (write), `users-api-go-production` (read).
`user-temp-consumers-production` grep kiya gaya hai TrustSeal-specific consumer/cron ke liye —
**kuch nahi mila** (section 6, 12).

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file:line diya gaya
hai). Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — team se confirm karo]** likh diya hai, guess nahi kiya.

**Scope decision (confirmed via code investigation)**: TrustSeal, TrustSeal Company, TS Misc
Details (Trade Affiliations/Std Quality/Safety Cert/Chamber/Export Status), TS Basic-read — sab
ek hi table-family hai (`TRUSTSEAL_ID`/`FK_TRUSTSEAL_ID` se joined) aur likhne ke liye ek hi
write-controller (`UserTrustsealController`) use karte hain. Isliye ek hi folder mein rakha
gaya — alag folders banana same write-flow ko baar-baar duplicate karta.
**Trust Verification** (generic per-attribute verification) genuinely alag system hai, dekho
[`../Trust Verification KT/`](../Trust%20Verification%20KT/Trust_Verification_Technical_Doc.md).

---

## 1. File Map

**Correction vs. purani doc**: purani doc ne `TSMiscDetailsController.go` ke function ko
`ActionTsDetails` bataya tha — yeh **galat** nikla. `TSMiscDetailsController.go` ka function
actually `TSMiscDetails` hai (file-naam se match karta hai, koi mismatch nahi). `ActionTsDetails`
ek **alag, poori tarah separate file** (`TsDetailsController.go`) mein hai jise purani doc ne
miss kar diya tha — yeh iss domain ka **paanchvan read-controller** hai, char nahi.

| Concern | Repo | File |
|---|---|---|
| TrustSeal write (single controller, TYPE-dispatched) | write | [`UserTrustsealController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTrustsealController.go), [`UserTrustsealModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTrustsealModel.go) |
| TrustSeal validation maps (field/type/length) | write | [`UsersValidationMaps.go:607-706`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) (`UserTrustsealMap`, `UserTrustCompMap`, `UserTrustsealDateKeyMap`) |
| TrustSeal mandatory-field checks | write | [`UserUtilsMandatory.go:281-`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go) (`MandatoryParamsCheckTrustseal`) |
| TrustSeal read (basic — GLID/verified-seller lookup) | read | [`TrustSealController.go`](../internal/controllers/UsersControllers/TrustSealController.go) (`ActionTrustSealController`), [`UserTrustSealModel.go`](../internal/models/users/UserTrustSealModel.go) (`GetTrustSealData`) |
| TrustSeal detail read (full aggregated profile — seal+contact+other-detail+bank) | read | [`TrustsealDetailController.go`](../internal/controllers/UsersControllers/TrustsealDetailController.go) (`TrustSealDetail`), [`UserTrustsealDetailModel.go`](../internal/models/users/UserTrustsealDetailModel.go) (`GetTrustSealDetails`, `Getcontactdetail`) |
| TrustSeal generic-lookup read (by TSCODE/TSID/COMPID/GLUSR_ID) | read | [`TsDetailsController.go`](../internal/controllers/UsersControllers/TsDetailsController.go) (`ActionTsDetails` — **the correctly-named-but-previously-missed file**), [`UserTsDetailsModel.go`](../internal/models/users/UserTsDetailsModel.go) (`UsersActionTsDetailsModel`) |
| TrustSeal Company details read | read | [`TSCompDetailsController.go`](../internal/controllers/UsersControllers/TSCompDetailsController.go) (`TSCompDetails`), [`UserTSCompDetailsModel.go`](../internal/models/users/UserTSCompDetailsModel.go) (`GetTSCompDetails`) |
| TrustSeal Misc details (Trade Affiliations/Std Quality/Safety Cert/Chamber/Export Status) read | read | [`TSMiscDetailsController.go`](../internal/controllers/UsersControllers/TSMiscDetailsController.go) (`TSMiscDetails` — file/function naam match karte hain), [`UserTSMiscDetailsModel.go`](../internal/models/users/UserTSMiscDetailsModel.go) (`GetTSMiscDetails`) |

---

## 2. Routes (confirmed from `routerUsers.go`, 3 identical route-groups mein registered)

| Method | Path | Repo | Controller |
|---|---|---|---|
| GET/POST | `/trustseal/*params` | read | `ActionTrustSealController` |
| GET/POST | `/trustsealdetail/*params` | read | `TrustSealDetail` |
| GET/POST | `/tsdetails/*params` | read | `ActionTsDetails` |
| GET/POST | `/tscompdetails/*params` | read | `TSCompDetails` |
| GET/POST | `/tsmiscdetails/*params` | read | `TSMiscDetails` |
| POST | `/user/trustseal` | write | `UserTrustsealController` (sirf `WEBERP` gateway) |

[`routerUsers.go:392-401,530-539,729-738`](../internal/api/users_router/routerUsers.go) — same
5 read routes teen alag jagah register hote hain (likely alag API-version/auth groups ke liye,
exact reason iss pass mein trace nahi hua).

---

## 3. Data Model — Tables

> **Verification note**: yeh sab table/column names Go code ke andar embedded SQL strings se
> liye gaye hain (har row ke saamne file:line diya hai). Live DB schema se pgAdmin pe
> cross-verify **nahi** kiya gaya hai.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `TRUSTSEAL` | meshpg (`GetPGDbConnection("meshpg")` write-side, `GetPG("mesh_pg_user")` read-side) | **Master record** — ek row per assigned seal, poora company-profile-jaisa snapshot (address, bank, turnover, registrations) bhi isi table mein embedded hai | `TRUSTSEAL_ID`, `TRUSTSEAL_CODE`, `FK_COMPANYID`, `TRUSTSEAL_GLUSR_USR_ID`, `TRUSTSEAL_CREATIONDATE`, `TRUSTSEAL_STARTDATE`, `TRUSTSEAL_ENDDATE`, `TRUSTSEAL_APPROV` (`'M'`/`'A'` seen — exact meaning [INFERRED]), `TRUSTSEAL_MODIFICATIONDATE`, plus ~70 more business-detail columns (bank, tax-registration numbers, turnover, sales-split, contact) — [`UsersValidationMaps.go:607-694`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |
| `TRUSTSEAL_COMPANY` | meshpg | Company(s) linked to a TrustSeal — supports a "primary" company flag | `TRUSTSEAL_COMPANY_ID`, `FK_TRUSTSEAL_ID`, `FK_COMPANY_ID`, `TRUSTSEAL_COMPANY_ISPRIMARY`, `FK_CUST_TO_SERV_ID` (customer-to-service/subscription link), `TRUSTSEAL_COMPANY_GLUSR_USR_ID`, `TRUSTSEAL_COMPANY_TSCODE`, `TRUSTSEAL_COMPANY_APPROV` — [`UsersValidationMaps.go:696-703`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go), [`UserTsDetailsModel.go:290`](../internal/models/users/UserTsDetailsModel.go) (`TRUSTSEAL_COMPANY_APPROV='A'` filter) |
| `TRUSTSEAL_HISTORY` | meshpg | Audit-trail — free-text history, **prepended** (not appended) to existing text on every update | `FK_TRUSTSEAL_ID`, `TRUSTSEAL_HISTORY_TEXT` — [`UserTrustsealModel.go:109,125,145,245,266`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTrustsealModel.go) |
| `TS_TRADE_AFFILIATIONS` | meshpg | Trade-affiliation memberships linked to a TrustSeal | `TS_TRADE_AFFILIATIONS_ID`, `FK_TRUSTSEAL_ID`, `FK_TRADE_AFFILIATIONS_ID`, `TS_TRADE_AFFILIATE_VALIDUPTO`, `TS_TRADE_AFFILIATE_IMGPATH` — [`UserTSMiscDetailsModel.go:122,126,129`](../internal/models/users/UserTSMiscDetailsModel.go) |
| `TRADE_AFFILIATIONS` | meshpg | Master/lookup — trade-affiliation names | `TRADE_AFFILIATIONS_ID`, `TRADE_AFFILIATIONS_NAME` — [`UserTSMiscDetailsModel.go:119,126,129`](../internal/models/users/UserTSMiscDetailsModel.go) |
| `TS_STANDARD_QUALITY_CERT` | meshpg | Standard-quality-certification memberships linked to a TrustSeal (structurally identical pattern to trade-affiliations) | `TS_STANDARD_QUALITY_CERT_ID`, `FK_TRUSTSEAL_ID`, `FK_STANDARD_QUALITY_CERT_ID`, `TS_STANDARD_QUALITY_VALIDUPTO`, `TS_STANDARD_QUALITY_IMGPATH` — [`UserTSMiscDetailsModel.go:138,142,145,148`](../internal/models/users/UserTSMiscDetailsModel.go) |
| `STANDARD_QUALITY_CERT` | meshpg | Master/lookup — quality-cert names, grouped by `FK_STANDARD_QUALITY_GRP_ID` | `STANDARD_QUALITY_CERT_ID`, `STANDARD_QUALITY_CERT_NAME`, `FK_STANDARD_QUALITY_GRP_ID` — [`UserTSMiscDetailsModel.go:135,142`](../internal/models/users/UserTSMiscDetailsModel.go) |
| `SAFETYCERT` | meshpg | Master/lookup — safety-certification names | `SAFETYCERT_ID`, `SAFETYCERT_NAME` — [`UserTSMiscDetailsModel.go:107`](../internal/models/users/UserTSMiscDetailsModel.go) |
| `CHAMBER` | meshpg | Master/lookup — chamber-of-commerce names | `CHAMBER_ID`, `CHAMBER_NAME` — [`UserTSMiscDetailsModel.go:112`](../internal/models/users/UserTSMiscDetailsModel.go) |
| `GLUSR_USR`, `GLUSR_USR_COMP_REGISTRATIONS`, `GLUSR_OTH_REM_DETAIL`, `GL_CITY`, `GL_DISTRICTS`, `GLUSR_RATING_AGGREGATE`, `GLUSR_USR_EXT` | meshpg | Read-side joins/enrichment — company name, contact, GST, registration numbers, city/district, average rating, extended profile fields (bank/turnover/legal-status) pulled in alongside TrustSeal for the "full profile" reads | Multiple, see queries in section 10 — [`UserTsDetailsModel.go:290`](../internal/models/users/UserTsDetailsModel.go), [`UserTrustsealDetailModel.go:66-94,298-329`](../internal/models/users/UserTrustsealDetailModel.go), [`UserTrustSealModel.go:125-158`](../internal/models/users/UserTrustSealModel.go) |
| `iil_verification_details` | meshpg | GST-verification-source-id lookup (`fk_gl_attribute_id = 352`) — used only inside `GetTrustSealData`'s "verified seller" branch | `iil_verification_user_src_id`, `fk_gl_attribute_id`, `fk_glusr_usr_id` — [`UserTrustSealModel.go:143`](../internal/models/users/UserTrustSealModel.go) |

---

## 4. `TRUSTSEAL_APPROV` / `TYPE` — Decode Attempt

Iss domain mein do "magic values" hain jo repeatedly compare hote hain: `TRUSTSEAL_APPROV`
(row-level status char) aur `TYPE` (request-dispatch string). Dono ke liye jitna evidence code
mein mila, neeche hai — jahan tak nahi pahuncha wahan **[INFERRED]** maaka hai.

### `TRUSTSEAL_APPROV` (and `TRUSTSEAL_COMPANY_APPROV`)

| Value | Where seen | Meaning [INFERRED — confirm with team] |
|---|---|---|
| `'A'` | Filter in `GetTrustSealData`'s tscode-lookup query: `tc.TRUSTSEAL_COMPANY_APPROV='A'` — [`UserTrustSealModel.go:290`](../internal/models/users/UserTrustSealModel.go); also part of the `IN ('M','A')` filter in the write-side company-linked end-date update — [`UserTrustsealModel.go:119`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTrustsealModel.go) | **Approved** — likely the only state that surfaces in the public/buyer-facing TSCODE lookup |
| `'M'` | Only appears in write-side `WHERE TRUSTSEAL_APPROV IN ('M','A')` filter — [`UserTrustsealModel.go:119`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTrustsealModel.go) | **Manual-approved / Marked** — treated as equally-eligible-for-update alongside `'A'`, but never appears as a standalone filter condition anywhere in the reads reviewed |

No enum, no comment, no master table referenced anywhere in the reviewed code for these two
characters. Team confirmation needed before using these in customer-facing copy or dashboards.

### `TYPE` dispatch values (write controller, `UserTrustsealController.go`)

| Value | What it does | Evidence |
|---|---|---|
| `TS` | Validates against `UserTrustsealMap` only; routes to `mesh_pg_ts_action` `TS` branch (single-record `TRUSTSEAL` UPDATE by `TRUSTSEAL_ID`, `GLUSR_ID`, or `CUST_TO_SERV_ID`) | [`UserTrustsealController.go:54-58`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTrustsealController.go), [`UserTrustsealModel.go:112-181`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTrustsealModel.go) |
| `TS_COMP` | Validates against `UserTrustCompMap` only; routes straight to `mesh_pg_ts_comp_action` (`TRUSTSEAL_COMPANY` insert/update/delete) | [`UserTrustsealController.go:59-60`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTrustsealController.go), [`UserTrustsealModel.go:34-36,293-407`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTrustsealModel.go) |
| `ALL` | Validates against **both** maps concatenated; routes to `mesh_pg_ts_action` `ALL` branch, which on `ACTION=I` **also internally calls `mesh_pg_ts_comp_action` with `ACTION=I`** — i.e. one `TYPE=ALL,ACTION=I` request creates both a `TRUSTSEAL` row and its linked `TRUSTSEAL_COMPANY` row in one call | [`UserTrustsealController.go:61-64`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTrustsealController.go), [`UserTrustsealModel.go:49-111,233-239`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTrustsealModel.go) |

No other `TYPE` value produces a non-empty `queries` slice — anything else falls through to
`"No or Incorrect values provided for TYPE:...and ACTION:..."` — [`UserTrustsealModel.go:184-186,384-386`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTrustsealModel.go).
`ACTION` itself is a simple `I`/`U`/`D` flag (Insert/Update/Delete), confirmed across both
`mesh_pg_ts_action` and `mesh_pg_ts_comp_action`.

`TSMiscDetailsController`'s `TYPE` param (read-side, separate from the write `TYPE` above) has
a **fully confirmed, closed list** because the controller itself validates it before calling the
model: `EXPORT_STATUS`, `TRADE_AFF`, `STD_QUALITY`, `CHAMBER`, `SAFETYCERT` —
[`TSMiscDetailsController.go:91`](../internal/controllers/UsersControllers/TSMiscDetailsController.go).

---

## 5. Business Rules & Validation (code se exhaustive list)

1. **Sirf `WEBERP` gateway allowed hai write-path pe** — `Gateway_v1([]string{"WEBERP"}, ...)`.
   [`UserTrustsealController.go:43`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTrustsealController.go)
2. **Reads bhi `MODID=weberp` explicitly require karte hain**: `TSCompDetails`, `TSMiscDetails`,
   `ActionTsDetails` sab case-insensitive `modid != "weberp"` check karke reject karte hain —
   yaani read-paths **bhi** internal-system-only hain, sirf write nahi. (`ActionTrustSealController`
   aur `TrustSealDetail` iss specific check ko skip karte hain, generic modid-validity check
   use karte hain — inconsistency, dekho section 11.)
   [`TSCompDetailsController.go:115`](../internal/controllers/UsersControllers/TSCompDetailsController.go), [`TSMiscDetailsController.go:131`](../internal/controllers/UsersControllers/TSMiscDetailsController.go), [`TsDetailsController.go:78`](../internal/controllers/UsersControllers/TsDetailsController.go)
3. **`TYPE` param request ko dispatch karta hai** (section 4) — `TS`, `TS_COMP`, `ALL` teen
   confirmed values write-side. `ALL,I` ek hi call mein `TRUSTSEAL` + `TRUSTSEAL_COMPANY` dono
   insert kar deta hai.
   [`UserTrustsealController.go:46,54-64`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTrustsealController.go)
4. **History-text prepend pattern, not append**: `TRUSTSEAL_HISTORY_TEXT=COALESCE($1,'') ||
   COALESCE(TRUSTSEAL_HISTORY_TEXT,'')` — naya text purane text ke **pehle** jud़ta hai, latest
   change hamesha text ke shuru mein milega.
   [`UserTrustsealModel.go:109,125,145,245`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTrustsealModel.go)
5. **History-insert-vs-update dono support karta hai** — agar `TS` type ke UPDATE-by-`TRUSTSEAL_ID`
   path mein history-row already exist nahi karti (UPDATE 0 rows affect kare), phir `INSERT INTO
   TRUSTSEAL_HISTORY` fallback hota hai. Baaki paths (`TS` by GLUSR_ID/CUST_TO_SERV_ID, `ALL`,
   `TS_COMP`) hamesha sirf UPDATE try karte hain, koi INSERT-fallback nahi — yaani agar in paths
   mein history-row exist hi nahi karti, update silently 0-rows-affected ho jaata hai, koi error
   nahi aata.
   [`UserTrustsealModel.go:243-286`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTrustsealModel.go)
6. **`TS` type, `GLUSR_ID` path — nested subquery apna `TRUSTSEAL_ID` khud resolve karta hai**,
   `WHERE ... TRUSTSEAL_APPROV IN ('M','A')` filter ke saath — matlab sirf `'M'`/`'A'` waale
   seals hi is path se update ho sakte hain, koi bhi pending/rejected seal is route se silently
   skip ho jaata hai (0 rows affected, no error).
   [`UserTrustsealModel.go:119`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTrustsealModel.go)
7. **`TS_COMP` insert ek pre-check karta hai**: `SELECT count(trustseal_id) FROM trustseal WHERE
   trustseal_id=$1` — agar 0, insert explicitly reject hota hai
   `"TRUSTSEAL ID =>... is not present in TRUSTSEAL"` ke saath — yaani orphan `TRUSTSEAL_COMPANY`
   row create nahi ho sakti, FK integrity application-level enforce hoti hai (DB-level FK
   constraint hai ya nahi, confirm nahi hua).
   [`UserTrustsealModel.go:302-325`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTrustsealModel.go)
8. **Misc-details (Trade Affiliations, Std Quality) do read-modes support karte hain**:
   linked (already-jud़i, `FK_TRUSTSEAL_ID` se, `isreq` param se triggered) vs. available
   (`NOT IN` subquery se, jo abhi link nahi hui) — ek classic "add new item" dropdown pattern,
   dono trade-affiliations aur std-quality-certs ke liye symmetric hai.
   [`UserTSMiscDetailsModel.go:124-131,140-150`](../internal/models/users/UserTSMiscDetailsModel.go)
9. **`EXPORT_STATUS` aur no-tsid `STD_QUALITY` static, in-memory data return karte hain, koi DB
   query nahi** — `components.ExportStatusMap` / `components.ExportstandardQualityGrpMap` se seedha
   serve hota hai.
   [`UserTSMiscDetailsModel.go:116,152,158-161`](../internal/models/users/UserTSMiscDetailsModel.go)
10. **`GetTrustSealData` (basic read) do completely different response-shapes deta hai** depending
    on `is_verified` param: `is_verified=1` + valid `glusrid` → masked "verified seller data"
    (3 parallel queries — company/contact info, GST-src-id lookup, GST+IEC); else → single-row
    TSCODE-based lookup (`GLUSR_USR_COMPANYNAME`, GST masked, avg rating, etc.). In dono paths ke
    field-list aur masking rules alag hain.
    [`UserTrustSealModel.go:106,265`](../internal/models/users/UserTrustSealModel.go)
11. **GSTIN aur email masking** — `is_verified` path mein email ka local-part first-char ke baad
    sab `x` se mask hota hai (`ax xxxx@domain.com`), aur agar `identified != 1` mobile bhi middle
    digits mask karta hai. Non-verified TSCODE-lookup path mein GST `maskGSTIN()` se mask hota hai
    (first 2 + last 2 chars visible, beech `*`). Yeh do alag masking implementations hain same
    controller ke andar.
    [`UserTrustSealModel.go:50-57,235-256,421-424`](../internal/models/users/UserTrustSealModel.go)
12. **`ActionTsDetails`/`UsersActionTsDetailsModel` mein `TSID` param full-row `SELECT *` karta
    hai** (`SELECT * FROM TRUSTSEAL WHERE TRUSTSEAL_ID=$1`) — yeh iss doc ke sab read-paths mein
    ek hi jagah hai jo `SELECT *` use karti hai; baaki sab explicit column-list select karte hain.
    Isse har naya column jo `TRUSTSEAL` table mein add ho, automatically iss response mein bhi
    aa jaayega bina code-change ke — API-contract stability ke liye risk.
    [`UserTsDetailsModel.go:45`](../internal/models/users/UserTsDetailsModel.go)
13. **`UserTrustSealModel.go` mein query-result package-level global variables use karte hain**
    (`trustsealID`, `gst`, `firstName`, etc. — 20+ package-scoped `var` declarations at file top,
    populated via `connection.QueryRow(...).Scan(&trustsealID, ...)`). Concurrent requests jo yeh
    hi function call karte hain **same memory location share karenge** — ek potential race
    condition agar iss endpoint pe concurrency high ho. Dekho section 12, optimization point #1.
    [`UserTrustSealModel.go:20-46,293-319`](../internal/models/users/UserTrustSealModel.go)

---

## 6. RabbitMQ

**Iss pass mein koi RabbitMQ usage nahi mila** TrustSeal write ya read paths mein — grep kiya
gaya `PushToQueue`/`SERVICENAME`/`Requeue` ke liye `UserTrustsealController.go` aur
`UserTrustsealModel.go` dono mein, koi match nahi. `user-temp-consumers-production` repo mein
bhi `trustseal`/`TRUSTSEAL` grep kiya — sirf 3 unrelated files match hue (generic user-attribute
maps jo coincidentally `TRUSTSEAL_CODE` field-name reference karte hain kisi baaki attribute-list
mein, TrustSeal-specific consumer nahi). **Conclusion: TrustSeal domain ke liye koi dedicated
RabbitMQ consumer nahi hai.**

## 7. Kafka

**Koi Kafka usage nahi mila.** Sab TrustSeal writes synchronous, single-request DB operations
hain — koi async messaging trigger nahi hota.

## 8. Redis

**Koi Redis usage nahi mila** — na write-controller mein, na kisi bhi 5 read-model mein. Har
read live meshpg query hai, koi cache-layer wire nahi hai (dekho section 12, Optimization Scope).

---

## 9. End-to-End Technical Flows

### Flow A — WEBERP naya TrustSeal + Company create karta hai (`TYPE=ALL, ACTION=I`)

```
WEBERP (internal system, sirf isse allowed)
    │
    ▼
[API — write]  POST /user/trustseal  {TYPE: "ALL", ACTION: "I", TRUSTSEAL_ENTITYNAME, ...,
                                       COMPANY_ID, TRUSTSEAL_COMPANY_ISPRIMARY, ...}
    │  UserTrustsealController.go
    │  1. Gateway check: WEBERP only (else reject)
    │  2. Mandatory-field check (MandatoryParamsCheckTrustseal, TYPE=ALL/ACTION=I branch)
    │  3. Length+type validation (UserTrustsealMap + UserTrustCompMap concatenated)
    ▼
[DB — meshpg]  INSERT INTO TRUSTSEAL (...) RETURNING TRUSTSEAL_ID
    │  (dynamic column-list built from whichever fields were passed + present in UserTrustsealMap)
    ▼
[DB — meshpg, same request]  mesh_pg_ts_comp_action(ACTION=I) called internally with the new
    │  TRUSTSEAL_ID as COMP_TRUSTSEAL_ID
    ├─ SELECT count(trustseal_id) FROM trustseal WHERE trustseal_id=$1   (existence check)
    ├─ INSERT INTO TRUSTSEAL_COMPANY (...)
    └─ INSERT INTO TRUSTSEAL_HISTORY (FK_TRUSTSEAL_ID, TRUSTSEAL_HISTORY_TEXT)
    ▼
Response: "SUCCESS || SUCCESS" (two sub-operations concatenated) or partial-failure string
```

### Flow B — WEBERP existing TrustSeal update karta hai (`TYPE=TS, ACTION=U`)

```
WEBERP
    │
    ▼
[API — write]  POST /user/trustseal  {TYPE: "TS", ACTION: "U", TRUSTSEAL_ID | GLUSR_ID |
                                       CUST_TO_SERV_ID, TRUSTSEAL_ENDDATE, HIST_COMMENT, ...}
    │  1. Gateway check: WEBERP only
    │  2. Mandatory-check: at least one of TRUSTSEAL_ID/GLUSR_ID/CUST_TO_SERV_ID required
    ▼
Dispatch on which identifier was passed (mutually exclusive branches in mesh_pg_ts_action):
    │
    ├─ GLUSR_ID present   → UPDATE TRUSTSEAL SET ENDDATE... WHERE TRUSTSEAL_ID =
    │                        (subquery joining TRUSTSEAL_COMPANY+TRUSTSEAL, APPROV IN ('M','A'))
    ├─ CUST_TO_SERV_ID     → UPDATE TRUSTSEAL SET ENDDATE... WHERE TRUSTSEAL_ID =
    │   present             (subquery on TRUSTSEAL_COMPANY.FK_CUST_TO_SERV_ID)
    └─ TRUSTSEAL_ID present→ UPDATE TRUSTSEAL SET <dynamic column list> WHERE TRUSTSEAL_ID=$n
                             (flag_hist=1 → triggers UPDATE-then-INSERT-fallback history write)
    │
    ▼
[DB]  UPDATE (or INSERT-fallback) TRUSTSEAL_HISTORY — prepend HIST_COMMENT
    ▼
Response: "SUCCESS" or "MESH PG query failure for TRUSTSEAL table..."
```

### Flow C — WEBERP company-link update/delete (`TYPE=TS_COMP`)

```
WEBERP
    │
    ▼
[API — write]  POST /user/trustseal  {TYPE: "TS_COMP", ACTION: "U"|"D", ...}
    │  Validated against UserTrustCompMap only
    ▼
mesh_pg_ts_comp_action dispatch on ACTION + which params present:
    │
    ├─ U + TRUSTSEAL_COMPANY_TSCODE  → UPDATE TRUSTSEAL_COMPANY SET TSCODE WHERE ...GLUSR_USR_ID=$2
    ├─ U + COMP_TRUSTSEAL_ID         → UPDATE TRUSTSEAL_COMPANY SET COMPANY_ID,ISPRIMARY
    │                                   WHERE FK_TRUSTSEAL_ID=$3 AND ...GLUSR_USR_ID=$4
    └─ D                             → DELETE FROM TRUSTSEAL_COMPANY WHERE ...GLUSR_USR_ID=$1
    │
    ▼
[DB]  UPDATE TRUSTSEAL_HISTORY (prepend) — same FK_TRUSTSEAL_ID target
    ▼
Response: "SUCCESS" or failure string
```

### Flow D — Basic TrustSeal read (verified-seller vs. TSCODE lookup)

```
Caller (internal — modid/glusrid required)
    │
    ▼
[API]  GET/POST /trustseal  {glusrid, is_verified?, identified?, tscode?}
    │  ActionTrustSealController.go — token/modid validation
    ▼
GetTrustSealData()
    │
    ├─ is_verified==1 AND glusrid!=0
    │       │
    │       ▼
    │   3 parallel goroutines (sync.WaitGroup):
    │   ├─ GLUSR_USR + GL_CITY + GL_DISTRICTS  → name/address/contact
    │   ├─ iil_verification_details (attr_id=352) → GST verification src_id
    │   └─ GLUSR_USR_COMP_REGISTRATIONS         → GST, DGFT/IEC code
    │       │
    │       ▼
    │   Mask email (local-part), mask mobile (if identified!=1) → "verified_seller_data"
    │
    └─ else (tscode-based path)
            │
            ├─ (if no tscode given) SELECT trustseal_code FROM trustseal WHERE glusr_usr_id=$1
            │
            ▼
        Single-row SELECT joining TRUSTSEAL+TRUSTSEAL_COMPANY+GLUSR_USR+
        GLUSR_USR_COMP_REGISTRATIONS+GLUSR_OTH_REM_DETAIL, filtered
        TRUSTSEAL_COMPANY_APPROV='A', ORDER BY TRUSTSEAL_ID DESC LIMIT 1
            │
            ▼
        Mask GST (maskGSTIN) → "trustseal_data"
```

### Flow E — Full aggregated TrustSeal-detail read

```
Caller (internal — modid/glusrid required)
    │
    ▼
[API]  GET/POST /trustsealdetail  {glusrid, modid}
    │  TrustSealDetail.go
    ▼
GetTrustSealDetails()
    │
    ├─ SELECT ... FROM trustseal t JOIN trustseal_company tc JOIN glusr_usr g
    │   LEFT JOIN glusr_usr_comp_registrations cr LEFT JOIN glusr_oth_rem_detail o
    │   WHERE t.trustseal_glusr_usr_id=$1                       (main TrustSeal snapshot)
    │
    ▼
    2 parallel goroutines (sync.WaitGroup):
    ├─ Getcontactdetail()      → SELECT ... FROM glusr_usr, glusr_usr_ext (contact+extended)
    └─ UserOtherDetailModel()  → reused generic "other detail" model (bank details,
                                  turnover, legal status, etc. — cross-domain, not TrustSeal-
                                  specific; see account_creation docs for that model)
    │
    ▼
Assemble combined response: GLUSR_DATA + OTHER_DETAIL + BANK_DETAIL + TRUSTSEAL_DATA
```

### Flow F — Generic lookup read (TSCODE / TSID / COMPID / GLUSR_ID)

```
Caller (internal — modid + TOKEN required)
    │
    ▼
[API]  GET/POST /tsdetails  {TSCODE | TSID | COMPID | GLUSR_ID, MODID}
    │  TsDetailsController.go — modid must literally equal "weberp" (case-insensitive)
    ▼
UsersActionTsDetailsModel() — exactly one branch fires, first-match wins:
    │
    ├─ TSCODE present → SELECT COUNT(TRUSTSEAL_CODE) FROM TRUSTSEAL WHERE CODE=LOWER($1)
    ├─ TSID present    → SELECT * FROM TRUSTSEAL WHERE TRUSTSEAL_ID=$1   (full-row, see §5.12)
    ├─ COMPID present  → SELECT (7 cols) FROM TRUSTSEAL WHERE FK_COMPANYID=$1 ORDER BY ID DESC
    └─ GLUSR_ID present→ SELECT (6 cols) FROM TRUSTSEAL WHERE TRUSTSEAL_GLUSR_USR_ID=$1
```

### Flow G — Company-details read (multi-branch identifier lookup)

```
Caller (internal — MODID=weberp required)
    │
    ▼
[API]  GET/POST /tscompdetails  {COMPID | TSID | GLUSR_ID | COMPCODE | tscompid, COMP_ISPRIMARY?}
    │  TSCompDetailsController.go
    ▼
GetTSCompDetails() — 9 distinct query-shapes depending on which identifier-combination is
    present (COMPID+isPrimary, COMPID+TSID, COMPID alone, GLID variants, COMPCODE variants,
    tscompid, TSID+isPrimary, TSID alone) — all SELECT-only against TRUSTSEAL_COMPANY
```

### Flow H — Misc-details read (Trade Affiliations / Std Quality / Safety Cert / Chamber / Export)

```
Caller (internal — MODID=weberp required)
    │
    ▼
[API]  GET/POST /tsmiscdetails  {TYPE, ...type-specific params}
    │  TSMiscDetailsController.go — TYPE must be one of the 5 confirmed values (section 4)
    ▼
GetTSMiscDetails() dispatch on TYPE:
    │
    ├─ SAFETYCERT    → SELECT SAFETYCERT_ID FROM SAFETYCERT WHERE name lookup
    ├─ CHAMBER       → SELECT CHAMBER_ID FROM CHAMBER WHERE name lookup
    ├─ EXPORT_STATUS → static in-memory data, no DB call
    ├─ TRADE_AFF     → 4 sub-branches: name-lookup / by-tradeaffid / linked-list(tsid+isreq) /
    │                   available-list(tsid, NOT IN subquery)
    └─ STD_QUALITY   → 5 sub-branches: name-lookup / by-certid / by-group(tsid+qualgrpid) /
                        linked-list(tsid+isreq) / available-list(tsid) / static-fallback
```

---

## 10. Flow-wise DB & Table Usage — Kaun sa DB, Kaun sa Table, Kis Liye

### Flow A — Create TrustSeal + Company (`TYPE=ALL, ACTION=I`)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `TRUSTSEAL` | INSERT ... RETURNING TRUSTSEAL_ID | Naya seal-record banata hai, dynamic column-list se |
| 2 | meshpg | `TRUSTSEAL` | SELECT count(*) WHERE trustseal_id=$1 | Existence-check before inserting the linked company row |
| 3 | meshpg | `TRUSTSEAL_COMPANY` | INSERT | Naya company-link banata hai (primary flag, cust-to-serv link) |
| 4 | meshpg | `TRUSTSEAL_HISTORY` | INSERT | Audit-entry, naye seal ke liye |

### Flow B — Update TrustSeal (`TYPE=TS, ACTION=U`)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `TRUSTSEAL` (join `TRUSTSEAL_COMPANY` in subquery for GLUSR_ID/CUST_TO_SERV_ID paths) | UPDATE | End-date/start-date/status update — exact target row identifier-dependent |
| 2 | meshpg | `TRUSTSEAL_HISTORY` | UPDATE, INSERT-fallback if 0 rows affected (TRUSTSEAL_ID path only) | Audit-trail prepend |

### Flow C — Company-link update/delete (`TYPE=TS_COMP`)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `TRUSTSEAL_COMPANY` | UPDATE or DELETE | Company-link ya primary-flag ya tscode change/removal |
| 2 | meshpg | `TRUSTSEAL_HISTORY` | UPDATE (prepend) | Audit-trail, no insert-fallback in this path |

### Flow D — Basic read (`GET /trustseal`)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1a | meshpg | `GLUSR_USR` + `GL_CITY` + `GL_DISTRICTS` | SELECT (parallel goroutine 1) | Name/address/city-district — is_verified path |
| 1b | meshpg | `iil_verification_details` | SELECT (parallel goroutine 2) | GST verification src_id — is_verified path |
| 1c | meshpg | `GLUSR_USR_COMP_REGISTRATIONS` | SELECT (parallel goroutine 3) | GST + DGFT/IEC — is_verified path |
| 2a | meshpg | `TRUSTSEAL` | SELECT (optional, only if tscode not given) | Resolve tscode from glusrid |
| 2b | meshpg | `TRUSTSEAL` + `TRUSTSEAL_COMPANY` + `GLUSR_USR` + `GLUSR_USR_COMP_REGISTRATIONS` + `GLUSR_OTH_REM_DETAIL` + `GL_CITY` + `GL_DISTRICTS` + `GLUSR_RATING_AGGREGATE` | SELECT (single 8-way join) | Full TSCODE-lookup profile — non-is_verified path |

**Note**: paths 1a-1c aur 2a-2b are mutually exclusive per-request (only one branch fires,
depending on `is_verified`), not additive.

### Flow E — Aggregated detail read (`GET /trustsealdetail`)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `TRUSTSEAL` + `TRUSTSEAL_COMPANY` + `GLUSR_USR` + `GLUSR_USR_COMP_REGISTRATIONS` + `GLUSR_OTH_REM_DETAIL` | SELECT (5-way join) | Main TrustSeal snapshot |
| 2 | meshpg | `GLUSR_USR` + `GLUSR_USR_EXT` | SELECT (parallel goroutine) | Contact + extended profile (bank/turnover/legal-status raw fields) |
| 3 | (multiple, via reused `UserOtherDetailModel`) | Multiple — see account_creation docs | SELECT (parallel goroutine, cross-domain reused model) | "Other detail" block — bank details, turnover, legal status, formatted |

### Flow F — Generic lookup (`GET /tsdetails`)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg (`mesh_pg_user`) | `TRUSTSEAL` | SELECT (exactly one of 4 shapes, per identifier passed) | TSCODE existence-check, TSID full-row, COMPID list, or GLUSR_ID list |

### Flow G — Company-details read (`GET /tscompdetails`)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `TRUSTSEAL_COMPANY` | SELECT (exactly one of 9 shapes, per identifier-combination) | Company-link lookup by company/seal/glusr/code identifiers |

### Flow H — Misc-details read (`GET /tsmiscdetails`)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `SAFETYCERT` / `CHAMBER` / `TRADE_AFFILIATIONS` / `TS_TRADE_AFFILIATIONS` / `STANDARD_QUALITY_CERT` / `TS_STANDARD_QUALITY_CERT` (exactly one, depending on TYPE+params) | SELECT | Name-lookup, linked-list, or available-list per the requested misc-detail type |
| — | (none) | `EXPORT_STATUS`, no-tsid `STD_QUALITY` | in-memory | No DB round-trip — static data served from `components` package |

---

## 11. Optimization Scope — DB / Response-Time Contribution

### High-impact

1. **Package-level global variables in `UserTrustSealModel.go` used for `QueryRow().Scan()`
   results** (`trustsealID`, `gst`, `firstName`, etc. — section 5.13). Agar iss endpoint
   (`GET /trustseal`, non-is_verified path) pe concurrent requests aayen, Go ka garbage
   collector inhe collect nahi karega (package scope), aur **do concurrent requests same
   memory location overwrite kar sakte hain** race condition ke through — ek request ka data
   doosre ke response mein leak ho sakta hai. **Concrete fix**: in variables ko function-local
   banao (already-existing pattern jo baaki sab models follow karte hain).
   [`UserTrustSealModel.go:20-46,293-319`](../internal/models/users/UserTrustSealModel.go)

### Medium-impact

2. **History-write pattern (prepend, `COALESCE(...) || COALESCE(...)`) grows unbounded** — ek
   TrustSeal jitna purana/frequently-updated hoga, `TRUSTSEAL_HISTORY_TEXT` utna bada single
   text-blob banta jaayega. Har naya update **poora purana text bhi dobara likhta hai**
   (read-modify-write ek hi column pe) — write-cost badhta jaayega jitni lambi history ho.
   **Concrete fix**: ek proper append-only history-table (multiple rows, ek row per change)
   row-growth ki jagah column-growth se better scale karega.
   [`UserTrustsealModel.go:109,245`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTrustsealModel.go)
3. **5 alag read-endpoints, koi combined/single response nahi** — agar ek UI ko seal + company
   + misc-details + aggregated-detail teeno-charo chahiye ek page pe, woh 4-5 separate API
   calls karega. Ek combined "TrustSeal full profile" endpoint response-time (network
   round-trips) kam kar sakta hai, agar yeh actual usage-pattern hai frontend/WEBERP mein.
4. **`GetTrustSealDetails` (Flow E) 3 sequential/parallel query-groups chalata hai across 2
   physical goroutine-batches** — main query sequential hai, phir 2 parallel goroutines. Yeh
   already partially optimized hai (parallelism present), lekin main query khud sequential
   hai baaki se — agar main-query ka result baaki 2 pe depend nahi karta (jo lagta hai, glusrid
   hi input hai teeno ke liye), teeno ko ek saath 3-way parallel kiya ja sakta hai, 2-way ki
   jagah.
   [`UserTrustsealDetailModel.go:64-178`](../internal/models/users/UserTrustsealDetailModel.go)
5. **Koi caching nahi hai kisi bhi TrustSeal read endpoint pe** — sab 5 reads har request pe
   live meshpg hit karte hain. TrustSeal data infrequently change hota hai (yeh ek internal-
   admin-driven, low-write-frequency domain hai) — yeh ek natural caching candidate hai, redis
   infra codebase mein already hai, bas iss domain mein wire nahi hai.

### Low-impact / good-practice already present

6. **`GetTrustSealData` aur `GetTrustSealDetails` dono already parallel goroutines use karte
   hain** independent queries ke liye (`sync.WaitGroup`) — yeh achi optimization pattern hai
   already in place.
7. **`ActionTsDetails`'s `TSID` path ka `SELECT *`** (section 5.12) low-impact response-time
   pe hai currently (single-row lookup), lekin agar `TRUSTSEAL` table mein columns badhte
   rahen, response-payload-size grow karega bina kisi intentional API-versioning ke — worth
   converting to explicit column-list preemptively.

---

## 12. Cron Inventory

**Koi TrustSeal-specific cron nahi mila.** `user-temp-consumers-production/.../crons/`
directory ko grep kiya `trustseal`/`TRUSTSEAL` ke liye — sirf 3 unrelated files match hue
(generic attribute-name lists), koi actual TrustSeal cron/consumer logic nahi. Yeh GST domain
(jisme daily `gst_tact_veri_cron.go` hai) se contrast karta hai — TrustSeal poori tarah
WEBERP-triggered, on-demand hai, koi background re-verification/reconciliation job nahi.

---

## 13. Edge Cases & Gotchas (technical POV)

1. **`TRUSTSEAL_APPROV`/`TRUSTSEAL_COMPANY_APPROV` values (`'M'`, `'A'`) ka exact meaning
   decode nahi hua** — likely "Manual"/"Auto" ya "Marked"/"Approved" — confirm karo
   master-table/enum se (section 4).
2. **Purani doc ka file-map galat tha**: `TSMiscDetailsController.go` ka function `TSMiscDetails`
   hai (file-naam se match), `ActionTsDetails` ek poori tarah alag file
   (`TsDetailsController.go`) mein hai jo purani doc mein completely missing thi. Is domain mein
   **5 read-controllers hain, 4 nahi.**
3. **`GLUSR_ID` aur `CUST_TO_SERV_ID` update-paths silently 0-rows-affect ho sakte hain**
   (section 5.6) agar target seal `TRUSTSEAL_APPROV NOT IN ('M','A')` ho — response phir bhi
   `"SUCCESS"` aayega (query error nahi hai, bas 0 rows match hui) — caller ko pata hi nahi
   chalega ki kuch update nahi hua.
4. **Race-condition risk package-level globals ki wajah se** (`UserTrustSealModel.go`,
   section 5.13, 11.1) — high-priority fix candidate agar iss endpoint pe concurrency badhe.
5. **History-table unbounded-growth risk** (section 11.2).
6. **5 alag reads, alag-alag `modid` validation strictness** — `TSCompDetails`/`TSMiscDetails`/
   `ActionTsDetails` sab literal `modid=="weberp"` require karte hain, jabki
   `ActionTrustSealController`/`TrustSealDetail` sirf generic modid-validity check karte hain
   (`CheckModId`/`CheckValidity`) bina `weberp`-specific hardcode ke. Agar koi non-WEBERP caller
   in do reads ko hit kar sakta hai jo baaki 3 nahi kar sakte, yeh ek access-control
   inconsistency hai — worth security-review.
7. **`SELECT *` ek jagah hai** (`ActionTsDetails`, TSID path) — baaki sab explicit column-list.

---

## 14. Open Questions

1. `TRUSTSEAL_APPROV`/`TRUSTSEAL_COMPANY_APPROV` ke exact status-values aur unka
   business-meaning?
2. TrustSeal ↔ GST-verification ke beech koi automatic trigger hai (jaise "GST verify hone pe
   TrustSeal-eligibility flag set ho"), ya poori tarah manual/WEBERP-driven hai? Code mein koi
   trigger nahi mila, lekin yeh confirm karna chahiye ki koi cross-repo/cross-team automation
   iss review ke scope se bahar toh nahi hai.
3. `FK_CUST_TO_SERV_ID` (customer-to-service link) — kaunsi paid-service iske through TrustSeal
   ko link karti hai?
4. Same 5 routes teen alag route-groups mein kyun register hote hain (`routerUsers.go:392-401,
   530-539,729-738`) — alag API-version/auth-group ke liye, ya duplicate registration jo consolidate
   ho sakta hai?
5. Kya `modid=="weberp"` strictness-inconsistency (section 13.6) intentional hai ya accidental
   drift?
6. Package-level global-variable race-condition (section 11.1) — kya production mein iska koi
   observed impact hua hai (data-leak bug reports)?
7. Live DB schema verification (column types, nullability, indexes, constraints, FK enforcement
   for `TRUSTSEAL_COMPANY.FK_TRUSTSEAL_ID`) — yeh doc sirf Go SQL strings jo imply karti hain
   wahi reflect karta hai.

---

## See also

- [`TrustSeal_Business_Doc.md`](./TrustSeal_Business_Doc.md) — product perspective, bina code ke
- [`../Trust Verification KT/Trust_Verification_Technical_Doc.md`](../Trust%20Verification%20KT/Trust_Verification_Technical_Doc.md) — related, separate generic attribute-verification system
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — depth/structure reference used for this doc, and a related "trust" domain that TrustSeal reads (GST, IEC) alongside its own data
- [`../trust_verification_compliance_read_write_picture.md`](../trust_verification_compliance_read_write_picture.md) — bigger Trust & Compliance technical trace

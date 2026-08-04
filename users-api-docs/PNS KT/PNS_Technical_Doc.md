# PNS (GSM Number Allocation / Transaction) — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`PNS_Business_Doc.md`](./PNS_Business_Doc.md) dekho.
Dono docs same flows cover karte hain, bas alag audience ke liye.

**Scope note**: PNS (yeh doc) aur **PNS Setting** genuinely **do alag concepts** hain — alag
tables (`GL_GSM_MASTER` vs `IIL_PNS_SETTING`), alag controllers, koi FK-relation ya
shared-code nahi mila is pass mein bhi. Dekho
[`../PNS Setting KT/PNS_Setting_Technical_Doc.md`](../PNS%20Setting%20KT/PNS_Setting_Technical_Doc.md).
Ek chhota overlap zaroor mila: `UserOtherDetailModel.go:290-291` mein ek read query
`IIL_PNS_SETTING` ko left-join karta hai sirf ek `PNS_MARKED` flag ke liye — yeh PNS Setting
ka apna column hai, PNS-transaction (`GL_GSM_MASTER`) domain se unrelated.

**Repos**: `service-api-go-production` (write), `users-api-go-production` (read),
`user-temp-consumers-production` (consumers). **Previous pass ne read-endpoints aur
consumers dono miss kiye the** — yeh pass dono repos mein deeper search kar ke unhe mila.

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file:line diya
gaya hai). Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — team se confirm karo]** likha hai, guess nahi kiya.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write — allocation/txn controller | write | [`PnsTxnController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/PnsTxnController.go) (`UserPnsAllocationController`) |
| Write — allocation/txn model (all business logic) | write | [`PnsTxnModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PnsTxnModel.go) (`UpsertPnstxn` + 7 helper funcs) |
| Validation | write | `MandatoryParamsCheckPnsTxn` — [`UserUtilsMandatory.go:1580-1707`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go), field-map `UserPnsTransactionMap` + `VendorSlice` — [`UsersValidationMaps.go:727-740`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |
| RabbitMQ routing config | write | [`rabbitmq.go:40,42,73`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go) — `SERVICENAME`→queue/exchange map |
| **Read — OBD/GSM number lookup by supplier (NEW, missed by previous pass)** | read | [`OBDNumberController.go`](../../users-api-go-production/internal/controllers/UsersControllers/OBDNumberController.go) (`ObdNumber`), [`UserOBDNumberModel.go`](../../users-api-go-production/internal/models/users/UserOBDNumberModel.go) (`OBDNumber`) |
| **Read — generic GL-master lookup incl. `gl_gsm_master` (NEW, missed by previous pass)** | read | [`detail.go`](../../users-api-go-production/internal/controllers/GlmasterController/detail.go) (`GlmasterDetail`), [`GlmasterDetail.go`](../../users-api-go-production/internal/models/master/GlmasterDetail.go) |
| Read — inline GSM-vendor-type lookups inside product-listing queries (not a dedicated PNS endpoint, incidental joins) | read | [`UsersPdp.go:1419,1622`](../../users-api-go-production/internal/models/products/UsersPdp.go), [`BulkProductDetail.go:174`](../../users-api-go-production/internal/models/products/BulkProductDetail.go) |
| Consumer — sync GL_GSM_MASTER to authPg replica | consumers | [`USER_PNS_AUTHPG.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PNS_AUTHPG.go) (`UserPnsAuthPg`) |
| Consumer — sync GL_GSM_MASTER to searchPg replica | consumers | [`USER_PNS_IMSDB.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PNS_IMSDB.go) (`UserPnsImsdb`) |
| Consumer — deallocation-reason audit table | consumers | [`USER_PNS_DELETION_REASON.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PNS_DELETION_REASON.go) (`UserPnsDeletionReason`) |

**Koi cron PNS-transaction domain ke liye nahi mila** — na `service-api-go-production/crons`
mein, na `user-temp-consumers-production` mein `*pns*` naam ke kisi standalone script mein
(section 12).

---

## 2. Routes

| Method | Path / serviceName | Repo | Controller |
|---|---|---|---|
| POST | serviceName=`GSM_MASTER_SERVICE` | write | `UserPnsAllocationController` — [`PnsTxnController.go:19`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/PnsTxnController.go) |
| GET/POST | `/obdnumber/*params` | read | `ObdNumber` — registered 3× in [`routerUsers.go:347-348,518-519,705-706`](../../users-api-go-production/internal/api/users_router/routerUsers.go) (looks like duplicated route groups for different auth-tiers, not PNS-specific pattern) |
| GET/POST | `/wservce/glmaster/detail/*params` | read | `GlmasterDetail` — [`routerMaster.go:159-160`](../../users-api-go-production/internal/api/master_router/routerMaster.go) (generic multi-table lookup; `gl_gsm_master` is one of six `detail_from` branches: `gl_gsm_master`, `PrimaryBiz`, `NoofEmp`, `Turnover`, `LegalStatus`, `Biz`, `Disposition`) |
| POST | `/glmaster/detail/*params` (test-router variant) | read (test) | [`routerTest.go:154`](../../users-api-go-production/internal/api/test_router/routerTest.go) — a parallel test-only copy of the same logic, `internal/test_controllers` + `internal/test_models` |

**Correction to previous doc**: "Koi read-controller kisi bhi repo mein nahi mila" thi galat
claim — do genuine read-paths hain (`/obdnumber`, `/wservce/glmaster/detail`), dono
`GL_GSM_MASTER` touch karte hain, dono `users-api-go-production` mein.

---

## 3. Data Model — Table

> **Verification note**: table/column names Go code ke andar embedded SQL strings se liye
> gaye hain. Live DB schema se pgAdmin pe cross-verify **nahi** kiya gaya hai.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GL_GSM_MASTER` | meshpg (write), authPg (replica), searchPg (replica) | Shared pool of GSM/virtual call-tracking numbers, per-vendor | `gl_gsm_number` (PK-like, unique constraint — insert conflicts on it), `gl_gsm_vendor_type`, `flag_is_available` (see section 4 — **six** distinct values seen in validation, not two), `gl_gsm_add_date`, `gl_gsm_mod_date`, `gl_gsm_added_by_empid` — [`PnsTxnModel.go:144,186-188,235-237,423-425`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PnsTxnModel.go) |
| `PNS_DELETION_REASON` | meshPg | Audit table: why/when a GSM number was released from a supplier | `FK_GLUSR_USR_ID`, `GLUSR_IM_GSM_VALUE`, `ADDED_DATE`, `ADDED_BY_EMPID`, `REASON` — [`USER_PNS_DELETION_REASON.go:99,114`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PNS_DELETION_REASON.go) |
| `GLUSR_USR` | meshpg (read side) | `glusr_usr_im_gsm` column holds the GSM number **currently assigned to a supplier's account** — this is the actual "which number does this supplier have" link, joined against `GL_GSM_MASTER` for vendor-type lookup | [`UserOBDNumberModel.go:47`](../../users-api-go-production/internal/models/users/UserOBDNumberModel.go): `SELECT gl_gsm_vendor_type FROM glusr_usr,gl_gsm_master WHERE glusr_usr_id=$1 and glusr_usr_im_gsm=gl_gsm_number` |

**New structural fact missed previously**: the write side never queries/updates
`GLUSR_USR.glusr_usr_im_gsm` itself — `PnsTxnModel.go` only touches `GL_GSM_MASTER`. The
actual "assign this GSM to this supplier's account" write happens in a **separate,
downstream HTTP service** called via `call_to_glusr_update_service()` (section 5, rule 4) —
PNS-transaction only manages the number-pool side, not the account-side assignment column.

---

## 4. `FLAG_IS_AVAILABLE` — Decoding the Magic Values

This is the single most important state field in the domain. Meanings are reconstructed from
how the code branches on them — **no documented enum found**, so several are marked
**[INFERRED — confirm with team]**.

| Value | Meaning | Evidence |
|---|---|---|
| `2` | **Available / unclaimed** in the pool for its vendor-type | `convert_procured_pns_pg`'s subquery selects `WHERE ... flag_is_available = 2 LIMIT 1` before claiming — [`PnsTxnModel.go:235,237`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PnsTxnModel.go) |
| `1` | **Actively assigned to a supplier** (in-use) | Default `FLAG_IS` on insert/update when no explicit value given — [`PnsTxnModel.go:139,179`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PnsTxnModel.go); also set to `"1"` explicitly on both branches of `update_mesh_pg`'s ACTION=3 path (`params["FLAG_IS"], params["FLAG_IS_AVAILABLE"] = "1", "1"` — lines 521, 530) |
| `0` | **[INFERRED]** Reclaimable/returned-but-not-yet-verified variant of "unassigned" for `PNS_TYPE="0"` numbers | `allocate_pns_to_glid_pg`'s `pnstyp=="0"` branch pulls from `WHERE flag_is_available=0` — [`PnsTxnModel.go:423`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PnsTxnModel.go); also set by `convert_procured_pns_pg` when `PNS_TYPE=="0"` (line 234) |
| `-1` | **[INFERRED]** Same as `0` but for `PNS_TYPE="-1"` numbers — a second, parallel "reclaimable pool" bucketed by `PNS_TYPE` | `allocate_pns_to_glid_pg`'s `else` branch pulls from `WHERE flag_is_available=-1` (line 425); set by `convert_procured_pns_pg` when `PNS_TYPE=="-1"` (line 236); also forced by `UpsertPnstxn` line 53 when the downstream glusr-update call fails (`params["FLAG_IS"]="-1"`) |
| `3`, `4`, `5`, `6` | **[INFERRED — not observed being SET anywhere in this codebase]** Accepted as *valid input* by validation (`MandatoryParamsCheckPnsTxn` allows `flgis` ∈ `{-1,0,1,2,3,4,5,6}` and, for ACTION=1/2, ∈ `{-1,0,2,3,4,5,6}` — [`UserUtilsMandatory.go:1683,1687,1691`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)), meaning callers CAN pass these to `insert_gsm_number_on_pg`/`update_gsm_number_on_pg` directly, but no branch in `PnsTxnModel.go` itself sets them — likely other vendor-integration states set by callers outside this file. **Open Question.** |

**`PNS_TYPE`** (separate from `FLAG_IS_AVAILABLE`, only `"0"` or `"-1"` accepted per
validation — [`UserUtilsMandatory.go:1695-1696`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)) appears to be a **vendor/pool-bucket selector**, not a lifecycle state — it picks which of the two reclaim-pools (`0` or `-1`) an operation targets. **[INFERRED]**

**Correction to previous doc**: the earlier pass said "two variant flag-values (`0` vs `-1`)
... distinguished by some input-condition not fully traced." This pass traces it fully: both
are driven by the `PNS_TYPE` input parameter (a caller-supplied bucket selector), consistently
across `allocate_pns_to_glid_pg` and `convert_procured_pns_pg`.

---

## 5. Business Rules & Validation (code se, exhaustive)

1. **Gateway allowlist (~33 entries)**: `MY`, `GLADMIN`, `TOLLFREE`, `Email Marketing`,
   `HTVENDOR`, `Weberp`, `LEAP`, `IPC-Admin`, `NSD Sales`, `MAPI`, `ENQUIRY`,
   `FREE-WEBSITE`, `M.INDIAMART.COM`, `TRADE`, `BL`, `TENDER`, `PAYNOW`,
   `CREDIT ALLOCATION`, `OVP Process`, `SAMPARK Process`, `TOLLFREE Process`,
   `VENDOR CITY Pin Correction`, `Notification Server`, `search`, `FCP`, `PNS`, `Merp`,
   `IMOB`, `SELLERMY`, `PAYWIM`, `BUYERS_FEEDBACK`, `FLPNS`, `LEAPIN`, `ANDROID`, `IOS`.
   [`PnsTxnController.go:52`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/PnsTxnController.go)
2. **Mandatory fields always**: `UPDATEDBY`, `VALIDATION_KEY`, `IP`, `IP_COUNTRY`, `ACTION`,
   `UPDATEDUSING`. `ACTION` must be one of `1`-`5`.
   [`UserUtilsMandatory.go:1667-1670`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
3. **`GSM_NUMBER` format**: if provided and >10 chars, must match
   `^\d{10}(,\d{3,4})?$` (10-digit base + optional comma + 3-4 digit extension, e.g. an
   extension/DID suffix); exactly-10-digit numbers must be purely numeric; under-10 is
   rejected outright.
   [`UserUtilsMandatory.go:1660-1663,1675-1680`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
4. **Vendor gating is currently hardcoded to a single active vendor**: `setvendor := "ALL"`
   is a fixed local variable (not read from config/DB in this function), and the checks
   `(setvendor=="KNOW" && gsmvendor!="KNOW") || ...` for each named vendor only fire when
   `setvendor` literally equals that vendor string — since `setvendor` is always `"ALL"`,
   **none of these per-vendor-lock branches can currently trigger**; they read as dead/
   legacy logic from an earlier single-active-vendor design.
   [`UserUtilsMandatory.go:1581,1671-1674,1681-1682`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
5. **`FLAG_IS`/`FLAG_IS_AVAILABLE` allowed-value set**: `{-1,0,1,2,3,4,5,6}` generally; for
   ACTION `1` and `2` specifically, `1` (assigned) is explicitly disallowed as an *input*
   value (`"PNS can not be inserted/updated as Assigned"`) — assignment (`1`) can only be
   reached internally via the ACTION=3 allocation path, never passed directly by a caller,
   **except** when `PNS_BLOCK=="1"` bypasses this check for ACTION=2.
   [`UserUtilsMandatory.go:1683,1687,1691`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
6. **ACTION `3`/`4` require `USR_ID`** (they operate against a specific supplier's account,
   not just the raw pool). [`UserUtilsMandatory.go:1693-1694`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
7. **ACTION `5` (bulk-convert) requires `LIMIT`, `GSM_VENDOR`, `PNS_TYPE`, both numeric, and
   `LIMIT` must be strictly between `0` and `50`** (string comparison `limit >= "50"` —
   lexicographic, not numeric, but functionally correct for the intended 1-49 range given
   the numeric-check gate already passed).
   [`UserUtilsMandatory.go:1697-1700`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
8. **Five distinct ACTION codes, each a different operation** (this was entirely missing
   from the previous doc, which only documented a generic "allocate" flow):

   | ACTION | Meaning | Code path |
   |---|---|---|
   | `1` | **Add a brand-new GSM number to the pool** (with explicit `GSM_NUMBER`+`GSM_VENDOR`) | `update_mesh_pg` → `insert_gsm_number_on_pg` |
   | `2` | **Update an existing GSM number's flag/vendor** (release/re-flag a specific number back into the pool) | `update_mesh_pg` → `update_gsm_number_on_pg` |
   | `3` (with `GSM_NUMBER`) | **Assign/re-flag a specific already-known GSM number to a supplier** — validates for dup-in-use first | `update_mesh_pg` |
   | `3` (no `GSM_NUMBER`) | **Auto-allocate any available number from the pool** for a vendor-type (random pick) | `allocate_pns_to_glid_pg` |
   | `4` | **Release/replace a supplier's currently-assigned number** — first calls the downstream glusr-update service; only on failure of that call does it locally flag the number `-1` | `call_to_glusr_update_service` → conditionally `update_mesh_pg` |
   | `5` | **Bulk-convert N numbers from "available" to a target flag** for a given vendor+type, up to `LIMIT` (max 50) | `convert_procured_pns_pg` — loops `LIMIT` times, one query per iteration (see Optimization Scope) |
   [`PnsTxnModel.go:46-107`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PnsTxnModel.go)
9. **Duplicate-in-use check before assigning** (ACTION `1`/`3`): `validate_pns_number()`
   calls a shared DB function `FN_CHECK_DUP_GLUSR($1..$9, gsm_number, ..., "IN")` — this is
   a **generic duplicate-check stored procedure reused across the domain** (not GSM-specific
   by name), called here with the GSM number in its 7th positional slot. If it reports a
   count `>0`, the request fails with `"GSM number already exist as assigned."`
   [`PnsTxnModel.go:367-405`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PnsTxnModel.go)
10. **ACTION=3-with-explicit-GSM-number branches on the number's current state vs. the
    requested `PNS_TYPE`**: if the number exists and its current `flag_is_available` matches
    the requested `pnstyp` mapping (`-1`↔`-1`, `0`↔`0`), it's claimed (`pnsFinal=2`, flags set
    to assigned `"1","1"`); if it exists with a mismatched state, the request fails
    (`"GSM number already exist as assigned."`); if it doesn't exist at all, it's inserted
    fresh as assigned (`pnsFinal=1`). [`PnsTxnModel.go:515-533`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PnsTxnModel.go)
11. **ACTION=4 (release) delegates the account-side unassignment to a downstream HTTP
    service first** — `call_to_glusr_update_service()` calls `configData.APIList["user_update_api"]`
    with `IM_GSM=""` (clearing the assignment) via `CurlReqWithRetry` (3 retries, 2s timeout,
    200ms backoff) wrapped in a circuit-breaker (`config.CB_api.Execute`). **Only if that
    downstream call reports failure** does `UpsertPnstxn` locally flag the GSM number `-1` in
    `GL_GSM_MASTER` as a fallback — meaning the *normal* success path for release does **not**
    touch `GL_GSM_MASTER` directly at all; it relies entirely on the downstream service.
    [`PnsTxnModel.go:50-56,268-365`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PnsTxnModel.go)
12. **On downstream-call failure or a non-"Update Success" response, an error email fires**
    to `soa.user@indiamart.com,pnsteam@indiamart.com` (prod only) with the raw input params
    and downstream response embedded. [`PnsTxnModel.go:330-346`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PnsTxnModel.go)
13. **A "PNS deletion" RabbitMQ event fires only in two specific cases**: ACTION=4 with a
    non-empty `REASON`, or ACTION=3 (regardless of reason) — see section 7 for the queue.
    [`PnsTxnModel.go:58-81`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PnsTxnModel.go)
14. **`userupdate_pns_allocation` (triggered when ACTION=3 succeeds AND the newly-claimed
    row's `FLAG_IS_AVAILABLE=="1"`) additionally calls the downstream glusr-update service
    again, then — if that succeeds — flags the *previous* GSM number (`rPns`, returned by the
    downstream service) back to available (`FLAG_IS=1`, ACTION=2)** — i.e. this is the code
    path that actually swaps a supplier's number: assign new, then release old, both via the
    same downstream service + local pool flag. [`PnsTxnModel.go:83-95,560-592`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PnsTxnModel.go)
15. **The originally-documented `PNS_SERVICE` RabbitMQ publish is DEAD CODE** — see section 7,
    this is the single biggest correction to the previous doc.

---

## 6. Data Flow Diagram — 5 Actions at a Glance

```
ACTION=1  →  update_mesh_pg → insert_gsm_number_on_pg           (add brand-new number)
ACTION=2  →  update_mesh_pg → update_gsm_number_on_pg           (re-flag existing number)
ACTION=3, GSM_NUMBER given   → update_mesh_pg (validate + insert/update)  (assign specific number)
ACTION=3, GSM_NUMBER empty   → allocate_pns_to_glid_pg                    (auto-pick from pool)
              └─ if claimed row was already "1" → userupdate_pns_allocation (swap old→new)
ACTION=4  →  call_to_glusr_update_service (release via downstream)
              └─ only on downstream FAILURE → local fallback update_mesh_pg (flag -1)
ACTION=5  →  convert_procured_pns_pg  (bulk pool-conversion, up to 50 rows, looped queries)
```

---

## 7. RabbitMQ

| Queue / `SERVICENAME` | Publisher | Consumer(s) | Purpose | Status |
|---|---|---|---|---|
| `PNS_SERVICE` → routing key `user.pns.*` on exchange `USER.topic` | `PnsTxnModel.go` builds this message (`rabbitmqdata`, lines 18-27) on every `UpsertPnstxn` call | [`USER_PNS_AUTHPG.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PNS_AUTHPG.go) (queue `USER_PNS_AUTH_PG` → authPg) and [`USER_PNS_IMSDB.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PNS_IMSDB.go) (queue `USER_PNS_IMSDB` → searchPg) both expect this exact message shape (`MESSAGE[0].COLUMNS.ACTION` ∈ `{1,2,5}`) to replicate `GL_GSM_MASTER` changes | **DEAD — the actual `utils.PushToQueue(...)` call that would send this message is commented out** ([`PnsTxnModel.go:109-125`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PnsTxnModel.go), the whole `if len(cassparams) > 0 { ... }` block is `//`-commented). **The two replica-sync consumers are therefore currently starved of live messages** — they're wired up (registered in `QueueFuncMap`, [`router.go:33,36`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Router/router.go)) but nothing publishes to them anymore in this codebase. See Edge Cases #1 and Open Questions #1. |
| `PNS_DELETION` → queue `USER_PNS_DELETION_REASON` (direct queue, no exchange) | `PnsTxnModel.go:60-81`, fires on ACTION=4-with-reason or ACTION=3 | [`USER_PNS_DELETION_REASON.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PNS_DELETION_REASON.go) | **Live.** ACTION=4 → `INSERT INTO PNS_DELETION_REASON`; ACTION=3 → `DELETE FROM PNS_DELETION_REASON WHERE FK_GLUSR_USR_ID=$1` (i.e. a fresh allocation clears any prior deletion-reason record for that supplier) — [`USER_PNS_DELETION_REASON.go:98-116`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PNS_DELETION_REASON.go) |

**Routing config source**: [`rabbitmq.go:40,42,73`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go)
(`serviceToQueueMap` and `exchangeSet` in `PushToQueue`).

**Correction to previous doc**: the earlier pass reported the `PNS_SERVICE` publish as live
and its consumer as "not confirmed." Both halves are now traceable — the consumers exist and
are wired up, but the publish call itself is commented-out dead code. This is a materially
different and more important finding than either half of the old doc's guess.

---

## 8. Kafka

**No Kafka usage found** in `PnsTxnController.go`, `PnsTxnModel.go`, or any of the three
consumer files (`USER_PNS_AUTHPG.go`, `USER_PNS_IMSDB.go`, `USER_PNS_DELETION_REASON.go`) —
grepped directly, zero matches. All PNS-transaction messaging is RabbitMQ-only (live or dead,
per section 7).

---

## 9. Redis

**No Redis usage found** anywhere in the write path (`PnsTxnModel.go`) or the two read paths
(`UserOBDNumberModel.go`, `GlmasterDetail.go`) — grepped directly, zero matches. Both reads hit
Postgres live on every request; `OBDNumber` additionally reads a local JSON file
(`pkg/utils/vendor_obdnumber.json`) from disk per-request as a fallback/primary vendor→OBD-number
map, with a further hardcoded in-code fallback map if the file read fails
([`UserOBDNumberModel.go:104-173`](../../users-api-go-production/internal/models/users/UserOBDNumberModel.go)) — this is a caching-adjacent gap worth flagging (see Optimization Scope).

---

## 10. End-to-End Technical Flows

### Flow 1 — ACTION=3, auto-allocate a number from the pool (the "classic" PNS allocation)

```
Internal-tool (~33-entry allowlist)
    │
    ▼
[API — write]  POST serviceName=GSM_MASTER_SERVICE  {ACTION:3, USR_ID, GSM_VENDOR, PNS_TYPE, no GSM_NUMBER}
    │  MandatoryParamsCheckPnsTxn() — length/format/vendor/action checks
    │  Gateway_v1() allowlist check
    ▼
UpsertPnstxn()  — action=="3" && gsmnum==""
    ▼
allocate_pns_to_glid_pg()
    │  UPDATE gl_gsm_master SET flag_is_available=1, ... WHERE gl_gsm_number =
    │    (SELECT gl_gsm_number FROM gl_gsm_master WHERE flag_is_available IN {0,-1 per PNS_TYPE}
    │     AND gl_gsm_vendor_type = COALESCE($vendor, gl_gsm_vendor_type) ORDER BY random() LIMIT 1)
    │  RETURNING gl_gsm_number, gl_gsm_vendor_type, gl_gsm_add_date, flag_is_available
    ▼
[DB — meshpg]
    │
    ├─ if claimed row's returned flag was already "1" (edge case, see Edge Cases #3):
    │     userupdate_pns_allocation()
    │       ├─ call_to_glusr_update_service() — assign new number on supplier's account (downstream HTTP)
    │       └─ if success → update_mesh_pg() flags the OLD number back to available (ACTION=2, FLAG_IS=1)
    │
    ▼
[RabbitMQ]  SERVICENAME=PNS_DELETION → queue USER_PNS_DELETION_REASON
    │  fires because action=="3" unconditionally
    │  → DELETE FROM PNS_DELETION_REASON WHERE FK_GLUSR_USR_ID=$1 (clear stale reason record)
    ▼
Response {"SUCCESS", GSM: <newly allocated number>, VENDOR: <vendor>}
```

**Note**: `[RabbitMQ] PNS_SERVICE` (replica-sync) does NOT fire — that publish is dead code
(section 7). So `GL_GSM_MASTER`'s replicas on authPg/searchPg do **not** get this allocation
reflected via this path currently.

### Flow 2 — ACTION=4, release a supplier's currently-assigned number

```
Internal-tool
    │
    ▼
[API — write]  POST serviceName=GSM_MASTER_SERVICE  {ACTION:4, USR_ID, REASON (optional)}
    │  Validation + Gateway check
    ▼
UpsertPnstxn()  — action=="4" && usrid != ""
    ▼
call_to_glusr_update_service()
    │  POST to configData.APIList["user_update_api"]  {IM_GSM:"", USR_ID, UPDATEDBY, ...}
    │  (via CB_api circuit breaker, 3 retries / 2s timeout / 200ms backoff)
    ▼
[Downstream HTTP service — clears GLUSR_USR.glusr_usr_im_gsm]
    │
    ├─ SUCCESS ("Update Success") → updatefailed=0
    │     → GL_GSM_MASTER is NOT touched by this request at all (no local flag change)
    │     → if REASON given: [RabbitMQ] PNS_DELETION → INSERT INTO PNS_DELETION_REASON
    │
    └─ FAILURE (any other response / error) → updatefailed=1
          → ErrorMail() to soa.user@indiamart.com,pnsteam@indiamart.com (prod only)
          → local fallback: update_mesh_pg() flags the returned old GSM number -1 in GL_GSM_MASTER
          → if REASON given: [RabbitMQ] PNS_DELETION → INSERT INTO PNS_DELETION_REASON
```

**Business-logic quirk worth flagging**: the boolean naming is inverted from what it looks
like — `updatefailed==1` in `call_to_glusr_update_service` actually means the downstream call
**succeeded** (`"Update Success"`), and `updatefailed==0` means it failed. This is confirmed by
reading the `if err != nil || result["MESSAGE"] != "Update Success"` branch (treated as the
failure path) vs. the `else` branch (treated as success, sets `updatefailed=1`).
[`PnsTxnModel.go:330-362`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PnsTxnModel.go)
This is exactly backwards-sounding and a likely source of future maintenance bugs.

### Flow 3 — ACTION=5, bulk-convert numbers from available to a target flag

```
Internal-tool
    │
    ▼
[API — write]  POST serviceName=GSM_MASTER_SERVICE  {ACTION:5, LIMIT (1-49), GSM_VENDOR, PNS_TYPE (0 or -1)}
    ▼
UpsertPnstxn()  — action=="5"
    ▼
convert_procured_pns_pg()
    │  for i := 1 to LIMIT:
    │      UPDATE gl_gsm_master SET flag_is_available = {0 or -1 per PNS_TYPE}, gl_gsm_mod_date=now()
    │      WHERE gl_gsm_number = (SELECT gl_gsm_number FROM gl_gsm_master
    │                              WHERE gl_gsm_vendor_type=$1 AND flag_is_available=2 LIMIT 1)
    │      RETURNING gl_gsm_number
    │      (one DB round-trip PER iteration — see Optimization Scope #1)
    ▼
Response {"SUCCESS", recordsUpdated: <count>, PNS_STR: "<comma-joined numbers>"}
```

### Flow 4 — Reads: OBD number lookup and generic GSM-vendor lookup

```
[API — read]  GET/POST /obdnumber/*params  {GLUSR_ID}
    │  ObdNumber() — token/GLUSR_ID/MODID validity check
    ▼
OBDNumber()  (users.OBDNumber)
    │  SELECT gl_gsm_vendor_type FROM glusr_usr, gl_gsm_master
    │    WHERE glusr_usr_id=$1 AND glusr_usr_im_gsm = gl_gsm_number
    │  (this is the ONLY place in the read path that connects a supplier's
    │   account-level assigned GSM number back to GL_GSM_MASTER's vendor-type)
    ▼
[if vendor_type found]  read local file pkg/utils/vendor_obdnumber.json (per-request disk read)
    │  → map vendor_type to an OBD (Outbound-Dial) phone number
    │  → on file-read failure or empty file, fall back to a hardcoded in-code vendor→number map
    ▼
Response {"DATA_FOUND", Data: {<vendor_type>: <obd_number>}}
```

```
[API — read]  GET/POST /wservce/glmaster/detail/*params  {detail_from: "gl_gsm_master", gsm_num}
    ▼
GlmasterDetail()  (master.GlmasterDetail)
    │  SELECT gl_gsm_vendor_type FROM gl_gsm_master WHERE GL_GSM_NUMBER=$1
    ▼
Response {"Success", Data: {GL_GSM_MASTER: {vendor_type: <...>}}}
```

---

## 11. Flow-wise DB & Table Usage

### Flow 1 — ACTION=3 auto-allocate

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1 | meshpg | `GL_GSM_MASTER` | UPDATE ... RETURNING (subquery claim) | Randomly picks and claims one available number for the vendor+type, in a single round-trip |
| 2 (conditional) | meshpg | `GL_GSM_MASTER` | UPDATE (via `update_mesh_pg`/`update_gsm_number_on_pg`) | Only if the claimed row was already flagged `1` — flips the *previous* number back to available after a successful downstream swap |
| 3 (external) | downstream `user_update_api` HTTP service | — | POST | Assigns the new GSM number on the supplier's account (`GLUSR_USR.glusr_usr_im_gsm`) — this repo never writes that column directly |

### Flow 2 — ACTION=4 release

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1 (external) | downstream `user_update_api` HTTP service | — | POST | Clears the supplier's assigned GSM number on their account |
| 2 (conditional, only on downstream failure) | meshpg | `GL_GSM_MASTER` | UPDATE (via `update_mesh_pg`) | Fallback: locally flag the number `-1` since the downstream service couldn't confirm the release |
| 3 (conditional, only if `REASON` given) | meshPg (consumer side) | `PNS_DELETION_REASON` | INSERT | Audit record: why/when this number was released |

### Flow 3 — ACTION=5 bulk-convert

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1..N (N = `LIMIT`, up to 49) | meshpg | `GL_GSM_MASTER` | UPDATE ... RETURNING (subquery claim), **one query per requested record, in a Go `for` loop** | Converts up to `LIMIT` available numbers from `flag_is_available=2` to the target flag — no batch/set-based SQL, see Optimization Scope #1 |

### Flow 4 — Reads

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1 | mesh_pg_user (read-side connection) | `glusr_usr` JOIN `gl_gsm_master` | SELECT | `/obdnumber` — resolves a supplier's assigned GSM number to its vendor-type |
| 2 (conditional) | local filesystem, not a DB | `pkg/utils/vendor_obdnumber.json` | file read | Maps vendor_type → OBD phone number; per-request, no cache |
| 3 | mesh_pg_with_product_user (read-side connection) | `gl_gsm_master` | SELECT | `/wservce/glmaster/detail` — resolves a raw GSM number to its vendor-type (generic lookup, GSM is one of 6 supported `detail_from` types in this shared endpoint) |

**Total DB round-trips for a typical single-number allocation (Flow 1, common case)**:
1 (allocate) — this is a genuinely lean single-query write, unlike GST's 9-10 round-trip
fan-out. The complexity in this domain is in the **conditional branching across 5 ACTION
types**, not in round-trip count per request.

---

## 12. Cron Inventory

**No PNS-transaction-specific cron found.** Searched `user-temp-consumers-production` for any
`*pns*`-named standalone script (the pattern used for GST's `gst_tact_cron_sync.go`/
`gst_tact_veri_cron.go`) — none exists. The three PNS-domain workers
(`USER_PNS_AUTHPG`, `USER_PNS_IMSDB`, `USER_PNS_DELETION_REASON`) are all live-queue
consumers, not scheduled jobs. **[Confirmed absence, not assumed.]**

---

## 13. Edge Cases & Gotchas (technical POV)

1. **The replica-sync RabbitMQ publish is commented-out dead code** (section 7) — the two
   consumers `USER_PNS_AUTH_PG` and `USER_PNS_IMSDB` are deployed and listening, but nothing
   publishes to `user.pns.*` anymore from `PnsTxnModel.go`. If `authPg`/`searchPg` copies of
   `GL_GSM_MASTER` are stale vs. `meshpg`, this dead code is almost certainly why. Worth
   confirming with the team whether this was disabled deliberately (e.g. replaced by a
   different sync mechanism) or accidentally left commented during a debugging session — the
   surrounding comment style (`// rabbitmqdata[...]`) suggests the latter.
2. **Inverted boolean naming**: `updatefailed==1` means the downstream call **succeeded**;
   `updatefailed==0` means it **failed** (section 10, Flow 2). Any future change to this
   function needs to read the actual condition, not trust the variable name.
3. **`allocate_pns_to_glid_pg` can theoretically claim a row that's already flagged `1`** —
   the WHERE clause filters on `flag_is_available IN {0,-1}` depending on `PNS_TYPE`, so in
   practice it shouldn't return an already-assigned row, but the subsequent code
   (`UpsertPnstxn` line 83) explicitly checks `cassparams["FLAG_IS_AVAILABLE"] == "1"` as if
   this were a real possibility worth handling — suggesting either historical bug-scarring or
   a race condition the team has seen in production. **[INFERRED — confirm with team.]**
4. **No `FOR UPDATE SKIP LOCKED`** on any of the pool-claiming subqueries
   (`allocate_pns_to_glid_pg`, `convert_procured_pns_pg`) — same concurrency-race caveat as
   the previous doc flagged, still applies, now confirmed present in **two** separate
   allocation code paths (not just one).
5. **ACTION=5's loop-of-N-queries** (section 11, Flow 3) means a `LIMIT=49` request makes 49
   sequential DB round-trips in a single HTTP request — no batching, no parallelism.
6. **`vendor_obdnumber.json` is read from disk on every `/obdnumber` request**, with
   `os.Getwd()`-relative pathing — fragile if the working directory ever changes across
   deployment environments, and definitely a per-request I/O cost with no caching.
7. **The `FLAG_IS_AVAILABLE` values `3`-`6` are accepted as valid input but never set by any
   code in this file** — if they're meaningful (e.g. set by a different, undocumented
   caller), this doc cannot confirm their semantics (section 4).
8. **`GlmasterController`'s hardcoded token** `const token = "imobile@15061981"` is declared
   in [`detail.go:28`](../../users-api-go-production/internal/controllers/GlmasterController/detail.go)
   but this pass did not fully trace whether/how it's compared against the request's `token`
   param — flagging for review since hardcoded auth tokens in source are a security-hygiene
   concern regardless of whether they're actively enforced.
9. **Vendor-lock validation appears to be dead logic** (section 5, rule 4) — `setvendor` is
   hardcoded to `"ALL"`, so branches checking `setvendor=="KNOW"` etc. can never fire. If the
   team intends per-vendor gating to be active, this is currently broken; if not, it's
   harmless dead code but worth cleaning up.
10. **Duplicate route registrations**: `/obdnumber/*params` is registered three times at
    different line ranges in `routerUsers.go` (347-348, 518-519, 705-706) — likely reflects
    multiple route groups with different middleware/auth tiers in the same router file, not
    confirmed conclusively in this pass.

---

## 14. Optimization Scope — DB Response-Time Contribution

### High-impact

1. **ACTION=5's bulk-convert loops one query per record instead of a set-based UPDATE.**
   `convert_procured_pns_pg` runs up to 49 sequential `UPDATE ... RETURNING` round-trips in a
   single HTTP request (section 11, Flow 3). **Concrete fix**: replace with a single
   set-based query using a CTE, e.g.
   `WITH picked AS (SELECT gl_gsm_number FROM gl_gsm_master WHERE gl_gsm_vendor_type=$1 AND
   flag_is_available=2 LIMIT $2 FOR UPDATE SKIP LOCKED) UPDATE gl_gsm_master SET
   flag_is_available=$3 WHERE gl_gsm_number IN (SELECT gl_gsm_number FROM picked) RETURNING
   gl_gsm_number` — this collapses up to 49 round-trips into 1, and also fixes the
   concurrency gap (Edge Case #4) via `FOR UPDATE SKIP LOCKED` in the same change.
   [`PnsTxnModel.go:225-266`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PnsTxnModel.go)

2. **The downstream `user_update_api` HTTP call sits in the critical path for ACTION=3
   (swap) and ACTION=4 (release), wrapped in a circuit breaker but still doing up to 3
   retries with a 2s timeout each** — worst case ~6+ seconds added to response time if the
   downstream service is slow/down. Since this call determines whether `GL_GSM_MASTER` even
   gets touched (Flow 2), a slow downstream service directly inflates PNS-transaction
   response times. Worth confirming the actual observed p99 for `user_update_api` and whether
   the retry/timeout budget here is appropriate for a synchronous request path.
   [`PnsTxnModel.go:308-324`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PnsTxnModel.go)

### Medium-impact

3. **No `FOR UPDATE SKIP LOCKED` on either pool-claiming query** (`allocate_pns_to_glid_pg`,
   `convert_procured_pns_pg`) — under concurrent-allocation load, requests can contend on the
   same candidate row; Postgres's MVCC prevents actual double-allocation of the same row via
   the subquery-in-UPDATE pattern, but throughput degrades under contention as competing
   transactions wait/retry. Same fix as optimization #1 applies here too.

4. **`/obdnumber` reads a JSON file from disk on every request** with no in-memory cache —
   `vendor_obdnumber.json` almost certainly changes rarely (it's a static vendor→number
   mapping). **Concrete fix**: load it once at startup into memory, or at minimum cache with
   a TTL, eliminating the per-request disk I/O.
   [`UserOBDNumberModel.go:104-119`](../../users-api-go-production/internal/models/users/UserOBDNumberModel.go)

### Low-impact / good-practice already present

5. **Flow 1's common-case allocation is a single DB round-trip** (`allocate_pns_to_glid_pg`'s
   one `UPDATE ... RETURNING`) — this is already a lean, well-optimized query shape for the
   simple case. The complexity/cost in this domain lives in the downstream HTTP dependency
   and the ACTION=5 loop, not in excessive DB chattiness for the core allocation path.

---

## 15. Open Questions

1. Is the commented-out `PNS_SERVICE` RabbitMQ publish (section 7) intentionally disabled, or
   an accidental leftover from debugging? If intentional, are `authPg`/`searchPg` copies of
   `GL_GSM_MASTER` kept in sync some other way now, or are they simply stale/deprecated?
2. What do `FLAG_IS_AVAILABLE` values `3`, `4`, `5`, `6` mean (section 4)? Accepted as valid
   input by validation but never set anywhere in `PnsTxnModel.go` — likely set by a caller or
   process outside this file/repo.
3. Is the vendor-lock validation logic (section 5, rule 4 — currently dead because
   `setvendor` is hardcoded to `"ALL"`) intentionally disabled, or a bug?
4. What exactly is `GlmasterController`'s hardcoded token (`imobile@15061981`,
   [`detail.go:28`](../../users-api-go-production/internal/controllers/GlmasterController/detail.go))
   used for, and is it still actively enforced/needed?
5. Can `allocate_pns_to_glid_pg` genuinely claim a row already flagged `1` in production
   (Edge Case #3)? The defensive check in `UpsertPnstxn` suggests this has happened.
6. Live DB schema verification (column types, nullability, indexes, constraints on
   `GL_GSM_MASTER` and `PNS_DELETION_REASON`) — this doc only reflects what Go SQL strings
   imply.

---

## See also

- [`PNS_Business_Doc.md`](./PNS_Business_Doc.md) — product perspective
- [`../PNS Setting KT/PNS_Setting_Technical_Doc.md`](../PNS%20Setting%20KT/PNS_Setting_Technical_Doc.md) —
  unrelated concept, documented separately (no shared table/FK/controller, one incidental
  read-side join for a `PNS_MARKED` flag noted at the top of this doc)
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — depth/structure
  reference this doc was rebuilt against
- [`../user_consumers_reference.md`](../user_consumers_reference.md) — poori consumer
  inventory, PNS section confirms the three workers documented here

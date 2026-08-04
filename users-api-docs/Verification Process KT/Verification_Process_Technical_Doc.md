# Verification Process (Attribute-Verification-Event Write) — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`Verification_Process_Business_Doc.md`](./Verification_Process_Business_Doc.md)
dekho.

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file path diya
gaya hai har jagah). Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — team se confirm karo]** likh diya hai, guess nahi kiya.

**Repos**: `service-api-go-production` (write), `users-api-go-production` (read),
`user-temp-consumers-production` (the RabbitMQ trigger — see section 6).

---

## Scope note — is doc ka *actual* boundary (important, deeper finding than pehli pass)

**Yeh doc sirf ek cheez apni "own" scope maanta hai**: the write path
`UserVerificationDetailsController.go` + `UserVerificationDetailsModel.go`
(`POST /user/userverificationdetails`, serviceName `TRUST_ATTR_VERI_DETAILS`) — jo
structured, primary+secondary-cross-linked verification-events ko `trustpg` ke stored
function `fn_insert_glusr_attribute_verification` ke through likhta hai.

**Yeh already `Trust Verification KT` mein bhi mention hai** — us KT ke
[`Trust_Verification_Technical_Doc.md`](../Trust%20Verification%20KT/Trust_Verification_Technical_Doc.md)
section 3/4 mein yeh exact controller "Trust-DB Attribute-Details Write — The Adjacent Path"
ke naam se already documented hai, attribute-value-map table samet, lekin explicitly
**"poora re-read is pass mein"** flag ke saath — yaani us KT-pass ne isse summarize kiya,
poori depth is doc (Verification Process KT) ke liye chhod di. Is pass mein humne
poora re-read kiya, aur ek naya, pehle na-mila cross-link discover kiya (section 6 dekho):
**yeh controller sirf WEBERP/SOA-WAPI se directly hit nahi hota — yeh ek internal RabbitMQ
consumer (`USER_VERIFICATION_TRUSTPG`) ka HTTP-loopback target bhi hai**, jo GST-verification
jaisi upstream flows se trigger hota hai. Pehli pass ka "Koi RabbitMQ nahi mila" claim
**is doc mein correct kiya gaya hai** — RabbitMQ khud is controller ke andar publish nahi
hota, lekin yeh controller khud ek RabbitMQ-consumer ka downstream target hai, jo effectively
isse messaging-driven bhi banata hai (indirectly).

**Read side is doc ki "own" scope mein NAHI hai**: `GetVerificationDetailController.go`
(`GET_ATTR_VERI_DETAILS`, stored functions `fn_get_glusr_attr_verify`/`fn_get_glusr_attributes`
on `trustpg`) aur `VerifiedDetailController.go`/`VerifiedDetailModel.go`
(`USERVERIFIEDDETAIL`, table `iil_verification_details` on `mesh_pg_user`) — dono **already
poori depth ke saath Trust Verification Technical Doc mein documented hain** (uss doc ke
section 3, "Verification read" row, aur sections jahan `fn_get_glusr_attr_verify` line
219-220 pe cover hota hai). Yeh dono read-endpoints is Verification Process feature ke liye
**exclusive nahi hain** — yeh Trust Verification KT ke `UserVerificationController`
(`POST /user/verification`) se likhe gaye data ko bhi wahi padhte hain. Isliye is doc mein
inhe sirf reference/cross-link ke roop mein rakha gaya hai, duplicate documentation nahi ki
gayi — dekho section 3 aur Open Questions.

**Bottom line relationship** (poochne layak sawaal ka jawaab): yeh **ek genuinely distinct
write-path hai** (alag validation, alag stored function, alag attribute-ID numbering scheme —
Trust_Verification_Technical_Doc.md §3 confirm karta hai `2721`=GST yahan vs `2106` waha alag
hain), **lekin same overall "verification" architecture ka hi ek downstream/adjacent hissa
hai** — is controller ko GST-verification chain khud call karti hai (section 6). Dono KTs
alag write-mechanisms document karte hain jo same `trustpg` verification-store ko target karte
hain, alag entry-points (external API-caller vs internal queue-triggered) se.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write (this feature's own scope) | write | [`UserVerificationDetailsController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserVerificationDetailsController.go), [`UserVerificationDetailsModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go) (`UserVerificationDetailsValidation`, `handlePrimaryAndSecondaryInsertions`, `upsertSellerDetails`, `prepareInputParamsMap`, `validateAttribute`, `validateSecondaryAttrs`) |
| RabbitMQ trigger (internal loopback caller) | consumers | [`USER_VERIFICATION_TRUSTPG.go`](../../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VERIFICATION_TRUSTPG.go) (`UserVerificationTrustPg`, `dbActionUserVerifyTrustPg`) |
| Read (shared with Trust Verification KT, not exclusive to this feature — reference only) | read | [`GetVerificationDetailController.go`](../internal/controllers/UsersControllers/GetVerificationDetailController.go) (`GetVerificationDetails`), [`GetVerificationDetailModel.go`](../internal/models/users/GetVerificationDetailModel.go) (`ActionGtVeriModel`), [`VerifiedDetailController.go`](../internal/controllers/UsersControllers/VerifiedDetailController.go) (`VerifiedDetailController`), [`VerifiedDetailModel.go`](../internal/models/users/VerifiedDetailModel.go) (`UserVerifiedDetail_v1`) — **poori depth ke liye** [`../Trust Verification KT/Trust_Verification_Technical_Doc.md`](../Trust%20Verification%20KT/Trust_Verification_Technical_Doc.md) dekho |

---

## 2. Routes

| Method | Path | serviceName | Repo | Controller |
|---|---|---|---|---|
| POST | `/user/userverificationdetails` | `TRUST_ATTR_VERI_DETAILS` | write | `UserVerificationDetailsController` — [`router.go:166,334`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) (do jagah registered — alag route-group/environment ke liye) |
| GET/POST | `getverificationdetails/*params` | `GET_ATTR_VERI_DETAILS` | read | `GetVerificationDetails` — [`routerUsers.go:174-175`](../internal/api/users_router/routerUsers.go) |
| GET/POST | `/verifieddetail/*params` | `USERVERIFIEDDETAIL` | read | `VerifiedDetailController` — [`routerUsers.go:404-405`](../internal/api/users_router/routerUsers.go) (3 baar registered alag route-groups mein: lines 404-405, 550-551, 741-742) |

*(Internal-only, no HTTP route)*: `USER_VERIFICATION_TRUSTPG` RabbitMQ queue → consumer calls
`POST /user/userverificationdetails` internally via `utils.CallWapiService` — dekho section 6.

---

## 3. Data Model

### 3.1 Attribute-ID → Category mapping (`attributeValueMap`, code se)

| `attributeValueMap` key(s) | Group | `attributeValueMap` key(s) | Group |
|---|---|---|---|
| `121`, `48`, `1293` | Mobile (`1`) | `157`, `109`, `1285` | Email (`2`) |
| `2721` | GST (`3`) | `348` | PAN (`4`) |
| `346` | CIN (`5`) | `347` | TAN (`6`) |
| `111` | Company Name (`7`) | `390` | Bank Account Number (`8`) |
| `2120` | Aadhar Number (`9`) | `112`, `113`, `1278`, `1279` | Address (`10`) |
| `114` | City (`11`) | `115` | State (`12`) |
| `106`, `108` | First/Last Name (`13`) | `352` | IEC Code (`14`) |
| `141`, `142` | CEO First/Last Name (`15`) | `117` | Zip (`16`) |
| `2882` | Udyam (`17`) | `2072`, `2073` | Latitude-Longitude (`18`) |

[`UserVerificationDetailsModel.go:11-40`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go) —
this is a hardcoded Go `map[int]int`, not sourced from a DB lookup-table, meaning any new
attribute-ID requires a code-deploy to support. **Important**: `2721` (GST) here is a
*different numbering scheme* from `2106` (GST's `FK_GL_ATTRIBUTE_ID` in the main
`gl_attribute`/`IIL_VERIFICATION_DETAILS` system used by Trust Verification's
`UserVerificationController`) — confirmed cross-referenced in
[`Trust_Verification_Technical_Doc.md:104-107`](../Trust%20Verification%20KT/Trust_Verification_Technical_Doc.md);
do not conflate the two ID-spaces.

### 3.2 Underlying storage

No `INSERT`/`UPDATE` SQL literal — the write goes entirely through a **stored PL/pgSQL
function**: `SELECT fn_insert_glusr_attribute_verification($1..$11) AS status` on
**`trustpg`**. The function's internal table-structure isn't visible from the Go code — it's
a black-box RPC-style call. Params passed (positional, `upsertSellerDetails`):
`glusrID, pAttrID, pAttrVal, sAttrID, sAttrVal, platform, status, verifyType, datetime,
sourceID, 0` (last param is a hardcoded literal `0`, purpose not documented in code).
[`UserVerificationDetailsModel.go:257-266`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go)

> **Verification note**: table/column names in this doc (and section 3.1's category names)
> come from Go code identifiers and comments, not a live-checked `trustpg` schema — the
> stored function itself is opaque from the Go side. Live DB/function-body verification
> should happen before using this doc for a migration or incident response.

---

## 4. Business Rules & Validation (code se, exhaustive)

1. **Gateway allowlist**: `WEBERP`, `SOA-WAPI` only — narrower than most write-endpoints in
   this codebase (`Gateway_v1([]string{"WEBERP", "SOA-WAPI"}, validationKey, "k")`).
   [`UserVerificationDetailsController.go:43`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserVerificationDetailsController.go)
2. **Two-stage validation before the core logic runs**: first `utils.LengthAndTypeValidations_v3`
   against `UserModels.UserDetailsMap` (generic length/type map reused from the detail-editor
   feature, `ConcatMap`'d in), *then* `UserVerificationDetailsValidation`'s own
   `prepareInputParamsMap` does a second, stricter pass.
   [`UserVerificationDetailsController.go:47-71`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserVerificationDetailsController.go)
3. **`prepareInputParamsMap` strictly type-checks every field** — `glid`/`status`/
   `primaryAttrID` must be JSON-numbers (`float64`, i.e. this endpoint expects a real
   JSON body, not form-encoded strings like most other controllers in this codebase);
   `fk_ref_id` defaults to `primaryAttrID` if not explicitly supplied and must be
   ≤10-digits and non-negative; `datetime` must match `yyyyMMddHHmmss` exactly
   (`time.Parse("20060102150405", val)`); `secondaryAttr` (if present) must be a
   `map[string]interface{}` with numeric-string keys.
   [`UserVerificationDetailsModel.go:57-170`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go)
4. **`primaryAttrID` and every key of `secondaryAttr` must exist in
   `attributeValueMap`** — a single invalid secondary-attribute-key fails the ENTIRE
   secondary-attribute-set (`allValid=false` short-circuits the whole batch), even if
   other secondary-attributes in the same request were valid.
   [`UserVerificationDetailsModel.go:179-190,201-206`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go)
5. **`primaryAttrVal` must be non-empty** — if empty, `handlePrimaryAndSecondaryInsertions`
   silently does nothing at all (no insert of any kind, primary or secondary), and the
   overall call still returns `"SUCCESS"` because `output` stays `""` (no error string was
   set). [`UserVerificationDetailsModel.go:234,251-254`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go)
   — flagged as an edge case in section 9.
6. **Primary-attribute-only insert only happens if there are ZERO secondary
   attributes** (`if len(input.SecondaryAttr) == 0`) — if secondary-attributes ARE
   present, the primary-alone record is **not** separately inserted; instead, one
   combined primary+secondary record is inserted per secondary-attribute.
   [`UserVerificationDetailsModel.go:234-249`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go)
7. **Empty-string secondary-attribute-values are silently skipped** (`if secAttrVal !=
   ""`) — no error, just no insert for that one entry.
   [`UserVerificationDetailsModel.go:242`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go)
8. **`upsertSellerDetails` reformats `datetime`**: if the supplied `yyyyMMddHHmmss`
   string fails to parse (already validated earlier in `prepareInputParamsMap`, so this is
   mostly a safety-net), falls back to `time.Now()`; otherwise reformats to
   `YYYY-MM-DD HH:MM:SS` for the stored-function call.
   [`UserVerificationDetailsModel.go:257-264`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go)
9. **Stored function's return-contract**: expects a `status` field in the response-row;
   `status != 1` is treated as a failure regardless of what the actual returned value
   means — the Go code has no visibility into *why* the stored function rejected the
   call, only a pass/fail signal.
   [`UserVerificationDetailsModel.go:275-284`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go)
10. **Per-request DB timeout is 1 second** (`utils.GetRowDataSqlContext(dbConn, query,
    params, 1*time.Second)`) for every `upsertSellerDetails` call — with N secondary
    attributes this is N separate 1-second-timeout-budgeted calls in sequence, not one
    shared budget for the whole request.
    [`UserVerificationDetailsModel.go:268`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go)

---

## 5. Redis

**Koi Redis usage nahi mila** in `UserVerificationDetailsController.go`/`Model.go` — koi
`RedisGet`/`RedisSet`-jaisi call grep se nahi mili. Purely synchronous DB-only write.

---

## 6. RabbitMQ — Yeh Controller Ek Consumer Ka Loopback-Target Bhi Hai

**Poori Verification Process write-path RabbitMQ publish khud nahi karti** (no `PushToQueue`/
`PubAPI` call anywhere in `UserVerificationDetailsController.go`/`Model.go`) — is baat mein
pehli pass sahi thi. **Lekin yeh controller khud ek RabbitMQ consumer ka HTTP-loopback target
hai**, jo is pass mein naya discover hua:

| Queue | Publisher(s) | Consumer | Kya karta hai |
|---|---|---|---|
| `USER_VERIFICATION_TRUSTPG` | `USER_BULK_VERIFICATION.go:751`, `USER_GST_DETAILS.go:193-201,654-662` (requeue on partial failure), `USER_GST_LAST_MODIFIED.go:726-734,2390-2399` (requeue on partial failure) | [`USER_VERIFICATION_TRUSTPG.go`](../../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VERIFICATION_TRUSTPG.go) (`UserVerificationTrustPg` → `dbActionUserVerifyTrustPg`) | Message se `glid`, `PRIMARY_ATTR_ID`, `PRIMARY_ATTR_VAL` (mobile/email ke liye `GLUSR_USR` table se auto-fill agar khali ho), `SOURCE`/`SOURCE_ID` (source category lookup via `IIL_VERIFICATION_SOURCE_CAT`), `FK_REF_ID`, `SECONDARY_ATTR_MAP` nikal ke, `utils.CallWapiService(...)` se **`POST /user/userverificationdetails` ko internal HTTP-loopback call karta hai**, `valdiation_key = "18833e9703636bce1205043f022a91de"` (hardcoded, `serviceName="user_trust_attr_veri_details"`) ke saath. [`USER_VERIFICATION_TRUSTPG.go:67-114`](../../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VERIFICATION_TRUSTPG.go) |
| `USER_VERIFICATION_TRUSTPG_FAIL` | Same publishers, on failure of the above | Same worker function (`dbActionUserVerifyTrustPg` registered for both queue names, [`IntializeMsgBroker.go:375-376`](../../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go)) | Retry/fail-lane for the above |

**Business/architecture implication**: is controller ka do tarah se hit hona possible hai —
(a) directly WEBERP/SOA-WAPI external callers se, aur (b) indirectly, jab GST verification
(ya bulk verification) flow mein koi attribute verify hota hai aur uska trust-record bhi
banana ho — `USER_GST_DETAILS.go`/`USER_GST_LAST_MODIFIED.go` (GST domain consumers,
[`GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) section 7 mein already documented
"trust database ke against ek verification event record karo" ke roop mein) is exact queue pe
publish karte hain. Isliye GST verification aur is feature ka data-flow **directly linked
hai**, sirf conceptually similar nahi.

**Cross-check**: `Trust_Verification_Technical_Doc.md:130-138` bhi is exact finding ko already
confirm karta hai ("`USER_VERIFICATION_TRUSTPG.go` ... calls this exact endpoint as an
internal HTTP loopback") — dono docs consistent hain, contradiction nahi hai.

---

## 7. Kafka

**Koi Kafka usage nahi mila** kisi bhi is doc ke files mein (`UserVerificationDetailsController.go`,
`.Model.go`, `USER_VERIFICATION_TRUSTPG.go`) — grep se koi `InitializeKafka`/Kafka-import nahi mila.

---

## 8. End-to-End Technical Flows

### Flow A — Direct external submission (WEBERP/SOA-WAPI)

```
WEBERP / SOA-WAPI (external internal-verification-process)
    │
    ▼
[API — write]  POST /user/userverificationdetails  serviceName=TRUST_ATTR_VERI_DETAILS
                {glid, status, primaryAttrID, primaryAttrVal, fk_ref_id?, source,
                 platform, verification_type, datetime, secondaryAttr?: {attrID: val, ...}}
    │  UserVerificationDetailsController.go
    │  1. Gateway_v1(["WEBERP","SOA-WAPI"]) validation
    │  2. LengthAndTypeValidations_v3() — generic UserDetailsMap-based check
    ▼
UserVerificationDetailsValidation()
    │  prepareInputParamsMap() — strict type/format checks (JSON-numbers, datetime format)
    │  validateAttribute(primaryAttrID) + validateSecondaryAttrs()
    ▼
handlePrimaryAndSecondaryInsertions()
    ├─ no secondary attrs → upsertSellerDetails(primary only)
    └─ secondary attrs present → upsertSellerDetails() once per non-empty secondary,
                                   each call passing BOTH primary and that secondary
    │  each call: SELECT fn_insert_glusr_attribute_verification(...) AS status  (1s timeout)
    ▼
[DB — trustpg]  (stored function, internal schema opaque to this code)
    ▼
Response {"STATUS":"SUCCESS"/"FAILURE", "CODE":"200"/"400"/"500", "MESSAGE":...}
```

### Flow B — Indirect trigger via GST/bulk-verification RabbitMQ chain

```
GST verification consumer (USER_GST_DETAILS.go / USER_GST_LAST_MODIFIED.go)
  or Bulk-verification consumer (USER_BULK_VERIFICATION.go)
    │  attribute verify ho gaya, ab trust-record banana hai
    ▼
[RabbitMQ publish]  SERVICENAME=USER_VERIFICATION_TRUSTPG  (or _FAIL on retry)
    ▼
[CONSUME]  USER_VERIFICATION_TRUSTPG.go → dbActionUserVerifyTrustPg()
    │  message se glid/attrID/attrVal/source/secondaryAttr nikalta hai
    │  mobile(121)/email(109) ke liye value khali ho toh GLUSR_USR se auto-fill
    │  source-category-name diya ho toh IIL_VERIFICATION_SOURCE_CAT se ID resolve
    ▼
utils.CallWapiService(inputData, "user_trust_attr_veri_details", ...)
    │  HTTP-loopback, hardcoded valdiation_key="18833e9703636bce1205043f022a91de"
    ▼
[API — write]  POST /user/userverificationdetails  (Flow A ke steps 2 onwards reuse hote hain)
    ▼
[DB — trustpg]  fn_insert_glusr_attribute_verification(...)
    ▼
d.Ack(false) on success / d.Nack(false, false) on HTTP-call failure (no explicit re-queue
loop visible in this file beyond what the queue infra itself does)
```

### Flow C — Reading verification data back (out of this doc's exclusive scope, reference only)

```
Any caller
    │
    ▼
[API — read]  GET_ATTR_VERI_DETAILS  {glusrid, modid, attrId, type?}
    │  GetVerificationDetails → ActionGtVeriModel()
    │  typ=="1" → fn_get_glusr_attr_verify($glid)  |  else → fn_get_glusr_attributes($glid)
    ▼
[DB — trustpg]  same stored-function family Trust Verification KT already documents in depth
    ▼
Response — verified-attribute JSON blob for the given glid

Any caller
    │
    ▼
[API — read]  USERVERIFIEDDETAIL  {glusrid, attribute_id, modid, token, userverified?}
    │  VerifiedDetailController → UserVerifiedDetail_v1()
    ▼
[DB — mesh_pg_user]  SELECT iil_verification_details LEFT JOIN iil_verification_source x3
    ▼
Response — per-attribute "Verified"/"Not Verified" + verifier process/screen/date
```

**Important open point (see section 12)**: is doc mein conclusively confirm nahi ho paaya ki
`fn_insert_glusr_attribute_verification` se likha gaya data actually `fn_get_glusr_attr_verify`/
`fn_get_glusr_attributes` se wapas milta hai ya nahi — dono stored functions opaque hain Go
code se. Agar write aur read alag underlying tables use karte hain, yeh ek silent data-gap ho
sakta hai.

---

## 9. Flow-wise DB & Table Usage

### Flow A/B — Verification-event write (converges to the same DB step once inside the controller)

| # | DB (physical) | Table/Function | Operation | Kya nikala/likha jaata hai, aur kyun |
|---|---|---|---|---|
| 1 | trustpg | `fn_insert_glusr_attribute_verification` | Function call, once per (primary-only OR primary+each-secondary) combination | Verification-event record likhta hai — glid, attribute-ID, attribute-value, source, platform, status, datetime sab ek call mein jaate hain. N secondary-attributes = N sequential calls. |

**Total DB round-trips ek single request ke liye**: 1 (primary-only) ya N (jitne
non-empty secondary-attributes hain), sab sequential, **koi parallelism nahi**
(dekho section 11, optimization point 1).

### Flow B — pre-write DB lookups inside the RabbitMQ consumer (before it even calls the API)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | (consumer's own DB conn, likely mesh_pg_user-class) | `IIL_VERIFICATION_SOURCE_CAT` | SELECT (conditional — sirf jab `SOURCE` diya ho, `SOURCE_ID` nahi) | Source-category-name ko numeric ID mein resolve karta hai |
| 2 | same | `GLUSR_USR` | SELECT (conditional — sirf jab `primaryAttrVal` khali ho AND attrID mobile(121)/email(109) ho) | Mobile/email value auto-fill karta hai agar message mein already nahi tha |

### Flow C (reference only — already fully tabled in Trust Verification Technical Doc)

Dekho [`Trust_Verification_Technical_Doc.md`](../Trust%20Verification%20KT/Trust_Verification_Technical_Doc.md)
section "Flow-wise DB & Table Usage" — `fn_get_glusr_attr_verify`/`fn_get_glusr_attributes`
(trustpg) aur `iil_verification_details` + `iil_verification_source` x3-join (mesh_pg_user)
dono waha already row-by-row documented hain; duplicate karna yahan noise hi hoga.

---

## 10. Optimization Scope — DB Response-Time Contribution

### Medium impact

1. **One stored-function-call per secondary-attribute, sequential in a `for` loop, each
   with its own 1-second DB-call timeout budget** — if a request has N secondary-attributes,
   this is N sequential round-trips to `trustpg`, each capable of taking up to ~1s before
   timing out. Given the stored-function encapsulates the actual insert-logic, this could
   potentially be redesigned as a single stored-function-call accepting an array of
   secondary-attributes, moving the loop into PL/pgSQL — but that would require DB-side
   changes outside this codebase's visibility.
   [`UserVerificationDetailsModel.go:241-249,268`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go)
2. **Two full validation passes happen before any DB call** (`LengthAndTypeValidations_v3`
   at the controller level, then `prepareInputParamsMap` inside the model) — functionally
   fine and cheap relative to the DB call, but worth noting as a minor redundancy if this
   controller is ever refactored.

### Low impact / good practice already present

3. **All validation happens in Go before any DB call** — good practice, avoids wasting
   a DB round-trip on malformed requests.
4. **Consumer-side pre-fetch (`GLUSR_USR`, `IIL_VERIFICATION_SOURCE_CAT`) is conditional,
   not unconditional** — only runs when the incoming queue message is missing data, so the
   common case (already-populated message) skips these two round-trips entirely.

---

## 11. Cron Inventory

**Koi dedicated cron is feature ke apne scope mein nahi mila** — grep kiya
`UserVerificationDetailsController`/`fn_insert_glusr_attribute_verification`/
`USER_VERIFICATION_TRUSTPG` ke liye `service-api-go-production` aur
`user-temp-consumers-production` ke `crons/`-jaisi directories mein, koi match nahi mila
except the consumer worker itself. GST domain ka apna daily cron (`gst_tact_veri_cron.go`,
dekho [`GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) section 13) is feature ko
**indirectly** trigger kar sakta hai (via `USER_GST_DETAILS`/`USER_GST_LAST_MODIFIED` →
`USER_VERIFICATION_TRUSTPG` chain, section 6) lekin uska "home" GST domain hai, is doc ka nahi.

---

## 12. Edge Cases & Gotchas (technical POV)

1. **A single invalid secondary-attribute-key fails the whole secondary-batch** — even
   if 4 out of 5 secondary-attributes were valid, one bad key rejects the entire
   request with no partial-success.
2. **Primary-only insert is skipped whenever secondary-attributes exist** — a caller
   expecting "the primary attribute itself gets its own record regardless" would be
   surprised; the primary's verification-status is only ever recorded jointly with each
   secondary, not standalone, when secondaries are present.
3. **Empty `primaryAttrVal` silently no-ops the entire request but still returns
   `"SUCCESS"`** — `handlePrimaryAndSecondaryInsertions` skips its whole body when
   `primaryAttrVal == ""`, leaving `output == ""`, which the caller (`UserVerificationDetailsValidation`)
   returns as `"SUCCESS"` since no error string was ever set. A caller could believe a record
   was created when nothing was written. [`UserVerificationDetailsModel.go:234,251-254`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go)
4. **Stored-function is a black-box** — table-structure, constraints, and exact
   failure-reasons live entirely in `trustpg`'s PL/pgSQL, invisible to this codebase;
   debugging a failed insert requires DB-access, not just code-reading.
5. **This controller has two independent entry points** (direct external API call vs.
   internal RabbitMQ-consumer HTTP loopback) sharing the *same* hardcoded gateway-allowlist
   path — the consumer's hardcoded `valdiation_key` acts as a "fake WEBERP/SOA-WAPI caller"
   credential baked into `USER_VERIFICATION_TRUSTPG.go`. If that key ever needs rotation,
   both the consumer and whatever gateway-validation logic checks it must change together.
6. **Write (`fn_insert_glusr_attribute_verification`) and read
   (`fn_get_glusr_attr_verify`/`fn_get_glusr_attributes`) are two separate opaque stored
   functions** — this doc cannot confirm from Go code alone that they operate on the same
   underlying table(s). Treat "I wrote a verification event, will it show up on read?" as
   an open question, not an assumption (section 13).
7. **This write-path and Trust Verification KT's main write-path
   (`UserVerificationController`/`UpsertVerification`) both touch "verification" concepts and
   both ultimately write to `trustpg`-adjacent storage** — but through different attribute-ID
   numbering schemes (section 3.1) and different stored functions. Confusing the two GST
   attribute-IDs (`2721` here vs `2106` there) would be a realistic support-debugging mistake.

---

## 13. Open Questions

1. Does `fn_insert_glusr_attribute_verification` write to the same underlying table(s) that
   `fn_get_glusr_attr_verify`/`fn_get_glusr_attributes` read from? Not confirmable from Go
   code — both are opaque `trustpg` stored functions.
2. What is the hardcoded trailing literal `0` (11th positional param) passed to
   `fn_insert_glusr_attribute_verification` in every call? [`UserVerificationDetailsModel.go:265`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go)
   — no comment or variable name gives a hint.
3. What do `status` (input, always `1` when sent by the `USER_VERIFICATION_TRUSTPG` consumer
   — [`USER_VERIFICATION_TRUSTPG.go:100`](../../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VERIFICATION_TRUSTPG.go))
   and `source`/`platform`/`verification_type` string/int-values actually represent in
   business terms — no enum/reference found in this codebase.
4. Is the hardcoded `valdiation_key = "18833e9703636bce1205043f022a91de"` in
   `USER_VERIFICATION_TRUSTPG.go` intentionally static/long-lived, or should it be rotated /
   sourced from config? (Same class of question Trust Verification KT already raised for its
   own hardcoded key used by `VerifyAttr()`.)
5. Live DB schema verification for `trustpg`'s internal tables behind both stored functions —
   this doc only reflects what the Go code's SQL/RPC calls imply.
6. Are there other publishers to `USER_VERIFICATION_TRUSTPG` beyond the three found in this
   repo (`USER_BULK_VERIFICATION.go`, `USER_GST_DETAILS.go`, `USER_GST_LAST_MODIFIED.go`)? This
   review only searched the three repos listed at the top of this doc.

---

## See also

- [`Verification_Process_Business_Doc.md`](./Verification_Process_Business_Doc.md) — product perspective
- [`../Trust Verification KT/Trust_Verification_Technical_Doc.md`](../Trust%20Verification%20KT/Trust_Verification_Technical_Doc.md) —
  main verification-write path (`UserVerificationController`), plus the shared read endpoints
  (`GetVerificationDetailController`/`VerifiedDetailController`) in full depth, and its own
  section on this exact controller ("Trust-DB Attribute-Details Write — The Adjacent Path")
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — section 7 documents
  `USER_VERIFICATION_TRUSTPG` from the GST-domain publisher side; this doc documents its
  consumer side

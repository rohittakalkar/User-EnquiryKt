# KWPL — Technical Doc (Code-Level Deep Dive)

Yeh doc KWPL feature ka **technical implementation** cover karta hai — APIs, DB table,
validation, aur explicitly-checked RabbitMQ/Kafka/Redis/cron absence, sab kuch code se
verify karke. Business/product perspective ke liye
[`KWPL_Business_Doc.md`](./KWPL_Business_Doc.md) dekho.

**Repos involved**: `service-api-go-production` (write — only repo that touches this
feature). `users-api-go-production` aur `user-temp-consumers-production` dono grep kiye
gaye case-insensitively "kwpl" ke liye — **zero matches** dono mein. Isliye: koi read
endpoint nahi hai, koi consumer nahi hai.

**Scope note**: KWPL aur Disposition genuinely **do alag concepts** hain — alag tables
(`PL_KWRD` vs `GLUSR_DISPOSITIONS`), alag controllers, koi FK-relation ya shared-code nahi
mila. Dekho [`../Disposition KT/Disposition_Technical_Doc.md`](../Disposition%20KT/Disposition_Technical_Doc.md).

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file path diya
gaya hai har jagah). Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — team se confirm karo]** likh diya hai, guess nahi kiya.

---

## 0. "KWPL" ka matlab kya hai

Feature ka koi inline code-comment nahi hai jo "KWPL" expand karta ho. Lekin
[`users_domain_product_stories.md:191`](../users_domain_product_stories.md) mein yeh
"Seller Growth & Recommendations" (paid-visibility features) section ke andar explicitly
likha hai: **"Paid listing keywords (PL/KWPL)"**. Table naam bhi `PL_KWRD` hai (Paid-Listing
KeyWoRD). Isse strongly suggest hota hai KWPL = **KeyWord Paid Listing** (ya "Paid Listing
KeyWord" — order clear nahi hai), ek paid-visibility/campaign feature jahan supplier ek
specific keyword ke liye ek city mein "serve" (paid placement) kharidta/paata hai, kisi
work-order (`WO_ID`) ke against.

**[INFERRED — team se confirm karo]**: exact expansion aur iska product-context (jaise: kya
yeh IndiaMART ke "TrustSEAL"/paid-listing package ka part hai, ya sales-team driven
promotional keyword placement hai) — dono conclusively code se nahi mila.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write controller | write | [`UserKwplController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserKwplController.go) |
| Write model (`UpsertKwpl`) | write | [`UserKwplModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserKwplModel.go) |
| Mandatory-field validation | write | `MandatoryParamsCheckKwpl` — [`UserUtilsMandatory.go:11-124`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go) |
| Length/type validation map | write | `UserkwplMap` — [`UsersValidationMaps.go:44-68`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |
| Route registration | write | [`router.go:153,321`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) (registered twice, seemingly two identical route-group blocks in the same file — see Edge Cases §9.5) |
| Documented elsewhere | write | [`service_api_write_reference.md:130`](../service_api_write_reference.md), [`users_domain_product_stories.md:191`](../users_domain_product_stories.md) |
| Coincidental field-name collision (NOT the same table) | write | [`comptomcat.go:485-487`](../../service-api-go-production/service-api-go-production/internal/controllers/ProductsControllers/comptomcat.go) — a completely unrelated "premium listing" (`GLUSR_PREMIUM_*`) validation map reuses the literal key `"PL_KWRD_TERM_UPPER"` as one of its field names. This does **not** touch the `PL_KWRD` table — confirmed by reading the surrounding function, it writes to `GLUSR_PREMIUM_LISTING`-style columns. Flagging only so nobody greps `PL_KWRD_TERM_UPPER` and assumes it's KWPL-related. |

**Confirmed absent (actively searched for, not assumed)**:
- `grep -rni "kwpl" users-api-go-production/internal` → 0 matches → no read/GET endpoint.
- `grep -rni "kwpl" user-temp-consumers-production` → 0 matches → no consumer.
- `grep -rli "kwpl" service-api-go-production` → only the 7 files listed above (controller,
  model, 2 validation files, router, middleware/requestValidation.go and
  pkg/utils/globalfunctions.go — both of which only reference "KWPL" as a literal
  serviceName string inside generic/shared plumbing, not KWPL-specific business logic).
- No RabbitMQ publish call, no Kafka reference, no Redis reference, no cron reference
  anywhere near `PL_KWRD`/`Kwpl` in any of the three repos.

---

## 2. Routes

| Method | Path | Repo | Controller | serviceName (Gateway) |
|---|---|---|---|---|
| POST | `/kwpl` | write | `UserKwplController` | `"k"` passed to `Gateway_v1` — [`UserKwplController.go:57`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserKwplController.go); response body reports `"SERVICE_NAME": "KWPL_SERVICE"` |

No corresponding `GET` route exists in either `users-api-go-production` or
`service-api-go-production` for this data.

---

## 3. Data Model — Table

> **Verification note**: table/column names neeche Go code ke andar embedded SQL strings se
> liye gaye hain ([`UserKwplModel.go:152-231`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserKwplModel.go)).
> Live DB schema se cross-verify **nahi** kiya gaya hai.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `PL_KWRD` | meshpg (`config.GetPGDbConnection("meshpg")`) | Ek paid-listing keyword "serve" record — supplier, city, company, keyword, validity window, work-order ID | `pl_kwrd_id` (PK, `RETURNING` on INSERT), `FK_GL_CITY_ID`, `PL_KWRD_MODID`, `FK_GLUSR_USR_ID`, `FK_COMPANY_ID`, `PL_KWRD_TERM_UPPER` (the keyword itself), `PL_KWRD_DESC`, `PL_KWRD_FROM`/`PL_KWRD_TO` (validity window), `PL_KWRD_ENABLE` (`-1` = active/enabled sentinel), `PL_KWRD_COMMENT`/`PL_KWRD_COMMENTS`, `PL_KWRD_WO_ID` (work-order ID), `PL_KWRD_UPDATEDBY_ID`/`_UPDATEDBY`/`_UPDATEDBY_AGENCY`, `PL_KWRD_UPDATESCREEN`, `PL_KWRD_IP`/`_IP_COUNTRY`, `PL_KWRD_UPDATEDUSING`, `PL_KWRD_HIST_COMMENTS`, `PL_KWRD_UPDATEDBY_URL`, `FK_CUST_TO_SERV_ID` (customer-to-service/"serve" ID) — [`UserKwplModel.go:152-231`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserKwplModel.go), field-name mapping cross-checked against [`UsersValidationMaps.go:44-68`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |

Note: the model file uses **two different column names for the same logical field** in two
places — the INSERT statement's parameter list uses `values["COMMENT"]` for both the
`PL_KWRD_COMMENT` slot (param `$10`) *and* redundantly overwrites the same key again at
[`UserKwplModel.go:103-107`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserKwplModel.go)
(a dead no-op duplicate of the same `if utils.KeyPresent(params, "COMMENT")` block already
present at line 58-62) — while `UserkwplMap` (validation layer) separately defines both
`"COMMENT"` → `PL_KWRD_COMMENTS` and `"HIST_COMMENT"` → `PL_KWRD_HIST_COMMENTS` as distinct
fields. The model itself never reads a `HIST_COMMENT` key from params at all — it reuses
`values["COMMENT"]` for the `PL_KWRD_HIST_COMMENTS` SQL parameter too, in every branch
(INSERT `$19`, DISABLE `$9`, UPDATE `$10`, SEARCH_UPDATE `$9`). **[INFERRED]**: this looks
like unintentional copy-paste — `PL_KWRD_HIST_COMMENTS` is very likely always populated with
the same input value as `PL_KWRD_COMMENT`/`PL_KWRD_COMMENTS`, never a distinct
"history comment," despite the validation map defining a separate `HIST_COMMENT` input key
that the model code never actually consumes. Confirm with team whether `HIST_COMMENT` was
meant to be wired in.

---

## 4. Business Rules & Validation (code se exhaustive list)

1. **Gateway allowlist**: only `GLADMIN` and `WEBERP` may call this endpoint —
   `utils.Gateway_v1([]string{"GLADMIN","WEBERP"}, validationKey, "k")` —
   [`UserKwplController.go:57`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserKwplController.go).
   Purely an internal/admin-tool feature, no supplier-facing or public path.
2. **Universally-mandatory fields** (all four actions): `UPDATEDBY`, `VALIDATION_KEY`,
   `SERVE_ID`, `IP`, `COUNTRY`, `SCREEN`, `ACTION`, `GLUSRID` — missing any one produces
   `"Please Enter Mandatory(UPDATEDBY/VALIDATION_KEY/SERVE ID/IP/COUNTRY/SCREEN/ACTION/GLUSRID) Fields"`.
   [`UserUtilsMandatory.go:47-48`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
3. **`ACTION` param drives a 4-way branch**, each with its own additional mandatory fields:
   - **`INSERT`** → also needs `WO_ID`, `FROM_DATE1`, `TO_DATE1`, `COMPID`, `PL_KEYWORDS`
     (error: `"Please Enter Mandatory Parameters(WO_ID/FROM_DATE1/TO_DATE1/COMPID/PL_KEYWORDS) Fields"`)
     → new row, all fields.
     [`UserUtilsMandatory.go:49-73`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
   - **`DISABLE`** → also needs `ENABLE`
     (error: `"Please Enter Mandatory Parameter(GLUSRID/ENABLE) Fields"`)
     → `UPDATE ... SET PL_KWRD_ENABLE=$1, ... WHERE FK_GLUSR_USR_ID=$11 AND
     FK_CUST_TO_SERV_ID=$12 AND PL_KWRD_ENABLE=-1` — only affects currently-enabled rows.
     [`UserUtilsMandatory.go:75-83`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go),
     [`UserKwplModel.go:158-175`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserKwplModel.go)
   - **`UPDATE`** → also needs `WO_ID`, `TO_DATE1`
     (error: `"Please Enter Mandatory Parameter(GLUSRID/WO_ID/TO_DATE1 ) Fields"`)
     → same active-row-guard (`PL_KWRD_ENABLE=-1`), updates `PL_KWRD_TO` (validity-end) and
     `PL_KWRD_WO_ID`.
     [`UserUtilsMandatory.go:85-96`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go),
     [`UserKwplModel.go:177-195`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserKwplModel.go)
   - **`SEARCH_UPDATE`** → also needs `UPDATEID`, `PL_KEYWORDS`
     (error: `"Please Enter Mandatory Parameter(UPDATEID/PL_KEYWORDS) Fields"`)
     → two sub-variants: if `ID` is present and non-empty, matches by `PL_KWRD_ID` directly;
     else matches by `(FK_GLUSR_USR_ID, FK_CUST_TO_SERV_ID)` — both still guarded by
     `PL_KWRD_ENABLE=-1`. Only updates the keyword-term itself (`PL_KWRD_TERM_UPPER`).
     [`UserUtilsMandatory.go:98-108`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go),
     [`UserKwplModel.go:196-234`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserKwplModel.go)
4. **Unrecognized `ACTION` values silently fall through** `UpsertKwpl`'s if/else-if chain —
   `sql` and `input_params` stay unset, `ExecuteQueryRows` runs an empty-string query, and
   the code proceeds to the `else { output = "INSERT SUCCESS" }` branch anyway (only the
   `flag == "INSERT"` branch does the `Scan` loop; every other value of `flag`, including an
   unrecognized one, takes the generic "assume success" branch) — see Edge Case §9.2, this is
   almost certainly a latent bug: an invalid `ACTION` that passes the earlier mandatory-field
   gate (see point 5) would report success while doing nothing.
   [`UserKwplModel.go:249-264`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserKwplModel.go)
5. **`MandatoryParamsCheckKwpl`'s final `else if ok_1` branch has an always-true dead
   condition**: `if val != 0 || val != 1 { output = "Please Enter Correct ENABLE parameter value" }`
   — for any integer `val`, at least one of `val != 0` / `val != 1` is always true (a number
   cannot simultaneously equal both 0 and 1), so this branch **always** produces the error
   message whenever it's reached (i.e., `ACTION` is none of INSERT/DISABLE/UPDATE/
   SEARCH_UPDATE, but `ENABLE` was supplied as a numeric string). The evident intent was
   `val != 0 && val != 1` (reject anything that isn't 0 or 1). Net effect: this is dead/
   always-failing validation code, functionally equivalent to "any unrecognized ACTION with
   an ENABLE param present is rejected" — but for the wrong stated reason.
   [`UserUtilsMandatory.go:110-118`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
6. **DISABLE/UPDATE/SEARCH_UPDATE (without `ID`) all match by `(FK_GLUSR_USR_ID,
   FK_CUST_TO_SERV_ID, PL_KWRD_ENABLE=-1)` only** — currently-enabled rows only,
   already-disabled records are never touched again by these actions.
7. **`PL_KWRD_ENABLE=-1` is used as the "active" sentinel** — not `1`/`true`, an
   inherited-legacy-convention worth noting for anyone querying this table directly.
   [`UserKwplModel.go:174,193,212,230`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserKwplModel.go)
8. **Length/type limits (`UserkwplMap`)**: `PL_KEYWORDS` ≤ 100 chars, `DESC` ≤ 1500 chars,
   `COMMENT` ≤ 300 chars, `HIST_COMMENT` ≤ 1000 chars (though never actually wired to the
   model, see §3), `UPDATEDBY` ≤ 60 chars, `COUNTRY` ≤ 40 chars, `UPDATEURL` ≤ 255 chars,
   `MODID` ≤ 8 chars, `CITY_ID` ≤ 10 digits, `ENABLE` ≤ 2 digits, `UPDATEID` ≤ 10 chars.
   [`UsersValidationMaps.go:44-68`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go)
9. **Response's success-check is always `"INSERT SUCCESS"` regardless of actual action** —
   `UpsertKwpl`'s success-output string is hardcoded the same for INSERT/DISABLE/UPDATE/
   SEARCH_UPDATE, and the controller-level HTTP status/code-decision also only looks for the
   literal string `"INSERT SUCCESS"` — [`UserKwplController.go:79`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserKwplController.go).
   Naming is misleading but functionally consistent across actions.
   [`UserKwplModel.go:252-263`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserKwplModel.go)
10. **`GLUSRID`/`unique_id` are stripped from the params map** (`delete(inputParams,
    "VALIDATION_KEY")`, `delete(inputParams,"unique_id")`) before length/type validation,
    but `VALIDATION_KEY` itself is captured earlier for the gateway check — standard pattern
    reused across this codebase, not KWPL-specific.
    [`UserKwplController.go:60-61`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserKwplController.go)

---

## 5. RabbitMQ

**Koi RabbitMQ usage nahi mila** in KWPL code paths — no `PushToQueue`/`PubAPI`/`Requeue`
call, no `SERVICENAME` constant, anywhere in `UserKwplController.go` or `UserKwplModel.go`.
Confirmed via `grep -n -i "SERVICENAME\|Publish\|rabbitmq\|amqp" UserKwplController.go
UserKwplModel.go` → 0 matches. `UpsertKwpl` is a purely synchronous single-table write with
no downstream fan-out.

## 6. Kafka

**Explicit no.** `grep -rni "kafka" service-api-go-production` limited to files touching
`Kwpl` → 0 matches. No Kafka topic, producer, or consumer anywhere near this feature.

## 7. Redis

**Explicit no.** `grep -rni "redis" service-api-go-production` limited to files touching
`Kwpl` → 0 matches. There is also no read endpoint to cache in the first place (§1) — the
only DB interaction is the single synchronous write query per request.

---

## 8. End-to-End Technical Flow

There is exactly one flow — this feature has a single entrypoint, no async fan-out, no
downstream consumers.

```
Internal-tool (GLADMIN / WebERP)
    │
    ▼
[API — write]  POST /kwpl  {ACTION(INSERT/DISABLE/UPDATE/SEARCH_UPDATE), GLUSRID, CITY_ID,
                COMPID, PL_KEYWORDS, DESC, FROM_DATE1, TO_DATE1, ENABLE, WO_ID, SERVE_ID,
                UPDATEID, UPDATEDBY, AGENCY, SCREEN, IP, COUNTRY, UPDATEUSING, COMMENT,
                UPDATEURL, ID, VALIDATION_KEY, unique_id}
    │  UserKwplController.go
    │  1. utils.InputJsonCheck(inputParams) — basic JSON shape check
    │  2. MandatoryParamsCheckKwpl() — action-specific mandatory-field gate (section 4.2-4.5)
    │  3. Gateway_v1(["GLADMIN","WEBERP"], validationKey, "k") — internal-caller allowlist
    │  4. LengthAndTypeValidations_v3(inputParams, UserkwplMap) — per-field length/type checks
    ▼
UpsertKwpl()
    │  meshpg connection acquired (config.GetPGDbConnection("meshpg"))
    ├─ ACTION=INSERT        → INSERT INTO PL_KWRD (21 columns) RETURNING pl_kwrd_id
    ├─ ACTION=DISABLE       → UPDATE ... WHERE (FK_GLUSR_USR_ID, FK_CUST_TO_SERV_ID) AND ENABLE=-1
    ├─ ACTION=UPDATE        → UPDATE (PL_KWRD_TO, PL_KWRD_WO_ID) WHERE (user, serve_id) AND ENABLE=-1
    ├─ ACTION=SEARCH_UPDATE → UPDATE (PL_KWRD_TERM_UPPER only) WHERE (ID) or (user, serve_id) AND ENABLE=-1
    └─ ACTION=<anything else> → sql/input_params left unset, query still executes (empty string),
                                  falls into "else { output = INSERT SUCCESS }" — see section 4.4
    ▼
[DB — meshpg, single round-trip]  PL_KWRD
    ▼
Response {"STATUS", "CODE", "MESSAGE": output, "SERVE_ID", "SERVICE_NAME": "KWPL_SERVICE", "RESPONSE_DATA": {...timing/Kibana fields...}}
    (HTTP 200/"SUCCESSFUL" iff output == "INSERT SUCCESS" exactly, else 500/"FAILED")
    │
    ▼
[Kibana logging]  utils.KibanaLogging_v1(...) — request/response timing, no downstream system call
```

---

## 9. Flow-wise DB & Table Usage

Only one flow exists; it does exactly one DB round-trip.

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg (`config.GetPGDbConnection("meshpg")`) | `PL_KWRD` | INSERT (action=INSERT) / UPDATE (action=DISABLE/UPDATE/SEARCH_UPDATE) | Single write per request — creates a new keyword-serve record, or mutates an existing enabled one, depending on `ACTION`. There is no preceding SELECT to check existence/uniqueness before INSERT, and no SELECT to confirm a row was actually matched/affected before returning success on the UPDATE-style actions (see Edge Case §11.3 — `RowsAffected` is never checked). |

**Total DB round-trips per request: exactly 1.** This is the simplest flow of any KT doc in
this docs folder — no joins, no fan-out, no parallelism, no retries. There is no
"optimization scope" in the DB sense because there's nothing to optimize at the query level
(see section 10).

---

## 10. Optimization Scope — DB Response-Time Contribution

### Low-impact / not applicable

This feature already does the minimum possible number of round-trips (1 connection + 1
query). There is no N+1 pattern, no sequential round-trips, no missing parallelism to fix.
The only concrete observations worth flagging:

1. **No `RowsAffected` check on UPDATE-style actions** — `UpsertKwpl` always returns
   `"INSERT SUCCESS"` for DISABLE/UPDATE/SEARCH_UPDATE as long as the query executes without
   a driver-level error, even if zero rows matched the `WHERE` clause (e.g., record already
   disabled, or `GLUSRID`/`SERVE_ID` typo). This isn't a response-time issue but is a
   correctness/observability gap — callers cannot distinguish "record updated" from "no
   matching record found." **Fix**: check `result.RowsAffected()` (or equivalent) and
   surface a distinct response when zero rows matched.
   [`UserKwplModel.go:261-263`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserKwplModel.go)
2. **Confirm a composite index exists on `(FK_GLUSR_USR_ID, FK_CUST_TO_SERV_ID,
   PL_KWRD_ENABLE)`** on `PL_KWRD` — every DISABLE/UPDATE/SEARCH_UPDATE-without-`ID` query
   filters on exactly this triple; without an index this becomes a sequential scan at scale.
   **[INFERRED — cannot confirm index existence from Go source, verify against live schema.]**
3. **`ExecuteQueryRows` is called with a 1-second timeout** (`1*time.Second`) —
   [`UserKwplModel.go:242`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserKwplModel.go)
   — consistent with a single fast indexed write; no evidence this is a bottleneck.

---

## 11. Edge Cases & Gotchas (technical POV)

1. **`PL_KWRD_ENABLE=-1` as the active-sentinel is non-obvious** — any new code touching
   this table must know this convention, not `1`/`true`.
2. **Unrecognized `ACTION` values report success without writing anything usable** — the
   `if/else-if` chain in `UpsertKwpl` has no final `else` that errors out; an `ACTION` value
   outside {INSERT, DISABLE, UPDATE, SEARCH_UPDATE} that somehow clears the mandatory-field
   gate (see point 3 below — the gate has its own gap) leaves `sql`/`input_params` unset,
   the query executes as an empty string, and the generic non-INSERT branch still reports
   `"INSERT SUCCESS"`. [`UserKwplModel.go:151-264`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserKwplModel.go)
3. **Mandatory-field validation has a dead/always-triggering branch** (section 4.5) —
   `val != 0 || val != 1` is a boolean tautology, always true. In practice this branch is
   only reachable when `ACTION` is *not* one of the four known values but `ENABLE` was
   supplied — so it does end up rejecting most stray `ACTION` values that include `ENABLE`,
   just via broken logic that happens to reject correctly for the wrong stated reason. An
   `ACTION` outside the four known values **without** `ENABLE` present would still fall
   through validation entirely (returns `output = ""`, i.e. "no error") and proceed into the
   bug described in point 2.
4. **`PL_KWRD_HIST_COMMENTS` is always populated from the `COMMENT` input, never a distinct
   value** (section 3) — the validation map defines a separate `HIST_COMMENT` field that the
   model never reads.
5. **Route registered twice in `router.go`** (lines 153 and 321, identical
   `users.POST("/kwpl", UserControllers.UserKwplController)`) — both inside what look like
   two structurally-parallel route-group blocks in the same file. **[INFERRED]**: likely two
   API-version or two auth-context route groups (the surrounding routes in both blocks are
   identical too, e.g. `/pnssetting`, `/user/dispositon`), not a duplicate-registration bug
   — but this wasn't traced further since it's a router-wide pattern, not KWPL-specific;
   confirm with team if the two blocks serve different purposes (e.g. versioned auth).
6. **`SEARCH_UPDATE` without `ID` falls back to matching by `(GLUSRID, SERVE_ID)`** — if
   multiple active rows exist for that pair (no uniqueness enforced in code or confirmed at
   the DB level), all matching rows would be updated in one call.
7. **No read-endpoint anywhere** — how this data is consumed/displayed (admin dashboard,
   direct DB query, a report) is not knowable from this codebase.
8. **Success detection is a brittle exact-string match** (`output == "INSERT SUCCESS"`,
   both in the model's own scan-loop and the controller's HTTP-status decision) — any future
   change to the returned string (e.g., adding more detail to the success message) would
   silently break the HTTP status code without a compile-time signal.

---

## 12. Cron Inventory

**None found.** `grep -rni "cron" service-api-go-production/service-api-go-production/crons`
and the KWPL-touching files → no cron references `PL_KWRD`, `Kwpl`, or `KWPL_SERVICE`. This
is a purely request-driven, synchronous-write feature with no scheduled/background
component.

---

## 13. Open Questions

1. What does "KWPL" actually stand for, and what product surface consumes/displays this
   data (section 0)? Only inferred from a sibling doc's one-line gloss
   ("Paid listing keywords (PL/KWPL)") and the table name `PL_KWRD`.
2. `FK_CUST_TO_SERV_ID`/`SERVE_ID`/`WO_ID` (work-order) — what upstream system originates
   these IDs? Likely a sales/campaign/lead-gen or paid-package system, not traceable from
   this repo alone.
3. Is `PL_KWRD_HIST_COMMENTS` always meant to mirror `PL_KWRD_COMMENT`/`PL_KWRD_COMMENTS`
   (section 3, 11.4), or was `HIST_COMMENT` supposed to be wired in as a separate field and
   this is a bug?
4. Is the unrecognized-`ACTION` "always reports success" behavior (section 11.2) known/
   accepted, or an unnoticed bug? No test or comment addresses it.
5. Do the two identical `/kwpl` route registrations (section 11.5) serve genuinely different
   purposes (versioning/auth context), or is this incidental duplication?
6. `(FK_GLUSR_USR_ID, FK_CUST_TO_SERV_ID, PL_KWRD_ENABLE)` composite-index existence on the
   live `PL_KWRD` table — cannot confirm from Go source.
7. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai; no pgAdmin/live-schema cross-check was done.

---

## See also

- [`KWPL_Business_Doc.md`](./KWPL_Business_Doc.md) — product perspective, bina code ke
- [`../Disposition KT/Disposition_Technical_Doc.md`](../Disposition%20KT/Disposition_Technical_Doc.md) —
  unrelated concept, documented separately (no shared table/FK/controller)
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — sibling KT doc,
  same depth/structure convention
- [`../users_domain_product_stories.md`](../users_domain_product_stories.md) — where KWPL is
  classified under "Seller Growth & Recommendations"
- [`../service_api_write_reference.md`](../service_api_write_reference.md) — one-line write-repo
  reference entry for `/kwpl`

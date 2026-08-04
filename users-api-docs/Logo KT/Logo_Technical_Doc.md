# Company Logo — Technical Doc (Code-Level Deep Dive)

**Scope note**: yeh doc sirf **Logo** (`GLUSR_USR_LOGO`) cover karta hai — **Image**
(`GLUSR_USR_IMAGE`, general branding photos) ek genuinely alag table/write-path hai, dekho
[`../Image KT/Image_Technical_Doc.md`](../Image%20KT/Image_Technical_Doc.md). Dono
`FK_GLUSR_USR_ID` se supplier se link hain, lekin ek doosre se FK-linked nahi hain — no
DB-level coupling mila.

Business/product perspective ke liye [`Logo_Business_Doc.md`](./Logo_Business_Doc.md) dekho.

**Repos**: `service-api-go-production` (write), `user-temp-consumers-production` (3 fan-out
consumers), `users-api-go-production` (reads — logo column consumed indirectly by other
endpoints, dekho section 2).

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file path diya
gaya hai har jagah). Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — team se confirm karo]** likh diya hai, guess nahi kiya. Yeh doc ka deep pass
hai — pichhle shallow pass ke saare Open Questions is baar resolve/re-verify kiye gaye hain.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Logo write (insert/update/delete/approve/reject — sab ek hi controller/model se) | write | [`UserLogoController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserLogoController.go), [`UserLogoModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserLogoModel.go) (`UpsertLogo`, `upsert_logo_pg`) |
| Field/type/length validation map | write | [`UsersValidationMaps.go:707-726`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) (`UserCompanyLogoMap`) |
| Mandatory-field validation | write | [`UserUtilsMandatory.go:1502-1578`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go) (`MandatoryParamsCheckCompanyLogo`) |
| Fan-out — search/IMSDB replica | consumers | [`USER_COMPSYNC_LOGO_IMSDB.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_COMPSYNC_LOGO_IMSDB.go) |
| Fan-out — LMS | consumers | [`USER_LOGO_LMSPG.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_LOGO_LMSPG.go) |
| Fan-out — general company-sync (meshPg) | consumers | [`USER_COMP_SYNC_LOGO.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_COMP_SYNC_LOGO.go) |
| Queue registration (all 3 consumers) | consumers | [`router.go:30,44,88`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Router/router.go) |
| Read — no dedicated `GET /user/logo` controller anywhere in `service-api-go-production` or `users-api-go-production` — confirmed by repo-wide grep this pass, not just repeated from before | — | — |
| Read — logo *is* consumed indirectly by at least one read endpoint: buyer-profile query joins `GLUSR_USR_LOGO` directly | read | [`UserBuyerProfileModel.go:130-136,250-253`](../../users-api-go-production/internal/models/users/UserBuyerProfileModel.go) — `LEFT JOIN GLUSR_USR_LOGO L ON L.FK_GLUSR_USR_ID = CONTACTS_GLID`, selects `GLUSR_USR_LOGO_IMG_90X90`. Served by `GET/POST buyerprofile/*params` → `ActionBuyerProfile` — [`routerUsers.go:214-215,232-233,498-499,556-557`](../../users-api-go-production/internal/api/users_router/routerUsers.go) |
| Read — logo also surfaces via a separately-synced search/listing dataset (`glusr_usr_logo_img_90x90` / `glusr_logo` columns), NOT a live join to `GLUSR_USR_LOGO` in the reviewed code | read | [`impcat_struct.go:99`](../../users-api-go-production/pkg/structs/impcat_struct.go) (`CompanyLogo` field, comment `// GLUSR_USR_LOGO_IMG`), [`SID_Detail_model.go:1270-1274,1904-1908,2060-2064`](../../users-api-go-production/internal/models/master/SID_Detail_model.go) (`glusr_logo` from a pre-built supplier-list dataset), [`UsersPdp.go:2060-2063`](../../users-api-go-production/internal/models/products/UsersPdp.go) (`glusr_usr_logo_img_90x90` from a joined/pre-aggregated row) — strong circumstantial evidence for why the IMSDB/search-replica fan-out consumer (section 5) exists: these read paths likely hit a replica/search-side dataset the fan-out keeps in sync, not `meshpg`'s `GLUSR_USR_LOGO` directly. **[INFERRED — exact source table/view not conclusively traced this pass, flagged in Open Questions]** |
| `otherdetail` endpoint | read | `UserOtherDetailController` via `GET/POST otherdetail/*params`. **Grepped this pass for any `GLUSR_USR_LOGO`/`GLUSR_LOGO` reference: none found.** The previous doc's claim that logo is "read via otherdetail" is **not supported by this pass's evidence** — corrected below (section 2). |

---

## 2. Routes

| Method | Path | Repo | Controller |
|---|---|---|---|
| POST | `/user/logo` | write | `UserLogoController` — [`router.go:167,335`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) (registered twice, likely two different router groups/versions) |
| GET/POST | `buyerprofile/*params` | read | `ActionBuyerProfile` — the only endpoint in this review that demonstrably reads `GLUSR_USR_LOGO` directly via SQL JOIN |

**Correction vs. previous doc**: the earlier version claimed logo reads happen "via the
`otherdetail` endpoint, no dedicated `GET /user/logo` controller." The second half is
correct (no dedicated logo-read controller exists — re-verified this pass). The first
half — that `otherdetail` is the read path — is **not supported**: a repo-wide,
case-insensitive grep for `GLUSR_USR_LOGO`/`GLUSR_LOGO` inside the `otherdetail`
controller/model returned no matches. The actual demonstrable logo-read paths are
`buyerprofile` (direct JOIN) and the search/PDP/SID listing datasets (likely via the IMSDB
replica, not conclusively proven this pass — see Open Questions).

---

## 3. Data Model — Table

> **Verification note**: column/table names below come from Go-embedded SQL strings (file:line
> cited per row). Live DB schema has **not** been cross-verified against pgAdmin — do that
> before using this for a migration or schema-change decision.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_USR_LOGO` | meshpg (write, primary) + searchPg (IMSDB replica) + lmsPg (LMS replica) + meshPg (2nd connection, general-sync consumer) | Logo record with an **approval-workflow** (Pending → Approved/Rejected/Deleted) | `GLUSR_LOGO_APPROVAL_ID` (PK, `RETURNING` on insert), `FK_GLUSR_USR_ID`, `GLUSR_LOGO_APPROVAL_STATUS` (`P`/`A`/`R`/`D` — decoded fully in section 4), `GLUSR_LOGO_REJECTION_REASON`, `GLUSR_LOGO_UPDATEDBY_ID`/`_UPDATEDBY`/`_UPDATEDBY_AGENCY`, `GLUSR_LOGO_UPDATESCREEN`, `GLUSR_LOGO_IP`/`_IP_COUNTRY`, `GLUSR_LOGO_UPDATEDUSING`, `GLUSR_LOGO_HIST_COMMENTS`, `GLUSR_LOGO_UPDATED_TIME`, `GLUSR_LOGO_UPDATEDBY_URL`, `GLUSR_USR_LOGO_IMG_90X90`/`_120X120`/`_ORIGINAL`/`_250X250` — [`UsersValidationMaps.go:707-726`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go), [`UserLogoModel.go:95-108,162-167`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserLogoModel.go) |

Only **one** table is involved in this feature (unlike GST's multi-table split) — write side
and all 3 fan-out consumers write/read the exact same `GLUSR_USR_LOGO` schema, just on
different physical Postgres instances (meshpg → searchPg / lmsPg / meshPg-again).

---

## 4. `GLUSR_LOGO_APPROVAL_STATUS` — Decoding the Status Values

**Fully decoded this pass** (previous doc flagged only `'P'` and left the rest as an Open
Question). All four values are set by literal string assignment in `upsert_logo_pg`:

| Value | Meaning | Set by | Evidence |
|---|---|---|---|
| `P` | **Pending** — new logo just submitted, awaiting review | Hardcoded on INSERT (`FLAG="I"`, `APPROVAL_ID` empty) — status is **not** client-supplied on insert, it's forced | [`UserLogoModel.go:96`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserLogoModel.go) — `VALUES($1,'P',...)` |
| `A` | **Approved** | Set when `APPROVAL_ID` is present **and** `STATUS != "R"` (approve path) | [`UserLogoModel.go:166,190`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserLogoModel.go) — `SET ... GLUSR_LOGO_APPROVAL_STATUS = 'A' ... WHERE ... AND GLUSR_LOGO_APPROVAL_STATUS = 'P'` |
| `R` | **Rejected** | Set when `APPROVAL_ID` is present **and** client sends `STATUS="R"` (rejection reason mandatory, section 8) | [`UserLogoModel.go:162,187`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserLogoModel.go) |
| `D` | **Deleted** (soft-delete — image columns nulled, not a row DELETE) | Set on the "no FLAG" branch (`APPROVAL_ID` empty, `FLAG` neither `I` nor `U`) — image columns forced to `NULL` | [`UserLogoModel.go:105,144`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserLogoModel.go) |

**Key finding this pass — resolves 2 of the previous doc's 3 Open Questions**: the
`APPROVAL_ID`-present branch (previously flagged "not fully traced") **is** the
approve/reject action, and it runs through the **same** `POST /user/logo` endpoint and the
**same** `upsert_logo_pg` function — there is no separate admin controller. The caller
distinguishes the four actions purely by which fields are present:

- `APPROVAL_ID` empty + `FLAG="I"` → **Insert** (new logo, forced to `P`)
- `APPROVAL_ID` empty + `FLAG="U"` → **Update** (image/metadata resubmission)
- `APPROVAL_ID` empty + `FLAG` neither `I` nor `U` (validated as `"D"` by mandatory-check, section 8) → **Soft-delete**
- `APPROVAL_ID` present → **Approve/Reject** — an admin/moderation action against an existing pending record, gated additionally by `AND GLUSR_LOGO_APPROVAL_STATUS = 'P'` in the WHERE clause (an already-approved/rejected record cannot be re-approved/re-rejected through this path — silent no-op, no error surfaced)
  [`UserLogoModel.go:162,166`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserLogoModel.go)

**Caveat / open item**: in the `FLAG="U"` branch, `GLUSR_LOGO_APPROVAL_STATUS` is set directly
to whatever the caller passes as `STATUS` (`UserLogoModel.go:100-101`, `$1 = params1["STATUS"]`)
— there is no code-level guarantee it gets reset to `'P'` on a resubmission after rejection.
Whether the calling UI always passes `STATUS='P'` on a supplier resubmit is
**[INFERRED — confirm with team]**, not verifiable from this file alone.

---

## 5. RabbitMQ

Logo publishing goes through the same shared `PushToQueue` helper used across the domain
([`rabbitmq.go`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go)).

| SERVICENAME | Resolved queue | Resolved exchange | Publisher | Consumer(s) |
|---|---|---|---|---|
| `COMPANY_LOGO_SERVICE` | `user.logo.<glid % 20>` (20-way sharded by GLID modulus) | `USER.topic` | [`UserLogoModel.go:222`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserLogoModel.go) (`upsert_logo_pg`, only fires if `rabbitarray["MESSAGE"]` non-empty — i.e. only after a successful DB write) | 3 independent consumers registered on this queue name in [`router.go:30,44,88`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Router/router.go): `USER_COMPSYNC_LOGO_IMSDB`, `USER_LOGO_LMSPG`, `USER_COMP_SYNC_LOGO` |
| `USER_COMPSYNC_LOGO_IMSDB_FAIL` | (via `Requeue` helper) | — | [`USER_COMPSYNC_LOGO_IMSDB.go:89`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_COMPSYNC_LOGO_IMSDB.go), on SearchPG upsert failure | **Not registered in `QueueFuncMap`** — grepped `router.go`, this fail-queue has no consumer wired up in this repo. Messages likely accumulate/get manually drained, or are consumed by a component outside this repo — **[INFERRED — confirm with team]** |
| `USER_COMP_SYNC_LOGO_FAIL` | (via `Requeue` helper) | — | [`USER_COMP_SYNC_LOGO.go:110`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_COMP_SYNC_LOGO.go), on MeshPG upsert failure | Same as above — **not registered in `QueueFuncMap`**, same open question |
| *(LMS consumer has no fail-queue)* | — | — | [`USER_LOGO_LMSPG.go:94`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_LOGO_LMSPG.go) | On failure this consumer just `d.Nack(false, false)`s the message — no explicit republish-to-fail-queue call, unlike its two siblings. Inconsistency across 3 near-identical consumers (section 13) |

**New finding this pass**: all 3 consumers are near-identical (same SQL-building logic, same
`FK_GLUSR_USR_ID`/`GLUSR_LOGO_APPROVAL_ID` handling), but their failure-handling diverges:
IMSDB and general-sync consumers explicitly republish to a `_FAIL` queue via `utils.Requeue`,
while LMS just Nacks. And even the two that do republish, republish to queues that are **not
registered anywhere in this consumer repo's `QueueFuncMap`** — meaning either those fail-queues
are dead-ends, consumed by another service entirely, or drained manually. The previous shallow
doc did not mention this at all.

**Race-condition guard** (re-verified this pass): all 3 consumers' UPDATE queries use
`WHERE FK_GLUSR_USR_ID=$N AND GLUSR_LOGO_APPROVAL_STATUS = 'P'` whenever an `APPROVAL_ID` is
present in the message — [`USER_COMPSYNC_LOGO_IMSDB.go:151`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_COMPSYNC_LOGO_IMSDB.go), [`USER_COMP_SYNC_LOGO.go:173`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_COMP_SYNC_LOGO.go), [`USER_LOGO_LMSPG.go:143`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_LOGO_LMSPG.go) — stale approve/reject messages silently no-op if the record moved on already.

---

## 6. Kafka

**No Kafka usage found** for Logo. Confirmed this pass by (a) reading `rabbitmq.go` in full —
`COMPANY_LOGO_SERVICE`'s `exchangeSet` entry (`"USER.topic"`) is a **RabbitMQ topic-exchange
name**, not a Kafka topic (same map pattern as every other RabbitMQ-routed service in that
file, e.g. `GST_LAST_MODIFIED`, `NEW_COMPANY`), and (b) a repo-wide grep for `logo`/`LOGO`
across `user-temp-consumers-production` found no Kafka-related file (`InitializeKafka` is used
elsewhere in that repo, e.g. GST bulk sync, but never for logo).

---

## 7. Redis

**No Redis usage found** for Logo — grepped `UserLogoModel.go`, `UserLogoController.go`, and
all 3 consumer files for `redis`/`Redis`, zero matches. Every write and the one demonstrable
direct read (`buyerprofile` JOIN) hit Postgres directly. Caching gap discussed in section 11.

---

## 8. Business Rules & Validation (exhaustive, code-cited)

1. **Gateway allowlist restricted to `MY`/`GLADMIN`** — only these two caller-types can hit
   `POST /user/logo` at all.
   [`UserLogoController.go:50`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserLogoController.go)
2. **New logo always starts `Pending` — status is not client-controlled on insert.** On
   `FLAG="I"`, `GLUSR_LOGO_APPROVAL_STATUS` is hardcoded `'P'` in the INSERT statement itself,
   regardless of any `STATUS` value the caller sends.
   [`UserLogoModel.go:96`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserLogoModel.go)
3. **Insert requires all 3 core image sizes to be present** (`img90`, `img120`, `imgorig` all
   mandatory) — the mandatory-check rejects an insert missing any of them with
   `"Please Enter Mandatory(...Image90x90/Image120x120/Image Original/FLAG) Fields"`.
   `GLUSR_USR_LOGO_IMG_250X250` is stored but **not** in this mandatory list — the 250x250
   size can be submitted blank on insert.
   [`UserUtilsMandatory.go:1560-1561`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
4. **Rejection requires a reason.** If `STATUS == "R"`, `REJECT_REASON` is mandatory —
   `"Please specify rejection reason in case of rejection status"`.
   [`UserUtilsMandatory.go:1562-1563`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
5. **Deletion requires `STATUS == "D"` to match `FLAG == "D"`** — `"For Deletion Status should
   be D"` if they mismatch. Enforced purely as an input-consistency check; the actual SQL
   branch that runs on deletion doesn't re-check `STATUS`, it keys off `FLAG` only (section 4).
   [`UserUtilsMandatory.go:1570-1571`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
6. **`FLAG` can only be `I`, `U`, or `D`** when `APPROVAL_ID` is empty — anything else is
   rejected with `"Value of flag can only be D or U or I"`.
   [`UserUtilsMandatory.go:1566-1567`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
7. **When `APPROVAL_ID` is present (approve/reject path), a different mandatory set applies**:
   `USR_ID`, `UPDATEDBY`, `VALIDATION_KEY`, `IP`, `IP_COUNTRY`, `MODULE`, `STATUS` are all
   required — `FLAG` is irrelevant here.
   [`UserUtilsMandatory.go:1558-1559`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
8. **Approve/reject only applies to still-Pending records** — the SQL WHERE clause
   additionally requires `GLUSR_LOGO_APPROVAL_STATUS = 'P'`, so a second approve/reject call
   against an already-decided `APPROVAL_ID` is a silent no-op (no rows affected, but the
   controller still reports `"UPDATE SUCCESS"` because `err` is nil — masks the no-op at the
   API-response level, see section 13).
   [`UserLogoModel.go:162,166`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserLogoModel.go)
9. **`"unique constraint"` DB errors are downgraded to a success-like message** — `output`
   becomes `"Record already exist"` rather than a raw DB error, and the controller treats this
   as HTTP 200/`SUCCESSFUL` alongside genuine `"UPDATE SUCCESS"`.
   [`UserLogoModel.go:210-211`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserLogoModel.go), [`UserLogoController.go:77`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserLogoController.go)
10. **Employee-name resolution on internal updates** — if `UPDATEDBY_ID` is present, an extra
    sequential DB lookup (`Employee_mesh_pg`) resolves it to a human-readable name before the
    main upsert, and the stored `UPDATEDBY` becomes `"<name> (<id>)"`.
    [`UserLogoModel.go:31-46`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserLogoModel.go)
11. **All 4 image-size fields are capped at 500 characters** (`type: string, length: 500`) —
    consistent with storing a URL/path rather than binary image data; the resize itself is not
    performed by this feature — it's given pre-generated URLs for 4 sizes.
    [`UsersValidationMaps.go:717-720`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go)
12. **RabbitMQ publish only fires on a successful DB write** — `rabbitarray["MESSAGE"]` is only
    populated inside the success branches, so a failed DB write never reaches `PushToQueue`.
    Fan-out consumers can never receive a message for a write that didn't actually happen.
    [`UserLogoModel.go:148-152,193-197,215`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserLogoModel.go)

---

## 9. End-to-End Technical Flows

### Flow A — Supplier submits a new logo (Insert)

```
Supplier (seller panel/app)
    │
    ▼
[API — write]  POST /user/logo  {FLAG:"I", USR_ID, IMG_FORM_90x90, IMG_FORM_120x120,
                                  IMG_FORM_ORI, IMG_FORM_250x250 (optional), VALIDATION_KEY,...}
    │  UserLogoController.go
    │  1. Gateway check (MY/GLADMIN only)
    │  2. Mandatory-field check (all 3 required image sizes + FLAG)
    │  3. Length/type validation (UserCompanyLogoMap)
    ▼
UpsertLogo()
    │  UPDATEDBY_ID present? → Employee_mesh_pg() lookup first (extra DB round-trip)
    ▼
upsert_logo_pg()  — APPROVAL_ID empty, FLAG="I"
    ▼
[DB — meshpg]  INSERT INTO GLUSR_USR_LOGO (... status forced 'P' ...) RETURNING GLUSR_LOGO_APPROVAL_ID
    ▼
[RabbitMQ publish]  SERVICENAME=COMPANY_LOGO_SERVICE → queue user.logo.<glid%20>, exchange USER.topic
    ▼
[CONSUME — 3-way parallel fan-out, independent queues, no ordering guarantee between them]
    ├─ USER_COMPSYNC_LOGO_IMSDB  → INSERT into GLUSR_USR_LOGO on searchPg
    ├─ USER_LOGO_LMSPG           → INSERT into GLUSR_USR_LOGO on lmsPg
    └─ USER_COMP_SYNC_LOGO       → INSERT into GLUSR_USR_LOGO on meshPg (2nd connection)
```

### Flow B — Supplier updates an existing logo's images (Update, pre-approval-cycle)

```
POST /user/logo  {FLAG:"U", USR_ID, STATUS, IMG_FORM_ORI, IMG_FORM_90x90, IMG_FORM_120x120,
                   IMG_FORM_250x250, REJECT_REASON (if STATUS="R"), ...}
    │  Mandatory check: USR_ID/VALIDATION_KEY/UPDATEDBY/IP/IP_COUNTRY/MODULE/STATUS required
    ▼
upsert_logo_pg()  — APPROVAL_ID empty, FLAG="U"
    ▼
[DB — meshpg]  UPDATE GLUSR_USR_LOGO SET status/images/reject-reason... WHERE FK_GLUSR_USR_ID=$N
    ▼
[RabbitMQ + 3-way fan-out]  same as Flow A
```

**Caveat**: unlike insert, update does **not** force status back to `'P'` in code — whatever
`STATUS` the caller sends is written directly. Whether the calling UI always resets to `'P'`
on a supplier-initiated resubmission is not verifiable from this file — Open Question.

### Flow C — Admin/moderation approves or rejects a Pending logo

```
POST /user/logo  {APPROVAL_ID: "<id>", USR_ID, STATUS:"A"|"R", REJECT_REASON (if "R"),
                   UPDATEDBY, VALIDATION_KEY, IP, IP_COUNTRY, MODULE}
    │  Mandatory check: different rule-set fires because APPROVAL_ID is present
    ▼
upsert_logo_pg()  — APPROVAL_ID present branch
    │
    ├─ STATUS == "R" → UPDATE ... SET GLUSR_LOGO_APPROVAL_STATUS='R', REJECTION_REASON=...
    │                    WHERE FK_GLUSR_USR_ID=$N AND GLUSR_LOGO_APPROVAL_ID=$M
    │                    AND GLUSR_LOGO_APPROVAL_STATUS='P'   ← only affects still-Pending rows
    │
    └─ STATUS != "R" → UPDATE ... SET GLUSR_LOGO_APPROVAL_STATUS='A', ...
                         (same WHERE guard)
    ▼
[RabbitMQ + 3-way fan-out]  same as Flow A — downstream replicas learn the approval/rejection
```

**Important**: no separate admin controller exists for this — it is the *same*
`POST /user/logo` endpoint, distinguished only by the presence of `APPROVAL_ID` in the
request body. Which internal tool/screen actually calls this with `APPROVAL_ID` set (i.e.
the actual moderation UI) is **not traceable from this repo** — Open Question.

### Flow D — Logo deleted (soft-delete)

```
POST /user/logo  {FLAG:"D", STATUS:"D", USR_ID, UPDATEDBY, VALIDATION_KEY, IP, IP_COUNTRY, MODULE}
    ▼
upsert_logo_pg()  — APPROVAL_ID empty, FLAG neither "I" nor "U" (validated as "D" upstream)
    ▼
[DB — meshpg]  UPDATE GLUSR_USR_LOGO SET APPROVAL_STATUS='D', all 4 image columns = NULL,
               REJECTION_REASON = NULL WHERE FK_GLUSR_USR_ID = $N
    ▼
[RabbitMQ + 3-way fan-out]  same as Flow A — downstream replicas null out their copies too
```

### Flow E — Logo surfaces on reads (indirect, no dedicated endpoint)

```
buyerprofile/*params  (GET/POST) ──► ActionBuyerProfile ──► GetBuyerProfileModel /
                                       getBuyerProfileData
                                       │
                                       ▼
                            LEFT JOIN GLUSR_USR_LOGO L ON L.FK_GLUSR_USR_ID = CONTACTS_GLID
                            SELECT GLUSR_USR_LOGO_IMG_90X90 (no approval-status filter seen
                            in the joined columns list — i.e. this join does NOT appear to
                            filter on GLUSR_LOGO_APPROVAL_STATUS='A' in the reviewed excerpt;
                            [INFERRED gap — confirm whether Pending/Rejected logos leak
                            through this join, see Open Questions])

Product/PDP & supplier-listing (SID) responses ──► pre-aggregated dataset with
                                       glusr_usr_logo_img_90x90 / glusr_logo columns
                                       (NOT a live join in the reviewed code — likely reads
                                       from the searchPg replica that USER_COMPSYNC_LOGO_IMSDB
                                       keeps in sync — [INFERRED, not conclusively traced])
```

---

## 10. Flow-wise DB & Table Usage

### Flow A — Insert (new logo)

| # | DB (physical) | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | (employee lookup, exact table not identified) | SELECT | Only if `UPDATEDBY_ID` present — resolves internal-employee name for the audit trail |
| 2 | meshpg | `GLUSR_USR_LOGO` | INSERT ... RETURNING `GLUSR_LOGO_APPROVAL_ID` | Primary write — creates the Pending logo record, forced `status='P'` |
| 3 | searchPg | `GLUSR_USR_LOGO` | INSERT (consumer: `USER_COMPSYNC_LOGO_IMSDB`) | Keeps the search/IMSDB replica's copy in sync |
| 4 | lmsPg | `GLUSR_USR_LOGO` | INSERT (consumer: `USER_LOGO_LMSPG`) | Keeps LMS's copy in sync |
| 5 | meshPg (2nd, consumer-owned connection) | `GLUSR_USR_LOGO` | INSERT (consumer: `USER_COMP_SYNC_LOGO`) | General company-sync replica — a *second*, independently-connected copy separate from the write-path's own meshpg connection |

**Total round-trips for one Insert: 5** (1 conditional employee-lookup + 1 write-path insert +
3 independent consumer inserts across 3 physical DBs). meshpg is written to from **two
separate code paths** (write API + general-sync consumer's own connection) — worth noting as
a potential double-write-path oddity (section 13).

### Flow B — Update (image/metadata resubmission)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | (optional) employee lookup | SELECT | Same as Flow A, conditional |
| 2 | meshpg | `GLUSR_USR_LOGO` | UPDATE (images + status + reject-reason) | Primary write |
| 3-5 | searchPg / lmsPg / meshPg | `GLUSR_USR_LOGO` | UPDATE (3 consumers) | Same 3-way fan-out as Flow A |

### Flow C — Approve/Reject

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `GLUSR_USR_LOGO` | UPDATE, guarded by `AND GLUSR_LOGO_APPROVAL_STATUS='P'` | Transitions Pending → Approved/Rejected; no-ops if already decided |
| 2-4 | searchPg / lmsPg / meshPg | `GLUSR_USR_LOGO` | UPDATE (3 consumers), same `'P'`-guard replicated in each consumer's own SQL | Fan-out the approval/rejection decision |

### Flow D — Delete (soft)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `GLUSR_USR_LOGO` | UPDATE (status→`'D'`, all 4 image columns → NULL) | Primary soft-delete |
| 2-4 | searchPg / lmsPg / meshPg | `GLUSR_USR_LOGO` | UPDATE (3 consumers) | Fan-out the deletion |

### Flow E — Reads

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | read-API's DB (buyerprofile query) | `GLUSR_USR_LOGO` (LEFT JOIN) | SELECT `GLUSR_USR_LOGO_IMG_90X90` | Only demonstrable direct read of the primary table found this pass |
| 2 | likely searchPg (IMSDB replica) | Unidentified pre-aggregated dataset/view exposing `glusr_usr_logo_img_90x90`/`glusr_logo` | SELECT | Product/PDP and supplier-listing (SID) responses — **[INFERRED]**, not conclusively traced to a specific table/view this pass |

---

## 11. Optimization Scope — DB Response-Time Contribution

### Medium-impact

1. **No caching anywhere in this feature** (section 7) — every read path (`buyerprofile` join,
   and whatever backs the PDP/SID datasets) hits Postgres live. Logo changes are infrequent
   per-supplier and go through an approval workflow, making this a reasonable Redis
   cache-aside candidate — no quantification possible without traffic data, flagged as
   opportunity only.
2. **`Employee_mesh_pg` lookup is a sequential pre-step** before the main upsert whenever
   `UPDATEDBY_ID` is present (section 8, point 10). If internal-tool-driven approve/reject
   actions are frequent, this could be removed from the critical path (e.g. cache
   employee-name lookups, or resolve async).
   [`UserLogoModel.go:31-46`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserLogoModel.go)
3. **Two of three fan-out consumers have unmonitored fail-queues** (section 5) — messages
   pushed to `USER_COMPSYNC_LOGO_IMSDB_FAIL`/`USER_COMP_SYNC_LOGO_FAIL` have no registered
   consumer in this repo's `QueueFuncMap`. If these queues aren't drained elsewhere, this is a
   **silent data-consistency risk** (replicas can permanently diverge from meshpg on any
   transient failure), not strictly a response-time issue but adjacent enough to flag here.

### Low-impact / good practice already present

4. **Approve/reject SQL already guards against races** with `AND GLUSR_LOGO_APPROVAL_STATUS =
   'P'` in both the write-path and all 3 consumers — a good defensive pattern, consistently
   applied.
5. **`"Record already exist"` treated as success** — idempotent-friendly, avoids
   client-retries surfacing as hard errors.
6. **RabbitMQ publish gated on successful DB write** — no wasted downstream fan-out on a
   failed write.

---

## 12. Cron Inventory

**No cron jobs found for Logo.** Grepped `user-temp-consumers-production` for
`cron`/`Cron` restricted to logo-named files — zero matches, and no logo-specific entry in
the repo's cron-shell-wrapper conventions (contrast with GST's `gst_tact_veri_cron.go`, which
has no Logo equivalent). This feature is entirely event-driven off the write API.

---

## 13. Edge Cases & Gotchas (technical POV)

1. **A no-op approve/reject still reports `"UPDATE SUCCESS"`** (section 8, point 8) — if
   `APPROVAL_ID` is present but the record already moved past `'P'`, the guarded UPDATE
   affects zero rows, `err` stays `nil`, and the controller returns HTTP 200. Callers cannot
   distinguish "I approved it" from "someone already approved/rejected it and this was a
   no-op" from the response alone.
2. **`meshpg` is written by two independent code paths** — the write-API's own `UpsertLogo`
   call, and the `USER_COMP_SYNC_LOGO` consumer's separate connection back to `meshPg`. Both
   ultimately touch the same table on the same physical database via different
   connections/pools — worth understanding if debugging a meshpg-side write-order or
   lock-contention issue.
3. **Update path does not force status back to `'P'`** (section 9, Flow B caveat) — a
   resubmission after rejection could theoretically leave stale status values if the calling
   UI doesn't explicitly reset `STATUS`. Not verifiable as a bug from code alone.
4. **Two of three consumers republish to fail-queues that have no registered consumer**
   (section 5, 11) — `USER_COMPSYNC_LOGO_IMSDB_FAIL` and `USER_COMP_SYNC_LOGO_FAIL` are
   pushed to but not drained anywhere found in this repo.
5. **The LMS consumer's failure handling differs from its two siblings** — it just Nacks
   instead of explicitly republishing to a fail-queue (section 5). Inconsistent behavior
   across 3 otherwise near-identical consumers.
6. **`buyerprofile`'s `GLUSR_USR_LOGO` join has no visible approval-status filter** in the
   reviewed excerpt (section 9, Flow E) — if true, Pending or Rejected logo images could be
   exposed through this read path. **[INFERRED — confirm with team, needs a closer read of
   the full `getBuyerProfileData` function beyond what this pass covered]**.
7. **Image resizing itself is out of scope for this feature** — `GLUSR_USR_LOGO_IMG_*` columns
   store URLs/paths (max 500 chars each), implying the 4 sizes are generated by an upstream
   image-processing step before hitting this API; this feature only persists references.

---

## 14. Open Questions

1. Which internal tool/screen calls `POST /user/logo` with `APPROVAL_ID` set (the actual
   moderation/approval UI)? Not traceable from `service-api-go-production` or
   `user-temp-consumers-production` alone.
2. Does the `buyerprofile` query's `GLUSR_USR_LOGO` join filter on
   `GLUSR_LOGO_APPROVAL_STATUS='A'` anywhere outside the excerpt reviewed this pass (section
   9, Flow E)? Needs a full read of `getBuyerProfileData` in
   [`UserBuyerProfileModel.go`](../../users-api-go-production/internal/models/users/UserBuyerProfileModel.go).
3. What exact table/view backs `glusr_usr_logo_img_90x90`/`glusr_logo` in the PDP
   ([`UsersPdp.go`](../../users-api-go-production/internal/models/products/UsersPdp.go)) and
   supplier-listing ([`SID_Detail_model.go`](../../users-api-go-production/internal/models/master/SID_Detail_model.go))
   responses? Strongly suspected to be the searchPg IMSDB replica this feature's own
   `USER_COMPSYNC_LOGO_IMSDB` consumer maintains, but not conclusively joined/traced in this
   pass.
4. Are `USER_COMPSYNC_LOGO_IMSDB_FAIL` and `USER_COMP_SYNC_LOGO_FAIL` consumed by anything
   (another repo, a manual ops process)? If not, failed replica-syncs may silently accumulate
   forever.
5. Does the calling UI reset `GLUSR_LOGO_APPROVAL_STATUS` to `'P'` on a supplier resubmission
   (`FLAG="U"`) after a rejection, or can a stale status persist? Not enforced in code.
6. Live DB schema verification (column types, nullability, indexes, constraints) for
   `GLUSR_USR_LOGO` — this doc only reflects what the Go SQL strings imply.

---

## See also

- [`Logo_Business_Doc.md`](./Logo_Business_Doc.md) — product perspective, same flows, no code
- [`../Image KT/Image_Technical_Doc.md`](../Image%20KT/Image_Technical_Doc.md) — related but
  separate general-image system (no approval-ID, no consumer fan-out)
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — depth/structure
  reference this doc was rebuilt to match
- [`../users_domain_product_stories.md`](../users_domain_product_stories.md) — Story 2
  "Profile & Business Identity Management"
- [`../user_consumers_reference.md`](../user_consumers_reference.md) — full consumer
  inventory across the domain

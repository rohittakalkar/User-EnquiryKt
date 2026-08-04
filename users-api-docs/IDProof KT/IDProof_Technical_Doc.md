# ID Proof — Technical Doc (Code-Level Deep Dive)

Yeh doc ID Proof feature ka **technical implementation** cover karta hai — routes, DB
table, validation, aur flows, sab code se verify karke. Business/product perspective
ke liye [`IDProof_Business_Doc.md`](./IDProof_Business_Doc.md) dekho — dono docs same
flows cover karte hain, bas alag audience ke liye.

**Repos**: `service-api-go-production` (write), `users-api-go-production` (read).
**Methodology**: har claim neeche real source code se trace kiya gaya hai (file path +
line number diya gaya hai har jagah). Jahan code se pura confirm nahi ho paaya, wahan
clearly **[INFERRED — team se confirm karo]** likh diya hai, guess nahi kiya.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write controller (insert/update) | write | [`IdProofController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/IdProofController.go) |
| Write model (insert/update SQL + select-then-branch) | write | [`IdProofModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/IdProofModel.go) — `IdProofModel()`, `CheckIdproofAction()` |
| Write — field-level validation map | write | [`UsersValidationMaps.go:440-454`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) — `IdProofMap` |
| Write — validation function | write | [`UsersValidationMaps.go:2241-2261`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) — `ValidationIdProof()` |
| Write — route registration | write | [`router.go:148,316`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) — `POST /user/idproof` |
| Read controller | read | [`IdProofController.go`](../internal/controllers/UsersControllers/IdProofController.go) — `ActionIdProof()` |
| Read model | read | [`UsersIdProofModel.go`](../internal/models/users/UsersIdProofModel.go) — `GetIdProofModel()` |
| Read — route registration | read | [`routerUsers.go`](../internal/api/users_router/routerUsers.go) (lines 266-267, 506-507, 646-647), [`routerUsers.go`](../internal/api/router/routerUsers.go) (lines 159-160) — `GET/POST idproof/*params` |

**Consumer/cron check**: grepped `IDPROOF|IdProof|IdentityProof|IDENTITY_PROOF` (case
variations) across `user-temp-consumers-production` in full — **zero matches**. No
consumer, worker, or cron in that repo touches ID Proof in any way. This is a purely
synchronous, two-repo (write + read) feature — same shape as Blocking, Social Review,
and Popup Details modules referenced in the previous shallow doc.

---

## 2. Routes

| Method | Path | Repo | Controller | serviceName |
|---|---|---|---|---|
| POST | `/user/idproof` | write (`service-api-go-production`) | `IdProofController` | `IDPROOF` — [`IdProofController.go:26`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/IdProofController.go) |
| GET, POST | `idproof/*params` | read (`users-api-go-production`) | `ActionIdProof` | `IDPROOF` — [`IdProofController.go:32`](../internal/controllers/UsersControllers/IdProofController.go) |

Read route is registered identically in three router files
(`internal/api/users_router/routerUsers.go` twice, `internal/api/router/routerUsers.go`
once) — this mirrors the multi-binary router-duplication pattern already seen in other
KTs in this repo (multiple `main.go` binaries share route tables).

---

## 3. Data Model — Table

> **Verification note**: table/column names below come from SQL strings embedded in Go
> code (file:line cited per row). This has **not** been cross-checked against a live DB
> schema (pgAdmin/psql `\d`) — do that before relying on nullability, index, or
> constraint assumptions.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_USR_IDENTITY_PROOFS` | write: `meshpg` ([`IdProofController.go:98`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/IdProofController.go)) / read: `mesh_pg_user` ([`UsersIdProofModel.go:20`](../internal/models/users/UsersIdProofModel.go)) | Per-user, per-proof-type identity-verification record | `FK_GLUSR_USR_ID` (numeric, len 10), `FK_IIL_IDENTITY_PROOF_ID` (numeric, len 2 — proof-type, composite key with user-ID), `IDENTITY_PROOF_NUMBER` (string, len 50), `IDENTITY_PROOF_REF_URL` (string, len 255), `APPROVAL_STATUS` (numeric, len 1, default `0`), `ADDED_DATE`, `MODIFIED_DATE` (both `CURRENT_TIMESTAMP` on write), `UPDATEDBY_ID`, `UPDATEDBY_SCREEN`, `UPDATEDBY_URL`, `UPDATEDBY_IP`, `UPDATEDBY_IP_COUNTRY`, `HIST_COMMENTS`, `UPDATED_BY`, `UPDATEDUSING` — [`IdProofModel.go:61,86-101`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/IdProofModel.go), [`UsersValidationMaps.go:440-453`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |

**No master/lookup table for `FK_IIL_IDENTITY_PROOF_ID` (proof-type)** was found in
either repo — grepped `IIL_IDENTITY_PROOF|identity_proof_master|IDENTITY_PROOF_MASTER`
in both `service-api-go-production` and `users-api-go-production`; the only hits are the
foreign-key column name itself in `IdProofModel.go`/`UsersIdProofModel.go`/
`UsersValidationMaps.go`. Valid proof-type ID values (e.g. what `1` vs `2` means —
Aadhaar? PAN card? Passport?) are **[INFERRED — confirm with team, not found in code]**.

---

## 4. Decode This: `APPROVAL_STATUS` values

Unlike GST's `FK_GST_VERIFICATION_SRC_ID` (5 distinct values with a rich state machine),
`APPROVAL_STATUS` here is a much simpler binary flag, confirmed directly from the
validation function itself — no need to infer from log strings:

| Value | Meaning | Evidence |
|---|---|---|
| `0` | Default / not yet approved (pending) | Insert-time default when `APPROVAL_STATUS` not supplied — [`IdProofModel.go:30-34`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/IdProofModel.go). Read-side also normalizes any falsy/non-numeric value back to `"0"` — [`UsersIdProofModel.go:57-65`](../internal/models/users/UsersIdProofModel.go) |
| `1` | Approved | Only other value the validator accepts — [`UsersValidationMaps.go:2247`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go): `!(APPROVAL_STATUS == "1" || APPROVAL_STATUS == "0")` → rejected as `lengthExceeded` |

**No explicit "rejected" value exists in the validator** — only `0` and `1` are legal
inputs on write. Any other numeric or non-numeric value fails validation with a
`"...length exceeded."` message (a slightly misleading error string for what is really
an enum-membership check — worth flagging to the team, [`UsersValidationMaps.go:2247-2248`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go)).
So from the write API's point of view this is a **binary pending/approved flag, not a
tri-state approve/reject/pending workflow** — this contradicts the assumption implied by
the old shallow doc's "approved/pending/rejected" language. **[INFERRED — confirm with
team whether rejection is modeled elsewhere, e.g. by deleting the row or setting some
other field not visible in this code path]**.

**Read-side normalization quirk**: `GetIdProofModel()` converts the stored
`approval_status` string to an `int` in the response **only if it's non-zero**; if it's
`"0"`, zero, or fails `strconv.Atoi`, the response field is forced back to the **string**
`"0"` rather than the int `0` — meaning API consumers may see a type-inconsistent field
(`int` when approved, `string "0"` when pending). [`UsersIdProofModel.go:57-65`](../internal/models/users/UsersIdProofModel.go)

---

## 5. Business Rules & Validation (code se exhaustive list)

1. **Mandatory fields on write**: `FK_GLUSR_USR_ID`, `FK_IIL_IDENTITY_PROOF_ID`,
   `IDENTITY_PROOF_REF_URL`, `UPDATED_BY`, `UPDATEDUSING` — if any is empty, request
   fails before any DB call with `"Please Enter Mandatory Fileds (...)"` (note: typo
   "Fileds" is in the actual error string).
   [`IdProofController.go:84-85`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/IdProofController.go)
2. **Gateway allowlist**: writers must be `GLADMIN`, `MAPI`, `ANDROID`, or `IOS` —
   enforced via `Gateway_v1()` against `components.ValidationHashMap`, keyed off the
   `VALIDATION_KEY` request field. If `VALIDATION_KEY` is non-empty but doesn't resolve
   to an allowed caller, request is rejected with `"Input gateway not validated"`.
   Notably, if `VALIDATION_KEY` is **empty**, this check is effectively bypassed
   (`gateway == "Input gateway not validated" && validationKey != ""` — the `&&`
   means an empty key skips the block) — **[INFERRED — worth confirming with team
   whether this is an intentional back-door for trusted internal callers or an
   oversight]**. [`IdProofController.go:87-90`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/IdProofController.go)
3. **Field-level validation** (`ValidationIdProof`): type (`number`/`string`) and length
   checks per `IdProofMap` — `FK_GLUSR_USR_ID` numeric ≤10 digits, `FK_IIL_IDENTITY_PROOF_ID`
   numeric ≤2 digits, `IDENTITY_PROOF_NUMBER` string ≤50 chars, `IDENTITY_PROOF_REF_URL`
   string ≤255 chars, `APPROVAL_STATUS` numeric ≤1 digit and must literally be `"0"` or
   `"1"` (section 4). [`UsersValidationMaps.go:440-453,2241-2260`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go)
4. **Insert-vs-update decided by a pre-check `SELECT`, not `ON CONFLICT`**:
   `CheckIdproofAction()` first queries
   `WHERE FK_GLUSR_USR_ID=$1 AND FK_IIL_IDENTITY_PROOF_ID=$2`; if any row is returned →
   action=`"UPDATE"` (and the row's current values are captured into `old`), else →
   action=`"INSERT"` — a **select-then-insert/update pattern**, not an atomic upsert.
   [`IdProofModel.go:127-184`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/IdProofModel.go)
5. **On UPDATE, unset fields fall back to the previously-fetched `old` values** — genuine
   partial-update semantics for `FK_IIL_IDENTITY_PROOF_ID`, `IDENTITY_PROOF_NUMBER`,
   `IDENTITY_PROOF_REF_URL`, `APPROVAL_STATUS` (each individually checked with
   `KeyPresent`, falling back to `old[...]` if absent from the request).
   [`IdProofModel.go:65-85`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/IdProofModel.go)
6. **On INSERT, `APPROVAL_STATUS` defaults to `0`** (the Go int literal `0`, not string)
   if not explicitly provided in the request.
   [`IdProofModel.go:30-34`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/IdProofModel.go)
7. **`(FK_GLUSR_USR_ID, FK_IIL_IDENTITY_PROOF_ID)` is the effective composite key** for
   both the pre-check select and the update's `WHERE` clause — a single user can have
   multiple `GLUSR_USR_IDENTITY_PROOFS` rows, one per `FK_IIL_IDENTITY_PROOF_ID` (proof
   type). No DB-level unique-constraint enforcement was found in code (see Open
   Questions).
8. **Empty string values are converted to SQL NULL before insert/update**, via
   `utils.ConvertEmptyToNull(params)` on both the INSERT and UPDATE param lists —
   applies uniformly to all fields, not just ID Proof specific ones.
   [`IdProofModel.go:64,104`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/IdProofModel.go)
9. **Read side has independent token/modid validity checks** distinct from the write
   side's gateway allowlist — `utils.CheckValidity([]string{"token","modid"}, ...)` with
   optional-case rules `token: [1,2,3]`, `modid: [5,5,3]`; also requires both `glusrid`
   and `proof_type_id` query params to be present **and** integer-parseable, else request
   fails before hitting the DB (`"GLUSRID"` error family, code paths `7/1/42` for missing,
   `7/1/43` for non-numeric). [`IdProofController.go:44-64`](../internal/controllers/UsersControllers/IdProofController.go)

---

## 6. RabbitMQ

**Koi RabbitMQ usage nahi mila.** Grepped `IDPROOF|IdProof|IdentityProof|IDENTITY_PROOF`
plus generic `PushToQueue|SERVICENAME|amqp` context in both the write controller/model
and the read controller/model — no `PushToQueue`, `PubAPI`, or queue-publish call
appears anywhere in the ID Proof write or read path. Also confirmed via a
repo-wide grep of `user-temp-consumers-production` for any ID-Proof-related consumer —
zero matches (section 1). **Explicit statement: no queue is touched by this feature,
in either direction.**

---

## 7. Kafka

**Koi Kafka usage nahi mila.** Same grep sweep as RabbitMQ (section 6) turned up no
`InitializeKafka`, `sub_topic`, or `consumer_group` reference anywhere near the ID Proof
code paths, and no dedicated Kafka consumer exists for it in
`user-temp-consumers-production`.

---

## 8. Redis

**Koi Redis usage nahi mila.** Both the write path (`IdProofController.go` →
`IdProofModel.go`) and read path (`IdProofController.go` → `UsersIdProofModel.go`) go
straight to Postgres (`meshpg` / `mesh_pg_user`) on every request — no `RedisGet`/
`RedisSet`/cache-aside pattern appears in either file. This is the same "no caching
layer" gap already flagged in the GST KT for its read endpoints (see
[`GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) section 9) — worth the same
consideration if ID Proof read volume/latency ever becomes a concern (section 11 below).

---

## 9. End-to-End Technical Flows

### Flow A — Supplier/GLADMIN/MAPI submits or updates an ID proof

```
Caller (Android/iOS app, GLADMIN, or MAPI)
    │
    ▼
[API — write]  POST /user/idproof
                {FK_GLUSR_USR_ID, FK_IIL_IDENTITY_PROOF_ID, IDENTITY_PROOF_NUMBER,
                 IDENTITY_PROOF_REF_URL, APPROVAL_STATUS?, UPDATED_BY, UPDATEDUSING,
                 VALIDATION_KEY?}
    │  IdProofController.go
    │  1. Mandatory-field check (glusrid/idProofId/refUrl/updatedBy/updatedUsing)
    │  2. Gateway_v1() — allowlist check (GLADMIN/MAPI/ANDROID/IOS) if VALIDATION_KEY present
    │  3. ValidationIdProof() — type/length/enum checks (IdProofMap + APPROVAL_STATUS 0/1)
    ▼
[DB — connect]  meshpg (GetPGDbConnection)
    ▼
CheckIdproofAction()  — Query1
    │  SELECT FK_IIL_IDENTITY_PROOF_ID, IDENTITY_PROOF_NUMBER, IDENTITY_PROOF_REF_URL,
    │         APPROVAL_STATUS
    │  FROM GLUSR_USR_IDENTITY_PROOFS
    │  WHERE FK_GLUSR_USR_ID=$1 AND FK_IIL_IDENTITY_PROOF_ID=$2
    ├─ row found     → action="UPDATE", old-values captured into `old` map
    └─ no row found  → action="INSERT"
    ▼
IdProofModel()  — Query2
    ├─ action=INSERT → INSERT INTO GLUSR_USR_IDENTITY_PROOFS (...) — APPROVAL_STATUS
    │                    defaults to 0 if absent; ADDED_DATE/MODIFIED_DATE = CURRENT_TIMESTAMP
    └─ action=UPDATE → UPDATE ... SET ... WHERE (FK_GLUSR_USR_ID, FK_IIL_IDENTITY_PROOF_ID)
                         — unset fields fall back to `old` values (partial-update);
                           MODIFIED_DATE refreshed to CURRENT_TIMESTAMP
    ▼
Response  {"STATUS", "CODE", "MESSAGE": "UPDATE SUCCESS IN PG", "SERVICE_NAME": "IDPROOF",
           "RESPONSE_DATA": {...timings...}, "ACTION": "INSERT"|"UPDATE"}
```

### Flow B — Read an ID-proof record for a (user, proof-type) pair

```
Caller (any authenticated system with valid token/modid)
    │
    ▼
[API — read]  GET/POST idproof/*params  {glusrid, proof_type_id, token, modid, unique_id}
    │  IdProofController.go (read) — ActionIdProof()
    │  1. CheckValidity(["token","modid"]) — auth/session validity
    │  2. glusrid + proof_type_id presence + integer-parseability check
    ▼
GetIdProofModel()
    │  connect: mesh_pg_user
    │  SELECT fk_glusr_usr_id, fk_iil_identity_proof_id, identity_proof_number,
    │         identity_proof_ref_url, approval_status
    │  FROM glusr_usr_identity_proofs
    │  WHERE FK_GLUSR_USR_ID=$1 AND FK_IIL_IDENTITY_PROOF_ID=$2
    ├─ 0 rows → "NO_DATA_FOUND" response, Data=[]
    └─ 1 row  → approval_status normalized (int if non-zero, else string "0")
                → "DATA_FOUND" response, Data={...row...}
    ▼
[DB — mesh_pg_user]
    ▼
Response {"Data": {...single row...}}  (Kibana-logged via KibanaLogging)
```

---

## 10. Flow-wise DB & Table Usage

### Flow A — Write (Insert or Update)

| # | DB (physical) | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `GLUSR_USR_IDENTITY_PROOFS` | SELECT (`CheckIdproofAction`) | Decide INSERT vs UPDATE, aur agar UPDATE hai toh purani values capture karo partial-update fallback ke liye |
| 2 | meshpg | `GLUSR_USR_IDENTITY_PROOFS` | INSERT or UPDATE (`IdProofModel`) | Naya record likho, ya existing record ko partial-update semantics ke saath refresh karo |

**Total round-trips per write request: 2, both to the same single database (`meshpg`)** —
this is a much simpler footprint than GST's 9-10 round-trips across 4 databases; there
is no cross-database fan-out, no external API call, and no async queue publish for ID
Proof.

### Flow B — Read

| # | DB (physical) | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | mesh_pg_user | `glusr_usr_identity_proofs` | SELECT | Ek single row nikalna (user, proof-type) composite-key se |

**Total round-trips per read request: 1.**

---

## 11. Optimization Scope — DB Response-Time Contribution

### Medium impact

1. **Select-then-insert/update is 2 sequential round-trips per write-request**, both to
   `meshpg` — this is the exact "check-then-act" pattern already flagged as an
   optimization opportunity in the GST and other KTs. Converting to a single
   `INSERT ... ON CONFLICT (FK_GLUSR_USR_ID, FK_IIL_IDENTITY_PROOF_ID) DO UPDATE SET
   IDENTITY_PROOF_NUMBER = COALESCE(EXCLUDED.IDENTITY_PROOF_NUMBER, GLUSR_USR_IDENTITY_PROOFS.IDENTITY_PROOF_NUMBER), ...`
   would reduce this to 1 round-trip — but **requires** a DB-level unique constraint on
   `(FK_GLUSR_USR_ID, FK_IIL_IDENTITY_PROOF_ID)` to exist first (currently unconfirmed,
   see Open Questions), and would move the partial-update-fallback logic from Go
   (`old[...]` map) into SQL `COALESCE(...)`.
   [`IdProofModel.go:60-104,127-184`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/IdProofModel.go)

### Low impact

2. **No caching on the read endpoint** (section 8) — `GET idproof/*params` hits
   `mesh_pg_user` on every call. Given ID-proof data likely changes infrequently per
   (user, proof-type) pair, this is a reasonable caching candidate similar to GST reads,
   though at this feature's likely traffic volume it is a lower-priority item than the
   select-then-write round-trip above.
3. **Read query is a simple 2-column-composite-key single-row lookup** — inherently
   low-cost regardless of caching; not a concern at current scope.

**No external API call and no cross-database fan-out exist in this feature** (unlike
GST's synchronous BI API call) — so the select-then-write round-trip (item 1) is the
single largest concrete lever available here.

---

## 12. Cron Inventory

Grepped for `cron|scheduler` context around `IdProof|IdentityProof|IDENTITY_PROOF` across
all three repos (`service-api-go-production`, `users-api-go-production`,
`user-temp-consumers-production`, including the crons directories referenced in the GST
KT) — **no cron job related to ID Proof was found**. This matches the earlier finding
that no consumer touches ID Proof at all (section 1/6): it is a fully synchronous,
request-scoped feature with **zero background processing**.

---

## 13. Edge Cases & Gotchas (technical POV)

1. **Race-condition risk in select-then-insert pattern**: if two concurrent requests for
   the same `(FK_GLUSR_USR_ID, FK_IIL_IDENTITY_PROOF_ID)` arrive close together, both
   could get "no row found" from `CheckIdproofAction`, and both could then attempt
   `INSERT` — if no DB-level unique constraint exists on that pair, this could produce
   duplicate rows for the same user+proof-type. [`IdProofModel.go:127-184`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/IdProofModel.go)
2. **Read `mesh_pg_user` vs write `meshpg`** — different DB-alias names used by the read
   and write paths for (implied) the same table; unconfirmed whether these point at the
   same physical database or replicas — same open-question pattern flagged in other
   recent KTs in this doc set (Market Place URL, Social Contacts, Disposition, GST).
3. **`VALIDATION_KEY` empty-string bypasses the gateway allowlist check** (section 5.2) —
   `Gateway_v1()`'s failure branch only fires when `validationKey != ""`, so a caller
   that simply omits `VALIDATION_KEY` skips the GLADMIN/MAPI/ANDROID/IOS restriction
   entirely, relying only on the (separate) mandatory-field check to have gotten that
   far. Worth a security-hygiene look. [`IdProofController.go:87-90`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/IdProofController.go)
4. **`APPROVAL_STATUS` is a strict binary `0`/`1` on write, but the read side treats `0`
   specially (string) and non-zero as `int`** (section 4) — API consumers parsing this
   field need to handle both types, or normalize client-side.
5. **Misleading validation error message**: an out-of-range `APPROVAL_STATUS` (anything
   other than `"0"`/`"1"`) is reported to the caller as `"...length exceeded."`, even
   though the actual failure is an enum-membership check, not a length check.
   [`UsersValidationMaps.go:2247-2248`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go)
6. **Typo in the mandatory-field error string** (`"Please Enter Mandatory Fileds"`) — low
   severity but confirms this exact string if grepping logs/monitoring for this failure
   mode. [`IdProofController.go:85`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/IdProofController.go)
7. **No proof-type master table found in code** — there is no visible validation that
   `FK_IIL_IDENTITY_PROOF_ID` corresponds to a real/known proof type; any value passing
   the numeric/length check (≤2 digits) will insert/update successfully, whether or not
   it maps to a real proof-type in whatever consumes this data downstream.

---

## 14. Open Questions

1. Does a DB-level unique constraint exist on `(FK_GLUSR_USR_ID, FK_IIL_IDENTITY_PROOF_ID)`
   in `GLUSR_USR_IDENTITY_PROOFS`? Could not confirm from Go code — this directly affects
   the race-condition risk in section 13.1 and the feasibility of the `ON CONFLICT`
   optimization in section 11.1.
2. Are `meshpg` (write) and `mesh_pg_user` (read) the same physical database, or
   replicas? If replicas, is there a replication-lag window where a just-written ID
   proof wouldn't yet be visible on read?
3. What is the master list of valid `FK_IIL_IDENTITY_PROOF_ID` values (proof types —
   Aadhaar, PAN, Passport, etc.)? No master/lookup table or enum was found in either repo.
4. Is `APPROVAL_STATUS` genuinely a binary pending(`0`)/approved(`1`) flag, or does
   "rejection" exist as a separate mechanism (e.g. row deletion, or a status tracked
   elsewhere) not visible in this code path? The validator only accepts `0`/`1`
   (section 4/5.3).
5. Who/what actually flips `APPROVAL_STATUS` from `0` to `1`? No dedicated
   admin-review-flow, consumer, or cron was found anywhere in the three repos searched —
   it appears this can only happen via the same `POST /user/idproof` write endpoint
   (e.g. a GLADMIN caller submitting `APPROVAL_STATUS=1`), but no such caller/flow was
   conclusively traced in this review.
6. Is the `VALIDATION_KEY`-empty gateway bypass (section 13.3) intentional (e.g. trusted
   internal-network callers) or an oversight?
7. Live DB schema verification (column types, nullability, indexes, constraints) for
   `GLUSR_USR_IDENTITY_PROOFS` — this doc only reflects what the Go SQL strings imply.

---

## See also

- [`IDProof_Business_Doc.md`](./IDProof_Business_Doc.md) — same flows, product/business
  perspective, bina code ke
- [`../Trust Verification KT/Trust_Verification_Business_Doc.md`](../Trust%20Verification%20KT/Trust_Verification_Business_Doc.md) /
  [`../Trust Verification KT/Trust_Verification_Technical_Doc.md`](../Trust%20Verification%20KT/Trust_Verification_Technical_Doc.md) —
  related but separate verification-workflow system (do not confuse the two — ID Proof
  is a narrower, single-table, single-flag feature; Trust Verification / TrustSeal is a
  much larger multi-status workflow)
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — a much richer
  verification-state feature in the same domain family, useful for contrast (RabbitMQ/
  Kafka/BigQuery/cron all present there, all absent here)

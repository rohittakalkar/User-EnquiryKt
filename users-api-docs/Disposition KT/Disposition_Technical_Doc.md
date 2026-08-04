# Disposition — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`Disposition_Business_Doc.md`](./Disposition_Business_Doc.md)
dekho.

**Scope note**: Disposition aur KWPL genuinely **do alag concepts** hain — alag tables
(`GLUSR_DISPOSITIONS` vs `PL_KWRD`), alag controllers, koi FK-relation ya shared-code nahi
mila. Dekho [`../KWPL KT/KWPL_Technical_Doc.md`](../KWPL%20KT/KWPL_Technical_Doc.md).

**Naming caveat**: `grep -i disposition` teeno repos mein do aur unrelated hits bhi deta hai —
`FK_RATING_DISPOSITION_ID` (Supplier Rating image-review, `UserSupplierRatingModel.go`) aur
`DISPOSITION_TYPE`/`DISPOSITION_TYPE_ACTIVITY` (Supplier Verification Log,
`IIL_SUPP_VERIFICATION_LOG` table, `UserSuppVerifyLogModel.go` +
`crons/recommend/Supp_verify_log_cron.go`). Dono **is doc ka scope nahi hain** — alag
tables, alag controllers, koi shared code `GLUSR_DISPOSITIONS`/`GL_DISPOSITIONS_MASTER` ke
saath nahi. Is doc mein "Disposition" hamesha `GLUSR_DISPOSITIONS` / `GL_DISPOSITIONS_MASTER`
wale feature ko refer karta hai.

**Repos**: `service-api-go-production` (write), `users-api-go-production` (read: attribute-
history read-endpoint, **aur** generic master-list read-endpoint — is baar dono cover kiye
gaye hain, purani doc mein master-list wala hissa miss ho gaya tha).

**Methodology**: har claim neeche real source code se trace kiya gaya hai. Jahan code se pura
confirm nahi ho paaya, wahan **[INFERRED — team se confirm karo]** likha hai.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write (record a disposition) | write | [`UserDispositionController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDispositionController.go), [`UserDispositionModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDispositionModel.go) (`UpsertDisposition`) |
| Validation (write) | write | `MandatoryParamsCheckDisposition` — [`UserUtilsMandatory.go:916`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go), `UserDispositionMap` — [`UsersValidationMaps.go:37-42`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) (`LengthAndTypeValidations_v3`) |
| Read — per-supplier attribute history | read | [`AttrDispositionController.go`](../internal/controllers/UsersControllers/AttrDispositionController.go), [`UserAttrDispositionModel.go`](../internal/models/users/UserAttrDispositionModel.go) (`GetAttrDispositions`) |
| Read — master-list (dropdown source, all attributes) **[new in this pass]** | read | [`GlmasterController/detail.go`](../internal/controllers/GlmasterController/detail.go), [`master/GlmasterDetail.go`](../internal/models/master/GlmasterDetail.go) (`detail_from=Disposition` branch, lines 70-71, 125-168) |
| Master-detail route auth-config | read | [`Validapi.go:150,196`](../pkg/components/Validapi.go) — gateway `"NA"`, authtype `"modid"` |

**Koi consumer/cron `GLUSR_DISPOSITIONS` ya `GL_DISPOSITIONS_MASTER` ko touch nahi karta** —
`user-temp-consumers-production` mein `grep -i disposition` sirf unrelated Supplier-Rating
consumers (`USER_RATING_NOTIFICATION_PG.go`, `USER_RATING_ARCHIVE.go`) return karta hai, jo
`FK_RATING_DISPOSITION_ID` use karte hain — is feature se koi lena-dena nahi (naming-caveat
upar dekho). Confirmed via direct grep, assumption nahi hai.

---

## 2. Routes

| Method | Path | serviceName | Repo | Controller |
|---|---|---|---|---|
| POST | `/user/dispositon` *(sic — codebase mein typo hai, "disposition" nahi)*, registered in `user_cluster2` aur (identical block) another cluster — [`router.go:154,322`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) | `USER_ATTR_DISPOSITION` | write | `UserDispositionController` |
| — | — | `USER_ATTR_DISPOSITIONS` | read | `AttrDisposition` |
| GET/POST | `/glmaster/detail/*params` with `detail_from=Disposition` — [`routerMaster.go:159-160`](../internal/api/master_router/routerMaster.go) | `Gl_Master_Detail` | read | `GlmasterController.GlmasterDetail` |

**Naya finding**: `/user/dispositon` do baar identical register hua hai (`router.go:154` aur
`:322`), ek gin route-group `user_cluster2` ke andar aur ek doosre cluster-setup-function ke
andar — yeh standard multi-cluster redundant-deployment pattern hai (baaki bahut saare routes
bhi isi tarah dono jagah dikhte hain), koi bug nahi, load-distribution/failover-clusters ke
liye hai. Route-path khud mein typo hai (`dispositon`, missing "i") — production mein already
live hai, isliye "fix" karna breaking-change hoga; sirf documentation ke liye flag kiya hai.

---

## 3. Data Model — Tables

> **Verification note**: yeh sab table/column names Go code ke andar embedded SQL strings se
> liye gaye hain (har row ke saamne file:line diya hai). Live DB schema se pgAdmin pe
> cross-verify **nahi** kiya gaya hai.

| Table | Physical DB (config-alias) | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_DISPOSITIONS` | `meshpg` (write) / `mesh_pg_user` (read, joined) | Per-supplier disposition/outcome-log, insert-only (history-preserving) | `FK_GLUSR_USR_ID`, `GLUSR_DISPOSITION_ADD_DATE` (`CURRENT_TIMESTAMP` on insert), `FK_GL_DISPOSITION_MASTER_ID`, `GLUSR_DISPOSITION_MODID` — [`UserDispositionModel.go:46-47`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDispositionModel.go) |
| `GL_DISPOSITIONS_MASTER` | `mesh_pg_user` (attribute-history read, joined) / `mesh_pg_with_product_user` (master-list read) | Master-list of valid disposition-codes, scoped per attribute via FK | `gl_disposition_master_id`, `gl_disposition` (the outcome text/code), `fk_gl_attribute_id`, `ENABLED` (used as filter `ENABLED = -1` in master-list query, i.e. `-1` means "true"/enabled in this codebase's boolean convention) — [`UserAttrDispositionModel.go:57`](../internal/models/users/UserAttrDispositionModel.go), [`GlmasterDetail.go:127`](../internal/models/master/GlmasterDetail.go) |
| `GL_ATTRIBUTE` | `mesh_pg_user` / `mesh_pg_with_product_user` (both joined) | Attribute master (resolves `gl_attribute_id` → human-readable name) | `gl_attribute_id`, `gl_attribute_column_name` (used in attribute-history read, hardcoded `= 2106` i.e. GST), `gl_attribute_display_name` (used in master-list read, joined for ALL attribute IDs, not filtered) |

**Key distinction the earlier doc missed**: the attribute-history read query
(`GetAttrDispositions`) hardcodes `fk_gl_attribute_id = 2106`, but the **master-list** read
query (`GlmasterDetail`, `detail_from=Disposition`) does **not** filter by attribute at all —
it returns every enabled disposition across every attribute, joined to that attribute's
display name. This confirms the schema/master-data really is multi-attribute by design; only
the attribute-*history* read endpoint is GST-locked.

---

## 4. Business Rules & Validation (code se, exhaustive)

1. **Gateway allowlist (write only)**: `GLADMIN`, `SELLERMY`, `IMOB`, `MSITE` —
   [`UserDispositionController.go:50`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDispositionController.go)
   (`utils.Gateway_v1([]string{"GLADMIN","SELLERMY","IMOB","MSITE"}, validationKey, "k")`).
2. **Mandatory (write)**: `GLID`, `MASTER_ID`, `ATTRIBUTE_NAME`, `DISPOSITION`,
   `VALIDATION_KEY` — all-or-nothing check, single combined error message.
   [`UserUtilsMandatory.go:916-946`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
3. **Length/type validation (write)**: `UserDispositionMap` enforces `GLID` numeric
   (len 10), `MASTER_ID` numeric (len 10), `ATTRIBUTE_NAME` string (len 20), `DISPOSITION`
   string (len 200), via shared `LengthAndTypeValidations_v3`.
   [`UsersValidationMaps.go:37-42`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go)
4. **Write is always an INSERT, never an UPDATE** — every disposition-submission creates
   a new row (`GLUSR_DISPOSITION_ADD_DATE = CURRENT_TIMESTAMP`), preserving full history
   per supplier; there is no upsert/dedupe logic despite the model function being named
   `UpsertDisposition`.
   [`UserDispositionModel.go:46`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDispositionModel.go)
5. **`ATTRIBUTE_NAME` and `DISPOSITION` are accepted/validated on write, but not stored
   directly in `GLUSR_DISPOSITIONS`** — only `MASTER_ID` (the disposition-master's ID) and
   `MODID` are persisted (`values["MASTER_ID"]`, `values["MODID"]`, `values["GLID"]` —
   `ATTRIBUTE_NAME`/`DISPOSITION` are read from `inputParams` for validation but never
   appear in the `values` map or the `INSERT` statement); the write-path trusts the caller
   to have already resolved `DISPOSITION`/`ATTRIBUTE_NAME` to the correct `MASTER_ID`
   (presumably via the master-list lookup, section 6 below).
   [`UserDispositionModel.go:12-27,46-47`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDispositionModel.go)
6. **No cross-check that `MASTER_ID` actually corresponds to `ATTRIBUTE_NAME`** — no DB
   read-before-write in `UpsertDisposition`; the insert is a blind write of whatever
   `MASTER_ID`/`GLID`/`MODID` the caller sent (after passing length/type validation).
7. **Read-side (attribute-history) is hardcoded to `gl_attribute_id = 2106` (GST)** —
   `GetAttrDispositions` explicitly checks `if attributeName != "GST" { return NO_DATA_FOUND }`
   before even querying — despite the master-list (section 6) being generically multi-attribute.
   [`UserAttrDispositionModel.go:25-33,57`](../internal/models/users/UserAttrDispositionModel.go)
8. **`all` param controls history-depth (attribute-history read)**: `all=1` → full history
   (all rows, no `ORDER BY`/`LIMIT` beyond insertion order); default (`all` absent, or any
   value other than the string `"1"`) → only the latest disposition
   (`ORDER BY glusr_disposition_add_date DESC LIMIT 1`).
   [`UserAttrDispositionModel.go:57-60`](../internal/models/users/UserAttrDispositionModel.go)
9. **Master-list read has no attribute filter and no allowlist gate** — `GlmasterDetail`
   (`detail_from=Disposition`) returns *every* `ENABLED = -1` row in `GL_DISPOSITIONS_MASTER`
   joined to `GL_ATTRIBUTE`, for any caller — route auth-config marks it gateway `"NA"`
   (no GLID-based gate) and authtype `"modid"` only.
   [`GlmasterDetail.go:125-127`](../internal/models/master/GlmasterDetail.go),
   [`Validapi.go:150,196`](../pkg/components/Validapi.go)
10. **`ENABLED = -1` is this codebase's boolean-true convention** — matches the pattern
    seen elsewhere in the `master` package (e.g. other `detail_from` branches); `0`/other
    values would exclude a disposition from the dropdown-list without deleting the row.

---

## 5. RabbitMQ

**No RabbitMQ usage found** in the write path (`UserDispositionModel.go` — no
`PushToQueue`/`PubAPI`/`Requeue` calls), the attribute-history read path, or the master-list
read path. Confirmed via direct grep for `PushToQueue|PubAPI|RabbitMQ` in
`UserDispositionModel.go` (zero matches) and via the consumer-repo grep in section 1 (zero
disposition-specific consumers exist to receive anything).

## 6. Kafka

**No Kafka usage found** — same grep, zero matches. No `InitializeKafka` reference
anywhere in the disposition write/read code paths.

## 7. Redis

**No Redis usage found** — zero `Redis`/`RedisGet`/`RedisSet` references in
`UserDispositionModel.go`, `AttrDispositionController.go`, `UserAttrDispositionModel.go`, or
`GlmasterDetail.go`. All three endpoints (write, attribute-history read, master-list read)
hit Postgres synchronously on every request, no caching layer.

**Conclusion**: Disposition is a purely synchronous, DB-only feature — no async messaging,
no consumers, no crons, no cache, in any of the three repos checked.

---

## 8. End-to-End Technical Flows

### Flow A — Write: internal-tool records a disposition

```
Internal-tool (GLADMIN / SELLERMY / IMOB / MSITE)
    │
    ▼
[API — write]  POST /user/dispositon  serviceName=USER_ATTR_DISPOSITION
                {GLID, MASTER_ID, ATTRIBUTE_NAME, DISPOSITION, MODID, VALIDATION_KEY}
    │  UserDispositionController.go
    │  1. utils.InputJsonCheck()
    │  2. MandatoryParamsCheckDisposition() — all 5 fields required
    │  3. Gateway_v1(["GLADMIN","SELLERMY","IMOB","MSITE"])
    │  4. LengthAndTypeValidations_v3(UserDispositionMap)
    ▼
UpsertDisposition()
    │  no read-before-write, no MASTER_ID<->ATTRIBUTE_NAME cross-check
    │  INSERT INTO GLUSR_DISPOSITIONS (FK_GLUSR_USR_ID, ADD_DATE=NOW(),
    │                                   FK_GL_DISPOSITION_MASTER_ID, MODID)
    ▼
[DB — meshpg]
    ▼
Response {"data inserted successfully" / "SUCCESS IN PG"}
```

### Flow B — Read: master-list lookup (dropdown population, any attribute)

```
Internal-tool / admin-screen
    │
    ▼
[API — read]  GET/POST /glmaster/detail/*params  serviceName=Gl_Master_Detail
                {detail_from: "Disposition", modid: "..."}
    │  GlmasterController/detail.go — no GLID-based gateway check (gateway="NA")
    ▼
master.GlmasterDetail()
    │  detail_from == "Disposition" → table_name = "GL_DISPOSITIONS_MASTER"
    │  SELECT A.GL_DISPOSITION_MASTER_ID, A.GL_DISPOSITION, B.GL_ATTRIBUTE_DISPLAY_NAME
    │  FROM GL_DISPOSITIONS_MASTER A, GL_ATTRIBUTE B
    │  WHERE GL_ATTRIBUTE_ID = FK_GL_ATTRIBUTE_ID AND ENABLED = -1
    │  (no attribute filter — every attribute's enabled dispositions returned)
    ▼
[DB — mesh_pg_with_product_user]
    ▼
Response {"GL_DISPOSITIONS_MASTER": [{gl_master_id, GL_DISPOSITION, GL_ATTRIBUTE_NAME}, ...]}
```

### Flow C — Read: per-supplier attribute-history (GST only)

```
Any caller
    │
    ▼
[API — read]  serviceName=USER_ATTR_DISPOSITIONS  {glid, attribute_name, all(0/1)}
    │  AttrDispositionController.go
    │  attribute_name != "GST" → short-circuit "NO_DATA_FOUND"
    ▼
GetAttrDispositions()
    │  SELECT ... FROM GLUSR_DISPOSITIONS, GL_DISPOSITIONS_MASTER
    │  WHERE fk_glusr_usr_id=$1 AND fk_gl_disposition_master_id=gl_disposition_master_id
    │        AND fk_gl_attribute_id=2106 (hardcoded GST)
    │  all=1 → all rows | else → ORDER BY add_date DESC LIMIT 1
    ▼
[DB — mesh_pg_user]
    ▼
Response {glusr_attribute_name, glusr_disposition, glusr_disposition_add_date, modid}
```

**Inferred (not code-confirmed) end-to-end usage pattern**: Flow B (master-list) is most
likely how a caller obtains a valid `MASTER_ID` before calling Flow A (write) — the UI
presumably shows `GL_DISPOSITION`/`GL_ATTRIBUTE_NAME` text and submits the corresponding
`gl_master_id` back as `MASTER_ID`. **No code was found that directly chains these two
calls together** (no shared request-flow/orchestrator) — this linkage is inferred from field
shape, not observed in a single call-site. **[INFERRED — confirm with team]**.

---

## 9. Flow-wise DB & Table Usage

### Flow A — Write

| # | DB (alias) | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | `meshpg` | `GLUSR_DISPOSITIONS` | INSERT | Naya disposition-record supplier ke against likha jaata hai, `FK_GL_DISPOSITION_MASTER_ID` + `MODID` ke saath, history preserve karte hue |

Total round-trips: **1** (single INSERT, no pre-read). Simplest flow in this domain.

### Flow B — Master-list read

| # | DB (alias) | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | `mesh_pg_with_product_user` | `GL_DISPOSITIONS_MASTER` join `GL_ATTRIBUTE` | SELECT | Saari enabled dispositions nikalta hai, unke attribute-display-name ke saath, dropdown-population ke liye |

Total round-trips: **1**.

### Flow C — Attribute-history read

| # | DB (alias) | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | `mesh_pg_user` | `GLUSR_DISPOSITIONS` join `GL_DISPOSITIONS_MASTER` (subquery) + `GL_ATTRIBUTE` (scalar subquery) | SELECT | Supplier ki GST-disposition history (ya latest entry) nikalta hai, attribute-name label ke saath |

Total round-trips: **1** (single query, subqueries are part of the same round-trip).

**Cross-flow observation**: all three flows are single-round-trip — this is the simplest
domain covered so far compared to GST (which needs 9-10 round-trips across 4 DBs for a
write). There is no fan-out, no parallel goroutines, no multi-DB write.

---

## 10. Optimization Scope

### Low impact

1. **Write and attribute-history-read use different DB aliases** (`meshpg` vs
   `mesh_pg_user`), and the master-list read uses yet a **third** alias
   (`mesh_pg_with_product_user`) — three different config aliases touching (at least
   partially) overlapping tables. Same open-question pattern flagged in the GST/Market
   Place URL/Social Contacts KTs: worth confirming whether these are aliases of the same
   physical DB or separate replicas with potential lag. If they are replicas, a disposition
   written via `meshpg` may not be immediately visible via `mesh_pg_user` or
   `mesh_pg_with_product_user` reads — a plausible source of "I just wrote it, why can't I
   read it back" support tickets.
2. **Master-list query has no attribute filter** — it returns every enabled disposition
   across every attribute in one shot. At current expected scale (a handful of attributes,
   presumably a small number of dispositions each) this is not a performance concern, but
   if the disposition master-data grows significantly, a `detail_from`-level attribute
   filter parameter could reduce payload size. Not a current bottleneck — flagged for
   awareness only.
3. **No caching anywhere in this domain** (section 7) — master-list data (Flow B) is a
   strong caching candidate: it is master/reference data that changes rarely (dispositions
   are curated/enabled-flagged, not user-generated), yet it is queried fresh from Postgres
   on every dropdown-population call. A short-TTL cache-aside (Redis, already used
   elsewhere in this codebase per other KT docs) would cut DB load for this specific read
   with low correctness risk. Low priority given the query itself is a simple 1-round-trip
   two-table join, likely already fast.

### Design-scope note (not a performance issue)

4. **Read-side (attribute-history, Flow C) hardcode limits an otherwise-generic schema to
   one attribute** — if more attributes need disposition-history exposed later (and the
   master-list, section 6, already treats dispositions as multi-attribute), the
   attribute-history read-model needs a generalization (attribute_id lookup instead of
   hardcoded `2106`), not a performance fix.
5. **`UpsertDisposition` never actually upserts** — the function name implies
   insert-or-update semantics, but the implementation is INSERT-only (by design, per
   section 4 rule 4, to preserve history). This is not a bug, but the naming is
   misleading for anyone reading the code cold — worth a rename or a comment if this file
   is touched again.

---

## 11. Cron Inventory

**No cron touches `GLUSR_DISPOSITIONS` or `GL_DISPOSITIONS_MASTER`.** Confirmed via grep of
`service-api-go-production/crons/` for `DISPOSITION` (case-insensitive) — the only hit is
`crons/recommend/Supp_verify_log_cron.go`, which uses the unrelated
`IIL_SUPP_VERIFICATION_LOG.DISPOSITION_TYPE` (Supplier Verification Log feature, see naming
caveat at the top of this doc), not `GLUSR_DISPOSITIONS`. This is a genuinely check-don't-assume
finding — the feature is entirely request-driven, no background job exists for it.

---

## 12. Edge Cases & Gotchas (technical POV)

1. **Read (attribute-history) is hardcoded to GST (`gl_attribute_id=2106`)** — any other
   `ATTRIBUTE_NAME` submitted on write will insert fine but can **never be read back**
   through `AttrDisposition` — a real functional gap if other attributes are actually being
   written, and the master-list (Flow B) actively encourages a multi-attribute mental model
   since it returns dispositions for all attributes without restriction.
2. **Write-path doesn't validate that `MASTER_ID` actually belongs to the given
   `ATTRIBUTE_NAME`** — no DB-level cross-check visible in code (section 4, rule 6); a
   caller could submit a `MASTER_ID` that belongs to a completely different attribute than
   the `ATTRIBUTE_NAME` string it also sent (which itself is validated for shape but not
   persisted or cross-checked).
3. **`meshpg` (write) vs `mesh_pg_user` (attribute-history read) vs `mesh_pg_with_product_user`
   (master-list read)** — three distinct config aliases, replica-consistency unconfirmed
   (section 10, point 1).
4. **Route path has a typo** (`/user/dispositon`, missing "i") — live in production across
   at least two cluster registrations; cannot be silently "fixed" without a coordinated
   client-side change, since any existing caller is already using the misspelled path.
5. **Master-list endpoint (`/glmaster/detail`, `detail_from=Disposition`) has no GLID-based
   gateway check** (`gateway="NA"` in `Validapi.go`) — unlike the write endpoint's strict
   allowlist (GLADMIN/SELLERMY/IMOB/MSITE), any caller with a valid `modid` can read the
   full disposition master-list. This is a read-only master-data endpoint (not
   supplier-specific data), so the exposure is limited to "which outcome-codes exist and
   for which attributes" — not itself a data-leak of supplier information, but worth
   knowing the asymmetry exists if this endpoint's scope ever expands.
6. **"Disposition" is an overloaded word in this codebase** (naming caveat, top of doc) —
   `FK_RATING_DISPOSITION_ID` (Supplier Rating) and `DISPOSITION_TYPE` (Supplier
   Verification Log) are unrelated features that share no code path with
   `GLUSR_DISPOSITIONS`; a keyword search for "disposition" alone will surface all three
   and can easily cause cross-feature confusion during KT or debugging.

---

## 13. Open Questions

1. Kya `GLUSR_DISPOSITIONS` mein non-GST attributes ke liye bhi data insert ho raha hai
   jo abhi attribute-history read-side se access nahi ho sakta? (Master-list, section 6,
   suggests the schema expects this to happen, but no direct evidence of *actual* non-GST
   inserts was found in this review — only that the write-path would accept them.)
2. `meshpg`, `mesh_pg_user`, aur `mesh_pg_with_product_user` — teeno same physical DB hain
   ya alag replicas? Config/connection-string source is out of scope for this code-only
   review.
3. `MASTER_ID`↔`ATTRIBUTE_NAME` consistency kaun enforce karta hai — koi client-side
   validation (the presumed Flow B → Flow A chain), ya purely trust-based? No orchestration
   code was found linking the two calls.
4. Kya Flow B (master-list) aur Flow A (write) sach mein ek hi UI-flow ka part hain — yeh
   is review mein sirf field-shape se inferred hai, koi shared caller/orchestrator code nahi
   mila.
5. Route-typo (`/user/dispositon`) — kya yeh intentional/legacy hai ya ek unnoticed bug jo
   ab breaking-change ke darr se fix nahi ho sakta?
6. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai (including `ENABLED = -1` convention, jo assumed hai standard codebase
   pattern se, column-level confirm nahi hua).

---

## See also

- [`Disposition_Business_Doc.md`](./Disposition_Business_Doc.md) — product perspective
- [`../KWPL KT/KWPL_Technical_Doc.md`](../KWPL%20KT/KWPL_Technical_Doc.md) —
  unrelated concept, documented separately (no shared table/FK/controller)
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) —
  the only attribute currently readable through the attribute-history Disposition
  read-endpoint (Flow C)

# Negative Mcat — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`Negative_Mcat_Business_Doc.md`](./Negative_Mcat_Business_Doc.md)
dekho.

**Repos**: `service-api-go-production` (write), `users-api-go-production` (read),
`user-temp-consumers-production` (3-way fan-out consumers).

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write | write | [`UserNegMcatController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserNegMcatController.go), [`UserNegMcatModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserNegMcatModel.go) (`NegMcatUpdateintoDB`) |
| Write — validation | write | `MandatoryFieldsNegMcat` — [`UserUtilsMandatory.go:1390`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go), `ValidationNegMcat` — [`UsersValidationMaps.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go), `UserNegMcatMap` (column-mapping) |
| Read | read | [`NegativeMcatController.go`](../../users-api-go-production/internal/controllers/UsersControllers/NegativeMcatController.go), [`UserNegativeMcatModel.go`](../../users-api-go-production/internal/models/users/UserNegativeMcatModel.go) (`GetNegativeMcat`, `getNegativeMcatforGlusr`, `getNegativeMcatforItems`) |
| Fan-out — search/IMSDB replica | consumers | [`USER_NEG_MCAT_IMSDB.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_NEG_MCAT_IMSDB.go) (`dbActionUserNegMcatImsdb`, writes `searchPg`) |
| Fan-out — alert/blacklist DB | consumers | [`USER_NEG_MCAT_IMBLPG.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_NEG_MCAT_IMBLPG.go) (`dbActionUserNegMcatIMBLPg`, writes `alertPg`) |
| Fan-out — forward to another queue | consumers | [`USER_NEG_MCAT_MLBL.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_NEG_MCAT_MLBL.go) (`dbActionUserNegMcatMLBL`) — **no DB write, re-publishes via `PubAPI` to `ETO_REJECTION_MASTER_QUEUE`** |

---

## 2. Routes

| Method | serviceName | Repo | Controller |
|---|---|---|---|
| POST | `NEGATIVE_MCAT_SERVICE` | write | `UserNegMcatController` |
| — | `NEGATIVE_MCAT` | read | `NegativeMcat` |

---

## 3. Data Model — Table

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `NEGATIVE_MCAT_FOR_PRODUCTS` | meshpg (write) / mesh_pg_user (read) | Supplier/product-level mcat-exclusion records | `GLCAT_MCAT_NEGATIVE_ID` (PK, `RETURNING` on insert), `FK_GLUSR_USR_ID`, `FK_PC_ITEM_ID`, `FK_GLCAT_MCAT_ID`, `EMPID`, `IS_DELETED` (soft-delete flag), `NEGATIVE_MCAT_UPDATED_DATE`, `added_date` — [`UserNegMcatModel.go:108-207`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserNegMcatModel.go) |

Read-side query also JOINs `GLCAT_MCAT` (mcat master table, for `glcat_mcat_name`) —
[`UserNegativeMcatModel.go:142`](../../users-api-go-production/internal/models/users/UserNegativeMcatModel.go).

---

## 4. Business Rules & Validation (code se)

1. **Gateway allowlist is unusually large** — `MY`, `GLADMIN`, `TOLLFREE`, `Email
   Marketing`, `HTVENDOR`, `Weberp`, `LEAP`, `IPC-Admin`, `NSD Sales`, `MAPI`, `ENQUIRY`,
   `FREE-WEBSITE`, `M.INDIAMART.COM`, `TRADE`, `BL`, `TENDER`, `PAYNOW`, `CREDIT
   ALLOCATION`, `OVP Process`, `SAMPARK Process`, `TOLLFREE Process`, `VENDOR CITY Pin
   Correction`, `Notification Server`, `PCAT-ADMIN`, `TOLEXO`, `IMOB`, `Merp` — this
   feature is touched by many internal teams/systems.
   [`UserNegMcatController.go:99`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserNegMcatController.go)
2. **Mandatory fields**: `item_id`, `UPDATED_BY`, `action_flag`, `mcat_id`,
   `GLUSR_USR_ID` — but only strictly enforced when `action_flag` is `I`(nsert) or
   `D`(elete); `item_id` empty-string explicitly checked separately (`itemIdFlag`).
   [`UserUtilsMandatory.go:1390-1419`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
3. **`empid` presence changes `UPDATED_BY` resolution**: if `empid != "-1"`, an
   `Employee_mesh_pg` lookup resolves the human-readable name (jaisa Logo/TrustSeal
   modules mein pehle bhi dekha gaya pattern); agar `empid == "-1"`, `UPDATED_BY` hardcoded
   `"User"` set hota hai.
   [`UserNegMcatController.go:144-156`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserNegMcatController.go)
4. **`action_flag=I` → INSERT**, dynamically-built column-list based on `UserNegMcatMap`
   (whitelist of accepted keys), `IS_DELETED='0'` hardcoded on insert.
5. **`action_flag` else-branch (implicitly D/delete)** → **soft-delete via UPDATE**:
   `SET IS_DELETED = 1, ... WHERE fk_glusr_usr_id=$1 AND fk_pc_item_id=$2 AND
   fk_glcat_mcat_id=$3` — record physically retained, matched by the (user, item, mcat)
   triple, not by `GLCAT_MCAT_NEGATIVE_ID`.
   [`UserNegMcatModel.go:160-206`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserNegMcatModel.go)
6. **`item_id` optional at the mandatory-field-check level but defaults to `-1`** if
   absent — matlab supplier-level (account-wide) negative-mcat records `-1` ke saath
   store hote hain, item-specific records actual `item_id` ke saath.
7. **Read-side dedups nothing** — simply filters `IS_DELETED=0`, joins `GLCAT_MCAT` for
   the human-readable name.
   [`UserNegativeMcatModel.go:142,200`](../../users-api-go-production/internal/models/users/UserNegativeMcatModel.go)

---

## 5. RabbitMQ / Kafka / Redis

| Queue / `SERVICENAME` | Publisher | Consumer(s) | Purpose |
|---|---|---|---|
| `USER_NEGATIVE_MCAT` → routes to `user.neg.*` on `USER.topic` exchange (per domain-wide `serviceToQueueMap`) | `UserNegMcatModel.go` (`NegMcatUpdateintoDB`, `utils.PushToQueue`) — **only fires on INSERT/UPDATE success** | [`USER_NEG_MCAT_IMSDB.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_NEG_MCAT_IMSDB.go) (searchPg replica), [`USER_NEG_MCAT_IMBLPG.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_NEG_MCAT_IMBLPG.go) (alertPg replica), [`USER_NEG_MCAT_MLBL.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_NEG_MCAT_MLBL.go) (forwards to `ETO_REJECTION_MASTER_QUEUE` — no local DB write) | 3-way fan-out: two DB-replica-syncs + one forward-only relay into another pipeline |

**Interesting finding**: `USER_NEG_MCAT_MLBL` consumer doesn't write to any local database
at all — it re-publishes the same message via `utils.PubAPI(...)` into a **different queue**
(`ETO_REJECTION_MASTER_QUEUE`), suggesting the negative-mcat event also feeds into some
separate ETO (likely "Extended/External Trade Operations" or similar)
rejection-processing pipeline outside the scope of these 3 repos.

**Koi Kafka ya Redis usage nahi mila** iss feature mein.

---

## 6. End-to-End Technical Flow

```
Internal-tool / Admin (GLADMIN/WebERP/Tollfree/PCAT-Admin/...)
    │
    ▼
[API — write]  POST serviceName=NEGATIVE_MCAT_SERVICE
                {GLUSR_USR_ID, item_id, mcat_id, action_flag(I/D), empid, VALIDATION_KEY}
    │  UserNegMcatController.go — Gateway check (large allowlist)
    │  empid present → Employee_mesh_pg() name-resolve
    │  MandatoryFieldsNegMcat() → ValidationNegMcat()
    ▼
NegMcatUpdateintoDB()
    ├─ action_flag=I → INSERT INTO NEGATIVE_MCAT_FOR_PRODUCTS (...) RETURNING ID
    └─ action_flag=D → UPDATE ... SET IS_DELETED=1 WHERE (user,item,mcat) match
    ▼
[DB — meshpg]
    │  on success:
    ▼
[RabbitMQ]  SERVICENAME=USER_NEGATIVE_MCAT → user.neg.* (USER.topic)
    ▼
[CONSUME — 3-way parallel fan-out]
    ├─ USER_NEG_MCAT_IMSDB   → writes searchPg (search/directory replica)
    ├─ USER_NEG_MCAT_IMBLPG  → writes alertPg (alert/blacklist replica)
    └─ USER_NEG_MCAT_MLBL    → forwards via PubAPI to ETO_REJECTION_MASTER_QUEUE
                                (no local DB write)

Any caller (glusrid / item_id)
    │
    ▼
[API — read]  serviceName=NEGATIVE_MCAT  {glusrid, item_id (optional, comma-list ok)}
    │  NegativeMcatController.go → GetNegativeMcat()
    ├─ getNegativeMcatforGlusr() — JOIN GLCAT_MCAT, WHERE fk_glusr_usr_id, is_deleted=0
    └─ getNegativeMcatforItems() — same JOIN, WHERE fk_pc_item_id IN (...)
    ▼
Response {Negative_mcats_for_gluser, Negative_mcats_for_items}
```

---

## 7. Optimization Scope — DB Response-Time Contribution

### Low-Medium impact

1. **`Employee_mesh_pg` lookup ek extra sequential DB-round-trip hai** before the main
   insert/update, sirf `empid` present hone par — same pattern already flagged in Logo KT;
   Redis-cache for employee-name-resolution is a repeatable suggestion across modules.
2. **3-way consumer fan-out** — standard replication-cost, no obvious inefficiency; the
   `MLBL` forward-only consumer is architecturally interesting but not a DB-performance
   concern.

### Low impact

3. **Insert/update column-building via `range` over a map + whitelist-lookup
   (`UserNegMcatMap`)** — per-request map-iteration overhead is negligible at this scale,
   no action needed.
4. **Soft-delete matched on (user, item, mcat) triple, not primary key** — if this triple
   isn't indexed together, delete-queries could be slower at scale; worth confirming a
   composite index exists.

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora Negative Mcat flowchart yahan dekho](https://lucid.app/lucidchart/3f3741ae-a07d-47dd-b9f7-59b7d39d7573/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **`item_id` defaults to `-1` when absent** — supplier-level vs item-level records are
   distinguished purely by this sentinel value, not a separate flag column; any downstream
   consumer must know this convention.
2. **Soft-delete's `WHERE` clause matches on (user, item, mcat), not the primary key** —
   if duplicate active rows ever exist for the same triple (no unique-constraint confirmed
   in code), a single delete-call could affect multiple rows unexpectedly.
3. **`USER_NEG_MCAT_MLBL`'s downstream `ETO_REJECTION_MASTER_QUEUE` pipeline is outside
   these 3 repos** — full picture of what consumes that queue is not traceable from this
   codebase alone.

---

## 10. Open Questions

1. `ETO_REJECTION_MASTER_QUEUE` (fed by `USER_NEG_MCAT_MLBL`) — which system consumes it,
   and what "ETO" stands for / what business-process it represents?
2. Composite index on `(fk_glusr_usr_id, fk_pc_item_id, fk_glcat_mcat_id)` exists kya —
   confirm for both correctness (duplicate-prevention) and performance?
3. Read query filters `is_deleted=0`, lekin poora historical soft-deleted data ka
   koi cleanup/archival-job hai kya?
4. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`Negative_Mcat_Business_Doc.md`](./Negative_Mcat_Business_Doc.md) — product perspective
- [`../Logo KT/Logo_Technical_Doc.md`](../Logo%20KT/Logo_Technical_Doc.md) — similar
  `Employee_mesh_pg` name-resolution pattern precedent

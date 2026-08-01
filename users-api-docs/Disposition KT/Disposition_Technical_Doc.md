# Disposition — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`Disposition_Business_Doc.md`](./Disposition_Business_Doc.md)
dekho.

**Scope note**: Disposition aur KWPL genuinely **do alag concepts** hain — alag tables
(`GLUSR_DISPOSITIONS` vs `PL_KWRD`), alag controllers, koi FK-relation ya shared-code nahi
mila. Dekho [`../KWPL KT/KWPL_Technical_Doc.md`](../KWPL%20KT/KWPL_Technical_Doc.md).

**Repos**: `service-api-go-production` (write), `users-api-go-production` (read).
**Koi RabbitMQ/Kafka/consumer nahi mila** — purely synchronous.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write | write | [`UserDispositionController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDispositionController.go), [`UserDispositionModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDispositionModel.go) (`UpsertDisposition`) |
| Validation | write | `MandatoryParamsCheckDisposition` — [`UserUtilsMandatory.go:916`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go), `UserDispositionMap` (`LengthAndTypeValidations_v3`) |
| Read | read | [`AttrDispositionController.go`](../../users-api-go-production/internal/controllers/UsersControllers/AttrDispositionController.go), [`UserAttrDispositionModel.go`](../../users-api-go-production/internal/models/users/UserAttrDispositionModel.go) (`GetAttrDispositions`) |

---

## 2. Routes

| Method | serviceName | Repo | Controller |
|---|---|---|---|
| POST | `USER_ATTR_DISPOSITION` | write | `UserDispositionController` |
| — | `USER_ATTR_DISPOSITIONS` | read | `AttrDisposition` |

---

## 3. Data Model — Tables

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_DISPOSITIONS` | meshpg (write) / mesh_pg_user (read) | Per-supplier disposition/outcome-log, insert-only (history-preserving) | `FK_GLUSR_USR_ID`, `GLUSR_DISPOSITION_ADD_DATE` (`CURRENT_TIMESTAMP` on insert), `FK_GL_DISPOSITION_MASTER_ID`, `GLUSR_DISPOSITION_MODID` — [`UserDispositionModel.go:46-47`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDispositionModel.go) |
| `GL_DISPOSITIONS_MASTER` | mesh_pg_user (read, joined) | Master-list of valid disposition-codes, scoped per attribute | `gl_disposition_master_id`, `gl_disposition` (the actual outcome text/code), `fk_gl_attribute_id` — [`UserAttrDispositionModel.go:57`](../../users-api-go-production/internal/models/users/UserAttrDispositionModel.go) |
| `GL_ATTRIBUTE` | mesh_pg_user (read, joined) | Attribute master (resolves `gl_attribute_id` → human-readable column-name) | `gl_attribute_id`, `gl_attribute_column_name` — read-query hardcodes `gl_attribute_id = 2106` (GST) |

---

## 4. Business Rules & Validation (code se)

1. **Gateway allowlist**: `GLADMIN`, `SELLERMY`, `IMOB`, `MSITE` — same allowlist pattern
   as Social Contacts.
   [`UserDispositionController.go:50`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDispositionController.go)
2. **Mandatory**: `GLID`, `MASTER_ID`, `ATTRIBUTE_NAME`, `DISPOSITION`, `VALIDATION_KEY`.
   [`UserUtilsMandatory.go:916-936`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
3. **Write is always an INSERT, never an update** — every disposition-submission creates
   a new row (`GLUSR_DISPOSITION_ADD_DATE = CURRENT_TIMESTAMP`), preserving full history
   per supplier.
   [`UserDispositionModel.go:46`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDispositionModel.go)
4. **`ATTRIBUTE_NAME` and `DISPOSITION` are accepted/validated on write, but not stored
   directly** — only `MASTER_ID` (the disposition-master's ID) and `MODID` are persisted;
   the write-path trusts the caller to have already resolved `DISPOSITION`/
   `ATTRIBUTE_NAME` to the correct `MASTER_ID`.
5. **Read-side is hardcoded to `gl_attribute_id = 2106` (GST)** — `GetAttrDispositions`
   explicitly checks `if attributeName != "GST" { return NO_DATA_FOUND }` before even
   querying — despite the schema (`GL_DISPOSITIONS_MASTER`/`GL_ATTRIBUTE` joined by
   `fk_gl_attribute_id`) being generically designed for multiple attributes.
   [`UserAttrDispositionModel.go:25-33,57`](../../users-api-go-production/internal/models/users/UserAttrDispositionModel.go)
6. **`all` param controls history-depth**: `all=1` → full history (all rows); default
   (`all` absent/`0`) → only the latest disposition (`ORDER BY ... DESC LIMIT 1`).
   [`UserAttrDispositionModel.go:57-60`](../../users-api-go-production/internal/models/users/UserAttrDispositionModel.go)

---

## 5. RabbitMQ / Kafka / Redis

**Koi RabbitMQ, Kafka, ya Redis usage nahi mila** — purely synchronous insert/read.

---

## 6. End-to-End Technical Flow

### Write
```
Internal-tool (GLADMIN / SELLERMY / IMOB / MSITE)
    │
    ▼
[API — write]  POST serviceName=USER_ATTR_DISPOSITION
                {GLID, MASTER_ID, ATTRIBUTE_NAME, DISPOSITION, MODID, VALIDATION_KEY}
    │  UserDispositionController.go — Gateway check
    │  MandatoryParamsCheckDisposition() → LengthAndTypeValidations_v3()
    ▼
UpsertDisposition()
    │  INSERT INTO GLUSR_DISPOSITIONS (FK_GLUSR_USR_ID, ADD_DATE=NOW(),
    │                                   FK_GL_DISPOSITION_MASTER_ID, MODID)
    ▼
[DB — meshpg]
    ▼
Response {"data inserted successfully" / "SUCCESS IN PG"}
```

### Read
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

---

## 7. Optimization Scope — DB Response-Time Contribution

### Low impact

1. **Write and read use different DB aliases** (`meshpg` vs `mesh_pg_user`) — same
   pattern/risk already flagged in Market Place URL KT; worth confirming these are
   aliases of the same physical DB, not separate replicas with lag.
2. **Read query joins two small master tables per-request** — low-cost given the
   attribute-filter narrows to a single hardcoded ID; no optimization concern at
   current scope.

### Design-scope note (not a performance issue)

3. **Read-side hardcode limits an otherwise-generic schema to one attribute** — if more
   attributes need disposition-history exposed later, the read-model needs a
   generalization (attribute_id lookup instead of hardcoded `2106`), not a performance
   fix.

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora Disposition flowchart yahan dekho](https://lucid.app/lucidchart/a3259975-e75c-4bc8-b7b4-1be111f541e6/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **Read is hardcoded to GST (`gl_attribute_id=2106`)** — any other `ATTRIBUTE_NAME`
   submitted on write will insert fine but can **never be read back** through this
   endpoint — a real functional gap if other attributes are actually being written.
2. **Write-path doesn't validate that `MASTER_ID` actually belongs to the given
   `ATTRIBUTE_NAME`** — caller-trusted linkage, no DB-level cross-check visible in code.
3. **`meshpg` (write) vs `mesh_pg_user` (read)** — same open-question pattern as Market
   Place URL and Social Contacts KTs.

---

## 10. Open Questions

1. Kya `GLUSR_DISPOSITIONS` mein non-GST attributes ke liye bhi data insert ho raha hai
   jo abhi read-side se access nahi ho sakta?
2. `meshpg` aur `mesh_pg_user` same physical DB hain ya replicas?
3. `MASTER_ID`↔`ATTRIBUTE_NAME` consistency kaun enforce karta hai — koi client-side
   validation, ya trust-based?
4. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`Disposition_Business_Doc.md`](./Disposition_Business_Doc.md) — product perspective
- [`../KWPL KT/KWPL_Technical_Doc.md`](../KWPL%20KT/KWPL_Technical_Doc.md) —
  unrelated concept, documented separately (no shared table/FK/controller)
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) —
  the only attribute currently readable through this Disposition read-endpoint

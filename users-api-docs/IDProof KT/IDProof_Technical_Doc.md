# ID Proof — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`IDProof_Business_Doc.md`](./IDProof_Business_Doc.md)
dekho.

**Repos**: `service-api-go-production` (write), `users-api-go-production` (read).
**Koi RabbitMQ/Kafka/consumer nahi mila** — purely synchronous, jaisa Blocking/Social-Review/
Popup Details modules.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write | write | [`IdProofController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/IdProofController.go), [`IdProofModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/IdProofModel.go) (`IdProofModel`, `CheckIdproofAction`) |
| Write — validation | write | `ValidationIdProof` — [`UsersValidationMaps.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |
| Read | read | [`IdProofController.go`](../../users-api-go-production/internal/controllers/UsersControllers/IdProofController.go) (`ActionIdProof`), [`UsersIdProofModel.go`](../../users-api-go-production/internal/models/users/UsersIdProofModel.go) (`GetIdProofModel`) |

---

## 2. Routes

| Method | serviceName | Repo | Controller |
|---|---|---|---|
| POST | `IDPROOF` | write | `IdProofController` |
| — | `IDPROOF` | read | `ActionIdProof` |

---

## 3. Data Model — Table

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_USR_IDENTITY_PROOFS` | write: meshpg / read: mesh_pg_user | Per-user, per-proof-type identity-verification record | `FK_GLUSR_USR_ID`, `FK_IIL_IDENTITY_PROOF_ID` (proof-type, composite key with user-ID), `IDENTITY_PROOF_NUMBER`, `IDENTITY_PROOF_REF_URL`, `APPROVAL_STATUS` (default `0`), `ADDED_DATE`, `MODIFIED_DATE`, `UPDATEDBY_ID`/`_SCREEN`/`_URL`/`_IP`/`_IP_COUNTRY`, `HIST_COMMENTS`, `UPDATED_BY`, `UPDATEDUSING` — [`IdProofModel.go:61,86-101`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/IdProofModel.go) |

**Note**: read aur write alag DB-alias use karte hain (`meshpg` vs `mesh_pg_user`) — same
pattern already flagged as an Open Question in Market Place URL, Social Contacts, aur
Disposition KTs.

---

## 4. Business Rules & Validation (code se)

1. **Gateway allowlist**: `GLADMIN`, `MAPI`, `ANDROID`, `IOS`.
   [`IdProofController.go:87`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/IdProofController.go)
2. **Mandatory fields**: `FK_GLUSR_USR_ID`, `FK_IIL_IDENTITY_PROOF_ID`,
   `IDENTITY_PROOF_REF_URL`, `UPDATED_BY`, `UPDATEDUSING`.
   [`IdProofController.go:84`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/IdProofController.go)
3. **Insert-vs-update decided by a pre-check `SELECT`**, not `ON CONFLICT`:
   `CheckIdproofAction()` first queries `WHERE FK_GLUSR_USR_ID=$1 AND
   FK_IIL_IDENTITY_PROOF_ID=$2`; if a row exists → `"UPDATE"`, else → `"INSERT"` — a
   **select-then-insert/update pattern**, unlike most other recent KTs (Flips, Popup
   Details, Social-Contact-async) which use a single `ON CONFLICT` upsert.
   [`IdProofModel.go:127-184`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/IdProofModel.go)
4. **On UPDATE, unset fields fall back to the previously-fetched `old` values** (from the
   pre-check select) — genuine partial-update semantics: `IDENTITY_PROOF_NUMBER`,
   `IDENTITY_PROOF_REF_URL`, `APPROVAL_STATUS`, `FK_IIL_IDENTITY_PROOF_ID` all
   individually fall back to `old[...]` if not present in the request.
   [`IdProofModel.go:65-85`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/IdProofModel.go)
5. **On INSERT, `APPROVAL_STATUS` defaults to `0`** if not explicitly provided.
   [`IdProofModel.go:30-34`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/IdProofModel.go)
6. **`(FK_GLUSR_USR_ID, FK_IIL_IDENTITY_PROOF_ID)` is the effective composite-key** for
   both the pre-check select and the update's `WHERE` clause — a user can have multiple
   proof-type records, one per `FK_IIL_IDENTITY_PROOF_ID`.

---

## 5. RabbitMQ / Kafka / Redis

**Koi RabbitMQ, Kafka, ya Redis usage nahi mila.**

---

## 6. End-to-End Technical Flow

### Write
```
Supplier (Android/iOS) / GLADMIN / MAPI
    │
    ▼
[API — write]  POST serviceName=IDPROOF
                {FK_GLUSR_USR_ID, FK_IIL_IDENTITY_PROOF_ID, IDENTITY_PROOF_NUMBER,
                 IDENTITY_PROOF_REF_URL, APPROVAL_STATUS?, UPDATED_BY, UPDATEDUSING}
    │  IdProofController.go — mandatory-field check → Gateway (GLADMIN/MAPI/ANDROID/IOS)
    │  ValidationIdProof()
    ▼
CheckIdproofAction()
    │  SELECT ... WHERE (FK_GLUSR_USR_ID, FK_IIL_IDENTITY_PROOF_ID)
    ├─ row found     → action="UPDATE", old-values captured
    └─ no row found  → action="INSERT"
    ▼
IdProofModel()
    ├─ INSERT INTO GLUSR_USR_IDENTITY_PROOFS (...) — APPROVAL_STATUS defaults 0
    └─ UPDATE ... SET ... WHERE (FK_GLUSR_USR_ID, FK_IIL_IDENTITY_PROOF_ID)
                — unset fields fall back to `old` values (partial-update)
    ▼
[DB — meshpg]
    ▼
Response {"UPDATE SUCCESS IN PG"}
```

### Read
```
Buyer / any caller
    │
    ▼
[API — read]  serviceName=IDPROOF  {glusrid, proof_type_id}
    │  IdProofController.go (read) — ActionIdProof() — token/modid validity checks
    ▼
GetIdProofModel()
    │  SELECT fk_glusr_usr_id, fk_iil_identity_proof_id, identity_proof_number,
    │         identity_proof_ref_url, approval_status
    │  FROM glusr_usr_identity_proofs WHERE FK_GLUSR_USR_ID=$1 AND
    │       FK_IIL_IDENTITY_PROOF_ID=$2
    ▼
[DB — mesh_pg_user]
    ▼
Response {Data: {...single row...}}
```

---

## 7. Optimization Scope — DB Response-Time Contribution

### Medium impact

1. **Select-then-insert/update is 2 sequential round-trips per write-request** — this is
   the exact pattern already flagged as an optimization-opportunity across GST, Rating,
   and other KTs: converting this to a single `INSERT ... ON CONFLICT
   (FK_GLUSR_USR_ID, FK_IIL_IDENTITY_PROOF_ID) DO UPDATE SET ... COALESCE(...)`
   would reduce this to 1 round-trip, though it would require restructuring the
   partial-update-fallback logic (currently done in Go using the pre-fetched `old`
   values, would need to move to SQL `COALESCE`).

### Low impact

2. **Read query is a simple 2-column-composite-key lookup** — low-cost, no concern at
   current scope.

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora ID Proof flowchart yahan dekho](https://lucid.app/lucidchart/87c000a2-05a5-4687-b048-4e6c7d0963d7/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **Race-condition risk in select-then-insert pattern**: agar do concurrent requests
   same (user, proof-type) ke liye ek saath aayein, dono ko `CheckIdproofAction` se
   "no row found" mil sakta hai, dono `INSERT` try kar sakte hain — agar composite-unique-
   constraint DB-level pe nahi hai, duplicate rows ban sakte hain (same risk-pattern
   already flagged in Image KT).
2. **Read `mesh_pg_user` vs write `meshpg`** — same open-question pattern as other
   recent KTs (Market Place URL, Social Contacts, Disposition).

---

## 10. Open Questions

1. `(FK_GLUSR_USR_ID, FK_IIL_IDENTITY_PROOF_ID)` pe DB-level unique-constraint hai kya?
2. `meshpg` aur `mesh_pg_user` same physical DB hain ya replicas?
3. `FK_IIL_IDENTITY_PROOF_ID` (proof-type master) — koi master-table/enum reference
   iss pass mein nahi mila, valid proof-type-values ki poori list confirm nahi hui.
4. Approval-status transitions (0 → approved/rejected) kaun trigger karta hai — koi
   dedicated admin-review-flow ya cron iss pass mein nahi mila.
5. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`IDProof_Business_Doc.md`](./IDProof_Business_Doc.md) — product perspective
- [`../Trust Verification KT/Trust_Verification_Technical_Doc.md`](../Trust%20Verification%20KT/Trust_Verification_Technical_Doc.md) —
  related but separate verification-workflow system

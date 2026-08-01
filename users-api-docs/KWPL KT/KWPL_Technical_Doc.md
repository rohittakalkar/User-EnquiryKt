# KWPL (Product-Listing Keyword) — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`KWPL_Business_Doc.md`](./KWPL_Business_Doc.md) dekho.

**Scope note**: KWPL aur Disposition genuinely **do alag concepts** hain — alag tables
(`PL_KWRD` vs `GLUSR_DISPOSITIONS`), alag controllers, koi FK-relation ya shared-code nahi
mila. Dekho [`../Disposition KT/Disposition_Technical_Doc.md`](../Disposition%20KT/Disposition_Technical_Doc.md).

**Repos**: `service-api-go-production` (write only). **Koi dedicated read-controller iss
pass mein nahi mila** — na read-repo mein, na koi RabbitMQ/Kafka/consumer. Purely a
write/sync internal-tool endpoint.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write | write | [`UserKwplController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserKwplController.go), [`UserKwplModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserKwplModel.go) (`UpsertKwpl`) |
| Validation | write | `MandatoryParamsCheckKwpl` — [`UserUtilsMandatory.go:11`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go), `UserkwplMap` (`LengthAndTypeValidations_v3`) |

---

## 2. Routes

| Method | serviceName | Repo | Controller |
|---|---|---|---|
| POST | `KWPL_SERVICE` | write | `UserKwplController` |

---

## 3. Data Model — Table

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `PL_KWRD` | meshpg | Product-listing keyword-serve record | `pl_kwrd_id` (PK, `RETURNING` on insert), `FK_GL_CITY_ID`, `PL_KWRD_MODID`, `FK_GLUSR_USR_ID`, `FK_COMPANY_ID`, `PL_KWRD_TERM_UPPER` (the keyword itself), `PL_KWRD_DESC`, `PL_KWRD_FROM`/`PL_KWRD_TO` (validity window), `PL_KWRD_ENABLE` (`-1` = active/enabled sentinel), `PL_KWRD_COMMENT`, `PL_KWRD_WO_ID` (work-order ID), `PL_KWRD_UPDATEDBY_ID`/`_UPDATEDBY`/`_UPDATEDBY_AGENCY`, `PL_KWRD_UPDATESCREEN`, `PL_KWRD_IP`/`_IP_COUNTRY`, `PL_KWRD_UPDATEDUSING`, `PL_KWRD_HIST_COMMENTS`, `PL_KWRD_UPDATEDBY_URL`, `FK_CUST_TO_SERV_ID` (customer-to-service/serve ID) — [`UserKwplModel.go:152-231`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserKwplModel.go) |

---

## 4. Business Rules & Validation (code se)

1. **Gateway allowlist**: `GLADMIN`, `WEBERP` only — purely internal-tool feature.
   [`UserKwplController.go:57`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserKwplController.go)
2. **`ACTION` param drives a 4-way branch inside `UpsertKwpl`**:
   - `INSERT` → new row, all fields.
   - `DISABLE` → `UPDATE ... SET PL_KWRD_ENABLE=$1, ... WHERE FK_GLUSR_USR_ID=$11 AND
     FK_CUST_TO_SERV_ID=$12 AND PL_KWRD_ENABLE=-1` — only affects currently-enabled rows.
   - `UPDATE` → same active-row-guard (`PL_KWRD_ENABLE=-1`), updates `PL_KWRD_TO`
     (validity-end) and `PL_KWRD_WO_ID`.
   - `SEARCH_UPDATE` → two sub-variants: if `ID` present, matches by `PL_KWRD_ID`
     directly; else matches by `(FK_GLUSR_USR_ID, FK_CUST_TO_SERV_ID)` — both still
     guarded by `PL_KWRD_ENABLE=-1`. Only updates the keyword-term itself
     (`PL_KWRD_TERM_UPPER`).
   [`UserKwplModel.go:151-234`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserKwplModel.go)
3. **`PL_KWRD_ENABLE=-1` is used as the "active" sentinel** — not `1`/`true`, an
   inherited-legacy-convention worth noting for anyone querying this table directly.
4. **Update-path response is always `"INSERT SUCCESS"` regardless of actual action** —
   `UpsertKwpl`'s success-output string is hardcoded the same for INSERT/DISABLE/UPDATE/
   SEARCH_UPDATE (controller-level success-check also only looks for `"INSERT SUCCESS"`)
   — naming is misleading but functionally consistent.
   [`UserKwplModel.go:252-263`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserKwplModel.go)

---

## 5. RabbitMQ / Kafka / Redis

**Koi RabbitMQ, Kafka, ya Redis usage nahi mila** — purely synchronous single-table write,
no downstream fan-out found.

---

## 6. End-to-End Technical Flow

```
Internal-tool (GLADMIN / WebERP)
    │
    ▼
[API — write]  POST serviceName=KWPL_SERVICE
                {ACTION(INSERT/DISABLE/UPDATE/SEARCH_UPDATE), GLUSRID, CITY_ID, COMPID,
                 PL_KEYWORDS, DESC, FROM_DATE1, TO_DATE1, ENABLE, WO_ID, SERVE_ID, ...}
    │  UserKwplController.go — Gateway (GLADMIN/WEBERP)
    │  MandatoryParamsCheckKwpl() → LengthAndTypeValidations_v3()
    ▼
UpsertKwpl()
    ├─ ACTION=INSERT       → INSERT INTO PL_KWRD (...) RETURNING pl_kwrd_id
    ├─ ACTION=DISABLE      → UPDATE ... WHERE (user, serve_id) AND ENABLE=-1
    ├─ ACTION=UPDATE       → UPDATE (validity/work-order) WHERE (user, serve_id) AND ENABLE=-1
    └─ ACTION=SEARCH_UPDATE → UPDATE (keyword-term only) WHERE (ID) or (user, serve_id) AND ENABLE=-1
    ▼
[DB — meshpg]  PL_KWRD
    ▼
Response {"INSERT SUCCESS" for all successful actions}
```

---

## 7. Optimization Scope — DB Response-Time Contribution

### Low impact

1. **Single-query-per-request, no sequential-round-trip issues** — straightforward,
   low-risk.
2. **All active-row-matching queries filter on `(FK_GLUSR_USR_ID, FK_CUST_TO_SERV_ID,
   PL_KWRD_ENABLE=-1)`** — confirm a composite index exists on this triple for update/
   disable performance at scale.

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora KWPL flowchart yahan dekho](https://lucid.app/lucidchart/3aaea5b9-2714-41f9-8a75-a4c1aa7c62e9/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **`PL_KWRD_ENABLE=-1` as the active-sentinel is non-obvious** — any new code touching
   this table must know this convention, not `1`/`true`.
2. **Success-output string always says `"INSERT SUCCESS"` even for DISABLE/UPDATE/
   SEARCH_UPDATE actions** — misleading if read literally in logs/Kibana without context.
3. **`SEARCH_UPDATE` without `ID` falls back to matching by `(user, serve_id)`** — if
   multiple active rows exist for that pair (no uniqueness enforced in code), all
   matching rows would be updated in one call.
4. **Koi read-endpoint nahi mila** — is data ko kaise consume/view kiya jaata hai (koi
   admin-dashboard, ya seedha-DB-query) confirm nahi hua is pass mein.

---

## 10. Open Questions

1. Is table ka read-side kahan hai — koi dedicated endpoint ya admin-tool jo yeh data
   dikhata hai?
2. `FK_CUST_TO_SERV_ID`/`SERVE_ID`/`WO_ID` (work-order) kis upstream-system se aate hain —
   koi lead-gen/campaign-management system ka reference hai kya?
3. `(FK_GLUSR_USR_ID, FK_CUST_TO_SERV_ID, PL_KWRD_ENABLE)` pe composite-index confirm
   karna baaki hai.
4. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`KWPL_Business_Doc.md`](./KWPL_Business_Doc.md) — product perspective
- [`../Disposition KT/Disposition_Technical_Doc.md`](../Disposition%20KT/Disposition_Technical_Doc.md) —
  unrelated concept, documented separately (no shared table/FK/controller)

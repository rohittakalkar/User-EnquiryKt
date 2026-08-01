# Fact Sheet — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`Fact_Sheet_Business_Doc.md`](./Fact_Sheet_Business_Doc.md)
dekho.

**Scope note**: Fact Sheet is NOT a standalone controller — it is one `detailType`
branch of the same **shared multi-purpose user-details write/read endpoint** that also
handles GST (`CompRgst`), Bank Details, and Form-8. Dekho
[`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) for the shared
controller's full architecture (gateway, dispatch-pattern, RabbitMQ-fan-out) — this doc
focuses only on the Fact-Sheet-specific branch/table.

**Repos**: `service-api-go-production` (write, shared `UserDetailsController`/
`UserDetailsModel`), `users-api-go-production` (read, shared `OtherDetail`-style
`detailType` dispatch).

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write (shared controller) | write | [`UserDetailsController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDetailsController.go), [`UserDetailsModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go) — `Type=="FactSheet"` branch at line 750 |
| Read (shared controller) | read | [`UserOtherDetailModel.go`](../../users-api-go-production/internal/models/users/UserOtherDetailModel.go) — `detailType=="FactSheet"` branch at line 137/228 |

---

## 2. Routes

| Method | serviceName | Repo | Controller |
|---|---|---|---|
| POST | (shared user-details write service, `TYPE=FactSheet`) | write | `UserDetailsController` |
| — | (shared other-detail read service, `detailType=FactSheet`) | read | `OtherDetail`-family controller |

---

## 3. Data Model — Table

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_USR_FACT_SHEET` | meshpg (write) / mesh_pg_user-family (read) | Supplier business-capability profile, one row per user | `FACT_SHEET_ID` (PK, `RETURNING` on insert), `FK_GLUSR_USR_ID`, `EXPORT_PERCENTAGE`, `DUN_BRANDSTREET_NUMBER`, `COFACE_NUMBER`, `AFTER_SALE_SUPPORT`, `SAMPLING_POLICY`, `SAMPLING_POLICY_PAID_VALUES`, `COMPETITIVE_ADVANTAGE`, `CONTRACT_MANUFACTURING`, `QUALITY_FACILITIES`, `CONSIGNMENTS_SPECIFICATION`, `TIME_OF_DELIVERY`, `PAYMENT_TERMS`, `OTHER_PAYMENT_TERMS`, `PAYMENT_MODE`, `SHIPING_MODE` (schema-typo, missing `P`), `NUMBER_OF_CONTROL_STAFF`, `NUMBER_OF_ENGG`, `NUMBER_OF_SKILLED_STAFF`, `NUMBER_OF_SEMI_SKILLED_STAFF`, `NUMBER_OF_CONSULTANTS`, `GLUSR_USR_FACT_UPDATEDBY_FLAG`, `FACT_SHEET_UPDATEDBY`/`_UPDATEDBY_ID`/`_UPDATEDBY_AGENCY`/`_UPDATESCREEN`/`_IP`/`_IP_COUNTRY`/`_UPDATEDBY_URL`/`_HIST_COMMENTS` — [`UserDetailsModel.go:750-829`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go) |

---

## 4. Business Rules & Validation (code se)

1. **Insert-vs-update decided by `id` (an internal record-lookup, not directly
   `FACT_SHEET_ID` from the caller) being empty or not** — same pattern as the `CompRgst`
   branch documented in GST KT: `id != "" → UPDATE`, `id == "" → INSERT`.
   [`UserDetailsModel.go:752,787`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)
2. **UPDATE is a full-record overwrite** — all 29 columns are set unconditionally in
   the `UPDATE` statement; there's no dynamic/whitelist-driven partial-update logic here
   (unlike, say, ID Proof or Negative Mcat) — every field must be supplied on update or
   it gets overwritten with the caller's (possibly empty) value.
   [`UserDetailsModel.go:754-786`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)
3. **`GLUSR_USR_FACT_SHEET` is part of the shared RabbitMQ-fan-out table-list**
   (`tabArr = ["GLUSR_USR_COMP_REGISTRATIONS", "GLUSR_BANK_DETAILS",
   "GLUSR_USR_COMP_FORM8", "GLUSR_USR_FACT_SHEET"]`) — meaning a successful Fact-Sheet
   write triggers the same RabbitMQ-message-construction path as GST/Bank-Details/
   Form-8 updates, with `ACTION=UPDATE` always set on the outgoing message regardless of
   whether it was actually an insert or update on this side.
   [`UserDetailsModel.go:4169-4184`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)
4. **Read-side query selects the single row directly** (`WHERE fk_glusr_usr_id=$1 LIMIT
   1`) — no windowing/history-dedup needed since only one row exists per user.
   [`UserOtherDetailModel.go:228-266`](../../users-api-go-production/internal/models/users/UserOtherDetailModel.go)
5. **Read-side is also exposed via the combined `json_agg` "everything" query** used for
   full-profile fetches (`fact_sheet_data` sub-select alongside `comp_reg_data`,
   `oth_rem_dtl_data`) — same embedded-JSON-subquery pattern seen in GST/Social-Contacts
   KTs.
   [`UserOtherDetailModel.go:506-511`](../../users-api-go-production/internal/models/users/UserOtherDetailModel.go)

---

## 5. RabbitMQ / Kafka / Redis

Shares the same RabbitMQ-fan-out mechanism as GST/Bank-Details/Form-8 within the shared
`UserDetailsModel.go` controller — dekho
[`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) §5 for the full
queue/consumer breakdown, which was documented there for the same shared write-path.

**Koi Kafka ya dedicated-Redis usage nahi mila** specific to Fact Sheet.

---

## 6. End-to-End Technical Flow

```
Supplier
    │
    ▼
[API — write]  POST (shared user-details service)  TYPE=FactSheet
                {export_percentage, dun_brandstreet_number, coface_number,
                 after_sale_support, sampling_policy, ..., number_of_consultants}
    │  UserDetailsController.go (shared) — Gateway check (same allowlist as GST branch)
    ▼
UserDetailsModel.go — Type=="FactSheet" branch
    │  id present? →
    ├─ yes → UPDATE GLUSR_USR_FACT_SHEET SET (all 29 columns) WHERE FK_GLUSR_USR_ID=$1
    └─ no  → INSERT INTO GLUSR_USR_FACT_SHEET (...) RETURNING FACT_SHEET_ID
    ▼
[DB — meshpg]
    │  on success (table in shared fan-out list):
    ▼
[RabbitMQ]  shared USER_DETAIL_* service (same as GST branch)
    ▼
Response

Buyer / any caller
    │
    ▼
[API — read]  shared other-detail service  detailType=FactSheet
    │  SELECT * FROM GLUSR_USR_FACT_SHEET WHERE fk_glusr_usr_id=$1 LIMIT 1
    │  (or embedded as fact_sheet_data inside the combined full-profile json_agg query)
    ▼
[DB — mesh_pg_user family]
    ▼
Response — fact-sheet fields
```

---

## 7. Optimization Scope — DB Response-Time Contribution

### Low impact

1. **Full-record-overwrite UPDATE (29 columns, no dynamic whitelist)** — simpler and
   more predictable than dynamic-column-building approaches elsewhere in this codebase,
   at the cost of requiring the caller to always resend the complete record; no
   DB-performance concern either way at this table's likely scale.
2. **Single-row lookup (`LIMIT 1` on a unique-per-user table)** — no optimization
   opportunity beyond what's already appropriate for this shape of data.

### Shared with GST KT

3. Any optimization findings for the shared `UserDetailsModel.go` controller's overall
   architecture (sequential DB-calls, RabbitMQ-fan-out cost, etc.) are already
   documented in [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) §7
   and apply equally to the Fact-Sheet branch.

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora Fact Sheet flowchart yahan dekho](https://lucid.app/lucidchart/7aa6aa09-fee2-4497-b8ec-ea878faa9776/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **Full-overwrite-on-update means partial-updates aren't supported** — a caller that
   only wants to change `EXPORT_PERCENTAGE` must resend the entire fact-sheet payload,
   or risk nulling out the other 28 fields.
2. **`SHIPING_MODE` column-name has a schema-level typo** (missing a `P` — should be
   `SHIPPING_MODE`) — cosmetic, but any future direct-SQL work on this table needs to
   match the existing (mis-)spelling.
3. **RabbitMQ message always says `ACTION=UPDATE`** even for a fresh insert (same
   quirk noted in GST KT for this shared controller) — downstream consumers relying on
   this field to distinguish insert-vs-update would need another signal.

---

## 10. Open Questions

1. Is there any validation ensuring numeric-looking fields (`EXPORT_PERCENTAGE`,
   staff-counts) are actually numeric — not confirmed in this pass (validation-map
   details not re-derived here, assume same shared-validation-map pattern as GST).
2. `DUN_BRANDSTREET_NUMBER`/`COFACE_NUMBER` — are these validated against the actual
   Dun & Bradstreet/Coface external registries, or just free-text storage?
3. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`Fact_Sheet_Business_Doc.md`](./Fact_Sheet_Business_Doc.md) — product perspective
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — the shared
  write/read controller this feature is a branch of; full gateway/RabbitMQ/dispatch
  details documented there

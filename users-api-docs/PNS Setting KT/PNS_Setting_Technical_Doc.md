# PNS Setting — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`PNS_Setting_Business_Doc.md`](./PNS_Setting_Business_Doc.md)
dekho.

**Scope note**: PNS Setting aur PNS genuinely **do alag concepts** hain — alag tables
(`IIL_PNS_SETTING` vs `GL_GSM_MASTER`), alag controllers, koi FK-relation ya
shared-code nahi mila. Dekho
[`../PNS KT/PNS_Technical_Doc.md`](../PNS%20KT/PNS_Technical_Doc.md).

**Repos**: `service-api-go-production` (write), `users-api-go-production` (read).
**Koi RabbitMQ/Kafka/consumer nahi mila.**

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write | write | [`UserPnsSettingController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserPnsSettingController.go) (`UserPnsSettingController`), [`UserPnsSettingModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserPnsSettingModel.go) (`UpsertPnsSetting`, `InsertPnsSetting`, `UpdatePnsSetting`, `pnsDelHistoryPG`) |
| Validation | write | `MandatoryParamscheckPnsSetting` — [`UserUtilsMandatory.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go), `UserPnsSettingMap` — [`UsersValidationMaps.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |
| Read | read | [`PnsSettingControllers.go`](../../users-api-go-production/internal/controllers/UsersControllers/PnsSettingControllers.go) (`ActionPnsSetting`), [`UserPnsSettingModel.go`](../../users-api-go-production/internal/models/users/UserPnsSettingModel.go) |

---

## 2. Routes

| Method | serviceName | Repo | Controller |
|---|---|---|---|
| POST | `PNS_SETTING_SERVICE` | write | `UserPnsSettingController` |
| — | `PNS_SETTING` | read | `ActionPnsSetting` |

---

## 3. Data Model — Table

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `IIL_PNS_SETTING` | meshpg | Per-user (and optionally per-additional-contact) PNS off-hours-call-routing preference | `FK_GLUSR_USR_ID`, `FK_GLUSR_USR_ADDT_CONTACT_ID` (nullable — links to a specific additional-contact record), `IIL_PNS_SETTING_TYPE` (`M1`/`M2`/`L1`/`L2`/`T`/`M`/`L` — type-codes, meaning not fully decoded), `IIL_PNS_SETTING_OFFHRS_FLAG`, `IIL_PNS_SETTING_CONTACT_NUMBER`, `IIL_PNS_SETTING_LAST_UPDATE`, `IIL_PNS_SETTING_UPDATED_ID`/`_NAME`/`_SCREEN` — [`UserPnsSettingModel.go:272,329,335`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserPnsSettingModel.go) |

---

## 4. Business Rules & Validation (code se)

1. **Gateway allowlist**: `GLADMIN`, `MAPI`, `BUYERMY`, `SELLERMY`, `ANDROID`, `IOS`.
   [`UserPnsSettingController.go:59`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserPnsSettingController.go)
2. **Insert uses `ON CONFLICT DO NOTHING`** — idempotent, duplicate-submissions are
   silently no-op'd rather than erroring.
   [`UserPnsSettingModel.go:272`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserPnsSettingModel.go)
3. **`IIL_PNS_SETTING_TYPE` drives two branch-families for both insert and update**:
   - Types `M1`/`M2`/`L1`/`L2` — treated as **not requiring** `contact_id` (general,
     per-user-only setting); the insert-path's mandatory-check explicitly exempts these
     4 types from needing `contact_id`.
   - Types `T`/`M`/`L` (and any type not in the first group, for insert) — **require**
     `contact_id`; matched by `(FK_GLUSR_USR_ID, FK_GLUSR_USR_ADDT_CONTACT_ID,
     IIL_PNS_SETTING_TYPE)` on update.
   [`UserPnsSettingModel.go:269,328-336`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserPnsSettingModel.go)
4. **Update's `M1`/`M2`/`L1`/`L2` variant matches only on `(FK_GLUSR_USR_ID,
   IIL_PNS_SETTING_TYPE)`** — no `contact_id` in the `WHERE` clause at all for these
   types, consistent with them being "general" settings.
5. **A "max 5 contact-mappings per user" limit and an "internal correction" cleanup-step
   exist in the code but are COMMENTED OUT** (`noOfPnsMapping`, `checkPNSDataCorrection`
   calls are all inside `//`-commented blocks) — this business-rule was apparently
   disabled at some point but the dead-code remains.
   [`UserPnsSettingModel.go:260-293`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserPnsSettingModel.go)
6. **A `pnsDelHistoryPG` function exists, called after certain operations** — suggests
   a delete/history-audit-trail companion mechanism (name only partially traced in this
   pass, called from a `delete_by_ID_PG`-style function at line 245).

---

## 5. RabbitMQ / Kafka / Redis

**Koi RabbitMQ, Kafka, ya Redis usage nahi mila** — purely synchronous.

---

## 6. End-to-End Technical Flow

```
Supplier (GLADMIN/MAPI/BUYERMY/SELLERMY/Android/iOS)
    │
    ▼
[API — write]  POST serviceName=PNS_SETTING_SERVICE
                {glusr_id, action_flag(i/u), contact_id?, IIL_PNS_SETTING_TYPE,
                 offhrs_flag, contact_number, VALIDATION_KEY}
    │  UserPnsSettingController.go — Gateway check
    │  MandatoryParamscheckPnsSetting() → LengthAndTypeValidations_v3()
    ▼
UpsertPnsSetting()
    ├─ action=i → InsertPnsSetting()
    │     type in {M1,M2,L1,L2} → contact_id optional
    │     else                  → contact_id required
    │     INSERT INTO IIL_PNS_SETTING (...) ON CONFLICT DO NOTHING
    └─ action=u → UpdatePnsSetting()
          type in {M1,M2,L1,L2} → UPDATE WHERE (user, type)
          type in {T,M,L}       → UPDATE WHERE (user, contact_id, type)
    ▼
[DB — meshpg]  IIL_PNS_SETTING
    ▼
Response {"UPDATE SUCCESS IN PG MESH"}

Any caller
    │
    ▼
[API — read]  serviceName=PNS_SETTING  {token, glusrid, modid}
    │  ActionPnsSetting → UserPnsSettingModel.go (read)
    ▼
Response — PNS-setting data
```

---

## 7. Optimization Scope — DB Response-Time Contribution

### Low impact / good practice already present

1. **`ON CONFLICT DO NOTHING` on insert** — already efficient, avoids a separate
   pre-check query for duplicate-detection.
2. **Type-based branching keeps queries targeted** (no unnecessary `contact_id` filter
   on the general-setting types) — appropriately scoped `WHERE` clauses.

### Code-hygiene note (not performance)

3. **Commented-out `noOfPnsMapping`/`checkPNSDataCorrection` logic** — if this
   "max 5 mappings" rule is truly obsolete, removing the dead code would improve
   readability; if it's meant to be re-enabled, it should be tracked as a known gap
   rather than silently disabled.

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora PNS Setting flowchart yahan dekho](https://lucid.app/lucidchart/f710b6bb-7b9a-4869-ace2-924ec0a7e375/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **"Max 5 contact-mappings per user" rule is currently unenforced** (dead/commented
   code) — a user could theoretically accumulate unlimited PNS-setting-rows if this
   was meant to be a hard cap.
2. **`IIL_PNS_SETTING_TYPE` codes (`M1`/`M2`/`L1`/`L2`/`T`/`M`/`L`) have no in-code
   enum/comment explaining their meaning** — likely Mobile/Landline variants (1/2 =
   primary/alternate?) but not confirmed.
3. **`pnsDelHistoryPG`'s full behavior wasn't traced** — worth a follow-up read if
   delete/audit-trail behavior needs to be documented in detail.

---

## 10. Open Questions

1. What do `IIL_PNS_SETTING_TYPE` codes `M1`/`M2`/`L1`/`L2`/`T`/`M`/`L` represent
   exactly?
2. Is the "max 5 contact-mappings" rule intentionally disabled, or an oversight that
   should be restored?
3. What does `pnsDelHistoryPG` record, and when is it triggered end-to-end?
4. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`PNS_Setting_Business_Doc.md`](./PNS_Setting_Business_Doc.md) — product perspective
- [`../PNS KT/PNS_Technical_Doc.md`](../PNS%20KT/PNS_Technical_Doc.md) —
  unrelated concept, documented separately (no shared table/FK/controller)

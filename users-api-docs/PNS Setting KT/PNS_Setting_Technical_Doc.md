# PNS Setting — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`PNS_Setting_Business_Doc.md`](./PNS_Setting_Business_Doc.md)
dekho — dono docs same flows cover karte hain, bas alag audience ke liye.

**Repos**: `service-api-go-production` (write), `users-api-go-production` (read),
`user-temp-consumers-production` (auto-sync consumer — naya finding, purane doc mein
missing tha).

**Scope note**: PNS Setting aur PNS genuinely **do alag concepts** hain — alag tables
(`IIL_PNS_SETTING` vs `GL_GSM_MASTER`), alag controllers, koi FK-relation ya
shared-code nahi mila. Dekho [`../PNS KT/PNS_Technical_Doc.md`](../PNS%20KT/PNS_Technical_Doc.md).

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file path diya
gaya hai har jagah). Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — team se confirm karo]** likh diya hai, guess nahi kiya.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write controller | write | [`UserPnsSettingController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserPnsSettingController.go) (`UserPnsSettingController`) |
| Write model — core upsert/delete logic | write | [`UserPnsSettingModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserPnsSettingModel.go) — `UpsertPnsSetting`, `getContactNumber`, `checkDupPnsMapping`, `HandleDuplicatePnsUpdate`, `InsertPnsSetting`, `UpdatePnsSetting`, `DeletePnsSetting`, `deleteById`, `isPnsMappingAvailable`, `noOfPnsMapping` (dead), `pnsDelHistoryPG`, `checkPNSDataCorrection`/`deleteFromMeshPG` (dead) |
| Validation | write | `MandatoryParamscheckPnsSetting` — [`UserUtilsMandatory.go:127-198`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go), `UserPnsSettingMap` — [`UsersValidationMaps.go:70-81`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |
| Read controller | read | [`PnsSettingControllers.go`](../internal/controllers/UsersControllers/PnsSettingControllers.go) (`ActionPnsSetting`) |
| Read model | read | [`UserPnsSettingModel.go`](../internal/models/users/UserPnsSettingModel.go) (`GetPnsSettingModel`) |
| **Auto-sync consumer (naya, purane doc mein missing tha)** | consumers | [`USER_UPSERT_ADDITIONAL.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_ADDITIONAL.go) (queue `USER_UPSERT_ADDITIONAL`) — jab supplier apna mobile/landline/GSM number kahin aur (general profile-edit) se change karta hai, yeh consumer automatically PNS-setting write-API ko HTTP call karke settings ko naye number se sync rakhta hai |
| Auto-sync helper functions | consumers | [`additional.go`](../../user-temp-consumers-production/user-temp-consumers-production/pkg/utils/additional.go) — `PnsRemoval`, `PnsUpdate`, `Export_pns`, `Update_mesh_pg_pns`, `CallService_v1`, `PnsCountPG`, `PnsCheck` |

---

## 2. Routes

| Method | Path / serviceName | Repo | Controller |
|---|---|---|---|
| POST | `/pnssetting` (serviceName `PNS_SETTING_SERVICE`) | write | `UserPnsSettingController` — [`router.go:151,319`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) |
| GET/POST | `/pnssetting/*params` (serviceName `PNS_SETTING`) | read | `ActionPnsSetting` — [`routerUsers.go:345-346,516-517,703-704`](../internal/api/users_router/routerUsers.go) (registered thrice, once per API "profile" e.g. users1/users2/etc.) |

**Cross-service call (not an external-facing route, but a real HTTP hop)**: `user-temp-consumers-production`
itself calls `POST http://service.intermesh.net/pnssetting` internally (`CallService_v1`,
[`additional.go:1246-1307`](../../user-temp-consumers-production/user-temp-consumers-production/pkg/utils/additional.go))
— i.e. the write endpoint has a second caller besides the documented GLADMIN/MAPI/etc.
front-ends: the platform's own consumer, auto-reconciling settings after contact-number
changes (see section 6, Flow C).

---

## 3. Data Model — Table

> **Verification note**: table/column names Go code ke andar embedded SQL strings se liye
> gaye hain. Live DB schema se cross-verify **nahi** kiya gaya hai.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `IIL_PNS_SETTING` | meshpg | Per-user (aur optionally per-additional-contact) PNS off-hours-call-routing preference | `IIL_PNS_SETTING_ID`, `FK_GLUSR_USR_ID`, `FK_GLUSR_USR_ADDT_CONTACT_ID` (nullable — links to a specific additional-contact record), `IIL_PNS_SETTING_TYPE` (`M1`/`M2`/`L1`/`L2`/`T`/`M`/`L` — **decoded in section 4**), `IIL_PNS_SETTING_OFFHRS_FLAG` (`0`/`1`/`2`/`3`), `IIL_PNS_SETTING_CONTACT_NUMBER`, `IIL_PNS_SETTING_LAST_UPDATE`, `IIL_PNS_SETTING_UPDATED_ID`/`_NAME`/`_SCREEN` — [`UserPnsSettingModel.go:272,329,335,365,369`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserPnsSettingModel.go) |
| `GLUSR_USR` | meshpg | Read-only reference — primary/alt mobile aur landline numbers, GSM number nikaale jaate hain PNS-type ke against contact-number resolve/validate karne ke liye | `GLUSR_USR_IM_GSM`, `GLUSR_USR_PH_NUMBER`, `GLUSR_USR_PH2_NUMBER`, `GLUSR_USR_PH_MOBILE`, `GLUSR_USR_PH_MOBILE_ALT` — [`getContactNumber`, `UserPnsSettingModel.go:154`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserPnsSettingModel.go); read-side query joins same columns — [`UserPnsSettingModel.go:97` (read repo)](../internal/models/users/UserPnsSettingModel.go) |
| `GLUSR_USR_ADDT_CONTACT` | meshpg | Read-only reference — supplier ke additional contact numbers (multiple ho sakte hain) | `GLUSR_USR_ADDT_CONTACT_ID`, `GLUSR_USR_ADDT_PH_NUMBER`, `GLUSR_USR_ADDT_PH_MOBILE`, `GLUSR_USR_ADDT_TOLLFREE_NUMBER` — [`getContactNumber`, `UserPnsSettingModel.go:157`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserPnsSettingModel.go) |
| History mechanism — **no dedicated table found**, uses a stored procedure | mainPg/meshpg (procedure call, DB not identifiable from Go alone) | `pnsDelHistoryPG` calls `pck_gl_history_sp_add_custom_history(...)` — a generic history stored-procedure (naming pattern matches GST domain's `GL_ALL_MASTER_HISTORY`/`gl_history` audit mechanism, but this repo review could not open the procedure body to confirm the target table) — [`UserPnsSettingModel.go:436-455`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserPnsSettingModel.go) | Comment string written: `"<number> contact number unmapped from PNS"`, action code `'U'` |

**Resolved from old doc's open question**: `pnsDelHistoryPG`'s full behavior — it is called
after every successful **delete** (`deleteById` and `DeletePnsSetting`, not on insert/update)
and invokes a stored procedure, not a plain `INSERT`. The exact underlying history table is
opaque from Go code (procedure body is server-side) — flagged in Open Questions.

---

## 4. `IIL_PNS_SETTING_TYPE` — Decoding the Type Codes (resolves old doc's open question)

The old doc flagged the 7 type codes as unexplained. Tracing the **read-side query**
([`UserPnsSettingModel.go` read repo, lines 81-134](../internal/models/users/UserPnsSettingModel.go))
and the write-side `getContactNumber`/`Export_pns` functions conclusively maps every code to
its source column:

| Type | Source of the phone number | Contact scope |
|---|---|---|
| `M1` | `GLUSR_USR_PH_MOBILE` (primary mobile) | General — per user, no `contact_id` |
| `M2` | `GLUSR_USR_PH_MOBILE_ALT` (alternate mobile) | General — per user, no `contact_id` |
| `L1` | `GLUSR_USR_PH_NUMBER` (primary landline, area+number) | General — per user, no `contact_id` |
| `L2` | `GLUSR_USR_PH2_NUMBER` (secondary landline, area+number) | General — per user, no `contact_id` |
| `M` | `GLUSR_USR_ADDT_CONTACT.GLUSR_USR_ADDT_PH_MOBILE` | Specific — per additional-contact, needs `contact_id` |
| `L` | `GLUSR_USR_ADDT_CONTACT.GLUSR_USR_ADDT_PH_NUMBER` | Specific — per additional-contact, needs `contact_id` |
| `T` | `GLUSR_USR_ADDT_CONTACT.GLUSR_USR_ADDT_TOLLFREE_NUMBER` | Specific — per additional-contact, needs `contact_id` |
| `D` (write-side only, not a real PNS type) | n/a | Special sentinel: `pns_type=="D"` + `action_flag=="d"` triggers `deleteById()` — delete **all** PNS-setting rows for a given `(glusr_id, contact_id)` pair, regardless of type. Distinct code path from the normal per-type delete. [`UpsertPnsSetting`, line 61](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserPnsSettingModel.go) |

`IIL_PNS_SETTING_OFFHRS_FLAG` values seen in code: mandatory-check requires `1`/`2`/`3` for
insert/update and exactly `0` for delete
([`UserUtilsMandatory.go:189`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)).
The read-side "flag=1 bulk-check" query filters `IIL_PNS_SETTING_OFFHRS_FLAG IN (2,3)`
([`UserPnsSettingModel.go:42`, read repo](../internal/models/users/UserPnsSettingModel.go)) —
i.e. `2`/`3` are treated as the "off-hours routing active" states while `1` reads as
default/on-hours. **Exact business meaning of 1 vs 2 vs 3 is not spelled out anywhere in
code** — **[INFERRED — confirm with PNS/telephony team]**.

---

## 5. Business Rules & Validation (exhaustive, code-cited)

1. **Gateway allowlist**: only `GLADMIN`, `MAPI`, `BUYERMY`, `SELLERMY`, `ANDROID`, `IOS` can
   call the write API. [`UserPnsSettingController.go:59`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserPnsSettingController.go)
2. **Mandatory fields**: `glusr_id`, `VALIDATION_KEY`, `ip`, `ipcountry`, `updatedby_name`,
   `updatedby_screen`, `updatedby_id`, `pns_type`, `offhrs_flag`, `action_flag` all required;
   `action_flag` must be one of `i`/`u`/`d`; `pns_type` must be one of `M1/M2/L1/L2/M/L/T/D`.
   [`UserUtilsMandatory.go:127-198`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
3. **Length/type validation**: `UserPnsSettingMap` enforces `contact_id` ≤10-digit number,
   `pns_type` ≤2 chars, `offhrs_flag` 1-digit number, `contact_number` ≤50 chars,
   `updatedby_screen` ≤255 chars. [`UsersValidationMaps.go:70-81`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go)
4. **Contact number is never trusted from the request — it is always re-derived server-side**
   from `GLUSR_USR`/`GLUSR_USR_ADDT_CONTACT` based on `pns_type` + `glusr_id`(+`contact_id`),
   via `getContactNumber()`. If the resolved number is non-numeric (regex `[^0-9]` match) or
   absent, the write is rejected as `"INVALID"`.
   [`UserPnsSettingModel.go:148-224`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserPnsSettingModel.go)
5. **GSM must exist** — if `glusr_usr_im_gsm` is empty for the user, the entire request fails
   with `"GSM Number does not exist"` — GSM is treated as a prerequisite identity anchor for
   PNS settings even though GSM itself is never written by this API.
   [`UpsertPnsSetting`, line 113-114](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserPnsSettingModel.go)
6. **Cross-type duplicate-number check on insert/update**: before insert/update,
   `checkDupPnsMapping()` checks whether the *same resolved contact number* is already mapped
   under **any** `IIL_PNS_SETTING_TYPE` for this user. If yes and it's a fresh mapping (not the
   exact same type+contact being updated), the request either gets rejected
   (`"THIS CONTACT NUMBER IS ALREADY MAPPED"`) or routed into a special duplicate-resolution
   path (rule 7). [`checkDupPnsMapping`, `UserPnsSettingModel.go:654-679`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserPnsSettingModel.go)
7. **Duplicate-update conflict resolution (`HandleDuplicatePnsUpdate`)** — if an `u` (update)
   request's number collides with an existing different type/contact mapping: compares the
   new `offhrs_flag` against the existing row's flag — if new flag ≤ old flag, the **existing**
   conflicting row is deleted; if new flag > old flag, the requested row is updated **and**
   the old conflicting row is separately deleted. Net effect: a contact number can only be
   mapped to PNS settings under **one** type at a time; the "higher/equal offhrs_flag wins"
   rule silently prunes the other. [`UserPnsSettingModel.go:121-146`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserPnsSettingModel.go)
8. **`contact_id` requirement split by type**: `M1`/`M2`/`L1`/`L2` (general types) never need
   `contact_id`; `M`/`L`/`T` (additional-contact types) require it — insert rejects with
   `"contact_id can not be empty."` otherwise, update/delete reject with
   `"No data present for update/delete."` if `contact_id` is blank.
   [`InsertPnsSetting:269-270`, `UpdatePnsSetting:332-333`, `DeletePnsSetting:368-372`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserPnsSettingModel.go)
9. **Insert is idempotent**: `INSERT ... ON CONFLICT DO NOTHING` — a byte-identical duplicate
   insert silently no-ops rather than erroring, but this is **guarded upstream** by rule 10
   below (`isPnsMappingAvailable`) which already blocks a duplicate insert with an explicit
   error before the SQL even runs — so `ON CONFLICT DO NOTHING` in practice is a defensive
   fallback, not the primary duplicate-guard.
   [`UserPnsSettingModel.go:272`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserPnsSettingModel.go)
10. **Explicit "already mapped" check before insert**: `isPnsMappingAvailable()` counts
    existing rows for `(glusr_id, pns_type[, contact_id])`; insert is rejected
    (`"PNS already exists with same PNS type."`) if a row already exists for that exact
    type(+contact). [`UpsertPnsSetting:87`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserPnsSettingModel.go)
11. **"Max 5 mappings" rule is DISABLED at the write-API layer** — `noOfPnsMapping()` and
    `checkPNSDataCorrection()` calls inside `InsertPnsSetting` are commented out
    (lines 260-293, 457-613) — **but the rule is still enforced one layer up**, in the
    auto-sync consumer (`user-temp-consumers-production`, see section 6 Flow C) via its own
    `PnsCountPG()` + `recordsCount < 5` check before calling the insert API. So: a *direct*
    caller (GLADMIN/MAPI/etc. hitting `/pnssetting` themselves) is **not** capped at 5 by the
    write API itself; only the automatic-sync path enforces the cap.
    [`UserPnsSettingModel.go:260-293`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserPnsSettingModel.go),
    [`additional.go:1128-1137,1177-1183`](../../user-temp-consumers-production/user-temp-consumers-production/pkg/utils/additional.go)
    — **this resolves the old doc's open question #2** ("is the max-5 rule intentionally
    disabled?") with a more precise answer: disabled in one path, alive in another.
12. **Delete-by-contact-id sentinel** (`pns_type=="D"` + `action_flag=="d"`) deletes **every**
    PNS-setting row for a `(glusr_id, contact_id)` pair in one shot — used when an entire
    additional-contact record is being removed (see section 6 Flow D), not when a single
    type mapping is being removed. [`UpsertPnsSetting:61`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserPnsSettingModel.go)
13. **Every successful delete writes a history/audit entry** via `pnsDelHistoryPG` (stored
    procedure call) — inserts, by contrast, do **not** write any audit trail.
    [`deleteById:245`, `DeletePnsSetting:393`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserPnsSettingModel.go)
14. **Employee name auto-lookup**: if `updatedby_name != "User"` (i.e. an internal caller),
    the system looks up the real employee name via `Employee_mesh_pg()` rather than trusting
    the caller-supplied name. [`UpsertPnsSetting:53-55`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserPnsSettingModel.go)

---

## 6. RabbitMQ / Kafka / Redis

- **RabbitMQ**: no queue is directly published or consumed by the PNS Setting write/read code
  itself (`UserPnsSettingController.go`, `UserPnsSettingModel.go` in either repo) — confirmed
  by grep across all three repos for `PNS_SETTING`, no `PushToQueue`/consumer registration
  found for it. The related but separate `PNS_RATIO` string appears once, only as a validation
  field name in an unrelated model ([`UserAddUpdateUtils.go:400`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddUpdateUtils.go)) — **not** the same
  feature, do not conflate.
- **However**, the write API is reached indirectly via a **queue-triggered HTTP call**: the
  `USER_UPSERT_ADDITIONAL` RabbitMQ consumer (part of the general contact/GSM-update fan-out,
  not PNS-specific) calls `PNS_SETTING_SERVICE`'s HTTP endpoint synchronously when it detects
  mobile/landline/GSM changes — see section 7, Flow C. This is a real cross-repo dependency
  that the old doc's "no RabbitMQ" claim missed nuance on.
- **Kafka**: no reference found in any PNS-Setting-touching file across all three repos.
- **Redis**: no reference found in any PNS-Setting-touching file across all three repos.
  Both reads (`GET /pnssetting`) hit Postgres live every time.

---

## 7. End-to-End Technical Flows

### Flow A — Direct write (GLADMIN/MAPI/BUYERMY/SELLERMY/Android/iOS insert or update)

```
Caller (GLADMIN/MAPI/BUYERMY/SELLERMY/Android/iOS)
    │
    ▼
[API — write]  POST /pnssetting  serviceName=PNS_SETTING_SERVICE
                {glusr_id, contact_id?, pns_type, offhrs_flag, action_flag(i/u),
                 updatedby_id, updatedby_screen, ip, ipcountry, VALIDATION_KEY}
    │  UserPnsSettingController.go — Gateway allowlist check
    │  MandatoryParamscheckPnsSetting() → LengthAndTypeValidations_v3()
    ▼
UpsertPnsSetting()
    │  1. getContactNumber() — re-derive actual phone number from GLUSR_USR /
    │     GLUSR_USR_ADDT_CONTACT server-side (never trust request's number)
    │  2. checkDupPnsMapping() — is this contact number already mapped to a
    │     DIFFERENT type/contact for this user?
    │       ├─ yes, conflicting → HandleDuplicatePnsUpdate() (flag-priority resolution)
    │       └─ no
    ▼
    3. noOfPnsMapping() [dead/commented] + isPnsMappingAvailable() — does a row
       already exist for this exact type(+contact)?
    ▼
    action=i, none exists → InsertPnsSetting()
        INSERT INTO IIL_PNS_SETTING (...) ON CONFLICT DO NOTHING
    action=u, one exists  → UpdatePnsSetting()
        type in {M1,M2,L1,L2} → UPDATE WHERE (glusr_id, type)
        type in {T,M,L}       → UPDATE WHERE (glusr_id, contact_id, type)
    ▼
[DB — meshpg]  IIL_PNS_SETTING
    ▼
Response {"CODE":"200","MESSAGE":"UPDATE SUCCESS"}
```

### Flow B — Direct delete (single type mapping)

```
Caller → POST /pnssetting {action_flag=d, offhrs_flag=0, pns_type=M1/M2/L1/L2/M/L/T}
    ▼
UpsertPnsSetting() → getContactNumber() (still re-resolves number for validation)
    ▼
DeletePnsSetting()
    type in {M1,M2,L1,L2} → DELETE WHERE (glusr_id, type)
    type in {T,M,L}       → DELETE WHERE (glusr_id, contact_id, type)
    ▼
[DB — meshpg]  IIL_PNS_SETTING (DELETE)
    ▼
pnsDelHistoryPG() → CALL pck_gl_history_sp_add_custom_history(...)  [audit trail]
    ▼
Response {"CODE":"200","MESSAGE":"UPDATE SUCCESS"}
```

### Flow C — Automatic re-sync when supplier changes mobile/landline/GSM elsewhere (NEW — missed by old doc)

```
Supplier changes mobile/landline/GSM number via the general profile-edit flow
(NOT this feature's own API — a different controller entirely, akin to the
GST domain's "UserDetailsController" contact-details path)
    │
    ▼
[RabbitMQ]  general profile-update fan-out publishes old-vs-new number diff
    │  (keys like OLD_GLUSR_USR_IM_GSM, old_GLUSR_USR_PH_MOBILE, IM_GSM_REMOVAL etc.)
    ▼
[CONSUME]  USER_UPSERT_ADDITIONAL.go
    │
    ├─ IM_GSM_REMOVAL present → PnsRemoval()
    │     → CallWapiService(..., "USER_UPSERT_ADDITIONAL")  [separate WAPI removal path]
    │
    └─ OLD_GLUSR_USR_IM_GSM / old mobile-alt / old landline keys present → PnsUpdate()
          → Export_pns()  — diffs M1/M2/L1/L2 old-vs-new values, decides
             insert vs update vs delete per type, also checks additional-contact
             numbers for matches
          → Update_mesh_pg_pns()
                for each affected type:
                ├─ CallService_v1() → HTTP POST http://service.intermesh.net/pnssetting
                │    (i.e. calls the SAME write API from Flow A, but as a service caller,
                │     not a human/app — action=i/u/d chosen per diffed value)
                └─ "max 5 mappings" cap enforced HERE via PnsCountPG() before any insert
                   call — NOT inside the write API itself (see rule 11)
    ▼
[DB — meshpg]  IIL_PNS_SETTING updated/deleted to match the supplier's new numbers
```

**Business-critical insight**: this means PNS settings are **not purely supplier-driven** —
they silently self-correct whenever the underlying phone number changes anywhere else in the
profile. A support ticket like "meri PNS setting apne aap badal gayi" is explained by this
flow, not by any bug.

### Flow D — Additional-contact deletion cascades into PNS-setting deletion (NEW — missed by old doc)

```
Supplier deletes an additional contact (Type="ContactDetails", flag_del="D")
via the general Contact-Details write path (UserDetailsModel.go — shared code
with several other profile features, not PNS-Setting-specific)
    │
    ▼
rabbitMqArrayFunc_v1() queues a MESSAGE entry:
    {"TABLES": "IIL_PNS_SETTING", "ACTION": "DELETE",
     "COLUMNS": {FK_GLUSR_USR_ADDT_CONTACT_ID, FK_GLUSR_USR_ID}}
    (queued alongside the GLUSR_USR_ADDT_CONTACT delete in the same RabbitMQ packet)
    ▼
[DB — synchronous, same request]  DELETE FROM GLUSR_USR_ADDT_CONTACT WHERE ...
    ▼
[RabbitMQ publish]  SERVICENAME=USER_DETAIL_BANNED_SERVICE (for ContactDetails/BankDetails/Franchise types)
    ▼
[CONSUME — downstream, not traced end-to-end in this pass]  presumably replicates the
    queued IIL_PNS_SETTING delete onto replica/downstream stores
```

**Note**: this pass confirmed the *write-side queuing* of the IIL_PNS_SETTING delete
instruction (`UserDetailsModel.go:1523-1542`) but did **not** conclusively trace which
consumer executes it end-to-end — flagged in Open Questions.

### Flow E — Read

```
Any caller (token+glusrid+modid, OR bulk glusrid list via a comma-separated glusrid + flag)
    │
    ▼
[API — read]  GET/POST /pnssetting/*params  serviceName=PNS_SETTING
    │  ActionPnsSetting — token/glusrid/modid validity check
    │  if glusrid contains "," → bulk-flag path (flag="1")
    ▼
GetPnsSettingModel()
    ├─ flag=="1" (bulk, ≤10 glids, digits+commas only):
    │     SELECT DISTINCT fk_glusr_usr_id FROM iil_pns_setting
    │     WHERE fk_glusr_usr_id IN (...) AND iil_pns_setting_offhrs_flag IN (2,3)
    │     → "which of these GLIDs currently have an off-hours-active PNS setting"
    │
    └─ flag=="" (single glusrid, full detail):
          One CTE query joins GLUSR_USR (M1/M2/L1/L2 slots) + GLUSR_USR_ADDT_CONTACT
          (M/L/T slots per additional-contact) against IIL_PNS_SETTING (dedup'd via
          ROW_NUMBER() partition), regex-strips non-digits from every phone column
          before matching against the stored (already-digit-only) contact_number
          → returns a keyed object: GSM, L1, L2, M1, M2, La{n}, Ma{n}, T{n} per
            additional contact, each with area_code/country_code/number/pns_type/
            pns_status/contact_id/id
    ▼
Response — PNS-setting data (JSON)
```

---

## 8. Flow-wise DB & Table Usage

### Flow A — Direct Insert/Update

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `GLUSR_USR` or `GLUSR_USR_ADDT_CONTACT` | SELECT (`getContactNumber`) | Actual phone number resolve karna type ke hisaab se, request ke number ko trust nahi karte |
| 2 | meshpg | `IIL_PNS_SETTING` | SELECT (`checkDupPnsMapping`) | Same number kisi aur type/contact ke against already mapped toh nahi |
| 3 | meshpg | `IIL_PNS_SETTING` | SELECT (`isPnsMappingAvailable`) | Exact type(+contact) ke liye already row exist karti hai kya (insert-vs-update gate) |
| 4 | meshpg | `IIL_PNS_SETTING` | INSERT / UPDATE | Actual write |

### Flow B — Direct Delete

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `GLUSR_USR`/`GLUSR_USR_ADDT_CONTACT` | SELECT (`getContactNumber`) | Same resolve step, delete mein bhi chalta hai |
| 2 | meshpg | `IIL_PNS_SETTING` | DELETE | Row(s) hataana |
| 3 | meshpg (procedure) | history (opaque) | CALL stored procedure | Audit-trail entry |

### Flow C — Auto-sync consumer (per changed type, repeated for each affected slot)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `GLUSR_USR_ADDT_CONTACT` | SELECT (`Export_pns`) | Additional-contact numbers fetch karna diff-check ke liye |
| 2 | meshpg | `IIL_PNS_SETTING` | SELECT count (`PnsCountPG`) | 5-mapping cap check (this layer's own enforcement of rule 11) |
| 3 | meshpg | `IIL_PNS_SETTING` | SELECT count-by-type / count-by-number (`Update_mesh_pg_pns`) | Decide insert vs update vs delete per changed type |
| 4 | (HTTP, not DB) | — | POST `/pnssetting` (`CallService_v1`) | Delegates the actual write back to Flow A's own API — **all of Flow A's DB round-trips repeat here per call** |

**Note**: this makes a single mobile-number change potentially trigger up to 4 separate
full `/pnssetting` write-cycles (M1/M2/L1/L2 slots), each with its own 3-4 DB round-trips —
a real multiplicative cost worth knowing (see section 9, optimization point #2).

### Flow D — Contact deletion cascade

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `GLUSR_USR_ADDT_CONTACT` | DELETE | Contact record removed |
| 2 | (RabbitMQ, not direct DB) | `IIL_PNS_SETTING` | queued DELETE instruction | Downstream consumer removes matching PNS-setting rows (not traced end-to-end) |

### Flow E — Read

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 (bulk) | meshpg | `iil_pns_setting` | SELECT DISTINCT | Kaunse GLIDs ka off-hours flag currently 2/3 hai |
| 1 (single) | meshpg | `iil_pns_setting` + `glusr_usr` + `glusr_usr_addt_contact` | SELECT (1 CTE query, 2 json_agg sub-selects) | Poora PNS-setting state ek response mein — general (M1/M2/L1/L2) aur additional-contact (M/L/T) dono ek hi query mein |

---

## 9. Optimization Scope — DB Response-Time Contribution

### High-impact

1. **Flow C (auto-sync) can fan out into up to 4 sequential full write-cycles for one
   mobile-number change** (M1/M2/L1/L2, each calling the write API over HTTP one at a time
   in a `for` loop, not parallelized) — each HTTP call itself carries the full 3-4 query cost
   of Flow A, *plus* network/HTTP overhead (`http.Client{Timeout: 5*time.Second}` per call,
   with up to 2 retries on failure) on top of whatever triggered it. A single profile-update
   event could realistically cost 4× write-API-latency + up to 8 HTTP attempts (2 retries ×
   4 types) if the write API is slow. [`Update_mesh_pg_pns`, `additional.go:1077-1244`](../../user-temp-consumers-production/user-temp-consumers-production/pkg/utils/additional.go)
   **Suggestion**: parallelize the per-type calls (they're independent), or better, expose an
   internal bulk-upsert path so the consumer does one DB-level batch write instead of N HTTP
   round-trips to itself.
2. **Every write request re-resolves the contact number from `GLUSR_USR`/
   `GLUSR_USR_ADDT_CONTACT` even for a plain delete**, where the number is arguably not
   needed to identify the row (delete matches on `glusr_id`+`type`+`contact_id`, not on the
   number). This is an extra SELECT that could be skipped for the delete path.
   [`UpsertPnsSetting:56`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserPnsSettingModel.go)

### Medium-impact

3. **No caching on the read endpoint** — `GET /pnssetting` runs a non-trivial multi-join CTE
   query (with regex-strip on 10+ columns) live every time. PNS settings change infrequently
   per supplier; a short-TTL cache-aside (Redis, not currently wired into this domain — see
   section 6) would cut this cost for repeat reads.
4. **`checkDupPnsMapping` + `isPnsMappingAvailable` are two separate SELECTs that could
   plausibly be merged** into one query returning both "is this number mapped elsewhere" and
   "does this exact type+contact already exist," saving one round-trip per write.

### Low-impact / good practice already present

5. **`ON CONFLICT DO NOTHING` on insert** — avoids a race-condition duplicate-insert error,
   good defensive pattern even though it's mostly redundant given rule 10's upstream check.
6. **Type-based branching keeps queries targeted** (no unnecessary `contact_id` filter on the
   general-setting types) — appropriately scoped `WHERE` clauses throughout.

### Code-hygiene note (not performance)

7. **~160 lines of fully commented-out logic** (`checkPNSDataCorrection`, `deleteFromMeshPG`,
   the `noOfPnsMapping` call-sites in `InsertPnsSetting`) sit dead in the write model — if the
   "max 5" + "data correction" rule is truly superseded by the consumer-side enforcement
   (section 5, rule 11), this dead code should be removed; if it was meant to be re-enabled
   as a defense-in-depth check, that's a real gap worth flagging to the team.
   [`UserPnsSettingModel.go:260-293,457-613`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserPnsSettingModel.go)

---

## 10. Cron Inventory

No PNS-Setting-related cron found. Searched `user-temp-consumers-production` (no `crons/`
directory in this repo) and both API repos for any scheduled job referencing
`IIL_PNS_SETTING`/`PNS_SETTING` — none found. All PNS-setting writes are either
direct-API-driven (Flow A/B) or event-driven via the `USER_UPSERT_ADDITIONAL` consumer
(Flow C) — nothing runs on a timer.

---

## 11. Edge Cases & Gotchas (technical POV)

1. **PNS settings are not purely supplier-controlled** — they auto-adjust whenever the
   underlying phone number changes elsewhere in the profile (Flow C). A supplier who never
   touches `/pnssetting` directly can still see their settings change.
2. **"Max 5 mappings" is enforced in one path and not the other** — direct API callers
   (GLADMIN/MAPI/etc.) are not capped by the write API itself (dead code, section 5 rule 11);
   only the auto-sync consumer respects the cap before inserting. A GLADMIN tool that calls
   the write API directly could, in theory, create more than 5 mappings for a user.
3. **A contact number can only carry one active PNS-type mapping at a time** — if the same
   number is reused across two different "slots" (e.g. it's both the additional-contact
   mobile *and* gets set as the user's primary mobile), `HandleDuplicatePnsUpdate`'s
   flag-priority logic will silently delete one of the two mappings (section 5, rule 7) —
   this is an easy source of "my setting disappeared" confusion.
4. **`pns_type="D"` is a special sentinel, not a real PNS-type** — `deleteById()` wipes
   *all* type-rows for a contact_id in one call; don't confuse it with a genuine per-type
   delete (`action_flag=d` + a real type code like `M`/`L`/`T`).
5. **Contact-deletion cascade into `IIL_PNS_SETTING` is queued via RabbitMQ, not executed
   synchronously in the same DB transaction as the contact deletion** (Flow D) — there is a
   window where a `GLUSR_USR_ADDT_CONTACT` row is gone but its `IIL_PNS_SETTING` row still
   exists until the downstream consumer processes the queued delete.
6. **Insert never writes an audit-history row; only delete does** — if debugging "who added
   this PNS mapping and when," `IIL_PNS_SETTING_LAST_UPDATE`/`_UPDATED_ID`/`_NAME`/`_SCREEN`
   on the row itself is the only trail; the history stored-procedure is delete-only.
7. **`~160` lines of commented-out "data correction" logic remain in the codebase** — if
   re-enabled accidentally (e.g. via a careless uncomment during a future change), its
   behavior has never been exercised against current data and could behave unpredictably.

---

## 12. Open Questions

1. What does `pck_gl_history_sp_add_custom_history` actually write to — which table, and
   is it shared with other domains (naming pattern suggests a generic `gl_history`-style
   mechanism, similar to GST's `GL_ALL_MASTER_HISTORY`)? Not confirmable from Go code alone.
2. Which consumer actually executes the queued `IIL_PNS_SETTING` DELETE from Flow D
   (contact-deletion cascade)? The write side queues the instruction
   (`UserDetailsModel.go:1523-1542`) but this pass could not conclusively identify the
   consumer that processes it.
3. Exact business meaning of `IIL_PNS_SETTING_OFFHRS_FLAG` values `1` vs `2` vs `3` (all
   valid for insert/update) — code treats `2`/`3` as "off-hours active" for the bulk-read
   check, but the distinction between `2` and `3`, and what `1` specifically represents,
   isn't documented in code. **[INFERRED — confirm with PNS/telephony team]**.
4. Is the direct-API "no max-5 cap" (section 5, rule 11) an intentional design (only the
   auto-sync path should be capped, direct admin tools are trusted) or an oversight where
   the write-API-level cap was disabled and never restored?
5. Live DB schema verification (column types, nullability, indexes, constraints) — this doc
   reflects only what Go SQL strings imply.

---

## See also

- [`PNS_Setting_Business_Doc.md`](./PNS_Setting_Business_Doc.md) — product perspective
- [`../PNS KT/PNS_Technical_Doc.md`](../PNS%20KT/PNS_Technical_Doc.md) —
  unrelated concept, documented separately (no shared table/FK/controller)
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — another domain
  showing the same "general write-model shared across many profile-detail types" pattern
  (`UserDetailsModel.go`) that Flow D's cascade originates from

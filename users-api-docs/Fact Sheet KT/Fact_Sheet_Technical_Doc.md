# Fact Sheet — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`Fact_Sheet_Business_Doc.md`](./Fact_Sheet_Business_Doc.md)
dekho — dono docs same flows cover karte hain, bas alag audience ke liye.

**Scope note**: Fact Sheet is NOT a standalone controller — it is one `Type`/`detailType`
branch of the same **shared multi-purpose user-details write/read endpoint** that also
handles GST (`CompRgst`), Bank Details, and Form-8. Dekho
[`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) aur
[`../Bank Details KT/Bank_Details_Technical_Doc.md`](../Bank%20Details%20KT/Bank_Details_Technical_Doc.md)
for the shared controller's overall architecture (gateway, dispatch-pattern) — this doc
focuses on the Fact-Sheet-specific branch/table, aur jahan bhi shared RabbitMQ plumbing
Fact-Sheet ko differently touch karti hai (jaise section 6 ka double-publish finding),
wahan poora detail diya gaya hai, sirf reference nahi.

**Repos**: `service-api-go-production` (write, shared `UserDetailsController`/
`UserDetailsModel`), `users-api-go-production` (read, shared `UserOtherDetailModel`
`detailType` dispatch). No dedicated Fact-Sheet consumer or cron found in
`user-temp-consumers-production` (confirmed via grep, section 10, 11).

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file path diya
gaya hai har jagah). Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — team se confirm karo]** likh diya hai, guess nahi kiya.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write controller (shared, gateway + mandatory-field check) | write | [`UserDetailsController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDetailsController.go) — gateway allowlist line 508-529 (generic, no FactSheet-specific gate restriction), delete-of-audit-keys block line 287 |
| Write model (shared, `Type=="FactSheet"` branch) | write | [`UserDetailsModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go) — branch **line 750-864**; shared RabbitMQ fan-out (`tabArr`) **line 4169-4222** |
| Validation map (length/type only, no mandatory-field check) | write | [`UsersValidationMaps.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) — `UserDetails_FactSheet` map **line 1663-1695**, dispatch **line 2449-2451** |
| `LengthAndTypeValidations_v2` helper (shared) | write | [`globalfunctions.go`](../../service-api-go-production/service-api-go-production/pkg/utils/globalfunctions.go) — **line 484-520** |
| Dead/unused constant referencing `GLUSR_USR_FACT_SHEET` | write | [`commonMaps.go`](../../service-api-go-production/service-api-go-production/pkg/components/commonMaps.go) — `DetailsAPItab` **line 114**, declared but **zero references** anywhere in the repo (confirmed via grep) |
| Write router | write | [`router.go`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) — `POST /details` registered line 107, 141, 369 |
| Read model (shared `detailType` dispatch) | read | [`UserOtherDetailModel.go`](../internal/models/users/UserOtherDetailModel.go) — table-name switch line 137, dedicated SELECT **line 228-267**, shared CompRgst/FactSheet post-processing **line 580-605** |
| Read model — combined full-profile (`tsDataCall`) query | read | [`UserOtherDetailModel.go`](../internal/models/users/UserOtherDetailModel.go) — `fact_sheet_data` sub-select **line 509**, merge-back **line 986** |

---

## 2. Routes

| Method | Path | Type / detailType | Repo | Controller |
|---|---|---|---|---|
| POST | `/details` | `type=FactSheet` (shared user-details write) | write | `UserDetailsController.go:107,141,369` |
| GET/POST | `otherdetail/*params` | `type=FactSheet` | read | `UserOtherDetailModel.go` (dedicated SELECT) |
| GET/POST | (combined full-profile call, `tsDataCall` param set) | — | read | `UserOtherDetailModel.go` — `fact_sheet_data` embedded in `json_agg` |

---

## 3. Data Model — Table

> **Verification note**: yeh sab table/column names Go code ke andar embedded SQL strings se
> liye gaye hain (har row ke saamne file path diya hai). Live DB schema se pgAdmin pe
> cross-verify **nahi** kiya gaya hai — kisi migration/integration se pehle woh zaroor karo.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_USR_FACT_SHEET` | meshpg (write) / read-replica-equivalent (read) | Supplier business-capability profile, **one row per user** (no history table, unlike GST-HSN) | `FACT_SHEET_ID` (PK, `RETURNING` on insert, **not used to target the UPDATE — see rule 1**), `FK_GLUSR_USR_ID`, `EXPORT_PERCENTAGE`, `DUN_BRANDSTREET_NUMBER`, `COFACE_NUMBER`, `AFTER_SALE_SUPPORT`, `SAMPLING_POLICY`, `SAMPLING_POLICY_PAID_VALUES`, `COMPETITIVE_ADVANTAGE`, `CONTRACT_MANUFACTURING`, `QUALITY_FACILITIES`, `CONSIGNMENTS_SPECIFICATION`, `TIME_OF_DELIVERY`, `PAYMENT_TERMS`, `OTHER_PAYMENT_TERMS`, `PAYMENT_MODE`, `SHIPING_MODE` (schema-typo, missing `P` — should be `SHIPPING_MODE`), `NUMBER_OF_CONTROL_STAFF`, `NUMBER_OF_ENGG`, `NUMBER_OF_SKILLED_STAFF`, `NUMBER_OF_SEMI_SKILLED_STAFF`, `NUMBER_OF_CONSULTANTS`, `GLUSR_USR_FACT_UPDATEDBY_FLAG`, `FACT_SHEET_UPDATEDBY`/`_UPDATEDBY_ID`/`_UPDATEDBY_AGENCY`/`_UPDATESCREEN`/`_IP`/`_IP_COUNTRY`/`_UPDATEDBY_URL`/`_HIST_COMMENTS` — [`UserDetailsModel.go:750-864`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go), read SELECT [`UserOtherDetailModel.go:228-267`](../internal/models/users/UserOtherDetailModel.go) |

**Field-count note**: 29 data/audit columns total are written on both INSERT and UPDATE
(matches the INSERT column list at [`UserDetailsModel.go:791`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)
and UPDATE `SET` list at [`UserDetailsModel.go:754-786`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)).

---

## 4. "Decode This": `id` Is a Flag, Not a Targeting Key

Unlike most branches of this shared controller, Fact Sheet's `id` parameter (mapped to
`FACT_SHEET_ID` in [`UsersValidationMaps.go:1664`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go))
is read directly from the caller at
[`UserDetailsModel.go:48`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)
(`id, _ := inputParams["id"].(string)`) — **not** derived from an internal DB lookup, contrary
to what a naive reading might suggest.

But here's the important part: the `UPDATE` statement's `WHERE` clause is
**`WHERE FK_GLUSR_USR_ID = $1`** only — it never references `id`/`FACT_SHEET_ID` at all
([`UserDetailsModel.go:785-786`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)).
So:

- `id != ""` → decides the branch takes the **UPDATE** path.
- `id == ""` → decides the branch takes the **INSERT** path.
- The actual **value** of `id` is never used to target a row — the UPDATE always targets
  "whichever single Fact-Sheet row belongs to this `FK_GLUSR_USR_ID`," because at most one
  exists per user. `id` is effectively **a boolean flag in string form**, not a foreign-key
  lookup value, despite being named/typed as `FACT_SHEET_ID`.

This is **not ambiguous or inferred** — it's directly visible in the SQL string — but it is
worth flagging because a caller passing a stale/wrong `FACT_SHEET_ID` as `id` (as long as
it's non-empty) would still silently update the correct (only) row for that user, masking a
potential caller-side bug.

---

## 5. Business Rules & Validation (code se exhaustive)

1. **No FactSheet-specific mandatory-field check.** Unlike Bank Details (which requires
   `ac_no`/`ifsc_code` on insert — see `Bank_Details_Technical_Doc.md` §5.2) or GST (15-char
   length + checksum), Fact Sheet only goes through the **generic** cross-type mandatory
   check (`type`/`glusrid`/`updatedby`/`VALIDATION_KEY`/`ip`/`ip country`/`updatescreen`/
   `updatedbyId`) at
   [`UserDetailsController.go:531-536`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDetailsController.go)
   — no individual Fact-Sheet business field (export%, staff counts, etc.) is required. A
   caller could technically insert an all-empty Fact-Sheet row.
2. **No FactSheet-specific gateway allowlist.** The generic `gatewayMap` (~30 gate values —
   `BUYERMY`, `GLADMIN`, `MAPI`, `SELLERMY`, `ERP_BT`, `ANDROID`, `BI`, etc.) applies; there is
   no extra `if Type == "FactSheet"` gate-restriction like Bank Details has (compare
   [`UserDetailsController.go:522-529`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDetailsController.go)
   which is BankDetails-only). Any gate on the generic allowlist can write Fact Sheet data.
3. **Insert-vs-update decided purely by whether `id` (as submitted by the caller) is empty**
   — see section 4. `id != "" → UPDATE`, `id == "" → INSERT`.
   [`UserDetailsModel.go:752,787`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)
4. **UPDATE is targeted by `FK_GLUSR_USR_ID`, not by `FACT_SHEET_ID`** — see section 4. This
   is only safe because of the one-row-per-user invariant; there is no DB-level unique
   constraint visible in this code path enforcing that invariant, only the application's
   insert/update branching logic.
5. **UPDATE is a full-record overwrite** — all 29 columns are set unconditionally; there's no
   dynamic/whitelist-driven partial-update logic here (unlike Bank Details' update path,
   which builds a dynamic `UPDATE` from a whitelist map). Every field must be resupplied on
   update or it gets overwritten with the caller's (possibly empty/nil) value.
   [`UserDetailsModel.go:754-786`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)
6. **Only length + numeric-type validation runs** (`LengthAndTypeValidations_v2` against
   `UserDetails_FactSheet` map) — checks (a) field is in the known-key map, (b) value doesn't
   exceed the declared `length`, (c) if `type=="number"`, value is numeric. No regex, no
   business-rule validation (e.g. `export_percentage` being 0-100, or staff counts being
   non-negative) exists anywhere in this branch.
   [`UsersValidationMaps.go:1663-1695,2449-2451`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go),
   [`globalfunctions.go:484-520`](../../service-api-go-production/service-api-go-production/pkg/utils/globalfunctions.go)
7. **`GLUSR_USR_FACT_SHEET` is part of the shared RabbitMQ-fan-out table-list**
   (`tabArr = ["GLUSR_USR_COMP_REGISTRATIONS", "GLUSR_BANK_DETAILS",
   "GLUSR_USR_COMP_FORM8", "GLUSR_USR_FACT_SHEET"]`) — a successful Fact-Sheet write
   triggers a **second** RabbitMQ publish beyond the standard one every write gets (see
   section 6 for the full two-publish mechanics), with `ACTION=UPDATE` always set on this
   second message regardless of whether it was actually an insert or update on this side.
   [`UserDetailsModel.go:4169-4189`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)
8. **Read-side query selects the single row directly** (`WHERE fk_glusr_usr_id=$1 LIMIT 1`) —
   no windowing/history-dedup needed since only one row exists per user.
   [`UserOtherDetailModel.go:228-267`](../internal/models/users/UserOtherDetailModel.go)
9. **Read-side response gets spurious empty fields injected** — the read code path shared
   between `CompRgst` and `FactSheet` (line 580-605) unconditionally sets `result["GST"]=""`
   if not already present, and unconditionally overwrites `result["UDYAM"]`/`result["AADHAR"]`
   via `ConvertStringtoSlice(FormatToString(result[...]))` even though the Fact-Sheet SELECT
   never selects `GST`, `UDYAM`, or `AADHAR` columns at all. Net effect: a `GET
   otherdetail?type=FactSheet` response includes `"GST": ""` and empty `"UDYAM"`/`"AADHAR"`
   array fields that have **no semantic meaning for Fact Sheet** — a byproduct of reusing
   CompRgst's post-processing code for FactSheet. Because `result["GST"] == ""`, the
   subsequent GST-verification-status curl-call block is correctly skipped (no extra
   latency), but the response shape itself is misleading to any client not expecting these
   fields. [`UserOtherDetailModel.go:580-609`](../internal/models/users/UserOtherDetailModel.go)
10. **Read-side is also exposed via the combined `json_agg` "everything" (`tsDataCall`) query**
    used for TrustSeal/full-profile fetches (`fact_sheet_data` sub-select alongside
    `comp_reg_data`, `oth_rem_dtl_data`) — same embedded-JSON-subquery pattern seen in
    GST/Bank-Details KTs.
    [`UserOtherDetailModel.go:507-511,986`](../internal/models/users/UserOtherDetailModel.go)
11. **A dead constant exists referencing this table**: `DetailsAPItab =
    []string{"GLUSR_USR_COMP_REGISTRATIONS", "GLUSR_BANK_DETAILS", "GLUSR_USR_FACT_SHEET"}`
    is declared in `commonMaps.go:114` but has **zero references** anywhere else in the
    `service-api-go-production` repo (confirmed via grep) — likely a superseded predecessor
    of the `tabArr` list actually used at `UserDetailsModel.go:4169` (which additionally
    includes `GLUSR_USR_COMP_FORM8`). Not a functional risk, but worth a cleanup flag.
    [`commonMaps.go:114`](../../service-api-go-production/service-api-go-production/pkg/components/commonMaps.go)

---

## 6. RabbitMQ

No Kafka, no Redis touch this table anywhere in the three repos (confirmed via grep across
`service-api-go-production`, `users-api-go-production`, `user-temp-consumers-production` —
see sections 7, 8). All messaging is RabbitMQ via the shared `PushToQueue` helper. Fact Sheet
is unusual among the branches documented in this KT series in that a single successful write
triggers **two separate publishes**, to two differently-named services:

| # | `SERVICENAME` | Routing key / exchange (`rabbitmq.go`) | Trigger | Consumer |
|---|---|---|---|---|
| 1 | `USER_DETAIL_SERVICE` (default for any type not `ContactDetails`/`BankDetails`/`Franchise`) | `comp.sync.<modulus>` on exchange `USER.topic` | Fires for **every** successful write on this shared endpoint, `Type=="FactSheet"` included — [`UserDetailsModel.go:4077-4123`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go) | Generic `comp.sync.*` sharded fan-out consumers (search/directory-replica sync — not Fact-Sheet-specific) |
| 2 | `USER_DETAILS_SERVICE` (**note: plural "DETAILS"**, a *different* SERVICENAME string from #1) | `user.additional.*` on exchange `USER.topic` | Fires **additionally**, only because `GLUSR_USR_FACT_SHEET` is in `tabArr` (section 5.7) | [`USER_UPSERT_ADDITIONAL.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_ADDITIONAL.go) (`ActionUserUpsertAdditional`) |

Evidence for the routing-key split:
[`rabbitmq.go:51-55,80-84`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go) —
`"USER_DETAIL_SERVICE": "comp.sync." + modulus` vs. `"USER_DETAILS_SERVICE": "user.additional.*"`,
both bound to exchange `USER.topic`.

**Important nuance on message #2's consumer**: `USER_UPSERT_ADDITIONAL.go` is a large,
generic consumer shared across many write-types (blacklist checks, big-buyer flagging,
Clearout email validation, PNS/mobile-change mail-SMS, disable-reason handling, etc.). It has
**no branch keyed on any Fact-Sheet column** (`EXPORT_PERCENTAGE`, `PAYMENT_TERMS`, etc.) —
its behavior is driven entirely by generic keys in `COLUMNS`/`OTHERS`
(`GLUSR_USR_EMAIL`, `GLUSR_USR_APPROV`, `Communication_Flag`, etc.) that a Fact-Sheet message
payload is unlikely to populate. **Practical effect**: message #2 for a Fact-Sheet write is
very likely a functional no-op in this consumer today — it's consumed, logged, acknowledged,
but doesn't trigger any Fact-Sheet-relevant side-effect. **[INFERRED — confirm with team]**:
this reading is based on the consumer having no FACT_SHEET-keyed branch; it wasn't proven by
tracing an actual message payload at runtime.

**Koi dedicated Fact-Sheet-specific consumer nahi mila** (no `FACT_SHEET`/`FactSheet` string
anywhere in `user-temp-consumers-production`, confirmed via grep) — unlike GST (5+ dedicated
consumers) or Bank Details (3 dedicated + 1 shared).

---

## 7. Kafka

**Koi Kafka usage nahi mila** anywhere touching `GLUSR_USR_FACT_SHEET` or the `FactSheet`
type string, across all three repos (grep for `GLUSR_USR_FACT_SHEET` combined with `Kafka`
returned zero hits).

---

## 8. Redis

**Koi Redis usage nahi mila** specific to Fact Sheet (grep for `FACT_SHEET` combined with
`redis`/`Redis` returned zero hits across all three repos). Reads go straight to Postgres —
no caching layer, same gap already documented for GST/Bank-Details reads.

---

## 9. End-to-End Technical Flows

### Flow A — Supplier submits Fact Sheet for the first time (insert)

```
Supplier (seller panel/app), any gate on the generic allowlist
    │
    ▼
[API — write]  POST /details  {type: "FactSheet", export_percentage, dun_brandstreet_number,
                coface_number, ..., number_of_consultants}   (id=="" or absent)
    │  UserDetailsController.go
    │  1. Generic gateway-allowlist check (no FactSheet-specific gate restriction)
    │  2. Generic mandatory-field check (type/glusrid/updatedby/VALIDATION_KEY/ip/...)
    │  3. LengthAndTypeValidations_v2 against UserDetails_FactSheet (length + numeric-type only)
    ▼
UserDetailsModel.go — Type=="FactSheet" branch, id==""
    ▼
[DB — meshpg]  INSERT INTO GLUSR_USR_FACT_SHEET (29 cols) VALUES (...) RETURNING FACT_SHEET_ID
    │
    ▼
[RabbitMQ publish #1]  SERVICENAME=USER_DETAIL_SERVICE → comp.sync.<modulus>
    │  generic search/directory-replica sync fan-out
    ▼
[RabbitMQ publish #2]  SERVICENAME=USER_DETAILS_SERVICE → user.additional.*
    │  (fires because GLUSR_USR_FACT_SHEET is in tabArr — section 5.7, section 6)
    ▼
[CONSUME]  USER_UPSERT_ADDITIONAL.go — no FACT_SHEET-keyed branch fires (section 6 nuance)
    ▼
Response — FACT_SHEET_ID returned
```

### Flow B — Supplier updates existing Fact Sheet

```
Supplier, id != "" (any non-empty value — not actually used to target the row, section 4)
    │
    ▼
[API — write]  POST /details  {type: "FactSheet", id: "<any-non-empty>", ...all 29 fields...}
    ▼
UserDetailsModel.go — Type=="FactSheet" branch, id!=""
    ▼
[DB — meshpg]  UPDATE GLUSR_USR_FACT_SHEET SET (all 29 cols) WHERE FK_GLUSR_USR_ID=$1
    │  (full-record overwrite — any field not resent gets nulled/overwritten)
    ▼
[RabbitMQ publish #1 + #2]  same as Flow A
    ▼
Response
```

### Flow C — Buyer/any caller reads Fact Sheet

```
Client
    │
    ▼
[API — read]  GET/POST otherdetail/*params  type=FactSheet
    │  UserOtherDetailModel.go
    ▼
[DB — read-DB]  SELECT (29 cols) FROM GLUSR_USR_FACT_SHEET WHERE fk_glusr_usr_id=$1 LIMIT 1
    │
    ▼
Shared CompRgst/FactSheet post-processing (section 5.9):
    │  result["GST"]="" injected if absent, UDYAM/AADHAR forced to [] — cosmetic/spurious
    │  for FactSheet, but result["GST"]=="" correctly short-circuits the GST-verification
    │  curl-call block (no wasted latency)
    ▼
Response — fact-sheet fields (+ spurious GST/UDYAM/AADHAR keys)
```

### Flow D — Full-profile / TrustSeal combined read

```
Client (tsDataCall param set)
    │
    ▼
[API — read]  UserOtherDetailModel.go — combined json_agg query
    │  (select json_agg(tab.*) from GLUSR_USR_FACT_SHEET tab where fk_glusr_usr_id=$1) fact_sheet_data
    │  alongside comp_reg_data, oth_rem_dtl_data, bank-details subquery, etc. — one round-trip
    ▼
Response — fact_sheet_data embedded as part of the larger combined payload
```

---

## 10. Flow-wise DB & Table Usage

### Flow A — Insert

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `GLUSR_USR_FACT_SHEET` | INSERT ... RETURNING FACT_SHEET_ID | Create the one-per-user Fact-Sheet record |

*(No pre-check SELECT before insert — unlike Bank Details' SYSTEM-reqType prime pre-check,
Fact Sheet's insert path does zero DB reads before writing.)*

### Flow B — Update

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `GLUSR_USR_FACT_SHEET` | UPDATE (all 29 cols) WHERE FK_GLUSR_USR_ID=$1 | Full-record overwrite, targeted by user not by the submitted `id` (section 4) |

*(No pre-fetch of old values either — unlike Bank Details' update path, which fetches
`oldBAcct`/`oldIfsc`/`wasPrime` first for change-detection. Fact Sheet has no downstream
consumer that needs an old-vs-new diff, so no pre-fetch is needed.)*

### Flow A/B — Downstream (RabbitMQ, no direct DB calls from the write-model itself)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| — | none | — | — | `USER_UPSERT_ADDITIONAL.go` consumes message #2 but (per section 6) triggers no
  FACT_SHEET-keyed DB write; message #1's generic `comp.sync.*` consumer is outside this
  branch's scope (shared with every other detail-type) |

### Flow C — Single-type read

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | read-DB | `GLUSR_USR_FACT_SHEET` | SELECT ... LIMIT 1 | Return the one Fact-Sheet row for this user |

### Flow D — Combined/TrustSeal read

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | read-DB | `GLUSR_USR_FACT_SHEET` (+ `GLUSR_USR_COMP_REGISTRATIONS`, `GLUSR_OTH_REM_DETAIL`, bank-details subquery, all in one query) | SELECT via `json_agg` sub-selects | Single round-trip fetches everything needed for a full-profile/TrustSeal view — an efficient pattern, no optimization needed here |

**Total DB round-trips for a single Fact-Sheet write event: exactly 1** (either the INSERT or
the UPDATE) — this is the leanest write path of any feature documented in this KT series so
far (GST: 9-10 round-trips across 4 DBs; Bank Details: up to 8 including external API hops).
The two RabbitMQ publishes add latency but not DB load on the write-API side itself.

---

## 11. Optimization Scope — DB / Response-Time Contributors

### Low impact (this branch is already lean)

1. **Single DB round-trip per write, no pre-check/pre-fetch queries** — Fact Sheet's write
   path is simpler and cheaper than GST or Bank Details specifically because there's no
   locking logic, no prime-account invariant, and no old-value diffing needed. No concrete
   optimization opportunity here; this is close to optimal already for a single-table
   insert-or-update.
2. **No caching on reads** (section 8) — same gap as GST/Bank-Details, but Fact-Sheet reads
   are likely lower-volume/lower-criticality than GST (not a trust/verification signal), so
   this is a lower priority than the equivalent GST gap.
3. **Second RabbitMQ publish (`USER_DETAILS_SERVICE` / `user.additional.*`) is likely a
   functional no-op for Fact Sheet** (section 6) — it still costs a full publish + consume +
   ack round-trip on every write, for a consumer that (per this pass's reading) does nothing
   Fact-Sheet-specific with it. If confirmed, removing `GLUSR_USR_FACT_SHEET` from `tabArr`
   (or adding an early-exit in `USER_UPSERT_ADDITIONAL.go` for this table) would save one
   RabbitMQ round-trip per Fact-Sheet write with no behavior change — **but confirm via
   message-payload tracing/logs before removing, in case some downstream generic check
   (blacklist, disable-reason, etc.) does legitimately need to run on Fact-Sheet-triggered
   messages too.**
4. **Read-side spurious-field injection (section 5.9)** is a correctness/API-contract
   cleanliness issue more than a performance one — negligible CPU cost, but worth fixing
   since it leaks CompRgst-shaped fields (`GST`, `UDYAM`, `AADHAR`) into FactSheet API
   responses where they don't belong.

### Shared with GST/Bank-Details KT

5. Any broader optimization findings for the shared `UserDetailsModel.go` controller's
   overall architecture are documented in
   [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) §12 and
   [`../Bank Details KT/Bank_Details_Technical_Doc.md`](../Bank%20Details%20KT/Bank_Details_Technical_Doc.md) §11,
   and apply equally where relevant (e.g. no caching on any of these read endpoints).

---

## 12. Cron Inventory

**No cron job touching `GLUSR_USR_FACT_SHEET` or the `FactSheet` type was found.** Searched
`user-temp-consumers-production`'s `internal/Workers` tree and any separate `crons/`-style
directory for `FACT_SHEET`/`FactSheet` — zero hits. Unlike GST (daily BigQuery-driven TACT
re-verification cron), Fact Sheet has no scheduled reconciliation job, which is consistent
with it having no verification-state concept to reconcile in the first place.

---

## 13. Edge Cases & Gotchas (technical POV)

1. **`id`/`FACT_SHEET_ID` is a boolean flag, not a targeting key** (section 4) — a caller
   passing any non-empty `id` value (even a wrong/stale one) still updates the correct row,
   because the UPDATE targets by `FK_GLUSR_USR_ID` only. This masks caller-side bugs that
   pass an incorrect `id`.
2. **Full-overwrite-on-update means partial-updates aren't supported** — a caller that only
   wants to change `EXPORT_PERCENTAGE` must resend the entire Fact-Sheet payload (all 29
   fields), or risk nulling out the rest.
3. **`SHIPING_MODE` column-name has a schema-level typo** (missing a `P`, should be
   `SHIPPING_MODE`) — cosmetic, but any future direct-SQL work on this table needs to match
   the existing (mis-)spelling.
4. **No mandatory-field or business-rule validation at all for Fact-Sheet-specific fields**
   (section 5.1, 5.6) — an all-empty Fact-Sheet row can be inserted; no range-check on
   `export_percentage`, no non-negative check on staff counts.
5. **No gateway restriction specific to Fact Sheet** (section 5.2) — any of the ~30 generic
   allowlisted gates can write Fact-Sheet data, a notably wider blast radius than Bank
   Details' 10-gate allowlist.
6. **Two RabbitMQ publishes per write, second one likely a no-op** (section 6, section 11.3)
   — costs latency without (per this pass's reading) any confirmed functional benefit for
   this branch specifically; needs runtime confirmation before treating as dead weight.
7. **Read responses leak CompRgst-shaped fields** (`GST`, `UDYAM`, `AADHAR` — section 5.9) —
   a byproduct of sharing post-processing code between `CompRgst` and `FactSheet` detail
   types; harmless functionally (correctly short-circuits the GST curl call) but confusing
   for any client inspecting the raw response shape.
8. **A dead constant (`DetailsAPItab`) exists that looks similar to the live `tabArr`**
   (section 5.11) — if someone edits `tabArr` (e.g. to add/remove a table from the RabbitMQ
   fan-out) they might mistakenly find and edit `DetailsAPItab` instead, since the two lists
   are easy to confuse and only one is live.

---

## 14. Open Questions

1. Does `USER_UPSERT_ADDITIONAL.go` (the consumer for the second, `user.additional.*`
   publish) actually do anything meaningful for Fact-Sheet-triggered messages in practice, or
   is it confirmed dead weight (section 6, 11.3)? This pass found no FACT_SHEET-keyed branch
   in that consumer, but didn't trace an actual live message payload.
2. Is `id`/`FACT_SHEET_ID` targeting-by-`FK_GLUSR_USR_ID`-only (section 4) an intentional
   design choice (because it's a one-row-per-user table, so it "doesn't matter"), or an
   oversight where the `id` parameter should have been used defensively in the `WHERE` clause
   too?
3. Is there any validation ensuring numeric-looking fields (`EXPORT_PERCENTAGE`, staff-counts)
   are actually sensible values (e.g. 0-100 for a percentage) anywhere upstream of this API
   (client-side/UI validation), given the API itself does none?
4. `DUN_BRANDSTREET_NUMBER`/`COFACE_NUMBER` — are these validated against the actual Dun &
   Bradstreet/Coface external registries anywhere in the platform, or just stored as free
   text end-to-end?
5. Is `DetailsAPItab` (section 5.11) truly dead code, or is it referenced via reflection/some
   indirect mechanism not caught by a literal-string grep?
6. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai (column existence, types, nullability, indexes, constraints not
   cross-checked against a live schema).

---

## See also

- [`Fact_Sheet_Business_Doc.md`](./Fact_Sheet_Business_Doc.md) — product perspective
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — the shared
  write/read controller this feature is a branch of; gateway/dispatch architecture
  documented there in full
- [`../Bank Details KT/Bank_Details_Technical_Doc.md`](../Bank%20Details%20KT/Bank_Details_Technical_Doc.md) —
  another branch of the same shared controller, useful contrast (Bank Details has 4
  dedicated consumers + external verification hop; Fact Sheet has none)
- [`../utils_programming_guide.md`](../utils_programming_guide.md) — shared
  `PushToQueue`/`PubAPI` helpers referenced in this doc

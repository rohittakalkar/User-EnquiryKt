# User Add / Update (Core Registration & Profile) — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`User_Add_Update_Business_Doc.md`](./User_Add_Update_Business_Doc.md)
dekho. Dono docs same flows cover karte hain, alag audience ke liye.

**Scope note**: Yeh users-domain ka **sabse bada aur sabse-heavily-integrated feature** hai —
`UserAddController.go`/`UserUpdateController.go` aur unke models (1009/1498 lines each) poora
core-registration-engine hain. Is doc mein GST-doc jaisi depth try ki gayi hai — sab claims
file:line cited hain, jahan conclusively confirm nahi ho paya wahan **[INFERRED]** ya Open
Questions mein flag kiya gaya hai.

**Repos**: `service-api-go-production` (write — Add/Update), `users-api-go-production` (read —
`otherdetail`/profile-detail queries), `user-temp-consumers-production` (largest
consumer-fan-out in the entire codebase).

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Add (registration) controller | write | [`UserAddController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserAddController.go) (248 lines) |
| Add (registration) model | write | [`UserAddModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddModel.go) (1009 lines — `InsertUser`, `InsertGlusrMeshPg`, `InsertAuthPG`, `PrepareInputParamsForInsert`, `CallDetailsService`) |
| Update (profile-edit) controller | write | [`UserUpdateController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserUpdateController.go) (561 lines) |
| Update (profile-edit) model | write | [`UserUpdateModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUpdateModel.go) (1498 lines — `UpdateGlusrMeshPg`, `UpdateAuthPG`, `Trigger`, per-attribute TrustSeal-unverification logic) |
| Shared allowlists/config/column-maps | write | [`UserAddUpdateUtils.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddUpdateUtils.go) (3404 lines — `UserAddModids`, `UserUpdateModids`, `TsNoUpdateVerify`, `TsVerifyMap`, `stopUnverification`, `RTF_master_arr`, `UserMapOracle`/`UserMap`/`UserMapAuthPG` column-maps, `CompareMap`, `LocationAttributes`) |
| Read (profile-detail, in `users-api-go-production` itself) | read | `UserDetailModel.go`, `UserOtherDetailModel.go` — same "big combined query" files referenced across GST, Fact Sheet, Social Contacts KT folders |
| Misc user-domain crons (not Add/Update-specific) | write repo `crons/users/` | [`OldGsmNumberTTL.go`](../../service-api-go-production/service-api-go-production/crons/users/OldGsmNumberTTL.go), [`VerificationSourceUpdate.go`](../../service-api-go-production/service-api-go-production/crons/users/VerificationSourceUpdate.go) — dekho section 13 |

---

## 2. Routes

| Method | serviceName | Repo | Controller |
|---|---|---|---|
| POST | `GLUSR_INSERT_SERVICE` | write | `UserAddController` — [`UserAddController.go:19,29`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserAddController.go) |
| POST | `GLUSR_UPDATE_SERVICE` | write | `UserUpdateController` |

Both routes are dispatched via the shared `serviceName`-based gateway router (consistent with
every other feature in this KT series — see `utils_programming_guide.md`), not REST-path
routing.

---

## 3. Data Model — Tables

> **Verification note**: table/column names Go code ke andar embedded SQL strings se liye
> gaye hain. Live DB schema se cross-verify **nahi** kiya gaya hai.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_USR` | meshpg | **The master user/company record** — every other feature in this KT series references `FK_GLUSR_USR_ID` back to this table | `GLUSR_USR_ID` (PK, `RETURNING` on insert), `GLUSR_USR_DISPLAY_ID`, plus ~100+ columns covering name/mobile/email/address/company/approval-status/legal-status/showroom/social-contact fields — full map in `UserMapOracle`/`UserMap` [`UserAddUpdateUtils.go:141-407`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddUpdateUtils.go). Column-set is built **at runtime from whichever keys are present in `inputParams`**, not a fixed literal list. |
| (auth-credentials table, exact name not confirmed) | authpg | Login-credentials, created/updated in lockstep with `GLUSR_USR` writes | `InsertAuthPG()` [`UserAddModel.go:136-141`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddModel.go); `UpdateAuthPG()` [`UserUpdateModel.go:~1420`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUpdateModel.go) via `UserMapAuthPG`/`UserUpdateMapAuthPG` column-subsets [`UserAddUpdateUtils.go:251-284`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddUpdateUtils.go) (only FIRSTNAME/LASTNAME/country-iso/phone/email/pass/approv/custtype/city-id/mobile-type — a much smaller field-set than the mesh side) |

**Insert uses `ON CONFLICT DO NOTHING`**: `INSERT INTO GLUSR_USR (%s) VALUES (%s) ON CONFLICT
DO NOTHING RETURNING GLUSR_USR_ID, GLUSR_USR_DISPLAY_ID` —
[`UserAddModel.go:776`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddModel.go).
Combined with the application-level duplicate-pre-check (mobile/email/GSM, section 4), this
gives a **two-layer duplicate-defense**: app-level lookup first, DB-level constraint as a
backstop for race conditions.

**Update is a fully dynamic partial statement**: `UPDATE GLUSR_USR SET %s WHERE
GLUSR_USR_ID=$%d` [`UserUpdateModel.go:855`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUpdateModel.go)
— column-list built purely from the keys present in the caller's request.

---

## 4. Duplicate-Detection & TrustSeal-Unverification "magic value" Decoding

### 4a. Add-flow duplicate signals (`m1_glid`/`e1_glid`/`g1_glid`)

`CheckUniqueMesh()` (called from `InsertUser`,
[`UserAddModel.go:45`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddModel.go))
returns a `uniq` map with keys `M1` (mobile-match GLID), `E1` (email-match GLID), `G1`
(GSM/business-registration-match GLID). `InsertUser` then classifies into exactly **6
outcome-branches** based on which of `m_glid`/`e_glid`/`g1_glid` are non-nil
[`UserAddModel.go:167-660`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddModel.go):

| Branch | Condition | Behavior |
|---|---|---|
| 1 | `GST` present in request AND any duplicate found | An extra `CallDetailsService()` HTTP-loopback fires first, using `e_glid` or `m_glid` (mobile wins) as the target GLID — [`UserAddModel.go:168-181`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddModel.go) |
| 2 | `usertype=="0"` (no email-match) AND (mobile-match OR GSM-match) | GSM-match is treated as equivalent to a mobile-match (`m_glid = g1_glid`); an internal-update `MakeCurlCall` is issued against `m_glid`, filling in ONLY fields that are currently empty on the existing record (fetched via `FindAlternateCass`) — existing non-empty values are never overwritten. If the new EMAIL conflicts with an existing non-empty email, the whole request is rejected with `"Primary Email id already exist"` [`UserAddModel.go:200-339`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddModel.go) |
| 3 | `usertype=="1"` (email-match) AND no mobile/GSM-match | Same gap-fill internal-update pattern, targeted at `e_glid` [`UserAddModel.go:340-462`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddModel.go) |
| 4 | `e_glid == m_glid` (same account matched by both signals) | Single gap-fill update against that one account (only FIRSTNAME/LASTNAME bind) [`UserAddModel.go:487-521`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddModel.go) |
| 5 | Mobile-match and email-match point to **different** accounts, and the mobile-matched account has no email on file | Email-conflict rejection path against the mobile-account [`UserAddModel.go:522-566`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddModel.go) |
| 6 | Mobile-match and email-match point to different accounts, both have data | Cross-merge: mobile-account gets an alt-mobile bind from the email-account's mobile if different; email-account gets its own gap-fill [`UserAddModel.go:567-660`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddModel.go) |

**None of these 6 branches insert a new `GLUSR_USR` row** — every duplicate-signal outcome
resolves to an update against an existing account via an internal HTTP loopback
(`MakeCurlCall`), never a direct second write path. Only the **zero-duplicate** case falls
through to `PrepareInputParamsForInsert` → `InsertGlusrMeshPg` (the true-insert path).

### 4b. Force-set fields on true insert

`APPROV="A"` (auto-approved), `LASTMODIFIED`/`LASTLOGIN`/`MEMBERSINCE="SYSDATE"`,
`PAID_SERV=0`, `LOC_PREF=4`, `FCP_FLAG=0` —
[`UserAddModel.go:104-114`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddModel.go).
New accounts are immediately active; no pending-approval gate exists at this layer.

### 4c. DB-constraint-name → friendly-error mapping

`EMAIL_UQ`/`EMAIL_DUP_UQ` → "Duplicate" email, `MOBCNTR_IDX` → "Duplicate" mobile,
`SYS_C00124106` → "Duplicate" PNS —
[`UserAddModel.go:151-160`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddModel.go).
Defense-in-depth against race conditions between the app-level pre-check and the actual
insert.

### 4d. Update-flow: per-attribute TrustSeal-unverification (`AttributeMap` / `att_id`)

This is the **most important decode in the Update flow** and was only partially traced in the
prior pass. `UpdateGlusrMeshPg` holds a literal `AttributeMap` mapping ~25 field-names to
`{mesh-column, numeric-attribute-id}` pairs (e.g. `COMPANYNAME→(GLUSR_USR_COMPANYNAME, "?")`,
`CITY→(GLUSR_USR_CITY, "114")`, `PH_MOBILE→(GLUSR_USR_PH_MOBILE, "121")`, `EMAIL_ALT→(...,
"157")`, `FK_GL_LEGAL_STATUS_ID→(...,"186")`) —
[`UserUpdateModel.go:~700-768`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUpdateModel.go).
For each incoming field present in `AttributeMap`:

1. If the field is in `TsVerifyMap` (a 26-field allowlist: COMPANYNAME, ADD1/2, CITY,
   CITY_OTHERS, STATE, STATE_OTHERS, ZIP, YEAR_OF_ESTB, TRADEMEMBERSHIP,
   FK_GL_LEGAL_STATUS_ID, COUNTRYNAME, FIRSTNAME, LASTNAME, PH2_NUMBER/AREA, PH_NUMBER,
   PH_COUNTRY, PH_AREA, FAX_NUMBER/COUNTRY/AREA, DESIGNATION, FK_GLUSR_TURNOVER_ID,
   FK_GLUSR_NOOF_EMP_ID, BIN, COMP_BRANCH —
   [`UserAddUpdateUtils.go:462`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddUpdateUtils.go))
   AND the new value differs from the old (after optional special-char-strip/lowercase
   normalization via `CompareMap` [`UserAddUpdateUtils.go:448-458`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddUpdateUtils.go)),
   its numeric attribute-id is appended to `att_id` — this is the payload that flags TrustSeal
   to treat that specific attribute as stale/unverified.
2. **Exception**: if `UPDATEDUSING == "TRUSTSEAL"`, the check only fires when the TrustSeal
   caller is *clearing* the value (`val == ""`) — i.e. TrustSeal itself writing a verified
   value does not self-trigger unverification of that same field
   [`UserUpdateModel.go:772`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUpdateModel.go).
3. If the changed attribute-id is in `LocationAttributes` (`114,115,116,117,47,152,153` —
   city/state/country/locality ids), a separate `location_unverification` flag is set, and
   `Latlong_attributes` (`2072,2073`) are appended to `att_id` too
   [`UserUpdateModel.go:790,890-892`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUpdateModel.go).
4. If the changed attribute-id is in `stopUnverification` (`120,156,121,48,157,109` — phone
   area/number, alt-phone, mobile-alt, email-alt ids), the **old and new values are packed
   into `unverifiedAttributeKeys`/`oldValues` strings** (`##<attrid>:<value>` format) sent
   downstream — a richer payload than a plain attribute-id list, presumably so the consumer
   can show "was X, now Y" [`UserUpdateModel.go:793-818`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUpdateModel.go).

5. **Separately**, `TsNoUpdateVerify` (14 fields: ADD1, ADD2, CITY, CITY_OTHERS, STATE,
   STATE_OTHERS, ZIP, COUNTRYNAME, PH_MOBILE, PH_MOBILE_ALT, EMAIL, EMAIL_ALT,
   FK_GL_CITY_ID, FK_GL_STATE_ID) is a **hard block-list**: when `UPDATEDUSING == "TRUSTSEAL"`,
   any of these keys are deleted from `inputParams` entirely before the update proceeds —
   [`UserUpdateModel.go:449-456`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUpdateModel.go).
   Confirmed (not just inferred, as the prior pass flagged): **TrustSeal cannot change
   address/city/state/zip/country/mobile/email fields via a normal TrustSeal-sourced update
   call at all** — these are excised from the request before any SQL runs.

**Net effect**: TrustSeal-verification is tracked **per-attribute** (not a single
account-wide flag) — a granular unverification model, confirmed end-to-end from the code
(not previously traced this deep).

### 4e. Cron-vs-normal update routing (resolves prior Open Question #3)

```go
if len(inputParams) != 0 && inputParams["UPDATEDBY"] == "GLUSR Manual cron" {
    rabbitmqdata["SERVICENAME"] = "GLUSR_UPDATE_CRON_SERVICE"
} else {
    rabbitmqdata["SERVICENAME"] = "GLUSR_UPDATE_SERVICE"
}
```
[`UserUpdateModel.go:1246-1250`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUpdateModel.go).
**Confirmed, not inferred**: routing is purely a string-literal check on the `UPDATEDBY`
field — no `modid`/gateway-based branching. Any caller that sets `UPDATEDBY=GLUSR Manual
cron` gets routed to the cron-lane, regardless of which app/modid it came from.

### 4f. Banned-content-detection payload

`glusrBannedContent` is populated whenever ADD1/ADD2/FIRSTNAME/LASTNAME/CFIRSTNAME/
CLASTNAME/COMPANYNAME/DESIGNATION/LANDMARK/LOCALITY/SELLINTEREST/COMPANY_DESC change
[`UserUpdateModel.go:1252-1287`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUpdateModel.go)
— strongly suggests a downstream banned-content/abuse-text scan consumes these free-text
fields whenever changed. Exact consumer not traced in this pass (see Open Questions).

### 4g. RTF ("Real Time Feed") master snapshot

When `UpdateGlobal["IS_RTF"] == 1`, the **old** values of a 13-field `RTF_master_arr`
(EMAIL, ADD1/2, URL, PH_MOBILE, PH_MOBILE_ALT, EMAIL_ALT, COMPANYNAME, PH_AREA, PH_NUMBER,
PH2_AREA/NUMBER, IM_GSM) are packed into `others_keys_glusr["RTF_MASTER"]` before publish —
[`UserUpdateModel.go:1289-1298`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUpdateModel.go).
This is the same RTF mechanism referenced in the GST Technical Doc (`GstRtfRequest`,
authPg upsert) — confirms RTF is a shared cross-domain sync framework, not GST-specific.

---

## 5. Business Rules & Validation (code se)

### Add (registration)

1. **Gateway allowlist (`UserAddModids`) — 42 entries**: `MY`, `GLADMIN`, `TOLLFREE`, `Email
   Marketing`, `HTVENDOR`, `Weberp`, `LEAP`, `IPC-Admin`, `NSD Sales`, `MAPI`, `ENQUIRY`,
   `FREE-WEBSITE`, `M.INDIAMART.COM`, `TRADE`, `BL`, `TENDER`, `PAYNOW`, `CREDIT ALLOCATION`,
   `OVP Process`, `SAMPARK Process`, `TOLLFREE Process`, `VENDOR CITY Pin Correction`,
   `Notification Server`, `search`, `FCP`, `PNS`, `Merp`, `IMOB`, `SELLERMY`, `PAYWIM`,
   `BUYERS_FEEDBACK`, `FLPNS`, `LEAPIN`, `User_Enrichment`, `IMLOGI`, `EXTLEAD`, `WHATSAPP`,
   `WA_9696`, `NSDMERP`, `EXPORT`, `FLIPS`, `PICSRCH`.
   [`UserAddUpdateUtils.go:423`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddUpdateUtils.go)
2. **Duplicate-detection runs BEFORE insert on 3 identity-signals** (mobile/email/GSM) with a
   **6-branch outcome resolver** that only ever redirects to internal-update, never a
   secondary insert — see section 4a for full decode.
3. **Force-set fields on true-insert path** — section 4b.
4. **DB-level constraint-violations pattern-matched to friendly duplicate errors** — section 4c.
5. **Successful `GLUSR_USR` insert immediately triggers `InsertAuthPG`** — a second,
   sequential write into `authpg` for login-credentials; if this second write fails, the
   function still returns based on the mesh-insert's success — the auth output is
   concatenated into `REASON` (`output + "," + insertPGOutput`) but doesn't roll back the
   mesh-insert [`UserAddModel.go:148`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddModel.go)
   (see Open Questions).
6. **If `GST` is present on an add-request that resolves to a duplicate**, an extra
   `CallDetailsService` HTTP-loopback fires (branch 1, section 4a) — an HTTP-loopback pattern
   consistent with Rating Usefulness, Trust Verification, Update With OTP documented
   elsewhere in this series.
7. **`MandatoryFieldsValidations`/`MandatoryFieldsValidationsExtended`** run before the
   duplicate-check — field-presence/format validation happens first, so a malformed request
   never reaches the duplicate-lookup at all
   [`UserAddController.go:148-162`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserAddController.go).
8. **`SessionKeyValidation` vs `MaskedMobileEmailValidation`** — if a numeric `USR_ID` is
   supplied with a `SESSION_KEY`, the request is treated as a session-authenticated update
   attempt through the Add endpoint (masked-mobile/email flow); without a session-key it's
   validated as a masked-mobile/email lookup instead
   [`UserAddController.go:128-144`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserAddController.go) — **[INFERRED — exact business purpose of this masked-mobile/email sub-flow through the Add endpoint not fully traced; confirm with team]**.

### Update (profile-edit)

9. **Gateway allowlist (`UserUpdateModids`) is nearly identical — 43 entries**, adding `BI`
   and `NSD` vs the Add-list.
   [`UserAddUpdateUtils.go:425`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddUpdateUtils.go)
10. **Update is genuinely dynamic/partial**: `UPDATE GLUSR_USR SET %s WHERE
    GLUSR_USR_ID=$%d` — column-list built at runtime from whichever fields the caller sent.
11. **Per-attribute TrustSeal-unverification decode** — the richest business-rule mechanism
    in this feature, fully decoded in section 4d.
12. **`TsNoUpdateVerify` hard-blocks TrustSeal from touching identity/address fields** — section
    4d point 5, confirmed not just inferred.
13. **`checkArr` gate before `Trigger()` runs** — the update only invokes the (presumably
    expensive) `Trigger()` validation/side-effect function
    [`UserUpdateModel.go:458-471`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUpdateModel.go)
    if the request touches at least one of ~50 specific fields (city/company/address/
    approval/listing-status/etc.) — a genuine short-circuit optimization for lightweight
    updates (e.g. a pure showroom-URL-only update skips `Trigger()` entirely).
14. **Two RabbitMQ service-names used depending on `UPDATEDBY` literal string** —
    `GLUSR_UPDATE_CRON_SERVICE` vs `GLUSR_UPDATE_SERVICE` — fully decoded in section 4e.
15. **A second, separate `authpg` UPDATE exists** (`UpdateAuthPG`, same
    `UPDATE GLUSR_USR SET %s WHERE GLUSR_USR_ID=$%d` shape, different DB) — confirms
    update-flow also keeps the two databases (mesh-profile + auth-credentials) in sync.
16. **`LASTLOGIN` is only refreshed if `UPDATEDBY == "USER"`** (case-insensitive) —
    [`UserUpdateModel.go:488-490`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUpdateModel.go)
    — admin/system-driven updates do not touch the login-timestamp, only genuine
    self-service user updates do.
17. **Banned-content-detection payload built for free-text field changes** — section 4f.
18. **RTF master-snapshot built when `IS_RTF` flag is set** — section 4g.

---

## 6. RabbitMQ

| Queue / `SERVICENAME` | Publisher | Consumer(s) | Purpose |
|---|---|---|---|
| `GLUSR_INSERT_SERVICE` | `InsertGlusrMeshPg` via `RabbitmqCentraliseInsertion` [`UserAddModel.go:839`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddModel.go) | Multiple `USER_UPSERT_*` consumers (below) | New-account fan-out |
| `GLUSR_UPDATE_SERVICE` | `UpdateGlusrMeshPg` (normal path, `UPDATEDBY != "GLUSR Manual cron"`) via `utils.PushToQueue` [`UserUpdateModel.go:1249,1350`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUpdateModel.go) | `dbActionUserCentralizedQueue` (via `USER_CENTRALIZED_QUEUE` — see below) | Profile-update fan-out |
| `GLUSR_UPDATE_CRON_SERVICE` | `UpdateGlusrMeshPg` (`UPDATEDBY == "GLUSR Manual cron"` exactly) | *(routes into the same downstream-consumer family per naming; distinct queue lets consumers treat bulk/automated updates differently)* | Distinguishes automated bulk-updates from user-driven ones |

**This is the largest downstream-consumer-fan-out found anywhere in this codebase.**
Consumers registered against this event-family
([`IntializeMsgBroker.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go),
confirmed by grep) include:

| Queue | Handler function |
|---|---|
| `USER_UPSERT_VERIFICATION` | `insertInQueueUserUpsertVerification` |
| `USER_UPSERT_ALERTPG` / `_BULK` | `channelConsumesData` |
| `USER_UPSERT_BLDISPLAY` / `_BULK` | `channelConsumesData` |
| `USER_UPSERT_TRUSTPG` / `_BULK` | `channelConsumesData` |
| `USER_UPSERT_ENQUIRYPG` / `_BULK` | `channelConsumesData` |
| `USER_UPSERT_LMSPG_FEDRATED` / `_BULK_FEDRATED` | `channelConsumesData` |
| `USER_UPSERT_CSL` / `_BULK` | `channelConsumesDataUserUpsertCSL` |
| `USER_CENTRALIZED_QUEUE` | `dbActionUserCentralizedQueue` |
| `USER_CENTRALIZED_QUEUE_FAIL` | `dbActionUserCentralizedQueueFail` |
| `USER_UPSERT_ADDITIONAL_GCP_IN` | `ActionUserUpsertAdditionalGcpIn` |
| `USER_UPSERT_ADDITIONAL` | `ActionUserUpsertAdditional` |
| `USER_UPSERT_CITY_OTHERS` | `dbActionUserUpsertCityOthers` |

`USER_CENTRALIZED_BANNED` (referenced in the prior doc pass and in the business-doc's
banned-content-detection flow) was **not found by exact-string grep in
`IntializeMsgBroker.go` in this pass** — it may be handled inside `dbActionUserCentralizedQueue`
itself as a routing branch rather than a separately-registered queue name. Flagged as an Open
Question rather than re-asserted as fact.

This confirms `GLUSR_USR` add/update is genuinely the **central nervous-system event** of the
whole platform — nearly every downstream domain (verification, alert, blacklist-display,
trust, enquiry, LMS, CSL/search) maintains its own synced copy fed from this single
event-family.

**Koi Kafka usage nahi mila directly in this write-path** (confirmed by grep — see section 7).

---

## 7. Kafka

**Koi Kafka usage nahi mila** in `UserAddModel.go` or `UserUpdateModel.go` — confirmed by
grep for `Kafka`/`InitializeKafka` across `internal/models/UserModels/`; the only hit in that
directory is `BsMatchMakingModel.go` (a different feature). Add/Update is **100% RabbitMQ**
for its own write-path fan-out.

---

## 8. Redis

**Koi Redis usage nahi mila** in `UserAddModel.go` or `UserUpdateModel.go` (confirmed by
grep — zero matches for `Redis`/`redis` in both files). No caching layer sits in front of the
duplicate-detection lookups (`CheckUniqueMesh`) or the update path — every Add/Update request
hits Postgres live for its uniqueness checks and writes.

---

## 9. End-to-End Technical Flows

### Flow A — Add (registration), zero-duplicate path

```
New user
    │
    ▼
[API — write]  POST serviceName=GLUSR_INSERT_SERVICE  {name, mobile, email, address,
                company-details, GST?, ...}
    │  UserAddController.go — mandatory-field validation first
    ▼
InsertUser() — SetMandatoryFieldsForInsert, NumberIsNotGsmCheck, ModIDValidation
    ▼
CheckUniqueMesh() — 3-signal lookup: mobile(M1) / email(E1) / GSM(G1)
    │
    └─ no match on any signal →
          PrepareInputParamsForInsert() — trigger-level checks
          force-set APPROV="A", LASTMODIFIED/LASTLOGIN/MEMBERSINCE=SYSDATE,
                     PAID_SERV=0, LOC_PREF=4, FCP_FLAG=0
          InsertGlusrMeshPg()
              INSERT INTO GLUSR_USR (...) ON CONFLICT DO NOTHING RETURNING ID
              ├─ constraint-violation → pattern-matched friendly duplicate-error
              └─ success →
                    InsertAuthPG()  [DB — authpg, sequential second write]
                    ▼
                    [RabbitMQ]  SERVICENAME=GLUSR_INSERT_SERVICE
                    ▼
                    [CONSUME — large fan-out]  USER_UPSERT_* / USER_CENTRALIZED_*
                                                (verification, alertPg, trustPg,
                                                 enquiryPg, LMS, CSL/search...)
```

### Flow B — Add (registration), duplicate-match path (6-branch resolver)

```
New user submits data that matches an existing account (mobile and/or email and/or GSM)
    │
    ▼
CheckUniqueMesh() → M1/E1/G1 non-empty
    │
    ▼
[if GST present] → CallDetailsService() HTTP-loopback (extra branch)
    ▼
Classify into one of 6 branches (section 4a) based on which signals matched and whether
they point to the same or different accounts
    │
    ▼
FindAlternateCass() — fetch existing account's current field values [DB read]
    ▼
Build `bind` map — ONLY fields currently empty on the existing account
    ▼
[HTTP loopback]  MakeCurlCall() → internally re-invokes the Update flow (Flow C) against
                 the matched GLID — no second INSERT ever happens
    ▼
Response reflects "Input data not unique" + the internal-update's result
```

### Flow C — Update (profile-edit)

```
Existing user
    │
    ▼
[API — write]  POST serviceName=GLUSR_UPDATE_SERVICE  {USR_ID, <any subset of fields>}
    │  UserUpdateController.go — Gateway(UserUpdateModids)
    ▼
Fetch old row (for compare) — `old` map used throughout
    ▼
[if UPDATEDUSING=="TRUSTSEAL"] → strip TsNoUpdateVerify fields from inputParams entirely
    ▼
checkArr gate — only invoke Trigger() if request touches one of ~50 gating fields
    ▼
Dynamic partial UPDATE
    │  UPDATE GLUSR_USR SET <only-supplied-columns> WHERE GLUSR_USR_ID=$N
    │  In parallel: per-attribute AttributeMap/TsVerifyMap diff builds att_id +
    │  unverifiedAttributeKeys/oldValues (section 4d) for TrustSeal-unverification signal
    │  banned-content payload built for free-text field changes (section 4f)
    │  RTF master-snapshot built if IS_RTF flag set (section 4g)
    ▼
[DB — meshpg]  on success →
    UpdateAuthPG()  [DB — authpg, second sequential write]
    ▼
[RabbitMQ]  UPDATEDBY=="GLUSR Manual cron" → GLUSR_UPDATE_CRON_SERVICE
            otherwise                      → GLUSR_UPDATE_SERVICE → USER_CENTRALIZED_QUEUE
    ▼
[CONSUME]  dbActionUserCentralizedQueue → fans out further downstream (verification,
           trust, alert, banned-content, LMS, CSL/search, enquiry...)
```

---

## 10. Flow-wise DB & Table Usage

### Flow A — Add, zero-duplicate (new account)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `GLUSR_USR` (via `CheckUniqueMesh`) | SELECT ×3 (mobile/email/GSM lookups — exact query shape, single-vs-multi-query, not fully confirmed) | Duplicate pre-check |
| 2 | meshpg | (trigger-level tables, via `PrepareInputParamsForInsert`) | SELECT | City/state/legal-status/etc. lookup + validation before insert |
| 3 | meshpg | `GLUSR_USR` | INSERT ... ON CONFLICT DO NOTHING RETURNING ID | The actual account-creation write |
| 4 | authpg | (auth-credentials table) | INSERT (`InsertAuthPG`) | Login-credentials for the new account — sequential second write |

### Flow B — Add, duplicate-match (internal-update redirect)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `GLUSR_USR` | SELECT ×3 | Duplicate pre-check (same as Flow A) |
| 2 | meshpg | `GLUSR_USR` (via `FindAlternateCass`) | SELECT | Fetch existing matched account's current field values, to know what's empty/fillable |
| 3 | *(internal HTTP loopback, not a direct DB call from this function)* | — | `MakeCurlCall` → re-enters the Update-flow's own DB operations (Flow C's table-list applies) | Gap-fill update against the existing account |
| 4 | *(conditional, if GST present)* meshpg/other (via `CallDetailsService`) | Unknown — not traced | HTTP call | GST-detail sync trigger |

### Flow C — Update (profile-edit)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `GLUSR_USR` | SELECT (`old` row fetch) | Baseline for diff — every attribute-change/TrustSeal-unverification decision depends on this |
| 2 | meshpg | (city/state/trigger-lookup tables, via `Trigger()`) | SELECT — conditional, only if `checkArr` intersection found | Validates city/state/legal-status/etc. changes before the update commits |
| 3 | meshpg | `GLUSR_USR` | UPDATE (dynamic column-set) | The actual partial-update write |
| 4 | authpg | (auth-credentials table) | UPDATE (`UpdateAuthPG`, dynamic column-set from `UserUpdateMapAuthPG`) | Keeps login-credentials mirror in sync — sequential second write |

**Total DB round-trips per Add (zero-dup): at least 4, across 2 physical databases
(meshpg, authpg). Per Update: at least 3-4, across the same 2 databases.** Both flows are
notably leaner than GST's 9-10-round-trip, 4-database chain — Add/Update touches fewer
physical databases directly (the fan-out to trust/alert/LMS/CSL/enquiry-pg happens entirely
downstream, async, in the RabbitMQ consumers, not synchronously in the write-path itself).

---

## 11. Optimization Scope — DB Response-Time Contribution

### High-impact

1. **Duplicate-match resolution (Flow B) round-trips through an internal HTTP loopback
   (`MakeCurlCall`) instead of calling the Update-model's DB functions directly in-process.**
   Every duplicate-detected Add-request pays for a full HTTP request/response cycle
   (serialize → local network hop → deserialize → re-authenticate/re-validate on the
   receiving side) purely to reach code that's already loaded in the same binary. This is
   the same anti-pattern flagged in the GST doc for `CallDetailsService`/OTP/Rating-Usefulness
   HTTP-loopbacks — a very common pattern in this codebase, and one of the highest-leverage
   fixes available (an in-process function call instead of `MakeCurlCall` would eliminate a
   full network round-trip on every duplicate-detected registration attempt, which given
   duplicate-detection is a core, frequently-hit path, is likely non-trivial traffic).
   [`UserAddModel.go:317,447,506,552,599,641`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddModel.go)

2. **`InsertAuthPG`/`UpdateAuthPG` are fully sequential second writes after the mesh write
   succeeds** — a genuine 2-round-trip cost, across 2 separate physical database instances,
   on every Add and every Update. Given both writes must logically succeed together (a user
   without auth-credentials is unusable), this is hard to avoid without cross-database
   transaction support, which Postgres doesn't offer across separate instances. Unlike GST's
   3-way-parallel-goroutine pattern, this appears to be strictly sequential — **[INFERRED —
   confirm whether InsertAuthPG/UpdateAuthPG could be fired in a goroutine in parallel with
   something else, or whether the sequential dependency is unavoidable given `USR_ID` must be
   known first]**.

### Medium-impact

3. **`CheckUniqueMesh`'s 3-signal duplicate lookup (mobile/email/GSM)** — not fully confirmed
   in this pass whether it's 1 combined `OR`-query or 3 separate SELECTs (function body itself
   not read in full in this pass). If separate, combining into one query would cut round-trips
   on the hottest path in the whole feature (every single Add-request hits this).
4. **`checkArr` short-circuit before `Trigger()`** is already a positive-pattern — a partial
   update touching only e.g. a showroom-URL field skips the (likely DB-heavy) `Trigger()`
   validation function entirely. Worth calling out as a good pattern already in place, not a
   gap.
5. **No caching anywhere** (section 8) — `GLUSR_USR` duplicate-lookups (mobile/email/GSM) are
   prime caching candidates given how frequently the same identity-signals get checked across
   near-simultaneous requests (e.g. a user double-submitting a signup form), though the
   correctness bar here is higher than GST's (a stale cache could let a genuine duplicate
   slip through) — any caching here would need very short TTLs or a cache-invalidate-on-write
   pattern, not a naive read-through cache.

### Low-impact / good-practice already present

6. **Dynamic/partial UPDATE column-building** (both mesh and auth sides) avoids the
   full-overwrite cost seen in Fact Sheet KT — an efficient pattern already in place given the
   partial-update business requirement.
7. **The RabbitMQ fan-out is enormous (11+ distinct consumer-queues, section 6)** —
   architecturally necessary given how many domains need to stay in sync with the master
   user-record; a single `GLUSR_USR` write has a very large "blast radius" of downstream
   processing, but this is fully async and doesn't block the write itself.

---

## 12. Cron Inventory

| Cron | Relevant to Add/Update core flow? | What it does |
|---|---|---|
| [`OldGsmNumberTTL.go`](../../service-api-go-production/service-api-go-production/crons/users/OldGsmNumberTTL.go) | Adjacent — touches `GLUSR_USR`-related GSM/number data, not the Add/Update write-path itself | **[INFERRED from filename — not read in full this pass]** likely a TTL/cleanup job for stale GSM number bindings |
| [`VerificationSourceUpdate.go`](../../service-api-go-production/service-api-go-production/crons/users/VerificationSourceUpdate.go) | Adjacent | **[INFERRED from filename — not read in full this pass]** likely updates a verification-source flag, possibly related to the TrustSeal/GST verification-source concept documented in the GST Technical Doc |
| `GLUSR Manual cron` (the `UPDATEDBY` literal, section 4e) | **Yes, directly** | Not a standalone binary found in this pass — it's a **label/convention**, not a discoverable cron job in these repos. Some external/internal process calls the standard `UserUpdateController` API with `UPDATEDBY="GLUSR Manual cron"` to get routed to the `GLUSR_UPDATE_CRON_SERVICE` queue. The actual caller of this pattern was **not located in this pass** — flagged as an Open Question. |

No dedicated "Add" or "Update" cron binary (analogous to GST's `gst_tact_veri_cron.go`) was
found in `service-api-go-production/crons/` or `user-temp-consumers-production` for this
specific feature — the closest thing is the `GLUSR Manual cron` UPDATEDBY-label convention
above, whose actual invoking process wasn't identified.

---

## 13. Edge Cases & Gotchas (technical POV)

1. **`InsertAuthPG`'s failure-handling isn't a visible rollback of the mesh-insert** — if the
   mesh-write succeeds but the auth-write fails, the code concatenates both outputs into
   `REASON` but the overall function still reports based on the mesh-result; a user could
   theoretically end up with a `GLUSR_USR` row but no usable login-credentials (not confirmed
   whether this has ever actually manifested — see Open Questions).
2. **New accounts skip any pending-approval step** (`APPROV="A"` forced) — any
   moderation/review of new-signups must happen via a separate downstream process (e.g. the
   fraud/banned-content-detection consumers), not at creation-time itself.
3. **Duplicate-detection race-condition window**: between `CheckUniqueMesh` and the actual
   `INSERT ... ON CONFLICT DO NOTHING`, a concurrent request for the same mobile/email could
   theoretically both pass the pre-check; the `ON CONFLICT DO NOTHING` backstop prevents a
   DB-level duplicate-row, but the losing request's exact response-handling wasn't traced in
   this pass.
4. **Duplicate-match resolution never inserts — always redirects to an internal HTTP-loopback
   update.** Support/debugging note: if a signup "silently doesn't create a new account," this
   is very likely the expected 6-branch duplicate-resolver (section 4a) at work, not a bug.
5. **TrustSeal-unverification is per-attribute, and asymmetric based on `UPDATEDUSING`** — a
   TrustSeal-originated write does NOT self-unverify the field it's writing (unless clearing
   it), but a non-TrustSeal write to the same field DOES unverify it. This asymmetry is easy
   to misread as a bug if not understood (section 4d point 2).
6. **`TsNoUpdateVerify` fields are silently stripped, not rejected, for TrustSeal callers** —
   if a TrustSeal-sourced request includes e.g. `EMAIL`, it is deleted from `inputParams`
   before the SQL runs; there's no error surfaced for this specific field being dropped
   (confirmed at [`UserUpdateModel.go:449-456`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUpdateModel.go)) — a caller naively checking "did my
   request succeed" wouldn't know this one field was silently ignored.
7. **Cron-vs-normal RabbitMQ routing is a bare string-equality check on `UPDATEDBY`** — any
   caller (not just a genuine automated cron) that happens to set
   `UPDATEDBY="GLUSR Manual cron"` would get routed to the cron-lane. No additional
   authorization/gating was found around this specific string check in this pass.
8. **`LASTLOGIN` only refreshes when `UPDATEDBY=="USER"` (case-insensitive)** — admin/system
   updates leave the login-timestamp untouched, which is correct behavior but worth knowing
   when debugging "why didn't LASTLOGIN change" tickets.

---

## 14. Open Questions

1. If `InsertAuthPG`/`UpdateAuthPG` fails after a successful mesh-write, is there any
   cleanup/retry, or can a "profile exists but can't log in" state occur in practice? Not
   traced beyond the code-level absence of a rollback in this pass.
2. Who actually calls the Update-API with `UPDATEDBY="GLUSR Manual cron"`? No standalone cron
   binary matching this label was located in either `service-api-go-production/crons/` or
   `user-temp-consumers-production` in this pass.
3. Exact shape/frequency of `CheckUniqueMesh`'s underlying queries (1 combined query vs 3
   separate) — not confirmed, function body not read in full this pass.
4. `USER_CENTRALIZED_BANNED` — referenced conceptually via the `glusrBannedContent` payload
   (section 4f) but not found as an exact registered queue-name in
   `IntializeMsgBroker.go` in this pass; confirm whether banned-content routing happens inside
   `dbActionUserCentralizedQueue` as an internal branch, or via a differently-named queue.
5. `CallDetailsService`'s exact downstream purpose (section 4a, branch 1) — confirmed to be
   an HTTP-loopback fired when GST is present on a duplicate-resolving Add-request, but the
   target service/its effect wasn't traced in this pass.
6. Exact purpose of the session-key/masked-mobile-email sub-flow reachable through the Add
   endpoint (section 5, rule 8) — is this a legitimate part of the registration UX (e.g.
   OTP-based signup) or a vestigial code path?
7. `OldGsmNumberTTL.go`/`VerificationSourceUpdate.go` crons — not read in full this pass;
   confirm their relationship (if any) to the core Add/Update duplicate-detection or
   TrustSeal-unverification mechanisms documented here.
8. Live DB schema verification — this doc sirf Go SQL strings jo imply karti hain wahi
   reflect karta hai, koi live pgAdmin cross-check nahi hua.

---

## See also

- [`User_Add_Update_Business_Doc.md`](./User_Add_Update_Business_Doc.md) — product perspective
- Virtually every other KT folder in `docs/` references `FK_GLUSR_USR_ID`/`GLUSR_USR_ID`
  created by this feature — this is the foundational identity-record for the entire
  users-domain.
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — shares the RTF
  (`RTF_master_arr`/`GstRtfRequest`) sync mechanism (section 4g) and the
  `FK_GST_VERIFICATION_SRC_ID` verification-source concept that parallels this feature's
  per-attribute TrustSeal-unverification model (section 4d)
- [`../Bank Details KT/Bank_Details_Technical_Doc.md`](../Bank%20Details%20KT/Bank_Details_Technical_Doc.md),
  [`../Fact Sheet KT/Fact_Sheet_Technical_Doc.md`](../Fact%20Sheet%20KT/Fact_Sheet_Technical_Doc.md),
  [`../Social Contacts KT/`](../Social%20Contacts%20KT/) — sibling branches of the broader
  shared user-details write-surface, all writing against the same `GLUSR_USR_ID`

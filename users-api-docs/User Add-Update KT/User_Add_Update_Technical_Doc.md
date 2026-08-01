# User Add / Update (Core Registration & Profile) — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`User_Add_Update_Business_Doc.md`](./User_Add_Update_Business_Doc.md)
dekho.

**Scope note**: Yeh users-domain ka **sabse bada aur sabse-heavily-integrated feature**
hai — `UserAddController.go`/`UserUpdateController.go` aur unke models
(1000+/1500+ lines each) poora core-registration-engine hain. Given the scale, yeh doc
**architecture-level depth** pe likha gaya hai (gateway, dual-DB-write, duplicate-
detection, RabbitMQ-fan-out) — line-by-line har field-mapping is doc ka scope nahi hai,
jaisa smaller single-purpose controllers ke liye kiya gaya tha.

**Repos**: `service-api-go-production` (write), `users-api-go-production` (read — main
profile-detail/`otherdetail` queries), `user-temp-consumers-production` (largest
consumer-fan-out in the entire codebase).

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Add (registration) | write | [`UserAddController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserAddController.go) (248 lines), [`UserAddModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddModel.go) (1009 lines — `InsertGlusrMeshPg`, `InsertAuthPG`, `PrepareInputParamsForInsert`) |
| Update (profile-edit) | write | [`UserUpdateController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserUpdateController.go) (561 lines), [`UserUpdateModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUpdateModel.go) (1498 lines) |
| Shared allowlists/config | write | [`UserAddUpdateUtils.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddUpdateUtils.go) (`UserAddModids`, `UserUpdateModids`, `TsNoUpdateVerify`) |
| Read (profile-detail) | read | `UserDetailModel.go`, `UserOtherDetailModel.go` — same "big combined query" files referenced across many other KT folders in this series (GST, Fact Sheet, Social Contacts) |

---

## 2. Routes

| Method | serviceName | Repo | Controller |
|---|---|---|---|
| POST | `GLUSR_INSERT_SERVICE` | write | `UserAddController` |
| POST | `GLUSR_UPDATE_SERVICE` | write | `UserUpdateController` |

---

## 3. Data Model — Tables

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_USR` | meshpg | **The master user/company record** — every other feature in this KT series references `FK_GLUSR_USR_ID` back to this table | `GLUSR_USR_ID` (PK, `RETURNING` on insert), `GLUSR_USR_DISPLAY_ID`, dynamically-built column-set covering name/mobile/email/address/company/approval-status/etc. — column-list built at runtime, not a fixed literal list (see §4) |
| (auth-credentials table, name not fully traced) | authpg | Login-credentials, created in lockstep with a successful `GLUSR_USR` insert | `InsertAuthPG()` — [`UserAddModel.go:136-141`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddModel.go) |

**Insert uses `ON CONFLICT DO NOTHING`**: `INSERT INTO GLUSR_USR (%s) VALUES (%s) ON
CONFLICT DO NOTHING RETURNING GLUSR_USR_ID, GLUSR_USR_DISPLAY_ID` —
[`UserAddModel.go:776`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddModel.go).
Combined with application-level duplicate-pre-checks (mobile/email/GSM), this gives a
**two-layer duplicate-defense**: app-level lookup first, DB-level constraint as a
backstop.

---

## 4. Business Rules & Validation (code se)

### Add (registration)

1. **Gateway allowlist (`UserAddModids`) is the largest in this entire KT series — 42
   entries**: `MY`, `GLADMIN`, `TOLLFREE`, `Email Marketing`, `HTVENDOR`, `Weberp`,
   `LEAP`, `IPC-Admin`, `NSD Sales`, `MAPI`, `ENQUIRY`, `FREE-WEBSITE`,
   `M.INDIAMART.COM`, `TRADE`, `BL`, `TENDER`, `PAYNOW`, `CREDIT ALLOCATION`,
   `OVP Process`, `SAMPARK Process`, `TOLLFREE Process`,
   `VENDOR CITY Pin Correction`, `Notification Server`, `search`, `FCP`, `PNS`, `Merp`,
   `IMOB`, `SELLERMY`, `PAYWIM`, `BUYERS_FEEDBACK`, `FLPNS`, `LEAPIN`,
   `User_Enrichment`, `IMLOGI`, `EXTLEAD`, `WHATSAPP`, `WA_9696`, `NSDMERP`, `EXPORT`,
   `FLIPS`, `PICSRCH`.
   [`UserAddUpdateUtils.go:423`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddUpdateUtils.go)
2. **Duplicate-detection runs BEFORE insert, on 3 identity-signals**: existing-mobile
   match (`m_glid`), existing-email match (`e_glid`), existing-GSM/business-registration
   match (`g1_glid`). If any resolves to an existing `GLUSR_USR_ID`, the flow
   **redirects to an internal-update path instead of inserting** — `output = "Input
   data not unique"` is only the fallback error-string if the redirect itself also fails.
   [`UserAddModel.go:167-199`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddModel.go)
3. **On the true-insert path, several fields are force-set regardless of caller input**:
   `APPROV="A"` (auto-approved), `LASTMODIFIED`/`LASTLOGIN`/`MEMBERSINCE="SYSDATE"`,
   `PAID_SERV=0`, `LOC_PREF=4`, `FCP_FLAG=0` — new accounts are immediately active, no
   pending-approval gate exists at this layer.
   [`UserAddModel.go:104-114`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddModel.go)
4. **DB-level unique-constraint-violations are pattern-matched by name** to produce
   friendlier errors: `EMAIL_UQ`/`EMAIL_DUP_UQ` → "Duplicate" email,
   `MOBCNTR_IDX` → "Duplicate" mobile, `SYS_C00124106` → "Duplicate" PNS — a
   defense-in-depth layer catching whatever the app-level pre-check missed (race
   conditions between the pre-check and the actual insert).
   [`UserAddModel.go:151-160`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddModel.go)
5. **Successful `GLUSR_USR` insert immediately triggers `InsertAuthPG`** — a second,
   sequential write into the separate `authpg` database for login-credentials; if this
   second write fails, the code still returns based on the mesh-insert's success (the
   auth-insert's own output is logged but doesn't appear to roll back the mesh-insert —
   worth flagging as an Open Question, see §9).
6. **If `GST` is present on an add-request that resolves to an existing user**, a
   separate `CallDetailsService` HTTP-call is made — yet another HTTP-loopback pattern
   (consistent with Rating Usefulness, Trust Verification, Update With OTP, App-Rating
   Feedback documented elsewhere in this series).
   [`UserAddModel.go:168-180`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddModel.go)

### Update (profile-edit)

7. **Gateway allowlist (`UserUpdateModids`) is nearly identical to the Add-allowlist**,
   with a few differences (`BI`, `NSD` added; slightly different set overall) — 43
   entries. [`UserAddUpdateUtils.go:425`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddUpdateUtils.go)
8. **Update is genuinely dynamic/partial**: `UPDATE GLUSR_USR SET %s WHERE
   GLUSR_USR_ID=$%d` — column-list built at runtime from whichever fields the caller
   sent, unlike Fact Sheet's full-overwrite pattern.
   [`UserUpdateModel.go:855`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUpdateModel.go)
9. **`TsNoUpdateVerify` is a field-list** (`ADD1`, `ADD2`, `CITY`, `CITY_OTHERS`,
   `STATE`, `STATE_OTHERS`, `ZIP`, `COUNTRYNAME`, `PH_MOBILE`, `PH_MOBILE_ALT`, `EMAIL`,
   `EMAIL_ALT`, `FK_GL_CITY_ID`, `FK_GL_STATE_ID`) — these are exactly the kind of
   identity/address-defining fields that, per the Business Doc, can affect TrustSeal
   verification-status; the name strongly suggests "fields that, if changed,
   TrustSeal-should-NOT-treat-as-still-verified."
   [`UserAddUpdateUtils.go:427`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddUpdateUtils.go)
10. **Two RabbitMQ service-names used depending on caller-type**:
    `GLUSR_UPDATE_CRON_SERVICE` (when the update originates from an automated/cron
    process) vs `GLUSR_UPDATE_SERVICE` (normal caller-driven updates) — allowing
    downstream consumers to potentially treat cron-driven bulk-updates differently from
    user-driven ones.
    [`UserUpdateModel.go:1247-1249`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUpdateModel.go)
11. **A second, separate `authpg` UPDATE also exists** (`UPDATE GLUSR_USR SET %s WHERE
    GLUSR_USR_ID=$%d` — same shape, different DB, line 1420) — confirming update-flow
    also needs to keep the two databases (mesh-profile + auth-credentials) in sync.

---

## 5. RabbitMQ / Kafka / Redis

| Queue / `SERVICENAME` | Publisher | Consumer(s) | Purpose |
|---|---|---|---|
| `GLUSR_INSERT_SERVICE` → `user.upsert.<modulus>` (`USER.topic`) | `UserAddModel.go` | Multiple `USER_UPSERT_*` consumers (see below) | New-account fan-out |
| `GLUSR_UPDATE_SERVICE` → `USER_CENTRALIZED_QUEUE` | `UserUpdateModel.go` (normal path) | `dbActionUserCentralizedQueue` | Profile-update fan-out |
| `GLUSR_UPDATE_CRON_SERVICE` | `UserUpdateModel.go` (cron-driven path) | (routes into the same centralized-queue family, per naming) | Distinguishes automated bulk-updates |

**This is the largest downstream-consumer-fan-out found anywhere in this codebase.**
Consumers registered against this event-family (per
[`IntializeMsgBroker.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go)) include:

- `USER_UPSERT_VERIFICATION`
- `USER_UPSERT_ALERTPG` / `USER_UPSERT_ALERTPG_BULK`
- `USER_UPSERT_BLDISPLAY` / `USER_UPSERT_BLDISPLAY_BULK`
- `USER_UPSERT_TRUSTPG` / `USER_UPSERT_TRUSTPG_BULK`
- `USER_UPSERT_ENQUIRYPG` / `USER_UPSERT_ENQUIRYPG_BULK`
- `USER_UPSERT_LMSPG_FEDRATED` / `USER_UPSERT_LMSPG_BULK_FEDRATED`
- `USER_UPSERT_CSL` / `USER_UPSERT_CSL_BULK`
- `USER_CENTRALIZED_QUEUE` / `USER_CENTRALIZED_QUEUE_FAIL` / `USER_CENTRALIZED_BANNED`
- `USER_UPSERT_ADDITIONAL_GCP_IN` / `USER_UPSERT_ADDITIONAL`
- `USER_UPSERT_CITY_OTHERS`

This confirms `GLUSR_USR` add/update is genuinely the **central nervous-system event**
of the whole platform — nearly every downstream domain (verification, alert, blacklist-
display, trust, enquiry, LMS, CSL/search, banned-detection) maintains its own synced
copy fed from this single event-family.

**Koi Kafka usage nahi mila directly in this write-path.**

---

## 6. End-to-End Technical Flow

### Add (registration)
```
New user
    │
    ▼
[API — write]  POST serviceName=GLUSR_INSERT_SERVICE  {name, mobile, email, address,
                company-details, GST?, ...}
    │  UserAddController.go
    ▼
Duplicate pre-check: mobile-match(m_glid) / email-match(e_glid) / GSM-match(g1_glid)
    │
    ├─ match found → redirect to internal-update path (existing account touched,
    │                 no new insert) [+ optional GST CallDetailsService HTTP-loopback]
    │
    └─ no match →
          PrepareInputParamsForInsert() — trigger-level checks
          force-set APPROV="A", LASTMODIFIED/LASTLOGIN/MEMBERSINCE=SYSDATE, etc.
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
                                                 enquiryPg, LMS, CSL/search, banned...)
```

### Update (profile-edit)
```
Existing user
    │
    ▼
[API — write]  POST serviceName=GLUSR_UPDATE_SERVICE  {USR_ID, <any subset of fields>}
    │  UserUpdateController.go — Gateway(UserUpdateModids)
    ▼
Dynamic partial UPDATE
    │  UPDATE GLUSR_USR SET <only-supplied-columns> WHERE GLUSR_USR_ID=$N
    │  (TsNoUpdateVerify-listed fields, if changed, affect TrustSeal-verify-status)
    │  second UPDATE against authpg (credentials-side sync)
    ▼
[DB — meshpg + authpg]
    │  on success:
    ▼
[RabbitMQ]  cron-driven → GLUSR_UPDATE_CRON_SERVICE
            normal      → GLUSR_UPDATE_SERVICE → USER_CENTRALIZED_QUEUE
    ▼
[CONSUME]  dbActionUserCentralizedQueue → fans out further downstream
```

---

## 7. Optimization Scope — DB Response-Time Contribution

### Medium-High impact

1. **Add-flow does a 3-signal duplicate pre-check (mobile/email/GSM) BEFORE the
   insert, then a second dual-write (mesh + auth) after** — this is inherently a
   multi-round-trip flow given the business-requirement (must know before committing
   whether this is a genuine new account), but the 3 duplicate-lookups could potentially
   be combined into a single query with `OR`-conditions instead of 3 separate lookups,
   if they aren't already (not fully confirmed which pattern is used without reading the
   lookup-helper functions in full).
2. **`InsertAuthPG` is a fully sequential second write after the mesh-insert succeeds**
   — a genuine 2-round-trip cost on every new-account creation; given both writes must
   logically succeed together (a user without auth-credentials is unusable), this is
   hard to avoid without cross-database-transaction support, which Postgres doesn't
   offer across separate database instances.

### Low-Medium impact

3. **Update-flow's dynamic-column-building avoids the full-overwrite cost seen in Fact
   Sheet KT** — already a reasonably efficient pattern given the partial-update
   requirement.
4. **The RabbitMQ fan-out is enormous (10+ distinct consumer-queues)** — this is
   architecturally necessary given how many domains need to stay in sync with the
   master user-record, but it does mean a single `GLUSR_USR` write has a very large
   "blast radius" of downstream processing; any slowness/backlog in one consumer
   doesn't block the write itself (async), but could create visible staleness in that
   one downstream domain.

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora User Add/Update flowchart yahan dekho](https://lucid.app/lucidchart/31664259-52e5-468b-8401-1014b55ee292/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **`InsertAuthPG`'s failure-handling isn't a visible rollback of the mesh-insert** —
   if the mesh-write succeeds but the auth-write fails, the code logs both outputs but
   the overall function still appears to report based on the mesh-result; worth
   confirming whether a user could end up with a `GLUSR_USR` row but no usable
   login-credentials.
2. **New accounts skip any pending-approval step** (`APPROV="A"` forced) — any
   moderation/review of new-signups must happen via a separate downstream process (e.g.
   the fraud/banned-detection consumers), not at creation-time itself.
3. **Duplicate-detection race-condition window**: between the app-level pre-check and
   the actual `INSERT ... ON CONFLICT DO NOTHING`, a concurrent request for the same
   mobile/email could theoretically both pass the pre-check; the `ON CONFLICT DO
   NOTHING` backstop prevents a DB-level duplicate-row, but the losing request's
   response-handling (does it correctly report "duplicate" or does it silently no-op
   with a misleading success?) wasn't traced in this pass.
4. **`TsNoUpdateVerify` field-list is inferred by name, not directly confirmed** to be
   consumed by TrustSeal's verification-reset logic in this pass — its usage-site
   wasn't traced beyond its definition.

---

## 10. Open Questions

1. If `InsertAuthPG` fails after a successful mesh-insert, is there any cleanup/retry,
   or can a "profile exists but can't log in" state occur?
2. How exactly does `TsNoUpdateVerify` get consumed — which code-path reads this list
   and what does it do with a changed field from it (not traced in this pass)?
3. What determines whether an update is routed to `GLUSR_UPDATE_CRON_SERVICE` vs
   `GLUSR_UPDATE_SERVICE` — is it based on the caller's `modid`, an explicit flag, or
   something else?
4. Full performance-profile of the 3-signal duplicate pre-check (single combined query
   vs 3 separate queries) — not confirmed in this pass, would need a deeper read of the
   lookup-helper functions.
5. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`User_Add_Update_Business_Doc.md`](./User_Add_Update_Business_Doc.md) — product perspective
- Virtually every other KT folder in `docs/` references `FK_GLUSR_USR_ID`/`GLUSR_USR_ID`
  created by this feature — this is the foundational identity-record for the entire
  users-domain.
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md),
  [`../Bank Details KT/Bank_Details_Technical_Doc.md`](../Bank%20Details%20KT/Bank_Details_Technical_Doc.md),
  [`../Fact Sheet KT/Fact_Sheet_Technical_Doc.md`](../Fact%20Sheet%20KT/Fact_Sheet_Technical_Doc.md) —
  sibling branches of the broader shared user-details write-surface

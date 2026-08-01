# Bank Details — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`Bank_Details_Business_Doc.md`](./Bank_Details_Business_Doc.md)
dekho.

**Scope note**: Bank Details is NOT a standalone controller — it is a `Type=="BankDetails"`
branch of the same **shared multi-purpose user-details write endpoint** documented in
[`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) and
[`../Fact Sheet KT/Fact_Sheet_Technical_Doc.md`](../Fact%20Sheet%20KT/Fact_Sheet_Technical_Doc.md).
This branch is by far the most deeply-integrated of the three — 3 dedicated downstream
consumers plus a shared banned-detection fan-out.

**Repos**: `service-api-go-production` (write, shared controller),
`user-temp-consumers-production` (3 dedicated consumers + shared banned-detection
consumer).

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write (shared controller) | write | [`UserDetailsController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDetailsController.go), [`UserDetailsModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go) — `Type=="BankDetails"` branch at line 400 |
| Consumer — general/search-sync | consumers | [`USER_BANK_DETAILS.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BANK_DETAILS.go) (`dbActionUserBankDetails`, `SERVICENAME=USER_DETAIL_SERVICE`, `meshPg`) |
| Consumer — trust-sync | consumers | [`USER_BANK_DETAILS_TRUSTPG.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BANK_DETAILS_TRUSTPG.go) (`dbActionBankDetailsTrustPg`, `meshPg` + `trustPg`) |
| Consumer — verification | consumers | [`USER_VERIFY_BANK_DETAILS.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VERIFY_BANK_DETAILS.go) (`processMessageUserVerifyBankDetails`, `SERVICENAME=BANK_DETAIL_SERVICE`) |
| Consumer — banned/fraud-detection (shared) | consumers | `USER_PROFILE_BANNED_DETECT.go` (`dbActionUserProfileBannedDetect`, `SERVICENAME=USER_DETAIL_BANNED_SERVICE`) — shared with `ContactDetails`/`Franchise` branches |

---

## 2. Routes

| Method | serviceName | Repo | Controller |
|---|---|---|---|
| POST | (shared user-details write service, `TYPE=BankDetails`) | write | `UserDetailsController` |

---

## 3. Data Model — Table

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_BANK_DETAILS` | meshpg | Per-supplier bank-account records, multiple rows allowed, one "prime" | `GLUSR_BANK_ID` (PK), `FK_GLUSR_USR_ID`, `GLUSR_BANK_ACCOUNT_NUMBER`, `GLUSR_BANK_IFSC`, `GLUSR_BANK_ISPRIME` (nullable — `1`=prime, `NULL`=not-prime), `ENABLED` (`-1`=soft-deleted), `GLUSR_BANK_UPDATEDBY_FLAG`, `GLUSR_BANK_LAST_MODIFIED_DATE`, `GLUSR_USR_UPDATEDBY_ID`/`_UPDATEDBY`/`_UPDATEDBY_AGENCY`/`_UPDATESCREEN`/`_UPDATEDBY_URL`/`_IP`/`_IP_COUNTRY`/`_HIST_COMMENTS` |
| `iil_verification_details` | meshpg (joined, read-only in this branch) | Verification-attribute linkage used to decide auto-prime for system-driven inserts | joined on `fk_glusr_usr_id` + `fk_gl_attribute_refid=glusr_bank_id`, filtered `fk_gl_attribute_id IN (398,390)` — [`UserDetailsModel.go:486-492`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go) |

---

## 4. Business Rules & Validation (code se)

1. **`reqType` classification drives prime-assignment logic**: `gate=="MERP"` →
   `"MERP"`; `gate` in `{MAPI, GLADMIN, BUYERMY, SELLERMY}` → `"USER"`; anything else →
   `"SYSTEM"`.
   [`UserDetailsModel.go:405-411`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)
2. **Delete is soft, flag-driven** (`flagDel=="D"`): sets `ENABLED=-1` on the targeted
   `GLUSR_BANK_ID`, no physical row removal.
   [`UserDetailsModel.go:416-480`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)
3. **New-account prime-assignment (insert-path, `id==""`)** depends on `reqType`:
   - `reqType=="USER"` → `bank_isprime` forced to `"1"` — a user-initiated new account
     is always prime by default.
   - `reqType=="SYSTEM"` → a pre-check query counts existing enabled+prime bank-rows
     that are **also linked via `iil_verification_details`** (attribute IDs `398`/`390`)
     for this user; if none found (`count==0`), the new account becomes prime, else it
     doesn't. This means system-driven inserts only auto-claim "prime" if no
     already-verified-prime-account exists.
   - `reqType=="MERP"` → `bank_isprime` explicitly `nil` (never auto-prime).
   [`UserDetailsModel.go:482-535`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)
4. **Update-path (`id!=""`)**: first fetches the existing account/IFSC/prime-status
   (`oldBAcct`/`oldIfsc`/`wasPrime` — pre-check `SELECT`), then dynamically builds the
   `UPDATE` from a whitelist map (`UserDetails_BankDetails`), excluding `id`,
   `attachment_type_id`, `attachment_url`, `glusridval`, `attachment_modid` from the
   updatable-column-set.
   [`UserDetailsModel.go:537-628`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)
5. **Explicit prime-set triggers an un-prime of every other account**: if
   `bank_isprime=="1"` in the request, a second query runs — `UPDATE GLUSR_BANK_DETAILS
   SET GLUSR_BANK_ISPRIME=NULL WHERE FK_GLUSR_USR_ID=$1 AND GLUSR_BANK_ID<>$2 AND
   GLUSR_BANK_ISPRIME=1 RETURNING GLUSR_BANK_ID` — enforcing the "only one prime account"
   invariant at write-time, not via a DB-constraint.
   [`UserDetailsModel.go:639-641`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)
6. **`GLUSR_BANK_ID` is echoed back in the RabbitMQ columns-payload** specifically for
   the `BankDetails` type (line 3169), unlike some other branches.
7. **BankDetails is part of the shared banned-detection fan-out**: alongside
   `ContactDetails`/`Franchise`, a successful BankDetails write fires
   `SERVICENAME=USER_DETAIL_BANNED_SERVICE` — a fraud/banned-account-pattern check.
   [`UserDetailsModel.go:4080-4096`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)

---

## 5. RabbitMQ / Kafka / Redis

| Queue / `SERVICENAME` | Publisher | Consumer | Purpose |
|---|---|---|---|
| `USER_DETAIL_SERVICE` → `comp.sync.<modulus>` (`USER.topic`) | shared `UserDetailsModel.go` write-path | [`USER_BANK_DETAILS.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BANK_DETAILS.go) (`dbActionUserBankDetails`, `meshPg`) | General search/directory-replica sync |
| `BANK_DETAIL_SERVICE` → `user.verifybankdetails.*` (`USER.topic`) | shared write-path | [`USER_VERIFY_BANK_DETAILS.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VERIFY_BANK_DETAILS.go) (`processMessageUserVerifyBankDetails`) | Bank-account verification pipeline |
| (Trust-sync, exact `SERVICENAME` not confirmed in this pass) | shared write-path | [`USER_BANK_DETAILS_TRUSTPG.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BANK_DETAILS_TRUSTPG.go) (`dbActionBankDetailsTrustPg`, `meshPg`+`trustPg`) | Syncs bank-detail-changes into the Trust-domain database (`trustPg`) — likely feeds TrustSeal-eligibility logic |
| `USER_DETAIL_BANNED_SERVICE` (empty routing-key, `USER.topic`) | shared write-path — for `BankDetails`/`ContactDetails`/`Franchise` types | `USER_PROFILE_BANNED_DETECT.go` (`dbActionUserProfileBannedDetect`) | Fraud/banned-account-pattern detection |

**Koi Kafka ya dedicated-Redis usage nahi mila.**

**Notable finding**: Bank Details has the **most downstream-fan-out of any feature
documented in this KT series** — 4 distinct consumers (general-sync, verification,
trust-sync, banned-detection), reflecting its role as a high-stakes trust/financial
data-point.

---

## 6. End-to-End Technical Flow

```
Supplier / MERP / MAPI / GLADMIN / BUYERMY / SELLERMY / SYSTEM
    │
    ▼
[API — write]  POST (shared user-details service)  TYPE=BankDetails
                {id?, flag_del?, account_number, ifsc, bank_isprime?, ...}
    │  UserDetailsController.go (shared) — Gateway check
    │  reqType classify: MERP / USER / SYSTEM (from gate)
    ▼
UserDetailsModel.go — Type=="BankDetails" branch
    │
    ├─ flag_del="D" → UPDATE ... SET ENABLED=-1 (soft-delete)
    │
    ├─ id=="" (insert) →
    │     reqType=USER   → bank_isprime forced "1"
    │     reqType=SYSTEM → pre-check verified-prime-count query → conditional prime
    │     reqType=MERP   → bank_isprime = nil
    │     INSERT INTO GLUSR_BANK_DETAILS (...)
    │
    └─ id!="" (update) →
          pre-check SELECT (old account/IFSC/prime)
          dynamic whitelist UPDATE
          if bank_isprime=="1" → UPDATE others SET ISPRIME=NULL (un-prime rest)
    ▼
[DB — meshpg]  GLUSR_BANK_DETAILS
    │  on success:
    ▼
[RabbitMQ] ─┬─ USER_DETAIL_SERVICE       → USER_BANK_DETAILS.go (search/general sync)
            ├─ BANK_DETAIL_SERVICE       → USER_VERIFY_BANK_DETAILS.go (verification)
            ├─ (trust-sync queue)        → USER_BANK_DETAILS_TRUSTPG.go (meshPg+trustPg)
            └─ USER_DETAIL_BANNED_SERVICE → USER_PROFILE_BANNED_DETECT.go (fraud-check)
```

---

## 7. Optimization Scope — DB Response-Time Contribution

### Medium impact

1. **Insert-path for `reqType=="SYSTEM"` does a pre-check JOIN-query before the actual
   insert** — a genuine extra sequential round-trip specifically for the auto-prime
   decision; unavoidable given the business-logic (must know if a verified-prime already
   exists), but worth being aware of as added latency on system-driven inserts
   specifically (not user-driven ones, which skip this check).
2. **Update-path always does a pre-check SELECT before the UPDATE** (`oldBAcct`/
   `oldIfsc`/`wasPrime`) — same "check-then-write" pattern flagged repeatedly across this
   KT series; here it's plausibly needed to detect *changes* (for the RabbitMQ payload
   or banned-detection comparison), not purely for insert-vs-update branching.
3. **Explicit prime-set triggers a second UPDATE query** (un-prime the rest) —
   unavoidable given no DB-level "only one true per group" constraint exists; a partial
   unique index (`WHERE GLUSR_BANK_ISPRIME=1`) at the DB level could enforce this
   invariant without the application needing a second round-trip, though it would change
   the failure-mode from "silent second-update" to "constraint-violation-error" and
   require app-level retry-logic instead.

### Low impact

4. **4-way RabbitMQ fan-out** — standard replication-cost, consistent with GST/Logo/
   Negative-Mcat patterns already documented in this series.

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora Bank Details flowchart yahan dekho](https://lucid.app/lucidchart/cc57d8ec-e0c3-4910-8368-53169ee06cbf/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **"Only one prime" is enforced at application-level, not DB-level** — a bug or a
   direct-DB-write bypassing this controller could leave multiple `GLUSR_BANK_ISPRIME=1`
   rows for one user; no DB-constraint would catch this.
2. **`reqType=="SYSTEM"`'s auto-prime pre-check has a 100ms query-timeout**
   (`timeOut := 100 * time.Millisecond`) — notably tighter than most other queries in
   this codebase (which typically use 1 second); a slow query here would silently fail
   this check (treated as `err != nil`, logged to `ERRORS`, but the code continues —
   worth confirming what `bank_isprime` ends up as if this query times out).
3. **Trust-sync consumer's exact `SERVICENAME`/queue wasn't confirmed in this pass** —
   only its DB-connections (`meshPg`+`trustPg`) were traced; a closer read of
   `IntializeMsgBroker.go`'s queue-name-mapping would be needed to pin down the exact
   RabbitMQ routing-key.
4. **Soft-delete doesn't reset `GLUSR_BANK_ISPRIME`** — if a prime-account is
   soft-deleted (`ENABLED=-1`), the code shown doesn't explicitly null its prime-flag;
   worth confirming whether a "disabled but still prime" state is possible and how
   downstream systems handle it.

---

## 10. Open Questions

1. What is the exact `SERVICENAME`/routing-key for the trust-sync queue consumed by
   `USER_BANK_DETAILS_TRUSTPG.go`?
2. What happens to `bank_isprime` if the 100ms system-auto-prime-check query times out —
   does it default to prime, not-prime, or is the request rejected?
3. Does soft-deleting a prime bank-account also clear its `GLUSR_BANK_ISPRIME` flag
   somewhere else in the codebase (cron/consumer), or can a disabled-but-prime state
   persist?
4. `fk_gl_attribute_id IN (398,390)` — what do these two attribute-IDs represent
   specifically (likely two different verification-types linked to bank-accounts)?
5. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`Bank_Details_Business_Doc.md`](./Bank_Details_Business_Doc.md) — product perspective
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — shared
  write-controller this feature is a branch of
- [`../Fact Sheet KT/Fact_Sheet_Technical_Doc.md`](../Fact%20Sheet%20KT/Fact_Sheet_Technical_Doc.md) —
  another branch of the same shared controller
- [`../Trust Verification KT/Trust_Verification_Technical_Doc.md`](../Trust%20Verification%20KT/Trust_Verification_Technical_Doc.md) —
  related verification-domain, likely intersects via `USER_BANK_DETAILS_TRUSTPG.go`

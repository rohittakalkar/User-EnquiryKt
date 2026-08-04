# Bank Details — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`Bank_Details_Business_Doc.md`](./Bank_Details_Business_Doc.md)
dekho — dono docs same flows cover karte hain, bas alag audience ke liye.

**Scope note**: Bank Details is NOT a standalone controller — it is a `Type=="BankDetails"`
branch of the same **shared multi-purpose user-details write endpoint** documented in
[`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) and
[`../Fact Sheet KT/Fact_Sheet_Technical_Doc.md`](../Fact%20Sheet%20KT/Fact_Sheet_Technical_Doc.md).
This branch is the most deeply-integrated of the three — 3 dedicated downstream consumers
plus a shared banned-detection fan-out, plus an external verification (PennyDrop) hop.

**Repos**: `service-api-go-production` (write, shared controller), `user-temp-consumers-production`
(3 dedicated consumers + shared banned-detection consumer), `users-api-go-production` (read).

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file path diya
gaya hai har jagah). Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — team se confirm karo]** likh diya hai, guess nahi kiya.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write (shared controller) | write | [`UserDetailsController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDetailsController.go) — `Type=="BankDetails"` gateway-allowlist check ~line 522-529 |
| Write (shared model) | write | [`UserDetailsModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go) — `BankDetails` branch **line 400-749**; SERVICENAME publishes at line 4081-4082 and 4185-4189 |
| Write router | write | [`router.go`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) — `POST /details` registered at line 107, 141, 369 |
| Read model | read | [`UserOtherDetailModel.go`](../internal/models/users/UserOtherDetailModel.go) — `case "BankDetails"` line 141-142, SELECT line 295-319, post-processing line 745-831 |
| Read controller | read | [`UserOtherDetailController.go`](../internal/controllers/UsersControllers/UserOtherDetailController.go) |
| Read router | read | [`routerUsers.go`](../internal/api/users_router/routerUsers.go) — `otherdetail/*params` line 264-265, 504-505 |
| RabbitMQ SERVICENAME→queue map | write | [`rabbitmq.go`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go) — `serviceToQueueMap`/`exchangeSet` line 51-82 |
| Consumer — general/search-sync | consumers | [`USER_BANK_DETAILS.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BANK_DETAILS.go) — queue `USER_BANK_DETAILS`, `serviceName="USER_DETAIL_SERVICE"` line 18 |
| Consumer — trust-sync | consumers | [`USER_BANK_DETAILS_TRUSTPG.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BANK_DETAILS_TRUSTPG.go) — queue `USER_BANK_DETAILS_TRUSTPG`, `meshPg`+`trustPg` |
| Consumer — verification (PennyDrop) | consumers | [`USER_VERIFY_BANK_DETAILS.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VERIFY_BANK_DETAILS.go) — queue `USER_VERIFY_BANK_DETAILS`, `serviceName="BANK_DETAIL_SERVICE"` line 16 |
| Consumer — banned/fraud-detection (shared) | consumers | [`USER_PROFILE_BANNED_DETECT.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PROFILE_BANNED_DETECT.go) — `TABLES=="GLUSR_BANK_DETAILS"` branch ~line 431-555, shared with `ContactDetails`/`Franchise` |
| Queue registration (consumer side) | consumers | [`router.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Router/router.go) line 80, 118; [`IntializeMsgBroker.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go) line 364, 374 |
| Column type-cast map (trust-sync) | consumers | `pkg/utils/map_index.go` — `UserBankDetailsTrustPgMap` line 1414 |

---

## 2. Routes

| Method | Path | serviceName / Type | Repo | Controller |
|---|---|---|---|---|
| POST | `/details` | `Type=BankDetails` (shared user-details write) | write | `UserDetailsController.go:107,141,369` |
| GET/POST | `otherdetail/*params` | `type=BankDetails` | read | `UserOtherDetailController.go` |

---

## 3. Data Model — Tables

> **Verification note**: yeh sab table/column names Go code ke andar embedded SQL strings
> se liye gaye hain (har row ke saamne file path diya hai). Live DB schema se pgAdmin pe
> cross-verify **nahi** kiya gaya hai — kisi migration/integration se pehle woh zaroor karo.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_BANK_DETAILS` | meshPg (primary write), trustPg (replica, via trust-sync consumer) | Per-supplier bank-account records, multiple rows allowed, one "prime" | `GLUSR_BANK_ID` (PK), `FK_GLUSR_USR_ID`, `GLUSR_BANK_NAME`, `GLUSR_BANK_ADDRESS_LINE1/2`, `GLUSR_BANK_CITY`, `GLUSR_BANK_COUNTRY`, `GLUSR_BANK_STATE`, `GLUSR_BANK_ACCOUNT_NUMBER`, `GLUSR_BANK_ACCOUNT_TYPE`, `GLUSR_BANK_ACCOUNT_SINCE`, `GLUSR_BANK_RELATIONSHIP_LENGTH`, `GLUSR_BANK_IFSC`, `GLUSR_BANK_ISPRIME` (nullable — `1`=prime, `NULL`=not-prime), `GLUSR_BANK_ACC_HOLDER_NAME`/`_NAME_1`/`_NAME_2`, `GLUSR_BANK_BRANCH`, `GLUSR_BANK_KYC_STATUS`, `ENABLED` (`-1`=soft-deleted), `GLUSR_BANK_UPDATEDBY_FLAG`, `GLUSR_BANK_ADD_DATE`, `GLUSR_BANK_LAST_MODIFIED_DATE`, `GLUSR_USR_UPDATEDBY_ID`/`_UPDATEDBY`/`_UPDATEDBY_AGENCY`/`_UPDATESCREEN`/`_UPDATEDBY_URL`/`_IP`/`_IP_COUNTRY`/`_HIST_COMMENTS`, `GLUSR_USR_MODID` — insert SQL [`UserDetailsModel.go:592`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go), read SELECT [`UserOtherDetailModel.go:295-319`](../internal/models/users/UserOtherDetailModel.go) |
| `iil_verification_details` | meshPg (joined, read-only in this branch) | Verification-attribute linkage used to decide auto-prime for system-driven inserts | joined on `fk_glusr_usr_id` + `fk_gl_attribute_refid=glusr_bank_id`, filtered `fk_gl_attribute_id IN (398,390)` — [`UserDetailsModel.go:486-492`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go) |
| `GLUSR_BANK_DETAIL_ATTACHMENT` [INFERRED — table name from alias `ba`/`upsertAttachmentMeshPg_v1`, not directly confirmed] | meshPg | Bank-account-proof attachment(s), one-to-many per bank record | joined in read-side SELECT — [`UserOtherDetailModel.go:295-319`](../internal/models/users/UserOtherDetailModel.go); upserted via `upsertAttachmentMeshPg_v1` goroutine on insert/update — [`UserDetailsModel.go:3230,3320`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go) |
| `GLUSR_USR` | meshPg | Read-only, fetched only to get supplier's name/mobile/email for the notification mail/SMS | `GLUSR_USR_ID`, name/mobile/email columns — `GetGlusrData` in [`USER_BANK_DETAILS.go:323-357`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BANK_DETAILS.go) |

---

## 4. Decode: `bank_isprime` and `reqType` — what drives the "one prime account" rule

There's no numeric status-code enum in this feature (unlike GST's `FK_GST_VERIFICATION_SRC_ID`).
The one "magic value" logic worth decoding is the **`gate → reqType` classification** that
drives all prime-assignment behavior:

| `gate` value | `reqType` | Prime behavior on insert |
|---|---|---|
| `MERP` | `"MERP"` | `bank_isprime` forced `nil` — never auto-prime |
| `MAPI`, `GLADMIN`, `BUYERMY`, `SELLERMY` | `"USER"` | `bank_isprime` forced `"1"` — always prime, no query |
| anything else (including the allowed gates `ERP_BT`, `ERP_NACH`, `WEBERP`, `PAYWIM`, `ANDWEB`) | `"SYSTEM"` | conditional — prime only if no existing enabled+prime+verified account found (100ms-timeout pre-check query) |

[`UserDetailsModel.go:402-411`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go).
Note the controller's gateway-allowlist (`MAPI, GLADMIN, BUYERMY, SELLERMY, ERP_BT, ERP_NACH,
WEBERP, PAYWIM, MERP, ANDWEB` — [`UserDetailsController.go:522-529`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDetailsController.go))
is broader than the three buckets the model actually branches on — every gate not explicitly
matched (`ERP_BT`/`ERP_NACH`/`WEBERP`/`PAYWIM`/`ANDWEB`) silently falls into `SYSTEM`.

**`fk_gl_attribute_id IN (398, 390)`** — the two attribute IDs used to identify a
"verified" bank-linked record in `iil_verification_details` for the SYSTEM auto-prime
pre-check. **[INFERRED — confirm with team]**: no enum, comment, or constant map for these
IDs was found anywhere across `service-api-go-production`, `user-temp-consumers-production`,
or `users-api-go-production` — their exact meaning (likely two different bank-verification
sub-types, e.g. penny-drop vs KYC) could not be conclusively determined from source code and
needs confirmation from the team or the `GL_ATTRIBUTE_MASTER` table data.

---

## 5. Business Rules & Validation (code se)

1. **Gateway allowlist for BankDetails writes**: only `MAPI, GLADMIN, BUYERMY, SELLERMY,
   ERP_BT, ERP_NACH, WEBERP, PAYWIM, MERP, ANDWEB` gates may write — else HTTP error
   `"Unauthorised request for updating bank details"`.
   [`UserDetailsController.go:522-529`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDetailsController.go)
2. **New-account mandatory fields**: if `id==""` (insert), `ac_no` (account number) and
   `ifsc_code` are required.
   [`UserDetailsController.go:553-565`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDetailsController.go)
3. **`reqType` classification drives prime-assignment logic** — see section 4 table above.
   [`UserDetailsModel.go:402-411`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)
4. **Delete is soft, flag-driven** (`flagDel=="D"`): `UPDATE GLUSR_BANK_DETAILS SET
   ENABLED=-1, ... WHERE GLUSR_BANK_ID=$10` — no physical row removal.
   [`UserDetailsModel.go:416-480`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)
5. **Soft-delete does NOT clear `GLUSR_BANK_ISPRIME`** — the delete `UPDATE` statement has
   no `GLUSR_BANK_ISPRIME` column at all, and no consumer resets it either (`USER_BANK_DETAILS.go`
   only skips mail/SMS on `ENABLED`-present messages, doesn't touch the prime flag). A
   soft-deleted (disabled) account can remain flagged prime indefinitely.
   [`UserDetailsModel.go:416-433`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go),
   [`USER_BANK_DETAILS.go:81-85`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BANK_DETAILS.go)
6. **New-account prime-assignment (insert-path, `id==""`)**:
   - `reqType=="USER"` → `bank_isprime` forced `"1"` unconditionally, no query.
   - `reqType=="SYSTEM"` → 100ms-timeout pre-check query joining `glusr_bank_details` +
     `iil_verification_details` (attribute IDs `398`/`390`); if `count==0`, new account
     becomes prime.
   - `reqType=="MERP"` → `bank_isprime` explicitly `nil` (never auto-prime).
   [`UserDetailsModel.go:481-535`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)
7. **SYSTEM auto-prime pre-check fails open on timeout**: the query uses a
   `timeOut := 100 * time.Millisecond` budget. On `err != nil` (including a timeout), the
   code only logs the error to `extraParamsForOutput["ERRORS"]["QUERY2_ERROR"]` and does
   **not** short-circuit — `count` stays at its Go zero-value (`0`), so execution falls
   through into the `count == 0` branch and sets `bank_isprime = "1"`. **A timed-out
   pre-check is therefore treated identically to "no existing prime account found," and the
   new SYSTEM-origin account is marked prime by default.**
   [`UserDetailsModel.go:502-532`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)
8. **Update-path (`id!=""`)**: first fetches the existing account/IFSC/prime-status via a
   1-second-timeout `SELECT GLUSR_BANK_ACCOUNT_NUMBER, GLUSR_BANK_IFSC,
   GLUSR_BANK_ISPRIME FROM GLUSR_BANK_DETAILS WHERE GLUSR_BANK_ID=$1`
   (`oldBAcct`/`oldIfsc`/`wasPrime`), then dynamically builds the `UPDATE` from a whitelist
   map (`UserDetails_BankDetails`).
   [`UserDetailsModel.go:537-637`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)
9. **Explicit prime-set triggers an un-prime of every other account**: if
   `bank_isprime=="1"` in the request, an async goroutine runs `UPDATE GLUSR_BANK_DETAILS
   SET GLUSR_BANK_ISPRIME=NULL WHERE FK_GLUSR_USR_ID=$1 AND GLUSR_BANK_ID<>$2 AND
   GLUSR_BANK_ISPRIME=1 RETURNING GLUSR_BANK_ID` — enforcing the "only one prime account"
   invariant at write-time, not via a DB-constraint. Runs after both insert (sentinel `"1"`,
   line 3172-3225) and update (sentinel `"2"`, line 3263-3315).
   [`UserDetailsModel.go:639-643`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)
10. **`GLUSR_BANK_ID` is echoed back in the RabbitMQ columns-payload** specifically for the
    `BankDetails` type, unlike some other branches, and also returned in the API response as
    `serviceResponse["BANK_DETAIL_ID"]`.
    [`UserDetailsController.go:429`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDetailsController.go)
11. **BankDetails is part of the shared banned-detection fan-out**: alongside
    `ContactDetails`/`Franchise`, a successful BankDetails write sets
    `rabbitMQData["SERVICENAME"] = "USER_DETAIL_BANNED_SERVICE"`.
    [`UserDetailsModel.go:4081-4082`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)
12. **Possible dead/inert code**: a second publish block sets
    `rabbitMQDataNew["SERVICENAME"] = "BANK_DETAIL_SERVICE"` for `Type=="BankDetails"`
    (else `"USER_DETAILS_SERVICE"`), but the subsequent `utils.PushToQueue(glusridval,
    rabbitMQData)` call uses `rabbitMQData` (the original map), **not** `rabbitMQDataNew`.
    This needs runtime/log verification to confirm whether the `BANK_DETAIL_SERVICE` SERVICENAME
    set here actually reaches the queue, or whether it's set elsewhere too (the
    `USER_VERIFY_BANK_DETAILS` consumer does receive `BANK_DETAIL_SERVICE`-tagged messages in
    practice, per its own `serviceName` constant, so the effective publish path for
    verification is presumed to work via a route not fully traced in this pass).
    [`UserDetailsModel.go:4185-4210`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go) — **[INFERRED — confirm with team]**
13. **Notification (mail/SMS) suppression flag**: system-driven writes that shouldn't
    trigger supplier-facing mail/SMS get `WAPI_INTERNAL_PROCESS="1"` appended to the RabbitMQ
    column-set (`bankdetailsNoMailSMS=="1"` case), which `USER_BANK_DETAILS.go` checks and
    skips (no-op ack) on.
    [`UserDetailsModel.go:746-749`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go),
    [`USER_BANK_DETAILS.go:86-90`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BANK_DETAILS.go)
14. **Update-path mail/SMS is skipped when nothing materially changed**: if account number,
    IFSC, and prime-flag are all unchanged from the old values, `USER_BANK_DETAILS.go` does
    not send mail/SMS for the update.
    [`USER_BANK_DETAILS.go:160`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BANK_DETAILS.go)
15. **Prime-account mail has a hard validation**: if `GLUSR_BANK_ISPRIME=="1"` and the
    account-holder name is blank, `sendmail` returns error `"account is prime and holder
    name is missing"` instead of sending.
    [`USER_BANK_DETAILS.go:231-234`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BANK_DETAILS.go)
16. **Banned/fraud-detection re-write uses a hardcoded shared `VALIDATION_KEY`**
    (`e27d039e38ae7b3d439e8d1fe870fc68`) to call back into the same write API and blank out
    any field flagged by `CallBannedApi` (profanity/banned-content checker) — identical
    pattern reused for `ContactDetails`.
    [`USER_PROFILE_BANNED_DETECT.go:542`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PROFILE_BANNED_DETECT.go)

---

## 6. RabbitMQ

GST-style shared `PushToQueue` helper drives all BankDetails messaging — no Kafka, no Redis
(see sections 7, 8).

| Queue / `SERVICENAME` | Publisher | Consumer | Exchange / routing (from `rabbitmq.go`) | Purpose |
|---|---|---|---|---|
| `USER_DETAIL_SERVICE` | shared write-path, all inserts/updates/deletes | [`USER_BANK_DETAILS.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BANK_DETAILS.go) (`dbActionUserBankDetails`, queue `USER_BANK_DETAILS`, `meshPg`) | `comp.sync.<modulus>` sharded fan-out | General search/directory-replica sync + supplier mail/SMS notification |
| `BANK_DETAIL_SERVICE` | shared write-path (line 4186-4189, see rule 12 — dead-code caveat) | [`USER_VERIFY_BANK_DETAILS.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VERIFY_BANK_DETAILS.go) (`processMessageUserVerifyBankDetails`, queue `USER_VERIFY_BANK_DETAILS`) | routing key `user.verifybankdetails.*`, exchange `USER.topic` — [`rabbitmq.go:53,82`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go) | Forwards bank-account to external **PennyDrop** verification API via `utils.PubAPI(...,"PennyDrop","BANK_DETAIL_SERVICE",...)` |
| (queue `USER_BANK_DETAILS_TRUSTPG`, exact `SERVICENAME` **not conclusively confirmed**) | shared write-path | [`USER_BANK_DETAILS_TRUSTPG.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BANK_DETAILS_TRUSTPG.go) (`dbActionBankDetailsTrustPg`, `meshPg`+`trustPg`) | queue name literal `USER_BANK_DETAILS_TRUSTPG`, registered [`router.go:118`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Router/router.go), [`IntializeMsgBroker.go:374`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go) | Syncs bank-detail changes into the Trust-domain database (`trustPg`), with a full-row-copy fallback (`BankDetailsSync`) if an UPDATE affects 0 rows — likely feeds TrustSeal-eligibility logic |
| `USER_DETAIL_BANNED_SERVICE` (direct queue, no exchange — [`rabbitmq.go:51,80`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go)) | shared write-path — for `BankDetails`/`ContactDetails`/`Franchise` types | [`USER_PROFILE_BANNED_DETECT.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PROFILE_BANNED_DETECT.go) (`dbActionUserProfileBannedDetect`, queue `USER_PROFILE_BANNED_DETECT`) | direct-to-queue, no routing key | Fraud/banned-content-pattern detection on free-text bank fields; blanks flagged fields via a self-call-back to the write API, else forwards unchanged to `USER_DETAIL_SERVICE` |

**Unresolved binding**: application code declares queue names but no explicit
`QueueBind`/routing-key-to-queue binding statement was found for `user.verifybankdetails.*`
→ `USER_BANK_DETAILS_TRUSTPG` vs `USER_VERIFY_BANK_DETAILS`. Naming and each consumer's own
`serviceName` constant strongly suggest `BANK_DETAIL_SERVICE`/`user.verifybankdetails.*`
routes to `USER_VERIFY_BANK_DETAILS`, and the trust-sync queue (`USER_BANK_DETAILS_TRUSTPG`)
is a separately-bound queue whose exact `SERVICENAME`/routing-key could not be found in any
`PushToQueue` call site in this pass — **[INFERRED — confirm against actual RabbitMQ broker
bindings/management UI]**.

**Notable finding**: Bank Details has the **most downstream fan-out of any feature
documented in this KT series** — 4 distinct consumers (general-sync, verification, trust-sync,
banned-detection) plus one external API hop (PennyDrop), reflecting its role as a high-stakes
trust/financial data-point.

---

## 7. Kafka

**Koi Kafka usage nahi mila** in any BankDetails-related file across all three repos.
`IntializeMsgBroker.go` does have a generic Kafka framework (`InitializeKafka`,
`KafkaDBFuncMap`) used by other features (e.g. GST's `USER_GST_DETAILS_BULK`), but grepping
for `Kafka` combined with `BANK` produced no hits. All BankDetails consumers register via
`InitializeRabbitMqv1`, confirming RabbitMQ-only.

---

## 8. Redis

**Koi Redis usage nahi mila** in any BankDetails-related file examined (controller, model
BankDetails branch, all four consumers). BankDetails reads (`GET otherdetail?type=BankDetails`)
go straight to Postgres — no caching layer. This was checked within the BankDetails-specific
files only, not exhaustively across the entire three repos.

---

## 9. End-to-End Technical Flows

### Flow A — Supplier (USER reqType) adds a new bank account

```
Supplier (seller panel/app), gate=MAPI/GLADMIN/BUYERMY/SELLERMY
    │
    ▼
[API — write]  POST /details  {type: "BankDetails", ac_no, ifsc_code, ...}  (id=="")
    │  UserDetailsController.go — gateway-allowlist check
    ▼
UserDetailsModel.go — Type=="BankDetails" branch
    │  reqType = "USER" (gate in MAPI/GLADMIN/BUYERMY/SELLERMY)
    │  bank_isprime forced "1" — no pre-check query
    ▼
[DB — meshPg]  INSERT INTO GLUSR_BANK_DETAILS (...) RETURNING GLUSR_BANK_ID
    │
    ├─ async goroutine: UPDATE others SET GLUSR_BANK_ISPRIME=NULL WHERE ... GLUSR_BANK_ISPRIME=1
    ├─ async goroutine: upsertAttachmentMeshPg_v1 (bank-proof attachment)
    │
    ▼
[RabbitMQ] ─┬─ USER_DETAIL_SERVICE        → USER_BANK_DETAILS.go (general sync + mail/SMS)
            ├─ BANK_DETAIL_SERVICE (caveat, rule 12) → USER_VERIFY_BANK_DETAILS.go (PennyDrop)
            ├─ USER_BANK_DETAILS_TRUSTPG queue → USER_BANK_DETAILS_TRUSTPG.go (trustPg sync)
            └─ USER_DETAIL_BANNED_SERVICE → USER_PROFILE_BANNED_DETECT.go (fraud-check)
```

### Flow B — System-driven insert (SYSTEM reqType, e.g. ERP/WEBERP-origin)

```
Automated system, gate=ERP_BT/ERP_NACH/WEBERP/PAYWIM/ANDWEB (or any unmatched gate)
    │
    ▼
UserDetailsModel.go — reqType = "SYSTEM"
    │
    ▼
[DB — meshPg, 100ms timeout]  SELECT count(1) FROM glusr_bank_details JOIN
    iil_verification_details ON fk_glusr_usr_id + fk_gl_attribute_refid=glusr_bank_id
    WHERE fk_gl_attribute_id IN (398,390) AND enabled=1 AND glusr_bank_isprime=1
    │
    ├─ err (incl. timeout) → count stays 0 → bank_isprime = "1" (fail-open, rule 7)
    ├─ count==0            → bank_isprime = "1"
    └─ count>0             → bank_isprime = nil
    ▼
[DB — meshPg]  INSERT INTO GLUSR_BANK_DETAILS (...)
    ▼
[RabbitMQ fan-out — same 4 consumers as Flow A]
```

### Flow C — Supplier updates an existing bank account (incl. explicit prime-switch)

```
Supplier / system, id != ""
    │
    ▼
UserDetailsModel.go
    │  1s-timeout pre-check SELECT (old account/IFSC/prime) — oldBAcct/oldIfsc/wasPrime
    ▼
[DB — meshPg]  dynamic whitelist UPDATE GLUSR_BANK_DETAILS SET ... WHERE GLUSR_BANK_ID=$id
    │
    ├─ if bank_isprime=="1" → async UPDATE others SET ISPRIME=NULL (un-prime rest)
    ├─ async upsertAttachmentMeshPg_v1
    ▼
[RabbitMQ] → USER_BANK_DETAILS.go
    │  compares new vs OLD_BANK_ACCOUNT/OLD_IFSC/WAS_PRIME
    │  unchanged on all 3 → skip mail/SMS
    │  changed → send mail/SMS (prime-holder-name-blank guard applies)
    → USER_BANK_DETAILS_TRUSTPG.go / USER_VERIFY_BANK_DETAILS.go / USER_PROFILE_BANNED_DETECT.go (same as Flow A)
```

### Flow D — Soft delete

```
Supplier / admin, flagDel == "D"
    │
    ▼
[DB — meshPg]  UPDATE GLUSR_BANK_DETAILS SET ENABLED=-1, ... WHERE GLUSR_BANK_ID=$id
    │  (no GLUSR_BANK_ISPRIME reset — rule 5)
    ▼
[RabbitMQ] → USER_BANK_DETAILS.go: sees ENABLED-present message → no-op ack, no mail/SMS
          → other consumers process as usual (trust-sync, banned-detect)
```

### Flow E — Banned-content detection & self-correcting write-back

```
[RabbitMQ]  USER_DETAIL_BANNED_SERVICE  (BankDetails/ContactDetails/Franchise)
    │
    ▼
USER_PROFILE_BANNED_DETECT.go — TABLES=="GLUSR_BANK_DETAILS" branch
    │  runs GLUSR_BANK_ADDRESS_LINE1/2, GLUSR_BANK_NAME, GLUSR_BANK_ACC_HOLDER_NAME/_NAME_1/_NAME_2
    │  through CallBannedApi (external banned-words checker)
    │
    ├─ flag=="2" (violation found) → blanks flagged fields → calls back into
    │      wapi_details_api (same write endpoint) with hardcoded VALIDATION_KEY
    │      → re-triggers Flow A/C write path with cleaned data
    │
    └─ nothing flagged → forwards message unchanged to USER_DETAIL_SERVICE/comp.sync.N
```

### Flow F — Bank-account read

```
Client
    │
    ▼
[API — read]  GET/POST otherdetail/*params?type=BankDetails
    │  UserOtherDetailController.go
    ▼
[DB — read-replica]  SELECT (bank cols) FROM GLUSR_BANK_DETAILS b LEFT JOIN
    <attachment table> ba ... WHERE FK_GLUSR_USR_ID=$1
    │  post-process: normalize account_since, format add/modified dates,
    │  reshape attachment rows into GLUSR_BANK_DETAIL_ATTACHMENTS array
    ▼
Response
```

---

## 10. Flow-wise DB & Table Usage

### Flow A/B — Insert (USER / SYSTEM reqType)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshPg | `iil_verification_details` JOIN `GLUSR_BANK_DETAILS` | SELECT (SYSTEM reqType only, 100ms timeout) | Decide whether the new account auto-claims "prime" — skipped entirely for USER/MERP reqType |
| 2 | meshPg | `GLUSR_BANK_DETAILS` | INSERT ... RETURNING GLUSR_BANK_ID | Create the bank-account record |
| 3 | meshPg | `GLUSR_BANK_DETAILS` | UPDATE (async, only if `bank_isprime=="1"`) | Un-prime every other account for this user |
| 4 | meshPg | attachment table | UPSERT (async) | Save/link bank-proof attachment metadata |
| 5 | meshPg (consumer) | `GLUSR_USR` | SELECT (`GetGlusrData`) | Fetch supplier name/mobile/email for the confirmation mail/SMS |
| 6 | trustPg (consumer) | `GLUSR_BANK_DETAILS` | INSERT (or fallback: SELECT * on meshPg + N inserts) | Trust-domain replica sync |
| 7 | — (external) | PennyDrop API | HTTP call | Bank-account verification trigger |
| 8 | — (external) | Banned-words API | HTTP call, per free-text field | Fraud/profanity check |

### Flow C — Update (incl. prime-switch)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshPg | `GLUSR_BANK_DETAILS` | SELECT (1s timeout) | Fetch old account/IFSC/prime-status for change-comparison |
| 2 | meshPg | `GLUSR_BANK_DETAILS` | UPDATE (dynamic whitelist) | Apply the requested field changes |
| 3 | meshPg | `GLUSR_BANK_DETAILS` | UPDATE (async, only if new `bank_isprime=="1"`) | Un-prime every other account |
| 4 | meshPg | attachment table | UPSERT (async) | Sync attachment metadata |
| 5-8 | same as Flow A/B rows 5-8 | | | Same downstream consumer fan-out |

### Flow D — Soft delete

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshPg | `GLUSR_BANK_DETAILS` | UPDATE (`ENABLED=-1`) | Soft-delete the record; no other DB call in this model path |
| 2 | meshPg (consumer) | `GLUSR_USR` | SELECT — **skipped** | `USER_BANK_DETAILS.go` no-ops on `ENABLED`-present messages, so no mail/SMS DB read happens |

### Flow E — Banned-content detect & correct

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| — | none directly | — | — | Consumer itself makes no direct DB call — it calls the external banned-words API, then (on violation) re-invokes the write API, which triggers the full Flow A/C DB round-trips again |

### Flow F — Read

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | read-DB | `GLUSR_BANK_DETAILS` LEFT JOIN attachment table | SELECT | Return all bank-account rows + attachments for a supplier |

---

## 11. Optimization Scope — DB / Response-Time Contributors

### High impact

1. **SYSTEM-reqType insert pre-check runs with a 100ms timeout and fails open toward
   "prime"**: on any timeout/error, `count` stays `0` and the code treats it as "no existing
   prime account" — silently marking the new record prime. This isn't just a latency
   concern, it's a **correctness risk under load**: a slow query (DB under load, connection
   pool exhaustion) can cause multiple accounts to end up marked prime for the same user,
   defeating the entire "one prime" invariant this feature is built around.
   [`UserDetailsModel.go:502-532`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)
2. **Explicit prime-set triggers a second, async UPDATE query** (un-prime the rest) —
   unavoidable given no DB-level "only one true per group" constraint exists; a partial
   unique index (`WHERE GLUSR_BANK_ISPRIME=1`) at the DB level could enforce this invariant
   without a second round-trip, at the cost of turning a silent double-write into a
   constraint-violation error requiring app-level retry logic.

### Medium impact

3. **Update-path always does a pre-check SELECT before the UPDATE** (`oldBAcct`/`oldIfsc`/
   `wasPrime`) — needed to detect *changes* for the downstream mail/SMS-suppression and
   RabbitMQ payload comparison, but it's still a synchronous extra round-trip on every
   update request.
4. **`USER_BANK_DETAILS_TRUSTPG.go`'s fallback resync path does N sequential INSERTs**
   (one per row) when the primary UPDATE affects 0 rows — a full per-row-copy loop rather
   than a batched multi-row insert; on a user with many bank records this is a measurable
   extra-latency path specifically on drift-recovery.
5. **Soft-deleted-but-still-prime state has no cleanup path** — since delete doesn't clear
   `GLUSR_BANK_ISPRIME`, any downstream code that queries "the prime account" without also
   filtering `ENABLED<>-1` risks silently returning a disabled account; every consumer that
   reads prime-status should double check its `WHERE` clause includes the enabled filter.

### Low impact

6. **4-way RabbitMQ fan-out + external PennyDrop hop** — standard replication cost,
   consistent with GST/Logo/Negative-Mcat patterns already documented in this series, plus
   one more external-API hop (PennyDrop) than most other features.
7. **No Redis caching on the read side** — bank-detail reads are infrequent relative to
   writes-with-verification-pipeline, so this is lower priority than GST's equivalent gap,
   but still a gap if `otherdetail?type=BankDetails` read volume ever becomes significant.

---

## 12. Cron Inventory

No dedicated cron job touching `GLUSR_BANK_DETAILS` or any bank-details table was found.
Searched `user-temp-consumers-production`'s `internal/Workers` tree (and any separate
`crons/`-style directory) for `*BANK*` — only the four Workers already covered in this doc
(`USER_BANK_DETAILS.go`, `USER_BANK_DETAILS_TRUSTPG.go`, `USER_VERIFY_BANK_DETAILS.go`, plus
the shared `USER_PROFILE_BANNED_DETECT.go`) were found; all are queue-consumers, not
scheduled crons. Unlike GST (which has a daily BigQuery-driven TACT re-verification cron),
**Bank Details has no equivalent scheduled reconciliation job** confirmed in this pass — the
closest analog is the trust-sync consumer's row-copy fallback (`BankDetailsSync`), which is
event-triggered (on a failed UPDATE), not time-scheduled.

---

## 13. Edge Cases & Gotchas (technical POV)

1. **"Only one prime" is enforced at application-level, not DB-level** — a bug, a direct-DB
   write bypassing this controller, or the timeout fail-open in rule 7 could leave multiple
   `GLUSR_BANK_ISPRIME=1` rows for one user; no DB-constraint would catch this.
2. **`reqType=="SYSTEM"`'s auto-prime pre-check has a 100ms query-timeout and fails open
   toward "prime"** — notably tighter than most other queries in this codebase (which
   typically use 1 second); a slow query here silently results in the new record becoming
   prime rather than not-prime. Confirmed from code (section 5, rule 7) — this was an open
   question in the prior pass, now resolved.
3. **Trust-sync consumer's exact `SERVICENAME`/routing-key binding to `USER_BANK_DETAILS_TRUSTPG`
   was not conclusively confirmed** — application code declares the queue name but the
   RabbitMQ-level binding (routing key → queue) isn't visible in source; needs verification
   against the actual broker config/management UI.
4. **Soft-delete doesn't reset `GLUSR_BANK_ISPRIME`** — confirmed from code (no such column
   in the delete UPDATE statement, and no consumer resets it either); a "disabled but still
   prime" state is possible and any downstream logic reading "the prime account" must also
   filter on `ENABLED`.
5. **Possibly-dead SERVICENAME assignment**: `rabbitMQDataNew["SERVICENAME"]` is set to
   `"BANK_DETAIL_SERVICE"` but the actual publish call uses `rabbitMQData`, not
   `rabbitMQDataNew` — worth a runtime/log trace to confirm whether the verification-queue
   publish actually happens via this code path or a different one not found in this pass.
6. **Shared hardcoded `VALIDATION_KEY`** in `USER_PROFILE_BANNED_DETECT.go` used to
   self-call the write API for both `BankDetails` and `ContactDetails` — a single point of
   compromise if this key ever leaks, since it grants write access to two sensitive detail
   types.

---

## 14. Open Questions

1. What exactly is the RabbitMQ broker-level binding (routing key → queue name) for
   `user.verifybankdetails.*` on exchange `USER.topic` — does it bind to
   `USER_BANK_DETAILS_TRUSTPG`, `USER_VERIFY_BANK_DETAILS`, or both? Application code only
   declares queue names, not bindings.
2. Does `rabbitMQDataNew["SERVICENAME"]="BANK_DETAIL_SERVICE"`
   ([`UserDetailsModel.go:4185-4189`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go))
   actually take effect, given the subsequent `PushToQueue` call at line 4210 uses
   `rabbitMQData` not `rabbitMQDataNew`? If it's dead code, how does `USER_VERIFY_BANK_DETAILS`
   actually receive its messages in production?
3. `fk_gl_attribute_id IN (398,390)` — what do these two attribute-IDs represent
   specifically? No enum/constant/comment found in any of the three repos.
4. What happens to `bank_isprime` if the 100ms system-auto-prime-check query times out under
   real load — is the fail-open-to-prime behavior (confirmed from code) actually the
   intended design, or an unnoticed bug?
5. Is there truly no scheduled reconciliation job for bank-details prime-consistency or
   trust-sync drift, or does one exist outside the searched directories?
6. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai (table/column existence, types, nullability, indexes, constraints not
   cross-checked against a live schema).
7. Does `UserDetailController` (singular, `detail/*params`) or `SetUserDetailController`
   (`verification/setuserdetail/*params`) also expose/mutate bank details? Not verified in
   this pass — only `UserOtherDetailController` (`otherdetail/*params`) was confirmed.

---

## See also

- [`Bank_Details_Business_Doc.md`](./Bank_Details_Business_Doc.md) — product perspective
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — same shared
  write-controller this feature is a branch of
- [`../Fact Sheet KT/Fact_Sheet_Technical_Doc.md`](../Fact%20Sheet%20KT/Fact_Sheet_Technical_Doc.md) —
  another branch of the same shared controller
- [`../Trust Verification KT/Trust_Verification_Technical_Doc.md`](../Trust%20Verification%20KT/Trust_Verification_Technical_Doc.md) —
  related verification-domain, likely intersects via `USER_BANK_DETAILS_TRUSTPG.go` and
  `USER_VERIFY_BANK_DETAILS.go`
- [`../utils_programming_guide.md`](../utils_programming_guide.md) — shared
  `PushToQueue`/`PubAPI` helpers referenced in this doc

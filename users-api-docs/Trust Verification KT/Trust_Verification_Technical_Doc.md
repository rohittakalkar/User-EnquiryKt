# Trust Verification — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye
[`Trust_Verification_Business_Doc.md`](./Trust_Verification_Business_Doc.md) dekho.

**Repos**: `service-api-go-production` (write), `users-api-go-production` (read),
`user-temp-consumers-production` (consumers + loopback callers).

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file:line diya gaya
hai). Yeh pass GST KT ke depth-bar ko follow karta hai — pichle pass mein do controller-bodies
explicitly "not re-read" flag ki gayi thin (`UserVerificationController.go`,
`VerifiedDetailController.go`); dono ab **poori tarah re-read** ho chuki hain is pass mein,
saath IIL-verification write/read internals (`UserVerificationModel.go`, `VerifiedDetailModel.go`)
bhi. Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — team se confirm karo]** likha hai.

**Scope decision**: yeh generic per-attribute verification system hai (GST, bank, mobile,
email, PAN, CIN, TAN, address — koi bhi attribute). [`../TrustSeal KT/`](../TrustSeal%20KT/TrustSeal_Technical_Doc.md)
se **alag** rakha gaya kyunki koi direct DB-level link (`TRUSTSEAL` table se) nahi mila — sirf
conceptual/business-level relation hai.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Verification write (single/bulk attribute, action_flag-driven) | write | [`UserVerificationController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserVerificationController.go) — **poora re-read is pass mein** |
| Verification write — core business logic | write | [`UserVerificationModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationModel.go) (1185 lines) — **poora re-read is pass mein** |
| Mandatory-params validation (verification write) | write | [`UserUtilsMandatory.go:2499-2556`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go) (`MandatoryParamsCheckVerification`) |
| Trust-DB attribute-verification-details write (adjacent, separate path) | write | [`UserVerificationDetailsController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserVerificationDetailsController.go), [`UserVerificationDetailsModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go) — **poora re-read is pass mein** |
| Verification read (attribute-level, aggregate) | read | [`GetVerificationDetailController.go`](../../internal/controllers/UsersControllers/GetVerificationDetailController.go), [`GetVerificationDetailModel.go`](../../internal/models/users/GetVerificationDetailModel.go) — stored functions `fn_get_glusr_attr_verify`, `fn_get_glusr_attributes` |
| Verification read (detailed, verified-only view) | read | [`VerifiedDetailController.go`](../../internal/controllers/UsersControllers/VerifiedDetailController.go), [`VerifiedDetailModel.go`](../../internal/models/users/VerifiedDetailModel.go) — **dono poore re-read is pass mein** |
| Loopback verification helper (consumer-side, used by GST & others) | consumers | [`USER_APPROVAL_ATTRIBUTE.go:535-560`](../../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_APPROVAL_ATTRIBUTE.go) (`VerifyAttr` function) |
| Bulk / fan-out verification consumer | consumers | [`USER_BULK_VERIFICATION.go`](../../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BULK_VERIFICATION.go) — consumes `USER_VERIFICATION_BULK_HISTORY` |
| Trust-DB replica-sync consumer (loopback into Trust-DB write path) | consumers | [`USER_VERIFICATION_TRUSTPG.go`](../../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VERIFICATION_TRUSTPG.go) — **poora re-read is pass mein**, consumes `USER_VERIFICATION_TRUSTPG` / `_FAIL` |
| Verification-fan-out retry/requeue consumer | consumers | [`USER_UPSERT_VERIFICATION.go`](../../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_VERIFICATION.go) — **newly found is pass mein**, consumes `USER_UPSERT_VERIFICATION` |
| Supplier-verification-log reporting cron (adjacent, distinct purpose) | write repo `crons/` | [`Supp_verify_log_cron.go`](../../../service-api-go-production/service-api-go-production/crons/recommend/Supp_verify_log_cron.go) — **newly found is pass mein** |

---

## 2. Routes (confirmed from controllers)

| Method | Path | Repo | Controller | Service-name (Kibana) |
|---|---|---|---|---|
| POST | `/user/verification` | write | `UserVerificationController` | `USER_VERIFICATION_SERVICE` |
| POST | `/user/userverificationdetails` | write | `UserVerificationDetailsController` | `TRUST_ATTR_VERI_DETAILS` |
| GET/POST | `/getverificationdetails/*params` | read | `GetVerificationDetails` (calls `ActionGtVeriModel`) | — |
| GET/POST | `/verifieddetail/*params` | read | `VerifiedDetailController` | `USERVERIFIEDDETAIL` |

---

## 3. Data Model — Tables/Functions

> **Verification note**: yeh sab table/column names Go code ke andar embedded SQL strings se
> liye gaye hain (file:line diya hai har jagah). Live DB schema se pgAdmin pe cross-verify
> **nahi** kiya gaya hai — kisi migration/integration se pehle woh zaroor karo.

| Table/Function | Physical DB | Purpose | Confirmed from |
|---|---|---|---|
| `IIL_VERIFICATION_DETAILS` | meshpg | **Primary verification-state record** — ek row per (GLID, attribute, ref-id) — teen source-slots track karti hai: `IIL_VERIFICATION_USER_SRC_ID/DATE` (end-user/OTP), `IIL_VERIFICATION_TAC_SRC_ID/DATE` (tactical/system), `IIL_VERIFICATION_TV_SRC_ID/DATE` (a third, higher-priority slot — inferred "trusted verification"?) | [`UserVerificationModel.go:800-953`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationModel.go), [`VerifiedDetailModel.go:78-100`](../../internal/models/users/VerifiedDetailModel.go) |
| `IIL_VERIFICATION_SOURCE` | meshpg | Lookup: kis "process+screen" combination ka kya `SOURCE_ID`/`SOURCE_TYPE` (`U`=user/OTP-driven, `T`=tactical, `X`=unmatched-default) hai, aur uski priority | [`UserVerificationModel.go:819-822`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationModel.go) |
| `IIL_VERIFICATION_SOURCE_CAT` | meshpg | Source-category lookup (`SOURCE_CAT_ID`), name→ID resolve karne ke liye | [`USER_VERIFICATION_TRUSTPG.go:70`](../../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VERIFICATION_TRUSTPG.go) |
| `gl_attribute` | meshpg | Attribute-ID → physical `(table_name, column_name)` mapping — verification-write iske through pata karta hai kaunsi table/column mein submitted value ko live DB-value se cross-check karna hai | [`UserVerificationModel.go:74-104`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationModel.go) |
| `GL_HISTORY<NN>` (glid % 100 se sharded, 00-99) | meshpg | Audit-trail har verify/unverify action ka — old value, new value, verified-by details, transaction-type (`V`=verify/`W`=unverify-ish, inferred from code context) | [`UserVerificationModel.go:555-682`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationModel.go) |
| `glusr_usr_otp` | authpg | Mobile/email OTP-verification records — checked/cleared during verify & unverify flows; Android-shortcut (§6) directly inserts a synthetic row here | [`UserVerificationModel.go:246,755`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationModel.go) |
| `GLUSR_USR` | meshpg + authpg | Meshpg side: mobile/email lookup for cross-check (`glusr_usr_ph_mobile`, `glusr_usr_email`). Authpg side: `MOBILE_VERIFICATION_DATE`/`EMAIL_VERIFICATION_DATE` columns updated on verify/unverify for attrs 121/109 | [`UserVerificationModel.go:291,1109-1159`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationModel.go) |
| `fn_insert_glusr_attribute_verification(11 params)` | **trustpg** (stored function) | Adjacent write-path: inserts/upserts a Trust-DB attribute-verification-detail row for a primary attribute + optional secondary attributes | [`UserVerificationDetailsModel.go:257-284`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go) |
| `fn_get_glusr_attr_verify(glid)` | trustpg (stored function) | Ek supplier ke saare verified-attributes ka status return karta hai (JSON) | [`GetVerificationDetailModel.go:23`](../../internal/models/users/GetVerificationDetailModel.go) |
| `fn_get_glusr_attributes(glid)` | trustpg (stored function) | Ek supplier ke saare attributes (verified ya nahi, dono) return karta hai (JSON) | [`GetVerificationDetailModel.go:25`](../../internal/models/users/GetVerificationDetailModel.go) |
| `iil_verification_details` (mesh_pg_user, read-side) | mesh_pg_user | `VerifiedDetailController`/`VerifiedDetailModel` ka source — same logical table as `IIL_VERIFICATION_DETAILS` above, teen-way self-joined with `IIL_VERIFICATION_SOURCE` (user/tac/tv aliases) to resolve human-readable verifier-process/screen | [`VerifiedDetailModel.go:78-100`](../../internal/models/users/VerifiedDetailModel.go) |

**Two separate attribute-ID reference systems (confirmed, source of confusion)**:

1. **Main system** — `gl_attribute` table (meshpg), used by `UserVerificationController`
   write-path and both read-paths. GST KT doc documents `2106` = GST verification-source-ID in
   this system.
2. **Trust-DB attribute-details system** — a small, **hardcoded Go map**, used only by
   `UserVerificationDetailsModel.go`, unrelated numbering:

   | Raw attribute ID(s) | Group index | Meaning |
   |---|---|---|
   | 121, 48, 1293 | 1 | Mobile |
   | 157, 109, 1285 | 2 | Email |
   | 2721 | 3 | GST |
   | 348 | 4 | PAN |
   | 346 | 5 | CIN |
   | 347 | 6 | TAN |
   | 111 | 7 | Company Name |
   | 390 | 8 | Bank Account Number |
   | 2120 | 9 | Aadhar Number |
   | 112, 113, 1278, 1279 | 10 | Address |
   | 114 | 11 | City |
   | 115 | 12 | State |
   | 106, 108 | 13 | First/Last Name |
   | 352 | 14 | IEC code |
   | 141, 142 | 15 | CEO First/Last Name |
   | 117 | 16 | Zip |
   | 2882 | 17 | Udyam |
   | 2072, 2073 | 18 | Latitude-Longitude |

   [`UserVerificationDetailsModel.go:11-40`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go)

   **Important**: `2721` here (GST) is a *different number* from `2106` (GST's
   `FK_GL_ATTRIBUTE_ID` in the main `gl_attribute`/`IIL_VERIFICATION_DETAILS` system, per GST
   Technical Doc §3). These are two independent numbering schemes for two independent write
   paths — do not conflate them.

---

## 4. Trust-DB Attribute-Details Write — The Adjacent Path (§3 Flow E in Business Doc)

`POST /user/userverificationdetails` (`UserVerificationDetailsController.go`) is a **separate,
simpler write path** from the main `POST /user/verification` flow:

- Validates caller via `Gateway_v1([]string{"WEBERP", "SOA-WAPI"}, ...)` — a *narrower*
  whitelist than the main verification API's 24-caller list (§5).
  [`UserVerificationDetailsController.go:43`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserVerificationDetailsController.go)
- Requires `glid`, `status`, `primaryAttrID`, `primaryAttrVal`, optional `secondaryAttr` map,
  `source`, `platform`, `verification_type`, `datetime` (format `yyyyMMddHHmmss`).
  [`UserVerificationDetailsModel.go:57-170`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go)
- Validates `primaryAttrID` (and every `secondaryAttr` key) against the **Trust-DB group map**
  (§3, table above) — unknown IDs are rejected with `"primaryAttrVal invalid"` /
  `"secondaryAttrVal invalid"`.
  [`UserVerificationDetailsModel.go:196-206`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go)
- Writes **directly** to `trustpg` via `fn_insert_glusr_attribute_verification(...)` — no
  RabbitMQ publish happens in this path (unlike the main verification-write path, §7).
  [`UserVerificationDetailsModel.go:257-284`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go)

**Who calls it**: `USER_VERIFICATION_TRUSTPG.go` (the consumer for queue
`USER_VERIFICATION_TRUSTPG`/`_FAIL`) calls this exact endpoint as an internal HTTP loopback,
service-name `"user_trust_attr_veri_details"`, with a **hardcoded** `valdiation_key` =
`"18833e9703636bce1205043f022a91de"` — a *different* hardcoded key from the one used by
`VerifyAttr()` (§5, point 8). [`USER_VERIFICATION_TRUSTPG.go:91-107`](../../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VERIFICATION_TRUSTPG.go)

This confirms and extends what GST_Technical_Doc.md §5/§7 documented about the
`USER_VERIFICATION_TRUSTPG` queue: it is not just "a trust-DB replica-sync event" in the
abstract — concretely, it is consumed by a worker that **loops back into this exact
`/user/userverificationdetails` API**, which then calls `fn_insert_glusr_attribute_verification`
on `trustpg`. No contradiction with the GST doc's findings; this is the missing detail behind
that queue.

---

## 5. Main Verification Write — `UserVerificationController` / `UpsertVerification` (Business Rules & Validation)

1. **Mandatory-params gate**: `GLUSR_USR_ID` (numeric, required), `ATTRIBUTE_ID` (required),
   `action_flag` (required), `VALIDATION_KEY` (required, checked against a **24-entry
   whitelist**: `MY, BUYERMY, GLADMIN, Weberp, MAPI, LEAP, SELLERMY, BUYRCALL, FLPNS, LEAPIN,
   Localization, DIR, IMOB, FCP, PRODDTL, ETO, MDC, IMHOME, M.INDIAMART.COM, BOUNCE_MGMT, BI,
   PAYWIM, MERP, WHATSAPP, WA_9696`).
   [`UserUtilsMandatory.go:2499-2556`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
2. **`action_flag` drives four distinct code branches** in `updateMeshPg`:
   `SP_VERIFY_ATTRIBUTE` (verify), `SP_BULK_VERIFY_ATTRIBUTE` (bulk verify — enqueues to
   `USER_VERIFICATION_BULK_HISTORY` and returns, no direct meshpg write in this branch),
   `UNVERIFIED` (unverify/delete), and `SUPP_VERIFY` which is **explicitly dead** — returns
   the literal string `"not in use"`.
   [`UserVerificationModel.go:219-232`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationModel.go)
3. **Mobile(121)/Email(109) precondition**: if the submitted `ATTRIBUTE_ID` contains 121 or
   109, the corresponding value (mobile/email) must already exist on `GLUSR_USR` for that
   GLID, else the whole request fails with `"EMPTY VALUE FOR ATTRIBUTE_ID <id>"`.
   [`UserVerificationModel.go:53-68`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationModel.go)
4. **Live-value cross-check (multi-attribute submit)**: when multiple `ATTRIBUTE_ID`s (comma-
   separated) + `##`-separated values are submitted together, the code resolves each
   attribute-ID to its physical `(table, column)` via `gl_attribute`, fetches the current DB
   value, and compares it against the submitted value — mismatches accumulate into
   `errorString` (non-fatal, logged) rather than always hard-failing the request; **numeric
   mobile-type attributes** (121/48/1293) use a substring-containment comparison instead of
   exact match. [`UserVerificationModel.go:70-181`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationModel.go)
5. **Source-priority MERGE logic (non-"U" source types, i.e. tactical/system)**: uses a
   Postgres `MERGE INTO iil_verification_details` statement — only overwrites the higher
   "trusted verification" slot (`iil_verification_tv_src_id`) if the new source's priority is
   `> 1` **and** greater-or-equal to the *existing* `tv`-slot's registered priority; otherwise
   only the lower "tactical" slot (`iil_verification_tac_src_id`) is updated. This is a
   genuine priority-ranking/anti-downgrade rule.
   [`UserVerificationModel.go:893-953`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationModel.go)
6. **"U"-type sources (user/OTP-driven)** instead do a plain UPDATE-if-exists /
   INSERT-on-conflict-do-nothing against `iil_verification_user_src_id`/`_date` — no priority
   comparison needed since there's only one user-verification slot.
   [`UserVerificationModel.go:856-891`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationModel.go)
7. **Mobile-verification 1-year staleness flag**: if `attrId == "121"` and the existing
   `iil_verification_user_src_date` is `>= 365` days old at verify-time, a
   `ONE_YEAR_OLD` flag is set in the Kibana response payload (does not block the write, purely
   observability). The *read*-side (`VerifiedDetailModel.go`, §8 point 6) independently
   enforces the same 365-day rule to actually flip display-status to "Not Verified".
   [`UserVerificationModel.go:869-874`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationModel.go)
8. **Android auto-verify shortcut**: if `VERIFIED_BY_SCREEN` contains `"ANDROID"` (case-
   insensitive) and the attribute is 157 or 109 (both map to Email in the main flow), the code
   directly `INSERT`s a synthetic row into `glusr_usr_otp` with `user_verified_flag=1` — i.e.
   an Android-originated email-verify request auto-manufactures a matching OTP-style
   confirmation row without the user actually completing an OTP flow.
   [`UserVerificationModel.go:697-699,746-766`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationModel.go)
9. **Unverify (`UNVERIFIED` action_flag) only special-cases 8 attribute IDs for the
   OTP-check sub-flow**: `121, 109, 157, 48, 1293, 120, 156, 1294` — any other attribute ID
   skips the OTP-verified-flag check entirely and goes straight to the delete-and-history
   path. [`UserVerificationModel.go:461-479`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationModel.go)
10. **`VERIFIED_BY_ID`/`VERIFIED_BY_NAME`/`VERIFIED_BY_AGENCY`/`VERIFIED_BY_SCREEN`/
    `VERIFIED_IP`/`VERIFIED_COMMENTS` are all persisted to `GL_HISTORY<NN>`** — full audit
    trail of who/what/when/how for every verify action.
    [`UserVerificationModel.go:556-567,671-682`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationModel.go)
11. **Hardcoded `VALIDATION_KEY` used in the loopback caller (`VerifyAttr`)**:
    `"e27d039e38ae7b3d439e8d1fe870fc68"` — same value also appears in
    `verifyUser()` inside `UserVerificationModel.go:1020`, confirming this is a shared,
    internal-only "consumer loopback" credential used across multiple call-sites in this
    domain (distinct from the Trust-DB loopback key in §4).
    [`USER_APPROVAL_ATTRIBUTE.go:549`](../../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_APPROVAL_ATTRIBUTE.go),
    [`UserVerificationModel.go:1020`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationModel.go)
12. **AuthPG updates are scoped to attrId 109/121 only** — `updateAuthpg()` short-circuits for
    every other attribute ID, meaning `GLUSR_USR.MOBILE_VERIFICATION_DATE` /
    `EMAIL_VERIFICATION_DATE` are the *only* AuthPG-side columns this whole domain touches.
    [`UserVerificationModel.go:1030-1068`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationModel.go)

---

## 6. Read Paths — Business Rules

### 6a. `GET /getverificationdetails` (`ActionGtVeriModel`)

- Two modes via a `typ` param: `typ=="1"` calls `fn_get_glusr_attr_verify(glid)`, else calls
  `fn_get_glusr_attributes(glid)` — both are Postgres functions returning a JSON array, parsed
  in Go. [`GetVerificationDetailModel.go:12-46`](../../internal/models/users/GetVerificationDetailModel.go)
- No filtering/business logic happens in Go — all the "is this verified" logic lives inside the
  stored functions (**not traced in this pass — Open Questions**).

### 6b. `GET /verifieddetail` (`VerifiedDetailController` / `UserVerifiedDetail_v1`)

1. **Input validation** (all confirmed, `VerifiedDetailController.go:45-85`): token/glusrid/
   modid gateway checks via `CheckValidity`; `glusrid` length must be `<= 12` chars else
   rejected; `attribute_id` is mandatory; `userverified` if present must be numeric;
   `attribute_id == "-1"` is explicitly rejected as a sentinel/invalid value.
2. **`userverified` flag changes which verification-slot wins** when both a user/tv-source and
   a tac-source exist for the same attribute:
   - `userverified == "1"`: prefers `user`/`tv` source (whichever has the later date) for the
     primary displayed status; **falls back to empty** (not tac) if neither user/tv exists.
   - `userverified == "0"` (default): prefers `user`/`tv` if present, **falls back to tac
     source** if not.
   [`VerifiedDetailModel.go:172-234`](../../internal/models/users/VerifiedDetailModel.go)
3. **1-year mobile staleness re-confirmed at read-time**: for `int_attr == 121`, if
   `time_diff >= 31557600` seconds (365.25 days) since the winning verification date, a
   `flag_notverified = 1` is set which forces `status = "X"` (Not Verified) regardless of what
   the DB says. [`VerifiedDetailModel.go:190-197,275-279`](../../internal/models/users/VerifiedDetailModel.go)
4. **Multi-value attributes get array-shaped results**: attribute IDs `1293, 1294, 398, 390`
   (secondary mobile, secondary email(?), an unidentified 398, and Bank Account Number) are
   **not** collapsed to a single status — instead every `fk_gl_attribute_refid` row for that
   attribute produces its own entry in the response, keyed by ref-ID. All other attribute IDs
   get a single flat status. `398`'s meaning is **[INFERRED — confirm with team]**, not found
   documented elsewhere in this pass.
   [`VerifiedDetailModel.go:158,286`](../../internal/models/users/VerifiedDetailModel.go)
5. **Any attribute ID requested but not present in the query result, or non-numeric, gets an
   explicit "Not Verified" fallback response** with a distinct `Detail` message
   (`"attribute id is non-numeric"` vs `"Attribute does not relate to glusr_usr table"`) —
   never silently omitted. [`VerifiedDetailModel.go:487-500`](../../internal/models/users/VerifiedDetailModel.go)

---

## 7. RabbitMQ

All messaging in this domain is RabbitMQ-based (`RabbitEnqueueVerification` /
`PubAPI` / `CallWapiService` loopback helpers) — **no Kafka usage found** (§9).

| Queue | Publisher | Consumer | Purpose |
|---|---|---|---|
| `USER_VERIFICATION_BULK_HISTORY` | `UserVerificationModel.go` — from **all three** action-flag branches: `SP_BULK_VERIFY_ATTRIBUTE` (line 372,421), `SP_VERIFY_ATTRIBUTE` (line 732), and `UNVERIFIED` (line 607) | [`USER_BULK_VERIFICATION.go`](../../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BULK_VERIFICATION.go) (`UserBulkVerification`, registered in `router.go:113`) | **Misleadingly named** — despite the name, this single queue/consumer handles fan-out notification for verify, unverify, *and* literal bulk-verify events, not just "bulk" ones. It is the domain's one general-purpose "a verification changed" notify path. |
| `USER_VERIFICATION_TRUSTPG` / `USER_VERIFICATION_TRUSTPG_FAIL` | GST domain (`USER_GST_DETAILS.go`, `USER_GST_LAST_MODIFIED.go` — per GST Technical Doc §5/§7) and likely other attribute-verification producers | [`USER_VERIFICATION_TRUSTPG.go`](../../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VERIFICATION_TRUSTPG.go) (`UserVerificationTrustPg`) — **same worker function handles both the normal and `_FAIL` queue names**, confirmed in `router.go:119,124` | Loops back into `POST /user/userverificationdetails` (§4) to sync a verification-event into the Trust-DB attribute-details table. `_FAIL` is a retry-queue mapped to the identical handler. |
| `USER_UPSERT_VERIFICATION` | Same worker's own failure path (self-requeue) via `utils.PubAPI(...)`, and `pkg/utils/additional.go:217` (`CallWapiService`, generic helper — caller not traced) | [`USER_UPSERT_VERIFICATION.go`](../../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_VERIFICATION.go) (`insertInQueueUserUpsertVerification`, registered `router.go:53`) | A generic upsert/requeue worker (shared shape with other `USER_UPSERT_*` queues in this codebase) — for verification-domain messages that need retry-with-backoff semantics distinct from the immediate `_FAIL` queues. |
| `USER_APPROVAL_ATTRIBUTE` | Multiple producers across domains needing an "attribute approval" workflow (e.g. GST Matchmaking, per `pkg/utils/functions.go:1886-1914` error-logging context) | [`USER_APPROVAL_ATTRIBUTE.go`](../../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_APPROVAL_ATTRIBUTE.go) (`UserApprovalAttribute`, uses `approvalPg` connection) | A related-but-distinct attribute-approval workflow. **Its `VerifyAttr()` helper function (§5 point 11) is exported and called by other consumers as a plain Go function/HTTP-loopback utility — it is not itself triggered by this queue.** Full approval-workflow business logic (what triggers this queue, what `approvalPg` schema looks like) was **not traced in this pass — Open Questions**. |

---

## 8. Kafka

**No Kafka usage found** in any verification-domain file across all three repos in this pass
(`grep -li kafka` on all `UserVerification*.go`, `*VerifiedDetail*.go`, and the four consumer
files returned nothing). This is consistent with GST_Technical_Doc.md §8's finding that Kafka
in the broader user-domain is isolated to `USER_GST_DETAILS_BULK` specifically — Trust
Verification does not share that exception.

---

## 9. Redis

**No Redis usage found** in any verification-domain file in this pass — same gap GST KT
documented for its own domain (GST_Technical_Doc.md §9). All reads (`/getverificationdetails`,
`/verifieddetail`) hit live Postgres (meshpg/trustpg/mesh_pg_user) on every request.

---

## 10. End-to-End Technical Flows

### Flow A — Single-attribute verify (`action_flag = SP_VERIFY_ATTRIBUTE`)

```
Caller (internal tool / GLADMIN screen / consumer loopback)
    │
    ▼
[API] POST /user/verification  {GLUSR_USR_ID, ATTRIBUTE_ID, ATTRIBUTE_VALUE, VALIDATION_KEY, ...}
    │  UserVerificationController.go
    │  1. MandatoryParamsCheckVerification — gateway whitelist, required fields
    ▼
UpsertVerification()
    │  2. If VERIFIED_BY_ID present → resolve employee name (Employee_mesh_pg)
    │  3. Mobile(121)/Email(109) precondition check (must exist on GLUSR_USR)
    │  4. Multi-attribute cross-check against live DB value via gl_attribute mapping
    ▼
[DB — meshpg]  SP_IIL_VERIFY_MESHPG()
    │  - SELECT existing IIL_VERIFICATION_DETAILS row
    │  - SELECT IIL_VERIFICATION_SOURCE (resolve source-id/type/priority)
    │  - UPDATE/INSERT (user-type) or MERGE (tac/tv-type) into IIL_VERIFICATION_DETAILS
    │  - INSERT into GL_HISTORY<glid%100> (audit trail)
    ▼
[DB — authpg, only if attrId 109/121]  SP_IIL_VERIFY_AUTHPG()
    │  UPDATE GLUSR_USR SET MOBILE_VERIFICATION_DATE/EMAIL_VERIFICATION_DATE
    ▼
[RabbitMQ publish]  USER_VERIFICATION_BULK_HISTORY  (fan-out notify)
    ▼
[CONSUME]  USER_BULK_VERIFICATION.go — downstream replica-sync (meshPg + authPg connections)
```

### Flow B — Unverify (`action_flag = UNVERIFIED`)

```
Caller
    │
    ▼
[API] POST /user/verification  {action_flag: "UNVERIFIED", ATTRIBUTE_ID, ...}
    ▼
IIL_VERIFICATION_UNVERIFIED_MESHPG()
    │  - For OTP-tracked attrs (121/109/157/48/1293/120/156/1294): CheckVerifiedFlag() against
    │    glusr_usr_otp first
    │  - SELECT existing IIL_VERIFICATION_DETAILS row count
    │  - If exists: DELETE row, INSERT GL_HISTORY<NN> audit record (transaction_type "W")
    │  - If not exists: output = "No Record To Unverify" (treated as a success-equivalent by
    │    the controller, not an error — HTTP 200)
    ▼
IIL_VERIFICATION_UNVERIFIED_AUTHPG()  — only for attrId 109/121
    │  UPDATE GLUSR_USR SET MOBILE_VERIFICATION_DATE/EMAIL_VERIFICATION_DATE = NULL
    ▼
[RabbitMQ publish]  USER_VERIFICATION_BULK_HISTORY  (same fan-out queue as verify)
```

### Flow C — Bulk verify (`action_flag = SP_BULK_VERIFY_ATTRIBUTE`)

```
Caller
    │
    ▼
[API] POST /user/verification  {action_flag: "SP_BULK_VERIFY_ATTRIBUTE", ATTRIBUTE_ID (comma list), ...}
    ▼
SP_BULK_VERIFY_ATTRIBUTE_MESHPG()
    │  - SELECT IIL_VERIFICATION_SOURCE (resolve MySourceId/sourceCatId/priority)
    │  - If VERIFIED_BY_AGENCY == "MOBILE"/"EMAIL" and no match → hardcoded fallback source
    │    IDs 4865/4866, sourceCatId "11"
    │  - Loop over each attribute ID → SP_IIL_VERIFY_MESHPG() per attribute (same as Flow A's
    │    core write, called once per attribute in the list)
    ▼
[RabbitMQ publish]  USER_VERIFICATION_BULK_HISTORY  — one publish per bulk-call (aggregated
                     MessageData array of all processed attributes), not one per attribute
    ▼
[CONSUME]  USER_BULK_VERIFICATION.go
```

### Flow D — Trust-DB attribute-details write (adjacent path, §4)

```
Internal caller (e.g. USER_VERIFICATION_TRUSTPG consumer, looping back from an upstream
verification event in GST or another domain)
    │
    ▼
[API] POST /user/userverificationdetails  {glid, primaryAttrID, primaryAttrVal, status,
                                            platform, verification_type, source, datetime,
                                            secondaryAttr?}
    │  UserVerificationDetailsController.go
    │  1. Gateway_v1(["WEBERP","SOA-WAPI"]) check
    │  2. LengthAndTypeValidations_v3 against UserDetailsMap
    │  3. Validate primaryAttrID (+ secondaryAttr keys) against Trust-DB group map (§3)
    ▼
[DB — trustpg]  fn_insert_glusr_attribute_verification(11 params)
    │  One call for the primary attribute; one additional call per non-empty secondary
    │  attribute (each secondary written as its own primary-to-secondary mapping row)
    ▼
No RabbitMQ publish in this path — the loop ends here.
```

### Flow E — GST → Trust Verification cross-domain link (confirms GST KT §5/§7)

```
GST domain event (GST verified/tactical-verified, per GST_Technical_Doc.md Flow A)
    │
    ▼
[RabbitMQ publish]  USER_VERIFICATION_TRUSTPG   (published by USER_GST_DETAILS.go /
                                                  USER_GST_LAST_MODIFIED.go)
    ▼
[CONSUME]  USER_VERIFICATION_TRUSTPG.go
    │  For each message entry: resolve SOURCE_CAT_ID if only SOURCE name given; backfill
    │  PRIMARY_ATTR_VAL from GLUSR_USR if empty and attr is 121/109
    ▼
[HTTP loopback]  POST /user/userverificationdetails  (hardcoded VALIDATION_KEY, §4)
    ▼
[DB — trustpg]  fn_insert_glusr_attribute_verification(...)
```

### Flow F — Read: "Is this attribute verified?" (`VerifiedDetailController`)

```
Caller (buyer-facing profile page, internal tool, etc.)
    │
    ▼
[API] GET/POST /verifieddetail?glusrid=...&attribute_id=...&userverified=0|1
    ▼
[DB — mesh_pg_user]  SELECT iil_verification_details LEFT JOIN iil_verification_source x3
                      (user/tac/tv aliases) WHERE fk_glusr_usr_id=$1 AND fk_gl_attribute_id IN (...)
    ▼
UserVerifiedDetail_v1()
    │  - Per requested attribute: resolve winning source (user/tv vs tac) per userverified flag
    │  - Apply 365-day mobile staleness override
    │  - Multi-value attrs (1293/1294/398/390) → per-ref-id array; others → flat status
    ▼
Response: {Status: "Verified"/"Not Verified", Detail, Tac_verification_date, User_verification_date, Tac_Detail}
```

---

## 11. Flow-wise DB & Table Usage — Kaun sa DB, Kaun sa Table, Kis Liye

### Flow A — Single-attribute Verify

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | (employee lookup, exact table not traced) | SELECT | `Employee_mesh_pg()` — resolve `VERIFIED_BY_NAME` from `VERIFIED_BY_ID` if given |
| 2 | meshpg | `GLUSR_USR` | SELECT | `FetchMobEmail()` — precondition check for attrs 121/109 |
| 3 | meshpg | `gl_attribute` | SELECT | Resolve attribute-ID(s) → `(table, column)` for the live-value cross-check |
| 4 | meshpg | Dynamic (per `gl_attribute` result — e.g. `GLUSR_USR`, `GLUSR_USR_ADDT_CONTACT`, `GLUSR_BANK_DETAILS`) | SELECT | Fetch live value to cross-check against submitted `ATTRIBUTE_VALUE` |
| 5 | meshpg | `IIL_VERIFICATION_DETAILS` | SELECT (`COUNT`) | Check existing verification row for this (glid, attrId, refId) |
| 6 | meshpg | `IIL_VERIFICATION_SOURCE` | SELECT | Resolve source-ID/type/priority for the `(VERIFIED_BY_AGENCY, VERIFIED_BY_SCREEN)` pair |
| 7 | meshpg | `IIL_VERIFICATION_DETAILS` | UPDATE / INSERT / MERGE | The actual verification write (§5 points 5-6) |
| 8 | meshpg | `IIL_VERIFICATION_DETAILS` | SELECT | Re-fetch the just-written row (`outMesh`) for downstream RabbitMQ payload construction |
| 9 | meshpg | `SEQ_GL_TRANSACTION_ID` (sequence) | SELECT NEXTVAL | Generate a transaction ID for the audit-history insert |
| 10 | meshpg | `GL_HISTORY<glid%100>` | INSERT | Audit-trail row (old/new value, verifier identity, timestamp) |
| 11 | authpg | `glusr_usr_otp` (conditional, Android+email shortcut) | INSERT | Synthetic OTP-verified row (§5 point 8) |
| 12 | authpg | `GLUSR_USR` | UPDATE (only attrId 109/121) | `MOBILE_VERIFICATION_DATE`/`EMAIL_VERIFICATION_DATE` |

**Total DB round-trips for a single verify call: 8-12 depending on attribute type and branch**
(fewer if attrId isn't 109/121, since step 12 and the Android shortcut are skipped) — all on 2
physical databases (meshpg, authpg).

### Flow B — Unverify

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg (conditional, OTP-tracked attrs only) | `glusr_usr_otp` | SELECT + conditional DELETE | `CheckVerifiedFlag()` — clear stale OTP record before unverify |
| 2 | meshpg | `IIL_VERIFICATION_DETAILS` | SELECT (`COUNT`) | Confirm a record exists to unverify |
| 3 | meshpg | `IIL_VERIFICATION_DETAILS` | DELETE | Remove the verification row |
| 4 | meshpg | `SEQ_GL_TRANSACTION_ID` | SELECT NEXTVAL | Transaction ID for audit row |
| 5 | meshpg | `GL_HISTORY<glid%100>` | INSERT | Audit-trail row for the unverify |
| 6 | authpg | `GLUSR_USR` | UPDATE (attrId 109/121 only) | Clear `MOBILE_VERIFICATION_DATE`/`EMAIL_VERIFICATION_DATE` |

### Flow C — Bulk Verify

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `IIL_VERIFICATION_SOURCE` | SELECT | Resolve source for the whole batch |
| 2..N | meshpg | *(Flow A steps 5-10 repeated per attribute in the list)* | — | Each attribute in the comma-list gets its own full verify-write cycle inside the loop |

**Business-critical insight**: a "bulk verify" of K attributes triggers roughly `K × 4-5` DB
round-trips sequentially inside a single HTTP request (no parallelism observed in this loop) —
worth checking for timeout risk on large K.

### Flow D — Trust-DB Attribute-Details Write

| # | DB | Table/Function | Operation | Kyun |
|---|---|---|---|---|
| 1 | trustpg | `fn_insert_glusr_attribute_verification` | Function call (SELECT-wrapped) | Primary attribute upsert |
| 2..N | trustpg | `fn_insert_glusr_attribute_verification` | Function call, once per non-empty secondary attribute | Each secondary attribute becomes its own function-call row |

### Flow F — Read (`/verifieddetail`)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | mesh_pg_user | `iil_verification_details` LEFT JOIN `iil_verification_source` (x3 aliases) | SELECT | Single query fetches all requested attributes + resolves all three source-slots' process/screen/type in one round-trip — **this is a positive finding**, no N+1 pattern here unlike some other domains |

### Flow F' — Read (`/getverificationdetails`)

| # | DB | Table/Function | Operation | Kyun |
|---|---|---|---|---|
| 1 | trustpg | `fn_get_glusr_attr_verify` or `fn_get_glusr_attributes` | Function call | Single round-trip, all logic pushed into the stored function |

---

## 12. Optimization Scope — DB Response-Time Contribution

### High-impact

1. **Bulk-verify loop is fully sequential, no parallelism** (§11 Flow C) — each attribute in a
   bulk request runs its full 4-5-query cycle one after another inside the same HTTP request.
   For a caller submitting many attributes at once, this scales linearly and risks request
   timeouts. **Suggestion**: parallelize per-attribute writes with goroutines (each attribute
   is logically independent — different `attrId`, same `glid`), similar to the pattern GST's
   `USER_GST_LAST_MODIFIED.go` already uses successfully (per GST_Technical_Doc.md §12 point 7).
   [`UserVerificationModel.go:382-398`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationModel.go)

2. **Single-attribute verify does up to 4 sequential SELECTs before the actual write**
   (employee lookup, mobile/email precondition, `gl_attribute` lookup, dynamic live-value
   cross-check) — none of these are cacheable as written (they depend on submitted params),
   but the `gl_attribute` table lookup (§11 Flow A step 3) is a static reference-table query
   that runs on **every single verification call** and could be cached in-process (it rarely
   changes — attribute-ID-to-column mappings are schema-level, not data-level).
   [`UserVerificationModel.go:74`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationModel.go)

### Medium-impact

3. **Two loopback HTTP hops exist in the GST→Trust sync chain** (Flow E): GST consumer →
   RabbitMQ → `USER_VERIFICATION_TRUSTPG` consumer → HTTP loopback →
   `UserVerificationDetailsController` → `trustpg`. Each hop adds latency and a failure-point.
   This mirrors the pattern already flagged in GST_Technical_Doc.md §12 point 1 (external/
   internal network calls in the critical path) — same risk class, worth the same fix
   (dedicated worker-pool / stricter timeout for this consumer specifically).

4. **No caching on either read endpoint** (§9) — `/verifieddetail` and
   `/getverificationdetails` hit live Postgres every time. Verification-state changes
   relatively infrequently per (glid, attribute) pair (it's either set once by a verify event
   or cleared once by an unverify event) — a short-TTL cache-aside pattern would likely be
   safe and effective here, same finding as GST KT §12 point 4.

5. **`GL_HISTORY<NN>` sharded-table pattern (100 shards by `glid % 100`) is good design for
   write-distribution**, but the shard-selection logic (`utils.FormatToInt(glid) % 100`,
   zero-padded) is **duplicated inline** in at least two places
   (`UserVerificationModel.go:550-554` and `:666-670`) rather than extracted to a shared
   helper — low-risk but a maintenance/consistency gap if the sharding scheme ever changes.

### Low-impact

6. **`errorString` accumulation pattern in `UpsertVerification`/cross-check logic (§5 point 4)
   does not always fail the request on mismatch** — value-mismatches are appended to
   `errorString` and surfaced in the Kibana log (`ERROR_STRING` field) but don't necessarily
   block the write. This is a design choice, not strictly a performance issue, but worth
   flagging: a caller might believe "the value I submitted was validated" when in practice a
   mismatch was only logged, not enforced. Confirm with team whether this is intentional
   (§14 Open Questions).

---

## 13. Cron Inventory

| Cron | Live hai? | Trigger | Kya karta hai |
|---|---|---|---|
| `Supp_verify_log_cron.go` / `.sh` | **Haan, live** (newly found in this pass) | External scheduler (shell wrapper, `crons/recommend/`) | Runs several sub-processes against `meshpg`+`mainpg` (`processCassUpdateMain`, `processPNSDefaulterRejection`, `processReceiptUsers`, `processCountryRejection`, plus `cntBeforeReceipt`/`cntAfterReceipt` counters) and mails a summary via Google Chat webhook (`SendMessageToGoogleSpace`). **This is a supplier-verification *reporting/reconciliation* cron, distinct from the attribute-verification write-path documented above** — its sub-processes were only partially traced (`processCassUpdateMain` reads `GLUSR_USR_CASS_UPDATE_MAIN`); several sub-processes referenced in comments are explicitly disabled (`processDataCopy`, `processC2CVerification`, `processHotLeadVerification`, `processFreeVerified`, `processQGFCPVGFCP` — all commented out in `populateSuppVerificationData`). **[INFERRED scope — exact business purpose of each sub-process not fully traced, confirm with team before relying on this for anything beyond "it runs and mails a report".]** |

No other cron touching `IIL_VERIFICATION_DETAILS`, `gl_attribute`, or the Trust-DB attribute
functions was found in this pass across all three repos.

---

## 14. Edge Cases & Gotchas (technical POV)

1. **Two independent write paths exist for "verification"** — the main
   `POST /user/verification` (writes `IIL_VERIFICATION_DETAILS` on meshpg + selectively
   authpg) and the adjacent `POST /user/userverificationdetails` (writes directly to trustpg
   via a stored function, §4). They use **different attribute-ID numbering systems** (§3).
   Debugging "why doesn't X look verified in system Y" requires knowing which of the two paths
   Y actually reads from.
2. **`USER_VERIFICATION_BULK_HISTORY` is a misleading queue name** (§7) — it is the general
   verify/unverify/bulk-verify fan-out queue, not exclusively for bulk operations. Anyone
   reading queue-traffic dashboards should not assume high volume here implies bulk-API usage
   specifically.
3. **`SUPP_VERIFY` action_flag is dead code** (`"not in use"`, §5 point 2) — if any caller is
   still sending this flag, their requests are silently no-ops from the DB's perspective while
   still returning without an explicit failure at the controller level (worth confirming no
   live caller still sends it).
4. **The Android email-verify shortcut manufactures OTP records without an actual OTP**
   (§5 point 8) — if OTP-integrity is ever audited, this code path is a legitimate,
   intentional exception, not a bug, but it should be documented for auditors.
5. **Mobile-verification staleness is enforced in two different places with two different
   thresholds expressed differently** — write-side uses `>= 365` days
   (`UserVerificationModel.go:871`), read-side uses `>= 31557600` seconds
   (`VerifiedDetailModel.go:194`), which is 365.25 days (accounts for leap years). These are
   *not* exactly the same threshold — a very minor drift (~6 hours) between the two checks.
   Low practical impact, but worth normalizing to one shared constant.
6. **`IIL_VERIFICATION_DETAILS` has three source-slots (user/tac/tv), not two** — code that
   only checks `iil_verification_user_src_id` and `iil_verification_tac_src_id` (as older
   integrations might) will miss the `tv` slot, which the current write-path (§5 point 5) can
   populate independently. `tv` = "trusted verification" is an **[INFERRED — confirm with
   team]** expansion of the abbreviation; not documented anywhere in code comments.
7. **Attribute `398` appears in the read-side multi-value special-case list
   (`VerifiedDetailModel.go:158`) but was not found defined/explained anywhere else in this
   pass** — flagged as **[INFERRED — confirm with team]** rather than guessed.
8. **Hardcoded loopback credentials differ by call-site** — `VerifyAttr()` and `verifyUser()`
   share `"e27d039e38ae7b3d439e8d1fe870fc68"`, while `USER_VERIFICATION_TRUSTPG.go` uses a
   *different* hardcoded key `"18833e9703636bce1205043f022a91de"` for the Trust-DB loopback.
   Two separate rotation-risk surfaces, not one, contrary to what might be assumed from the
   GST KT doc's single-key framing (GST_Technical_Doc.md §5 point 3, §14 point 3 — that doc
   only observed the `e27d...` key; this pass found a second, distinct key used specifically
   for the Trust-DB path).

---

## 15. Open Questions

1. What do `fn_get_glusr_attr_verify` and `fn_get_glusr_attributes` (trustpg stored functions)
   actually compute internally? Not traced in this pass — only their call-signature and return
   shape (JSON array) are confirmed from Go code.
2. What does `iil_verification_tv_src_id`/`_date` ("tv" slot) formally stand for, and under
   what business scenario does a `T`-type source populate it vs the `tac` slot only? The MERGE
   logic (§5 point 5) implies a priority-based hierarchy but the *business meaning* of "tv" vs
   "tac" was not documented in code comments.
3. Attribute ID `398` (VerifiedDetailModel.go multi-value list) — meaning unknown, not found
   in either attribute-ID reference system documented in §3.
4. `USER_APPROVAL_ATTRIBUTE` queue's own consumer logic (`UserApprovalAttribute` /
   `dbActionUserApprovalAttribute`, `approvalPg`-backed) was not traced beyond its `VerifyAttr`
   exported helper — what business workflow actually triggers messages *into* this queue, and
   what does `approvalPg` schema look like?
5. `USER_UPSERT_VERIFICATION`'s full retry/backoff semantics and every producer that targets
   it via `pkg/utils/additional.go:217` were not fully traced — only the self-requeue path
   inside `USER_UPSERT_VERIFICATION.go` itself was confirmed.
6. `Supp_verify_log_cron.go`'s sub-processes (`processCassUpdateMain`,
   `processPNSDefaulterRejection`, `processReceiptUsers`, `processCountryRejection`) were found
   but not deeply traced table-by-table — confirm exact business purpose and whether this cron
   is still actively monitored (several sibling sub-processes are commented out, suggesting
   partial deprecation).
7. Does a mismatch in the live-value cross-check (§5 point 4) ever hard-fail a verification
   request, or is it always just logged to `ERROR_STRING`? The code path suggests the latter
   in most cases but this was not exhaustively confirmed for every branch.
8. Live DB schema verification (column types, nullability, indexes, constraints) for every
   table listed in §3 — this doc reflects only what Go SQL strings imply.

---

## See also

- [`Trust_Verification_Business_Doc.md`](./Trust_Verification_Business_Doc.md) — product
  perspective
- [`../TrustSeal KT/TrustSeal_Technical_Doc.md`](../TrustSeal%20KT/TrustSeal_Technical_Doc.md) —
  related, separate premium trust-badge system
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — GST verification as a
  concrete example of this generic system; `USER_VERIFICATION_TRUSTPG`/`_FAIL` and
  `USER_VERIFICATION_BULK_HISTORY` are shared, cross-referenced queues (this doc's §4/§7/Flow E
  fill in the consumer-side detail that GST KT's §5/§7 referenced but did not expand)
- [`../Rating Usefulness KT/Rating_Usefulness_Technical_Doc.md`](../Rating%20Usefulness%20KT/Rating_Usefulness_Technical_Doc.md) —
  same HTTP-loopback architecture pattern (consumer → internal API call), useful comparison

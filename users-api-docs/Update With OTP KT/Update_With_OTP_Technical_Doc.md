# Update With OTP — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`Update_With_OTP_Business_Doc.md`](./Update_With_OTP_Business_Doc.md)
dekho — dono docs same flow cover karte hain, bas alag audience ke liye.

**Scope note**: yeh feature GST/Bank-Details jaisa multi-branch shared controller nahi hai —
apna **dedicated single-purpose controller** hai jiska **koi DB-write khud nahi hai**. Yeh ek
**two-factor OTP verification-gate + HTTP-loopback** hai: do sequential OTPs verify karta hai
`glusr_usr_otp` (authpg) ke against, phir asli update `UserUpdateController` (`POST
/user/update`, `SERVICENAME=GLUSR_UPDATE_SERVICE`) ko internally call karke trigger karta hai —
same architectural pattern jo Rating Usefulness aur Trust Verification KTs mein already dekha
gaya hai.

**Repos**: `service-api-go-production` only (controller + model). OTP *generation*/SMS-sending
mechanism iss review mein **kahin nahi mila** teeno repos (`users-api-go-production`,
`service-api-go-production`, `user-temp-consumers-production`) mein `glusr_usr_otp` grep karne
ke baad bhi — sirf 3 files touch karte hain is table ko, sab verification/read-side
(`UpdateWithOtpController.go`, `UpdateWithOtpModel.go`, `UserVerificationModel.go`). OTP
generate/send karne wala code is review ke scope se bahar hai — **[INFERRED — likely a
separate SMS/email-gateway service, confirm with team]**.

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file path diya
gaya hai har jagah). Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — team se confirm karo]** likh diya hai, guess nahi kiya.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Controller | write | [`UpdateWithOtpController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UpdateWithOtpController.go) (`UserUpdateWithOTPController`) |
| Model | write | [`UpdateWithOtpModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UpdateWithOtpModel.go) (`GetOtp`, `TriggerRequest`) |
| Router registration | write | [`router.go`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) — `POST /user/updatewithotp` registered twice, line 160 and 328 (two route-groups, same handler) |
| RabbitMQ `SERVICENAME`→routing-key/exchange map | write | [`rabbitmq.go`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go) — `UPDATEWITHOTP` → `user.additional.*` (line 31), exchange `USER.topic` (line 67) |
| Downstream target — actual update controller (`user_update_api`) | write | [`UserUpdateController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserUpdateController.go) (`UserUpdateController`, `SERVICENAME="GLUSR_UPDATE_SERVICE"`), registered at `POST /user/update` — [`router.go:108,140,370`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) |
| `user_update_api` config value (env-specific) | write | [`config.yml`](../../service-api-go-production/service-api-go-production/data/config.yml) — e.g. line 687 (prod): `http://service.intermesh.net/user/update` |
| Related OTP-table read/write (different feature, same table) | write | [`UserVerificationModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationModel.go) — `CheckVerifiedFlag` reads/deletes `glusr_usr_otp` (line 246,259,267), `markFlag` inserts a row without an OTP code (line 755) — this is a **separate verification flow**, not called by Update-With-OTP, listed here only because it's the only other code touching the same table |

**Koi dedicated consumer nahi mila for `UPDATEWITHOTP`** — searched `user-temp-consumers-production`
for `UPDATEWITHOTP` and for the routing key `user.additional.*` / `USER_ADDITIONAL`; zero hits.
See section 6 for the implication.

---

## 2. Routes

| Method | Path | serviceName | Repo | Controller |
|---|---|---|---|---|
| POST | `/user/updatewithotp` | `UPDATEWITHOTP` | write | `UserUpdateWithOTPController` |

Registered identically in two route-groups (`router.go:160` and `router.go:328`) — same handler,
no version/gate distinction found between the two registrations in this pass.

---

## 3. Data Model — Tables

> **Verification note**: yeh table/column names Go code ke andar embedded SQL strings se liye
> gaye hain. Live DB schema se pgAdmin pe cross-verify **nahi** kiya gaya hai.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `glusr_usr_otp` | authpg | OTP lookup-store (read-only from this controller) | `glusr_usr_value` (mobile/email, matched case-insensitively via `lower()`), `glusr_usr_otp_code` — [`UpdateWithOtpModel.go:26`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UpdateWithOtpModel.go) |

**No write to any table happens in this controller.** The actual `GLUSR_USR` (or whichever
attribute) update is delegated entirely to a downstream HTTP call
(`UserUpdateController`/`user_update_api`) which has its own, separate DB-write path — not
traced in this pass (see Open Questions).

**Note on `glusr_usr_otp` schema breadth**: `UserVerificationModel.go:246` selects `*` from this
table and references `user_verified_flag` and `glusr_usr_misscall_otp_code` columns — meaning
the table has more columns than the two this feature reads (`glusr_usr_value`,
`glusr_usr_otp_code`). This feature only touches the two columns it needs.

---

## 4. Decode: `OTP1_ATTR` / `OTP2_ATTR` — the "which field is being verified" mechanism

There's no numeric status-code enum in this feature (unlike GST's `FK_GST_VERIFICATION_SRC_ID`).
The one "magic value" worth decoding is the **hardcoded `attributeArray` map** that determines
whether an OTP-lookup value is treated as mobile or email:

| Attribute key | `attribute_id` | Meaning (inferred from key name) |
|---|---|---|
| `EMAIL` | `109` | Primary email |
| `EMAIL_ALT` | `157` | Alternate email |
| `PH_MOBILE` | `121` | Primary mobile |
| `PH_MOBILE_ALT` | `48` | Alternate mobile |

[`UpdateWithOtpController.go:112-117`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UpdateWithOtpController.go)

**Cross-reference note**: these exact 4 attribute IDs (`109`, `157`, `121`, `48`) also appear in
the GST domain's daily TACT cron filter list (`{120,121,48,156,157,109,1293,1294,2074,1285}` —
[`GST_Technical_Doc.md` Flow C](../GST%20KT/GST_Technical_Doc.md)) — strongly suggesting these
are stable, shared, cross-domain attribute IDs (likely from a `GL_ATTRIBUTE_MASTER` table), not
something unique to this feature. Their human-readable meaning (mobile/email) is confirmed
directly by the key names in the code, so this one is **not** flagged INFERRED — unlike GST's
`FK_GST_VERIFICATION_SRC_ID`, there's no ambiguity here.

**Format**: `OTP1_ATTR`/`OTP2_ATTR` values are `"<value>$$<attribute_id>"` strings, split on
`$$`. Only `OTP1_ATTR`'s `attribute_id` half is actually used (to resolve mob-vs-email for the
Step-1 lookup) — `OTP2_ATTR`'s attribute-id half is never parsed/used; only the whole
`OTP2_ATTR` string is compared against `key_val` (see rule 4 below).
[`UpdateWithOtpController.go:138-166`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UpdateWithOtpController.go)

---

## 5. Business Rules & Validation (code se, exhaustive)

1. **Mandatory fields, checked inline (no shared mandatory-fields helper)**: `USR_ID`
   (numeric), `UPDATEDBY`, `UPDATEDUSING`, `VALIDATION_KEY`, `IP`, `IP_COUNTRY` — missing any
   → `412`, `"Please enter all required fields..."`.
   [`UpdateWithOtpController.go:121-128`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UpdateWithOtpController.go)
2. **OTP fields mandatory, checked separately with a different error code**: `OTP1`, `OTP2`,
   `OTP1_ATTR`, `OTP2_ATTR` presence → `403` `"OTP validation keys are mandatory"` if any key
   absent; then a second check for blank-but-present values → `403` `"OTP validation keys
   cannot be blank"`.
   [`UpdateWithOtpController.go:129-136`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UpdateWithOtpController.go)
3. **`UPDATEDBY_ID` defaults to `"-1"`** if not supplied — not treated as a validation failure,
   just a silent default. [`UpdateWithOtpController.go:66-70`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UpdateWithOtpController.go)
4. **No `Gateway_v1`/allowlist-check found anywhere in this controller** —
   `VALIDATION_KEY` is required as non-empty but is **never validated against any modid
   allowlist**, unlike nearly every other write-controller documented across this KT series
   (GST's `checkScreenPermisson`, Bank Details' gateway-allowlist, etc.). See Open Questions.
5. **Step 1 — OTP1 verification, hard-gated, sequential**: `OTP1_ATTR` split on `$$` →
   `(otp1Attr value, attribute_id)`; `attribute_id` resolved against `attributeArray`
   (section 4) to decide mob-vs-email; `GetOtp(mob/email, "OTP1")` fetches
   `glusr_usr_otp_code` for that value (case-insensitive match); compared to `OTP1`:
   - lookup returns empty → `500`, `"OTP1 not Found"`
   - mismatch → `204`, `"OTP1 not Verified"` — **Step 2 never runs**
   - match → proceed
   [`UpdateWithOtpController.go:185-256`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UpdateWithOtpController.go)
6. **After Step 1 succeeds (before Step 2 even starts), a RabbitMQ message is pushed
   unconditionally** — `SERVICENAME=UPDATEWITHOTP`, `TABLES=GLUSR_USR`, `ACTION=UPDATE`,
   payload is metadata only (`UPDATEDBY`/`UPDATEDBY_ID`/`UPDATEDUSING`/`IP`/`IP_COUNTRY`/
   `HIST_COMMENTS` + `att_id`/`old_values` in `OTHERS`) — **no actual new field-value is in
   this payload**, and it fires **regardless of whether OTP2 later succeeds or fails**. Looks
   like an audit/history-log push, not the real data-update.
   [`UpdateWithOtpController.go:206-249`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UpdateWithOtpController.go)
7. **Step 2 target value (`key_val`) is discovered by scanning the whole request**, not read
   from a named field: `inputParams` is iterated, and the first key that exists in
   `attributeArray` (i.e. literally `EMAIL`, `EMAIL_ALT`, `PH_MOBILE`, or `PH_MOBILE_ALT` as a
   top-level request key) becomes `key_val` — the new value the supplier is trying to set. If
   the request has **more than one** of these keys present, only the last one iterated wins
   (Go map iteration order is non-deterministic) — **potential ambiguity if a caller ever sends
   multiple attribute keys in one request**, not explicitly guarded against.
   [`UpdateWithOtpController.go:199-204`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UpdateWithOtpController.go)
8. **Step 2 — OTP2 verification against `key_val`, not `OTP1_ATTR`'s value**:
   `GetOtp(key_val, email, "OTP2")` — note `email` here is still the **Step-1-resolved** email
   variable (only populated if OTP1's attribute was EMAIL/EMAIL_ALT), so if OTP1 was a mobile
   attribute, this call passes `key_val` as the mobile-position argument and an empty email
   (`GetOtp` picks whichever of `mob`/`email` is non-empty — see `UpdateWithOtpModel.go:18-24`).
   - lookup returns empty → `500`, `"OTP2 not Found"`
   - mismatch → `204`, `"OTP2 not Verified"`
   - match, but `key_val != otp2_attr` → `204`, `"OTP2 Attribute value does not match with
     update param"` — the OTP2 code matched *some* record, but the value it was generated for
     doesn't match what's actually being submitted, so it's rejected anyway.
   - match, and `key_val == otp2_attr` → **proceed to `TriggerRequest`**
   [`UpdateWithOtpController.go:257-286`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UpdateWithOtpController.go)
9. **`TriggerRequest()` strips OTP-specific fields and loops back into the real update API**:
   removes `OTP1`, `OTP1_ATTR`, `OTP2`, `OTP2_ATTR`, `token`, `modid` from the payload, then
   `POST`s everything else to `config.GetConfig().APIList["user_update_api"]` — resolved to
   `POST /user/update` → `UserUpdateController` (`SERVICENAME="GLUSR_UPDATE_SERVICE"`), the
   **standard profile-update endpoint** used by the rest of the system (e.g.
   [`PnsTxnModel.go:275`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PnsTxnModel.go),
   [`UserAddUpdateUtils.go:2966`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddUpdateUtils.go)
   call the same config key elsewhere). This confirms the doc's earlier open question — the
   downstream controller is `UserUpdateController`, not an unnamed black box.
   [`UpdateWithOtpModel.go:63-93`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UpdateWithOtpModel.go)
10. **Downstream-response translation**: `TriggerRequest` checks `jsonData["CODE"] != "200"`
    from `UserUpdateController`'s response — on non-200 it re-wraps as its own `500`
    `Failure` with `MESSAGE` pulled from `ERR_MSG.REASON` if present, else the raw response
    body; on `200` it re-wraps as `SUCCESS`. So a downstream validation failure in
    `UserUpdateController` (e.g. a field-level rule failing) surfaces to the OTP-caller as a
    generic `500`, not the original error code.
    [`UpdateWithOtpModel.go:105-129`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UpdateWithOtpModel.go)
11. **Response payload always includes both raw OTP codes fetched from the DB**
    (`Otpredis1`, `Otpredis2` — the *actual* `glusr_usr_otp_code` values looked up, not the
    supplier-submitted ones) alongside the submitted `Otp1`/`Otp2` and the resolved
    `Keyredis1`/`Keyredis2` (mob/email values used for each lookup) — **in every response**,
    success or failure. [`UpdateWithOtpController.go:355-360`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UpdateWithOtpController.go)
    — see Edge Cases for why this is worth flagging.

---

## 6. RabbitMQ

| Queue / `SERVICENAME` | Routing key | Exchange | Publisher | Consumer | Purpose |
|---|---|---|---|---|---|
| `UPDATEWITHOTP` | `user.additional.*` | `USER.topic` | `UpdateWithOtpController.go` — fires **right after OTP1 succeeds**, regardless of OTP2 outcome | **not conclusively found** — see below | Metadata-only audit event: `TABLES=GLUSR_USR, ACTION=UPDATE`, `att_id`/`old_values` in `OTHERS`, no actual new-value in payload |

[`rabbitmq.go:31,67`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go)
confirms the routing key: `UPDATEWITHOTP` maps to `user.additional.*` on exchange `USER.topic`.

**New finding vs. prior doc pass**: this is the **same routing key** (`user.additional.*`) used
by `USER_DETAILS_SERVICE` (line 54 of the same map) — a different, more general `SERVICENAME`.
Since RabbitMQ topic-exchange routing is by routing-key pattern, messages published under
`UPDATEWITHOTP` and under `USER_DETAILS_SERVICE` land in the **same queue(s)** bound to
`user.additional.*`, if such a binding exists. However, **no consumer file in
`user-temp-consumers-production` was found that references either `UPDATEWITHOTP` or the string
`user.additional`/`USER_ADDITIONAL`** — grepped both directly. This means either:
- (a) a consumer for this routing key exists but under a naming convention not searched here, or
- (b) this queue is genuinely unconsumed/orphaned in the current consumer codebase, or
- (c) the binding/queue lives entirely in RabbitMQ broker config (management UI), not in
  application code, and the actual consumer is a different service not part of this KT's repo
  scope.

**[INFERRED — confirm with team]**: whether this message is actively consumed anywhere, and if
so by what. This is a materially different (and more actionable) finding than the prior doc
pass, which only speculated the message "looks like an audit push."

---

## 7. Kafka

**Koi Kafka usage nahi mila** in either file (`UpdateWithOtpController.go`,
`UpdateWithOtpModel.go`) — grepped for `Kafka`/`kafka`, zero hits. Consistent with this being a
synchronous request/response controller with no bulk-ingestion use case.

---

## 8. Redis

**Koi Redis usage nahi mila** in either file — grepped for `Redis`/`redis`, zero hits. Both OTP
lookups (`GetOtp` calls) go straight to Postgres (`authpg`) on every request, no caching layer.

**However**, the response field names are literally `Otpredis1`, `Otpredis2`, `Keyredis1`,
`Keyredis2` (section 5, rule 11) — this naming strongly suggests the OTP-lookup mechanism was
originally designed/intended to be **Redis-backed**, even though the code that actually runs
today reads from Postgres (`glusr_usr_otp` table). This is either:
- a leftover naming artifact from an earlier or planned Redis-based design that was never
  finished migrating away from, or
- evidence that a *sibling* OTP-verification flow elsewhere in the codebase (not part of this
  feature) is Redis-backed and this response shape was copy-pasted from it.

**[INFERRED — confirm with team]** which of these it is; not resolvable from this feature's code
alone.

---

## 9. End-to-End Technical Flow

```
Supplier / internal-app
    │
    ▼
[API]  POST /user/updatewithotp
        {USR_ID, UPDATEDBY, UPDATEDBY_ID?, UPDATEDUSING, VALIDATION_KEY, IP, IP_COUNTRY,
         HIST_COMMENTS?, OTP1, OTP1_ATTR("value$$attr_id"), OTP2, OTP2_ATTR,
         <new-attribute-value, e.g. PH_MOBILE="...">}
    │  UserUpdateWithOTPController.go — inline mandatory-field checks (rules 1-3)
    │  parse OTP1_ATTR → (otp1Attr value, attribute_id) → resolve mob/email via attributeArray
    ▼
[DB — authpg]  GetOtp(mob/email, "OTP1")
    SELECT glusr_usr_otp_code FROM glusr_usr_otp WHERE lower(glusr_usr_value)=lower($1)
    │
    ├─ empty → "OTP1 not Found" (500), STOP
    ├─ mismatch → "OTP1 not Verified" (204), STOP
    │
    └─ match →
         key_val = first attributeArray-key found among top-level request keys (rule 7)
         │
         [RabbitMQ]  SERVICENAME=UPDATEWITHOTP → routing key user.additional.*, exchange
                      USER.topic (audit/history push, fires now, metadata-only payload)
         │
         ▼
       [DB — authpg]  GetOtp(key_val, email, "OTP2")
         │
         ├─ empty → "OTP2 not Found" (500), STOP
         ├─ mismatch → "OTP2 not Verified" (204), STOP
         ├─ key_val != OTP2_ATTR → "OTP2 Attribute value does not match with update param" (204), STOP
         │
         └─ match →
              TriggerRequest()
                  │  strip OTP1/OTP1_ATTR/OTP2/OTP2_ATTR/token/modid fields
                  ▼
              [HTTP POST]  user_update_api → /user/update → UserUpdateController
                  │  SERVICENAME=GLUSR_UPDATE_SERVICE — standard profile-update path,
                  │  its own DB-write + downstream fan-out (not traced in this pass)
                  ▼
              CODE!="200" → re-wrap as 500/Failure | CODE=="200" → re-wrap as SUCCESS
                  ▼
              Response relayed back to caller, with Otpredis1/2, Keyredis1/2, Otp1/2 always attached
```

---

## 10. Flow-wise DB & Table Usage

Only one flow exists in this feature (no branches like GST/Bank-Details) — every request follows
the same path up to where it terminates (early-reject vs full-success).

| # | DB | Table | Operation | Kya nikala/likha jaata hai, aur kyun |
|---|---|---|---|---|
| 1 | authpg | `glusr_usr_otp` | SELECT | Step 1 — fetch `glusr_usr_otp_code` for the mob/email resolved from `OTP1_ATTR`, compare against submitted `OTP1` |
| 2 | authpg | `glusr_usr_otp` | SELECT (only if step 1 passes) | Step 2 — fetch `glusr_usr_otp_code` for `key_val` (the new value being submitted), compare against submitted `OTP2` |
| — | (RabbitMQ, not DB) | `UPDATEWITHOTP` → `user.additional.*` | PUBLISH (only if step 1 passes, before step 2 runs) | Metadata-only audit event — no field write |
| — | (HTTP, not DB) | — | POST to `UserUpdateController` (only if both steps pass) | Triggers the actual `GLUSR_USR`/attribute write — **that write's own DB round-trips belong to `UserUpdateController`, not traced here** |

**Total direct DB round-trips in this controller: exactly 2**, both SELECTs against the same
table, both against `authpg` — this is by a wide margin the lightest-weight DB footprint of any
feature documented in this KT series (GST: ~9-10 round-trips across 4 databases per write; Bank
Details: up to 8 across 2 databases + 2 external APIs). All of the "real" write cost lives inside
`UserUpdateController`, entirely outside this controller's scope.

---

## 11. Optimization Scope — DB Response-Time Contribution

### Low impact

1. **Two sequential OTP-lookup queries against `authpg`** — inherent to the two-factor design;
   can't be parallelized since OTP2 must not be checked before OTP1 passes, by design (a
   security requirement, not a performance oversight).
2. **Both queries use a 1-second timeout** (`utils.ExecuteQueryRows(..., 1*time.Second)` —
   [`UpdateWithOtpModel.go:36`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UpdateWithOtpModel.go))
   — standard for this codebase (matches GST/Bank-Details' typical 1s query timeouts), no
   fail-open risk like Bank Details' 100ms SYSTEM-reqType pre-check, since a timeout here just
   surfaces as a `500` error (`err != nil` path), not a silent default.
3. **Downstream `TriggerRequest` is an HTTP call, not a DB call** — its latency is outside this
   controller's direct DB-optimization scope; whatever `UserUpdateController` does internally
   (DB writes, RabbitMQ fan-out) would need its own KT pass to optimize.

### Design-scope note (not performance)

4. **RabbitMQ audit-push fires after OTP1 alone, before OTP2 is even attempted** (section 5,
   rule 6) — if this message is meant to represent "user data was changed," it's technically
   premature (fired before the change is confirmed/executed); if it's meant as an
   "OTP1-verification-attempt log," the current unconsumed-queue finding (section 6) means it
   may not even be doing that job today. Either way this is a **correctness/design question**
   worth resolving before optimizing anything else in this feature — optimizing the two DB
   queries (which are already lightweight) would be a low-value use of engineering time compared
   to resolving whether the RabbitMQ message is actually consumed by anyone.

---

## 12. Cron Inventory

**No cron job related to this feature was found.** Searched `user-temp-consumers-production`'s
`internal/Workers` tree and any `crons/`-style directory for `UPDATEWITHOTP`,
`glusr_usr_otp`, `OTP` — no scheduled job touches this table or feature. OTP records presumably
expire/get cleaned up by whatever system generates them (out of scope, section header note), not
by anything in these three repos.

---

## 13. Edge Cases & Gotchas (technical POV)

1. **No `Gateway_v1`/allowlist-check** — every other controller documented in this KT series
   validates the caller via a `Gateway_v1([]string{...}, validationKey, "k")`-style check
   against a hardcoded `modid` allowlist (see GST's `checkScreenPermisson`, Bank Details'
   gateway-allowlist); this controller only checks that `VALIDATION_KEY` is non-empty, never
   validates it against any allowlist. Given this feature is explicitly meant to **bypass a
   security lock** (overriding OTP/Tactical-verified GST, per the GST KT's section 4 locking
   rule), the absence of a caller-allowlist check here is a more consequential gap than in a
   typical read/write controller. Worth confirming whether this is intentional (maybe validated
   elsewhere/upstream, e.g. at an API-gateway layer) or a genuine gap.
2. **RabbitMQ audit-push fires on OTP1-success alone, and its consumer is not conclusively
   identified** (section 6) — a caller who fails OTP2 still causes this message to have been
   sent, and it's unclear anyone is even listening for it.
3. **Response field names (`Otpredis1`/`Keyredis2` etc.) suggest a Redis-based design that isn't
   what's actually implemented** (section 8) — possible sign of an incomplete migration or
   copy-pasted response-shape from another OTP-flow.
4. **Full raw OTP codes (`Otpredis1`/`Otpredis2`) are echoed back in every API response**,
   success or failure — including the case where `OTP1`/`OTP2` mismatched. If this response is
   ever logged verbatim or returned to a less-trusted client layer, this leaks the actual valid
   OTP code back to the caller — worth a security-review look, since it somewhat defeats the
   purpose of a two-factor OTP challenge if the correct answer is handed back on a failed
   attempt.
5. **`TriggerRequest`'s failure-handling**: if the downstream `user_update_api`/
   `UserUpdateController` call itself fails (network/5xx) or returns a non-200 business error,
   the OTP-verification has already fully succeeded — the caller gets a generic `500` and would
   need to retry the whole two-OTP flow again (OTPs are presumably single-use/short-lived, even
   though the actual failure was in the update-call, not the verification).
6. **Ambiguous `key_val` resolution if multiple attribute keys are present** (section 5, rule 7)
   — Go map iteration order is non-deterministic, so a request containing e.g. both `PH_MOBILE`
   and `EMAIL` top-level keys could non-deterministically pick either as `key_val`.
7. **Two identical route registrations** (`router.go:160` and `:328`) — same handler, same path,
   no observable difference found in this pass; likely just two route-groups (e.g. versioned/
   legacy) both wired to the same controller, not a bug, but worth confirming there's no
   divergence planned between them.

---

## 14. Open Questions

1. Is `VALIDATION_KEY` validated anywhere for this endpoint (no `Gateway_v1`/allowlist call
   found in this controller) — confirm whether that's a genuine gap or handled upstream (e.g.
   API gateway).
2. Who/what consumes the `UPDATEWITHOTP` RabbitMQ message (routing key `user.additional.*`,
   exchange `USER.topic`)? No consumer file matching this routing key or servicename was found
   in `user-temp-consumers-production` in this pass — confirm whether a consumer exists outside
   the searched paths, or whether this message is effectively unconsumed today.
3. Are `glusr_usr_otp` entries ever Redis-backed anywhere upstream (response-field naming
   strongly suggests it — `Otpredis1`/`Keyredis2`), or is Postgres (`authpg`) the sole OTP-store
   today for this feature?
4. Where/how are `glusr_usr_otp` rows populated with an actual `glusr_usr_otp_code` (i.e. OTP
   generation + SMS/email dispatch)? Not found in any of the three repos in scope for this KT —
   confirm which service owns OTP generation.
5. Is returning the raw `glusr_usr_otp_code` values (`Otpredis1`/`Otpredis2`) in every API
   response (including failed-verification responses) an intentional debug/internal-only
   behavior, or an oversight? Worth a security review.
6. `UserUpdateController`'s own validation/DB-write behavior for the specific attribute types
   this feature is designed to override (mobile/email, and per the GST KT cross-reference,
   potentially GST/locked fields too) — not traced in this pass; a future KT on the core
   `/user/update` endpoint would need to cover this.
7. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai (column existence, types, nullability, indexes not cross-checked against a
   live schema).

---

## See also

- [`Update_With_OTP_Business_Doc.md`](./Update_With_OTP_Business_Doc.md) — product perspective
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) section 4-5 — describes
  this feature as one of the two ways a Tactical/OTP-verified GST lock can be bypassed
- [`../Bank Details KT/Bank_Details_Technical_Doc.md`](../Bank%20Details%20KT/Bank_Details_Technical_Doc.md) —
  same shared-write-API-loopback pattern, and a sibling example of the "verified/locked field"
  concept this feature exists to override
- [`../Rating Usefulness KT/Rating_Usefulness_Technical_Doc.md`](../Rating%20Usefulness%20KT/Rating_Usefulness_Technical_Doc.md) —
  same HTTP-loopback architectural pattern (verify, then call back into another internal API)
- [`../Trust Verification KT/Trust_Verification_Technical_Doc.md`](../Trust%20Verification%20KT/Trust_Verification_Technical_Doc.md) —
  same HTTP-loopback architectural pattern

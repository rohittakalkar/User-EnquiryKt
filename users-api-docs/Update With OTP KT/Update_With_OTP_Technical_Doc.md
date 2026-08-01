# Update With OTP — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`Update_With_OTP_Business_Doc.md`](./Update_With_OTP_Business_Doc.md)
dekho.

**Repos**: `service-api-go-production` only. **No dedicated DB-write of its own** —
this is a **verification-gate + HTTP-loopback** controller (same architectural pattern
already seen in Rating Usefulness and Trust Verification KTs): it verifies two OTPs, then
calls back into another internal API (`user_update_api`) to perform the actual update.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Controller | write | [`UpdateWithOtpController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UpdateWithOtpController.go) (`UserUpdateWithOTPController`) |
| Model | write | [`UpdateWithOtpModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UpdateWithOtpModel.go) (`GetOtp`, `TriggerRequest`) |

---

## 2. Routes

| Method | serviceName | Repo | Controller |
|---|---|---|---|
| POST | `UPDATEWITHOTP` | write | `UserUpdateWithOTPController` |

---

## 3. Data Model — Tables

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `glusr_usr_otp` | authpg | OTP lookup-store (read-only from this controller) | `glusr_usr_value` (mobile/email, matched case-insensitively), `glusr_usr_otp_code` — [`UpdateWithOtpModel.go:26`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UpdateWithOtpModel.go) |

**No write to any table happens in this controller** — the actual `GLUSR_USR` update is
delegated entirely to a downstream API call (`user_update_api`, configured via
`config.GetConfig().APIList`).

---

## 4. Business Rules & Validation (code se)

1. **Mandatory fields**: `USR_ID` (numeric), `UPDATEDBY`, `UPDATEDUSING`,
   `VALIDATION_KEY`, `IP`, `IP_COUNTRY`, `OTP1`, `OTP2`, `OTP1_ATTR`, `OTP2_ATTR` — all
   validated inline (no separate mandatory-fields helper function, unlike most other
   controllers in this codebase).
   [`UpdateWithOtpController.go:121-136`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UpdateWithOtpController.go)
2. **No `Gateway_v1` allowlist-check found** — `VALIDATION_KEY` is required as a
   mandatory field but never passed through the standard `Gateway_v1` allowlist-validator
   used by nearly every other controller in this codebase — worth flagging (see Open
   Questions/Edge Cases).
3. **`OTP1_ATTR` format is `"<value>$$<attribute_id>"`** — split on `$$`; the
   `attribute_id` (e.g. `109`=EMAIL, `157`=EMAIL_ALT, `121`=PH_MOBILE, `48`=PH_MOBILE_ALT,
   per the hardcoded `attributeArray` map) determines whether the OTP-lookup value is
   treated as a mobile-number or email.
   [`UpdateWithOtpController.go:112-166`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UpdateWithOtpController.go)
4. **Sequential two-step OTP verification, hard-gated**:
   - **Step 1**: `GetOtp(mob/email from OTP1_ATTR)` fetched from `glusr_usr_otp`;
     compared against `OTP1`. Mismatch → `"OTP1 not Verified"` (code 204), Step 2 never
     runs.
   - **Step 2** (only if Step 1 passes): the request's actual new-value (found by
     scanning `inputParams` for any key present in `attributeArray`, i.e. `EMAIL`,
     `EMAIL_ALT`, `PH_MOBILE`, `PH_MOBILE_ALT`) is looked up as `key_val`, then
     `GetOtp(key_val, email)` fetched again for `OTP2` comparison. Mismatch →
     `"OTP2 not Verified"`.
   - **Extra check on Step 2 success**: `key_val != otp2_attr` → reject even if the OTP
     code matched — `"OTP2 Attribute value does not match with update param"` — ensures
     the OTP2 was generated for the exact value being submitted, not just any valid OTP.
   [`UpdateWithOtpController.go:190-286`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UpdateWithOtpController.go)
5. **After Step 1 passes (before Step 2), a RabbitMQ message is already pushed** —
   `SERVICENAME=UPDATEWITHOTP`, `TABLES=GLUSR_USR`, `ACTION=UPDATE` — this fires as soon
   as OTP1 verifies, **independent of whether OTP2 later succeeds or fails**. This looks
   like an audit/history-log push (columns are metadata like `UPDATEDBY`/`IP`, not the
   actual new value), not the real data-update — but it's worth confirming this doesn't
   represent a premature side-effect before full two-factor verification completes.
   [`UpdateWithOtpController.go:206-249`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UpdateWithOtpController.go)
6. **`TriggerRequest()` — the actual update, only on full success**: strips OTP-specific
   fields (`OTP1`, `OTP1_ATTR`, `OTP2`, `OTP2_ATTR`, `token`, `modid`) from the payload,
   then `POST`s the remaining params to a configured `user_update_api` URL — an
   **HTTP-loopback into another internal update-endpoint** (same pattern as Rating
   Usefulness's `callRatingWrite` and Trust Verification's `VerifyAttr`), not a direct DB
   write from this controller.
   [`UpdateWithOtpModel.go:63-93`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UpdateWithOtpModel.go)

---

## 5. RabbitMQ / Kafka / Redis

| Queue / `SERVICENAME` | Publisher | Purpose |
|---|---|---|
| `UPDATEWITHOTP` (routed per domain-wide `serviceToQueueMap`) | `UpdateWithOtpController.go` — fires **right after OTP1 succeeds**, regardless of OTP2 outcome | Likely an audit/history-log event for `GLUSR_USR` (metadata-only payload — `UPDATEDBY`/`IP`/`att_id`, no actual new-value) |

**Koi Kafka ya Redis usage nahi mila** — though the response fields literally named
`Otpredis1`/`Otpredis2`/`Keyredis1`/`Keyredis2` in the response payload strongly suggest
OTPs were originally intended to be Redis-backed (naming convention), even though the
actual lookup in this code reads from Postgres (`glusr_usr_otp`) — worth confirming if
Redis was involved historically or in a related service.

---

## 6. End-to-End Technical Flow

```
Supplier / internal-app
    │
    ▼
[API]  POST serviceName=UPDATEWITHOTP
        {USR_ID, UPDATEDBY, UPDATEDUSING, VALIDATION_KEY, IP, IP_COUNTRY,
         OTP1, OTP1_ATTR("value$$attr_id"), OTP2, OTP2_ATTR, <new-attribute-value>}
    │  UserUpdateWithOTPController.go — inline mandatory-field checks
    │  parse OTP1_ATTR → (otp1Attr value, attribute_id) → resolve mob/email
    ▼
GetOtp(mob/email, "OTP1")  — SELECT glusr_usr_otp_code FROM glusr_usr_otp WHERE value=$1
    │
    ├─ mismatch → "OTP1 not Verified" (204), STOP
    │
    └─ match →
         [RabbitMQ]  SERVICENAME=UPDATEWITHOTP (audit/history push, fires now)
         │
         ▼
       GetOtp(key_val, email, "OTP2")
         │
         ├─ mismatch → "OTP2 not Verified" (204), STOP
         ├─ key_val != OTP2_ATTR → "OTP2 Attribute value does not match" (204), STOP
         │
         └─ match →
              TriggerRequest()
                  │  strip OTP*/token/modid fields
                  ▼
              [HTTP POST]  user_update_api (internal loopback — actual update-endpoint)
                  ▼
              Response relayed back to caller
```

---

## 7. Optimization Scope — DB Response-Time Contribution

### Low impact

1. **Two sequential OTP-lookup queries against `authpg`** — inherent to the two-factor
   design, not really reducible without weakening the security-gate (can't parallelize
   since OTP2 shouldn't be checked until OTP1 passes, by design).
2. **Downstream `TriggerRequest` is an HTTP call, not a DB call** — its latency is
   outside this controller's direct DB-optimization scope; the actual write-performance
   would be documented wherever `user_update_api`'s target controller lives.

### Design-scope note (not performance)

3. **RabbitMQ push happens after OTP1 alone, not after full 2-factor success** — if this
   message represents "user data was changed," it's technically premature (fired before
   the change is confirmed/executed); if it's meant as "OTP1-verification-attempt log,"
   the naming/consumer-side handling should make that intent clear.

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora Update With OTP flowchart yahan dekho](https://lucid.app/lucidchart/26960898-6427-40d3-8225-1ee2222f80a2/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **No `Gateway_v1` allowlist-check** — every other controller documented in this KT
   series validates the caller via `Gateway_v1([]string{...}, validationKey, "k")`
   against a hardcoded `modid` allowlist; this controller only checks that
   `VALIDATION_KEY` is non-empty, never validates it against any allowlist. Worth
   confirming whether this is intentional (maybe validated elsewhere/upstream) or a gap.
2. **RabbitMQ audit-push fires on OTP1-success alone** — see point 5 above; a caller who
   fails OTP2 still causes this message to have been sent.
3. **Response field names (`Otpredis1`/`Keyredis2` etc.) suggest a Redis-based design
   that isn't what's actually implemented** (current code reads Postgres) — possible
   sign of an incomplete migration or copy-pasted response-shape from another OTP-flow.
4. **`TriggerRequest`'s failure-handling**: if the downstream `user_update_api` call
   itself fails (network/5xx), the OTP-verification has already fully succeeded — the
   caller would need to retry the whole OTP flow again since OTPs are presumably
   single-use/short-lived, even though the actual failure was in the update-call, not
   the verification.

---

## 10. Open Questions

1. Is `VALIDATION_KEY` validated anywhere for this endpoint (no `Gateway_v1` call
   found) — confirm whether that's a genuine gap or handled elsewhere.
2. What does the `UPDATEWITHOTP` RabbitMQ message's consumer actually do with it — pure
   audit-log, or does it also perform a side-effect?
3. Are `glusr_usr_otp` entries Redis-backed anywhere upstream (response-field naming
   suggests it), or is Postgres the sole OTP-store today?
4. `user_update_api`'s actual target-controller (which performs the real `GLUSR_USR`
   write) — not traced in this pass; likely `UserDetailsController` or similar, worth
   cross-referencing if a future KT documents core profile-update.
5. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`Update_With_OTP_Business_Doc.md`](./Update_With_OTP_Business_Doc.md) — product perspective
- [`../Rating Usefulness KT/Rating_Usefulness_Technical_Doc.md`](../Rating%20Usefulness%20KT/Rating_Usefulness_Technical_Doc.md) —
  same HTTP-loopback architectural pattern
- [`../Trust Verification KT/Trust_Verification_Technical_Doc.md`](../Trust%20Verification%20KT/Trust_Verification_Technical_Doc.md) —
  same HTTP-loopback architectural pattern

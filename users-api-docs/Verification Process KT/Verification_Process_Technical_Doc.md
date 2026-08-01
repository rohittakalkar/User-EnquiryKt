# Verification Process (Attribute Verification) — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`Verification_Process_Business_Doc.md`](./Verification_Process_Business_Doc.md)
dekho.

**Scope note**: Distinct from [`../Trust Verification KT/`](../Trust%20Verification%20KT/)
— that KT covers `UserVerificationController.go` (single-attribute read/write against
`mesh_pg_user`, stored functions `fn_get_glusr_attr_verify`/`fn_get_glusr_attributes`)
and `USER_APPROVAL_ATTRIBUTE.go`'s `VerifyAttr()` HTTP-loopback. This KT covers a
**separate, more recently-written** write/read pair (`UserVerificationDetailsController`
+ `GetVerificationDetails`) that records structured, primary+secondary-cross-linked
verification-events against **`trustPg`** via a dedicated stored function. Both systems
appear to serve the trust/verification domain but through different code-paths and
different physical databases (`trustpg` here vs `mesh_pg_user` there) — their exact
relationship (do they feed each other? are they redundant?) is flagged as an Open
Question.

**Repos**: `service-api-go-production` (write), `users-api-go-production` (read).
**Koi RabbitMQ/Kafka/consumer nahi mila** for this specific write-path — purely
synchronous.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write | write | [`UserVerificationDetailsController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserVerificationDetailsController.go), [`UserVerificationDetailsModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go) (`UserVerificationDetailsValidation`, `handlePrimaryAndSecondaryInsertions`, `upsertSellerDetails`) |
| Read | read | [`GetVerificationDetailController.go`](../../users-api-go-production/internal/controllers/UsersControllers/GetVerificationDetailController.go) (`GetVerificationDetails`), [`VerifiedDetailController.go`](../../users-api-go-production/internal/controllers/UsersControllers/VerifiedDetailController.go) (`VerifiedDetailController`), [`VerifiedDetailModel.go`](../../users-api-go-production/internal/models/users/VerifiedDetailModel.go) |

---

## 2. Routes

| Method | serviceName | Repo | Controller |
|---|---|---|---|
| POST | `TRUST_ATTR_VERI_DETAILS` | write | `UserVerificationDetailsController` |
| — | `GET_ATTR_VERI_DETAILS` | read | `GetVerificationDetails` |
| — | `USERVERIFIEDDETAIL` | read | `VerifiedDetailController` |

---

## 3. Data Model

| Attribute-ID → Category mapping (`attributeValueMap`, code se) | |
|---|---|
| `121`, `48`, `1293` → **Mobile** | `157`, `109`, `1285` → **Email** |
| `2721` → **GST** | `348` → **PAN** |
| `346` → **CIN** | `347` → **TAN** |
| `111` → **Company Name** | `390` → **Bank Account Number** |
| `2120` → **Aadhar Number** | `112`, `113`, `1278`, `1279` → **Address** |
| `114` → **City** | `115` → **State** |
| `106`, `108` → **First/Last Name** | `352` → **IEC Code** |
| `141`, `142` → **CEO First/Last Name** | `117` → **Zip** |
| `2882` → **Udyam** | `2072`, `2073` → **Latitude-Longitude** |

[`UserVerificationDetailsModel.go:11-40`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go) —
this is a hardcoded Go map, not sourced from a DB lookup-table, meaning any new
attribute-ID requires a code-deploy to support.

**Underlying storage**: no `INSERT`/`UPDATE` SQL literal — the write goes entirely
through a **stored PL/pgSQL function**: `SELECT fn_insert_glusr_attribute_verification($1
..$11) AS status` on **`trustpg`**. The function's internal table-structure isn't
visible from the Go code — it's a black-box RPC-style call.
[`UserVerificationDetailsModel.go:266`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go)

---

## 4. Business Rules & Validation (code se)

1. **Gateway allowlist**: `WEBERP`, `SOA-WAPI` — narrower than most write-endpoints in
   this codebase. [`UserVerificationDetailsController.go:43`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserVerificationDetailsController.go)
2. **`prepareInputParamsMap` strictly type-checks every field** — `glid`/`status`/
   `primaryAttrID` must be JSON-numbers (`float64`, i.e. this endpoint expects a real
   JSON body, not form-encoded strings like most other controllers in this codebase);
   `fk_ref_id` defaults to `primaryAttrID` if not explicitly supplied and must be
   ≤10-digits; `datetime` must match `yyyyMMddHHmmss`.
   [`UserVerificationDetailsModel.go:57-170`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go)
3. **`primaryAttrID` and every key of `secondaryAttr` must exist in
   `attributeValueMap`** — a single invalid secondary-attribute-key fails the ENTIRE
   secondary-attribute-set (`allValid=false` short-circuits the whole batch), even if
   other secondary-attributes in the same request were valid.
   [`UserVerificationDetailsModel.go:179-190`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go)
4. **Primary-attribute-only insert only happens if there are ZERO secondary
   attributes** (`if len(input.SecondaryAttr) == 0`) — if secondary-attributes ARE
   present, the primary-alone record is **not** separately inserted; instead, one
   combined primary+secondary record is inserted per secondary-attribute.
   [`UserVerificationDetailsModel.go:234-249`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go)
5. **Empty-string secondary-attribute-values are silently skipped** (`if secAttrVal !=
   ""`) — no error, just no insert for that one entry.
6. **`upsertSellerDetails` reformats `datetime`**: if the supplied `yyyyMMddHHmmss`
   string fails to parse (already validated earlier, so this is mostly a safety-net),
   falls back to `time.Now()`; otherwise reformats to `YYYY-MM-DD HH:MM:SS` for the
   stored-function call.
7. **Stored function's return-contract**: expects a `status` field in the response-row;
   `status != 1` is treated as a failure regardless of what the actual returned value
   means — the Go code has no visibility into *why* the stored function rejected the
   call, only a pass/fail signal.
   [`UserVerificationDetailsModel.go:257-285`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserVerificationDetailsModel.go)

---

## 5. RabbitMQ / Kafka / Redis

**Koi RabbitMQ, Kafka, ya Redis usage nahi mila** — purely synchronous, all logic
delegated to the `trustpg` stored function.

---

## 6. End-to-End Technical Flow

```
Internal-verification-process (WEBERP / SOA-WAPI)
    │
    ▼
[API — write]  POST serviceName=TRUST_ATTR_VERI_DETAILS
                {glid, status, primaryAttrID, primaryAttrVal, fk_ref_id?, source,
                 platform, verification_type, datetime, secondaryAttr?: {attrID: val, ...}}
    │  UserVerificationDetailsController.go — Gateway (WEBERP/SOA-WAPI)
    │  LengthAndTypeValidations_v3()
    ▼
UserVerificationDetailsValidation()
    │  prepareInputParamsMap() — strict type/format checks
    │  validateAttribute(primaryAttrID) + validateSecondaryAttrs()
    ▼
handlePrimaryAndSecondaryInsertions()
    ├─ no secondary attrs → upsertSellerDetails(primary only)
    └─ secondary attrs present → upsertSellerDetails() once per non-empty secondary,
                                   each call passing BOTH primary and that secondary
    │  each call: SELECT fn_insert_glusr_attribute_verification(...) AS status
    ▼
[DB — trustpg]  (stored function, internal schema opaque to this code)
    ▼
Response {"SUCCESS" / error-message}

Any caller
    │
    ▼
[API — read]  serviceName=GET_ATTR_VERI_DETAILS  {glusrid, modid, attrId, type?}
    │  GetVerificationDetails → (VerifiedDetailModel.go / underlying query, not
    │  fully traced in this pass)
    ▼
Response — verified-attribute data for the given user/attribute
```

---

## 7. Optimization Scope — DB Response-Time Contribution

### Medium impact

1. **One stored-function-call per secondary-attribute, sequential in a `for` loop** — if
   a request has N secondary-attributes, this is N sequential round-trips to `trustpg`.
   Given the stored-function encapsulates the actual insert-logic, this could
   potentially be redesigned as a single stored-function-call accepting an array of
   secondary-attributes, moving the loop into PL/pgSQL — but that would require
   DB-side changes outside this codebase's visibility.

### Low impact

2. **All validation happens in Go before any DB call** — good practice, avoids wasting
   a DB round-trip on malformed requests.

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora Verification Process flowchart yahan dekho](https://lucid.app/lucidchart/ec4c7cee-dabc-419a-9739-ab108dba7e3d/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **A single invalid secondary-attribute-key fails the whole secondary-batch** — even
   if 4 out of 5 secondary-attributes were valid, one bad key rejects the entire
   request with no partial-success.
2. **Primary-only insert is skipped whenever secondary-attributes exist** — a caller
   expecting "the primary attribute itself gets its own record regardless" would be
   surprised; the primary's verification-status is only ever recorded jointly with each
   secondary, not standalone, when secondaries are present.
3. **Stored-function is a black-box** — table-structure, constraints, and exact
   failure-reasons live entirely in `trustpg`'s PL/pgSQL, invisible to this codebase;
   debugging a failed insert requires DB-access, not just code-reading.
4. **This write-path and Trust Verification KT's write-path both touch "verification"
   concepts but write to different databases** (`trustpg` here, the other via
   `mesh_pg_user`-adjacent stored functions) — their data-consistency relationship isn't
   traceable from code alone.

---

## 10. Open Questions

1. What is the actual relationship between this `trustpg`-based verification-recording
   system and the `UserVerificationController`/`USER_APPROVAL_ATTRIBUTE.go` system
   documented in Trust Verification KT — do they feed each other, overlap, or serve
   fully separate purposes?
2. `fn_insert_glusr_attribute_verification`'s internal logic/schema — not visible from
   Go code, would need direct DB access to fully understand.
3. What do `status` (input) and `source`/`platform`/`verification_type` string-values
   actually represent — no enum/reference found in this codebase.
4. `VerifiedDetailController.go`'s full logic wasn't traced in this pass — only its
   header/param-parsing was read; worth a follow-up if this endpoint needs deeper
   documentation.
5. Live DB schema verification — is doc ne sirf Go SQL/RPC-calls jo imply karti hain
   wahi reflect kiya hai.

---

## See also

- [`Verification_Process_Business_Doc.md`](./Verification_Process_Business_Doc.md) — product perspective
- [`../Trust Verification KT/Trust_Verification_Technical_Doc.md`](../Trust%20Verification%20KT/Trust_Verification_Technical_Doc.md) —
  related but separate verification-domain write/read path

# Payment Transaction (International) — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`Payment_Txn_Business_Doc.md`](./Payment_Txn_Business_Doc.md)
dekho.

**Repos**: `service-api-go-production` (write only). **Koi read-controller, RabbitMQ,
Kafka, ya consumer nahi mila** — write-only transaction-log, purely synchronous.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write | write | [`UserAddPaymentTxnDetailsController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserAddPaymentTxnDetailsController.go), [`UserAddPaymentTxnDetailsModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddPaymentTxnDetailsModel.go) (`UpsertPaymentTxnDetails`) |
| Validation | write | `MandatoryParamsCheckPaymentTxnDetails` — [`UserUtilsMandatory.go:1898`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go), `UserPaymentTxnMap` (`LengthAndTypeValidations_v3`) — [`UsersValidationMaps.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |

---

## 2. Routes

| Method | serviceName | Repo | Controller |
|---|---|---|---|
| POST | `PAYMENT_TXN_DETAILS` | write | `UserAddPaymentTxnDetailsController` |

---

## 3. Data Model — Table

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `Glusr_international_txn_detail` | meshpg | International buyer-to-seller transaction log | `txn_creation_date`, `fk_seller_glusr_id`, `fk_buyer_iso` (country-ISO), `fk_buyer_glusr_id`, `fk_buyer_name`, `fk_currency_id`, `invoice_amount`, `vendor_invoice_url`, `vendor_invoice_id` (derived), `fk_buyer_email`, `fk_txn_product_details`, `usr_txn_details` (JSON) — [`UserAddPaymentTxnDetailsModel.go:38-58`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddPaymentTxnDetailsModel.go) |

**No `RETURNING` clause on insert** — unlike almost every other insert-controller
documented in this KT series, this one doesn't retrieve/return a generated primary-key
(no auto-increment ID column referenced anywhere in the insert or update queries) — the
table appears to be keyed for lookups by `(vendor_invoice_id, fk_seller_glusr_id)`
instead.

---

## 4. Business Rules & Validation (code se)

1. **Gateway allowlist**: `GLADMIN`, `SELLERMY`, `IMOB`, `ANDROID`, `IOS`.
   [`UserUtilsMandatory.go:1965`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
2. **`action` must be `i` or `u`.**
3. **Insert (`action=i`) mandatory fields**: `BUYER_ID`, `SELLER_ID`, `BUYER_ISO`,
   `BUYER_NAME`, `CURRENCY_ID`, `AMOUNT`, `INVOICE_URL` — with an unusual numeric-check:
   `buyer_id`/`seller_id`/`amount` must be numeric, but `buyer_iso`/`buyer_name`/
   `currency_id` must **NOT** be numeric (`!utils.IsNumeric(...)` inverted for these
   three — i.e., these are validated as non-numeric strings, which is the correct intent
   for a country-ISO-code/name/currency-code, just an easy detail to misread in the
   `||`-chained condition).
   [`UserUtilsMandatory.go:1975,1981-1982`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
4. **Update (`action=u`) mandatory fields**: `ID`, `SELLER_ID` only.
5. **`txn_creation_date` format strictly validated**: must be exactly 14 characters,
   parseable as `YYYYMMDDHHMMSS`.
6. **`txn_details` (if present) must be a JSON object** (`map[string]interface{}`),
   not a string/array/scalar — validated before insert/update.
7. **Invoice-ID is derived from `inv_url`, not supplied directly**: everything after the
   first `#` character in the URL becomes `vendor_invoice_id`. If no `#` is present,
   invoice-ID is left empty.
   [`UserAddPaymentTxnDetailsModel.go:33-36`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddPaymentTxnDetailsModel.go)
8. **Insert writes the full record**; **update only ever touches `usr_txn_details`** —
   the `for k, v := range params` loop's `switch` statement has exactly one `case`
   (`"txn_details"`), so even if other fields are passed on an update request, they're
   silently ignored — only `usr_txn_details` can ever be changed post-insert.
   [`UserAddPaymentTxnDetailsModel.go:42-59`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddPaymentTxnDetailsModel.go)
9. **Update's `WHERE` clause matches on `(vendor_invoice_id, fk_seller_glusr_id)`** —
   using the request's `id` param mapped to `vendor_invoice_id` (not the derived
   invoice-ID from `inv_url` — on update, `inv_url` isn't even required/parsed since
   `action=u` doesn't require `INVOICE_URL`).
10. **Update guarded by `len(input_params) != 0`** — if `txn_details` wasn't present in
    the update-request, `query` remains just `"UPDATE Glusr_international_txn_detail SET
    "` (no columns, no WHERE) and is never executed (implicitly, since the `WHERE`
    clause is only appended inside that `if` block) — meaning a no-op update silently
    "succeeds" with an unexecuted/malformed query never reaching the DB... actually
    executes an **empty/invalid query string** if not guarded properly; worth a closer
    look (see Open Questions).

---

## 5. RabbitMQ / Kafka / Redis

**Koi RabbitMQ, Kafka, ya Redis usage nahi mila** — purely synchronous single-table
insert/update.

---

## 6. End-to-End Technical Flow

### Insert
```
International Buyer → Supplier payment (via GLADMIN/SELLERMY/IMOB/ANDROID/IOS)
    │
    ▼
[API — write]  POST serviceName=PAYMENT_TXN_DETAILS
                {action="i", buyer_id, seller_id, buyer_iso, buyer_name, currency_id,
                 amount, inv_url, buyer_email?, prod_details?, txn_creation_date?,
                 txn_details?}
    │  UserAddPaymentTxnDetailsController.go
    │  MandatoryParamsCheckPaymentTxnDetails() — Gateway + type/format checks
    ▼
UpsertPaymentTxnDetails()
    │  invoice-ID extracted from inv_url (text after '#')
    │  INSERT INTO Glusr_international_txn_detail (...)
    ▼
[DB — meshpg]
    ▼
Response {"INSERT SUCCESS"}
```

### Update
```
[API — write]  POST serviceName=PAYMENT_TXN_DETAILS
                {action="u", id (=vendor_invoice_id), seller_id, txn_details}
    │  MandatoryParamsCheckPaymentTxnDetails()
    ▼
UpsertPaymentTxnDetails()
    │  UPDATE Glusr_international_txn_detail SET usr_txn_details=$1
    │  WHERE vendor_invoice_id=$2 AND fk_seller_glusr_id=$3
    ▼
[DB — meshpg]
    ▼
Response {"UPDATE SUCCESS"}
```

---

## 7. Optimization Scope — DB Response-Time Contribution

### Low impact

1. **Single-query, no sequential round-trips** — straightforward, low-risk write path
   for both insert and update.
2. **No `RETURNING` clause on insert** — since no generated-ID is needed by the caller
   (lookup key is invoice-ID, supplied by caller), this actually avoids an unnecessary
   round-trip-shape that other controllers incur.

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora Payment Txn flowchart yahan dekho](https://lucid.app/lucidchart/45ab8086-c005-4199-944b-e712e6b98b45/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **Update silently only affects `usr_txn_details`** — a caller expecting to correct
   `amount`/`buyer_name`/other fields via `action=u` will find those fields are simply
   ignored, with no error indicating so.
2. **Empty/guard-condition on update when `txn_details` absent** (§4, point 10) — worth
   verifying in a live environment whether an update-request without `txn_details`
   actually results in a clean no-op response, or an unexpected empty-query execution
   attempt; the code path is subtle enough to warrant a direct test.
3. **No `RETURNING`/generated-ID** means the caller must already know/track the
   invoice-derived-ID to ever update a record later — if `inv_url` didn't contain a `#`
   at insert-time, `vendor_invoice_id` is empty, and that record becomes effectively
   un-updatable via this endpoint (no way to `WHERE vendor_invoice_id=''` meaningfully
   target it later).
4. **`buyer_iso`/`buyer_name`/`currency_id` "must not be numeric" validation** is easy
   to misread in the combined `||` condition — worth double-checking if a legitimately
   numeric-currency-code (some systems use numeric ISO 4217 codes) would incorrectly
   fail validation.

---

## 10. Open Questions

1. What actually happens when `action=u` is submitted without `txn_details` — clean
   no-op, or a malformed-query execution attempt? Needs direct verification.
2. Is there any recovery-path for a transaction whose `inv_url` lacked a `#` (empty
   `vendor_invoice_id`) — can it ever be updated later?
3. Numeric-currency-code (ISO 4217 numeric) suppliers — would they fail the
   `currency_id`-must-not-be-numeric check?
4. Any downstream consumer of this table (financial-reporting/reconciliation) — not
   traceable from this codebase.
5. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`Payment_Txn_Business_Doc.md`](./Payment_Txn_Business_Doc.md) — product perspective

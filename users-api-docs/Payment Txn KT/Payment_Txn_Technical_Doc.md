# Payment Transaction (International) — Technical Doc (Code-Level Deep Dive)

Yeh doc Payment Txn feature ka **technical implementation** cover karta hai — API, DB table,
queries, RabbitMQ/Kafka/Redis (confirmed absent), crons, sab kuch code se verify karke.
Business/product perspective ke liye [`Payment_Txn_Business_Doc.md`](./Payment_Txn_Business_Doc.md)
dekho — dono docs same flow cover karte hain, bas alag audience ke liye.

**Repos involved (verified this pass)**: sirf `service-api-go-production` (write). Iss feature
ke liye `users-api-go-production` (read repo) mein koi controller/model nahi mila, aur
`user-temp-consumers-production` (consumers+crons repo) mein bhi koi consumer/cron reference
nahi mila — dono repos explicitly grep kiye gaye the (patterns: `PaymentTxn`, `payment_txn`,
`international_txn`, `PAYMENT_TXN`, `txn_detail`), zero hits. Independent confirmation:
[`service_api_write_reference.md`](../service_api_write_reference.md) mein bhi is controller
ke against `Side effects: none` likha hai.

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file:line diya gaya
hai). Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — team se confirm karo]** likh diya hai, guess nahi kiya.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write (insert/update) | write | [`UserAddPaymentTxnDetailsController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserAddPaymentTxnDetailsController.go) |
| Write model (SQL, invoice-ID derivation) | write | [`UserAddPaymentTxnDetailsModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddPaymentTxnDetailsModel.go) (`UpsertPaymentTxnDetails`) |
| Mandatory-field/type validation | write | `MandatoryParamsCheckPaymentTxnDetails` — [`UserUtilsMandatory.go:1898-1994`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go) |
| Length/type/column-name map | write | `UserPaymentTxnMap` — [`UsersValidationMaps.go:796-809`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) (used by `LengthAndTypeValidations_v3`) |
| Route registration | write | [`router.go:175,343`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) — `POST /user/addpaymenttxndetails` registered **twice** (two router groups, same handler; likely one public + one gateway-scoped mount — **[INFERRED — confirm with team why duplicated]**) |
| Read | — | **Not found.** No controller/model in `users-api-go-production` (read repo) references `PaymentTxn`, `payment_txn`, `international_txn`, or `Glusr_international_txn_detail`. Grepped `internal/` tree fully, zero hits. |
| Consumer/queue | — | **Not found.** No file in `user-temp-consumers-production` references `PaymentTxn`, `payment_txn`, `international_txn`, `PAYMENT_TXN`, or `txn_detail`. Grepped full repo tree, zero hits. |
| Cron | — | **Not found.** No `crons/` directory match, no cron file references this table/feature. |
| Sibling write controllers (same file area, not this feature but worth knowing exists) | write | [`UserAddMechantToVendorController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserAddMechantToVendorController.go) (`Glusr_payment_vendor_account`), [`UserAddPaymentVendorController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserAddPaymentVendorController.go) (`payment_vendor_master`) — different tables, not part of this feature, listed only so they aren't confused with Payment Txn during KT |

---

## 2. Routes

| Method | Path | serviceName | Repo | Controller |
|---|---|---|---|---|
| POST | `/user/addpaymenttxndetails` | `PAYMENT_TXN_DETAILS` | write | `UserAddPaymentTxnDetailsController` — [`router.go:175`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) |
| POST | `/user/addpaymenttxndetails` (second mount) | `PAYMENT_TXN_DETAILS` | write | Same controller — [`router.go:343`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) |

No GET route exists anywhere for this data in any of the three repos searched.

---

## 3. Data Model — Table

> **Verification note**: column names/types come from Go SQL strings and the
> `UserPaymentTxnMap` validation map (not a live-checked schema) — cross-verify against actual
> Postgres schema before any migration/integration work.

| Table | Physical DB | Purpose | Columns (code se, with validation-map type/length) |
|---|---|---|---|
| `Glusr_international_txn_detail` | meshpg (`config.GetPGDbConnection("meshpg")`) | International buyer→seller payment-transaction log, invoice-linked | `txn_creation_date` (date), `fk_seller_glusr_id` (number, len 10), `fk_buyer_iso` (string, len 2 — country ISO code), `fk_buyer_glusr_id` (number, len 10), `fk_buyer_name` (string, len 255), `fk_currency_id` (int), `invoice_amount` (number, len 10), `vendor_invoice_url` (string, len 255), `vendor_invoice_id` (string, len 50 — **derived**, not submitted directly), `fk_buyer_email` (string, len 100), `fk_txn_product_details` (string, len 255), `usr_txn_details` (jsonb) — [`UsersValidationMaps.go:796-809`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go), insert list at [`UserAddPaymentTxnDetailsModel.go:38`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddPaymentTxnDetailsModel.go) |

**No `RETURNING` clause on insert** — unlike GST-domain controllers documented in this KT
series, this one doesn't retrieve/return a generated primary key; no auto-increment ID column
is referenced anywhere in the insert or update queries. The table is effectively keyed for
later lookups by `(vendor_invoice_id, fk_seller_glusr_id)` — [`UserAddPaymentTxnDetailsModel.go:38,57`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddPaymentTxnDetailsModel.go).

---

## 4. Business Rules & Validation (exhaustive, code se)

1. **Gateway allowlist**: only `GLADMIN`, `SELLERMY`, `IMOB`, `ANDROID`, `IOS` callers pass
   `Gateway_v1` — anything else gets `"Input gateway not validated"`.
   [`UserUtilsMandatory.go:1965,1985-1986`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
2. **`action` must be exactly `"i"` or `"u"`** — anything else returns `"Please Enter valid
   Action"`. [`UserUtilsMandatory.go:1973-1974`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
3. **Insert (`action=i`) mandatory fields**: `buyer_id`, `seller_id`, `buyer_iso`, `buyer_name`,
   `currency_id`, `amount`, `inv_url` — all must be non-empty strings.
   [`UserUtilsMandatory.go:1975-1976`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
4. **Update (`action=u`) mandatory fields**: only `id` and `seller_id`.
   [`UserUtilsMandatory.go:1977-1978`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
5. **`txn_creation_date` format strictly validated**: must be exactly 14 characters AND
   parseable via Go layout `20060102150405` (`YYYYMMDDHHMMSS`); otherwise marked `"INVALID"` →
   rejected with `" Invalid Transaction Creation Date"`.
   [`UserUtilsMandatory.go:1948-1960,1979-1980`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
6. **Numeric/non-numeric type-check on 6 fields in a single chained condition** —
   `buyer_id`, `seller_id`, `amount` must be numeric; `buyer_iso`, `buyer_name`, `currency_id`
   must **NOT** be numeric (`!IsNumeric` inverted for these three, which is correct intent for
   a country-ISO-code/name/currency-code but easy to misread in the combined `||`).
   [`UserUtilsMandatory.go:1981-1982`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
7. **`txn_details` (if present) must be a JSON object** (`map[string]interface{}`), not a
   string/array/scalar — checked before insert/update via a type-assertion; failing this sets
   `txncheck="INVALID"` → `" Invalid Transaction Details"`.
   [`UserUtilsMandatory.go:1934-1943,1983-1984`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
8. **Length/type validation via shared map** — `LengthAndTypeValidations_v3(params,
   UserPaymentTxnMap)` runs after the manual checks above and re-validates length/type against
   `UserPaymentTxnMap` (section 3); its error (if any) takes priority over the generic-success
   path but after the manual field checks.
   [`UserUtilsMandatory.go:1970,1987-1988`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
9. **Invoice-ID is derived from `inv_url`, not supplied by the caller**: everything after the
   first `#` character in the URL becomes `vendor_invoice_id`. If no `#` is present, or `#` is
   the last character, `invoiceId` stays empty (`hashIndex != -1 && hashIndex < len(invURL)-1`
   guard). [`UserAddPaymentTxnDetailsModel.go:33-36`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddPaymentTxnDetailsModel.go)
10. **Insert writes the full record** (12 columns) in one `INSERT`.
    [`UserAddPaymentTxnDetailsModel.go:38-41`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddPaymentTxnDetailsModel.go)
11. **Update only ever touches `usr_txn_details`** — the `for k, v := range params` loop's
    `switch` statement has exactly one `case` (`"txn_details"`), so even if other fields
    (`buyer_name`, `amount`, etc.) are passed on an `action=u` request, they are silently
    ignored — no error, no partial-update of those columns.
    [`UserAddPaymentTxnDetailsModel.go:46-54`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddPaymentTxnDetailsModel.go)
12. **Update's `WHERE` clause matches on `(vendor_invoice_id, fk_seller_glusr_id)`** — using the
    request's `id` param mapped straight to `vendor_invoice_id` (**not** the invoice-ID derived
    from `inv_url` during insert — on update, `inv_url` isn't even required/parsed since
    `action=u` doesn't mandate `INVOICE_URL`, per rule 4). Caller must already know/track the
    invoice-ID from the original insert response context.
    [`UserAddPaymentTxnDetailsModel.go:57-58`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddPaymentTxnDetailsModel.go)
13. **Update guarded by `len(input_params) != 0`** — the `WHERE` clause and its two bind params
    (`vendor_invoice_id`, `fk_seller_glusr_id`) are only appended **inside** the
    `if len(input_params) != 0` block, i.e. only if `txn_details` was actually present in the
    switch loop above (rule 11). If `txn_details` is absent on an `action=u` call, `query`
    remains the bare string `"UPDATE Glusr_international_txn_detail SET "` with **zero**
    bind params, and this malformed string is still passed to
    `utils.ExecuteQueryRows(meshpgconn, query, input_params, 1*time.Second)` on the next line —
    behavior in that case is **not conclusively traceable from this file alone** (depends on
    how the shared `ExecuteQueryRows` helper and Postgres driver react to an incomplete `SET `
    clause with no `WHERE`/no params — likely a syntax error from Postgres, surfaced as
    `"Execution Failed ..."`, but not directly confirmed here).
    [`UserAddPaymentTxnDetailsModel.go:42-65`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddPaymentTxnDetailsModel.go) — see Open Questions §11.1.
14. **Query timeout is 1 second** (`1*time.Second` passed to `ExecuteQueryRows`) — tighter than
    some other write paths in this codebase; a slow meshpg under load would fail this specific
    call faster than a typical multi-second timeout elsewhere.
    [`UserAddPaymentTxnDetailsModel.go:65`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddPaymentTxnDetailsModel.go)
15. **Response codes are binary on exact string match**: HTTP-level `code`/`status` in the
    controller are `"200"/"SUCCESSFUL"` only if `output` is exactly `"INSERT SUCCESS"` or
    `"UPDATE SUCCESS"` — every validation-rejection message and DB-error message falls through
    to `"500"/"FAILED"`, so callers must string-match on `MESSAGE`, not just `CODE`, to
    distinguish "bad input" from "DB error." [`UserAddPaymentTxnDetailsController.go:60-67`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserAddPaymentTxnDetailsController.go)

---

## 5. RabbitMQ

**Zero RabbitMQ usage found.** Grepped the controller and model files for `PushToQueue`,
`PubAPI`, `Requeue`, `SERVICENAME` publish-calls — none present. No `queue`/`Publish`/`Rabbit`
identifiers anywhere in [`UserAddPaymentTxnDetailsController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserAddPaymentTxnDetailsController.go)
or [`UserAddPaymentTxnDetailsModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserAddPaymentTxnDetailsModel.go).
This is a pure synchronous request/response write with no fan-out.

---

## 6. Kafka

**Zero Kafka usage found.** No `InitializeKafka`, `sub_topic`, `consumer_group`, or `Kafka`
identifiers in either file for this feature.

---

## 7. Redis

**Zero Redis usage found.** No `RedisGet`/`RedisSet`/cache-related identifiers in either file.
There is also no read endpoint for this data at all (section 1), so there is nothing to cache
against currently — a caching gap is moot until a read path exists.

---

## 8. End-to-End Technical Flow

### Insert

```
International Buyer → pays Supplier (via GLADMIN / SELLERMY / IMOB / ANDROID / IOS app)
    │
    ▼
[API — write]  POST /user/addpaymenttxndetails   SERVICENAME=PAYMENT_TXN_DETAILS
                {action="i", buyer_id, seller_id, buyer_iso, buyer_name, currency_id,
                 amount, inv_url, buyer_email?, prod_details?, txn_creation_date?,
                 txn_details?, VALIDATION_KEY}
    │  UserAddPaymentTxnDetailsController.go
    │  1. Gateway allowlist check (GLADMIN/SELLERMY/IMOB/ANDROID/IOS)
    │  2. Mandatory-field presence + numeric/non-numeric type checks
    │  3. txn_creation_date format check (14-char YYYYMMDDHHMMSS)
    │  4. txn_details JSON-object shape check (if present)
    │  5. LengthAndTypeValidations_v3 against UserPaymentTxnMap
    ▼
UpsertPaymentTxnDetails()
    │  invoice-ID extracted from inv_url (text after first '#')
    │  txn_creation_date string parsed into time.Time
    │  txn_details marshaled to JSON bytes
    │  INSERT INTO Glusr_international_txn_detail (12 columns) VALUES (...)
    ▼
[DB — meshpg]   (1-second query timeout)
    ▼
Response {"STATUS":"SUCCESSFUL","CODE":"200","MESSAGE":"INSERT SUCCESS", BUYER_ID, SELLER_ID, SERVICE_NAME}
    + KibanaLogging_v1() call logs timing/execution metrics
```

### Update

```
[API — write]  POST /user/addpaymenttxndetails   SERVICENAME=PAYMENT_TXN_DETAILS
                {action="u", id (=vendor_invoice_id), seller_id, txn_details?, VALIDATION_KEY}
    │  MandatoryParamsCheckPaymentTxnDetails() — only id + seller_id mandatory this path
    ▼
UpsertPaymentTxnDetails()
    │  loop over params, switch only matches "txn_details" key
    │  if txn_details present:
    │      UPDATE Glusr_international_txn_detail SET usr_txn_details=$1
    │      WHERE vendor_invoice_id=$2 AND fk_seller_glusr_id=$3
    │  if txn_details absent:
    │      query string left incomplete ("...SET " with no columns/WHERE) — see §4.13, §11.1
    ▼
[DB — meshpg]   (1-second query timeout)
    ▼
Response {"STATUS":"SUCCESSFUL","CODE":"200","MESSAGE":"UPDATE SUCCESS", ...}  (happy path)
```

No other flow variants exist — there is no read flow, no async fan-out flow, and no
cron-triggered flow for this feature (sections 1, 5-7).

---

## 9. Flow-wise DB & Table Usage — Kaun sa DB, Kaun sa Table, Kis Liye

### Insert flow

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `Glusr_international_txn_detail` | INSERT | Poora transaction record ek hi query mein likha jaata hai — buyer/seller IDs, amount, currency, invoice URL+derived-ID, email, product-details, aur custom `txn_details` JSON |

**Total DB round-trips for insert: 1** (plus the connection-acquire itself, timed separately
as `DB1_CONNECTION_TIME` in the controller's response metadata).

### Update flow

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `Glusr_international_txn_detail` | UPDATE (`usr_txn_details` column only) | Matches on `(vendor_invoice_id, fk_seller_glusr_id)` — sirf custom-JSON field revise karta hai, core transaction data touch nahi hota |

**Total DB round-trips for update: 1** (when `txn_details` is present; see §4.13 for the
guarded no-`txn_details` case).

This is, by a wide margin, the simplest flow documented across this KT series — a single query
per action, no parallel goroutines, no cross-database fan-out, no external API calls.

---

## 10. Optimization Scope — DB Response-Time Contribution

### Low-impact / already efficient

1. **Single-query, no sequential round-trips for either action** — this is about as
   low-latency a write path as the codebase has; there is no obvious query-count optimization
   available here (unlike GST's multi-table fan-out).
2. **No `RETURNING` clause on insert** — since no generated ID is needed by the caller (lookup
   key is the invoice-ID the caller derives itself from `inv_url`), this avoids an
   unnecessary extra round-trip-shape that some other insert-controllers in this codebase
   incur. Not a gap — a deliberate/incidental efficiency.
3. **1-second query timeout** is already tight (§4.14) — no further tuning obviously needed
   from a latency-contributor perspective; if anything, it's tight enough that transient meshpg
   slowness could cause false failures before genuinely slow queries would in other endpoints.
   Worth knowing if this feature ever shows elevated `"Execution Failed"` rates under load —
   check whether it's a real query problem or just this tighter timeout tripping.
4. **No caching applicable** — there is no read endpoint for this data (section 1/7), so a
   Redis-cache recommendation (as made in the GST doc) doesn't apply here; nothing to optimize
   on the read side because there is no read side.

There are no Medium or High-impact findings for this feature — it is a minimal, single-table,
single-query write path with no external dependencies, no message-queue fan-out, and no
consumer chain to introduce latency or failure modes.

---

## 11. Cron Inventory

**None found.** Grepped `user-temp-consumers-production` for any `crons/` directory and for any
file referencing `PaymentTxn`/`payment_txn`/`international_txn`/`txn_detail` — zero matches.
There is no scheduled job that touches `Glusr_international_txn_detail` anywhere in the three
repos searched for this KT.

---

## 12. Edge Cases & Gotchas (technical POV)

1. **Update silently only affects `usr_txn_details`** (§4.11) — a caller expecting to correct
   `amount`/`buyer_name`/other fields via `action=u` will find those fields are simply ignored,
   with no error indicating so. If a wrong amount is inserted, there is **no API-level way to
   fix it** — only a direct DB fix, or a fresh insert (which would create a duplicate record
   with no de-dup mechanism visible in this code).
2. **Guard-condition on update when `txn_details` absent (§4.13, §11.1)** — worth verifying in a
   live/staging environment whether an `action=u` request without `txn_details` results in a
   clean no-op response, or an actual malformed-query execution attempt against meshpg. The
   code as written does **not** short-circuit before calling `ExecuteQueryRows` — it always
   reaches that call, just with an incomplete query string in that specific case.
3. **No `RETURNING`/generated-ID (§3) means the caller must already know/track the
   invoice-derived-ID** to ever update a record later. If `inv_url` didn't contain a `#` at
   insert-time (§4.9), `vendor_invoice_id` is stored empty, and that record becomes effectively
   un-updatable via this endpoint (no meaningful way to `WHERE vendor_invoice_id=''` target a
   single row later, since multiple such rows could exist per seller).
4. **`buyer_iso`/`buyer_name`/`currency_id` "must not be numeric" validation (§4.6)** is easy to
   misread in the combined `||` condition at [`UserUtilsMandatory.go:1981`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
   — worth double-checking if a legitimately numeric currency code (some systems use numeric
   ISO 4217 codes, e.g. `840` for USD) would incorrectly fail this validation and reject an
   otherwise-valid transaction.
5. **Route registered twice** (`router.go:175` and `router.go:343`, same path, same handler) —
   functionally harmless (same controller either way) but worth understanding why, in case one
   router group has different middleware/rate-limits than the other.
6. **1-second query timeout (§4.14)** is tighter than typical — a genuinely slow meshpg query
   under load fails fast here rather than hanging, which is good for the caller's own latency
   but means transient DB blips show up as hard failures more readily than on endpoints with
   longer timeouts.
7. **No read endpoint anywhere** — if a downstream team asks "how do I fetch international
   transaction records," the honest answer per this codebase is: there isn't an API for it;
   direct DB access or a new endpoint would be needed.

---

## 13. Open Questions

1. **What actually happens when `action=u` is submitted without `txn_details`** — clean no-op,
   or a malformed-query execution attempt against meshpg (§4.13, §12.2)? Not conclusively
   traceable from `UpsertPaymentTxnDetails` alone; needs either a direct test against
   `ExecuteQueryRows`'s behavior with an incomplete SQL string, or a live-environment check.
2. **Is there any recovery path for a transaction whose `inv_url` lacked a `#`** (empty
   `vendor_invoice_id`, §12.3) — can it ever be corrected/updated later via this endpoint, or
   any other endpoint/tool?
3. **Numeric-currency-code suppliers** (ISO 4217 numeric codes) — would they fail the
   `currency_id`-must-not-be-numeric check (§4.6, §12.4)? Not testable from code alone without
   knowing what currency codes the actual caller apps send.
4. **Any downstream consumer of this table** (financial-reporting/reconciliation/GST or
   InstaFinance-adjacent systems) — genuinely searched for and **not found** in
   `user-temp-consumers-production` or `users-api-go-production`; either such a consumer lives
   in a repo outside the three searched here, or this data is consumed exclusively via direct
   DB access/BI tooling outside this codebase entirely. Confirm with the finance/BI team.
5. **Why is `POST /user/addpaymenttxndetails` registered on two separate router groups**
   (`router.go:175` and `:343`, §12.5) — same handler both times; is this intentional
   (e.g., public vs. internal-gateway mount) or leftover duplication?
6. **Live DB schema verification** — column types, nullability, indexes, constraints on
   `Glusr_international_txn_detail` — this doc only reflects what the Go SQL strings and
   `UserPaymentTxnMap` imply, not a live schema inspection.

---

## See also

- [`Payment_Txn_Business_Doc.md`](./Payment_Txn_Business_Doc.md) — same flow, product/business
  perspective, bina code ke
- [`../service_api_write_reference.md`](../service_api_write_reference.md) — full write-API
  inventory, this controller's one-line entry corroborates "no side effects" finding
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — depth/structure
  reference this doc was rebuilt against

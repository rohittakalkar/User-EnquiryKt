# Terms And Condition Acceptance — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`Terms_And_Condition_Business_Doc.md`](./Terms_And_Condition_Business_Doc.md)
dekho.

**Repos**: `service-api-go-production` (write only). **Koi read-controller kisi bhi repo
mein nahi mila** — write-only compliance-log, jaisa Popup Details/SellOnIM Log.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write | write | [`UserTncAcceptanceController.go.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTncAcceptanceController.go.go) (`UserTncAcceptanceController`), [`UserTncAcceptanceModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTncAcceptanceModel.go) (`UpsertTncAcceptance`) |
| Validation | write | `MandatoryParamsCheckTncAccp` — [`UserUtilsMandatory.go:200`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go), `UserTncMap` (`LengthAndTypeValidations_v3`) — [`UsersValidationMaps.go:29`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |

**Note**: the controller's filename has a double `.go.go` extension in the repo as-found
— likely an accidental duplicate-extension from a rename, harmless to Go's build (file
still ends in `.go`) but worth cleaning up.

---

## 2. Routes

| Method | serviceName | Repo | Controller |
|---|---|---|---|
| POST | `USER_TERMS_AND_CONDITIONS` | write | `UserTncAcceptanceController` |

---

## 3. Data Model — Table

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `IIL_TERMS_COND_ACCEPTANCE` | meshpg | Append-only per-acceptance compliance log | `fk_glusr_usr_id`, `IIL_TERMS_COND_IP`, `IIL_TERMS_COND_IP_country`, `IIL_TERMS_COND_COUNTRY_ISO`, `IIL_TERMS_COND_USER_AGENT`, `IIL_TERMS_COND_DATE` (`YYYYMMDD` string format, app-generated not `CURRENT_TIMESTAMP`) — [`UserTncAcceptanceModel.go:29-45`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTncAcceptanceModel.go) |

---

## 4. Business Rules & Validation (code se)

1. **Gateway allowlist is very large (33 entries)** — same "widely-integrated" pattern
   as Popup Details and Negative Mcat: `MY`, `GLADMIN`, `TOLLFREE`, `Email Marketing`,
   `HTVENDOR`, `Weberp`, `LEAP`, `IPC-Admin`, `NSD Sales`, `MAPI`, `ENQUIRY`,
   `FREE-WEBSITE`, `M.INDIAMART.COM`, `TRADE`, `BL`, `TENDER`, `PAYNOW`,
   `CREDIT ALLOCATION`, `OVP Process`, `SAMPARK Process`, `TOLLFREE Process`,
   `VENDOR CITY Pin Correction`, `Notification Server`, `search`, `FCP`, `PNS`, `Merp`,
   `IMOB`, `SELLERMY`, `PAYWIM`, `BUYERS_FEEDBACK`, `FLPNS`, `LEAPIN`, `ANDROID`, `IOS`.
   [`UserTncAcceptanceController.go.go:51`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTncAcceptanceController.go.go)
2. **Mandatory fields**: `MODID`, `VALIDATION_KEY`, `IP`, `USR_ID`, `USER_AGENT` (plus
   `IP_COUNTRY`/`IP_COUNTRY_ISO` per `UserTncMap`).
   [`UserUtilsMandatory.go:200-225`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
3. **Unusual validation-error-forgiveness**: if the length/type-validation result
   contains `"is a wrong parameter."` but does NOT also contain `" length exceeded."` or
   `" should be numeric."`, the request is **still allowed to proceed** to the insert —
   only length/numeric-type failures are treated as hard-blocking; an unrecognized/
   "wrong parameter" name alone doesn't stop the insert. This is more lenient than every
   other controller documented in this KT series so far.
   [`UserTncAcceptanceController.go.go:60`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTncAcceptanceController.go.go)
4. **Dynamic column-list is built by iterating `UserTncMap`'s keys**, not the request
   params — only fields present in both the request AND `UserTncMap` get included;
   `IIL_TERMS_COND_DATE` is always appended (app-generated `time.Now().Format("20060102")`
   — a plain date-string, not a DB-timestamp function).
   [`UserTncAcceptanceModel.go:31-45`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTncAcceptanceModel.go)
5. **Custom string-escaping via `formatinput()`** — uses `strconv.QuoteToASCII` then
   strips the surrounding quotes, rather than relying purely on parameterized-query
   escaping (which the code already uses via `$1`/`$2` placeholders) — a redundant/
   unusual extra escaping-layer worth noting, though not necessarily harmful since it's
   layered on top of (not instead of) parameterized queries.
   [`UserTncAcceptanceModel.go:94-103`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTncAcceptanceModel.go)
6. **On INSERT success, a RabbitMQ event fires** with `SERVICENAME=TNC_ACCEPTANCE`,
   `STATE="9"` — a hardcoded state-code with no in-code explanation of what state `9`
   means.
   [`UserTncAcceptanceModel.go:70-75`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTncAcceptanceModel.go)
7. **Detailed error-diagnostics captured on failure**: `getErrorLine()` uses
   `runtime.Caller(1)` to capture the file/line of the error for the response/log — an
   unusually developer-facing debug-aid embedded directly in the production response-path
   (`extraParamsForOutput["Error"]`).

---

## 5. RabbitMQ / Kafka / Redis

| Queue / `SERVICENAME` | Publisher | Consumer | Purpose |
|---|---|---|---|
| `TNC_ACCEPTANCE` → routes to `SESSION_UPDATE` (per domain-wide `serviceToQueueMap`) | `UserTncAcceptanceModel.go` (`UpsertTncAcceptance`) — only on insert success | **Not found in `user-temp-consumers-production`** | Presumably updates supplier's session/status somewhere downstream — consumer not traceable from these 3 repos |

**Koi Kafka ya Redis usage nahi mila.**

---

## 6. End-to-End Technical Flow

```
Supplier (via any of ~33 allowed internal-apps/channels)
    │
    ▼
[API — write]  POST serviceName=USER_TERMS_AND_CONDITIONS
                {USR_ID, MODID, VALIDATION_KEY, IP, IP_COUNTRY, IP_COUNTRY_ISO,
                 USER_AGENT}
    │  UserTncAcceptanceController.go.go
    │  MandatoryParamsCheckTncAccp() → Gateway (~33-entry allowlist)
    │  LengthAndTypeValidations_v3() — lenient on unrecognized-param-name errors
    ▼
UpsertTncAcceptance()
    │  INSERT INTO IIL_TERMS_COND_ACCEPTANCE (dynamic columns from UserTncMap,
    │  + IIL_TERMS_COND_DATE = app-generated YYYYMMDD)
    ▼
[DB — meshpg]
    │  on success:
    ▼
[RabbitMQ]  SERVICENAME=TNC_ACCEPTANCE → SESSION_UPDATE  {STATE:"9", GLUSR_ID}
    ▼
Response {"INSERT SUCCESS INTO MESH PG"}
```

---

## 7. Optimization Scope — DB Response-Time Contribution

### Low impact / good practice already present

1. **Single-insert, no sequential round-trips** — straightforward, low-risk write path.
2. **No async fan-out complexity beyond the single RabbitMQ push** — simple architecture.

### Design-scope note (not performance)

3. **Custom `formatinput()` escaping layered on top of parameterized queries** — not a
   performance concern, but a maintenance-clarity one: a future reader might assume it's
   the sole injection-defense and be confused about why it coexists with `$N` placeholders.

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora Terms And Condition flowchart yahan dekho](https://lucid.app/lucidchart/5b2f20ba-6da1-4777-bcf8-951dd08a9978/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **Lenient validation-forgiveness for "wrong parameter" errors** (§4, point 3) — a
   caller could send an unrecognized extra field-name and still succeed, unlike most
   other controllers where any validation-failure blocks the write; if this was meant to
   only be lenient for genuinely-extra/ignorable fields, confirm it doesn't accidentally
   let malformed *known* fields with typo'd names silently get dropped rather than
   flagged.
2. **`SESSION_UPDATE` consumer not found in this codebase** — same "queue with no
   traceable in-repo consumer" pattern seen with Social Contacts' async producer (there
   it was a missing producer, here it's a missing consumer) — downstream effect can't be
   fully verified from code alone.
3. **`STATE="9"` is an unexplained magic-value** — no enum/comment describing what
   session-states exist or what `9` represents.
4. **Filename typo** (`UserTncAcceptanceController.go.go`) — cosmetic but worth a cleanup
   PR at some point.

---

## 10. Open Questions

1. What does `STATE="9"` represent in the `SESSION_UPDATE` message?
2. Which system actually consumes the `SESSION_UPDATE` queue and what does it do with
   a TNC-acceptance event?
3. Is the validation-leniency for "wrong parameter" errors (§4, point 3) intentional
   design, or an oversight?
4. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`Terms_And_Condition_Business_Doc.md`](./Terms_And_Condition_Business_Doc.md) — product perspective

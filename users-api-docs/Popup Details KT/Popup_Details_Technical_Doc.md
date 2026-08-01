# Popup Details — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`Popup_Details_Business_Doc.md`](./Popup_Details_Business_Doc.md)
dekho.

**Repos**: `service-api-go-production` (write only). **Koi read-controller, RabbitMQ,
Kafka, ya consumer nahi mila** — sabse simplest/smallest feature is series mein abhi tak,
sirf ek write-only upsert-endpoint.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write | write | [`PopupDetailsController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/PopupDetailsController.go), [`PopupDetailsModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PopupDetailsModel.go) (`PopupDetailsModel`) |

---

## 2. Routes

| Method | serviceName | Repo | Controller |
|---|---|---|---|
| POST | `POPUP_DETAILS` | write | `PopupDetailsController` |

---

## 3. Data Model — Table

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `TRACK_POPUP_MDC` | meshpg | Latest popup-interest-status per supplier | `fk_glusr_usr_id` (unique — `ON CONFLICT` target), `is_interested` (0/1), `last_entry_date` — [`PopupDetailsModel.go:42`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PopupDetailsModel.go) |

**Note**: table-name suffix `MDC` ka exact matlab code se resolve nahi hua — dekho Open
Questions.

---

## 4. Business Rules & Validation (code se)

1. **Gateway allowlist is the largest seen across all KT folders so far** — 30 entries:
   `MY`, `M.INDIAMART.COM`, `GLADMIN`, `TOLLFREE`, `Email Marketing`, `HTVENDOR`,
   `WEBERP`, `LEAP`, `IPC-Admin`, `NSD Sales`, `MAPI`, `ENQUIRY`, `FREE-WEBSITE`, `BL`,
   `TRADE`, `TENDER`, `PAYNOW`, `CREDIT ALLOCATION`, `OVP Process`, `SAMPARK Process`,
   `TOLLFREE Process`, `VENDOR CITY Pin Correction`, `Notification Server`, `search`,
   `FCP`, `PNS`, `Merp`, `IMOB`, `SELLERMY`, `PAYWIM`, `BUYERS_FEEDBACK`.
   [`PopupDetailsController.go:15-47`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/PopupDetailsController.go)
2. **In-app validations before DB-call**: `glusridval` mandatory + numeric,
   `is_interested` must be `"0"` or `"1"` — all checked without a DB round-trip first.
   [`PopupDetailsController.go:101-111`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/PopupDetailsController.go)
3. **Single upsert statement**: `INSERT ... ON CONFLICT (fk_glusr_usr_id) DO UPDATE SET
   is_interested=$2, last_entry_date=CURRENT_TIMESTAMP` — one query handles both
   first-time and repeat submissions, no explicit insert-vs-update branching needed
   (best-practice pattern, consistent with Social Contact's async-path and Flips'
   `GLUSR_FLIPS_MAP`).
   [`PopupDetailsModel.go:42`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PopupDetailsModel.go)
4. **No history preserved** — each submission overwrites the previous `is_interested`
   value; only the current/latest status is ever queryable from this table.

---

## 5. RabbitMQ / Kafka / Redis

**Koi RabbitMQ, Kafka, ya Redis usage nahi mila.**

---

## 6. End-to-End Technical Flow

```
Supplier (via any of ~30 allowed internal-apps)
    │
    ▼
[API — write]  POST serviceName=POPUP_DETAILS  {glusridval, is_interested(0/1), VALIDATION_KEY}
    │  PopupDetailsController.go
    │  In-app checks: glusridval numeric, is_interested in {0,1}
    │  Gateway_v1() — ~30-entry allowlist
    ▼
PopupDetailsModel()
    │  INSERT INTO TRACK_POPUP_MDC (fk_glusr_usr_id, is_interested, last_entry_date)
    │  VALUES ($1,$2,CURRENT_TIMESTAMP)
    │  ON CONFLICT (fk_glusr_usr_id) DO UPDATE SET is_interested=$2, last_entry_date=NOW()
    ▼
[DB — meshpg]  TRACK_POPUP_MDC
    ▼
Response {"INSERT SUCCESS"}
```

---

## 7. Optimization Scope — DB Response-Time Contribution

### Low impact / good practice already present

1. **Single `ON CONFLICT` upsert-statement** — already the most efficient pattern for
   this kind of latest-status-only write; no round-trip-reduction opportunity here.
2. **In-app validation before DB-connection** — invalid `is_interested`/`glusridval`
   values reject before ever touching the DB, avoiding wasted connections.
3. **No async/consumer complexity** — simplest, lowest-risk architecture in this KT
   series so far.

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora Popup Details flowchart yahan dekho](https://lucid.app/lucidchart/37196057-6da9-4c54-afab-3556775a2e2b/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **No history** — if a business-need ever arises to see *when* a supplier changed
   their interest-status multiple times, this table cannot answer that; only the latest
   value survives.
2. **Table-name `MDC` suffix is unexplained** — any future debugging/dashboard work
   referencing this table should first confirm what "MDC" stands for.

---

## 10. Open Questions

1. `TRACK_POPUP_MDC` — what does "MDC" stand for, and which specific popup/campaign
   does this track? Not resolvable from code alone.
2. Which screen(s)/apps actually show this popup to the supplier — not traceable from
   this codebase (client-side/frontend concern).
3. Is there any downstream consumer of this data (reporting, targeting) outside these
   3 repos?
4. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`Popup_Details_Business_Doc.md`](./Popup_Details_Business_Doc.md) — product perspective

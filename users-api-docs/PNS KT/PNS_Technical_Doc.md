# PNS (GSM Number Allocation) — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`PNS_Business_Doc.md`](./PNS_Business_Doc.md) dekho.

**Scope note**: PNS aur PNS Setting genuinely **do alag concepts** hain — alag tables
(`GL_GSM_MASTER` vs `IIL_PNS_SETTING`), alag controllers, koi FK-relation ya
shared-code nahi mila. Dekho
[`../PNS Setting KT/PNS_Setting_Technical_Doc.md`](../PNS%20Setting%20KT/PNS_Setting_Technical_Doc.md).

**Repos**: `service-api-go-production` (write only). **Koi read-controller kisi bhi
repo mein nahi mila.**

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write | write | [`PnsTxnController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/PnsTxnController.go) (`UserPnsAllocationController`), [`PnsTxnModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PnsTxnModel.go) (`UpsertPnstxn`) |
| Validation | write | `MandatoryParamsCheckPnsTxn` — [`UserUtilsMandatory.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go) |

---

## 2. Routes

| Method | serviceName | Repo | Controller |
|---|---|---|---|
| POST | `GSM_MASTER_SERVICE` | write | `UserPnsAllocationController` |

---

## 3. Data Model — Table

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GL_GSM_MASTER` | meshpg | Shared pool of GSM/virtual call-tracking numbers | `gl_gsm_number`, `gl_gsm_vendor_type`, `FLAG_IS_AVAILABLE` (`2`=available, `0`/`-1`=allocated-variants), `GL_GSM_MOD_DATE` — [`PnsTxnModel.go:235,237`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PnsTxnModel.go) |

---

## 4. Business Rules & Validation (code se)

1. **Gateway allowlist**: same large ~33-entry allowlist seen in Terms And Condition/
   Popup Details KTs (`MY`, `GLADMIN`, `TOLLFREE`, `Email Marketing`, `HTVENDOR`,
   `Weberp`, `LEAP`, `IPC-Admin`, `NSD Sales`, `MAPI`, `ENQUIRY`, `FREE-WEBSITE`,
   `M.INDIAMART.COM`, `TRADE`, `BL`, `TENDER`, `PAYNOW`, `CREDIT ALLOCATION`,
   `OVP Process`, `SAMPARK Process`, `TOLLFREE Process`,
   `VENDOR CITY Pin Correction`, `Notification Server`, `search`, `FCP`, `PNS`,
   `Merp`, `IMOB`, `SELLERMY`, `PAYWIM`, `BUYERS_FEEDBACK`, `FLPNS`, `LEAPIN`,
   `ANDROID`, `IOS`). [`PnsTxnController.go:52`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/PnsTxnController.go)
2. **Allocation query picks the first available-number for the requested vendor-type**:
   `WHERE gl_gsm_number = (SELECT gl_gsm_number FROM gl_gsm_master WHERE
   gl_gsm_vendor_type=$1 AND flag_is_available=2 LIMIT 1) RETURNING gl_gsm_number` —
   a subquery-based "claim one row" pattern, not `SELECT ... FOR UPDATE SKIP LOCKED` or
   similar concurrency-safe claiming mechanism (see Edge Cases).
3. **Two variant flag-values on allocation** (`FLAG_IS_AVAILABLE = 0` vs `= -1`) —
   distinguished by some input-condition not fully traced in this pass (likely an
   action-type/flag param), suggesting two different allocation-outcomes/states beyond
   simple available/unavailable.
   [`PnsTxnModel.go:235,237`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PnsTxnModel.go)
4. **On success, a RabbitMQ event fires** with `SERVICENAME=PNS_SERVICE`, `TABLES=
   GL_GSM_MASTER` embedded in the message.

---

## 5. RabbitMQ / Kafka / Redis

| Queue / `SERVICENAME` | Publisher | Consumer | Purpose |
|---|---|---|---|
| `PNS_SERVICE` | `PnsTxnModel.go` (`UpsertPnstxn`) | **Not confirmed in this pass** | Downstream sync of GSM-allocation events |

**Koi Kafka ya Redis usage nahi mila.**

---

## 6. End-to-End Technical Flow

```
Internal-tool (~33-entry allowlist)
    │
    ▼
[API — write]  POST serviceName=GSM_MASTER_SERVICE  {USR_ID, vendor_type, ...}
    │  UserPnsAllocationController.go — Gateway check
    │  MandatoryParamsCheckPnsTxn()
    ▼
UpsertPnstxn()
    │  UPDATE GL_GSM_MASTER SET FLAG_IS_AVAILABLE=0/-1, GL_GSM_MOD_DATE=NOW()
    │  WHERE gl_gsm_number = (SELECT ... vendor_type match, available=2, LIMIT 1)
    │  RETURNING gl_gsm_number
    ▼
[DB — meshpg]
    │  on success:
    ▼
[RabbitMQ]  SERVICENAME=PNS_SERVICE  {TABLES: GL_GSM_MASTER}
    ▼
Response {"SUCCESS"}
```

---

## 7. Optimization Scope — DB Response-Time Contribution

### Low impact

1. **Single subquery-based UPDATE, no sequential round-trips** — straightforward
   pattern for a pool-claiming operation.

### Concurrency-scope note (not raw performance)

2. **Subquery `LIMIT 1` without row-locking** (no `FOR UPDATE SKIP LOCKED`) — under
   concurrent allocation-requests, this pattern can be prone to two requests racing for
   the same row between the subquery's read and the outer `UPDATE`'s write; Postgres's
   MVCC generally protects against a double-allocation of the exact same row in this
   specific subquery-in-UPDATE form (the UPDATE re-checks the WHERE-matched row), but
   it's still worth flagging if allocation-throughput becomes high enough for this to
   matter (see Edge Cases).

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora PNS flowchart yahan dekho](https://lucid.app/lucidchart/e05f5144-4253-42c5-af97-aa43b40da7f0/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **No `FOR UPDATE SKIP LOCKED` pattern** — under heavy concurrent-allocation-load,
   requests could contend on the same candidate-row before one successfully claims it;
   worth load-testing if allocation-volume ever spikes.
2. **Two flag-values (`0` vs `-1`) on "allocated" state aren't fully explained in
   code** — the branching condition wasn't traced in this pass.
3. **Koi read-endpoint nahi mila** — checking a supplier's currently-allocated GSM-
   number requires either direct-DB-access or another undocumented endpoint.

---

## 10. Open Questions

1. What distinguishes the `FLAG_IS_AVAILABLE=0` vs `=-1` allocation-outcome paths?
2. Who consumes the `PNS_SERVICE` RabbitMQ event — not found in this pass.
3. Is there a release/deallocation path (numbers returning to the pool) — not found in
   this controller; may exist elsewhere.
4. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`PNS_Business_Doc.md`](./PNS_Business_Doc.md) — product perspective
- [`../PNS Setting KT/PNS_Setting_Technical_Doc.md`](../PNS%20Setting%20KT/PNS_Setting_Technical_Doc.md) —
  unrelated concept, documented separately (no shared table/FK/controller)

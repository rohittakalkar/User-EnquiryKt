# Company Logo — Technical Doc (Code-Level Deep Dive)

**Scope note**: yeh doc sirf **Logo** (`GLUSR_USR_LOGO`) cover karta hai — **Image**
(`GLUSR_USR_IMAGE`, general branding photos) ek genuinely alag table/write-path hai, dekho
[`../Image KT/Image_Technical_Doc.md`](../Image%20KT/Image_Technical_Doc.md). Dono
`FK_GLUSR_USR_ID` se supplier se link hain, lekin ek doosre se FK-linked nahi hain — no
DB-level coupling mila.

Business/product perspective ke liye [`Logo_Business_Doc.md`](./Logo_Business_Doc.md) dekho.

**Repos**: `service-api-go-production` (write), `user-temp-consumers-production` (3 fan-out
consumers).

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Logo write | write | [`UserLogoController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserLogoController.go), [`UserLogoModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserLogoModel.go) (`UpsertLogo`, `upsert_logo_pg`) |
| Read (embedded elsewhere) | read | `otherdetail` endpoint ke through reads hota hai (per earlier product-story research), koi dedicated `GET /user/logo` controller nahi mila |
| Fan-out — search/IMSDB replica | consumers | [`USER_COMPSYNC_LOGO_IMSDB.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_COMPSYNC_LOGO_IMSDB.go) |
| Fan-out — LMS | consumers | `USER_LOGO_LMSPG.go` |
| Fan-out — general company-sync | consumers | `USER_COMP_SYNC_LOGO.go` |

---

## 2. Routes

| Method | Path | Repo | Controller |
|---|---|---|---|
| POST | `/user/logo` | write | `UserLogoController` |

---

## 3. Data Model — Table

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_USR_LOGO` | meshpg (write) + replica synced via consumers | Logo record with **approval-workflow** | `GLUSR_LOGO_APPROVAL_ID` (PK, `RETURNING` on insert), `FK_GLUSR_USR_ID`, `GLUSR_LOGO_APPROVAL_STATUS` (`'P'` = Pending on insert), `GLUSR_LOGO_REJECTION_REASON`, `GLUSR_LOGO_UPDATEDBY_ID`/`_UPDATEDBY`/`_UPDATEDBY_AGENCY`, `GLUSR_LOGO_UPDATESCREEN`, `GLUSR_LOGO_IP`/`_IP_COUNTRY`, `GLUSR_LOGO_HIST_COMMENTS`, `GLUSR_LOGO_UPDATED_TIME`, `GLUSR_USR_LOGO_IMG_90X90`/`_120X120`/`_ORIGINAL`/`_250X250` — [`UserLogoModel.go:95-97`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserLogoModel.go) |

---

## 4. Business Rules & Validation (code se)

1. **Gateway allowlist chhota hai — `MY`/`GLADMIN`**.
   [`UserLogoController.go:50`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserLogoController.go)
2. **Three-way branch on `FLAG`**: `"I"` (Insert — new logo, `GLUSR_LOGO_APPROVAL_STATUS`
   hardcoded to `'P'`), `"U"` (Update — existing logo's image/status), aur ek third
   fallback-branch (metadata-only update, no image columns) — confirmed via
   `upsert_logo_pg`'s three query-variants.
   [`UserLogoModel.go:93-108`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserLogoModel.go)
3. **`APPROVAL_ID` presence decides insert-vs-update path entirely** — agar `APPROVAL_ID`
   empty hai, tabhi `FLAG`-based branching hoti hai; agar already present hai, ek alag
   (`else`) path lega — **is pass mein `APPROVAL_ID`-present branch ka poora code trace
   nahi hua**, dekho Open Questions.
4. **`UPDATEDBY_ID` present ho toh, employee-name lookup hota hai** (`Employee_mesh_pg`)
   pehle DB-write se — matlab agar koi internal-employee ne logo update kiya, unka naam
   record mein human-readable form mein save hota hai.
   [`UserLogoModel.go:31-46`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserLogoModel.go)
5. **`"Record already exist"` bhi success treat hota hai** (controller-level check:
   `output == "UPDATE SUCCESS" || ... || output == "Record already exist"` → code 200) —
   idempotent-friendly design, duplicate-submission error jaisa nahi treat hota.
   [`UserLogoController.go:77`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserLogoController.go)

---

## 5. RabbitMQ

| Queue / `SERVICENAME` | Publisher | Consumer(s) | Purpose |
|---|---|---|---|
| `COMPANY_LOGO_SERVICE` → routes to `user.logo.<glid%20>` (per domain-wide `serviceToQueueMap`, confirmed earlier this session) | `UserLogoModel.go` (`upsert_logo_pg`) | [`USER_COMPSYNC_LOGO_IMSDB.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_COMPSYNC_LOGO_IMSDB.go), `USER_LOGO_LMSPG.go`, `USER_COMP_SYNC_LOGO.go` | 3-way fan-out — search/IMSDB replica, LMS, general company-sync all independently keep their own `GLUSR_USR_LOGO` copy in sync |

**Interesting finding**: `USER_COMPSYNC_LOGO_IMSDB.go`'s UPDATE query has a conditional
variant — `WHERE FK_GLUSR_USR_ID=$N AND GLUSR_LOGO_APPROVAL_STATUS = 'P'` — matlab **kuch
updates sirf tabhi apply hote hain jab record abhi bhi "Pending" ho**, shayad race-condition
avoid karne ke liye (agar beech mein koi aur process already approve/reject kar chuka ho,
yeh stale update silently no-op ho jaaye).
[`USER_COMPSYNC_LOGO_IMSDB.go:151`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_COMPSYNC_LOGO_IMSDB.go)

---

## 6. Kafka / Redis

**Koi Kafka ya Redis usage nahi mila** iss feature mein.

---

## 7. End-to-End Technical Flow

```
Supplier
    │
    ▼
[API — write]  POST /user/logo  {FLAG: "I"/"U", USR_ID, IMG_FORM_90x90, ...}
    │  UserLogoController.go
    │  Gateway check (MY/GLADMIN)
    ▼
UpsertLogo()
    │
    ├─ UPDATEDBY_ID present → Employee_mesh_pg() lookup (employee-name resolve)
    │
    ▼
upsert_logo_pg()
    │
    ├─ APPROVAL_ID empty + FLAG="I" → INSERT (status forced to 'P') RETURNING GLUSR_LOGO_APPROVAL_ID
    ├─ APPROVAL_ID empty + FLAG="U" → UPDATE (status/images/reject-reason)
    ├─ APPROVAL_ID empty + no FLAG   → UPDATE (metadata-only, no images)
    └─ APPROVAL_ID present           → *(not fully traced in this pass)*
    ▼
[RabbitMQ]  SERVICENAME=COMPANY_LOGO_SERVICE → user.logo.<glid%20>
    ▼
[CONSUME — 3-way parallel fan-out]
    ├─ USER_COMPSYNC_LOGO_IMSDB (search replica, conditional on approval-status)
    ├─ USER_LOGO_LMSPG (LMS sync)
    └─ USER_COMP_SYNC_LOGO (general company-sync)
```

---

## 8. Optimization Scope — DB Response-Time Contribution

### Low-Medium impact

1. **`Employee_mesh_pg` lookup ek extra sequential DB-round-trip hai** before the main
   upsert — sirf `UPDATEDBY_ID` present hone par. Agar high-frequency internal-tool se aata
   hai, yeh do sequential queries ek transaction mein combine ho sakti hain, ya
   employee-name-cache (Redis) se skip ho sakti hain agar employee-list slow-changing hai.
2. **3-way consumer fan-out, sab independent RabbitMQ consumers** — GST/Rating domains jaisa
   hi pattern, koi obvious inefficiency nahi, standard replication-cost hai.

### Low-impact / good practice already present

3. **Conditional `WHERE ... AND GLUSR_LOGO_APPROVAL_STATUS = 'P'`** (§5) — achi
   race-condition-awareness, stale writes silently ignore hoti hain.
4. **"Record already exist" ko success treat karna** (§4, point 5) — idempotent-friendly,
   client-retries ko error nahi dikhata.

---

## 9. Full Flow Diagram (Lucid, icon-based)

**[Poora Logo flowchart yahan dekho](https://lucid.app/lucidchart/6d2b3365-55df-4e92-b1e0-c700b58a9786/edit)**

---

## 10. Edge Cases & Gotchas (technical POV)

1. **`APPROVAL_ID`-present branch ka code trace nahi hua** (§4, point 3) — agar update/edit
   flow debug karna pade jahan `APPROVAL_ID` already pass ho raha ho, is file ko dobara
   padhna padega.
2. **`GLUSR_LOGO_APPROVAL_STATUS` ke poore possible values (`P` ke alawa) decode nahi hue** —
   likely Approved/Rejected codes bhi hain, exact letters/values confirm nahi hue.
3. **Koi dedicated read-controller nahi mila** — logo `otherdetail` ke through hi read hota
   hai (per earlier session research) — agar "logo dikh nahi raha" ka debugging ho, wahan
   dekhna padega.

---

## 11. Open Questions

1. `APPROVAL_ID`-present write-path ka poora code kya karta hai?
2. `GLUSR_LOGO_APPROVAL_STATUS` ke saare possible values aur unka meaning?
3. Actual approval-action (Pending → Approved/Rejected transition) kaun trigger karta hai —
   koi admin-tool/consumer iss pass mein nahi mila?
4. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`Logo_Business_Doc.md`](./Logo_Business_Doc.md) — product perspective
- [`../Image KT/Image_Technical_Doc.md`](../Image%20KT/Image_Technical_Doc.md) — related but
  separate general-image system (no approval-ID, no consumer fan-out)
- [`../users_domain_product_stories.md`](../users_domain_product_stories.md) — Story 2
  "Profile & Business Identity Management"

# Visiting Card — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`Visiting_Card_Business_Doc.md`](./Visiting_Card_Business_Doc.md)
dekho.

**Repos**: `service-api-go-production` (write), `users-api-go-production` (read),
`user-temp-consumers-production` (approval-consumer). This is one of the most
`TYPE`/`UPDATE_FLAG`-branch-heavy controllers documented in this KT series — comparable
in complexity to TrustSeal/GST.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write | write | [`VisitingCardController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/VisitingCardController.go), [`VisitingCardModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/VisitingCardModel.go) (`UpsertVisitingCard`, `updatePG`, `upsertNoVisitCard`) |
| Validation | write | `MandatoryParamsCheckVisitingCard` — [`UserUtilsMandatory.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go), `UserVisitingCardMap` — [`UsersValidationMaps.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |
| Approval-consumer | consumers | [`USER_VISITINGCARD_APPROVAL.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VISITINGCARD_APPROVAL.go) (`dbActionUserVisitingCardApproval`) — writes `approvalPg` |
| Read (display) | read | [`VcardDisplayControllers.go`](../../users-api-go-production/internal/controllers/UsersControllers/VcardDisplayControllers.go) (`ActionVcardDisplay`), [`UserVcardDisplayModel.go`](../../users-api-go-production/internal/models/users/UserVcardDisplayModel.go) |

---

## 2. Routes

| Method | serviceName | Repo | Controller |
|---|---|---|---|
| POST | `VISITING_CARD_SERVICE` | write | `VisitingCardController` |
| — | `DISPLAY_VISITING_CARD` | read | `ActionVcardDisplay` |

---

## 3. Data Model — Tables

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `STS_COMP_VISIT_CARD` | meshpg (write) | The visiting-card record itself | `VISITING_CARD_ID` (PK, `RETURNING` on insert), `FK_GLUSR_USR_ID`, `FRONT_ATTACHMENT`/`BACK_ATTACHMENT`, `FRONT_ENTERED_DATE`/`BACK_ENTERED_DATE`, `FRONT_ENTERED_BY`/`BACK_ENTERED_BY`, `FRONT_DESIGN_UPDATE`/`BACK_DESIGN_UPDATED`, `ENRICHED_FLAG`/`ENRICHED_BY`/`ENRICHED_DATE`, `APPROV_STATUS`, `DESIGN_FLAG`/`DESIGN_APPROVED_BY`/`DESIGN_APPROVED_DATE`, `FLAG_DOCS` (hardcoded `2` on write), `STS_COMP_VC_LATITUDE`/`_LONGITUDE`, `VISITING_CARD_ADMIN_COMMENT`, `STS_COMP_VC_UPDATEDBY`/`_UPDATESCREEN`/`_IP`/`_IP_COUNTRY`/`_UPDATEDUSING`/`_UPDATEDBY_URL`/`_UPDATEDBY_ID`, `UPDATE_DATE`, `UPDATED_BY_USING` |
| `STS_COMP_NO_VISIT_CARD` | meshpg | "Supplier has no visiting card" status, upsert on `FK_GLUSR_USR_ID` | `FK_GLUSR_USR_ID` (unique — `ON CONFLICT` target), `STS_COMP_NO_VC_APPROVAL_STATUS` (`Y`/`N`/`P`), `STS_COMP_NO_VC_UPDATEDBY`/`_UPDATEDUSING`/`_UPDATEDDATE`/`_ENTEREDDATE` — [`VisitingCardModel.go:60-73`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/VisitingCardModel.go) |

---

## 4. Business Rules & Validation (code se)

1. **Gateway allowlist**: `GLADMIN`, `MY`, `Weberp`, `M.INDIAMART.COM`, `MAPI`, `Merp`.
   [`VisitingCardController.go:50`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/VisitingCardController.go)
2. **`out_iu` (insert-vs-update) determined by 3 checks in priority order**:
   `VISITING_CARD_ID` present → `UPDATE`; else `TYPE=="NO_VC"` → `NO_VC REQUEST`; else
   `INSERT`. [`VisitingCardModel.go:25-31`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/VisitingCardModel.go)
3. **`TYPE=NO_VC` (regardless of insert/update-path) → `upsertNoVisitCard()`** — a
   direct `ON CONFLICT (FK_GLUSR_USR_ID) DO UPDATE` upsert into
   `STS_COMP_NO_VISIT_CARD` with status `'N'`.
4. **Update-path, when `VISITING_CARD_ID` is given, branches on `TYPE`**:
   - **`TYPE=GLADMIN`**: dynamic whitelist-driven UPDATE (via `UserVisitingCardMap`) on
     `STS_COMP_VISIT_CARD` — mainly used for admin-comments (`OPENING_COMMENT` →
     `VISITING_CARD_ADMIN_COMMENT`); several `*_BY` fields get `CURRENT_TIMESTAMP`
     forced instead of their literal value if present in the request (treated as
     "touch this date" markers, not literal date-values).
   - **`TYPE=WEBERP`**: dispatches further on `UPDATE_FLAG`:
     - `ENRICH` — sets `ENRICHED_DATE`/`ENRICHED_BY`/`ENRICHED_FLAG` + admin-comment.
     - `APPROVE` — sets `APPROV_STATUS` + `UPDATE_DATE`.
     - `DESIGN` — sets `DESIGN_APPROVED_DATE`/`DESIGN_APPROVED_BY`/`DESIGN_FLAG`.
     - `ENRICHDESIGN` — does BOTH enrich and design fields in one statement
       (`ENRICHED_FLAG=1` hardcoded), plus admin-comment.
     - anything else → `"UPDATE_FLAG can not same as desired."`
   - **Neither GLADMIN nor WEBERP (i.e. field-capture path), branches on `MODID=="0"`**:
     - `modid=="0"` (field-capture): updates `FRONT_ATTACHMENT`/`BACK_ATTACHMENT`
       (whichever provided — 3 query-variants for front-only/back-only/both), and
       **explicitly resets `ENRICHED_FLAG`/`APPROV_STATUS`/`DESIGN_FLAG`/
       `DESIGN_APPROVED_BY`/`DESIGN_APPROVED_DATE` to `NULL`** — a fresh photo restarts
       the review-cycle. Also sets `FLAG_DOCS=2`, captures lat/long.
     - else (design-update path): updates `FRONT_DESIGN_UPDATE`/`BACK_DESIGN_UPDATED`
       (3 variants similarly), sets `DESIGN_FLAG=1`, same review-reset pattern.
   [`VisitingCardModel.go:96-390`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/VisitingCardModel.go)
5. **After a successful update (any branch), a second query conditionally
   upserts `STS_COMP_NO_VISIT_CARD`**: `APPROV_STATUS=='R'` (rejected) AND
   `COUNT==0` → set No-VC status to `'N'`; otherwise → set to `'Y'` — meaning **any
   successful visiting-card-update (other than a first-rejection) marks "has visiting
   card = Yes"** in the companion table.
   [`VisitingCardModel.go:417-451`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/VisitingCardModel.go)
6. **Insert-path (`VISITING_CARD_ID` absent, `TYPE!="NO_VC"`)**: requires at least one
   of `FRONT_ATTACHMENT`/`BACK_ATTACHMENT`/`FRONT_DESIGN_UPDATE`/`BACK_DESIGN_UPDATED`;
   inserts into `STS_COMP_VISIT_CARD` (`FLAG_DOCS=2` hardcoded), then always follows
   with `upsertNoVisitCard(..., "Y")` — a fresh insert immediately marks "has visiting
   card" true.
7. **RabbitMQ push condition**: fires only if `vc_id != ""` AND `APPROV_STATUS` is `"P"`
   or empty — i.e., **not** on a definitively-approved/rejected terminal-state, only on
   pending/unset states. Worth confirming this is intentional (vs. firing on every
   change).
   [`VisitingCardModel.go:505-532`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/VisitingCardModel.go)

---

## 5. RabbitMQ / Kafka / Redis

| Queue / `SERVICENAME` | Publisher | Consumer | Purpose |
|---|---|---|---|
| `VISITING_CARD_SERVICE` | `VisitingCardModel.go` (`updatePG`) — only when `APPROV_STATUS` is pending/empty | [`USER_VISITINGCARD_APPROVAL.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VISITINGCARD_APPROVAL.go) (`dbActionUserVisitingCardApproval`) — writes `approvalPg` | Syncs the visiting-card event into a separate approval-tracking database |

**Koi Kafka ya Redis usage nahi mila.**

---

## 6. End-to-End Technical Flow

```
Field-team/Supplier (capture) / GLADMIN (comments) / WebERP (enrich/design/approve)
    │
    ▼
[API — write]  POST serviceName=VISITING_CARD_SERVICE
                {FK_GLUSR_USR_ID, VISITING_CARD_ID?, TYPE, UPDATE_FLAG?,
                 FRONT_ATTACHMENT?, BACK_ATTACHMENT?, ...}
    │  VisitingCardController.go — Gateway (GLADMIN/MY/Weberp/M.INDIAMART.COM/MAPI/Merp)
    ▼
UpsertVisitingCard()  — out_iu decision (UPDATE / NO_VC REQUEST / INSERT)
    ▼
updatePG()
    ├─ TYPE=NO_VC              → upsertNoVisitCard() [ON CONFLICT upsert]
    ├─ VISITING_CARD_ID given:
    │     ├─ TYPE=GLADMIN      → dynamic UPDATE (admin-comment)
    │     ├─ TYPE=WEBERP       → ENRICH / APPROVE / DESIGN / ENRICHDESIGN branch
    │     └─ else (capture)    → MODID=="0" front/back-attachment UPDATE
    │                             (resets review-flags)
    │                          → else design-update UPDATE (resets review-flags)
    │     then → conditional upsertNoVisitCard("Y"/"N" based on APPROV_STATUS)
    └─ VISITING_CARD_ID absent → INSERT INTO STS_COMP_VISIT_CARD RETURNING ID
                                → upsertNoVisitCard("Y")
    ▼
[DB — meshpg]  STS_COMP_VISIT_CARD / STS_COMP_NO_VISIT_CARD
    │  if APPROV_STATUS pending/empty:
    ▼
[RabbitMQ]  SERVICENAME=VISITING_CARD_SERVICE
    ▼
[CONSUME]  dbActionUserVisitingCardApproval → writes approvalPg

Buyer / any caller
    │
    ▼
[API — read]  serviceName=DISPLAY_VISITING_CARD  {token, glusrid, modid}
    │  ActionVcardDisplay → UserVcardDisplayModel.go
    ▼
Response — visiting-card display data
```

---

## 7. Optimization Scope — DB Response-Time Contribution

### Low-Medium impact

1. **Successful update always triggers a second sequential query** (the
   `STS_COMP_NO_VISIT_CARD` upsert) — inherent to keeping the two tables in sync, not
   easily avoidable without a trigger/combined-statement, but worth being aware of as a
   consistent 2-round-trip cost on every successful visiting-card write.
2. **`updatePG`'s branching produces many hand-written near-duplicate UPDATE
   statements** (front/back/both variants repeated for both the field-capture and
   design-update paths) — not a runtime-performance issue, but a maintenance-cost:
   6 near-identical query-blocks (q8-q10, q16-q18) could likely be consolidated into a
   single dynamically-built statement (similar to the GLADMIN branch's
   whitelist-driven approach), reducing duplication risk on future schema-changes.

### Low impact

3. **RabbitMQ push condition (pending/empty-status only)** — appropriately limits
   downstream sync-traffic to non-terminal-state changes.

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora Visiting Card flowchart yahan dekho](https://lucid.app/lucidchart/e04f9bc0-0716-4381-975c-b23d91e0c504/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **RabbitMQ doesn't fire on definitively approved/rejected updates** — if a
   downstream-system needs to know about final-approval/rejection specifically, it
   won't be notified via this queue; confirm whether that's intentional (final states
   are non-time-sensitive) or a gap.
2. **A fresh photo-upload silently resets all review-progress** (`ENRICHED_FLAG`/
   `APPROV_STATUS`/`DESIGN_FLAG` → `NULL`) — anyone re-submitting a photo effectively
   restarts the WebERP-workflow from scratch; if this isn't communicated to internal
   reviewers, it could cause confusion about "lost" approvals.
3. **Highly duplicated near-identical UPDATE-statement variants** (§7, point 2) — a
   future schema-change to `STS_COMP_VISIT_CARD` risks being applied inconsistently
   across the 6+ hand-written query-blocks.
4. **`TYPE=GLADMIN`'s `keydate` logic overwrites literal input-values with
   `CURRENT_TIMESTAMP`** for certain keys (`UPDATE_DATE` and any `*_BY` field present) —
   a caller sending an explicit historical date for these fields would have it silently
   ignored in favor of "now".

---

## 10. Open Questions

1. Why does the RabbitMQ push exclude definitively approved/rejected states — is this
   an intentional scope-limitation?
2. `FLAG_DOCS` semantics (`2` hardcoded on write) — what do other `FLAG_DOCS` values
   mean, and who sets them?
3. `COUNT` param (used in the `APPROV_STATUS=='R'` branch's `cnt==0` check) — where does
   the caller source this value from, and what does it represent?
4. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`Visiting_Card_Business_Doc.md`](./Visiting_Card_Business_Doc.md) — product perspective
- [`../TrustSeal KT/TrustSeal_Technical_Doc.md`](../TrustSeal%20KT/TrustSeal_Technical_Doc.md) —
  similar multi-stage internal-approval-workflow precedent

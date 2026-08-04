# Visiting Card — Technical Doc (Code-Level Deep Dive)

Yeh doc Visiting-Card feature ka **technical implementation** cover karta hai — APIs, DB
tables, queries, RabbitMQ, approval-consumer, sab kuch code se verify karke. Business/product
perspective ke liye [`Visiting_Card_Business_Doc.md`](./Visiting_Card_Business_Doc.md) dekho —
dono docs same flows cover karte hain, bas alag audience ke liye.

**Repos**: `service-api-go-production` (write), `users-api-go-production` (read),
`user-temp-consumers-production` (approval-consumer, writes a separate `approvalPg` database).

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file:line diya gaya
hai har jagah). Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — team se confirm karo]** likha hai, guess nahi kiya.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Write (submit/update/comment/enrich/design/approve/no-vc) | write | [`VisitingCardController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/VisitingCardController.go) |
| Write — core upsert logic | write | [`VisitingCardModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/VisitingCardModel.go) (`UpsertVisitingCard`, `updatePG`, `upsertNoVisitCard`) |
| Mandatory-field validation | write | [`UserUtilsMandatory.go:237-279`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go) (`MandatoryParamsCheckVisitingCard`) |
| Type/length validation map | write | [`UsersValidationMaps.go:741-779`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) (`UserVisitingCardMap`) |
| Shared field-formatting helper | write | [`PnsTxnModel.go:407-413`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/PnsTxnModel.go) (`FormatValues` — nil-vs-present coercion, reused across many domains, not VC-specific) |
| Approval-consumer (writes `approvalPg`) | consumers | [`USER_VISITINGCARD_APPROVAL.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VISITINGCARD_APPROVAL.go) (`dbActionUserVisitingCardApproval`, `DeleteInsertInVCApproval`, `InsertInAuditTable`, `GetIspaidCusttypeId`) |
| SERVICENAME→queue mapping | write | [`rabbitmq.go:43`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go) — `"VISITING_CARD_SERVICE": "USER_VISITINGCARD_APPROVAL"` |
| Consumer registration | consumers | [`router.go:57`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Router/router.go) — `"USER_VISITINGCARD_APPROVAL": workers.UserVisitingCardApproval` |
| Read (display) — controller | read | [`VcardDisplayControllers.go`](../../users-api-go-production/internal/controllers/UsersControllers/VcardDisplayControllers.go) (`ActionVcardDisplay`) |
| Read (display) — model | read | [`UserVcardDisplayModel.go`](../../users-api-go-production/internal/models/users/UserVcardDisplayModel.go) (`GetVcardDisplayModel`, `getVCData`) |

---

## 2. Routes (confirmed from router files)

| Method | Path | Repo | Controller | serviceName |
|---|---|---|---|---|
| POST | `/visiting/card` | write | `VisitingCardController` | `VISITING_CARD_SERVICE` [`VisitingCardController.go:17`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/VisitingCardController.go) |
| GET/POST | `/wservce/vcard/display/*params` | read | `ActionVcardDisplay` | `DISPLAY_VISITING_CARD` [`routerUsers.go:408-412`](../../users-api-go-production/internal/api/users_router/routerUsers.go) |

Write route confirmed twice in [`router.go:172,340`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go)
(registered in two router groups, same controller). Read route also appears repeated across
multiple router groups (`routerUsers.go:411-412, 548-549, 569-570, 749-750`) — same controller,
different auth/middleware groupings; **exact difference between these groups not traced in this
review** — flagged in Open Questions.

---

## 3. Data Model — Tables

> **Verification note**: yeh sab table/column names Go code ke andar embedded SQL strings se
> liye gaye hain (har row ke saamne file:line diya hai). Live DB schema se pgAdmin pe
> cross-verify **nahi** kiya gaya hai.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `STS_COMP_VISIT_CARD` | meshpg (write) | The visiting-card record itself | `VISITING_CARD_ID` (PK, `RETURNING` on insert — [`VisitingCardModel.go:466`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/VisitingCardModel.go)), `FK_GLUSR_USR_ID`, `FRONT_ATTACHMENT`/`BACK_ATTACHMENT`, `FRONT_ENTERED_DATE`/`BACK_ENTERED_DATE`, `FRONT_ENTERED_BY`/`BACK_ENTERED_BY`, `FRONT_DESIGN_UPDATE`/`BACK_DESIGN_UPDATED`, `ENRICHED_FLAG`/`ENRICHED_BY`/`ENRICHED_DATE`, `APPROV_STATUS`, `REJECT_REASON`, `DESIGN_FLAG`/`DESIGN_APPROVED_BY`/`DESIGN_APPROVED_DATE`, `FLAG_DOCS` (hardcoded `2` on every write), `STS_COMP_VC_LATITUDE`/`_LONGITUDE`, `VISITING_CARD_ADMIN_COMMENT`, `STS_COMP_VC_UPDATEDBY`/`_UPDATESCREEN`/`_IP`/`_IP_COUNTRY`/`_UPDATEDUSING`/`_UPDATEDBY_URL`/`_UPDATEDBY_ID`, `UPDATE_DATE`, `UPDATED_BY_USING`, `CONTENT_REENRICHED_*`, `DESIGN_REENRICHED_*` (present in read query, not written by any traced write-path — see Open Questions), `VC_PRODUCTS` |
| `STS_COMP_NO_VISIT_CARD` | meshpg | "Supplier has no visiting card" companion-status, one row per GLID (upsert on `FK_GLUSR_USR_ID`) | `FK_GLUSR_USR_ID` (unique — `ON CONFLICT` target — [`VisitingCardModel.go:68`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/VisitingCardModel.go)), `STS_COMP_NO_VC_APPROVAL_STATUS` (`Y`/`N`), `STS_COMP_NO_VC_UPDATEDBY`/`_UPDATEDUSING`/`_UPDATEDDATE`/`_ENTEREDDATE` |
| `iil_visit_card_audit` | approvalPg (separate DB, consumer-side) | Append-only audit log of every visiting-card-service event the approval-consumer processes | `fk_glusr_usr_id`, `fk_VISITING_CARD_ID`, `fk_custtype_id`, `iil_vc_modid`, `iil_vc_updatedby`, `iil_vc_date` (`CURRENT_TIMESTAMP`), `iil_vc_APPROV_STATUS` — [`USER_VISITINGCARD_APPROVAL.go:132-138`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VISITINGCARD_APPROVAL.go) |
| `IIL_visit_card_APPR_PENDING` | approvalPg | The **current pending-approval queue** for visiting cards — delete-then-reinsert per event, so it always reflects the latest state per (glusrid, vc_id) pair | `fk_glusr_usr_id`, `fk_visiting_card_id`, `is_paid`, `WIP_WORKORDER_CNT`, `FK_IIL_ENTERPRISE_TYPE_ID` — [`USER_VISITINGCARD_APPROVAL.go:141-176`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VISITINGCARD_APPROVAL.go) |
| `glusr_usr` | approvalPg (consumer connects to this DB and reads `glusr_usr` from it) | Lookup, `glusr_usr_custtype_id` for the GLID | [`USER_VISITINGCARD_APPROVAL.go:179-206`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VISITINGCARD_APPROVAL.go) (`GetCuttypeId`) — **note**: function exists but is *not called* by `dbActionUserVisitingCardApproval` (see §4, point 9) |
| `WORKORDER` | approvalPg | Count of work-orders for the GLID (`WIP_WORKORDER_CNT`) | `WO_GLUSR_USR_ID` — [`USER_VISITINGCARD_APPROVAL.go:209-230`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VISITINGCARD_APPROVAL.go) (`WorkorderCount`) |
| `IIL_ENTERPRISE_GLUSR` | approvalPg | Enterprise-type classification for the GLID | `FK_IIL_ENTERPRISE_TYPE_ID`, `GLUSR_USR_ID` — [`USER_VISITINGCARD_APPROVAL.go:233-254`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VISITINGCARD_APPROVAL.go) (`EnterpriseTypeId`) |
| `sts_comp_no_visit_card` (read path) | mesh_pg_user (read-API's own connection name for the mesh) | Read-side lookup of `no_vc_status` alongside the display response | `sts_comp_no_vc_approval_status` — [`UserVcardDisplayModel.go:37`](../../users-api-go-production/internal/models/users/UserVcardDisplayModel.go) |
| `sts_comp_visit_card` (read path) | mesh_pg_user | Latest visiting-card record per `approv_status` partition, returned to callers | See columns in [`UserVcardDisplayModel.go:99`](../../users-api-go-production/internal/models/users/UserVcardDisplayModel.go) query |

---

## 4. Business Rules & Validation (code se exhaustive list)

1. **Mandatory-field gate**: `FK_GLUSR_USR_ID`, `VALIDATION_KEY`, `UPDATED_BY`,
   `VC_UPDATEDUSING`, `UPDATE_SCREEN` sab non-empty hone chahiye, warna generic error
   returns. `UPDATED_BY_ID` (agar diya ho) numeric hona chahiye.
   [`UserUtilsMandatory.go:237-279`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
2. **Gateway allowlist**: sirf `GLADMIN`, `MY`, `Weberp`, `M.INDIAMART.COM`, `MAPI`, `Merp`
   callers allowed hain (`Gateway_v1`). [`VisitingCardController.go:50`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/VisitingCardController.go)
3. **`out_iu` (insert-vs-update-vs-no-vc) 3-way priority decision**: `VISITING_CARD_ID`
   present → `UPDATE`; else `TYPE=="NO_VC"` → `NO_VC REQUEST`; else `INSERT`.
   [`VisitingCardModel.go:25-31`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/VisitingCardModel.go)
4. **`TYPE=NO_VC` (irrespective of insert/update path) → `upsertNoVisitCard(..., status="N")`**
   — a direct `ON CONFLICT (FK_GLUSR_USR_ID) DO UPDATE` upsert into `STS_COMP_NO_VISIT_CARD`
   forcing status `'N'` (no visiting card). This branch does **not** touch
   `STS_COMP_VISIT_CARD` at all. [`VisitingCardModel.go:103-104`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/VisitingCardModel.go)
5. **Update-path (`VISITING_CARD_ID` given), branches on `TYPE`**:
   - **`TYPE=GLADMIN`** — dynamic whitelist-driven UPDATE built by iterating request params
     against `UserVisitingCardMap` (only keys present in the map get included in the SQL);
     mainly used for admin-comments (`OPENING_COMMENT`→`VISITING_CARD_ADMIN_COMMENT`).
     Several keys (`UPDATE_DATE` always, plus `DESIGN_REENRICHED_BY`/`CONTENT_REENRICHED_BY`/
     `DESIGN_APPROVED_BY`/`ENRICHED_BY` **if present in the request**) are force-set to
     `CURRENT_TIMESTAMP` in SQL rather than using the literal client-supplied value — i.e.
     these are treated as "touch this timestamp now" markers, not literal date-values.
     [`VisitingCardModel.go:109-168`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/VisitingCardModel.go)
   - **`TYPE=WEBERP`** — further dispatches on `UPDATE_FLAG`:
     - `ENRICH` — sets `ENRICHED_DATE`/`ENRICHED_BY`/`ENRICHED_FLAG` + admin-comment. If
       `ENRICH_DATE` supplied it's parsed (`20060102150405` layout) and used; else
       `CURRENT_TIMESTAMP`. [`VisitingCardModel.go:171-181`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/VisitingCardModel.go)
     - `APPROVE` — sets `APPROV_STATUS`, `UPDATED_BY`, `UPDATE_DATE`.
       [`VisitingCardModel.go:182-185`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/VisitingCardModel.go)
     - `DESIGN` — sets `DESIGN_APPROVED_DATE`(=now)/`DESIGN_APPROVED_BY`/`DESIGN_FLAG`.
       [`VisitingCardModel.go:186-189`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/VisitingCardModel.go)
     - `ENRICHDESIGN` — does BOTH design and enrich fields in **one** UPDATE statement,
       `ENRICHED_FLAG` hardcoded to `1` (not taken from request). Same `ENRICH_DATE`
       parse-or-now logic as plain `ENRICH`.
       [`VisitingCardModel.go:190-200`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/VisitingCardModel.go)
     - anything else → `"UPDATE_FLAG can not same as desired."` (rejected, no write).
       [`VisitingCardModel.go:201-202,225`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/VisitingCardModel.go)
   - **Neither GLADMIN nor WEBERP (field-capture path), branches on `MODID=="0"`**:
     - `MODID=="0"` (photo-capture) — updates `FRONT_ATTACHMENT`/`BACK_ATTACHMENT`
       (3 query-variants depending on which of front/back is present — both, front-only,
       back-only), and **explicitly resets `ENRICHED_FLAG`/`APPROV_STATUS`/`DESIGN_FLAG`/
       `DESIGN_APPROVED_BY`/`DESIGN_APPROVED_DATE` to `NULL`** — a fresh photo restarts the
       WebERP review-cycle from scratch. Also force-sets `FLAG_DOCS=2`, captures lat/long.
       If neither `FRONT_ATTACHMENT` nor `BACK_ATTACHMENT` present → rejected with
       `"FRONT_ATTACHMENT,BACK_ATTACHMENT both cannot be empty."`
       [`VisitingCardModel.go:228-312,392-393`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/VisitingCardModel.go)
     - else (design-update path, `MODID!="0"`) — updates `FRONT_DESIGN_UPDATE`/
       `BACK_DESIGN_UPDATED` (same 3-variant pattern), sets `DESIGN_FLAG=1` (hardcoded),
       same review-flag-reset pattern as above. If neither present → rejected with
       `"FRONT_DESIGN_UPDATE,BACK_DESIGN_UPDATED both cannot be empty."`
       [`VisitingCardModel.go:313-389,394-395`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/VisitingCardModel.go)
6. **After ANY successful update-path write, a mandatory second query conditionally
   upserts `STS_COMP_NO_VISIT_CARD`** (applies to all three of GLADMIN/WEBERP/field-capture
   branches — this block runs unconditionally after `output=="SUCCESS"` regardless of which
   branch executed): if `APPROV_STATUS=='R'` (rejected) **and** `COUNT==0` (a caller-supplied
   `COUNT` param, meaning not conclusively traced — see Open Questions) → set No-VC status to
   `'N'`; **in every other case** (including `APPROV_STATUS` empty, `P`, `Y`, or `R` with
   `COUNT!=0`) → set No-VC status to `'Y'`. Net effect: almost any successful
   visiting-card-update marks "has visiting card = Yes" in the companion table, and only a
   specific reject-with-zero-count combination flips it back to "No".
   [`VisitingCardModel.go:417-451`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/VisitingCardModel.go)
7. **Insert-path (`VISITING_CARD_ID` absent, `TYPE!="NO_VC"`)**: requires at least one of
   `FRONT_ATTACHMENT`/`BACK_ATTACHMENT`/`FRONT_DESIGN_UPDATE`/`BACK_DESIGN_UPDATED` to be
   non-null, else rejected with `"FRONT_ATTACHMENT,BACK_ATTACHMENT,FRONT_DESIGN_UPDATE,
   BACK_DESIGN_UPDATED(All) can not be empty."`; inserts a brand-new
   `STS_COMP_VISIT_CARD` row (`FLAG_DOCS=2` hardcoded, `RETURNING VISITING_CARD_ID`), then
   **always** follows with `upsertNoVisitCard(..., "Y")` — a fresh insert immediately marks
   "has visiting card = Yes" regardless of any other field.
   [`VisitingCardModel.go:453-502`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/VisitingCardModel.go)
8. **RabbitMQ push condition**: fires only if `vc_id != ""` **and** `APPROV_STATUS` is `"P"`
   (pending) or empty — i.e. it does **not** fire on a definitively-approved (`Y`) or
   definitively-rejected (`R`) terminal write. Worth confirming with the team whether this is
   intentional (approval-consumer only needs to know about pending states) or a gap.
   [`VisitingCardModel.go:505-532`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/VisitingCardModel.go)
9. **Approval-consumer computes `isPaid`/`custtype`/`workorderCount`/`enterpriseTypeId` via
   `GetIspaidCusttypeId`**, which internally: (a) `SELECT glusr_usr_custtype_id FROM
   glusr_usr` for the GLID, (b) looks up `isPaid` from a **local JSON file**
   (`custtypeJsonFile` config path) keyed by custtype-id, splitting a `##`-delimited string
   and checking if the 7th field (`arr[6]`, 0-indexed) equals `"-1"` → paid, (c)
   `SELECT count(1) FROM WORKORDER`, (d) `SELECT FK_IIL_ENTERPRISE_TYPE_ID FROM
   IIL_ENTERPRISE_GLUSR LIMIT 1`. **All four steps run sequentially, and if any one fails the
   whole message is marked `Failed_Flag=1`** (still attempts the subsequent DB writes with
   partial/zero-value data, then Nacks the message for requeue).
   [`USER_VISITINGCARD_APPROVAL.go:94-101,285-309`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VISITINGCARD_APPROVAL.go)
10. **`updated_by` resolution in the consumer has a 3-tier fallback**: `UPDATED_BY_ID` from
    message if present and non-empty → use directly; else if `UPDATED_BY` present and equals
    (case-insensitive) `"USER"` → hardcode to `"-1"`; else → `"0"`.
    [`USER_VISITINGCARD_APPROVAL.go:86-92`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VISITINGCARD_APPROVAL.go)
11. **`DeleteInsertInVCApproval` is a delete-then-insert inside a single transaction** (not an
    upsert) — first deletes any existing `(fk_glusr_usr_id, fk_visiting_card_id)` row from
    `IIL_visit_card_APPR_PENDING`, then inserts a fresh row with the newly-computed
    `is_paid`/`WIP_WORKORDER_CNT`/`FK_IIL_ENTERPRISE_TYPE_ID`. Effect: this table always
    reflects only the **latest** pending-approval snapshot per card, not history (history is
    `iil_visit_card_audit`'s job, via a separate always-insert call).
    [`USER_VISITINGCARD_APPROVAL.go:141-176`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VISITINGCARD_APPROVAL.go)
12. **Read-side "current card" selection logic**: `getVCData` uses a windowed query —
    `row_number() OVER (PARTITION BY approv_status ORDER BY visiting_card_id DESC)` — to pick
    the most-recent row **per distinct `approv_status` value** for the GLID, then joins back
    to return full rows for all of those latest-per-status IDs (not just a single "the"
    current card). So a supplier can have multiple visiting-card rows returned simultaneously
    if they have cards sitting in different approval-states.
    [`UserVcardDisplayModel.go:99`](../../users-api-go-production/internal/models/users/UserVcardDisplayModel.go)
13. **Read-side `approv_status` display default**: if `NULL`/empty, displayed as `"P"`
    (pending) via `ternary_operator`. [`UserVcardDisplayModel.go:120`](../../users-api-go-production/internal/models/users/UserVcardDisplayModel.go)
14. **Attachment URLs are constructed, not stored as full URLs**: `front_attachment`/
    `back_attachment`/`front_design_update`/`back_design_updated` values (filenames) are
    prefixed at read-time into `https://imdocs.indiamart.com/Sts_Attachment/<filename>`.
    [`UserVcardDisplayModel.go:185-193`](../../users-api-go-production/internal/models/users/UserVcardDisplayModel.go)

---

## 5. RabbitMQ

Ek hi queue/SERVICENAME pair pura VC domain mein use hota hai:

| SERVICENAME (published) | Resolves to queue | Publisher | Consumer | Purpose |
|---|---|---|---|---|
| `VISITING_CARD_SERVICE` | `USER_VISITINGCARD_APPROVAL` [`rabbitmq.go:43`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go) | `VisitingCardModel.go` (`updatePG`) — only when `vc_id!=""` and `APPROV_STATUS` is pending/empty (§4.8) | [`USER_VISITINGCARD_APPROVAL.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VISITINGCARD_APPROVAL.go) (`UserVisitingCardApproval` → `dbActionUserVisitingCardApproval`), registered in [`router.go:57`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Router/router.go) | Syncs the visiting-card event into a separate `approvalPg` database — writes an audit-row and refreshes the pending-approval table |

**Message shape published** (`vc_array` in `updatePG`): `modid` (the gateway-validated caller
type, e.g. `GLADMIN`/`Weberp`), `UPDATED_BY`, `APPROV_STATUS`, `glusrid`, `vc_id`,
`SERVICENAME`, `TIMESTAMP`, and conditionally `UPDATED_BY_ID`.
[`VisitingCardModel.go:508-519`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/VisitingCardModel.go)

No other RabbitMQ queue was found in any VC-related file across all three repos.

---

## 6. Kafka

**Koi Kafka usage nahi mila** kisi bhi VC-related file mein, in teeno repos ke andar. (The
only Kafka-mentioning files found in a repo-wide grep are generic router/broker-init files
shared across all domains — not VC-specific.)

---

## 7. Redis

**Koi Redis usage nahi mila** kisi bhi VC-related file mein. Har read
(`GET /wservce/vcard/display/*params`) live Postgres hit karta hai — koi VC-specific caching
layer nahi hai (dekho §9, Optimization Scope point 4).

---

## 8. End-to-End Technical Flow

```
Field-team/Supplier (photo-capture) / GLADMIN (admin-comment) / WebERP (enrich/design/approve)
    │
    ▼
[API — write]  POST /visiting/card
                {FK_GLUSR_USR_ID, VISITING_CARD_ID?, TYPE?, UPDATE_FLAG?, MODID?,
                 FRONT_ATTACHMENT?, BACK_ATTACHMENT?, FRONT_DESIGN_UPDATE?, BACK_DESIGN_UPDATED?, ...}
    │  VisitingCardController.go
    │  1. Mandatory-field check (MandatoryParamsCheckVisitingCard)
    │  2. Gateway allowlist check (GLADMIN/MY/Weberp/M.INDIAMART.COM/MAPI/Merp)
    │  3. Type/length validation against UserVisitingCardMap (LengthAndTypeValidations_v3)
    ▼
UpsertVisitingCard()  — out_iu decision: UPDATE / NO_VC REQUEST / INSERT
    ▼
updatePG()
    ├─ TYPE=NO_VC                  → upsertNoVisitCard(status="N")  [STS_COMP_NO_VISIT_CARD only]
    ├─ VISITING_CARD_ID given:
    │     ├─ TYPE=GLADMIN          → dynamic whitelist UPDATE (admin-comment, forced-now timestamps)
    │     ├─ TYPE=WEBERP           → ENRICH / APPROVE / DESIGN / ENRICHDESIGN branch
    │     └─ else (field-capture)  → MODID=="0": front/back-attachment UPDATE (resets review-flags)
    │                                else: design-update UPDATE (resets review-flags, DESIGN_FLAG=1)
    │     then (if SUCCESS)        → conditional upsertNoVisitCard("Y" unless APPROV_STATUS=='R' && COUNT==0)
    └─ VISITING_CARD_ID absent     → INSERT INTO STS_COMP_VISIT_CARD RETURNING ID
                                    → always upsertNoVisitCard("Y")
    ▼
[DB — meshpg]  STS_COMP_VISIT_CARD / STS_COMP_NO_VISIT_CARD
    │  if vc_id != "" AND APPROV_STATUS is "P" or empty (not a terminal Y/R state):
    ▼
[RabbitMQ publish]  SERVICENAME=VISITING_CARD_SERVICE → queue USER_VISITINGCARD_APPROVAL
    ▼
[CONSUME]  dbActionUserVisitingCardApproval (approvalPg)
    │  1. GetIspaidCusttypeId() — 4 sequential SELECTs: custtype (glusr_usr),
    │     isPaid (local JSON-file lookup), workorderCount (WORKORDER),
    │     enterpriseTypeId (IIL_ENTERPRISE_GLUSR)
    │  2. DeleteInsertInVCApproval() — transaction: DELETE + INSERT into
    │     IIL_visit_card_APPR_PENDING (latest-snapshot table)
    │  3. InsertInAuditTable() — INSERT into iil_visit_card_audit (append-only history)
    │  any step failing → Failed_Flag=1 → message Nack'd (requeued)
    │  all steps ok     → message Ack'd

Buyer / any caller
    │
    ▼
[API — read]  GET/POST /wservce/vcard/display/*params  {token, glusrid, modid}
    │  ActionVcardDisplay → GetVcardDisplayModel
    │  Query 1: STS_COMP_NO_VISIT_CARD.sts_comp_no_vc_approval_status for glusrid
    │  Query 2 (getVCData): windowed query — latest STS_COMP_VISIT_CARD row PER approv_status
    │           value for glusrid, attachment filenames rewritten into imdocs.indiamart.com URLs
    ▼
Response — visiting-card display data (one entry per distinct approval-state the supplier has)
```

---

## 9. Flow-wise DB & Table Usage — Kaun sa DB, Kaun sa Table, Kis Liye

### Flow A — Field-capture / GLADMIN-comment / WEBERP-update (write, meshpg only)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `STS_COMP_VISIT_CARD` | UPDATE (one of 8+ hand-written variants depending on TYPE/UPDATE_FLAG/MODID/front-back-presence) | The core write — front/back attachment, design-update, enrich, approve, or admin-comment fields depending on branch (§4.5) |
| 2 | meshpg | `STS_COMP_NO_VISIT_CARD` | UPDATE (conditional, runs after any successful #1) | Keeps the companion "has visiting card" flag in sync (§4.6) |
| 3 | meshpg (approvalPg-bound, async via queue) | — | — | See Flow C below — happens out-of-band via RabbitMQ, not in the same HTTP request |

### Flow B — New visiting-card INSERT (no `VISITING_CARD_ID` given)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `STS_COMP_VISIT_CARD` | INSERT ... RETURNING VISITING_CARD_ID | Creates the new record; front/back attachment or design-update fields, `FLAG_DOCS=2` |
| 2 | meshpg | `STS_COMP_NO_VISIT_CARD` | UPSERT (`ON CONFLICT DO UPDATE`, status forced `'Y'`) | A fresh insert always means "has visiting card = yes" |

### Flow C — NO_VC request

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `STS_COMP_NO_VISIT_CARD` | UPSERT (`ON CONFLICT DO UPDATE`, status forced `'N'`) | Only table touched — no `STS_COMP_VISIT_CARD` interaction at all in this branch |

### Flow D — Approval-consumer (async, `approvalPg`) — the most DB-heavy flow in this domain

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | approvalPg | `glusr_usr` | SELECT `glusr_usr_custtype_id` | Resolve customer-type for the GLID |
| 2 | *(local file, not DB)* | `custtypeJsonFile` (JSON on disk) | Read + parse | Determine `isPaid` from the custtype (`arr[6]=="-1"`) — a local static lookup, not a DB round-trip |
| 3 | approvalPg | `WORKORDER` | SELECT `count(1)` | Work-order count for the GLID (`WIP_WORKORDER_CNT`) |
| 4 | approvalPg | `IIL_ENTERPRISE_GLUSR` | SELECT `FK_IIL_ENTERPRISE_TYPE_ID` (LIMIT 1) | Enterprise-type classification for the GLID |
| 5 | approvalPg | `IIL_visit_card_APPR_PENDING` | DELETE (by glusrid+vc_id) | Remove any stale pending-approval snapshot before reinsert |
| 6 | approvalPg | `IIL_visit_card_APPR_PENDING` | INSERT | Insert fresh pending-approval snapshot with the just-computed isPaid/workorder/enterprise data |
| 7 | approvalPg | `iil_visit_card_audit` | INSERT | Append-only audit-trail row for this event |

**Total DB round-trips for one approval-consumer message: 7** (plus 1 local-file read), all
against a **single physical database** (`approvalPg`) — unlike GST's multi-database fan-out,
this consumer is single-DB but query-heavy per message, and steps 1-4 run strictly
sequentially even though 1/3/4 are independent of each other (see §10, Optimization point 2).

### Flow E — Read (display)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | mesh_pg_user (read-API's connection) | `sts_comp_no_visit_card` | SELECT `sts_comp_no_vc_approval_status` | The companion no-vc-status flag returned alongside card data |
| 2 | mesh_pg_user | `sts_comp_visit_card` | SELECT (windowed, `row_number() OVER (PARTITION BY approv_status ...)`) | Latest card per distinct approval-status for the GLID |

---

## 10. Optimization Scope — DB Response-Time Contribution

### High-impact

1. **Approval-consumer runs 4 sequential SELECT/file-read operations before any write**
   (`GetIspaidCusttypeId` — custtype, JSON-file, workorder-count, enterprise-type; §9 Flow D
   #1-4), none of which depend on each other's *output* value being fed into the next call
   (each takes only `glusrid`/`custtypeid` as pre-known input). **Concrete fix**: parallelize
   these via goroutines (same pattern GST's `USER_GST_LAST_MODIFIED` already uses
   successfully) — could meaningfully cut consumer per-message latency, especially useful if
   `approvalPg` or the JSON-file read is ever slow.
   [`USER_VISITINGCARD_APPROVAL.go:285-309`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VISITINGCARD_APPROVAL.go)
2. **`GetIspaidCusttypeId`'s failure handling still proceeds to write with partial/zero-value
   data.** If e.g. `WorkorderCount` fails, `Failed_Flag=1` is set but `DeleteInsertInVCApproval`
   and `InsertInAuditTable` still execute with whatever zero-value the failed field defaulted
   to (`workorderCount="0"` etc.), and only the message-Nack happens afterward — meaning a
   **stale/incorrect row can be committed to `approvalPg`** before the retry (on redelivery)
   overwrites it. Worth confirming with the team whether this is acceptable (eventual
   consistency via requeue) or should short-circuit before any write on partial failure.
   [`USER_VISITINGCARD_APPROVAL.go:94-116`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_VISITINGCARD_APPROVAL.go)

### Medium-impact

3. **`updatePG`'s branching produces many hand-written near-duplicate UPDATE statements**
   (6 variants across front/back/both for both the field-capture and design-update paths:
   `q8`-`q10`, `q16`-`q18`) — not a runtime-performance issue per se, but a maintenance-cost:
   these could be consolidated into a single dynamically-built statement, similar to the
   GLADMIN branch's whitelist-driven approach (`UserVisitingCardMap`-based loop). Reduces the
   risk of a future schema-change being applied inconsistently across only some variants.
   [`VisitingCardModel.go:228-390`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/VisitingCardModel.go)
4. **No caching anywhere on the read path** (§7) — `GET /wservce/vcard/display/*params` hits
   Postgres on every call (2 sequential queries). Visiting-card data changes relatively
   infrequently per-supplier (bounded by how often WebERP/field-team touch a given card) —
   a short-TTL cache-aside pattern could reduce read latency, though the multi-approval-state
   windowed query (§4.12) would need careful cache-key/invalidation design since results can
   span multiple rows per GLID.
5. **Every successful write does a mandatory second sequential query** (the
   `STS_COMP_NO_VISIT_CARD` upsert, §4.6) — inherent to keeping the two tables in sync, not
   easily avoidable without a DB trigger or combined statement, but worth being aware of as a
   consistent 2-round-trip-minimum cost on every successful write.

### Low impact / good practice already present

6. **RabbitMQ push condition (pending/empty-status only, §4.8)** appropriately limits
   downstream sync-traffic to non-terminal-state changes — the approval-consumer isn't
   invoked redundantly on every single field-capture/admin-comment write that doesn't change
   approval status.

---

## 11. Cron Inventory

**Koi visiting-card-specific cron nahi mila** — repo-wide grep for "visit" + "cron" patterns
returned no matches in any of the three repos. Unlike GST (which has a daily
BigQuery-driven re-verification cron), the visiting-card review/approval workflow appears to
be entirely event-driven (RabbitMQ, triggered only by explicit API writes) with no scheduled
background job.

---

## 12. Edge Cases & Gotchas (technical POV)

1. **A fresh photo-upload silently resets all review-progress** (`ENRICHED_FLAG`/
   `APPROV_STATUS`/`DESIGN_FLAG`/`DESIGN_APPROVED_BY`/`DESIGN_APPROVED_DATE` → `NULL`) —
   anyone re-submitting a photo effectively restarts the WebERP-workflow from scratch. If not
   communicated clearly to internal reviewers, could cause confusion about "lost" approvals.
   [`VisitingCardModel.go:243-247,269-273,295-299`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/VisitingCardModel.go)
2. **RabbitMQ does not fire on definitively-approved/rejected updates** (§4.8) — if any
   downstream consumer besides `USER_VISITINGCARD_APPROVAL` ever needs to know about final
   approval/rejection specifically, it will not be notified via this queue as currently wired.
3. **`GetCuttypeId`/`GetIspaidCusttypeId` reads `glusr_usr` from the `approvalPg` connection**,
   not from meshpg/mainPg — implies `glusr_usr` (or a replica of it) is also present inside
   `approvalPg`. Worth confirming this is an intentional replicated table and not a
   stale/divergent copy, since the write-side of VC never touches `glusr_usr` directly.
4. **`DESIGN_REENRICHED_*`/`CONTENT_REENRICHED_*` columns are read** (in the display query,
   `UserVcardDisplayModel.go:99`) **and validated** (in `UserVisitingCardMap`), and even
   participate in the GLADMIN branch's "force to CURRENT_TIMESTAMP" `keydate` logic
   (`VisitingCardModel.go:123-128`) — but **no write-path in this review actually sets
   `DESIGN_REENRICHED_FLAG`/`CONTENT_REENRICHED_FLAG`/`DESIGN_REENRICHED_DATE`/
   `CONTENT_REENRICHED_DATE` values through any of the traced branches**, only the `*_BY`
   variants are conditionally force-timestamped if present in the request. Either there's an
   upstream/other caller path not covered by this review, or these are legacy columns.
   Flagged in Open Questions.
5. **The `COUNT` param controlling the reject→No-VC-status-flip logic (§4.6) is never set by
   any code in `updatePG` itself** — it's read via `utils.FormatToInt(params["COUNT"])`,
   meaning it must come from the caller's request payload. No documentation of what a caller
   is meant to pass here was found. Flagged in Open Questions.
6. **Approval-consumer proceeds to write even on partial upstream-lookup failure** (§10,
   point 2) — a subtle data-quality risk if requeue/retry semantics aren't airtight.
7. **Same controller (`VisitingCardController`) serves capture, admin-comment, and all three
   WebERP sub-workflows** through a single flat `TYPE`/`UPDATE_FLAG`/`MODID` parameter space —
   easy to send a slightly-wrong flag combination and land in an unintended branch (e.g.
   omitting `TYPE` entirely routes into the field-capture/design-update branch rather than
   erroring).

---

## 13. Open Questions

1. What does the `COUNT` param represent in the `APPROV_STATUS=='R'` branch's `cnt==0` check
   (§4.6, §12.5) — where is the caller meant to source this value from?
2. `DESIGN_REENRICHED_*`/`CONTENT_REENRICHED_*` columns (§12.4) — validated and read, but no
   write-path in this review sets their flag/date values. Legacy columns, or is there another
   caller/screen not covered here?
3. Why are there 4 near-duplicate registrations of the read route
   (`/wservce/vcard/display/*params` in `routerUsers.go` at lines 411-412, 548-549, 569-570,
   749-750) across different router groups — different auth/middleware contexts? Not traced
   in this review.
4. Is `glusr_usr` inside `approvalPg` (used by `GetCuttypeId`) a live-synced replica or could
   it drift from the source-of-truth `glusr_usr`? Confirm replication mechanism with the DBA/
   platform team.
5. Is the RabbitMQ push exclusion of terminal approve/reject states (§4.8) an intentional
   scope-limitation, or should the approval-consumer also be notified on those?
6. Is `GetIspaidCusttypeId`'s partial-failure-then-write behavior (§10 point 2) intentional
   (rely on requeue to eventually correct the row) or should it short-circuit before writing?
7. Live DB schema verification (column types, nullability, indexes, constraints, and
   confirming `approvalPg`'s `glusr_usr`/`WORKORDER`/`IIL_ENTERPRISE_GLUSR` are the same
   physical tables referenced elsewhere or local replicas) — this doc only reflects what the
   Go SQL strings imply.

---

## See also

- [`Visiting_Card_Business_Doc.md`](./Visiting_Card_Business_Doc.md) — same flows,
  product/business perspective, bina code ke
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — comparable
  multi-database, multi-stage verification-workflow precedent

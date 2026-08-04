# Rating Usefulness — Technical Doc (Code-Level Deep Dive)

**Scope note**: yeh doc sirf **Rating Usefulness** ("helpful?" vote) cover karta hai. Yeh
**Rating** (1-5 star) se alag hai — dekho
[`../Rating KT/Rating_Technical_Doc.md`](../Rating%20KT/Rating_Technical_Doc.md), jo iss pass
mein poori tarah re-read aur cross-verify kiya gaya hai (§4, §5, §6, §11 neeche dekho).

Business/product perspective ke liye
[`Rating_Usefulness_Business_Doc.md`](./Rating_Usefulness_Business_Doc.md) dekho.

**Repos**: `service-api-go-production` (write), `user-temp-consumers-production` (2
consumers, not 1 — dekho §1).

**Methodology**: har claim source code se trace kiya gaya hai, exact file:line citations ke
saath. Jahan code se pura confirm nahi ho paaya, **[INFERRED — confirm with team]** likha hai
ya Open Questions mein daala hai. Yeh pass pichle (shallow) doc se deeper hai — is baar
`USER_RATING_NOTIFICATION_PG.go` ka approvalPg-replication branch bhi trace kiya gaya hai (jo
pehle miss ho gaya tha), aur exact response-field names (`RATING_REVIEW_USEFULNESS` /
`RATING_REVIEW_ABUSE_CNT`) read-model se confirm kiye gaye hain.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Vote submit (write) | write | [`UserRatingUsefulnessController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserRatingUsefulnessController.go) (192 lines), [`UserRatingUsefulnessModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserRatingUsefulnessModel.go) (130 lines — func `RatingUsefulInsert`) |
| Vote validation map | write | [`UsersValidationMaps.go:456-463`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) — `RatingUsefulMap` |
| Allowed-caller list (gateway) | write | [`UserRatingUsefulnessController.go:15`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserRatingUsefulnessController.go) — `RatingUsefulHash` (30-entry list, defined inline in this file, not shared with Rating-submit's own allowlist) |
| Count sync-back consumer (loopback to write-API) | consumer | [`USER_RATING_USEFULNESS.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_USEFULNESS.go) (84 lines) — func `dbActionUserRatingUsefulness` / `callRatingWrite` |
| Rating record whose counts get updated — **reused controller** | write | [`UserSupplierRatingController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSupplierRatingController.go) / [`UserSupplierRatingModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSupplierRatingModel.go) (`UpdateRating`, `CALLEDFROM=="RATING_USEFULNESS_WRITE"` branch) — dekho §8 |
| **Second, previously-undocumented downstream consumer** — replicates the count-update (and, via a separate/likely-dead code path, a vote record) into `approvalPg` | consumer | [`USER_RATING_NOTIFICATION_PG.go:148-199`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_NOTIFICATION_PG.go) — dekho §5 point 4, §8 |
| Read-side | — | **Koi dedicated read controller nahi hai.** Helpful/abuse counts `GET /supplierrating` aur `GET /supplierrating_versions` ke response ke andar hi aate hain, confirmed is pass mein exact field names ke saath — dekho §3, §9 |

---

## 2. Routes

| Method | Path | Repo | Controller |
|---|---|---|---|
| POST | `/rating_usefulness` | write | [`router.go:161,329`](../../service-api-go-production/service-api-go-production/internal/api/users_router/router.go) → `UserControllers.UserRatingUsefulnessController` (registered twice, same pattern as `/supplierrating` in the Rating tech doc — both registrations point to the same controller) |

---

## 3. Data Model — Tables

> **Verification note**: table/column names Go embedded SQL strings se liye gaye hain, live
> DB schema se cross-verify **nahi** kiya gaya hai.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_RATING_USEFULNESS` | meshpg | **Primary vote record** — ek row per (buyer, rating) pair | `FK_GLUSR_RATING_ID`, `FK_GLUSR_USR_ID`, `IS_RATING_USEFUL` (`1`=helpful, `-1`=abuse), `FK_IIL_SCREEN_ID` (source_id), `ENTRY_DATE` — [`UserRatingUsefulnessModel.go:38`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserRatingUsefulnessModel.go) |
| `GLUSR_RATING` | meshpg **+ approvalPg (replica, confirmed this pass)** | Denormalized counts **also** live on the parent rating row — columns confirmed via the validation-map literal, **not** `HELPFUL_COUNT`/`ABUSE_COUNT` as a shallower read of this domain might suggest | `RATING_USEFULNESS_COUNT`, `RATING_ABUSE_COUNT` — [`UsersValidationMaps.go:509-510,533-534`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) (two separate map literals, `MapSupplierRatingQuery` used for the actual `UPDATE ... SET` and a second `LengthAndTypeValidations` map, both agree on the same physical column names). Full `GLUSR_RATING` schema: [`../Rating KT/Rating_Technical_Doc.md`](../Rating%20KT/Rating_Technical_Doc.md) §3 |

**Correction vs. prior shallow doc**: the physical column names are `RATING_USEFULNESS_COUNT`
/ `RATING_ABUSE_COUNT`, not `HELPFUL_COUNT`/`ABUSE_COUNT`. The API-level *input* field names
(`HELPFUL_COUNT`/`ABUSE_COUNT`) map onto those columns — see the map literal above. The
*output* field names on read are different again (`RATING_REVIEW_USEFULNESS` /
`RATING_REVIEW_ABUSE_CNT`, see §9) — three different names for the same underlying data at
three different layers (input API field → DB column → output API field). Worth knowing when
grepping for this data across the codebase.

---

## 4. Business Rules & Validation (code se)

1. **`IS_HELPFUL` sirf `"1"` ya `"-1"` accept hota hai** — koi aur value:
   `"IS_HELPFUL can be either 1 or -1"`.
   [`UserRatingUsefulnessController.go:92-104`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserRatingUsefulnessController.go)
2. **Mandatory fields**: `GLUSR_ID` (buyer), `RATING_ID`, `SOURCE_ID`, `VALIDATION_KEY`,
   `IS_HELPFUL`, `IP`, `IP_COUNTRY` — missing any one gives a single combined error message.
   [`UserRatingUsefulnessController.go:90-91`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserRatingUsefulnessController.go)
3. **Gateway validation uses its own, separate allowed-caller list** (`RatingUsefulHash`, 30
   entries — `MY`, `GLADMIN`, `TOLLFREE`, `MAPI`, `SELLERMY`, etc.) — this is **defined
   locally in `UserRatingUsefulnessController.go`**, not shared with `UserSupplierRatingController`'s
   own allowlist, even though the two features are functionally linked (§8). Any caller
   added to one list is not automatically added to the other.
   [`UserRatingUsefulnessController.go:15`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserRatingUsefulnessController.go)
4. **Field-length/type validation** via `LengthAndTypeValidations_v3(inputParams, UserModels.RatingUsefulMap)`
   — `GLUSR_ID`/`RATING_ID` numeric ≤10 digits, `SOURCE_ID` numeric ≤5 digits, `IS_HELPFUL`
   numeric ≤5 digits (despite only ever being `1`/`-1`), `IP`/`IP_COUNTRY` varchar ≤40.
   [`UsersValidationMaps.go:456-463`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go)
5. **Duplicate-vote handling is a genuinely two-step, non-obvious code path — fully re-traced
   this pass**: `RatingUsefulInsert` always runs the `INSERT`, and regardless of whether it
   succeeds or fails, **unconditionally runs a second `SELECT COUNT(...)` query afterwards
   only inside the success branch of the Go `if/else`**. The actual duplicate-suppression
   happens via a **string-match on the error text** (`strings.Contains(output, "execution
   failed")`) at the very end of the function, converting *any* insert failure — not
   specifically a duplicate-key violation — into the message `"Record already exists"` with
   `flag=1` (which the controller reports back as **HTTP status SUCCESS/200**, not a failure).
   This means: a genuine transient DB error on the INSERT (e.g. a connection blip) will be
   silently reported to the caller as "you already voted," not as an actual error — no
   specific Postgres error-code/constraint-name check exists.
   [`UserRatingUsefulnessModel.go:50-58,91-93,122-125`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserRatingUsefulnessModel.go)
6. **Count-sync (RabbitMQ push) only happens when the INSERT was a genuine new row** — the
   `output == "INSERT SUCCESS"` branch (as opposed to the reclassified `"Record already
   exists"` branch) is the only path that computes `HELPFUL_COUNT`/`ABUSE_COUNT` and pushes to
   `USER_RATING_USEFULNESS`. Duplicate votes never trigger a downstream write anywhere.
   [`UserRatingUsefulnessModel.go:94-119`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserRatingUsefulnessModel.go)
7. **No self-vote block found** — nothing in `UserRatingUsefulnessController.go` or
   `RatingUsefulInsert` compares `GLUSR_ID` (voting buyer) against the rating's own
   `FK_GLUSR_BUYER_ID`. Confirmed absent in this pass too, not just inferred — see Open
   Questions.
8. **`VALIDATION_KEY` and `unique_id` are stripped from `inputParams` before the JSON-shape
   check** (`InputJsonCheck`) — same defensive pattern used elsewhere in this codebase to keep
   secrets/logging-IDs out of a generic "extra unexpected fields" check.
   [`UserRatingUsefulnessController.go:85-88`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserRatingUsefulnessController.go)
9. **The write-back into `UserSupplierRatingController` has its own, narrower mandatory-field
   set** confirmed cross-referenced against [`../Rating KT/Rating_Technical_Doc.md`](../Rating%20KT/Rating_Technical_Doc.md)
   §5 point 8: `CALLEDFROM=="RATING_USEFULNESS_WRITE"` + `UPDATE_FLAG=="U"` requires
   `RATING_ID/VALIDATION_KEY/UPDATEDBY/HELPFUL_COUNT/ABUSE_COUNT/IP/IP_COUNTRY` — **not**
   `GLUSR_ID`/`SOURCE_ID` (those never leave the vote-insert step; the write-back is
   per-rating, not per-voter).
10. **The loopback call's `VALIDATION_KEY` is a hardcoded literal**
    (`e27d039e38ae7b3d439e8d1fe870fc68`), not read from a secrets store or config file, baked
    directly into the consumer.
    [`USER_RATING_USEFULNESS.go:79`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_USEFULNESS.go)

---

## 5. RabbitMQ — Queues Used

Fully traced this pass against `service-api-go-production`'s `serviceToQueueMap`/`exchangeSet`
([`rabbitmq.go:20-91`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go)):

| `SERVICENAME` | Routing | Publisher | Consumer(s) | Purpose |
|---|---|---|---|---|
| `USER_RATING_USEFULNESS` | **Direct queue** (`serviceToQueueMap["USER_RATING_USEFULNESS"] = "USER_RATING_USEFULNESS"`, **not** present in `exchangeSet` → no topic-exchange fan-out, single dedicated consumer) — [`rabbitmq.go:29`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go) | `UserRatingUsefulnessModel.go` (`RatingUsefulInsert`, only on a genuine new-vote insert) | [`USER_RATING_USEFULNESS.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_USEFULNESS.go) (registered in [`router.go:48`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Router/router.go) and [`IntializeMsgBroker.go:209`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/IntializeMsgBroker.go)) | Trigger the count write-back into `GLUSR_RATING` |
| `SUPPLIER_RATING_NEW` (2nd hop, indirect — pushed by `UpdateRating` itself, **not** by this feature's own model code) | routing key `user.ratingnotification.*`, exchange `USER.topic` — [`rabbitmq.go:68`](../../service-api-go-production/service-api-go-production/pkg/utils/rabbitmq.go), confirmed in [`../Rating KT/Rating_Technical_Doc.md`](../Rating%20KT/Rating_Technical_Doc.md) §6 | `UpdateRating`'s `CALLEDFROM=="RATING_USEFULNESS_WRITE"` branch, keyed by `RATING_ID` not `SUPPLIER_ID` | `USER_RATING_NOTIFICATION` (no-op for this message shape, §8) **and** `USER_RATING_NOTIFICATION_PG` (approvalPg replication, §8 — this is the previously-undocumented second consumer) | Fan the helpful/abuse count-update out to the two consumers already bound on `SUPPLIER_RATING_NEW` for the Rating domain generally — Rating Usefulness rides on this existing pipe rather than having its own |

**Correction vs. prior shallow doc**: it's not "just one queue in this feature." The primary
vote → count-recompute → write-back hop is a single dedicated queue, **but** that write-back
itself (being a call into the shared `UpdateRating`) triggers a **second**, indirect RabbitMQ
hop (`SUPPLIER_RATING_NEW`) that this feature doesn't own but silently rides on — landing in
`USER_RATING_NOTIFICATION_PG`, which replicates the new counts into `approvalPg` too (§8).

---

## 6. Kafka

**Koi Kafka usage nahi mila** in either file in this feature (`UserRatingUsefulnessController.go`,
`UserRatingUsefulnessModel.go`, `USER_RATING_USEFULNESS.go` — all grepped, zero hits).
`USER_RATING_USEFULNESS.go` uses `InitializeRabbitMq`, not Kafka. (Contrast with the Rating
domain proper, where `USER_RATING_SELLER_RISK.go` does use Kafka — but that consumer is
triggered off fresh-rating-insert events, not usefulness votes, per [`../Rating KT/Rating_Technical_Doc.md`](../Rating%20KT/Rating_Technical_Doc.md) §7.)

---

## 7. Redis

**Koi Redis usage nahi mila** in this feature's files (grepped, zero hits). Helpful/abuse
counts come back live from `GLUSR_RATING` on every `GET /supplierrating`/`/supplierrating_versions`
call — no caching layer, consistent with the Rating domain's read side generally
([`../Rating KT/Rating_Technical_Doc.md`](../Rating%20KT/Rating_Technical_Doc.md) §8).

---

## 8. End-to-End Technical Flow

### Flow A — Buyer votes helpful/abuse (primary, live path)

```
Buyer (rating padh raha, "helpful" ya "abuse" click karta hai)
    │
    ▼
[API — write]  POST /rating_usefulness
    │  UserRatingUsefulnessController.go
    │  1. Mandatory fields: GLUSR_ID/RATING_ID/SOURCE_ID/VALIDATION_KEY/IS_HELPFUL/IP/IP_COUNTRY
    │  2. IS_HELPFUL must be "1" or "-1"
    │  3. Gateway validation (RatingUsefulHash allowlist — its own, separate from Rating-submit's)
    │  4. LengthAndTypeValidations_v3(RatingUsefulMap)
    ▼
[DB #1 — meshpg]  INSERT INTO GLUSR_RATING_USEFULNESS (rating_id, buyer_id, is_helpful, source_id, entry_date)
    │
    ├─ FAIL (any reason — not just duplicate-key, §4 point 5) → reclassified as
    │  "Record already exists", HTTP SUCCESS/200, stop here, no queue push
    │
    └─ SUCCESS →
          [DB #2 — meshpg]  SELECT COUNT(helpful), COUNT(abuse) FROM GLUSR_RATING_USEFULNESS
                             WHERE FK_GLUSR_RATING_ID = $1
          │
          ▼
          [RabbitMQ — direct queue publish]  SERVICENAME=USER_RATING_USEFULNESS
          │  carries: RATING_ID, HELPFUL_COUNT, ABUSE_COUNT, IP, IP_COUNTRY, CALLEDFROM=RATING_USEFULNESS_WRITE
          ▼
          [CONSUME]  USER_RATING_USEFULNESS.go — dbActionUserRatingUsefulness
          │  — does NOT write to DB directly
          │  — makes an internal HTTP loopback call (utils.CallWapiService,
          │    config key "rating_write_service" → e.g. http://service.intermesh.net/supplierrating)
          ▼
          [HTTP POST — internal loopback]  /supplierrating
          │  payload: {RATING_ID, HELPFUL_COUNT, ABUSE_COUNT, IP, IP_COUNTRY,
          │            CALLEDFROM: "RATING_USEFULNESS_WRITE", UPDATE_FLAG: "U",
          │            UPDATEDBY: "Internal Process (WAPI)",
          │            VALIDATION_KEY: <hardcoded literal, §4 point 10>}
          ▼
          [Re-enters]  UserSupplierRatingController.go → UserModels.UpdateRating()
          │  CALLEDFROM=="RATING_USEFULNESS_WRITE" branch (not IS_ADMIN — a distinct
          │  third branch of UpdateRating, see ../Rating KT/Rating_Technical_Doc.md §5 point 9)
          ▼
          [DB #3 — meshpg]  UPDATE GLUSR_RATING SET RATING_USEFULNESS_COUNT=..., RATING_ABUSE_COUNT=...
          │  WHERE GLUSR_RATING_ID=$1  RETURNING FK_GLUSR_SUPPLIER_ID, FK_GLUSR_BUYER_ID
          ▼
          [RabbitMQ publish]  SERVICENAME=SUPPLIER_RATING_NEW → routing key user.ratingnotification.*
          │  (this is the SAME queue Rating's own admin-review/generic-update paths use —
          │   Rating Usefulness doesn't own this hop, it rides on it)
          ▼
          [Fan-out to 2 bound consumers]
          ├─ USER_RATING_NOTIFICATION.go   → checks for a "MESSAGE" key; RATING_USEFULNESS_WRITE
          │                                   payloads don't carry one → silently Ack's, no push
          │                                   notification sent to the supplier (§4-adjacent finding)
          └─ USER_RATING_NOTIFICATION_PG.go → sees UPDATE_HELPFUL_COUNT marker (set internally by
                                               UpdateRating, temp_arr["UPDATE_HELPFUL_COUNT"]="1")
                                               → [DB #4 — approvalPg] UPDATE GLUSR_RATING SET
                                                 RATING_USEFULNESS_COUNT=..., RATING_ABUSE_COUNT=...
                                                 (replicates the count into a THIRD physical DB)
```

### Flow B — Legacy/likely-dead `MARK_USEFUL_GLID` path (found in `USER_RATING_NOTIFICATION_PG.go`, not reachable from current code)

```
USER_RATING_NOTIFICATION_PG.go has a branch:
    if updateData["MARK_USEFUL_GLID"] present →
        INSERT INTO GLUSR_RATING_USEFULNESS (approvalPg copy) with FK_GLUSR_USR_ID = MARK_USEFUL_GLID

"MARK_USEFUL_GLID" IS a recognized field (UsersValidationMaps.go:502, "SupplierRatingDetailsMap"-
style map) but is NEVER set by RatingUsefulInsert or by any other traced write path in either
repo in this pass. This looks like a leftover from an earlier design where the vote itself
(not just its counts) was meant to be replicated into approvalPg via GLUSR_RATING_USEFULNESS —
current live code (Flow A) only ever sends UPDATE_HELPFUL_COUNT, never MARK_USEFUL_GLID.
Flagged as an Open Question rather than stated as fact.
```

---

## 9. Read Side — Exact Response Field Names (confirmed this pass, corrects prior doc)

No dedicated GET endpoint exists for Rating Usefulness. The counts surface inside
`GET /supplierrating` / `/supplierrating_versions` (read repo, `users-api-go-production`),
which both select `rating_usefulness_count, rating_abuse_count` directly off `GLUSR_RATING`
in their raw SQL — [`UserSupplierRatingModel.go:141,152,184`](../../internal/models/users/UserSupplierRatingModel.go),
same in `UserSupplierRatingModel_v1.go`.

**The prior shallow doc's claimed field names (`HELPFUL_COUNT`/`ABUSE_COUNT`) do not appear in
the actual JSON response.** The real output fields, built by post-processing the raw row
([`UserSupplierRatingModel.go:424-431,492-500`](../../internal/models/users/UserSupplierRatingModel.go)):

| Output field | Derived from | Behavior |
|---|---|---|
| `RATING_REVIEW_ABUSE_CNT` | `rating_abuse_count` column | `null` if the column is null or `"0"` — **a zero abuse-count is never shown as `0`, only as absent** |
| `RATING_REVIEW_USEFULNESS` | `rating_usefulness_count` column | Same null-if-zero rule |
| `RATING_REVIEW_USEFUL_LABEL` | Computed from `RATING_REVIEW_USEFULNESS` | Human-readable string: `"<n> user found this helpful"` (n=1) or `"<n> users found this helpful"` (n>1); `null` if the count is `0`/absent — [`UserSupplierRatingModel.go:492-500`](../../internal/models/users/UserSupplierRatingModel.go) |

This is a genuinely useful finding: **any UI or downstream consumer reading this API for
"how many people found this helpful" must handle `null` as "zero," not treat null as missing
data.**

---

## 10. Cron Inventory

**No cron found for this feature.** Grepped `service-api-go-production/crons/` for
"usefulness" — zero hits. Unlike the Rating domain (which has `avg_rating.go` and
`rating_suspect_cron.go`), Rating Usefulness has no scheduled batch job of its own; it is a
purely event-driven (RabbitMQ) feature.

---

## 11. Flow-wise DB & Table Usage Matrix

### Flow A — Helpful/abuse vote (full round-trip)

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1 | meshpg | `GLUSR_RATING_USEFULNESS` | INSERT | Record the individual vote (per buyer, per rating) |
| 2 | meshpg | `GLUSR_RATING_USEFULNESS` | SELECT COUNT (conditional CASE-aggregation) | Recompute the current helpful/abuse totals for this rating, only reached if #1 succeeded |
| 3 (async, via HTTP loopback + `UpdateRating`) | meshpg | `GLUSR_RATING` | UPDATE (`RATING_USEFULNESS_COUNT`, `RATING_ABUSE_COUNT`) | Persist the new counts onto the parent rating row — this is what `GET /supplierrating` reads |
| 4 (async, `USER_RATING_NOTIFICATION_PG`, via `SUPPLIER_RATING_NEW` fan-out) | approvalPg | `GLUSR_RATING` | UPDATE (same two columns) | Replicate the new counts into the approval-workflow database copy — **not documented in the prior pass** |

**Total DB round-trips for one vote: at least 4**, spanning **2 physical databases** (meshpg,
approvalPg) — more than the prior doc's picture of "1 insert + 1 count + 1 update."

### Flow B — Duplicate vote (fast-fail path)

| # | DB | Table | Operation | Why |
|---|---|---|---|---|
| 1 | meshpg | `GLUSR_RATING_USEFULNESS` | INSERT (fails, e.g. unique-constraint violation) | Attempted insert triggers the failure that gets reclassified as "Record already exists" |

No count-recompute, no queue push, no downstream writes — the cheapest path in this feature,
by design (§4 point 6).

---

## 12. Optimization Scope — DB/Response-Time Contribution

Ranked High/Medium/Low, code-cited.

### High-impact

1. **Consumer → HTTP-loopback-call → dobara controller-validation → phir DB write, replicated
   into a SECOND DB via a THIRD hop** — the full chain for a single vote is: DB insert → DB
   count query → RabbitMQ publish → consumer dequeue → HTTP call → controller re-validation →
   DB update (meshpg) → **second** RabbitMQ publish (`SUPPLIER_RATING_NEW`) → **second**
   consumer dequeue → DB update (approvalPg). If `rating_write_service` (pointing at
   `/supplierrating`) is ever slow or down, **both** `USER_RATING_USEFULNESS` and (indirectly)
   `USER_RATING_NOTIFICATION_PG`'s backlog grow — this is a longer, more indirect dependency
   chain than the prior doc captured (which stopped at the first HTTP hop).
   [`USER_RATING_USEFULNESS.go:82`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_RATING_USEFULNESS.go)
2. **Duplicate-vote check depends on a failed INSERT** (§4 point 5) — the database absorbs a
   failed write-attempt (with rollback overhead) on every repeat vote, instead of a cheap
   `SELECT EXISTS(...)` pre-check. **Concrete fix**: check existence first, skip the INSERT
   attempt entirely for known duplicates.

### Medium-impact

3. **Count-recompute is a full `COUNT(...)` scan of `GLUSR_RATING_USEFULNESS` per vote** — fine
   at small per-rating vote volumes (a rating going viral enough to have hundreds of votes is
   unlikely but not impossible), low-priority to optimize proactively but worth an index check
   on `FK_GLUSR_RATING_ID` if it isn't already indexed.
4. **Two separate map literals independently define the same `HELPFUL_COUNT`→`RATING_USEFULNESS_COUNT`
   / `ABUSE_COUNT`→`RATING_ABUSE_COUNT` mapping** (`RatingUsefulMap`-adjacent literal at
   [`UsersValidationMaps.go:509-510`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go)
   and `MapSupplierRatingQuery` at [`UsersValidationMaps.go:533-534`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go))
   — a future rename of either column requires editing both, with no compiler-level link
   between them.

### Low-impact / good practice already present

5. **Count-sync only fires on a genuine new vote** (§4 point 6) — duplicate votes generate zero
   downstream RabbitMQ traffic or HTTP calls, which is efficient filtering already in place.
6. **`USER_RATING_NOTIFICATION.go` correctly no-ops on this message shape** (no `MESSAGE` key)
   rather than sending a spurious push notification to the supplier for every helpful/abuse
   vote — a good absence-of-side-effect, worth confirming is intentional rather than
   accidental (see Open Questions).

---

## 13. Edge Cases & Gotchas (technical POV)

1. **HTTP-loopback dependency, now confirmed two hops deep** (§8, §12 point 1) — if
   `rating_write_service` config-key ever points at the wrong URL or is down, helpful/abuse
   counts go stale in `GLUSR_RATING` (meshpg) **and** the `approvalPg` replica never gets the
   update either, since it's entirely downstream of the same HTTP call succeeding.
2. **Self-vote block missing** (confirmed absent in code, §4 point 7) — no check compares the
   voting buyer against the rating's original buyer.
3. **Generic failed-INSERT-as-duplicate assumption, and it silently returns HTTP SUCCESS**
   (§4 point 5) — a genuine transient DB error (not a duplicate) is indistinguishable from a
   real duplicate to the caller; both come back as 200/"Record already exists."
4. **Zero-count is represented as `null`, not `0`, on read** (§9) — any UI/consumer must treat
   `RATING_REVIEW_USEFULNESS: null` as zero, not as "data missing."
5. **Legacy/dead `MARK_USEFUL_GLID` branch in `USER_RATING_NOTIFICATION_PG.go`** (§8 Flow B) —
   references a field no live write path sets. Either genuinely dead code, or a hint that an
   older design replicated the vote record itself (not just the count) into `approvalPg`.
6. **Two separate allowed-caller lists for functionally-linked features** (§4 point 3) —
   `RatingUsefulHash` (this feature) vs. Rating-submit's own allowlist — adding a new caller
   to one doesn't automatically enable it for the other.
7. **Three different names for the same data at three layers** (§3) — API input
   (`HELPFUL_COUNT`/`ABUSE_COUNT`) → DB column (`RATING_USEFULNESS_COUNT`/`RATING_ABUSE_COUNT`)
   → API output (`RATING_REVIEW_USEFULNESS`/`RATING_REVIEW_ABUSE_CNT`) — easy to grep for the
   wrong name and conclude a piece of this pipeline doesn't exist.

---

## 14. Open Questions

1. Is the `MARK_USEFUL_GLID` branch in `USER_RATING_NOTIFICATION_PG.go` (§8 Flow B) truly dead
   code, or does some other, untraced write path still set it? No live publisher was found in
   either repo in this pass.
2. Self-vote (voting on your own rating) — genuinely allowed, or an accidental gap? No
   explicit block found anywhere in this domain.
3. Does `USER_RATING_NOTIFICATION.go`'s silent no-op on `RATING_USEFULNESS_WRITE`-shaped
   messages (§8, §12 point 6) reflect an intentional design ("no notification for
   helpful/abuse votes") or is a `MESSAGE`-key omission from the payload an oversight that
   accidentally suppresses a notification that was meant to exist?
4. `RatingUsefulHash` vs. Rating-submit's allowlist — should these be merged into one shared
   config, given both gate the same underlying rating record?
5. Is there a unique constraint on `(FK_GLUSR_RATING_ID, FK_GLUSR_USR_ID)` in
   `GLUSR_RATING_USEFULNESS` enforcing "one vote per buyer per rating" at the DB level, or is
   this only enforced by the application's failed-insert-as-duplicate assumption (§4 point 5)?
   Live schema not verified in this pass.
6. Live DB schema verification generally — this doc reflects only what Go SQL strings imply.

---

## See also

- [`Rating_Usefulness_Business_Doc.md`](./Rating_Usefulness_Business_Doc.md) — product
  perspective
- [`../Rating KT/Rating_Technical_Doc.md`](../Rating%20KT/Rating_Technical_Doc.md) — Rating
  (star-rating) domain, jispe yeh feature depend karta hai (`UpdateRating`,
  `CALLEDFROM=="RATING_USEFULNESS_WRITE"` branch §5 point 8-9, `SUPPLIER_RATING_NEW` routing §6,
  `USER_RATING_NOTIFICATION_PG`'s misleading name §13 point 1)
- [`../ratings_reviews_product_overview.md`](../ratings_reviews_product_overview.md) —
  bigger Ratings & Reviews story

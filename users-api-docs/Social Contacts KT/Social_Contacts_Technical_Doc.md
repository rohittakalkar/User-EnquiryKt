# Social Contacts — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`Social_Contacts_Business_Doc.md`](./Social_Contacts_Business_Doc.md)
dekho. Yeh doc GST_Technical_Doc.md jaisi depth follow karta hai — har claim file:line ke
saath, jahan conclusively confirm nahi ho paaya wahan **[INFERRED — confirm with team]** ya
Open Questions mein flag kiya hai.

**Scope note**: yeh **personal social-contact/communication-preference** data
(`GLUSR_USR_SOCIAL_CONTACT`) cover karta hai — **Market Place URL**
(`GLUSR_MKTP_URL`, business-presence links, Google/FB/Instagram business-listing) ek
genuinely alag table/feature hai, dekho
[`../Market Place URL KT/MarketPlaceUrl_Technical_Doc.md`](../Market%20Place%20URL%20KT/MarketPlaceUrl_Technical_Doc.md).
Confusingly, `UserBuyerProfileModel.go`'s own SQL labels its `GLUSR_MKTP_URL` subquery as
`SOCIAL_PROFILES` (section 3) — that alias is **not** this feature, it's Market Place URL.

**Repos**: `service-api-go-production` (sync write), `user-temp-consumers-production`
(async writes — **two independent paths**, see section 4), `users-api-go-production`
(read, embedded — **two independent read sites**, see section 3).

**Methodology**: har claim neeche real source code se trace kiya gaya hai. Jahan code se
pura confirm nahi ho paaya, wahan clearly **[INFERRED — confirm with team]** likha hai.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Sync write (dedicated API) | write | [`UserSocialContactsController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSocialContactsController.go), [`UserSocialContactModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSocialContactModel.go) (`UpsertSocialContact`) |
| Validation (sync path) | write | `MandatoryParamsCheckSocialContacts` — [`UserUtilsMandatory.go:1455`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go), `UserSocialContactMap` (`LengthAndTypeValidations_v3`) — [`UsersValidationMaps.go:585`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |
| Async write — dedicated queue | consumers | [`USER_SOCIAL_CONTACT.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SOCIAL_CONTACT.go) (`workerUserSocialContact`, `db_action`), queue `USER_SOCIAL_CONTACT` |
| Async write — **generic dispatcher, NEW finding** | consumers | [`USER_UPSERT_ADDITIONAL.go:306-320`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_ADDITIONAL.go), calling `utils.SocialInfopg` — [`additional.go:597`](../../user-temp-consumers-production/user-temp-consumers-production/pkg/utils/additional.go), keyed off `SocialMap` — [`map_index.go:1298`](../../user-temp-consumers-production/user-temp-consumers-production/pkg/utils/map_index.go). Queue `USER_UPSERT_ADDITIONAL`, one of the codebase's two "central dispatcher" workers per [`user_consumers_reference.md`](../user_consumers_reference.md) — **not documented in the shallow prior version of this doc at all**. |
| Read — embedded, profile-detail | read | [`UserDetailModel.go:356-361`](../../users-api-go-production/internal/models/users/UserDetailModel.go) — sub-`json_agg` inside the big user-profile-detail query, no dedicated read-controller |
| Read — embedded, buyer-profile **(second read site, NEW finding)** | read | [`UserBuyerProfileModel.go:158-159,278-279`](../../users-api-go-production/internal/models/users/UserBuyerProfileModel.go) — `LEFT JOIN GLUSR_USR_SOCIAL_CONTACT` inside `getBuyerProfileData()`, exposes only `linkedin_url` + `is_whatsapp_active` |

---

## 2. Routes

| Method | serviceName / queue | Repo | Handler |
|---|---|---|---|
| POST | `USER_SOCIAL_CONTACT` (sync, direct-API, `serviceResponse.SERVICE_NAME`) | write | `UserSocialContactsController` |
| — | `USER_SOCIAL_CONTACT` (async, RabbitMQ queue, dedicated) | consumers | `workerUserSocialContact` |
| — | `USER_UPSERT_ADDITIONAL` (async, RabbitMQ queue, generic dispatcher — writes social fields only when message's `OTHERS` block contains `SocialMap` keys) | consumers | `ActionUserUpsertAdditional` → `utils.SocialInfopg` |
| — | embedded in `GET`/profile-detail read (no separate route, part of big `UserDetailModel.go` query) | read | (part of profile-detail response) |
| — | embedded in buyer-profile read (`ActionBuyerProfile` → `GetBuyerProfileModel` → `getBuyerProfileData`) | read | [`BuyerProfileController.go:14`](../../users-api-go-production/internal/controllers/UsersControllers/BuyerProfileController.go) |

No route explicitly named "get social contacts" exists in either read-API's router — confirmed by grep across `users-api-go-production`; both read-sites are sub-selects/joins embedded in larger responses.

---

## 3. Data Model — Table

> **Verification note**: table/column names Go code ke andar embedded SQL strings se liye
> gaye hain (file:line har row ke saamne diya hai). Live DB schema se cross-verify **nahi**
> kiya gaya hai.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_USR_SOCIAL_CONTACT` | meshPg | Personal social-profile links + communication-preference flags | `FK_GLUSR_USR_ID` (unique — `ON CONFLICT` target in dedicated-consumer path, `WHERE` target in sync/generic-dispatcher paths), `GLUSR_USR_GENDER`, `GLUSR_USR_FB_URL`, `GLUSR_USR_LINKEDIN_URL`, `GLUSR_USR_GOGGLEPLUS_URL`, `GLUSR_USR_TWITTER_URL`, `GLUSR_USR_KLOUT_URL`, `GLUSR_USR_INSTAGRAM_URL` (**only** written by the dedicated-consumer path — [`USER_SOCIAL_CONTACT.go:133`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SOCIAL_CONTACT.go); never written by sync-API or generic-dispatcher path, and never read back by either read-site — see Edge Cases #1), `IS_WHATSAPP_ACTIVE`, `WHATSAPP_ACTIVE_MARKED_DATE`, `IS_UPI_ACTIVE` (**dedicated-consumer-path only**, same gap as Instagram), `SOCIAL_CONTACT_LAST_MODIFIED`, `SOCIAL_CONTACT_UPDATED_BY_ID`/`_NAME`/`_SCREEN`/`_URL`/`_IP`/`_IP_COUNTRY` (audit columns, **dedicated-consumer-path only** — neither sync-API nor the generic-dispatcher path stamps these) |

Three write-paths, three different column-coverage subsets — see section 5 for the exact
diff.

---

## 4. The Three Write-Paths — Precise Relationship

This is the central technical finding of this review. There are **not two** but **three**
independent code-paths that can write `GLUSR_USR_SOCIAL_CONTACT`, each with different
triggers, different field-coverage, and different payload shapes:

| # | Path | Trigger | Payload shape | Fields it can touch |
|---|---|---|---|---|
| A | **Sync API** — `UserSocialContactsController`/`UpsertSocialContact` | Direct HTTP call, `serviceName=USER_SOCIAL_CONTACT`, gated to `GLADMIN`/`SELLERMY`/`IMOB`/`MSITE` | lowercase keys: `glusrid`, `gender`, `fb_url`, `linkedin_url`, `gplus_url`, `twitter_url`, `klout_url`, `is_active`, `action` | gender, fb/linkedin/gplus/twitter/klout URLs, whatsapp-active flag |
| B | **Dedicated async consumer** — `workerUserSocialContact`/`db_action`, queue `USER_SOCIAL_CONTACT` | RabbitMQ message on queue `USER_SOCIAL_CONTACT` — **producer not found anywhere in `service-api-go-production`, `users-api-go-production`, or the rest of `user-temp-consumers-production`** | UPPERCASE keys: `FK_GLUSR_USR_ID`, `UPDATEDBY_NAME`, `UPDATEDBY_SCREEN`, `UPDATEDBY_ID/URL/IP/IPCOUNTRY`, plus whichever `colMap` keys (§3) are present | Everything path A can, **plus** `GLUSR_USR_INSTAGRAM_URL`, `IS_UPI_ACTIVE`, and all `SOCIAL_CONTACT_UPDATED_BY_*` audit columns |
| C | **Generic dispatcher** — `ActionUserUpsertAdditional` → `utils.SocialInfopg`, queue `USER_UPSERT_ADDITIONAL` | RabbitMQ message on queue `USER_UPSERT_ADDITIONAL` — one of the codebase's two "central dispatcher" workers, per [`user_consumers_reference.md`](../user_consumers_reference.md), triggered off **any generic `GLUSR_USR` change event**; social-write only fires if the message's `MESSAGE[].OTHERS` block contains keys matching `SocialMap` — [`USER_UPSERT_ADDITIONAL.go:306-320`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_ADDITIONAL.go). Producer of `USER_UPSERT_ADDITIONAL` also **not found in-repo** (see Open Questions) | `SocialMap` keys: `USR_ID`, `USR_GENDER`, `USR_FB_URL`, `USR_LINKEDIN_URL`, `USR_GOGGLEPLUS_URL`, `USR_TWITTER_URL`, `USR_KLOUT_URL` — [`map_index.go:1298-1306`](../../user-temp-consumers-production/user-temp-consumers-production/pkg/utils/map_index.go) | gender + 4 URL fields only — **no** `is_active`/whatsapp, **no** instagram, **no** upi |

**Are they alternative entry-points for the same event, or genuinely different triggers?**
Genuinely different, on the evidence:

1. **Field-name casing/shape differs across all three** (`glusrid` vs `FK_GLUSR_USR_ID` vs
   `USR_ID`) — three independently-designed payload contracts, not one shape reused three
   times.
2. **Path B's queue name (`USER_SOCIAL_CONTACT`) is dedicated and single-purpose.** Its
   consumer does nothing else. This looks like a purpose-built async mirror of the sync
   API's concept (though its own producer isn't traceable in-repo — see Open Questions).
3. **Path C's queue (`USER_UPSERT_ADDITIONAL`) is a generic, multi-domain dispatcher** —
   social-contact writing is one of *dozens* of conditional sub-processes it runs (blacklist
   check, big-buyer flag, bounce-email, WAPI privacy-setting call, Clearout email
   validation, etc. — see [`USER_UPSERT_ADDITIONAL.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_ADDITIONAL.go) lines 90-320).
   This is almost certainly what fires whenever a supplier updates gender/social-URL fields
   through the **general** profile-update flow (e.g. a generic `/details` or similar
   composite-update call whose message happens to carry social keys in its `OTHERS` block),
   not through the dedicated Social Contacts API.
4. **Column coverage strictly nests**: C ⊂ A ⊂ B. Path C covers the fewest fields (gender +
   4 URLs), Path A covers those plus the whatsapp flag, Path B covers everything plus
   Instagram/UPI/audit columns. This is consistent with three teams/eras building
   increasingly complete versions of the same write, not one write duplicated.

**Conclusion**: this is structurally *unlike* a simple dual-write pattern (where a
write-API publishes an event that its own consumer replays). All three paths look like
**genuinely separate integrations that happen to target the same table**, each likely
serving a different upstream caller/flow. None of the three in-repo code paths conclusively
shows *itself* publishing to either of the two async queues — both `USER_SOCIAL_CONTACT`
and `USER_UPSERT_ADDITIONAL`'s producers sit outside these three repos (flagged in Open
Questions).

---

## 5. Business Rules & Validation (code se)

### Path A — Sync write (`UserSocialContactsController` / `UpsertSocialContact`)

1. **Gateway allowlist**: `GLADMIN`, `SELLERMY`, `IMOB`, `MSITE`.
   [`UserSocialContactsController.go:46`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSocialContactsController.go)
2. **Mandatory**: `glusrid`, `VALIDATION_KEY`, `action` (only `i`/`u`). `gender` (if present)
   only `Male`/`Female` (case-insensitive); `is_active` (if present) only `0`/`1`.
   [`UserUtilsMandatory.go:1455-1490`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
3. **Insert (`action=i`) requires at least 2 non-empty string values** among the 8 bound
   params (`glusrid, gender, fb_url, linkedin_url, gplus_url, twitter_url, klout_url,
   is_active`) — a raw count-loop check in application code, not a DB constraint.
   [`UserSocialContactModel.go:82-93`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSocialContactModel.go)
4. **Update (`action=u`) is dynamically built** from whichever keys are present in
   `params` (a Go `switch` over map keys) — genuine partial-update, no need to resend
   unchanged fields.
   [`UserSocialContactModel.go:39-79`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSocialContactModel.go)
5. **`is_active` present → `WHATSAPP_ACTIVE_MARKED_DATE` also stamped** in both insert and
   update branches. [`UserSocialContactModel.go:34-37,75-77`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSocialContactModel.go)
6. **This path never writes `GLUSR_USR_INSTAGRAM_URL` or `IS_UPI_ACTIVE`** — those columns
   are only reachable via Path B's `colMap`. Real field-coverage gap.

### Path B — Dedicated async consumer (`workerUserSocialContact` / `db_action`)

7. **Mandatory**: `FK_GLUSR_USR_ID`, `UPDATEDBY_NAME`, `UPDATEDBY_SCREEN` — missing any of
   these causes the message to be `Ack`'d (silently dropped, not retried) with a Kibana
   FAILURE log, not a requeue.
   [`USER_SOCIAL_CONTACT.go:65-68`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SOCIAL_CONTACT.go)
8. **Always does `INSERT ... ON CONFLICT (FK_GLUSR_USR_ID) DO UPDATE`** — a single statement
   covers both insert and update, unlike Path A's explicit branch.
   [`USER_SOCIAL_CONTACT.go:110-111`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SOCIAL_CONTACT.go)
9. **`IS_WHATSAPP_ACTIVE` present in the message → `WHATSAPP_ACTIVE_MARKED_DATE` also set**
   to `CURRENT_TIMESTAMP` — same business rule as Path A, independently re-implemented.
   [`USER_SOCIAL_CONTACT.go:95-99`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SOCIAL_CONTACT.go)
10. **This queue's producer is not found anywhere in `service-api-go-production`,
    `users-api-go-production`, or the rest of `user-temp-consumers-production`** — flagged
    as Open Question.
11. **DB errors trigger `Nack(false, false)`** (requeue-eligible), while missing-mandatory-
    field or unmarshal errors trigger `Ack(false)` (message dropped) — two different failure
    handling behaviors depending on failure type.
    [`USER_SOCIAL_CONTACT.go:49-51,65-68,71-74`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SOCIAL_CONTACT.go)

### Path C — Generic dispatcher (`ActionUserUpsertAdditional` / `utils.SocialInfopg`)

12. **Fires conditionally, not unconditionally**: for every key in the message's `OTHERS`
    map, if that key exists in `SocialMap`, `SocialInfopg` runs — meaning a
    `USER_UPSERT_ADDITIONAL` message only touches `GLUSR_USR_SOCIAL_CONTACT` when its
    `OTHERS` payload happens to carry social-field keys; otherwise this branch is skipped
    entirely (other sub-processes in the same worker still run independently).
    [`USER_UPSERT_ADDITIONAL.go:306-320`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_ADDITIONAL.go)
13. **`action` (`INSERT`/`UPDATE`) comes from the outer message's top-level `ACTION` field**,
    shared with every other sub-process in this dispatcher — not independently decided per
    social-contact write. [`USER_UPSERT_ADDITIONAL.go:103-104`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_ADDITIONAL.go)
14. **Self-healing UPDATE→INSERT fallback**: if the `UPDATE` statement affects 0 rows,
    `SocialInfopg` recursively calls itself with `action="INSERT"` — a resync pattern also
    seen elsewhere in this consumer repo (`USER_ADDT_CONTACT_TRUSTPG`, `USER_BANK_DETAILS_TRUSTPG`
    per [`user_consumers_reference.md`](../user_consumers_reference.md)).
    [`additional.go:638-641`](../../user-temp-consumers-production/user-temp-consumers-production/pkg/utils/additional.go)
15. **Narrowest field coverage of the three paths** — only gender + 4 URL fields
    (`SocialMap`, §4); does not touch whatsapp-active, Instagram, or UPI at all.
16. **Failure here triggers a full-message requeue** (`utils.Requeue`) of the *entire*
    `USER_UPSERT_ADDITIONAL` message, not just the social sub-process — meaning a
    social-write failure can cause blacklist/big-buyer/email-validation sub-processes to
    re-run too on retry. [`USER_UPSERT_ADDITIONAL.go:316-320`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_ADDITIONAL.go)

---

## 6. RabbitMQ

| Queue / serviceName | Publisher | Consumer | Purpose |
|---|---|---|---|
| `USER_SOCIAL_CONTACT` | **Not found in any of the 3 repos** — external/legacy producer, or a system outside this codebase's scope | [`USER_SOCIAL_CONTACT.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SOCIAL_CONTACT.go) (`workerUserSocialContact`), concurrency 5 goroutines ([`router.go:239`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Router/router.go)) | Independent async upsert path into `GLUSR_USR_SOCIAL_CONTACT`, widest field-coverage of the 3 write-paths |
| `USER_UPSERT_ADDITIONAL` | **Not found in any of the 3 repos either** — [`user_consumers_reference.md`](../user_consumers_reference.md) describes it only as "triggered off any GLUSR update," without naming the publisher | [`USER_UPSERT_ADDITIONAL.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_UPSERT_ADDITIONAL.go) (`ActionUserUpsertAdditional`) | Generic multi-domain dispatcher; writes `GLUSR_USR_SOCIAL_CONTACT` only when its message's `OTHERS` block carries `SocialMap` keys — one of dozens of conditional sub-processes it runs |

No queue in this feature's scope is confirmed as producer-traced within the 3 repos —
both are consumed but never published in-repo. This mirrors the shallow doc's original
finding for `USER_SOCIAL_CONTACT` and extends it to a second queue newly found in this pass.

---

## 7. Kafka

**Koi Kafka usage nahi mila** iss feature (any of the 3 write-paths, or either read-site)
mein — confirmed via grep across all three repos for `social_contact`/`SocialContact`
alongside Kafka-specific helpers (`InitializeKafka`, `sub_topic`, `consumer_group`). All
messaging touching this table is RabbitMQ-based.

---

## 8. Redis

**Koi Redis usage nahi mila** iss feature mein, across all three write-paths and both
read-sites. Both reads (`UserDetailModel.go`'s `json_agg` sub-select and
`UserBuyerProfileModel.go`'s `LEFT JOIN`) hit Postgres live on every request — no caching
layer wired in for this table.

---

## 9. End-to-End Technical Flows

### Flow A — Sync write (direct API)
```
Supplier / GLADMIN / SELLERMY / IMOB / MSITE
    │
    ▼
[API — write]  POST serviceName=USER_SOCIAL_CONTACT
                {glusrid, action(i/u), gender, fb_url, linkedin_url, gplus_url,
                 twitter_url, klout_url, is_active, VALIDATION_KEY}
    │  UserSocialContactsController.go — Gateway check (GLADMIN/SELLERMY/IMOB/MSITE)
    │  MandatoryParamsCheckSocialContacts() → LengthAndTypeValidations_v3()
    ▼
UpsertSocialContact()
    ├─ action=i → INSERT (min. 2 non-empty values required)
    └─ action=u → dynamically-built partial UPDATE
    ▼
[DB — meshpg]  GLUSR_USR_SOCIAL_CONTACT
    ▼
Response {SUCCESS IN PG}
```

### Flow B — Dedicated async write
```
[External/legacy producer — not found in this codebase]
    │
    ▼
[RabbitMQ]  queue=USER_SOCIAL_CONTACT
    ▼
[CONSUME]  workerUserSocialContact() → db_action()
    │  requires FK_GLUSR_USR_ID, UPDATEDBY_NAME, UPDATEDBY_SCREEN
    │  (missing → Ack+drop; DB error → Nack+requeue)
    ▼
[DB — meshPg]  INSERT ... ON CONFLICT (FK_GLUSR_USR_ID) DO UPDATE
                GLUSR_USR_SOCIAL_CONTACT (includes INSTAGRAM_URL, IS_UPI_ACTIVE,
                full audit columns — widest of the 3 paths)
```

### Flow C — Generic-dispatcher async write (NEW finding)
```
[Producer — not found in this codebase, likely a generic profile-update publisher]
    │
    ▼
[RabbitMQ]  queue=USER_UPSERT_ADDITIONAL
    ▼
[CONSUME]  ActionUserUpsertAdditional()
    │  runs dozens of unrelated sub-processes first (blacklist, big-buyer, WAPI privacy
    │  setting, Clearout email validation, ...)
    │
    ├─ for key in message.OTHERS: if key ∈ SocialMap →
    │       utils.SocialInfopg(action, otherProcess, glid)
    │       ├─ UPDATE GLUSR_USR_SOCIAL_CONTACT SET ... WHERE FK_GLUSR_USR_ID=$n
    │       └─ if 0 rows affected → recursive self-call with action=INSERT
    │
    └─ any sub-process failure → full-message Requeue (all sub-processes may re-run)
    ▼
[DB — meshPg]  GLUSR_USR_SOCIAL_CONTACT (gender + 4 URL fields only)
```

### Flow D — Read #1: embedded in profile-detail (no dedicated endpoint)
```
Buyer / any caller of supplier-profile-detail
    │
    ▼
[API — read]  UserDetailModel.go big profile query
    │  sub-select: json_agg(...) FROM GLUSR_USR_SOCIAL_CONTACT WHERE fk_glusr_usr_id=$1
    │  columns returned: fb_url, gender, goggleplus_url, klout_url, linkedin_url,
    │                    twitter_url — NOT whatsapp/upi/instagram
    ▼
Response — social-contact fields embedded as a JSON sub-object inside the larger
           profile-detail response
```

### Flow E — Read #2: embedded in buyer-profile (second read site, NEW finding)
```
Buyer viewing a supplier's profile (matchmaking/enquiry context)
    │
    ▼
[API — read]  ActionBuyerProfile → GetBuyerProfileModel → getBuyerProfileData
    │  LEFT JOIN GLUSR_USR_SOCIAL_CONTACT ON A.CONTACTS_GLID = ...FK_GLUSR_USR_ID
    │  exposes only: linkedin_url, is_whatsapp_active (COALESCE'd to 0 if null)
    ▼
Response — flat fields inside the larger buyer-profile response (NOT the "SOCIAL_PROFILES"
           key — that's a different subquery over GLUSR_MKTP_URL, see scope note above)
```

---

## 10. Flow-wise DB & Table Usage

### Flow A — Sync write

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `GLUSR_USR_SOCIAL_CONTACT` | INSERT or UPDATE (single statement, chosen by `action`) | The entire write, in one round-trip — no pre-read/lock-check unlike GST's flow |

### Flow B — Dedicated async write

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshPg | `GLUSR_USR_SOCIAL_CONTACT` | INSERT ... ON CONFLICT DO UPDATE (single statement) | Single round-trip upsert, most fields covered of the 3 paths |

### Flow C — Generic-dispatcher write

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshPg | `GLUSR_USR_SOCIAL_CONTACT` | UPDATE | First attempt, using whatever `SocialMap` keys are present |
| 2 | meshPg | `GLUSR_USR_SOCIAL_CONTACT` | INSERT (only if step 1 affected 0 rows) | Self-healing fallback — this makes Flow C potentially **2 round-trips** on a first-time write for a GLID, vs. 1 for Flows A/B |

*(Note: Flow C's DB calls happen after several other unrelated sub-process DB/HTTP calls in
the same message-handling function — blacklist check, big-buyer mesh write, WAPI calls — so
the total round-trip count for a `USER_UPSERT_ADDITIONAL` message is much higher than what's
shown here; only the social-contact-relevant calls are listed.)*

### Flow D — Read #1 (profile-detail)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | read-API's DB (meshpg-equivalent) | `GLUSR_USR_SOCIAL_CONTACT` | SELECT (sub-`json_agg`, part of one larger multi-table query) | Embeds gender/URL fields into the profile-detail response; not a separate round-trip from the rest of that query since it's a correlated sub-select in the same statement |

### Flow E — Read #2 (buyer-profile)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | read-API's DB | `GLUSR_USR_SOCIAL_CONTACT` | SELECT (`LEFT JOIN`, part of one larger multi-table query) | Pulls `linkedin_url` + `is_whatsapp_active` into the buyer-facing profile response; again not a separate round-trip, joined into the same statement |

---

## 11. Optimization Scope — DB Response-Time Contribution

### Medium impact

1. **Flow C (generic dispatcher) can cost 2 sequential round-trips on a fresh GLID**
   (UPDATE-then-INSERT-on-0-rows, §10 Flow C) vs. 1 for the sync API and dedicated
   consumer — same self-healing pattern used elsewhere in this consumer repo, so likely an
   accepted tradeoff, but worth knowing this specific path is not single-round-trip.
   [`additional.go:630-641`](../../user-temp-consumers-production/user-temp-consumers-production/pkg/utils/additional.go)
2. **Three independent write-paths targeting one table means three independent
   query-plans/indexes to reason about** if this table is ever a write-hotspot — no
   pooling/serialization concern is visible in code, but any future locking/contention
   investigation on `GLUSR_USR_SOCIAL_CONTACT` needs to consider all three paths, not just
   the obvious sync API.
3. **No caching on either read-site** — both `UserDetailModel.go`'s `json_agg` sub-select
   and `UserBuyerProfileModel.go`'s `LEFT JOIN` hit Postgres live on every request. Given
   this data changes infrequently per-supplier (occasional profile edits), and both reads
   are embedded in already-large composite queries, this is a low-urgency but real caching
   gap — same shape of gap the GST doc flags for its own read endpoints.

### Low impact

4. **Sync-path insert/update's field-count-check (`count < 2`) is an in-app loop** — DB
   round-trip se pehle hi reject hota hai agar kaafi values nahi hain, achi practice hai
   (avoids an unnecessary DB call for near-empty inserts).
   [`UserSocialContactModel.go:82-93`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSocialContactModel.go)
5. **Dedicated-consumer path's single `ON CONFLICT` statement** is already efficient —
   single round-trip covers both insert and update, the best pattern of the three paths.
6. **Field-coverage gap across the 3 paths (§5, points 6/15)** — no DB-response-time
   impact, but a real data-consistency risk: a caller relying only on the sync API or the
   generic dispatcher will never see Instagram-URL/UPI-status take effect, and won't know
   those fields are silently unreachable from their integration.

---

## 12. Cron Inventory

Grepped `service-api-go-production/crons`, `user-temp-consumers-production` for any
cron/standalone-binary referencing `GLUSR_USR_SOCIAL_CONTACT`, `social_contact`, or
`SocialContact` (case-insensitive) — **no matches found**. **No cron touches this feature.**

---

## 13. Edge Cases & Gotchas (technical POV)

1. **Instagram-URL and UPI-active-flag are write-only, dead-end data**: only Path B
   (dedicated consumer) can write `GLUSR_USR_INSTAGRAM_URL`/`IS_UPI_ACTIVE`, and **neither
   read-site selects these columns** (`UserDetailModel.go`'s `json_agg` list and
   `UserBuyerProfileModel.go`'s `LEFT JOIN` both omit them). If these fields are actually
   populated by Path B in production, that data currently has no confirmed way to reach any
   API response in these 3 repos — worth confirming with the team whether another repo
   reads them, or whether they're genuinely unused.
2. **Three, not two, write-paths** — anyone debugging "why did this supplier's social data
   change unexpectedly" needs to check all three: sync API logs, `USER_SOCIAL_CONTACT`
   consumer logs, **and** `USER_UPSERT_ADDITIONAL` consumer logs (where social-write is one
   of dozens of buried sub-processes, easy to miss).
3. **Neither async queue's producer is traceable in-repo** — debugging "where did this
   message come from" requires looking outside `service-api-go-production`,
   `users-api-go-production`, and `user-temp-consumers-production`.
4. **Field-name-casing differs across all three paths** (`glusrid`/`fb_url` lowercase in
   Path A; `FK_GLUSR_USR_ID`/`GLUSR_USR_FB_URL` uppercase in Path B; `USR_ID`/`USR_FB_URL`
   in Path C) — confirms these are three genuinely-separate integrations, not the same
   contract reused.
5. **Path C's failure handling is coarse-grained**: a `SocialInfopg` error causes the
   *entire* `USER_UPSERT_ADDITIONAL` message to be requeued, which re-runs every other
   sub-process in that dispatcher (blacklist check, big-buyer flag, WAPI calls, Clearout
   email validation) — not just the social-contact write. A transient social-write failure
   can cause unrelated side-effects to fire twice.
6. **"SOCIAL_PROFILES" naming trap** — `UserBuyerProfileModel.go`'s own SQL aliases a
   *different* table's subquery (`GLUSR_MKTP_URL`) as `SOCIAL_PROFILES`, right next to the
   actual `GLUSR_USR_SOCIAL_CONTACT` join a few lines later. Easy to misattribute business
   meaning if reading the query casually.
7. **`SOCIAL_CONTACT_UPDATED_BY_*` audit columns are only ever populated by Path B** — if a
   record was last touched by Path A or Path C, these audit columns simply retain whatever
   value they had from the last Path-B write (or remain null if never touched by Path B) —
   not a rolling "last modified by" trail across all writers.

---

## 14. Open Questions

1. Who publishes to the `USER_SOCIAL_CONTACT` RabbitMQ queue (Path B)? Not found in any of
   the 3 repos.
2. Who publishes to the `USER_UPSERT_ADDITIONAL` RabbitMQ queue (Path C)? Also not found in
   any of the 3 repos — [`user_consumers_reference.md`](../user_consumers_reference.md)
   only describes it as "triggered off any GLUSR update" without naming the publisher.
3. Is the `GLUSR_USR_INSTAGRAM_URL`/`IS_UPI_ACTIVE` data (write-only via Path B, per Edge
   Case #1) actually read anywhere — a different repo, a BI/reporting pipeline, a
   client-side cache built from some other source? Not confirmable from these 3 repos.
4. Why does Path C (`SocialMap`) cover only 5 of the ~9 writable fields, and why does it
   exist as a side-effect of a *generic* dispatcher rather than being consolidated into
   Path A or B? Intentional historical design, or an accreted duplicate?
5. Live DB schema verification — this doc reflects only what Go SQL strings imply, not a
   pgAdmin/schema cross-check.

---

## See also

- [`Social_Contacts_Business_Doc.md`](./Social_Contacts_Business_Doc.md) — product perspective
- [`../Market Place URL KT/MarketPlaceUrl_Technical_Doc.md`](../Market%20Place%20URL%20KT/MarketPlaceUrl_Technical_Doc.md) —
  related but separate business-presence-URL system
- [`../user_consumers_reference.md`](../user_consumers_reference.md) — full consumer
  inventory, including the `USER_UPSERT_ADDITIONAL`/`USER_CENTRALIZED_QUEUE` "central
  dispatcher" pattern referenced in section 4

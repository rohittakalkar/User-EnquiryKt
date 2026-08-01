# Social Contacts — Technical Doc (Code-Level Deep Dive)

Business/product perspective ke liye [`Social_Contacts_Business_Doc.md`](./Social_Contacts_Business_Doc.md)
dekho.

**Scope note**: yeh **personal social-contact/communication-preference** data
(`GLUSR_USR_SOCIAL_CONTACT`) cover karta hai — **Market Place URL**
(`GLUSR_MKTP_URL`, business-presence links) ek genuinely alag table/feature hai, dekho
[`../Market Place URL KT/MarketPlaceUrl_Technical_Doc.md`](../Market%20Place%20URL%20KT/MarketPlaceUrl_Technical_Doc.md).

**Repos**: `service-api-go-production` (write, sync), `user-temp-consumers-production`
(write, async — separate/independent path), `users-api-go-production` (read, embedded).

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Sync write | write | [`UserSocialContactsController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSocialContactsController.go), [`UserSocialContactModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSocialContactModel.go) (`UpsertSocialContact`) |
| Validation | write | `MandatoryParamsCheckSocialContacts` — [`UserUtilsMandatory.go:1455`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go), `UserSocialContactMap` (`LengthAndTypeValidations_v3`) — [`UsersValidationMaps.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) |
| Async write (separate path) | consumers | [`USER_SOCIAL_CONTACT.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SOCIAL_CONTACT.go) (`workerUserSocialContact`, `db_action`) — RabbitMQ-driven |
| Read (embedded) | read | [`UserDetailModel.go:356-361`](../../users-api-go-production/internal/models/users/UserDetailModel.go) — sub-`json_agg` inside the big user-profile-detail query, **no dedicated read-controller** |

---

## 2. Routes

| Method | serviceName | Repo | Controller |
|---|---|---|---|
| POST | `USER_SOCIAL_CONTACT` (sync, direct-API) | write | `UserSocialContactsController` |
| — | `USER_SOCIAL_CONTACT` (async, RabbitMQ queue) | consumers | `workerUserSocialContact` |
| — | embedded in supplier-profile-detail read | read | (part of `UserDetailModel.go`'s big query) |

---

## 3. Data Model — Table

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_USR_SOCIAL_CONTACT` | meshPg | Personal social-profile links + communication-preference flags | `FK_GLUSR_USR_ID` (unique — `ON CONFLICT` target in consumer path), `GLUSR_USR_GENDER`, `GLUSR_USR_FB_URL`, `GLUSR_USR_LINKEDIN_URL`, `GLUSR_USR_GOGGLEPLUS_URL`, `GLUSR_USR_TWITTER_URL`, `GLUSR_USR_KLOUT_URL`, `GLUSR_USR_INSTAGRAM_URL` (consumer-path-only column, sync-controller doesn't write it — see Open Questions), `IS_WHATSAPP_ACTIVE`, `WHATSAPP_ACTIVE_MARKED_DATE`, `IS_UPI_ACTIVE` (consumer-path-only), `SOCIAL_CONTACT_LAST_MODIFIED`, `SOCIAL_CONTACT_UPDATED_BY_ID`/`_NAME`/`_SCREEN`/`_URL`/`_IP`/`_IP_COUNTRY` (consumer-path-only audit columns) |

---

## 4. Business Rules & Validation (code se)

### Sync write-path (`UserSocialContactsController` / `UpsertSocialContact`)

1. **Gateway allowlist**: `GLADMIN`, `SELLERMY`, `IMOB`, `MSITE`.
   [`UserSocialContactsController.go:46`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserSocialContactsController.go)
2. **Mandatory**: `glusrid`, `VALIDATION_KEY`, `action` (only `i`/`u`). `gender` (if present)
   only `Male`/`Female`; `is_active` (if present) only `0`/`1`.
   [`UserUtilsMandatory.go:1455-1485`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserUtilsMandatory.go)
3. **Insert (`action=i`) requires at least 2 non-empty values** among the optional fields —
   raw count-loop check, not a DB constraint.
   [`UserSocialContactModel.go:82-93`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserSocialContactModel.go)
4. **Update (`action=u`) is dynamically built** from whichever keys are present in
   `params` — genuine partial-update, no need to resend unchanged fields.
5. **`is_active` present → `WHATSAPP_ACTIVE_MARKED_DATE` also stamped** (insert and
   update both).
6. **This path does NOT write `GLUSR_USR_INSTAGRAM_URL` or `IS_UPI_ACTIVE`** — those
   columns are only handled by the async consumer path (`colMap` in
   [`USER_SOCIAL_CONTACT.go:124-141`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SOCIAL_CONTACT.go)) — a real field-coverage gap between
   the two write-paths.

### Async write-path (`workerUserSocialContact` / `db_action`)

7. **Mandatory**: `FK_GLUSR_USR_ID`, `UPDATEDBY_NAME`, `UPDATEDBY_SCREEN` — different
   shape/casing from the sync-path's params (`glusrid` vs `FK_GLUSR_USR_ID`), confirming
   this is a **separate, independently-designed producer/payload**, not just an async
   mirror of the sync-API's own request.
8. **Always does `INSERT ... ON CONFLICT (FK_GLUSR_USR_ID) DO UPDATE`** — single
   statement covers both insert and update, unlike the sync-path's explicit branch.
9. **This queue's producer is not found anywhere in `service-api-go-production` or
   `users-api-go-production`** — likely an external/legacy system (outside these 3 repos)
   publishes to it directly (registration-flow candidate, not confirmed) — flagged as Open
   Question.

---

## 5. RabbitMQ / Kafka / Redis

| Queue / serviceName | Publisher | Consumer | Purpose |
|---|---|---|---|
| `USER_SOCIAL_CONTACT` | **not found in this codebase** (external/legacy producer, not `service-api-go-production`) | [`USER_SOCIAL_CONTACT.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_SOCIAL_CONTACT.go) (`workerUserSocialContact`) | Independent async upsert path into the same `GLUSR_USR_SOCIAL_CONTACT` table |

**Koi Kafka ya Redis usage nahi mila** iss feature mein.

**Important finding**: is codebase mein do **completely independent** write-paths hain
jo same table target karte hain — (a) `service-api-go-production`'s synchronous
`UserSocialContactsController` (direct API call), aur (b) `user-temp-consumers-production`'s
RabbitMQ-consumer `USER_SOCIAL_CONTACT` (whose publisher/producer in-repo nahi mila).
Yeh Flips KT jaisa "dual-write" pattern nahi hai (wahan write-API khud publish karta tha) —
yahan dono paths genuinely **alag entry-points** lagte hain, alag column-coverage ke saath.

---

## 6. End-to-End Technical Flow

### Sync write (direct API)
```
Supplier / GLADMIN / SELLERMY / IMOB / MSITE
    │
    ▼
[API — write]  POST serviceName=USER_SOCIAL_CONTACT
                {glusrid, action(i/u), gender, fb_url, linkedin_url, gplus_url,
                 twitter_url, klout_url, is_active, VALIDATION_KEY}
    │  UserSocialContactsController.go — Gateway check
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

### Async write (independent path)
```
[External/legacy producer — not in this codebase]
    │
    ▼
[RabbitMQ]  queue=USER_SOCIAL_CONTACT
    ▼
[CONSUME]  workerUserSocialContact() → db_action()
    │  requires FK_GLUSR_USR_ID, UPDATEDBY_NAME, UPDATEDBY_SCREEN
    ▼
[DB — meshPg]  INSERT ... ON CONFLICT (FK_GLUSR_USR_ID) DO UPDATE
                GLUSR_USR_SOCIAL_CONTACT (includes INSTAGRAM_URL, IS_UPI_ACTIVE —
                fields sync-path doesn't cover)
```

### Read (embedded, no dedicated endpoint)
```
Buyer / any caller of supplier-profile-detail
    │
    ▼
[API — read]  UserDetailModel.go big profile query
    │  sub-select: json_agg(...) FROM GLUSR_USR_SOCIAL_CONTACT WHERE fk_glusr_usr_id=$1
    ▼
Response — social-contact fields embedded as a JSON sub-object inside the larger
           profile-detail response
```

---

## 7. Optimization Scope — DB Response-Time Contribution

### Low-Medium impact

1. **Sync-path insert/update ka field-count-check (`count < 2`) ek in-app loop hai** — DB
   round-trip se pehle hi reject hota hai agar kaafi values nahi hain, achi practice hai
   (unnecessary DB call avoid).
2. **Async-path ka single `ON CONFLICT` statement** already efficient hai — GST/Rating
   docs mein bhi yeh pattern best-practice ke roop mein suggest hua hai.
3. **Read-side embedded `json_agg` subquery** — chhoti table hai (max 1 row per user given
   the unique-conflict-target), low-cost, koi optimization-concern nahi.

### Low impact

4. **Do independent write-paths ka field-coverage-gap (§4, point 6)** — koi DB-response-
   time-impact nahi, lekin data-consistency-risk hai: agar Instagram-URL/UPI-status sirf
   async-path se aata hai, sync-API se update karne wale caller ko pata nahi chalega ki
   yeh fields miss ho rahi hain.

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora Social Contacts flowchart yahan dekho](https://lucid.app/lucidchart/fcb0d002-72d3-4c0c-9086-7f012e715a18/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **Instagram-URL aur UPI-active-flag sirf async-consumer-path se set ho sakte hain** —
   agar koi caller sirf sync-API use karta hai, yeh fields kabhi update nahi ho paayenge
   us path se.
2. **Async-path ka producer in-repo nahi mila** — agar iska debugging/tracing karna pade
   (jaise "yeh data kahan se aaya"), external/legacy system ki taraf dekhna padega.
3. **Sync-path ka field-name-casing (`glusrid`, `fb_url`, lowercase) async-path se
   (`FK_GLUSR_USR_ID`, `GLUSR_USR_FB_URL`, uppercase) bilkul alag hai** — confirm karta
   hai ki yeh do genuinely-separate integrations hain, ek hi cheez ke do naam nahi.

---

## 10. Open Questions

1. `USER_SOCIAL_CONTACT` RabbitMQ queue ka producer kaun hai — koi legacy/external system,
   ya koi is codebase ke bahar ka service? In-repo nahi mila.
2. Sync-path Instagram-URL/UPI-active kyun nahi handle karta — intentional scope-limitation
   hai ya missed field?
3. Read-side ke alawa koi dedicated "GET social contacts" endpoint hai kya, ya hamesha
   embedded hi rehta hai?
4. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`Social_Contacts_Business_Doc.md`](./Social_Contacts_Business_Doc.md) — product perspective
- [`../Market Place URL KT/MarketPlaceUrl_Technical_Doc.md`](../Market%20Place%20URL%20KT/MarketPlaceUrl_Technical_Doc.md) —
  related but separate business-presence-URL system

# User Blacklist & Blacklist Values — Technical Doc (Code-Level Deep Dive)

Yeh doc "Blacklist" naam ke neeche chal rahe **teen alag data-sources** ka technical
implementation cover karta hai — APIs, DB tables, queries, Kafka/RabbitMQ/Redis, consumers,
crons, sab kuch code se verify karke. Business/product perspective ke liye
[`Blacklist_Business_Doc.md`](./Blacklist_Business_Doc.md) dekho — dono docs same flows cover
karte hain, bas alag audience ke liye.

**Repos**: `service-api-go-production` (write), `users-api-go-production` (read),
`user-temp-consumers-production` (consumers).

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file path + line
diya gaya hai har jagah). Jahan code se pura confirm nahi ho paaya, wahan clearly
**[INFERRED — confirm with team]** likha hai, guess nahi kiya gaya.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Manual blacklist-value update (write) | write | [`BlacklistValuesController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/BlacklistValuesController.go), [`BlacklistValuesModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/BlacklistValuesModel.go) |
| Manual blacklist-value read (by attribute) + fraud-detection (by glid) | read | [`BlacklistValuesController.go`](../internal/controllers/UsersControllers/BlacklistValuesController.go) (function `GetBlacklistvalues`), [`UserBlacklistValuesModel.go`](../internal/models/users/UserBlacklistValuesModel.go) |
| ML fraud-suspect read | read | [`ReadMLFraudController.go`](../internal/controllers/UsersControllers/ReadMLFraudController.go), [`UserReadMLFraudModel.go`](../internal/models/users/UserReadMLFraudModel.go) |
| Domain-level blacklist inline check | consumers | [`USER_BUSINESS_FEEDS.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go) (`CheckUserBlocked` function) |
| Chat-derived blacklist signal ingestion | consumers | [`USER_CHAT_BLACKLIST_DATA.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_CHAT_BLACKLIST_DATA.go) (`UserChatLmsBlklist`) |
| **Dead/deprecated write API** | write | [`UserBlacklistController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserBlacklistController.go) — hardcoded `"This API is deprecated now"`, business logic commented out |
| **Orphaned model behind the dead API** | write | [`UserBlacklistModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserBlacklistModel.go) — `UserBlacklistUpdateintoDB`, no live caller anywhere in the 3 repos |
| Write-side validation map | write | `UsersValidationMaps.go:431` (`BlacklistValuesMap` — field-level mandatory/type rules; contents not read in this pass, see Open Questions) |
| API-gateway path config | read | [`Validapi.go`](../pkg/components/Validapi.go) — `/wservce/users/blacklistvalues/`, `/wservce/users/readmlfraud/` entries (lines 28, 30, 116, 125, 177, 224) |

---

## 2. Routes

| Method | Path | Repo | Controller | Status |
|---|---|---|---|---|
| POST | `/user/blacklist` | write | `UserControllers.UserBlacklistController` | **Dead** — always returns hardcoded deprecation message, regardless of input. Still registered live at `users_router/router.go:157,325` (service-api-go-production). |
| POST | `/user/blacklistvalues` | write | `UserControllers.BlacklistValuesController` | **Live** — registered at `users_router/router.go:159,213,327` |
| GET/POST | `/readmlfraud/*params` | read | `UsersControllers.ReadMLFraudController` | **Live** — `internal/api/users_router/routerUsers.go:218-219,500-501` |
| GET/POST | `/blacklistvalues/*params` | read | `UsersControllers.GetBlacklistvalues` | **Live** — `internal/api/users_router/routerUsers.go:353-354,524-525,707-708` |
| GET/POST | `/Blacklistvalues/*params` (capitalized) | read | `UsersControllers.GetBlacklistvalues` | **Live**, but in a **second, separate router file**: `internal/api/router/routerUsers.go:99-100,126-127,161-162,188-189` (registered 4×) |

**Structural oddity worth flagging**: `users-api-go-production` has **two parallel router
files** that both wire up blacklist-related routes — `internal/api/users_router/routerUsers.go`
(has both `readmlfraud` and lowercase `blacklistvalues`) and `internal/api/router/routerUsers.go`
(has only capitalized `Blacklistvalues`, no `readmlfraud` at all). Which file is actually wired
into the live deployed binary was **not traced to a `main.go` entrypoint** in this review —
**[INFERRED — confirm with team]** which router is authoritative; if both are live, `readmlfraud`
may only be reachable through one of the two API surfaces.

---

## 3. Data Model — Tables

> **Verification note**: yeh sab table/column names Go code ke andar embedded SQL strings se
> liye gaye hain (file:line diya hai har row ke saamne). Live DB schema se pgAdmin pe
> cross-verify **nahi** kiya gaya hai — kisi migration/integration se pehle woh zaroor karo.

| Table / Function | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GL_BLACKLIST_VALUES` | meshpg (write), mesh_pg_user (read) | **Manual blacklist** — admin-curated list of flagged attribute values (mobile/email/etc.) | `GL_BLACKLIST_VALUE`, `GL_BLACKLIST_STATUS`, `GL_BLACKLIST_ADD_DATE`, `GL_BLACKLIST_UPDATE_DATE`, `GL_BLACKLIST_UPDATED_BY`, `FK_ORIGNAL_GLUSR_ID`, `COMMENTS`, `FK_ATTRIBUTE_ID` — [`BlacklistValuesModel.go:18-30`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/BlacklistValuesModel.go), [`UserBlacklistValuesModel.go:84-94`](../internal/models/users/UserBlacklistValuesModel.go) |
| `gl_attribute` | meshpg / mesh_pg_user | Master table — attribute-type names (e.g. mobile, email) joined against `GL_BLACKLIST_VALUES.FK_ATTRIBUTE_ID` | `GL_ATTRIBUTE_ID`, `GL_ATTRIBUTE_COLUMN_NAME` — [`UserBlacklistValuesModel.go:84`](../internal/models/users/UserBlacklistValuesModel.go), also joined in [`USER_CHAT_BLACKLIST_DATA.go:91`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_CHAT_BLACKLIST_DATA.go) |
| `fn_sp_suspected_fraud_detection(glid)` | mesh_pg_user, **stored function** | Fraud-check driven by the **glid-mode** of `GET /blacklistvalues` — returns a JSON array, not a table row | Called via [`UserBlacklistValuesModel.go:47`](../internal/models/users/UserBlacklistValuesModel.go). Function body lives in the DB, **not present in any of the 3 Go repos** — see §4 and Open Questions |
| `ML_FRAUD_SUSPECTED_USERS` | buyer_profile_pg_pooler (fallback: buyer_profile_pg) | **ML-model-driven** suspected-fraud accounts | `FK_GLUSR_USR_ID`, `SUSPECT_DATE`, `SUSPECT_COUNTER`, `SUSPECT_STATUS`, `SUSPECT_PROBABILITY`, `GLUSR_USR_CUSTTYPE_ID` — [`UserReadMLFraudModel.go:50`](../internal/models/users/UserReadMLFraudModel.go) |
| `ML_FRAUD_SUSPECT_EXCEPTIONS` | buyer_profile_pg_pooler | Manual exceptions/exemptions, LEFT OUTER JOINed against `ML_FRAUD_SUSPECTED_USERS` | `FK_FRAUD_SUSPECT_EXCEPTION_ID`, `FRAUD_SUSPECT_EXCEPTION`, `EXCEPTION_DATE`, `EXCEPTION_ADDED_BY` — [`UserReadMLFraudModel.go:50`](../internal/models/users/UserReadMLFraudModel.go) |
| `QUERY_APPROVAL_BLACKLIST` | meshPg (consumer-side) | **Domain-level real-time check** — email/mobile/email-domain blacklist, inline-consulted by other features | `IS_DOMAIN`, `BLACKLIST_EMAIL`, `BLACKLIST_COMMENT`, `EMP_NAME`, `EMP_ID`, `IS_BLACKHOLED` — [`USER_BUSINESS_FEEDS.go:318-354`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go), write shape in [`UserBlacklistModel.go:104-110`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserBlacklistModel.go) (dead-code path, but shows the intended insert/delete/update shape) |
| `GLUSR_USR` | meshPg (consumer-side, aliased `BLOCKER`) | Base user record `CheckUserBlocked` reads from (mobile, alt-mobile, email, approval status) | `GLUSR_USR_PH_MOBILE`, `GLUSR_USR_PH_MOBILE_ALT`, `GLUSR_USR_EMAIL`, `GLUSR_USR_APPROV`, `FK_GL_COUNTRY_ISO` — [`USER_BUSINESS_FEEDS.go:314-354`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go) |
| `employee` | meshPg (consumer-side) | Checked to flag employee accounts (excluded from normal feed logic) | `mobile`, `FK_GL_COUNTRY_ISO`, `working`, `DESIGNATION` — [`USER_BUSINESS_FEEDS.go:355-362`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go) |
| `PC_ITEM` | meshPg (consumer-side) | Checked (conditionally) for out-of-stock status on the viewed catalog item | `PC_ITEM_STATUS_APPROVAL` — [`USER_BUSINESS_FEEDS.go:401-411`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go) |

---

## 4. Decode-the-Magic-Values

None of the numeric/status codes below are backed by a named Go const, enum, or DB-side
comment found anywhere in the 3 repos — every meaning is reconstructed from surrounding
context (column names, adjacent literals, error strings). Treat everything here as
**[INFERRED — confirm with team]** unless noted "self-evident."

| Value | Where | Meaning (inferred) | Evidence |
|---|---|---|---|
| `WHITELISTBYGLID == "1"` | `BlacklistValuesController.go:82,117`, `BlacklistValuesModel.go:18` | "Yes, this is a whitelist-override-by-glid request" — switches both the mandatory-field set and the SQL WHERE clause | Param name + branching logic around it |
| `GL_BLACKLIST_STATUS == "1"` (as compared in the consumer) | `USER_CHAT_BLACKLIST_DATA.go:122` | "Active/currently-blacklisted" row — only status `"1"` rows get forwarded to `gladmin_blcklist_service` | Compared as a string; no other value ever checked |
| `GL_ATTRIBUTE_COLUMN_NAME IN ("GLUSR_USR_PH_MOBILE","GLUSR_USR_EMAIL")` | `USER_CHAT_BLACKLIST_DATA.go:122` | Only mobile/email attribute-typed blacklist rows trigger the chat-derived forward; other `gl_attribute` rows are ignored by this consumer | Self-evident from column names; full universe of `gl_attribute` values not enumerated in this review |
| `IS_DOMAIN = 0` | `USER_BUSINESS_FEEDS.go:335` | Email-based blacklist entry | Compared against `GLUSR_USR_EMAIL` |
| `IS_DOMAIN = 1` | `USER_BUSINESS_FEEDS.go:341` | Email-domain-based blacklist entry (whole domain blacklisted, e.g. a spam-mail-service domain) | Compared against the domain-part of the user's email; explicitly excludes `'indiamart.com'` |
| `IS_DOMAIN = 2` | `USER_BUSINESS_FEEDS.go:323,329` | Mobile-based blacklist entry (used for both primary and alternate mobile) | Compared against `GLUSR_USR_PH_MOBILE` and `GLUSR_USR_PH_MOBILE_ALT` |
| `GLUSR_USR_APPROV <> 'A'` | `USER_BUSINESS_FEEDS.go:354` | `'A'` = Approved; anything else means "disabled" for this check | Standard IndiaMART approval-flag convention, not defined in this file |
| `SUBSTRING(GLUSR_USR_PH_MOBILE,1,1) = '1'` (with `FK_GL_COUNTRY_ISO='IN'`) | `USER_BUSINESS_FEEDS.go:349-353` | Indian mobile numbers legitimately start 6-9; a leading `1` signals garbage/test data → flagged `invalid_mobile` | Inferred from Indian mobile numbering convention |
| `working = -1` | `USER_BUSINESS_FEEDS.go:358` | "Active employee" (`-1`/`0` boolean-as-int convention) | Also seen in the `IS_BLACKHOLED` encoding in `UserBlacklistModel.go:50-56` — appears to be a repo-wide convention, not blacklist-specific |
| `DESIGNATION NOT IN ('IB','MANTHAN')` | `USER_BUSINESS_FEEDS.go:359-361` | Specific internal designation codes excluded from the employee-block rule | No comment/definition found anywhere in the 3 repos |
| `PC_ITEM_STATUS_APPROVAL = 9` | `USER_BUSINESS_FEEDS.go:406` | Out-of-stock status code for a catalog item | No enum found anywhere in the 3 repos for this status family |
| `FLAG == "I" / "D" / "U"` | `UserBlacklistModel.go:104,107,110` | Insert / Delete / Update dispatch on `QUERY_APPROVAL_BLACKLIST` | Self-evident (dead-code path, unreachable — see §1) |
| `BLACKHOLE == "0"` → `IS_BLACKHOLED = 0`; else `= -1` | `UserBlacklistModel.go:50-56` | Inverted boolean-as-int encoding for "blackholed" (silently-dropped) values | Dead-code path; same `-1`/`0` convention as `working` above |

---

## 5. Business Rules & Validation

1. **`BlacklistValuesController` (write) accepts only `GLADMIN`/`MAPI` callers** — tightest
   allowlist among all the blacklist-family endpoints. No seller/buyer can touch this API.
   [`BlacklistValuesController.go:104`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/BlacklistValuesController.go)
2. **Write API is UPDATE-only — no INSERT path exists in this file.** Two mandatory-field
   branches: normal update requires `BLACKLIST_VALUE`, `UPDATEDBY`, `ORIGNAL_GLUSR_ID`,
   `COMMENTS`, `VALIDATION_KEY`; whitelist-by-glid (`WHITELISTBYGLID` key present) requires
   `whitelistByGlusrId=="1"` plus `glusrid`, `blacklistStatus`, `comments`.
   [`BlacklistValuesController.go:111-122`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/BlacklistValuesController.go)
3. **Query shape switches on `WHITELISTBYGLID`**: whitelist branch matches
   `WHERE FK_ORIGNAL_GLUSR_ID=$3` (updates every blacklist row tied to that glid, does not
   touch `GL_BLACKLIST_VALUE` or `GL_BLACKLIST_UPDATED_BY`); normal branch matches
   `WHERE UPPER(TRIM(GL_BLACKLIST_VALUE))=$5` (updates by the flagged value itself, case-
   insensitive).
   [`BlacklistValuesModel.go:13-31`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/BlacklistValuesModel.go)
4. **If no row matches, the API reports `"No Record Exist"`, not an error** —
   `RowsAffected()==0` is treated as a distinct, non-fatal outcome (`CODE:"500"` internally,
   but a specific message, not a generic failure).
   [`BlacklistValuesModel.go:90-96`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/BlacklistValuesModel.go)
5. **Case-normalization inconsistency between write and read**: write-side uppercases+trims
   (`UPPER(TRIM(...))`), read-side (`attribute_value` mode) lowercases+trims (`lower(...)`),
   and the chat-ingestion consumer also lowercases. Functionally equivalent (both
   case-insensitive), but the two conventions differ — worth normalizing to one if a third
   path is ever added.
   [`BlacklistValuesModel.go:25`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/BlacklistValuesModel.go), [`UserBlacklistValuesModel.go:88`](../internal/models/users/UserBlacklistValuesModel.go)
6. **`GET /blacklistvalues` supports two mutually-exclusive modes, `glid` takes priority over
   `attribute_value`**: `glid` present → calls stored function `fn_sp_suspected_fraud_detection`;
   else if `attribute_value` present (comma-separated) → direct `GL_BLACKLIST_VALUES` join
   lookup, multiple values at once.
   [`UserBlacklistValuesModel.go:46,83`](../internal/models/users/UserBlacklistValuesModel.go)
7. **`attribute_value` mode silently swallows query errors** — `result, _ := ...` at line 102
   discards the error, unlike `glid` mode which does check it. A DB failure in this mode
   returns an empty/partial result rather than an explicit error.
   [`UserBlacklistValuesModel.go:102`](../internal/models/users/UserBlacklistValuesModel.go)
8. **Output-key casing is inconsistent between the two read models**: `UserBlacklistValuesModel.go`'s
   `createOutputFormat` **uppercases** all keys and turns `nil` into `""`; `UserReadMLFraudModel.go`'s
   `changeOutputFormat` **lowercases** all keys and turns empty-string into `nil` — opposite
   conventions on two sibling "blacklist family" read paths.
   [`UserBlacklistValuesModel.go:119-136`](../internal/models/users/UserBlacklistValuesModel.go), [`UserReadMLFraudModel.go:144-162`](../internal/models/users/UserReadMLFraudModel.go)
9. **ML fraud read supports multi-glid, capped at 10, silently truncated** — if more than 10
   comma-separated glids are supplied, only the first 10 are queried, with no error/warning
   surfaced to the caller.
   [`UserReadMLFraudModel.go:68`](../internal/models/users/UserReadMLFraudModel.go)
10. **ML fraud read builds SQL via raw string concatenation, not parameterized queries** — the
    `glid`-list and optional `datetime` filter are `fmt.Sprintf`'d directly into the query
    string. It is only "safe" because `glid` is pre-validated numeric via `strconv.Atoi`
    before concatenation — a fragile pattern, not a parameterized-query guarantee.
    [`UserReadMLFraudModel.go:54-98`](../internal/models/users/UserReadMLFraudModel.go)
11. **ML fraud read falls back to a second connection pool** (`buyer_profile_pg_pooler` →
    `buyer_profile_pg`) on connect failure — the blacklist-values read path has no equivalent
    fallback (single pool, `mesh_pg_user`), a structural asymmetry between the two "blacklist
    family" reads.
    [`UserReadMLFraudModel.go:31-38`](../internal/models/users/UserReadMLFraudModel.go)
12. **Domain-level inline check (`CheckUserBlocked`) blocks on ANY of 8 independent flags**:
    `is_disabled`, blacklist-email, blacklist-mobile, blacklist-alt-mobile,
    blacklist-email-domain, out-of-stock (conditional), invalid-mobile, is-employee. Any single
    `true` → the whole catalog-view event is dropped (never written to `glusr_usr_biz_feeds`).
    [`USER_BUSINESS_FEEDS.go:433-438`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go)
13. **`indiamart.com` is hardcoded-exempt from the email-domain blacklist check** — even if
    `IS_DOMAIN=1` rows existed matching it, the query explicitly excludes it.
    [`USER_BUSINESS_FEEDS.go:342`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go)
14. **User-not-found is NOT treated as blocked** — if `CheckUserBlocked`'s query returns zero
    rows, it returns an error (`"No Data found"`) and `isBlock=false`; the caller logs the
    error string but does not change flow based on it (i.e. an unknown user is allowed
    through, not blocked-by-default).
    [`USER_BUSINESS_FEEDS.go:421-423`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_BUSINESS_FEEDS.go)
15. **Chat-derived ingestion only acts on active + mobile/email-typed rows, and does not write
    to any local table itself** — it looks up `GL_BLACKLIST_VALUES` to see if the chat message
    contained a known-blacklisted mobile/email, and if `GL_BLACKLIST_STATUS=="1"`, forwards a
    summary to an external service (`gladmin_blcklist_service`) rather than writing DB rows
    directly.
    [`USER_CHAT_BLACKLIST_DATA.go:122-154`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_CHAT_BLACKLIST_DATA.go)
16. **`UserBlacklistController` (dead) always returns HTTP 200** with a canned failure body
    (`STATUS:"FAILURE"`, `CODE:"500"` internally, `MESSAGE:"This API is deprecated now"`),
    regardless of input — not a 4xx/5xx at the transport level, just a deprecation message
    inside a 200 response.
    [`UserBlacklistController.go:139,167-177`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserBlacklistController.go)
17. **No INSERT path for `GL_BLACKLIST_VALUES` was found anywhere in the 3 repos** — only
    UPDATE (write API) and SELECT (both read paths, and the chat-ingestion consumer). How new
    rows get created is an **open question** (see §14/Open Questions).

---

## 6. RabbitMQ

**No RabbitMQ usage was found in any blacklist-related file** across all 3 repos — no
`PushToQueue`/`PubAPI`/direct publish calls in `BlacklistValuesController.go` (either repo),
`BlacklistValuesModel.go`, `UserBlacklistValuesModel.go`, `ReadMLFraudController.go`,
`UserReadMLFraudModel.go`, `UserBlacklistController.go`, or `USER_BUSINESS_FEEDS.go`'s
`CheckUserBlocked`.

The one outbound service call found — `USER_CHAT_BLACKLIST_DATA.go:154`,
`utils.CallWapiService(inputData, "gladmin_blcklist_service", utils.GetConsumerName(),
"application/json")` — is a **WAPI-style call**, not a confirmed RabbitMQ publish.
`CallWapiService`'s internal transport (RabbitMQ RPC vs plain HTTP) was not inspected in this
review — **[INFERRED — confirm with team]** whether this is RabbitMQ-backed.

| Call | Target | Purpose |
|---|---|---|
| `CallWapiService(...)` in `USER_CHAT_BLACKLIST_DATA.go:154` | `"gladmin_blcklist_service"` | Forwards a blacklist-match summary (sender/receiver glid, matched value, attribute name, timestamp) after a chat message contains a known-blacklisted mobile/email. What `gladmin_blcklist_service` does with this payload is outside these 3 repos — **[INFERRED — confirm with team]** |

---

## 7. Kafka

Two of the three consumers are **Kafka-driven** (via `kafka-go`, not RabbitMQ):

| Consumer | Consumer name / registration | Topic | Purpose |
|---|---|---|---|
| `USER_BUSINESS_FEEDS.go` (`UserBusinessFeeds`) | Registered as `USER_BUSINESS_FEED` and `CATALOG_VIEW_BACKFILLING` in `user-temp-consumers-production/internal/Router/router.go:77,110` | Catalog-view events (topic name not captured in this pass) | Runs `CheckUserBlocked` inline before writing a catalog-view row — this IS the domain-level blacklist gate (§5.12) |
| `USER_CHAT_BLACKLIST_DATA.go` (`UserChatLmsBlklist`) | Registered as `USER_CHAT_BLACKLIST_DATA` in `user-temp-consumers-production/internal/Router/router.go:38` | Env-driven (`sub_topic` env var); a stale code comment suggests `lms_email_mob` (`USER_CHAT_BLACKLIST_DATA.go:55`) but this is **not confirmed against live deployment config** | Ingests chat/LMS-derived signals (`sender_id`, `receiver_id`, `mobile[]`/`email[]`) and checks them against `GL_BLACKLIST_VALUES` |

No other consumer in any of the 3 repos references any blacklist-related table.

---

## 8. Redis

**Only one Redis usage exists anywhere in the blacklist-adjacent code, and it is NOT a
blacklist check**: `USER_BUSINESS_FEEDS.go:441-450`, `checkSellerClientMeeting()` —
`redisConn.Exists(ctx, buyerstring)` — a meeting-log bypass check that runs strictly *after*
`CheckUserBlocked`, only if the user was not already blocked.

Redis is **explicitly absent** (confirmed via grep, zero matches) from: both
`BlacklistValuesController.go` files, `BlacklistValuesModel.go`, `UserBlacklistValuesModel.go`,
`ReadMLFraudController.go`, `UserReadMLFraudModel.go`, `UserBlacklistController.go`,
`UserBlacklistModel.go`, `USER_CHAT_BLACKLIST_DATA.go`.

If read latency ever becomes a concern on the read endpoints (`GET /blacklistvalues`,
`GET /readmlfraud`), there is no existing cache to lean on — see §11 Optimization Scope.

---

## 9. End-to-End Technical Flows

### Flow A — Admin updates a blacklist value

```
Admin (GLADMIN/MAPI)
    │
    ▼
[API — write]  POST /user/blacklistvalues
    │  BlacklistValuesController.go
    │  1. Gateway validation (GLADMIN/MAPI only)
    │  2. Mandatory-field check — branches on WHITELISTBYGLID key
    ▼
[DB]  UPDATE GL_BLACKLIST_VALUES
    │  WHERE FK_ORIGNAL_GLUSR_ID=$3 (whitelist branch)
    │  or  WHERE UPPER(TRIM(GL_BLACKLIST_VALUE))=$5 (normal branch)
    ▼
Response: "Data Updated Successfully" (rows affected > 0)
          or "No Record Exist" (rows affected == 0 — no INSERT fallback)
```

### Flow B — Read blacklist values / stored-function fraud-check

```
Caller (internal tool/admin)
    │
    ▼
[API — read]  GET /blacklistvalues
    │
    ├─ glid param →
    │     [DB — stored function]  SELECT fn_sp_suspected_fraud_detection($1)
    │     └─ JSON-array result parsed, 100ms timeout
    │
    └─ attribute_value param (comma-separated) →
          [DB]  SELECT b.*, a.GL_ATTRIBUTE_COLUMN_NAME
                FROM GL_BLACKLIST_VALUES b JOIN gl_attribute a ...
                WHERE lower(b.GL_BLACKLIST_VALUE) IN (...)
          (errors from this branch are silently discarded, §5.7)
```

### Flow C — Chat-derived blacklist signal ingestion (Kafka)

```
Chat/LMS system (Kafka producer, topic likely "lms_email_mob" — unconfirmed)
    │
    ▼
[Kafka consume]  USER_CHAT_BLACKLIST_DATA.go → UserChatLmsBlklist
    │  message: sender_id, receiver_id, mobile[]/email[] (at least one required)
    ▼
[DB]  SELECT b.*, a.GL_ATTRIBUTE_COLUMN_NAME
      FROM GL_BLACKLIST_VALUES b JOIN gl_attribute a ...
      WHERE lower(b.GL_BLACKLIST_VALUE) IN (mobiles ∪ emails from message)
    │
    ├─ match found, GL_BLACKLIST_STATUS=="1", column is mobile/email →
    │     [WAPI call]  CallWapiService(..., "gladmin_blcklist_service", ...)
    │     (forwards sender/receiver glid, matched value, attribute name, timestamp)
    │
    └─ no match / status != "1" → "Nothing to update", no further action
```

### Flow D — Domain-level real-time check (inline, no dedicated API)

```
USER_BUSINESS_FEEDS Kafka consumer (catalog-view event)
    │  glusr_id != catalog_owner_glusr_id (viewer ≠ seller)
    ▼
CheckUserBlocked(meshPg, glusrid, sellerid, displayId)  — synchronous inline call
    │
    ▼
[DB — single query, 1s timeout]  GLUSR_USR (BLOCKER) LEFT-correlated against
      QUERY_APPROVAL_BLACKLIST (×4 subqueries: mobile, alt-mobile, email, email-domain)
      + employee join + conditional PC_ITEM out-of-stock check
    │
    ▼
isBlock = ANY(is_disabled, blkst_email, blkst_mob, blkst_mobalt,
              blkst_emaildomain, out_of_stock, invalid_mob, is_employee)
    │
    ├─ isBlock == true  → catalog-view row is NOT written (silently dropped)
    └─ isBlock == false → proceeds to checkSellerClientMeeting() (Redis) → normal insert
```

### Flow E — ML Fraud Suspect read

```
Caller
    │
    ▼
[API — read]  GET /readmlfraud
    │  ReadMLFraudController.go → UserReadMLFraudModel.go
    ▼
[DB — buyer_profile_pg_pooler, fallback buyer_profile_pg, 100ms timeout]
      SELECT ... FROM ML_FRAUD_SUSPECTED_USERS
      LEFT OUTER JOIN ML_FRAUD_SUSPECT_EXCEPTIONS ON ...
      WHERE FK_GLUSR_USR_ID = <id> (or IN (<up to 10 ids>))
      [optional AND date(SUSPECT_DATE) = '<datetime>']
    ▼
Response: suspect status, probability score, exception details (if any)
```

### Flow F — Dead write API call (for completeness)

```
Any caller → POST /user/blacklist
    │
    ▼
UserBlacklistController.go — all real logic commented out
    ▼
Always: HTTP 200, body {"STATUS":"FAILURE","MESSAGE":"This API is deprecated now",
                         "CODE":"500","SERVICE_NAME":"BLACKLIST_SERVICE","RESPONSE_DATA":{}}
```

---

## 10. Flow-wise DB & Table Usage

### Flow A — Blacklist-value update (1 DB, 1 write)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshpg | `GL_BLACKLIST_VALUES` | UPDATE | Status/value update on an existing row — matched either by value or by glid, depending on `WHITELISTBYGLID` |

### Flow B — Read (2 modes, mutually exclusive per request)

| # | DB | Table/Function | Operation | Kyun |
|---|---|---|---|---|
| 1a | mesh_pg_user | `fn_sp_suspected_fraud_detection` | SELECT (stored function call) | glid-mode: fraud-check result as JSON |
| 1b | mesh_pg_user | `GL_BLACKLIST_VALUES` JOIN `gl_attribute` | SELECT | attribute_value-mode: direct lookup, case-insensitive, multi-value |

### Flow C — Chat-derived ingestion (1 DB read, 1 external forward)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshPgLB | `GL_BLACKLIST_VALUES` JOIN `gl_attribute` | SELECT | Check if chat-shared mobile/email is a known blacklisted value |
| 2 | *(none — external call)* | `gladmin_blcklist_service` via `CallWapiService` | — | Forward match summary; does not write to any local table |

### Flow D — Domain-level inline check (heaviest single-query flow)

| # | DB | Table(s) | Operation | Kyun |
|---|---|---|---|---|
| 1 | meshPg | `GLUSR_USR` + `QUERY_APPROVAL_BLACKLIST` (×4 correlated subqueries) + `employee` + conditional `PC_ITEM` | SELECT (single composite query, 1s timeout) | Computes 8 independent block-reason flags in one round-trip — this is deliberately consolidated into a single query rather than 8 separate round-trips |
| 2 | bizfeedRedis | *(not a blacklist table — meeting-log key)* | EXISTS | Only reached if isBlock==false; unrelated bypass check, included here to show it sits right after the blacklist gate in the same consumer |

**Note**: unlike GST's domain, this flow is already well-consolidated — 8 flags in 1 query is
efficient, not a spread-out anti-pattern.

### Flow E — ML Fraud read

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | buyer_profile_pg_pooler (fallback: buyer_profile_pg) | `ML_FRAUD_SUSPECTED_USERS` LEFT JOIN `ML_FRAUD_SUSPECT_EXCEPTIONS` | SELECT | Suspect status + probability + any manual exception, for up to 10 glids at once |

### Flow F — Dead write API

No DB round-trips — the controller returns a hardcoded response before any DB call.

---

## 11. Optimization Scope

### High-impact / correctness risk

1. **Two parallel fraud-check paths with unclear relationship**: `GET /blacklistvalues?glid=`
   calls `fn_sp_suspected_fraud_detection` (mesh_pg_user), while `GET /readmlfraud` reads
   `ML_FRAUD_SUSPECTED_USERS` directly (buyer_profile_pg_pooler) — different connection pools,
   different tables/functions, both answering "is this glid fraud-suspect." If they're meant
   to be the same underlying signal, this is a **two-sources-of-truth risk**; if genuinely
   different, the naming makes them easy to confuse. **The stored function's body is not in
   any of the 3 repos**, so this cannot be resolved from code — needs a DB-side lookup or a
   conversation with the team that owns `fn_sp_suspected_fraud_detection`.
2. **`GET /blacklistvalues` (`attribute_value` mode) silently discards query errors**
   (`result, _ := ...`, `UserBlacklistValuesModel.go:102`) — a DB failure returns an empty/
   partial result set indistinguishable from "no matches," which could mask real outages
   from callers relying on this for a fraud-safety decision. **Fix**: propagate the error the
   same way the `glid` mode already does.
3. **`ML_FRAUD` read query is unparameterized string concatenation** (`UserReadMLFraudModel.go:54-98`)
   — safe today only because glid values are pre-validated via `strconv.Atoi`, but any future
   edit that loosens that validation reopens a SQL-injection surface. **Fix**: convert to
   parameterized placeholders.

### Medium-impact

4. **No caching on any read endpoint** — `GET /blacklistvalues` and `GET /readmlfraud` hit
   Postgres on every call. Blacklist/fraud-suspect status for a given glid/value changes
   relatively infrequently; a short-TTL Redis cache-aside layer (Redis infra is already used
   elsewhere in this same consumer repo, §8) is a natural fit and would reduce read latency
   without much correctness risk.
5. **Case-normalization inconsistency** (write `UPPER()`, read `lower()`, §5.5) — functionally
   harmless today, but worth converging on one convention so a future third write/read path
   doesn't introduce a silent mismatch.
6. **Multi-glid cap of 10 in ML fraud read is silent** (`UserReadMLFraudModel.go:68`) — a
   caller passing 15 glids gets results for only the first 10 with no indication of
   truncation. **Fix**: either raise the cap or return an explicit warning/count in the
   response.
7. **Output-key casing asymmetry between the two sibling read models** (§5.8) — not a
   performance issue, but a maintenance/consumer-integration risk: any client that
   generalizes handling across `GET /blacklistvalues` and `GET /readmlfraud` responses will
   hit inconsistent key casing (`UPPERCASE` vs `lowercase`) and null-vs-empty-string
   conventions.

### Low-impact / dead-code cleanup

8. **`UserBlacklistController` + `UserBlacklistModel.go` are fully dead** but still consume a
   live route (`POST /user/blacklist`, registered twice in the router) and still get compiled/
   deployed. Safe to remove the controller body and the orphaned model, or at minimum remove
   the route registration, once confirmed no external client depends on receiving the
   specific "deprecated" message.
9. **`GL_BLACKLIST_VALUES` has no confirmed INSERT path anywhere in the 3 repos** — not
   strictly a performance issue, but worth investigating: if new rows are created by a manual
   DB script or an out-of-repo process, that's an operational gap (no audit trail in Go code)
   worth documenting or replacing with a proper API.

---

## 12. Cron Inventory

Checked all cron directories in the 3 repos (only `service-api-go-production/service-api-go-production/crons/`
has cron files — `users/`, `recommend/`, `ML_Retail/` subfolders). Specifically checked
`rating_suspect_cron.go` (name overlap with "suspect") — **zero blacklist references found**.

**No cron file in any of the 3 repos references `GL_BLACKLIST_VALUES`, `ML_FRAUD_SUSPECTED_USERS`,
`ML_FRAUD_SUSPECT_EXCEPTIONS`, or `QUERY_APPROVAL_BLACKLIST`.** Unlike GST (which has a daily
BigQuery-driven re-verification cron), **the Blacklist domain has no scheduled/cron job
touching it anywhere in this codebase** — confirmed absent, not assumed.

---

## 13. Edge Cases & Gotchas (technical POV)

1. **`GL_BLACKLIST_VALUES` growth path is unclear** — no INSERT was found in application code
   across all 3 repos; only UPDATE (write API) and SELECT (both read paths + chat consumer).
   If debugging "where did this blacklist entry come from," start by checking for an
   out-of-repo DB script or a process outside these 3 repos.
2. **`fn_sp_suspected_fraud_detection` vs `ML_FRAUD_SUSPECTED_USERS` relationship is
   unconfirmed** (§11.1) — the single most important open question in this domain.
3. **Two parallel router files in `users-api-go-production`** register overlapping-but-not-
   identical sets of blacklist routes (§2) — if `readmlfraud` behaves unexpectedly in one
   environment vs another, check which router file that environment's binary actually uses.
4. **`USER_CHAT_BLACKLIST_DATA`'s exact Kafka topic is not confirmed** — only an env var
   (`sub_topic`) plus a stale code comment (`//"lms_email_mob"`). Don't assume the comment
   reflects the live deployed topic name without checking the actual env config.
5. **Dead `UserBlacklistController` still returns HTTP 200** — any old integration calling
   `POST /user/blacklist` gets a 200 with a "deprecated" message inside the body, not an
   HTTP-level error, so naive status-code-only monitoring would not catch this as a failure.
6. **User-not-found in `CheckUserBlocked` means "not blocked," not "blocked by default"**
   (§5.14) — a missing/deleted user's catalog-view events pass through the blacklist gate
   rather than being dropped.
7. **`GET /blacklistvalues` attribute_value-mode swallows DB errors silently** (§5.7,
   §11.2) — a genuine outage in this path looks identical to "no matches found" from the
   caller's perspective.
8. **Employee-exclusion designation codes (`'IB'`, `'MANTHAN'`) are unexplained** in code —
   if the employee-block-bypass logic ever needs to change, whoever touches it will need to
   go find out what these designation codes mean outside the codebase.

---

## 14. Open Questions

1. Where/how are new rows inserted into `GL_BLACKLIST_VALUES`? No `INSERT INTO
   GL_BLACKLIST_VALUES` was found in application code in this review.
2. Does `fn_sp_suspected_fraud_detection` (called by `GET /blacklistvalues?glid=`) read from
   `ML_FRAUD_SUSPECTED_USERS`, or a separate source? The function body is not present in any
   of the 3 Go repos (it lives DB-side).
3. `BlacklistValuesMap` (`UsersValidationMaps.go:431`) field-level validation rules were not
   read in this pass — confirm exact mandatory/type/length constraints before relying on this
   doc for API-contract questions.
4. Which of the two parallel router files (`internal/api/users_router/routerUsers.go` vs
   `internal/api/router/routerUsers.go`) in `users-api-go-production` is actually wired into
   the live production binary — or are both live simultaneously?
5. `CallWapiService`'s transport (RabbitMQ-backed RPC vs plain HTTP) was not inspected — this
   affects whether §6 (RabbitMQ) should list `gladmin_blcklist_service` as a queue or not.
6. Exact Kafka topic name for `USER_CHAT_BLACKLIST_DATA` — confirm against live deployment
   config rather than the stale in-code comment.
7. Meaning of every INFERRED literal in §4 (validation optcase numeric codes, `SUSPECT_STATUS`
   values, `PC_ITEM_STATUS_APPROVAL=9`, `working=-1`, `DESIGNATION IN ('IB','MANTHAN')`) — none
   are backed by named constants/enums/comments anywhere in the 3 repos.
8. Several grep-matched files (`UserUtilsMandatory.go`, `UserBuyerProfileModel.go`, and a few
   `StandardProductsController`/product-listing files) were not opened to confirm whether they
   contain real blacklist logic or are incidental keyword matches — worth a follow-up pass if
   this doc is used to plan a migration/refactor.
9. Live DB schema verification (column types, nullability, indexes, constraints) for every
   table in §3 — this doc only reflects what Go SQL strings imply.
10. What does `gladmin_blcklist_service` (the external target of the chat-derived forward,
    §6) do with the payload it receives? Out of scope of these 3 repos.

---

## See also

- [`Blacklist_Business_Doc.md`](./Blacklist_Business_Doc.md) — same flows, product/business
  perspective, bina code ke
- [`../trust_verification_compliance_read_write_picture.md`](../trust_verification_compliance_read_write_picture.md) —
  bigger Trust & Compliance technical trace, jahan `UserBlacklistController` dead-code-finding
  pehle bhi flag hui thi
- [`../utils_programming_guide.md`](../utils_programming_guide.md) — shared Kafka/consumer/
  Redis patterns jo iss doc mein reference hue hain
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — same-depth reference
  doc for the GST domain, in the same trust/verification family

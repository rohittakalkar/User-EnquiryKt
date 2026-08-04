# Privacy Setting — Technical Doc (Code-Level Deep Dive)

Yeh doc Privacy Setting feature ka **technical implementation** cover karta hai — APIs, DB
tables, queries, RabbitMQ, consumers, sab code se verify karke. Business/product perspective
ke liye [`Privacy_Setting_Business_Doc.md`](./Privacy_Setting_Business_Doc.md) dekho.

**Repos**: `users-api-go-production` (read), `service-api-go-production` (write),
`user-temp-consumers-production` (consumers).

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file path diya
gaya hai har jagah). Jahan code se pura confirm nahi ho paaya, wahan
**[INFERRED — team se confirm karo]** likha hai.

**Note on depth of this pass**: yeh doc pehle se relatively deep tha. Iss pass mein sabse
badi cheez jo naya mila woh hai **read-side (`GET /setting`) ki asli complexity** — purana
doc isse ek "simple SELECT" jaisa treat kar raha tha, jabki asal mein yeh teen alag,
independently-evolved code-paths hain (section 2, 8-Flow-E/F). Yeh correction hai, downgrade
nahi — purani findings sahi thi jahan tak gayi, bas poori tasveer nahi thi.

---

## 1. Privacy Setting Kahan-Kahan Hai — File Map

| Concern | Repo | File |
|---|---|---|
| Privacy setting submit/update (general detail-editor ke through) | write | [`UserDetailsController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDetailsController.go), [`UserDetailsModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go) |
| Validation rules/field map | write | [`UsersValidationMaps.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) (`UserDetails_PrivSetting` map) |
| Hardcoded priv_id arrays (write side) | write | [`commonMaps.go`](../../service-api-go-production/service-api-go-production/pkg/components/commonMaps.go) — `DetailsAPIautomatedIsEnableArray`, `DetailsAPIdisableDefaultArray` |
| Setting read/write bridge, "old" path (Android ≤ 12.3.2, or `version` query param absent) | read | [`SettingController.go`](../internal/controllers/UsersControllers/SettingController.go), [`UserSettingModel.go`](../internal/models/users/UserSettingModel.go) — functions `GetUserPrivacySettingsModel` (v1-style) and `GetUserPrivacySettingsModelV2` |
| Setting read/write bridge, "v2" path (query param `version=v2`) | read | [`SettingController_v1.go`](../internal/controllers/UsersControllers/SettingController_v1.go), [`UserSettingModel_v1.go`](../internal/models/users/UserSettingModel_v1.go) — function `GetUserPrivSetting` |
| Master privacy-setting catalog (file-based, not DB) + startup cache | read | [`utils.go:3181-3227`](../pkg/utils/utils.go) (`ReadMasterDataPrivSetting`, `ReadPrivSettDefBehavior`, `ReadPrivSettDefBehaviorPgx`), [`configuration.go:190-199`](../pkg/config/configuration.go) (`PrivSettingMasterData`, `PrivSettingDefBehavior` package-level caches) |
| UK-token alternate auth (obfuscated link-based access) | read | Duplicated in both [`UserSettingModel.go`](../internal/models/users/UserSettingModel.go) (`CheckUK`/`UKDecryption`) and [`UserSettingModel_v1.go`](../internal/models/users/UserSettingModel_v1.go) (`CheckUK_v1`/reuses `UKDecryption`) |
| Change fan-out — search/IMSDB database | consumers | [`USER_PRIVACYSETTING_IMSDB.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_IMSDB.go) |
| Change fan-out — auth DB (denormalized flag) + alert DB (email unsubscribe) | consumers | [`USER_PRIVACYSETTING_ALERTPG.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_ALERTPG.go) |

**Related but separate feature**: `PnsSettingControllers` (PNS/call-routing off-hours
preferences, routes `/pnssetting/*params`) is a **different** feature that also uses the word
"setting" — don't confuse it with Privacy Setting. It's covered under the "Notifications,
Alerts & Preferences" product story, not this doc.

**No cron found for this domain** — confirmed by grepping both `user-temp-consumers-production`
and `service-api-go-production/crons` for anything Privacy-Setting related; only GST-domain
crons exist there (section 12).

---

## 2. Routes (confirmed from router.go) — more layered than it looks

| Method | Path | Repo | Controller |
|---|---|---|---|
| POST | `/details` (type=PrivSetting) | write | `UserDetailsController` |
| GET/POST | `/setting/*params` | read | `SettingController` |

**`/setting` is not one simple read** — inside `SettingController.go`, three distinct code
paths exist depending on query params, and two of the three actually **write** by proxying
to the write API:

```
GET/POST /setting
    │
    ├─ query param version == "v2"?
    │    └─ YES → SettingController_v1() → GetUserPrivSetting()  [`SettingController.go:62-63`]
    │
    └─ NO → inline in SettingController, branch on app_version_no vs "12.3.2":
             ├─ Android AND app_version_no <= 12.3.2 → GetUserPrivacySettingsModel()   (older, JSON-file-driven)
             └─ everything else                       → GetUserPrivacySettingsModelV2() (newer, adds UK-auth + sub-settings)
             [`SettingController.go:132-138`]
```

Inside **all three** of these functions (`GetUserPrivacySettingsModel`,
`GetUserPrivacySettingsModelV2`, `GetUserPrivSetting`), the request's `flag` param decides
what actually happens — and only `flag=="2"` is a genuine read:

| `flag` value | What actually happens |
|---|---|
| `"0"` | **Not a DB read** — builds a `PrivSetting` payload (`NEW_VAL=Enabled`) and does an internal HTTP `POST` (`utils.CurlReqWithRetry`) to the **write API's** `/details` endpoint (`detailwapi` config URL). Effectively "enable this setting" via a GET-shaped façade. [`UserSettingModel.go:66`](../internal/models/users/UserSettingModel.go) |
| `"1"` | Same, but `flag_del="D"` — "disable this setting", again via internal curl to write API. |
| `"2"` | The actual read: queries `GLUSR_USR_PRIVACY_SETTING` directly, merges with the file-based master catalog, returns `Disable_Setting_ids`. |
| anything else | `400`/`INVALID_FLAG` — rejected before reaching any of the above. [`SettingController.go:111-114`] |

**Why this matters**: a caller hitting `GET /setting?flag=0` is not "reading" anything — it's
triggering a write (an outbound HTTP call to another controller in the same domain), inside
what looks like a read-only GET endpoint. Anyone debugging "why did my read call change data"
should check this first.

---

## 3. Data Model — Tables

> **Verification note**: table/column names Go code ke embedded SQL strings se liye gaye
> hain. Live DB schema se cross-verify nahi kiya gaya — kisi migration se pehle woh zaroor
> karo.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_USR_PRIVACY_SETTING` | Primary write DB (`dbConn` in write API — same connection `UserDetailsModel` uses for other detail-types) + replicated to searchPg (IMSDB) + read via meshPg on the read API | **Primary record** — har supplier ke har setting ka current state | `FK_GLUSR_USR_ID`, `FK_MY_PRIVACY_SETTING_ID`, `GLUSR_PRIVACY_UPDATEDBY_FLAG`, `GLUSR_PRIVACY_UPDATEDBY_ID`, `GLUSR_PRIVACY_UPDATEDBY`, `GLUSR_PRIVACY_UPDATESCREEN`, `GLUSR_PRIVACY_IP`, `GLUSR_PRIVACY_IP_COUNTRY`, `GLUSR_PRIVACY_HIST_COMMENTS`, `GLUSR_PRIVACY_SETTING_REMARKS`, `GLUSR_PRIVACY_IS_ENABLE`, `GLUSR_PRIVACY_IS_AUTOMATED`, `GLUSR_USR_PRIV_MODIFIED_DATE` — [`UserDetailsModel.go:2512,2691,2886`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go), [`USER_PRIVACYSETTING_IMSDB.go:119,136,176`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_IMSDB.go), read-side [`UserSettingModel.go:165,719`](../internal/models/users/UserSettingModel.go), [`UserSettingModel_v1.go:198`](../internal/models/users/UserSettingModel_v1.go) |
| `MY_PRIVACY_SETTING` | Same write DB (also queried directly, read-only, from the read API) | **Master/catalog table** — har setting-type ka definition, incl. default behavior | `MY_PRIVACY_SETTING_ID`, `MY_PRIVACY_DEFAULT_BEHAVIOR` (0/1 — decode in section 5) — [`UserDetailsModel.go:2532`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go), [`utils.go:3196,3231`](../pkg/utils/utils.go) (`ReadPrivSettDefBehaviorPgx`/`ReadPrivSettDefBehavior`, read-side) |
| `GLUSR_USR` | authPg | Specific privacy IDs (`45`, `90`, `20`, `153`, `175` only) ke liye, ek **denormalized comma-separated list** column maintain karta hai — quick lookup ke liye, bina main table join kiye | `FK_MY_PRIVACY_SETTING_IDS` (comma-separated string of enabled privacy-setting IDs) — [`USER_PRIVACYSETTING_ALERTPG.go:140,167,196`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_ALERTPG.go) |
| `GLUSR_EMAIL_UNSUBSCRIBERS` | alertPg | Compliance-critical — kaun-kaunse supplier ne kaunsi email-category se unsubscribe kiya | `FK_GLUSR_USR_ID`, `FK_MY_PRIVACY_SETTING_ID`, `FK_IIL_PROCESS_MASTER_ID`, `EMAIL_UNSUBSCRIBE_REMARKS`, `GLUSR_EMAIL_UNSUBSCRIBE_DATE` — [`USER_PRIVACYSETTING_ALERTPG.go:250-292`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_ALERTPG.go) |
| `glusr_seller_assistant_details` (**new, not in previous pass of this doc**) | meshPg-equivalent (read API's `mesh_pg_user` connection) | Extra JSON blob per (GLID, priv_id) — attached to privacy-setting responses **only** for `priv_id` `1250`/`1248`/`1246` | `fk_glusr_usr_id`, `fk_glusr_privacy_setting_id`, `setting_details` (JSON string), `last_modified_date` (used to pick latest row via `DISTINCT ON`) — [`UserSettingModel.go:1250-1256`](../internal/models/users/UserSettingModel.go), [`UserSettingModel_v1.go:673-678`](../internal/models/users/UserSettingModel_v1.go). **Business meaning [INFERRED — confirm with team]**: `1250`/`1248`/`1246` decode to `seller_chat_ai_active` / `seller_vani_active` / `seller_vani_all_call_active` based on a fallback-key list seen in `GetUserPrivSetting` (`UserSettingModel_v1.go:502-513`) — this strongly suggests these three IDs are "Seller Assistant"/AI-calling-related settings, but a definitive mapping wasn't found in code. |
| History (stored procedure, table name explicit nahi mila) | Likely mainPg/Oracle | Audit-trail — "Privacy Setting Change" comment ke saath | `CALL pck_gl_history_sp_add_custom_history(...)` — [`UserDetailsModel.go:2759,2856,2925`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go) |

---

## 4. Decode This — Magic `priv_id` Values & Hardcoded Arrays

Privacy Setting ka poora behavior bahut saari **hardcoded `priv_id` arrays** pe depend karta
hai, spread across both write and read code. Yeh sabse zyada "trap" hai iss domain mein —
neeche ek jagah sab consolidate kiya hai.

### 4.1 Write-side arrays (`service-api-go-production/pkg/components/commonMaps.go:112-113`)

| Array | Values | Meaning |
|---|---|---|
| `DetailsAPIautomatedIsEnableArray` | `{3, 19, 62, 51}` | In-flight priv_ids ke liye hi `is_enable`/`is_automated` columns set hote hain (section 5, point 1). Baaki sab ke liye yeh dono fields blank force ho jaate hain. |
| `DetailsAPIdisableDefaultArray` | `{76, 91, 81}` | In IDs ka "default" state normal se **ulta** hai — combined with `flag_del`, controls a dedupe-check branch in `UserDetailsController.go:747` (section 5, point 10). |
| `DetailsAPIIMGlids` | `{"13035342"}` | Not Privacy-Setting-specific — this is the GST test-bypass allowlist (see GST tech doc), shows up in the same file, easy to confuse. |

### 4.2 Inverted-action priv_ids (Controller **and** Model, `UserDetailsController.go:183-193,331-340`, `UserDetailsModel.go:2517`)

For `priv_id == "3"` **or** `priv_id == "81"` specifically, `flag_del == "D"` means **Insert**
and non-`"D"` means **Update** — the reverse of every other priv_id (where `id==""` decides
Insert vs Update). The exact in-code comment is:

```go
// Inverting the action for setting id's 3, 53      [UserDetailsModel.go:2517]
```

**Correction vs. previous pass of this doc**: the comment literally says "3, 53", but the
actual `if` conditions checked in code only test for `"3"` and `"81"` (`UserDetailsController.go:183,331,337`)
— `53` never appears in any inversion condition found in this review. Either the comment is
stale (53 used to be inverted and isn't anymore), or 53's inversion happens somewhere not
found in this pass. **[INFERRED — confirm with team which is true]**.

### 4.3 The real "default behavior" state machine (`UserDetailsModel.go:2666-2687`, `subcat==""` + `setting_status` request shape)

This is the actual, fully-code-confirmed decode of "inverted logic" — when a request carries
`setting_status` (`"enabled"`/`"disabled"`), the action taken is a 3-input truth table:

| `setting_status` | `MY_PRIVACY_DEFAULT_BEHAVIOR` (from `MY_PRIVACY_SETTING`) | Row already exists? | Action |
|---|---|---|---|
| enabled | 0 (default=off) | No | no action |
| enabled | 0 (default=off) | Yes | **delete** |
| enabled | 1 (default=on) | No | **insert** |
| enabled | 1 (default=on) | Yes | no action |
| disabled | 0 (default=off) | No | **insert** |
| disabled | 0 (default=off) | Yes | no action |
| disabled | 1 (default=on) | No | no action |
| disabled | 1 (default=on) | Yes | **delete** |

In plain terms: a row in `GLUSR_USR_PRIVACY_SETTING` only ever exists to record the
**non-default** state. If a setting defaults to "on" for everyone, a row is only created when
someone explicitly turns it **off**; if it defaults to "off", a row is only created when
someone turns it **on**. This is why "Insert = enabled" for one setting and "Insert =
disabled" for another isn't a bug — it depends entirely on that setting's row in
`MY_PRIVACY_SETTING.MY_PRIVACY_DEFAULT_BEHAVIOR`.

### 4.4 Read-side default/enabled arrays (three independently-hardcoded, **inconsistent**, copies)

Each of the three read functions (`GetUserPrivacySettingsModel`, `GetUserPrivacySettingsModelV2`,
`GetUserPrivSetting`) computes a **UI-facing** "is this checked by default" flag using its own
hardcoded array — separate from, and **not always consistent with**, the write-side default
behavior in 4.3:

| Function | "Always shows enabled" array | "Default-disabled, flips to shown-enabled" array |
|---|---|---|
| `GetUserPrivacySettingsModel` (old/v1-style) | `{"3","19","62"}` | `{"47","53","69","70"}` — [`UserSettingModel.go:242-243`] |
| `GetUserPrivacySettingsModelV2` | `{"19","62","51"}` (only if `GLUSR_PRIVACY_IS_ENABLE==0`) | `{"76","91","153","83","84","3","53","81"}` combined with `{"47","69","70"}` — [`UserSettingModel.go:792-793,828`] |
| `GetUserPrivSetting` (the true `version=v2` path) | *(none — uses live `MY_PRIVACY_DEFAULT_BEHAVIOR` from DB via `ReadPrivSettDefBehavior`, not a hardcoded array)* | — [`UserSettingModel_v1.go:372-406`] |

**This is a genuine inconsistency, not just three views of the same thing**: `53` moved from
`GetUserPrivacySettingsModel`'s "disabled" bucket into `GetUserPrivacySettingsModelV2`'s
"always enabled" bucket; `81` and `3` appear in `GetUserPrivacySettingsModelV2`'s array but not
in the older one's. Only `GetUserPrivSetting` avoids the whole problem by reading
`MY_PRIVACY_SETTING` live instead of hardcoding a parallel copy. **[INFERRED — confirm with
team]** whether the two older functions are legacy/frozen (kept only for old app versions) or
still actively need to track master-table changes — if the latter, they will silently drift
out of sync with `MY_PRIVACY_SETTING` every time a new setting is added.

### 4.5 Denormalized-flag priv_ids (`USER_PRIVACYSETTING_ALERTPG.go:133`)

`{45, 90, 20, 153, 175}` — see section 3. Note `153` also appears independently in the
read-side "default disabled" array 4.4 (`GetUserPrivacySettingsModelV2`) — these are two
unrelated mechanisms that happen to share an ID; don't assume a connection without
confirming.

### 4.6 Android-only priv_ids (`UserSettingModel.go:174`, `UserSettingModel_v1.go:260`)

`{42, 43, 44, 57}` — when `mod_id=="ANDROID"` and app version is old, the response is
filtered down to **only** these IDs, in a stripped `{FK_MY_PRIVACY_SETTING_ID}`-only shape
(no enable/disable value) — an old-Android-app compatibility carve-out.

### 4.7 Seller-Assistant sub-setting priv_ids (`UserSettingModel.go:427,446`, `UserSettingModel_v1.go:444,460`)

`{1250, 1248, 1246}` — only these three get the `glusr_seller_assistant_details` JSON blob
attached (section 3).

### 4.8 Night Notification (`priv_id 1243`) — proposed, not yet live

See [`Task 1 - Night Notification Manual vs Automated Flag.md`](./Task%201%20-%20Night%20Notification%20Manual%20vs%20Automated%20Flag.md)
for a full write-up. Short version: `1243` is **not currently in** `DetailsAPIautomatedIsEnableArray`
(4.1), so `is_enable`/`is_automated` are always blanked for it today — a proposed change would
add it, but as of this review that change is **not implemented** in `commonMaps.go`.

---

## 5. Business Rules & Validation (code se)

1. **Setting-type-specific fields**: `priv_id` (`FK_MY_PRIVACY_SETTING_ID`) decide karta hai
   konse extra columns (`is_enable`, `is_automated`) apply hote hain — sirf
   `DetailsAPIautomatedIsEnableArray` (section 4.1) ke members ke liye.
   [`UserDetailsController.go:214-235`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDetailsController.go)
2. **"Inverted action" for priv_ids 3 and 81**: see section 4.2 for the exact mechanics and a
   correction vs. the previous version of this doc.
3. **`mail_frequency` sub-category bypasses the entire normal write path**: agar
   `Type == "PrivSetting"` AUR `subCat == "mail_frequency"`, koi DB write hoti hi nahi
   (`UserDetailUpdateDB` call hi skip ho jaata hai) — sirf seedha ek RabbitMQ message push
   hota hai `SERVICENAME=MAIL_FREQUENCY_ALERT` ke saath.
   [`UserDetailsController.go:240-261`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDetailsController.go)
4. **`sub_setting` param sirf `PrivSetting` type ke liye allowed hai** — kisi aur detail-type
   ke saath use karne pe explicit error: `"sub_setting is only allowed for type PrivSetting"`.
   [`UserDetailsModel.go:2827-2830`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)
5. **History logging conditional hai**: non-INSERT operations ke liye ek alag goroutine
   `pck_gl_history_sp_add_custom_history` stored procedure call karti hai audit ke liye —
   `'Privacy Setting Change'` comment ke saath hardcoded.
   [`UserDetailsModel.go:2759,2856,2925`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)
6. **Denormalized `FK_MY_PRIVACY_SETTING_IDS` sirf 5 specific setting IDs ke liye maintain
   hota hai**: `45`, `90`, `20`, `153`, `175` (section 4.5).
   [`USER_PRIVACYSETTING_ALERTPG.go:133`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_ALERTPG.go)
7. **Email-unsubscribe sirf specific condition pe trigger hota hai**: `NEW_VAL == "Disabled"`
   AUR `SOURCE_TYPE` non-empty ho, dono saath — agar `SOURCE_TYPE` missing hai, unsubscribe
   list mein add nahi hoga.
   [`USER_PRIVACYSETTING_ALERTPG.go:265`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_ALERTPG.go)
8. **`NEW_VAL == "Enabled"` reverse karta hai unsubscribe ko** — matching row
   `GLUSR_EMAIL_UNSUBSCRIBERS` se DELETE ho jaata hai.
   [`USER_PRIVACYSETTING_ALERTPG.go:281`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_ALERTPG.go)
9. **`privacyID == "175"` special-cased**: baaki 4 special IDs ke unlike, isके liye
   `sendPackettoDetailEncr` (session-cache-invalidation call, `SESSION_UPDATE` queue) skip ho
   jaata hai — bina explanation ke code mein.
   [`USER_PRIVACYSETTING_ALERTPG.go:174,210`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_ALERTPG.go)
10. **Default-value dedupe-check gate (`DetailsAPIdisableDefaultArray`)**: before checking
    whether a `GLUSR_USR_PRIVACY_SETTING` row already exists, the controller only skips that
    check for the one specific combination "turning ON a setting whose default IS on"
    (`flag_del != "D"` AND priv_id not in `{76,91,81}`) — every other combination (including
    all `flag_del=="D"` cases) runs the existence check first.
    [`UserDetailsController.go:747`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDetailsController.go)
11. **App-version gate on `/setting`**: Android `< 13.2.9` and iOS `< 13.4.7` get an explicit
    `404 "This version is deprecated"` before any privacy-setting logic runs at all —
    independent of, and stricter than, the internal `12.3.2`/Android-old-version branching
    used inside the handlers themselves.
    [`SettingController.go:90-99`](../internal/controllers/UsersControllers/SettingController.go)
12. **UK (obfuscated token) alternate auth path**: if a `UK` param is present (and the caller
    isn't on the old-Android carve-out), the read path decrypts it with a hardcoded RC4-like
    key `"iilc2062016"`, extracts an embedded email + date, rejects if the token is **>7 days
    old**, and requires the decrypted email to match `glusr_usr.glusr_usr_email` for the given
    GLID — this looks like a signed-link mechanism (e.g. an emailed "manage your settings"
    link) bypassing normal token auth. The exact same key and near-identical logic is
    **duplicated** in both `UserSettingModel.go` (`UKDecryption`/`CheckUK`) and
    `UserSettingModel_v1.go` (`CheckUK_v1`, reuses the same `UKDecryption`) —
    [`UserSettingModel.go:1119-1225`](../internal/models/users/UserSettingModel.go),
    [`UserSettingModel_v1.go:604-649`](../internal/models/users/UserSettingModel_v1.go).

---

## 6. RabbitMQ — Queues Used

| Queue / `SERVICENAME` | Publisher | Consumer | Purpose |
|---|---|---|---|
| `MAIL_FREQUENCY_ALERT` | `UserDetailsController.go`, sirf `subCat=="mail_frequency"` ke liye | *(iss review mein consumer trace nahi kiya — email-frequency engine, alag scope)* | Email-frequency preference seedha email-engine ko forward karta hai, apni DB table update kiye bina |
| `USER_DETAILS_SERVICE` (generic) → `SERVICENAME` set hota hai [`UserDetailsModel.go:4188`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go) | Normal PrivSetting writes (mail_frequency ke alawa sab) | — | Yeh generic queue hai jo har detail-type (PrivSetting samet) ke liye common hai — GST domain ke `comp.sync.<glid%20>` pattern jaisa hi, koi PrivSetting-specific queue seedha yahan publish nahi hoti |
| `USER_PRIVACYSETTING_IMSDB`, `USER_PRIVACYSETTING_ALERTPG` | **[INFERRED — publisher iss review mein code mein nahi mila]** | [`USER_PRIVACYSETTING_IMSDB.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_IMSDB.go), [`USER_PRIVACYSETTING_ALERTPG.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_ALERTPG.go) | Yeh dono consumer apni exact queue naam se register hain (`internal/Router/router.go`), lekin write-API code mein inn exact queue-names ko seedha publish karta koi call nahi mila. **Sabse likely explanation**: `USER_DETAILS_SERVICE` message ek RabbitMQ topic exchange (`USER.topic`) pe publish hota hai, aur yeh dono queues us exchange se **infra-level bindings** (RabbitMQ config, Go code ke bahar) ke through automatically message paate hain — confirm karo infra/RabbitMQ-admin team se. |
| `SESSION_UPDATE` | `USER_PRIVACYSETTING_ALERTPG.go` (`sendPackettoDetailEncr`), sirf privacyID `45`/`90`/`20`/`153` ke liye (175 exclude) | *(session-cache consumer, alag scope)* | Jab denormalized `FK_MY_PRIVACY_SETTING_IDS` update hoti hai, session-level cache ko invalidate/refresh karne ka signal bhejta hai |

---

## 7. Kafka

**Privacy Setting domain mein koi Kafka usage nahi mila.** Dono consumers
(`USER_PRIVACYSETTING_IMSDB`, `USER_PRIVACYSETTING_ALERTPG`) `InitializeRabbitMq` use karte
hain, `InitializeKafka` nahi. Agar koi poochhe "kya Privacy Setting Kafka use karta hai" —
seedha jawaab hai: **nahi, sab RabbitMQ hai.**

---

## 8. Redis

**Koi Redis usage nahi mila** Privacy Setting ke kisi bhi write ya read DB query path mein.
`GET /setting` (flag=2) har request pe live Postgres hit karta hai for both queries
(`GLUSR_USR_PRIVACY_SETTING` row, aur — in `GetUserPrivSetting` — `MY_PRIVACY_SETTING`
default-behavior). **Correction/addition vs. previous pass**: there IS an in-process
(non-Redis) cache for the **master catalog JSON file** — `config.PrivSettingMasterData`, set
at app startup (`SetMasterData`), and `GetUserPrivSetting` uses it when populated instead of
re-reading the file per-request (`UserSettingModel_v1.go:229-238`). A parallel package-level
var `config.PrivSettingDefBehavior` exists (`configuration.go:191`) apparently meant to cache
the `MY_PRIVACY_SETTING` default-behavior table the same way, but `GetUserPrivSetting` does
**not** use it — it always re-queries `MY_PRIVACY_SETTING` live via `ReadPrivSettDefBehavior`
(`UserSettingModel_v1.go:220`). This looks like a half-finished caching effort — see section 9,
point 4.

---

## 9. End-to-End Technical Flows

### Flow A — Normal setting toggle (e.g. "show my mobile number")

```
Supplier (seller panel)
    │
    ▼
[API — write] POST /details {type: "PrivSetting", priv_id: "...", flag_del: "..."}
    │  UserDetailsController.go
    │  1. is_enable/is_automated fields set hote hain agar priv_id automated-array mein ho
    │  2. Validation (UserDetails_PrivSetting map)
    ▼
[DB — parallel goroutines]
    ├─ SELECT MY_PRIVACY_SETTING.MY_PRIVACY_DEFAULT_BEHAVIOR (default check)
    └─ SELECT GLUSR_USR_PRIVACY_SETTING EXISTS_FLAG (record already hai?)
    ▼
[Decision]  Truth-table from section 4.3 → no action / insert / delete
    ▼
[DB write]  GLUSR_USR_PRIVACY_SETTING — INSERT (naya) ya UPDATE (existing) ya DELETE (agar off kiya)
    │
    ├─ [Conditional, parallel]  History stored-proc call — "Privacy Setting Change" audit
    │
    ▼
[RabbitMQ publish]  SERVICENAME=USER_DETAILS_SERVICE (generic detail-change event)
    │  [INFERRED] RabbitMQ exchange bindings fan-out karte hain multiple queues ko
    ▼
[CONSUME]  USER_PRIVACYSETTING_IMSDB.go → searchPg replica sync
[CONSUME]  USER_PRIVACYSETTING_ALERTPG.go → conditional authPg + alertPg writes (sequential, Flow B/D dekho)
```

### Flow B — Denormalized flag update (sirf 5 specific priv IDs)

```
[CONSUME]  USER_PRIVACYSETTING_ALERTPG.go → authPgUpsert()  (called FIRST, sequentially)
    │
    ▼
privacyID IN (45, 90, 20, 153, 175)?
    │
    ├─ Nahi → "No updation required", kuch nahi hota
    │
    └─ Haan →
        [DB read]   authPg: SELECT FK_MY_PRIVACY_SETTING_IDS FROM GLUSR_USR WHERE GLUSR_USR_ID=$1
        [Compute]   comma-separated list mein se ID add/remove karo (in-memory string manipulation)
        [DB write]  authPg: UPDATE GLUSR_USR SET FK_MY_PRIVACY_SETTING_IDS=...
        │
        └─ agar privacyID != "175" →
               [RabbitMQ]  SESSION_UPDATE queue notify (cache invalidation signal)
```

### Flow C — Mail Frequency (bypass flow, no DB write in this domain)

```
Supplier
    │
    ▼
[API]  POST /details {type: "PrivSetting", sub_setting/subCat: "mail_frequency", ...}
    │  UserDetailsController.go
    │  NO database write yahan — UserDetailUpdateDB() call hi skip ho jaata hai
    ▼
[RabbitMQ publish]  SERVICENAME=MAIL_FREQUENCY_ALERT
    ▼
Email-frequency engine (alag scope, iss review mein trace nahi kiya)
```

### Flow D — Email Unsubscribe tracking

```
[CONSUME]  USER_PRIVACYSETTING_ALERTPG.go → emailUnsubsUpsert()  (called SECOND, sequentially, after authPgUpsert)
    │
    ▼
NEW_VAL == "Disabled" AND SOURCE_TYPE != "" ?
    │
    ├─ Haan →
    │     [DB read]   alertPg: SELECT count(1) FROM GLUSR_EMAIL_UNSUBSCRIBERS WHERE glid+privacyID
    │     ├─ Record already exists → UPDATE GLUSR_EMAIL_UNSUBSCRIBE_DATE=NOW()
    │     └─ Record nahi hai → INSERT naya unsubscribe record
    │
    └─ NEW_VAL == "Enabled" →
          [DB write]  alertPg: DELETE FROM GLUSR_EMAIL_UNSUBSCRIBERS WHERE glid+privacyID
```

### Flow E — Settings read, path 1 (`GetUserPrivacySettingsModel`/`V2`, no `version=v2` param)

```
Seller Panel UI / App
    │
    ▼
[API — read]  GET /setting  (Android AND app_version_no <= 12.3.2 → V1 function, else → V2 function)
    │
    ├─ flag == "0" or "1" ?
    │     → NOT a read — internal HTTP POST to write API's /details (Flow A triggered)
    │
    └─ flag == "2" (actual read) →
        [DB read]  SELECT fk_glusr_usr_id, fk_my_privacy_setting_id, glusr_privacy_is_enable
                   FROM GLUSR_USR_PRIVACY_SETTING WHERE FK_GLUSR_USR_ID = $1
        [File/merge]  hardcoded hot-path arrays (section 4.4) decide default CHECKED/ISENABLE
                      per-setting, using a file-based master catalog (`masterprivacydata[_gke]`)
                      for names/titles — no live MY_PRIVACY_SETTING query in these two functions
        │
        └─ Response: full per-setting enabled/disabled map
```

### Flow F — Settings read, path 2 (`version=v2` → `GetUserPrivSetting`, the "true" newer path)

```
Seller Panel UI / App
    │
    ▼
[API — read]  GET /setting?version=v2
    │  SettingController → SettingController_v1 → GetUserPrivSetting
    │
    ├─ flag == "0"/"1" → same internal-curl-to-write-API bypass as Flow E
    │
    └─ flag == "2" →
        ├─ UK param present (and not old-Android)? → CheckUK_v1: decrypt, check 7-day expiry,
        │   match decrypted email against glusr_usr.glusr_usr_email — reject (401/402/403) if any fail
        │
        [DB — 3 parallel goroutines]
        ├─ SELECT FK_GLUSR_USR_ID, FK_MY_PRIVACY_SETTING_ID, GLUSR_USR_PRIV_MODIFIED_DATE
        │   FROM GLUSR_USR_PRIVACY_SETTING WHERE FK_GLUSR_USR_ID=$1
        ├─ ReadPrivSettDefBehavior() → SELECT MY_PRIVACY_SETTING_ID, MY_PRIVACY_DEFAULT_BEHAVIOR
        │   FROM MY_PRIVACY_SETTING   (live query — NOT using the config.PrivSettingDefBehavior cache)
        └─ Master catalog: use config.PrivSettingMasterData cache if populated, else re-read
            file from disk (ReadMasterDataPrivSetting)
        │
        ▼
        [Conditional DB read]  glusr_seller_assistant_details — only if priv_ids 1250/1248/1246
                                are among the settings being returned (section 4.7)
        │
        ▼
        Response: per-setting enabled/disabled + MY_PRIVACY_SETTING_MOD_DATE + optional SUB_SETTING_JSON
```

---

## 10. Flow-wise DB & Table Usage — Kaun sa DB, Kaun sa Table, Kis Liye

### Flow A — Setting Toggle (write)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | Write DB (mesh-equivalent) | `MY_PRIVACY_SETTING` | SELECT (parallel goroutine) | `MY_PRIVACY_DEFAULT_BEHAVIOR` nikalna — decide karta hai insert vs delete vs no-action (section 4.3) |
| 2 | Write DB | `GLUSR_USR_PRIVACY_SETTING` | SELECT EXISTS (parallel goroutine) | Kya row already hai — same decision ke liye |
| 3 | Write DB | `GLUSR_USR_PRIVACY_SETTING` | INSERT / UPDATE / DELETE | Actual state-change |
| 4 | mainPg/Oracle (likely) | History table (name unconfirmed) | CALL stored-proc | Audit trail — conditional, non-INSERT ops ke liye |

### Flow B — Denormalized Flag (consumer)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | authPg | `GLUSR_USR` | SELECT `FK_MY_PRIVACY_SETTING_IDS` | Current comma-list nikalna |
| 2 | authPg | `GLUSR_USR` | UPDATE `FK_MY_PRIVACY_SETTING_IDS` | Naya comma-list likhna (ID add/remove ke baad) |

### Flow C — Mail Frequency

*(No DB in this domain — see section 9, Flow C)*

### Flow D — Email Unsubscribe (consumer, runs sequentially after Flow B in the same message)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | alertPg | `GLUSR_EMAIL_UNSUBSCRIBERS` | SELECT count | Existing unsubscribe record check |
| 2 | alertPg | `GLUSR_EMAIL_UNSUBSCRIBERS` | UPDATE or INSERT | Unsubscribe date refresh, ya naya record |
| 2b | alertPg | `GLUSR_EMAIL_UNSUBSCRIBERS` | DELETE | (alternate path) supplier ne wapas "Enabled" kiya |

**Combined Flow A→B→D total DB round-trips for one setting-toggle event: at least 8** across
4 physical databases (write-DB, authPg, alertPg, searchPg via IMSDB) — plus the audit-history
call. `authPgUpsert` aur `emailUnsubsUpsert` **sequentially** run (not parallel) inside the same
consumer delivery handler — see Optimization Scope, point 2.

### Flow E — Settings Read, path 1 (flag=2, `GetUserPrivacySettingsModel`/`V2`)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 1 | read-API's mesh_pg_user connection | `GLUSR_USR_PRIVACY_SETTING` | SELECT | Current per-setting enable state |
| — | *(file, not DB)* | `masterprivacydata`/`masterprivacydata_gke` JSON | file read (per-request in `GetUserPrivacySettingsModel`; not cached in this path) | Setting names/titles/alert-types + which IDs to show for a given `setting_type` |

### Flow F — Settings Read, path 2 (flag=2, `version=v2` → `GetUserPrivSetting`)

| # | DB | Table | Operation | Kyun |
|---|---|---|---|---|
| 0 | read-API's `mesh_pg_user` | `GLUSR_USR` | SELECT `glusr_usr_email` | Only if `UK` param present — validates the signed-link token |
| 1 | read-API's `mesh_pg_user` | `GLUSR_USR_PRIVACY_SETTING` | SELECT (parallel goroutine) | Per-setting current enable state + modified date |
| 2 | read-API's `mesh_pg_user` | `MY_PRIVACY_SETTING` | SELECT (parallel goroutine, `ReadPrivSettDefBehavior`) | Live default-behavior per setting — **not cached**, despite a cache variable existing for exactly this (section 8) |
| 3 | read-API's `mesh_pg_user` | `glusr_seller_assistant_details` | SELECT (`DISTINCT ON`) | Only if priv_ids 1250/1248/1246 present — sub-setting JSON blob |

---

## 11. Optimization Scope — DB Response-Time Contribution

### High-impact

1. **Flow A mein do parallel SELECT queries + ek write, ek hi HTTP request ke andar** — yeh
   already goroutines mein parallelize hai (achi baat), lekin **history stored-proc call bhi
   ek teesri parallel query hai** jo synchronously wait ho sakti hai response se pehle
   (depends on `wg.Wait()` placement) — confirm karo yeh truly async hai ya request ko block
   karti hai.
2. **`USER_PRIVACYSETTING_ALERTPG` consumer sequentially do independent DB operations karta
   hai** — `authPgUpsert()` (authPg) phir `emailUnsubsUpsert()` (alertPg), ek ke baad ek
   (`internal/Workers/USER_PRIVACYSETTING_ALERTPG.go:81,83`), jabki yeh do alag databases pe
   alag operations hain aur ek doosre pe depend nahi karte. **Concrete fix**: inhe bhi
   `USER_GST_LAST_MODIFIED.go` jaise goroutines mein parallel chalao.
3. **Default-behavior query re-run har `flag=2` request pe `GetUserPrivSetting` mein, bina
   cache ke** — `config.PrivSettingDefBehavior` package-level var already exists exactly iske
   liye (`configuration.go:191`), lekin `GetUserPrivSetting` isse use hi nahi karta, seedha
   live `ReadPrivSettDefBehavior` query chala deta hai (`UserSettingModel_v1.go:220`). Master
   catalog cache (`PrivSettingMasterData`) already correctly wired hai next line pe — yeh ek
   **half-finished optimization** lagta hai. **Concrete fix**: `PrivSettingDefBehavior` ko bhi
   startup pe populate karo aur read-path mein use karo, jaise master-data cache already
   hota hai.

### Medium-impact

4. **Read-side default/enabled arrays teen jagah independently duplicate hain aur already
   drift kar chuke hain** (section 4.4) — `53`, `76`, `81`, `83`, `84` jaise IDs ek function
   mein hain, doosre mein nahi. Agar `MY_PRIVACY_SETTING` mein koi naya setting add ho ya
   kisi ka default-behavior change ho, teen jagah manually update karna padega —
   `GetUserPrivSetting` already isse avoid karta hai (live DB read), baaki do functions
   isi approach pe migrate ho sakte hain agar woh abhi bhi actively maintained hain.
5. **`authPgUpsert` read-modify-write race condition risk** on `GLUSR_USR.FK_MY_PRIVACY_SETTING_IDS`
   — do concurrent changes (same 5 special priv-IDs mein se) ek dusre ko silently overwrite
   kar sakte hain. Postgres array/jsonb column se atomic banaya ja sakta hai.
6. **Koi caching nahi hai `GET /setting` ki DB queries pe** (dono `GLUSR_USR_PRIVACY_SETTING`
   aur `MY_PRIVACY_SETTING` reads) — cache-invalidate-on-write pattern safe hoga.
7. **Hardcoded UK-decryption key (`"iilc2062016"`) duplicated in two files** — `UserSettingModel.go`
   aur `UserSettingModel_v1.go` dono mein same literal, same RC4-jaisa algorithm. Agar key
   kabhi rotate karni ho, dono jagah change karna padega — ek shared function/const mein
   consolidate karna maintenance risk kam karega.
8. **`authPgUpsert`'s `meshconn *sql.DB` parameter is unused** — the function signature takes
   a mesh connection but internally opens its own separate `authPg` connection
   (`utils.GetDb("authPg","postgres")`) instead of using the passed-in parameter
   (`USER_PRIVACYSETTING_ALERTPG.go:119,135`) — dead parameter, harmless but confusing.

### Low-impact / good practice already present

9. **Flow A ke do initial SELECTs already parallel hain** (goroutines + channels pattern).
10. **Denormalized `FK_MY_PRIVACY_SETTING_IDS` column** (sirf 5 hot-path settings ke liye) khud
    ek optimization hai.
11. **Master-catalog JSON already cached at startup** (`config.PrivSettingMasterData`) and used
    correctly by `GetUserPrivSetting` — a genuine win missed by the older two read functions,
    which re-read the file per-request via `ReadMasterDataPrivSetting`/inline
    `utils.ReadJsonFileInOrder` calls.

---

## 12. Cron Inventory

**No cron found for the Privacy Setting domain.** Checked both
`user-temp-consumers-production/internal/Workers` (only GST-related and unrelated files match
"cron") and `service-api-go-production/crons/*` (three subfolders exist —
`ML_Retail`, `recommend`, `users` — the `users` folder contains only `OldGsmNumberTTL.go` and
`VerificationSourceUpdate.go`, neither of which touches Privacy Setting tables or code paths).
If a scheduled reconciliation job for Privacy Setting is added in the future (e.g. for the
proposed Night Notification auto-toggle in the Task 1 doc), this section should be updated.

---

## 13. Edge Cases & Gotchas (technical POV)

1. **`GET /setting` is not read-only** — `flag=0`/`flag=1` internally proxy to a write-API
   `POST /details` call (section 2). Anyone assuming this endpoint is safe to call repeatedly
   for polling/monitoring without side effects is wrong for these two flag values.
2. **Three independent read implementations exist, with genuinely different behavior**, not
   just three thin wrappers — `GetUserPrivacySettingsModel`, `GetUserPrivacySettingsModelV2`,
   `GetUserPrivSetting` each hardcode their own default/enabled arrays (section 4.4), and only
   the last one avoids drifting from the live `MY_PRIVACY_SETTING` table. Routing between
   them depends on **two independent signals** — the `version` query param and the client's
   `mod_id`/`app_version_no` — that are easy to conflate when debugging ("it's version 2" could
   mean either).
3. **Publisher of `USER_PRIVACYSETTING_IMSDB`/`USER_PRIVACYSETTING_ALERTPG` queues code mein
   trace nahi hua** (section 6) — likely RabbitMQ exchange-binding infra config, application
   code ke bahar.
4. **`privacyID == "175"` ka special-case exclusion** (section 5, point 9) bina explanation
   ke hai — worth confirming why.
5. **Read-modify-write race condition** on `GLUSR_USR.FK_MY_PRIVACY_SETTING_IDS` (section 11,
   point 5).
6. **`mail_frequency` sub-setting apni khud ki koi persistent record nahi rakhta iss domain
   mein** — poori tarah email-engine pe dependent hai.
7. **Hardcoded UK-decryption key duplicated across two files** (section 11, point 7) — a
   security-hygiene item worth a review, independent of the optimization angle.
8. **Comment says "3, 53" are inverted, code only checks "3" and "81"** (section 4.2) — a
   stale comment or a missing code path; don't trust the comment text over the actual `if`
   conditions when debugging inversion behavior.

---

## 14. Open Questions

1. `USER_PRIVACYSETTING_IMSDB` aur `USER_PRIVACYSETTING_ALERTPG` queues ka exact publisher/
   binding kya hai (section 6)? Application code se trace nahi hua.
2. `MAIL_FREQUENCY_ALERT` queue ka consumer kaun hai, aur kya woh apni koi persistent record
   rakhta hai? Iss review ke scope se bahar tha.
3. `privacyID == "175"` ke liye `SESSION_UPDATE` skip kyun hai (section 5, point 9)?
4. History stored-procedure (`pck_gl_history_sp_add_custom_history`) kis exact table mein
   likhta hai, aur kya woh async hai ya request ko block karta hai (section 11, point 1)?
5. Comment "Inverting the action for setting id's 3, 53" vs. code that only checks 3/81
   (section 4.2) — which is authoritative, and is `53` supposed to be inverted somewhere else?
6. `glusr_seller_assistant_details` ka business purpose aur `1250`/`1248`/`1246` ka exact
   naam-mapping (section 3, section 4.7) — inferred from a fallback key list, not confirmed.
7. `GetUserPrivacySettingsModel`/`V2` still actively used/maintained hain, ya legacy freeze
   ho chuke hain (section 4.4, 11 point 4)? Agar active hain, unka default-array drift ek
   real bug-source hai.
8. Night Notification (`priv_id 1243`) automation task — status kya hai, implement hua ya
   nahi (section 4.8)?
9. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`Privacy_Setting_Business_Doc.md`](./Privacy_Setting_Business_Doc.md) — same flows,
  product/business perspective
- [`Task 1 - Night Notification Manual vs Automated Flag.md`](./Task%201%20-%20Night%20Notification%20Manual%20vs%20Automated%20Flag.md) —
  proposed `is_automated` activation for priv_id 1243, references section 4.1/4.8 of this doc
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — similar-shape
  domain doc, useful comparison for the goroutine-parallelization pattern referenced in
  section 11
- [`../users_domain_product_stories.md`](../users_domain_product_stories.md) — Privacy
  Setting ka role bigger "Notifications, Alerts & Preferences" story mein
- [`../utils_programming_guide.md`](../utils_programming_guide.md) — shared `PushToQueue`/
  RabbitMQ helpers jo iss doc mein reference hue hain

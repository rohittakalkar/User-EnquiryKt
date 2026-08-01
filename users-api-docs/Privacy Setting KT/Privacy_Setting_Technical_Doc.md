# Privacy Setting — Technical Doc (Code-Level Deep Dive)

Yeh doc Privacy Setting feature ka **technical implementation** cover karta hai — APIs, DB
tables, queries, RabbitMQ, consumers, sab code se verify karke. Business/product perspective
ke liye [`Privacy_Setting_Business_Doc.md`](./Privacy_Setting_Business_Doc.md) dekho.

**Repos**: `users-api-go-production` (read), `service-api-go-production` (write),
`user-temp-consumers-production` (consumers).

**Methodology**: har claim neeche real source code se trace kiya gaya hai (file path diya
gaya hai har jagah). Jahan code se pura confirm nahi ho paaya, wahan
**[INFERRED — team se confirm karo]** likha hai.

---

## 1. Privacy Setting Kahan-Kahan Hai — File Map

| Concern | Repo | File |
|---|---|---|
| Privacy setting submit/update (general detail-editor ke through) | write | [`UserDetailsController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDetailsController.go), [`UserDetailsModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go) |
| Validation rules/field map | write | [`UsersValidationMaps.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UsersValidationMaps.go) (`UserDetails_PrivSetting` map) |
| Setting read (v1 aur v2, dono variants) | read | [`SettingController.go`](../internal/controllers/UsersControllers/SettingController.go), [`SettingController_v1.go`](../internal/controllers/UsersControllers/SettingController_v1.go), [`UserSettingModel.go`](../internal/models/users/UserSettingModel.go), [`UserSettingModel_v1.go`](../internal/models/users/UserSettingModel_v1.go) |
| Change fan-out — search/IMSDB database | consumers | [`USER_PRIVACYSETTING_IMSDB.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_IMSDB.go) |
| Change fan-out — auth DB (denormalized flag) + alert DB (email unsubscribe) | consumers | [`USER_PRIVACYSETTING_ALERTPG.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_ALERTPG.go) |

**Related but separate feature**: `PnsSettingControllers` (PNS/call-routing off-hours
preferences, routes `/pnssetting/*params`) is a **different** feature that also uses the word
"setting" — don't confuse it with Privacy Setting. It's covered under the "Notifications,
Alerts & Preferences" product story, not this doc.

---

## 2. Routes (confirmed from router.go)

| Method | Path | Repo | Controller |
|---|---|---|---|
| POST | `/details` (type=PrivSetting) | write | `UserDetailsController` |
| GET/POST | `/setting/*params` | read | `SettingController` (delegates to `SettingController_v1` if `version=="v2"`) |

---

## 3. Data Model — Tables

> **Verification note**: table/column names Go code ke embedded SQL strings se liye gaye
> hain. Live DB schema se cross-verify nahi kiya gaya — kisi migration se pehle woh zaroor
> karo.

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_USR_PRIVACY_SETTING` | Primary write DB (`dbConn` in write API — same connection `UserDetailsModel` uses for other detail-types) + replicated to searchPg (IMSDB) | **Primary record** — har supplier ke har setting ka current state | `FK_GLUSR_USR_ID`, `FK_MY_PRIVACY_SETTING_ID`, `GLUSR_PRIVACY_UPDATEDBY_FLAG`, `GLUSR_PRIVACY_UPDATEDBY_ID`, `GLUSR_PRIVACY_UPDATEDBY`, `GLUSR_PRIVACY_UPDATESCREEN`, `GLUSR_PRIVACY_IP`, `GLUSR_PRIVACY_IP_COUNTRY`, `GLUSR_PRIVACY_HIST_COMMENTS`, `GLUSR_PRIVACY_SETTING_REMARKS`, `GLUSR_PRIVACY_IS_ENABLE`, `GLUSR_PRIVACY_IS_AUTOMATED`, `GLUSR_USR_PRIV_MODIFIED_DATE` — [`UserDetailsModel.go:2512,2817`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go), [`USER_PRIVACYSETTING_IMSDB.go:119,136,176`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_IMSDB.go) |
| `MY_PRIVACY_SETTING` | Same write DB | **Master/catalog table** — har setting-type ka definition, incl. default behavior | `MY_PRIVACY_SETTING_ID`, `MY_PRIVACY_DEFAULT_BEHAVIOR` — [`UserDetailsModel.go:2532`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go) |
| `GLUSR_USR` | authPg | Specific privacy IDs (`45`, `90`, `20`, `153`, `175` only) ke liye, ek **denormalized comma-separated list** column maintain karta hai — quick lookup ke liye, bina main table join kiye | `FK_MY_PRIVACY_SETTING_IDS` (comma-separated string of enabled privacy-setting IDs) — [`USER_PRIVACYSETTING_ALERTPG.go:140,167,196`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_ALERTPG.go) |
| `GLUSR_EMAIL_UNSUBSCRIBERS` | alertPg | Compliance-critical — kaun-kaunse supplier ne kaunsi email-category se unsubscribe kiya | `FK_GLUSR_USR_ID`, `FK_MY_PRIVACY_SETTING_ID`, `FK_IIL_PROCESS_MASTER_ID`, `EMAIL_UNSUBSCRIBE_REMARKS`, `GLUSR_EMAIL_UNSUBSCRIBE_DATE` — [`USER_PRIVACYSETTING_ALERTPG.go:250-292`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_ALERTPG.go) |
| History (stored procedure, table name explicit nahi mila) | Likely mainPg/Oracle | Audit-trail — "Privacy Setting Change" comment ke saath | `CALL pck_gl_history_sp_add_custom_history(...)` — [`UserDetailsModel.go:2856`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go) |

---

## 4. Business Rules & Validation (code se)

1. **Setting-type-specific fields**: `priv_id` (`FK_MY_PRIVACY_SETTING_ID`) decide karta hai
   konse extra columns (`is_enable`, `is_automated`) apply hote hain. Yeh sirf specific IDs
   (`components.DetailsAPIautomatedIsEnableArray` mein list ki hui) ke liye set hote hain —
   baaki settings ke liye yeh fields khali chhod diye jaate hain.
   [`UserDetailsController.go:214-235`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDetailsController.go)
2. **"Inverted action" for specific setting IDs**: `priv_id == "3"` (aur INSERT-time logic
   mein `19`, `62`, `51` bhi flagged hain) ke liye, `flag_del == "D"` ka matlab actually
   **"Insert"** treat hota hai, aur non-D ka matlab **"Update"** — baaki settings ke liye
   yeh seedha (D=Delete, non-D=Insert/Update) hai. Comment code mein khud kehta hai:
   `"Inverting the action for setting id's 3, 53"`.
   [`UserDetailsController.go:183-187`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDetailsController.go)
3. **`mail_frequency` sub-category bypasses the entire normal write path**: agar
   `Type == "PrivSetting"` AUR `subCat == "mail_frequency"`, koi DB write hoti hi nahi
   (`UserDetailUpdateDB` call hi skip ho jaata hai) — sirf seedha ek RabbitMQ message push
   hota hai `SERVICENAME=MAIL_FREQUENCY_ALERT` ke saath.
   [`UserDetailsController.go:240-261`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserDetailsController.go)
4. **`sub_setting` param sirf `PrivSetting` type ke liye allowed hai** — kisi aur detail-type
   ke saath use karne pe explicit error: `"sub_setting is only allowed for type PrivSetting"`.
   [`UserDetailsModel.go:2827-2830`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)
5. **History logging conditional hai**: agar operation "Insert" jaisa nahi hai (non-INSERT
   ka koi specific check), ek alag goroutine `pck_gl_history_sp_add_custom_history` stored
   procedure call karti hai audit ke liye — yeh privacy-setting-specific comment
   (`'Privacy Setting Change'`) ke saath hardcoded hai.
   [`UserDetailsModel.go:2852-2883`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go)
6. **Denormalized `FK_MY_PRIVACY_SETTING_IDS` sirf 5 specific setting IDs ke liye maintain
   hota hai**: `45`, `90`, `20`, `153`, `175`. Baaki sab settings ke liye yeh column touch
   hi nahi hota — yeh performance-optimization jaisa lagta hai (specific hot-path settings
   ke liye ek fast comma-list lookup, poori table join kiye bina).
   [`USER_PRIVACYSETTING_ALERTPG.go:133`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_ALERTPG.go)
7. **Email-unsubscribe sirf specific condition pe trigger hota hai**: `NEW_VAL == "Disabled"`
   AUR `SOURCE_TYPE` non-empty ho, dono saath — agar `SOURCE_TYPE` missing hai, unsubscribe
   list mein add nahi hoga (Business Doc §5.2 ka technical root-cause yahi hai).
   [`USER_PRIVACYSETTING_ALERTPG.go:265`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_ALERTPG.go)
8. **`NEW_VAL == "Enabled"` reverse karta hai unsubscribe ko** — matching row
   `GLUSR_EMAIL_UNSUBSCRIBERS` se DELETE ho jaata hai.
   [`USER_PRIVACYSETTING_ALERTPG.go:281`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_ALERTPG.go)
9. **`privacyID == "175"` special-cased**: baaki 4 special IDs ke unlike, isके liye
   `sendPackettoDetailEncr` (session-cache-invalidation call, `SESSION_UPDATE` queue) skip ho
   jaata hai — bina explanation ke code mein, worth confirming with team why 175 is
   different.
   [`USER_PRIVACYSETTING_ALERTPG.go:174,210`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_ALERTPG.go)

---

## 5. RabbitMQ — Queues Used

| Queue / `SERVICENAME` | Publisher | Consumer | Purpose |
|---|---|---|---|
| `MAIL_FREQUENCY_ALERT` | `UserDetailsController.go`, sirf `subCat=="mail_frequency"` ke liye | *(iss review mein consumer trace nahi kiya — email-frequency engine, alag scope)* | Email-frequency preference seedha email-engine ko forward karta hai, apni DB table update kiye bina |
| `USER_DETAILS_SERVICE` (generic) → `SERVICENAME` set hota hai [`UserDetailsModel.go:4188`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserDetailsModel.go) | Normal PrivSetting writes (mail_frequency ke alawa sab) | — | Yeh generic queue hai jo har detail-type (PrivSetting samet) ke liye common hai — **GST domain ke `comp.sync.<glid%20>` pattern jaisa hi**, koi PrivSetting-specific queue seedha yahan publish nahi hoti |
| `USER_PRIVACYSETTING_IMSDB`, `USER_PRIVACYSETTING_ALERTPG` | **[INFERRED — publisher iss review mein code mein nahi mila]** | [`USER_PRIVACYSETTING_IMSDB.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_IMSDB.go), [`USER_PRIVACYSETTING_ALERTPG.go`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_ALERTPG.go) | Yeh dono consumer apni exact queue naam se register hain (`internal/Router/router.go`), lekin write-API code mein inn exact queue-names ko seedha publish karta koi call nahi mila. **Sabse likely explanation**: `USER_DETAILS_SERVICE` message ek RabbitMQ topic exchange (`USER.topic`) pe publish hota hai, aur yeh dono queues us exchange se **infra-level bindings** (RabbitMQ config, Go code ke bahar) ke through automatically message paate hain. Yeh confirm karo infra/RabbitMQ-admin team se — application code se yeh trace nahi ho sakta. |
| `SESSION_UPDATE` | `USER_PRIVACYSETTING_ALERTPG.go` (`sendPackettoDetailEncr`), sirf privacyID `45`/`90`/`20`/`153` ke liye (175 exclude) | *(session-cache consumer, alag scope)* | Jab denormalized `FK_MY_PRIVACY_SETTING_IDS` update hoti hai, session-level cache ko invalidate/refresh karne ka signal bhejta hai |

---

## 6. Kafka

**Privacy Setting domain mein koi Kafka usage nahi mila.** Dono consumers
(`USER_PRIVACYSETTING_IMSDB`, `USER_PRIVACYSETTING_ALERTPG`) `InitializeRabbitMq` use karte
hain, `InitializeKafka` nahi. Agar koi poochhe "kya Privacy Setting Kafka use karta hai" —
seedha jawaab hai: **nahi, sab RabbitMQ hai.**

---

## 7. Redis

**Koi Redis usage nahi mila** Privacy Setting ke kisi bhi write ya read path mein. `GET
/setting` har request pe live DB hit karta hai. Settings kaafi frequently change ho sakte
hain (compared to GST, jo lock ho jaata hai) — isliye caching yahan GST jitna straightforward
win nahi hai (TTL bahut short rakhna padega ya cache-invalidation-on-write karna padega), par
still ek genuine gap hai (dekho section 9, Optimization Scope).

---

## 8. End-to-End Technical Flows

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
[DB write]  GLUSR_USR_PRIVACY_SETTING — INSERT (naya) ya UPDATE (existing) ya DELETE (agar off kiya)
    │
    ├─ [Conditional, parallel]  History stored-proc call — "Privacy Setting Change" audit
    │
    ▼
[RabbitMQ publish]  SERVICENAME=USER_DETAILS_SERVICE (generic detail-change event)
    │  [INFERRED] RabbitMQ exchange bindings fan-out karte hain multiple queues ko
    ▼
[CONSUME]  USER_PRIVACYSETTING_IMSDB.go → searchPg replica sync
[CONSUME]  USER_PRIVACYSETTING_ALERTPG.go → conditional authPg + alertPg writes (Flow B/D dekho)
```

### Flow B — Denormalized flag update (sirf 5 specific priv IDs)

```
[CONSUME]  USER_PRIVACYSETTING_ALERTPG.go → authPgUpsert()
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
[CONSUME]  USER_PRIVACYSETTING_ALERTPG.go → emailUnsubsUpsert()
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

### Flow E — Settings read back

```
Supplier / Seller Panel UI
    │
    ▼
[API — read]  GET /setting/*params  (ya version=v2 → SettingController_v1 delegate)
    │
    ▼
[DB read]  SELECT fk_glusr_usr_id, fk_my_privacy_setting_id, glusr_privacy_is_enable
           FROM GLUSR_USR_PRIVACY_SETTING WHERE FK_GLUSR_USR_ID = $1
    │
    ▼
Response — har setting ka current on/off state, ek saath
```

---

## 9. Optimization Scope — DB Response-Time Contribution

Jaisa GST doc mein establish kiya (DB hi max contribute karta hai response time mein), yahan
bhi similar pattern dikhta hai, kuch alag nuances ke saath:

### High-impact

1. **Flow A mein do parallel SELECT queries + ek write, ek hi HTTP request ke andar** — yeh
   already goroutines mein parallelize hai (achi baat), lekin **history stored-proc call bhi
   ek teesri parallel query hai** jo synchronously wait ho sakti hai response se pehle
   (depends on `wg.Wait()` placement) — confirm karo yeh truly async hai ya request ko block
   karti hai.
2. **`USER_PRIVACYSETTING_ALERTPG` consumer sequentially do independent DB operations karta
   hai** — `authPgUpsert()` (authPg) phir `emailUnsubsUpsert()` (alertPg), ek ke baad ek,
   jabki yeh do alag databases pe alag operations hain aur ek doosre pe depend nahi karte.
   **Concrete fix**: inhe bhi `USER_GST_LAST_MODIFIED.go` jaise goroutines mein parallel
   chalao (GST tech doc section 12, point #7 ka positive pattern yahan copy ho sakta hai).
   [`USER_PRIVACYSETTING_ALERTPG.go:81-83`](../../user-temp-consumers-production/user-temp-consumers-production/internal/Workers/USER_PRIVACYSETTING_ALERTPG.go)

### Medium-impact

3. **`authPgUpsert` ek extra SELECT karta hai comma-separated list nikalne ke liye, phir
   in-memory string-split/manipulate karta hai, phir wapas UPDATE karta hai poori string.**
   Yeh ek **read-modify-write race condition risk** bhi hai — agar do settings changes (dono
   in 5 special priv-IDs mein se) ek saath aayein same supplier ke liye, dono ka SELECT
   purani value padh sakta hai, aur last-write-wins ho sakta hai (ek update overwrite ho
   sakta hai). Postgres array column ya `jsonb` set-operations use karke isse atomic banaya
   ja sakta hai (single UPDATE statement se, bina application-level read-modify-write ke).
4. **Koi caching nahi hai `GET /setting` pe** — GST ke uलट, privacy settings zyada
   frequently change ho sakte hain, isliye simple long-TTL cache risky hai. Lekin
   **cache-invalidate-on-write pattern** (write hone pe `RedisSet`/`RedisDel` turant call
   karo) safe hoga aur read-latency kaafi kam kar sakta hai high-read, low-write pattern
   waale suppliers ke liye.

### Low-impact / good practice already present

5. **Flow A ke do initial SELECTs already parallel hain** (goroutines + channels pattern) —
   yeh achi baat hai, GST ke `USER_GST_LAST_MODIFIED` jaisa hi pattern.
6. **Denormalized `FK_MY_PRIVACY_SETTING_IDS` column** (sirf 5 hot-path settings ke liye) khud
   ek optimization hai — poori `GLUSR_USR_PRIVACY_SETTING` table join kiye bina, ek fast
   comma-list check se kaam chal jaata hai un specific cases mein. Yeh acha design decision
   hai jab tak race-condition (point #3) fix na ho jaaye.

---

## 10. Full Flow Diagrams (Lucid, icon-based)

**[Poora Privacy Setting flowchart yahan dekho](https://lucid.app/lucidchart/a760d219-f330-4e38-b059-df6b243d1923/edit)**

| Page | Content |
|---|---|
| **0. Superset — All Flows** | Ek page pe poora Privacy Setting domain |
| **A. Normal Toggle** | Section 8 Flow A — parallel SELECTs, write, history, fan-out |
| **B. Denormalized Flag Update** | Section 8 Flow B — sirf 5 special priv-IDs, authPg update, SESSION_UPDATE notify |
| **C. Mail Frequency Bypass** | Section 8 Flow C — seedha RabbitMQ, koi DB write nahi |
| **D. Email Unsubscribe** | Section 8 Flow D — Disabled/Enabled decision tree |
| **E. Settings Read** | Section 8 Flow E — simple read path |

---

## 11. Edge Cases & Gotchas (technical POV)

1. **Publisher of `USER_PRIVACYSETTING_IMSDB`/`USER_PRIVACYSETTING_ALERTPG` queues code mein
   trace nahi hua** (section 5) — likely RabbitMQ exchange-binding infra config, application
   code ke bahar. Agar in consumers mein messages aana band ho jaayein, pehle RabbitMQ
   bindings check karo, application code nahi.
2. **`privacyID == "175"` ka special-case exclusion** (section 4, point 9) bina explanation
   ke hai — worth confirming why.
3. **Read-modify-write race condition** on `GLUSR_USR.FK_MY_PRIVACY_SETTING_IDS` (section 9,
   point 3) — do concurrent privacy-setting changes ek dusre ko silently overwrite kar sakte
   hain agar dono 5 special priv-IDs mein se hon.
4. **`mail_frequency` sub-setting apni khud ki koi persistent record nahi rakhta iss domain
   mein** — poori tarah email-engine pe dependent hai. Agar kabhi "mera email frequency
   history dikhao" jaisa feature chahiye ho, iske liye naya storage design karna padega.

---

## 12. Open Questions

1. `USER_PRIVACYSETTING_IMSDB` aur `USER_PRIVACYSETTING_ALERTPG` queues ka exact publisher/
   binding kya hai (section 5)? Application code se trace nahi hua.
2. `MAIL_FREQUENCY_ALERT` queue ka consumer kaun hai, aur kya woh apni koi persistent record
   rakhta hai? Iss review ke scope se bahar tha.
3. `privacyID == "175"` ke liye `SESSION_UPDATE` skip kyun hai (section 4, point 9)?
4. History stored-procedure (`pck_gl_history_sp_add_custom_history`) kis exact table mein
   likhta hai, aur kya woh async hai ya request ko block karta hai (section 9, point 1)?
5. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`Privacy_Setting_Business_Doc.md`](./Privacy_Setting_Business_Doc.md) — same flows,
  product/business perspective
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — similar-shape
  domain doc, useful comparison for the goroutine-parallelization pattern referenced in
  section 9
- [`../users_domain_product_stories.md`](../users_domain_product_stories.md) — Privacy
  Setting ka role bigger "Notifications, Alerts & Preferences" story mein
- [`../utils_programming_guide.md`](../utils_programming_guide.md) — shared `PushToQueue`/
  RabbitMQ helpers jo iss doc mein reference hue hain

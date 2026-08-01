# Task 1 — Night Notification (priv_id 1243): Manual vs Automated Change Flag

**Status**: Proposed (design ready, implementation pending)
**Requested by**: BL-Alerting team
**Related domain doc**: [`Privacy_Setting_Technical_Doc.md`](./Privacy_Setting_Technical_Doc.md)

---

## 1. Objective

BL-Alerting ko ek flag/identifier chahiye jisse yeh pata chal sake ki Night Notification
Privacy Setting (`priv_id = 1243`) ka current state **user ne manually** set kiya tha, ya
**backend system ne automatically**.

---

## 2. Background

BL-Alerting team **"Dynamic Night Notification Privacy Setting Based on Seller Purchase
Behaviour"** feature bana rahi hai — seller ke behaviour ke basis pe Night Notification
setting ko automatically enable/disable karna.

**Core business rule**: agar user ne khud manually koi preference set kar rakhi hai, system
usse kabhi automatically override nahi karega. Automatic changes sirf tab honi chahiye jab
setting kabhi manually configure hi nahi hui ho.

Is rule ko enforce karne ke liye backend ko pata hona chahiye: **"current state manual hai ya
auto hai"** — abhi yeh tracking exist nahi karti.

---

## 3. Discussion Points (DB team + API team ke saath)

- DB team seedha privacy settings update nahi kar sakti — yeh API team (Privacy Setting
  API, current write path) ke through hi hona chahiye.
- API team ne confirm kiya: abhi koi tarika nahi hai yeh identify karne ka ki Night
  Notification setting user ne enable ki hai ya nahi (currently sab manual maana jaata hai,
  koi tracking nahi).
- BL-Alerting ke daily auto-enable/disable process ke liye yeh identification pehle banani
  padegi, phir warehouse-end pe daily process design hoga.

---

## 4. Investigation Findings (code se verify kiya gaya)

Maine actual codebase check kiya — BL-Alerting/API team ka claim **bilkul sahi hai**, aur
achi baat yeh hai ki iska zyada tar infrastructure **already exist karta hai**, sirf
`priv_id 1243` ko usme wire nahi kiya gaya:

1. **`GLUSR_USR_PRIVACY_SETTING` table mein `GLUSR_PRIVACY_IS_AUTOMATED` column already
   maujood hai** — bilkul isi purpose ke liye pehle se bana hua (kisi aur setting ke liye
   use ho raha hai).
2. **Yeh column sirf 4 hardcoded `priv_id`s ke liye "on" hai**:
   ```go
   // service-api-go-production/pkg/components/commonMaps.go:113
   var DetailsAPIautomatedIsEnableArray = []int{3, 19, 62, 51}
   ```
   `1243` iss list mein nahi hai — isliye jab bhi Night Notification change hoti hai,
   `UserDetailsController.go:232-235` `is_automated` ko explicitly khaali kar deta hai:
   ```go
   } else {
       inputParams["is_enable"] = ""
       inputParams["is_automated"] = ""
   }
   ```
3. **Consumer-side (searchPg replica sync) mein bhi wahi gap hai** —
   `USER_PRIVACYSETTING_IMSDB.go` ka hardcoded `privId` map (`{"3", "19", "62", "51"}`) mein
   bhi `1243` nahi hai.
4. **Read API (`GET /setting`) is_automated column select hi nahi karta** — `UserSettingModel.go`
   aur `UserSettingModel_v1.go` (v2) ki query sirf `glusr_privacy_is_enable` laati hai, `is_automated`
   nahi. Matlab write-side fix karne ke baad bhi, BL-Alerting ko yeh value response mein
   tabhi milegi jab read query bhi update ho.

**Conclusion**: naya schema/table banane ki zaroorat nahi hai — jo column already hai, usse
sirf `1243` ke liye "activate" karna hai, 4 chhoti jagah.

---

## 5. Proposed Solution

### 5.1 — Four code changes

| # | Kya change | Kahan |
|---|---|---|
| 1 | `1243` ko allowed-array mein add karo | `service-api-go-production/pkg/components/commonMaps.go` → `DetailsAPIautomatedIsEnableArray = []int{3, 19, 62, 51, 1243}` |
| 2 | `1243` ko consumer ke hardcoded map mein add karo | `user-temp-consumers-production/.../USER_PRIVACYSETTING_IMSDB.go` → `privId` map |
| 3 | Read query mein `glusr_privacy_is_automated` column add karo | `users-api-go-production/.../UserSettingModel.go` **aur** `UserSettingModel_v1.go` (dono, v1 + v2) |
| 4 | Response payload mein `is_automated` field expose karo | Same read models, response-building portion |

### 5.2 — Semantics contract

- **`is_automated = 1`** → is setting ka last change system/backend ne kiya tha
- **`is_automated = 0`** (ya record hi exist nahi karta) → last change user ne khud manually
  kiya tha

### 5.3 — BL-Alerting ke daily process ka logic

```
Har seller ke liye:
    IF is_automated == 1  OR  record does not exist:
        → safe hai, auto-toggle kar sakte hain (aur is_automated=1 hi rakhna, write ke saath)
    ELSE (is_automated == 0):
        → SKIP — user ki manual preference respect karo, touch mat karo
```

---

## 6. ⚠️ Critical Risk — Design Mein Zaroor Cover Karna Hai

`is_automated` ki value abhi **client ke request se seedhe li jaati hai**
(`inputParams["is_automated"]`) — server khud decide nahi karta. Agar isi tarah rakha gaya,
toh:

- Agar seller-panel ka frontend kabhi galti se (ya kisi bug se) `is_automated=1` bhej de ek
  manual toggle ke saath, woh silently "auto" maan liya jaayega — aur BL-Alerting ka job
  bhavishya mein use freely override kar dega. **Yehi exact bug hai jisse yeh poora
  requirement bachna chahta hai.**

### Recommendation

`is_automated` client-supplied input pe trust mat karo. Iski jagah:

1. BL-Alerting ke daily job ka apna **dedicated `VALIDATION_KEY`/`modid`** ho (jaisa har
   controller mein already `Gateway_v1` allowlist pattern hai).
2. Server internally decide kare: sirf tabhi `is_automated=1` set ho jab caller ka `modid`
   BL-Alerting ke dedicated identifier se match kare — **client se aaya raw value ignore
   karo**.
3. Har normal seller-panel/app call ke liye `is_automated` **hamesha `0` force-set** ho,
   chahe request mein kuch bhi aaya ho.

Yeh implementation mein ek chhota sa change hai, lekin poore feature ki reliability isi ek
decision pe depend karti hai.

---

## 7. Proposed API Contract (for BL-Alerting)

```
GET /setting?glusrid=<id>&setting_type=1243

Response:
{
  ...
  "priv_id": 1243,
  "is_enable": 1,
  "is_automated": 0     // 0 = user ne manually set kiya, 1 = system/BL-Alerting ne last set kiya
}
```

Write side (BL-Alerting ka daily job, jab auto-toggle karna ho):
```
POST /details
{
  "type": "PrivSetting",
  "priv_id": "1243",
  "is_automated": "1",   // server BL-Alerting ke dedicated modid se hi isse honor karega
  ...
}
```

---

## 8. Open Items / Next Steps

- [ ] `commonMaps.go` mein `1243` add karna (Privacy Setting API team)
- [ ] Consumer `privId` map update karna (Privacy Setting API team)
- [ ] Read query + response mein `is_automated` expose karna, dono versions (v1 + v2)
  (Privacy Setting API team)
- [ ] BL-Alerting ke daily job ke liye ek dedicated `modid`/`VALIDATION_KEY` allocate karna,
  aur server-side pe `is_automated` ko client-input se decouple karna (point 6)
- [ ] Confirm karo: kya existing (agar koi hai) live `1243` settings ka backfill chahiye
  (unka current state pata nahi hai manual tha ya nahi — default kya rakhoge un purane
  records ke liye?)
- [ ] BL-Alerting team ke saath API contract (section 7) finalize karna
- [ ] Warehouse-end daily process design (BL-Alerting/DB team scope, is task ke baad)

---

## See also

- [`Privacy_Setting_Business_Doc.md`](./Privacy_Setting_Business_Doc.md) — feature ka
  business overview
- [`Privacy_Setting_Technical_Doc.md`](./Privacy_Setting_Technical_Doc.md) — full technical
  reference, `is_automated`/`is_enable` ka existing behavior (§4, point 1) is task ka base hai

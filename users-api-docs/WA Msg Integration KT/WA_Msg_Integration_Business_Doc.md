# WhatsApp Message Integration — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`WA_Msg_Integration_Technical_Doc.md`](./WA_Msg_Integration_Technical_Doc.md) dekho.

**Note**: Yeh doc **do genuinely alag cheezein** cover karta hai jo dono "WhatsApp" naam
share karte hain lekin technically aur business-purpose dono se independent hain:

1. **WhatsApp Business Platform (BSP) Integration** — supplier apna WhatsApp Business
   account IndiaMART ke saath link karta hai (Meta/Facebook-backed onboarding, ongoing
   status-tracked relationship)
2. **Outbound "App Install" WhatsApp Nudge** — IndiaMART khud kisi user (buyer ya supplier)
   ko ek WhatsApp message bhejta hai app-install ke liye, ek generic SMS/WhatsApp
   notification-dispatch system ke through

Ek **teesra**, alag se fully-documented feature bhi WhatsApp naam use karta hai lekin ismein
cover nahi hai: supplier ke public profile pe "WhatsApp contact-method" toggle
(`IS_WHATSAPP_ACTIVE`) — uska apna KT hai
[`Social Contacts Business Doc`](../Social%20Contacts%20KT/Social_Contacts_Business_Doc.md) mein,
jahan deeper review ke baad bhi confirm hua ki yeh iss doc ke dono features se genuinely
alag hai (dekho section 3 neeche).

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

### 1.1 — WhatsApp Business Integration (linking)

Supplier apna official **WhatsApp Business Account** IndiaMART platform se link kar sakta
hai. Yeh Meta/Facebook ke WhatsApp Business Platform ke through hota hai, ek external
"BSP" (Business Solution Provider — jaise AiSensy, ValueFirst) vendor ke through — ek proper
business-messaging onboarding, jaisa kisi bhi enterprise WhatsApp integration mein hoti hai
(KYC, business-manager verification, quality-rating tracking, sab saath).

**Business impact**: Ek baar link ho jaaye, supplier IndiaMART ke through WhatsApp pe buyers
se baat kar sakta hai. Yeh ek premium/value-added messaging capability hai suppliers ke liye,
aur poora status internal admin/LMS team se manage hota hai (supplier khud seedha yeh record
edit nahi karta).

### 1.2 — Outbound "App Install" WhatsApp Nudge

Yeh ek alag, chhota utility hai jo IndiaMART khud use karta hai kisi user ko ek "IndiaMART app
install karo" WhatsApp message bhejne ke liye. Yeh koi supplier ka apna linked-account use
nahi karta — IndiaMART ka apna vendor-backed sending mechanism hai, aur yehi mechanism
**SMS ke liye bhi use hota hai** (same underlying dispatch point, bas message-channel
alag ho jaata hai kis "source" pe based hai).

**Business impact**: Ek marketing/engagement tool — users ko re-engage karne ke liye WhatsApp
jaisa high-open-rate channel use kiya jaata hai, bina spam-jaisa lagne ke (24-hour
frequency-cap se protected).

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Apna WhatsApp Business account link karta hai (Integration feature, §1.1) |
| **Meta/Facebook (external)** | Business-verification, KYC, quality-rating provide karta hai |
| **BSP (Business Solution Provider, external vendor — AiSensy/Meta/ValueFirst)** | WhatsApp messaging infrastructure provide karta hai jispe integration based hai |
| **Internal Admin / LMS team** | Integration record ko GLADMIN/LMS gateway ke through create/update karta hai — supplier khud seedha nahi karta |
| **IndiaMART internal systems (enq/click/PNS/verify jaise triggers)** | Outbound app-install nudges trigger karte hain, WhatsApp aur SMS dono ke liye same dispatch point se |
| **Buyer/Supplier (nudge receiver)** | App-install nudge receive karta hai, apne 24-hour rate-limit ke andar |

---

## 3. Do Features — Alag-Alag Samjho (aur teesra kyun involved nahi hai)

### 3.1 — WhatsApp Business Integration Record

**Business flow**:
```
1. Internal admin (GLADMIN) ya LMS caller WhatsApp Business account link karne ka process
   start karta hai — supplier khud yeh API seedha nahi call karta
2. System ek integration-record banata hai — phone number, business ID, token, service dates
3. Meta/Facebook-side verification status track hoti hai: business-manager verification,
   display-name verification, KYC status, quality-rating, aur onboarding-flow type (BSP/tech
   partner/self-serve)
4. Status changes (jaise account "reinstated"/"unlocked" ho jaana, ya "locked"/"deleted" ho
   jaana) ek audit-log mein bhi record hote hain — lekin sirf specific transitions ke liye
   (pehli baar "live" hona khud ek log-worthy transition nahi hai, sirf state-mein-wapas-aana
   ya state-se-bahar-jaana hai)
5. Agar account "Scam"/"Spam" flag ho jaaye Meta ki taraf se, yeh ek alag, independent
   audit-entry ke saath track hota hai
6. Har successful change downstream systems ko notify hoti hai (async, generic company-sync
   channel ke through — koi dedicated "yeh WhatsApp-specific hai" consumer nahi mila)
```

**Business impact**: Yeh ek **ongoing relationship** hai, ek one-time setup nahi — status
badalta rehta hai, aur system inhe continuously track karta hai.

### 3.2 — Outbound "App Install" Nudge

**Business flow**:
```
1. Koi internal trigger (jaise enquiry flow, missed-call flow, ya verification flow) ek
   generic "notify this user" API call karta hai, jisme "source" batata hai kaunsa channel
   chahiye
2. Agar source WhatsApp-specific hai (do specific source-codes), system ek WhatsApp
   message-send ki koshish karta hai; kisi bhi doosre allowed source ke liye, yehi API SMS
   bhejta hai — **ek hi entry-point, do channels**
3. Iससे pehle, system multiple layers check karta hai: kya yeh mobile number pehle se
   app-installed device se linked hai (agar haan, nudge skip), kya iss mobile ko already SMS
   bheja gaya hai recently (SMS-side ka apna check), aur phir WhatsApp ka apna
   24-hour frequency check (iss specific mobile ko pichle 24 ghante mein yeh WhatsApp message
   pehle nahi bheja gaya ho)
4. Agar sab checks pass hote hain, ek pre-approved message-template (ya free-text fallback)
   WhatsApp pe bhej diya jaata hai, ek external vendor ke through
5. Agar already bheja ja chuka hai 24 ghante mein, message skip ho jaata hai — spam-prevention
```

**Business impact**: Ek disciplined, rate-limited outreach mechanism, jo SMS ke saath
infrastructure share karta hai — users ko baar-baar same message se irritate nahi karta, aur
IndiaMART ke liye ek hi system dono channels manage karta hai.

### 3.3 — Kyun teesra "WhatsApp toggle" feature yahaan involved nahi hai

Social Contacts KT ke deeper review ke baad yeh confirm hota hai: supplier ke public profile
pe jo "WhatsApp active hai kya" toggle hai (`IS_WHATSAPP_ACTIVE`), woh **teen alag write-paths**
se manage hota hai jinka is doc ke dono features se koi table/code overlap nahi hai — woh sirf
ek boolean flag hai jo batata hai "kya buyers ko iss supplier ka WhatsApp contact-method
dikhna chahiye," na ki koi actual messaging-integration ya sending mechanism. Poora detail
[`Social Contacts Business Doc`](../Social%20Contacts%20KT/Social_Contacts_Business_Doc.md)
mein hai.

---

## 4. Business Rules — Plain Language Mein

1. **Integration record insert-vs-update dono support karta hai** — pehli baar link karna
   ya existing link ko update karna, dono ek hi API se hote hain (`action` parameter se decide
   hota hai).
2. **Integration write sirf internal admin/LMS ke through hoti hai** — supplier khud seedha
   iss specific API ko call nahi karta.
3. **Status-change tracking automatic hai, lekin selective hai** — jab bhi integration ka
   status ek specific range mein change hota hai (jaise "reinstated"/"unlocked" ban jaana, ya
   "locked"/"deleted" ho jaana), system khud ek log-entry bana deta hai — lekin sirf tab jab
   status **actually badla ho** (same status dobara submit karne se duplicate log nahi banta).
4. **Scam/Spam flagging ek special, independent path hai** — agar Meta ki taraf se account
   scam/spam flag ho, system ismein specific comment ke saath record karta hai, status-change
   logging se alag.
5. **Outbound nudges ek strict 24-hour frequency-cap follow karte hain, per mobile number**
   — same "App Install" nudge dobara nahi bheja jaata agar pichle 24 ghante mein pehle se
   successfully bheja gaya ho. Agar pichli koshish **fail** ho gayi thi, woh count nahi hota
   — matlab ek failed attempt ke baad jaldi retry ho sakta hai.
6. **Mobile number format automatically normalize hota hai** — agar `+91` prefix missing ho,
   system khud add kar deta hai bhejne se pehle.
7. **WhatsApp aur SMS same dispatch-point share karte hain** — yeh ek important business
   insight hai: "App Install nudge bhejo" ek hi generic API hai; kaunsa channel (WhatsApp ya
   SMS) use hoga, yeh sirf caller ke diye hue `source` value pe depend karta hai. Do specific
   source-values WhatsApp trigger karte hain, baaki sab allowed sources SMS trigger karte hain.
8. **Nudge bhejne se pehle ek "already app-installed hai kya" check bhi hota hai** — agar
   system ko pata hai ki yeh user already app-installed device se jud chuka hai, nudge poori
   tarah skip ho jaata hai (kyunki install-karo message bhejne ka matlab nahi jab already
   installed hai).

---

## 5. Notifications — Kab Kya Hota Hai

| Kab | Kya hota hai |
|---|---|
| Integration record status change ho (specific ranges — jaise "reinstated"/"locked") | Internal audit-log entry banti hai (supplier ko koi direct notification nahi, yeh internal tracking hai) |
| Meta ki taraf se Scam/Spam flag lage | Internal audit-log entry, alag comment ke saath |
| "App Install" nudge trigger ho aur allowed ho (24hr cap ke andar) | WhatsApp (ya SMS, source ke hisaab se) message bhej diya jaata hai user ko |
| "App Install" nudge trigger ho lekin 24hr ke andar already bheja ja chuka ho | Koi message nahi jaata — silently skip, spam-prevention |
| User already app-installed device se linked ho | Nudge bhejne ki koshish hi nahi hoti |

---

## 6. Edge Cases — Product/Business Perspective se

1. **"Maine WhatsApp integration link kiya, lekin messages nahi bhej pa raha"** — Yeh do
   alag systems hain (§3.1 vs §3.2) — supplier ka apna linked-account (§3.1) aur IndiaMART ka
   apna nudge-sending mechanism (§3.2) do alag cheezein hain. Sirf account link hona kaafi
   nahi, Meta-side verification bhi complete honi chahiye.
2. **"Mujhe baar-baar app-install ka WhatsApp message aa raha hai"** — Agar 24-hour
   frequency-cap kaam kar raha hai, yeh nahi hona chahiye. Ek possible legit reason: pichli
   koshish fail ho gayi thi (jo cap se exempt hai) — agar consistently repeat ho raha hai,
   technical investigation chahiye.
3. **"Meri WhatsApp integration achanak scam maar gayi"** — Yeh Meta ki taraf se ek external
   decision hai, IndiaMART ka apna nahi — system sirf isse track/reflect karta hai.
4. **"Mujhe WhatsApp ki jagah SMS mil raha hai, ya vice-versa, jab maine WhatsApp maanga tha"**
   — WhatsApp vs SMS decision purely uss internal system pe depend karta hai jo nudge trigger
   kar raha hai (kaunsa "source" bheja gaya) — agar galat channel mil raha hai, upstream
   trigger ka source-value check karo, WhatsApp-specific logic mein bug dhundne se pehle.
5. **Integration status "live" pehli baar hona vs. "reinstated" hona — dono same nahi dikhte
   audit-trail mein** — pehli baar live hona apne-aap ek status-change-log entry nahi banata
   (jabki dobara-live-hona, lock ke baad, banata hai) — agar audit history mein "pehli
   activation" missing lage, yeh expected hai, bug nahi.

---

## 7. Quick Summary

- **WhatsApp Business Integration** = supplier ka apna account link karna (ongoing,
  status-tracked relationship, internal-admin-driven), external BSP vendor ke through.
- **App Install WhatsApp Nudge** = IndiaMART ka apna, rate-limited, transactional
  message-sending tool — WhatsApp aur SMS same dispatch-point share karte hain.
- **Social-contact WhatsApp toggle** (buyers ko WhatsApp contact dikhana ya na dikhana) ek
  poori tarah alag, teesra feature hai — dekho Social Contacts KT.
- Dono iss doc ke features independent hain, sirf naam se WhatsApp share karte hain.
- Integration mein Meta/Facebook-side verification/KYC/quality-rating sab track hota hai.
- Nudges 24-hour frequency-cap follow karte hain (per mobile number), spam-jaisa nahi lagte.

---

## See also

- [`WA_Msg_Integration_Technical_Doc.md`](./WA_Msg_Integration_Technical_Doc.md) —
  code-level detail, including the exact multi-layer trigger condition for the outbound nudge
- [`../Social Contacts KT/Social_Contacts_Business_Doc.md`](../Social%20Contacts%20KT/Social_Contacts_Business_Doc.md) —
  the separate WhatsApp-contact-toggle feature on supplier profiles
- [`../GST KT/GST_Business_Doc.md`](../GST%20KT/GST_Business_Doc.md) — depth/structure
  reference this doc follows
- [`../users_domain_product_stories.md`](../users_domain_product_stories.md) —
  "Notifications, Alerts & Preferences" story mein messaging-platform integration pehle bhi
  mention hui hai

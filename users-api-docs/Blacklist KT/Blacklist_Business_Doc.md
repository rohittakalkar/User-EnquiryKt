# User Blacklist & Blacklist Values — Business Doc (Product Perspective)

Yeh doc Blacklist feature ko **business/product nazariye** se samjhata hai — koi code nahi,
sirf "kya hota hai, kyun hota hai, aur supplier/buyer ke liye iska matlab kya hai." Technical
implementation (APIs, DB, queues, code) ke liye
[`Blacklist_Technical_Doc.md`](./Blacklist_Technical_Doc.md) dekho.

**Important upfront finding**: "Blacklist" naam ke neeche actually **teen alag mechanisms**
hain, jo alag-alag purpose serve karte hain lekin naam se overlap lagte hain. Yeh doc sabko
clearly separate karke samjhata hai — confusion na ho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

IndiaMART ko fraud, spam, aur abusive accounts se platform ko protect karna hota hai. Iske
liye teen alag defense-layers hain, jo sab milke ek "multi-layered fraud-defense" banate hain:

1. **Manual Blacklist** — ek admin-maintained list of specific values (mobile number, email,
   ya doosra attribute) jo explicitly flag ki gayi hain kisi fraud/spam incident ki wajah se.
2. **ML Fraud Suspects** — ek machine-learning model ka output — suspicious accounts jo
   system ne khud, behavior patterns dekh ke, detect kiye, bina kisi manual input ke.
3. **Domain-level real-time check** — ek inline, background check jo doosre features (jaise
   catalog-view/business-feed tracking) use karte hain, yeh dekhne ke liye ki koi user
   (mobile/email ke through) pehle se flagged hai ya nahi — real-time, transaction ke waqt hi.

**Bottom line**: Yeh teeno independent hain — manual (human-curated), automated (ML-driven),
aur real-time-enforcement (inline check jo doosre features use karte hain). Koi ek doosre ko
automatically trigger nahi karta.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Internal admin (GLADMIN/BI team)** | Manual blacklist values ka status update karta hai (block/unblock), ya kisi GLID ko explicitly "whitelist" karta hai |
| **ML/Fraud detection system (automated)** | Suspicious accounts ko automatically flag karta hai, background mein, koi manual input ke bina |
| **Doosre platform features** (jaise catalog-view/Business Feed tracking) | Real-time domain-level blacklist-check consult karte hain, apna khud ka processing aage badhane se pehle |
| **Chat/LMS system** | Conversations se mile signals (jaise suspicious mobile/email jo kisi chat mein share hua) blacklist ke against check karwata hai, aur match milne pe internal admin-service ko alert bhejta hai |
| **GLADMIN Blacklist Service (internal, downstream)** | Chat se aaye matches ko receive karta hai — is service ke andar kya hota hai, yeh review scope se bahar hai |

---

## 3. Teeno Mechanisms — Alag-Alag Samjho

### 3.1 — Manual Blacklist Values

**Kya hai**: Ek explicit list of "bad" values — jaise ek mobile number jo pehle fraud mein
use hua tha, ya ek email jo spam ke liye jaana jaata hai.

**Kaun manage karta hai**: Sirf internal admin/BI team (`GLADMIN`, `MAPI` callers) — koi
seller/buyer khud isse touch nahi kar sakta. Yeh sabse tight-access mechanism hai teeno mein
se.

**Business flow**:
```
1. Admin ek existing blacklist-value ka status update karta hai (block/unblock)
   — ya ek specific GLID ko explicitly "whitelist" karta hai (override, poore GLID ke
   saare blacklist-records ke against)
2. Value ke against ek GLUSR_ID (kisne yeh value flag ki thi) aur comments record hote hain
3. Doosre systems (jaise chat-monitoring, catalog-tracking) is value-list ko consult kar
   sakte hain apne khud ke checks ke liye
```

**Interesting finding (confirmed dobara iss deeper pass mein)**: iss API se koi bhi **naya**
blacklist-value add nahi hota, sirf **existing values ka status update** hota hai. Agar
system mein pehle se woh value hi nahi hai, update fail ho jaata hai "No Record Exist" ke
saath — insert nahi hota. Matlab naye blacklist entries kahin aur se aane chahiye — most
likely chat-monitoring system se (dekho §3.4), lekin **yeh conclusively confirm nahi ho paaya
iss review mein ki naye blacklist-values ka original data-entry point exactly kahan hai**.

### 3.2 — ML Fraud Suspects

**Kya hai**: Ek machine-learning model suspicious accounts ko score karta hai
(probability ke saath) aur ek list maintain karta hai — koi bhi supplier/buyer ka current
fraud-suspect status yahan se check kiya ja sakta hai.

**Kaun manage karta hai**: Yeh poori tarah automated hai — koi manual admin-input iss list
mein likhte hue nahi mila code mein. Ek "exception" concept hai jahan koi manually kisi ko
is suspect-list se exempt/excuse kar sakta hai (false-positive handle karne ke liye).

**Business flow**:
```
1. ML model background mein accounts ko evaluate karta hai
2. Suspicious lagne wale accounts ML Fraud Suspects list mein record ho jaate hain
   (probability score ke saath)
3. Agar koi false-positive ho, admin ek "exception" add kar sakta hai us account ke liye
4. Koi bhi caller ek ya ek saath 10 tak accounts ka current fraud-suspect status check
   kar sakta hai (10 se zyada diye jaayein toh baaki silently ignore ho jaate hain)
```

### 3.3 — Ek Doosra, Alag Fraud-Check Path (Naya finding)

Iss deeper pass mein ek **teesra fraud-check path** mila jo pehle document nahi hua tha:
manual-blacklist-read API (`GET /blacklistvalues`) mein agar sirf ek GLID diya jaaye
(value nahi), toh yeh ek **poori tarah alag mechanism** call karta hai — ek database-side
fraud-detection function, jo `ML Fraud Suspects` list se seedha related nahi lagta (alag
database, alag connection).

**Business impact/risk**: Do alag jagah se "yeh account fraud-suspect hai ya nahi" ka jawaab
milta hai — ek `GET /blacklistvalues?glid=...` se, ek `GET /readmlfraud` se. Agar dono same
underlying signal represent karte hain, toh yeh sirf ek naming-confusion hai. Agar alag
signals hain, toh yeh do independent fraud-detection systems hain jo team ko clearly pata
hona chahiye. **Iska exact jawaab code se nahi mil paaya** — team se confirm karna chahiye,
kyunki iska galat interpretation business-critical fraud-decisions ko affect kar sakta hai.

### 3.4 — Chat-se-Aane-Wala Signal (naya-confirm hua mechanism)

**Kya hai**: Jab bhi buyer-seller chat mein koi mobile number ya email share hota hai, ek
background system usse existing blacklist ke against check karta hai.

**Business flow**:
```
1. Chat/LMS system ek message ka mobile/email content background system ko bhejta hai
2. System check karta hai: kya yeh mobile/email pehle se Manual Blacklist mein hai,
   aur "active" status mein hai?
3. Agar haan → ek alert/summary internal GLADMIN Blacklist Service ko forward hota hai
   (sender, receiver, matched value, timestamp ke saath)
4. Agar nahi → kuch nahi hota
```

**Important nuance**: yeh system khud kahin data **save nahi karta** — yeh sirf ek existing
blacklist-value se match check karta hai aur match milne pe ek doosre internal service ko
batata hai. Yeh khud naye blacklist-values create nahi karta — iska matlab §3.1 ka open
question (naye values kahan se aate hain) iss mechanism se bhi resolve nahi hota.

### 3.5 — Domain-level Real-Time Check (inline safety-check)

**Kya hai**: Jab koi buyer kisi seller ka catalog dekhta hai (ya related activity karta hai),
background mein ek check chalta hai — kya buyer blacklisted hai (mobile, alt-mobile, email,
ya email-domain ke through)?

**Business flow**: Yeh koi standalone user-facing feature nahi hai — yeh ek **internal
safety-check** hai jo Business-Feed/catalog-tracking apne pipeline ke andar automatically
consult karta hai, bina kisi alag API ke. Agar buyer blacklisted nikle (ya account disabled
ho, ya mobile invalid format ka ho, ya woh khud IndiaMART employee ho, ya jis product ko
dekha gaya woh out-of-stock ho) — us particular catalog-view event ko silently drop kar diya
jaata hai, track hi nahi hota.

**Business impact**: Yeh ek "invisible" gate hai — koi error nahi dikhta buyer ko, bas uska
activity silently record nahi hota. Iska use-case fraud-prevention ke saath-saath data-quality
(fake/invalid activity ko analytics se bahar rakhna) bhi hai.

**Interesting nuance**: agar buyer ka record hi system mein nahi milta (jaise deleted user),
usse "blocked" nahi maana jaata — uska activity normally track hoti hai. Blocking sirf tab
hoti hai jab koi specific match/flag mile.

---

## 4. Status/State Reference

| Value seen in system | Mechanism | Business meaning |
|---|---|---|
| Blacklist status "active" | Manual Blacklist | Value flagged hai, checks isse block karenge |
| Blacklist status "inactive/other" | Manual Blacklist | Value ab active flag mein nahi hai — sirf "1" status wale hi live checks mein action trigger karte hain |
| Whitelist override (by GLID) | Manual Blacklist | Poore GLID ka blacklist-record explicitly "trusted/whitelisted" mark ho jaata hai, ek doosri workflow se (value-specific update se alag) |
| ML Fraud Suspect + probability score | ML Fraud Suspects | Model ka confidence score ki yeh account suspicious hai |
| Fraud Suspect Exception | ML Fraud Suspects | Admin ne is account ko manually exempt kiya hai — false-positive handling |
| Domain-check "blocked" | Domain-level real-time check | Buyer ka activity is category mein aata hai: blacklisted mobile/email/domain, disabled account, invalid mobile format, internal employee, ya out-of-stock item — koi bhi ek trigger ho toh block |

---

## 5. Business Rules — Plain Language Mein

1. **Manual blacklist-values sirf update ho sakte hain iss admin-API se, naye add nahi ho
   sakte** — agar value system mein hai hi nahi, update "No Record Exist" ke saath fail ho
   jaata hai. Naye entries ka original source alag hai (confirmed nahi iss review mein).
2. **"Whitelist override" ek special path hai** — ek poore GLID ko explicitly "trusted" mark
   kiya ja sakta hai, jo uske saare blacklist-value-records ko affect karta hai ek saath
   (individual value-level update se alag).
3. **ML Fraud suspects fully automated hain** — koi manual "add karo" action nahi mila, sirf
   "exception/exempt karo" action milta hai. Matlab agar koi galat flag hua, usse hataya nahi
   ja sakta, sirf "iss case mein ignore karo" mark kiya ja sakta hai.
4. **Fraud-suspect status ek saath max 10 accounts ke liye check ho sakta hai** — zyada diye
   jaayein toh baaki silently drop ho jaate hain, koi warning nahi milti.
5. **Teeno core mechanisms independent hain, ek doosre ko seedha trigger nahi karte** —
   matlab agar koi value Manual Blacklist mein hai, zaroori nahi ki woh ML Fraud Suspects
   list mein bhi ho, ya vice versa.
6. **`indiamart.com` domain khud kabhi bhi domain-blacklist se block nahi ho sakta** —
   yeh explicitly hardcoded exception hai real-time check mein, taaki internal/company
   emails kabhi galti se block na ho jaayein.
7. **Agar buyer ka record hi nahi milta (jaise deleted account), usse "blocked" nahi maana
   jaata** — real-time check sirf tab block karta hai jab koi specific negative-match mile,
   missing-data ko automatically suspicious nahi maana jaata.
8. **Ek purana blacklist-write API poori tarah dead/deprecated hai** — hardcoded
   "This API is deprecated now" response deta hai, chahe koi bhi input aaye. Yeh abhi bhi
   technically callable hai (route removed nahi hua), lekin kabhi kaam nahi karega.
9. **Chat-se-aaya signal khud kuch save nahi karta, sirf forward karta hai** — yeh matches
   ko ek doosre internal service ko bhejta hai, khud blacklist list mein naya entry create
   nahi karta.

---

## 6. Notifications

| Kab | Kya hota hai |
|---|---|
| Admin ne manual blacklist-value update kiya | Koi email/notification supplier/buyer ko nahi jaati — yeh purely internal admin action hai |
| ML model ne kisi account ko fraud-suspect flag kiya | Koi email/notification user ko nahi jaati — yeh silent, background flagging hai |
| Chat mein blacklisted mobile/email match mila | User ko kuch nahi dikhta — sirf internal GLADMIN Blacklist Service ko alert forward hota hai |
| Buyer ka catalog-view real-time check mein block ho gaya | Buyer ko koi error/message nahi dikhta — activity bas silently track nahi hoti, poori tarah invisible hai user ke liye |
| Koi bhi in teeno mechanisms mein se | **Koi customer-facing email/SMS notification kahin nahi mili iss poore domain mein** — yeh poora area purely internal/backend hai, GST jaisa customer-facing email-flow yahan nahi hai |

---

## 7. Edge Cases — Product/Business Perspective se

1. **"Maine ek account ko blacklist mein dekha, lekin fraud-check pass ho gaya"** — Yeh do
   alag systems ho sakte hain (Manual Blacklist vs ML Fraud Suspects vs stored-function-based
   glid check) — teeno independent hain, ek doosre ko automatically sync nahi karte.
2. **"Naya blacklist-value add karna hai, API error de raha hai"** — Expected hai agar value
   already exist nahi karti — current admin-API sirf UPDATE support karta hai, INSERT nahi.
   Naya value add karne ka actual mechanism iss review mein clearly identify nahi hua —
   product/eng team se confirm karna chahiye agar manual-add ka genuine business-need hai.
3. **"Buyer ka activity track nahi ho raha, koi error bhi nahi dikha"** — Yeh domain-level
   real-time check ka expected/silent behavior ho sakta hai (§3.5) — blacklisted, disabled,
   invalid-mobile, employee, ya out-of-stock — in mein se koi bhi wajah ho sakti hai.
4. **Dead/legacy code bhi mila**: ek purana blacklist-write API hai jo ab poori tarah
   deprecated hai (hardcoded "This API is deprecated now" response deta hai) — agar koi
   purani documentation ya integration abhi bhi isse reference kare, woh kaam nahi karegi,
   lekin ek clear "deprecated" message milega (silent failure nahi).
5. **Do alag fraud-check APIs hain jo naam se same lagte hain** (§3.3) — team ko clearly
   pata hona chahiye `GET /blacklistvalues?glid=...` aur `GET /readmlfraud` mein kya farak
   hai, warna galat data pe decision ban sakta hai.
6. **Koi bhi automated cleanup/re-verification job nahi mila iss domain mein** — GST jaisa
   koi daily background-reconciliation job Blacklist domain mein exist nahi karta (confirmed
   absent, na ki assume kiya gaya) — matlab agar koi data drift ho jaaye, usse fix karne ka
   koi automated safety-net nahi hai.

---

## 8. Quick Summary

- "Blacklist" ek naam ke neeche kai systems hain: manual values, ML fraud-suspects, ek
  alag glid-based fraud-check function, chat-se-aane-wala signal, aur domain-level
  real-time inline check.
- Manual blacklist sirf update hota hai admin se, naya add karne ka path clear nahi hai iss
  review mein.
- ML fraud-suspects poori tarah automated hai, sirf exceptions manual hoti hain.
- Do alag "fraud check for a glid" answers milte hain do alag APIs se — inka exact
  relationship confirm nahi hua, business-critical open question hai.
- Domain-level real-time check ek invisible, silent gate hai — buyer ko kabhi pata nahi
  chalta agar unka activity block ho gaya.
- Poore domain mein koi customer-facing email/notification nahi hai — sab kuch internal hai.
- Koi automated daily-reconciliation cron nahi hai iss domain mein (GST ke ulat).
- Ek purana blacklist-write API completely dead/deprecated hai, lekin abhi bhi callable hai.

---

## See also

- [`Blacklist_Technical_Doc.md`](./Blacklist_Technical_Doc.md) — code-level detail (APIs, DB
  tables, queries, Kafka, consumers)
- [`../trust_verification_compliance_product_overview.md`](../trust_verification_compliance_product_overview.md) —
  Blacklist ka role bigger Trust & Compliance story mein
- [`../GST KT/GST_Business_Doc.md`](../GST%20KT/GST_Business_Doc.md) — same-depth reference
  doc for the GST domain, same trust/verification family

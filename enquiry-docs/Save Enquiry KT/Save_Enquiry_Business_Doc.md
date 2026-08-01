# Save Enquiry — Business Doc (Product Perspective)

Yeh doc "Save Enquiry" feature ko **business/product nazariye** se samjhata hai — koi code
nahi, sirf "kya hota hai, kyun hota hai, aur buyer/supplier ke liye iska matlab kya hai."
Technical implementation (APIs, DB, queues, code) ke liye
[`Save_Enquiry_Technical_Doc.md`](./Save_Enquiry_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

IndiaMART ek B2B marketplace hai jahan buyer, supplier ke products/services mein interest
dikhata hai — isko "Enquiry" (query) kehte hain. **Save Enquiry** woh core write-endpoint hai
jo har naye enquiry ko system mein permanently create karta hai, chahe woh kahin se bhi aaya
ho — website form, mobile app (iOS/Android), WhatsApp, product-catalog page, ya kisi third-party
integration se.

Yeh sirf ek "form submit" nahi hai. Iske peeche ek poora business process chalta hai jiska
maksad hai:

1. **Buyer ki requirement ko supplier tak turant pahunchana** — jitna jaldi supplier ko lead
   milegi, utni jaldi woh respond kar sakta hai, jo buyer experience aur conversion dono ke
   liye critical hai.
2. **Spam/abusive content filter karna** — enquiry ka description (requirement text) ek "Banned
   Keyword" check se guzarta hai, taaki abusive/spam/fraud content wale enquiry supplier tak na
   pahunchein.
3. **Enquiry ko sahi jagah route karna** — har enquiry ek "destination" decide karta hai (kya
   yeh direct supplier ko dikhegi, kya yeh review ke liye rukegi, kya yeh reject ho jayegi) — yeh
   fraud-prevention aur quality-control ka core mechanism hai.
4. **Downstream systems ko turant inform karna** — jaise hi enquiry save hoti hai, kai teams
   (Lead Management, Personalization/recommendation engine, Preferred-Channel/WhatsApp) ko
   turant notify kiya jaata hai taaki woh apna kaam (jaise WhatsApp par bhejna, buyer ko
   recommendation dikhana) shuru kar sakein.

**Bottom line**: Save Enquiry woh single choke-point hai jahan se har buyer-supplier
interaction shuru hoti hai. Agar yeh fail ho, buyer ki requirement kabhi supplier tak nahi
pahunchti.

---

## 2. Kaun-kaun involved hai iss process mein

| Kaun | Kya role hai |
|---|---|
| **Buyer** | Apni requirement (enquiry/query) submit karta hai — website, app, WhatsApp, etc. se |
| **Supplier (Receiver)** | Enquiry ka target — jise yeh lead milni hai |
| **Frontend/App clients (WEB, IOS, ANDROID, IMOB, PNSBLENQ, FCP, MY, etc.)** | Alag-alag channels jo apna khud ka "ModID" bhej ke enquiry submit karte hain — har channel ka thoda alag treatment hota hai |
| **BAN API (internal content-moderation service)** | Enquiry ke description ko scan karta hai spam/abusive keywords ke liye |
| **Lead Management System (LMS)** | Enquiry ko ek "lead" ke roop mein track karta hai — supplier ke response, buyer-response, enrichment data yahan jaata hai |
| **Personalization Engine** | Buyer ke behavior ko track karta hai taaki future recommendations improve ho sakein |
| **Preferred-Channel Worker (WhatsApp/BL-transfer)** | Decide karta hai enquiry supplier ko kis channel (email/WhatsApp/app) se pahunchani hai |
| **Kibana/monitoring team** | Har save-enquiry request ka detailed log dekhti hai debugging/monitoring ke liye |

---

## 3. QueryDestination — Enquiry ka "kahan jaana hai" wala decision

Har save hui enquiry ek **QueryDestination** code carry karti hai. Yeh business perspective se
sabse important cheez hai kyunki iska seedha asar hota hai ki enquiry supplier ko turant
dikhegi, waiting mein jayegi, ya reject ho jayegi.

| QueryDestination | Business meaning | Supplier ko dikhti hai? |
|---|---|---|
| `1` | Normal, seedha supplier tak jaane wali enquiry | Haan, turant |
| `2` | Kisi wajah se "waiting"/hold mein — jaise banned-keyword match ho gaya | Nahi, review/waiting mein |
| `3` | Ek "processed" / dusre type ki valid enquiry (jaise bounced ya waiting-flow se reclassify hui) | Haan, alag treatment ke saath |
| `5` | Ek intermediate state jo aakhir mein `3` mein normalize ho jaati hai downstream | Depends |
| `8` | Export-marked — seller ne Indian leads receive karne ka flag set kiya hai (implemented 02-Jul-2025) | Haan, lekin `3` jaisi treat hoti hai response mein |
| `9` | HRS (High Risk Supplier?) marked enquiry (implemented 04-Jun-2025) | Haan, lekin `3` jaisi treat hoti hai response mein |

**Business rule jo yaad rakhne layak hai**: Codes `8` aur `9` sirf **internal classification**
ke liye hain — jab response buyer/supplier-facing system ko wapas jaata hai, dono `3` mein
convert kar diye jaate hain. Yaani agar koi support ticket aaye "meri enquiry destination `8`
kyun thi lekin frontend pe `3` dikh rahi hai," yeh expected behavior hai, bug nahi.

Codes `8` aur `9` ka **exact business meaning conclusively code se nahi mila** — sirf comments
se pata chalta hai (HRS-marked, export-mark seller). **[INFERRED — Enquiry/Trust team se
confirm karo exact business definition]**.

---

## 4. Main Business Flows (step-by-step, no code)

### Flow A — Buyer normal enquiry submit karta hai (happy path)

```
1. Buyer product/service page pe "Get Best Price" / "Send Enquiry" jaisa button dabata hai
2. Frontend/app buyer ki requirement (description), sender details, receiver (supplier) details
   ke saath ek request bhejta hai
3. System check karta hai: description khaali toh nahi, sender/receiver ID missing toh nahi
4. Agar description "templated" (ek generic auto-filled text) nahi hai, toh content-moderation
   (BAN API) check hota hai spam/abusive content ke liye
5. Agar sab sahi hai -> enquiry DB mein save ho jaati hai, ek unique Query ID milta hai
6. Turant background mein: Lead Management, Personalization, aur Preferred-Channel (WhatsApp
   routing) teeno systems ko is naye enquiry ke baare mein bataya jaata hai
7. Buyer ko response milta hai success ke saath (Query ID, destination code)
```

**Business impact**: Yeh sabse common path hai — bulk of IndiaMART ka lead-generation isi flow
se hota hai. Speed critical hai kyunki jitni jaldi supplier ko pata chale, utni jaldi wo
respond karega.

### Flow B — Enquiry ka description banned/spam keyword match karta hai

```
1. Buyer requirement likhta hai jo generic template jaisi nahi hai
2. System BAN API ko call karta hai description check karne ke liye
3. Agar keyword banned nikla -> enquiry ka InForceDestination "2" (waiting) set ho jaata hai
   aur waiting-reason ek special code (-999) le leta hai
4. Enquiry phir bhi save hoti hai (queryid milta hai), lekin supplier ko turant nahi dikhti —
   review/waiting state mein chali jaati hai
```

**Business impact**: Yeh spam-prevention hai. Bina isse, koi bhi abusive ya spam content wala
enquiry direct supplier tak pahunch jaata, jo platform ki trust ko nuksaan pahunchata.

### Flow C — Enquiry "templated" description ke saath aati hai

```
1. Kai baar buyer khud kuch nahi likhta — app/website default ek generic text bhar deta hai
   (jaise "Hi, I am interested in X. Kindly send details")
2. System pehchanta hai ki yeh ek known "template" pattern hai
3. Aisi enquiries ke liye BAN API call hi **skip** kar diya jaata hai
```

**Business impact**: Yeh ek performance optimization + false-positive-avoidance hai — generic
templated text ko baar-baar external moderation API pe bhejna waste hai, aur templated text
kabhi bhi genuinely abusive nahi hota.

### Flow D — Product-detail auto-fill (Product Catalog se enquiry)

```
1. Buyer kisi product-catalog page se enquiry karta hai lekin product ka naam explicitly nahi
   bhejta — sirf ek product-reference-ID (modrefid) bhejta hai
2. System background mein product ka naam DB se fetch karta hai (parallel goroutine mein, saath
   hi sender aur receiver details bhi fetch ho rahe hote hain)
3. Fetched product name automatically enquiry ke saath attach ho jaata hai
```

**Business impact**: Buyer ko manually product name type nahi karna padta — jo naam catalog
mein already hai wahi enquiry mein use hota hai, consistency ke liye.

### Flow E — Kuch zaroori field missing hai (Sender ya Receiver ID)

```
1. Agar request mein Sender ID ya Receiver (supplier) GL-ID missing/invalid hai
2. System enquiry save nahi karta — turant ek "error" response deta hai (queryid = -2,
   destination = 3) bina kisi DB round-trip ke
```

**Business impact**: Yeh ek guardrail hai jo galat/incomplete requests ko database tak
pahunchne se pehle hi reject kar deta hai — system resources bachata hai aur galat data DB mein
jaane se rokta hai.

### Flow F — Backup/failure recovery (queue-sync backup mechanism)

```
1. Agar enquiry DB mein save ho gayi (Query ID mil gaya), lekin downstream queues (Lead
   Management/Personalization/WhatsApp-routing) mein se koi bhi push fail ho jaaye ya panic aa
   jaaye — ya khud DB-save hi timeout/error ke saath fail ho jaaye —
2. System is poore request ko ek backup JSON file mein save kar leta hai server pe
3. Ek daily cron (`QueryParse`) is backup file ko dobara padhta hai aur poori enquiry ko
   `saveEnquiry` endpoint pe wapas replay (POST) kar deta hai — effectively poora Flow A
   dobara chalta hai us backed-up enquiry ke liye
4. Ek doosra manual-trigger cron (`EnquiryParseManual`) bhi similar kaam karta hai, likely
   ad-hoc/backfill situations ke liye (exact trigger-mechanism confirm karna baaki hai team se)
```

**Business impact**: Yeh ek genuine safety-net hai — agar DB-save ya downstream-notify kisi
bhi wajah se fail ho jaaye, enquiry data lost nahi hota. Ek automated daily cron usse pick karke
next-day replay kar deta hai, bina kisi manual intervention ke. (Iska matlab yeh nahi ki
notification turant ho jaati hai — agar original request fail hui thi, replay agle din tak
delay ho sakta hai.)

---

## 5. Business Rules — Plain Language Mein

Yeh sab rules hain jo Save Enquiry process ko govern karte hain, jo maine actual code padh ke
nikale hain:

1. **Sender ID aur Receiver (Supplier) GL-ID dono mandatory hain** — dono missing ya invalid
   (negative/`-1`) hone par enquiry reject ho jaati hai bina DB tak pahunche.
2. **Requirement (Description) khaali nahi ho sakti** — kuch specific channels (jinke paas
   Sender ID hai, ya jo IMOB/ANDROID app se aa rahe hain) ke liye description mandatory hai.
3. **PNSBLENQ channel automatically "InForceDestination = 1" set hota hai** — yeh ek special
   channel hai jo hamesha normal/direct-visible enquiry banata hai.
4. **Templated (auto-generated) description ke liye spam-check skip hota hai** — sirf genuine,
   buyer-typed text hi BAN API se guzarta hai.
5. **Naam se "Mr./Mrs./Dr./Ms." titles automatically hata diye jaate hain** — sirf clean naam
   store hota hai.
6. **Mobile number automatically formatted hota hai country ke hisaab se** — agar India ka
   number hai aur 10-digit hai, `+91-` prefix apne aap lag jaata hai. Agar "mobile number" ya
   "null" jaisa junk text hai, khaali kar diya jaata hai.
7. **Field lengths automatically trim hoti hain** — jaise sender ka naam max 80 characters,
   email max 60, address max 150, description max 4000 — koi bhi lamba text silently trim ho
   jaata hai, error nahi aata.
8. **Non-standard/invalid characters (UTF-8 issues) automatically clean ho jaate hain** — agar
   description ya koi text field corrupt-encoding ke saath aaye, system usse saaf karke aage
   badhta hai, lekin isse ek "UTF flag" internally set hota hai jo logging mein track hota hai.
9. **QueryDestination codes 8 aur 9 buyer/supplier-facing response mein hamesha 3 dikhte hain**
   — yeh internal classification ke liye hain, external systems ko simplified value milta hai.
10. **Duplicate/parallel downstream failures se poori enquiry fail nahi hoti** — agar Lead
    Management ya Personalization mein push fail ho jaaye, enquiry phir bhi successfully save
    maani jaati hai (Query ID mil chuka hota hai), sirf downstream sync backup mechanism
    (Flow F) trigger hoti hai.
11. **System ek "panic-safe" design follow karta hai** — agar processing ke beech kahin bhi
    unexpected crash ho jaaye, buyer ko ek generic "service unavailable" (503) response milta
    hai, lekin poori request details internally backed-up ho jaati hain taaki kuch lost na ho.

---

## 6. Notifications / Downstream Awareness — Kab Kaun Ko Pata Chalta Hai

Save Enquiry khud koi email nahi bhejta buyer/supplier ko (yeh alag "Enrich/Finish Enquiry"
jaisi flows ka kaam hai) — iska primary output "downstream systems ko turant batana" hai:

| Kab | Kise pata chalta hai | Kaise |
|---|---|---|
| Enquiry successfully save hui (Query ID mila) | Lead Management System | Background queue push |
| Enquiry successfully save hui | Personalization/Recommendation engine | Background queue push |
| Enquiry successfully save hui | Preferred-Channel worker (jo aage WhatsApp/BL-transfer decide karta hai) | Background queue push (`ENQ_DATA`) |
| Enquiry ek "FENQ" (waiting/review) category mein hai | FENQ-specific downstream flow (jo aage waiting-enquiry ko manage karta hai) | Wahi `ENQ_DATA` message ke andar embedded flag |
| Downstream push fail ho jaaye (ya DB-save khud fail ho jaaye) | **Koi turant notification nahi** — silently backup file ban jaati hai, jo daily `QueryParse` cron dwara agle din automatically replay ho jaati hai | Backup JSON file → daily cron replay |
| Buyer/supplier ko koi email/SMS | **Save Enquiry ke scope mein nahi hai** — yeh sirf enquiry "create" karta hai; notification alag downstream flows ka kaam hai (jaise LMS ya alag notification-service se) | — |

---

## 7. Edge Cases — Product/Business Perspective se

1. **"Maine enquiry submit ki, success bhi mila, lekin supplier bola usse koi lead nahi mili"**
   — Pehle check karo QueryDestination `2` (waiting/banned-content) toh nahi thi. Agar
   description mein koi flagged keyword tha, enquiry supplier tak turant nahi pahunchi hogi.

2. **"Meri enquiry ka description automatically chhota ho gaya"** — Yeh expected hai agar
   description 4000 characters se zyada tha — system silently trim karta hai, error nahi deta
   (business rule #7).

3. **"Same enquiry do baar system mein dikh rahi hai"** — Ho sakta hai original request
   backup-file mein gaya ho (kisi transient failure ki wajah se) aur daily cron ne usse
   replay kar diya ho, jabki original request bhi kisi tarah se successfully process ho gayi
   thi — dono scenarios (original + replay) ek genuine duplicate create kar sakte hain agar
   original failure sirf partial tha. Support ko backup-file/cron-replay logs check karne
   chahiye (Flow F dekho).

4. **"Mobile number format expected se alag dikh raha hai"** — System automatically country-code
   prefix karta hai based on sender ka detected country — yeh manual entry override kar sakta
   hai kuch edge cases mein (business rule #6).

5. **"QueryDestination 8/9 kya hote hain, main confuse hoon"** — Yeh purely internal
   classification codes hain (export-marked / HRS-marked), buyer/supplier-facing response mein
   hamesha `3` ban jaate hain. Inka exact business definition team se confirm karna baaki hai
   (section 3).

6. **Bahut saare alag "channels" (ModID) same endpoint use karte hain** — WEB, IOS, ANDROID,
   IMOB, PNSBLENQ, FCP, MY, aur bhi kai — har channel ka thoda alag treatment hota hai (jaise
   mobile-number formatting sirf IOS/ANDROID ke liye alag hai). Agar koi naya channel/app-version
   issue report kare, pehle uska `ModID` confirm karo.

---

## 8. Quick Summary — Ek Line Mein Har Cheez

- Save Enquiry = har buyer-supplier interaction ka starting point, poore IndiaMART lead-gen ka core.
- Spam/abusive content BAN API se filter hota hai, templated text ke liye skip ho jaata hai.
- QueryDestination decide karta hai enquiry supplier ko turant dikhegi ya waiting/hold mein jayegi.
- Save hone ke turant baad Lead Management, Personalization, aur Preferred-Channel (WhatsApp)
  teeno ko background mein inform kiya jaata hai.
- Agar downstream push fail ho, ek backup-file + daily-cron safety-net enquiry ko lost hone se
  bachata hai.
- Save Enquiry khud koi buyer/supplier-facing email nahi bhejta — sirf enquiry create + downstream
  ko notify karta hai.

---

## See also

- [`Save_Enquiry_Technical_Doc.md`](./Save_Enquiry_Technical_Doc.md) — same flows, code-level
  detail (APIs, DB tables, queries, RabbitMQ/Kafka, consumers, crons)

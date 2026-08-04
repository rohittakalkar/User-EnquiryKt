# Rating (Supplier Rating) — Business Doc (Product Perspective)

**Note**: yeh doc sirf **Rating** (buyer supplier ko 1-5 star deta hai) cover karta hai.
**Rating Usefulness** ("was this rating helpful?" vote) ek alag concept hai — uska apna
KT folder already ban chuka hai:
[`../Rating Usefulness KT/Rating_Usefulness_Business_Doc.md`](../Rating%20Usefulness%20KT/Rating_Usefulness_Business_Doc.md).
Confuse mat karo dono ko — dono ek hi write-controller share karte hain code-level pe, lekin
business meaning bilkul alag hai.

Yeh doc Rating feature ko **business/product nazariye** se samjhata hai — koi code nahi.
Technical implementation ke liye [`Rating_Technical_Doc.md`](./Rating_Technical_Doc.md)
dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Jab ek buyer kisi supplier se interact karta hai (enquiry, order, conversation), IndiaMART
unse poochta hai: **is supplier ko 1-5 stars do, ek comment likho, aur bataao specific
cheezein (Response time, Quality, Delivery) achi thi ya nahi, chaahe toh photos bhi lagao.**

Yeh rating baad mein supplier ke public profile pe dikhti hai, taaki **doosre buyers** decide
kar sakein ki is supplier pe trust karna hai ya nahi.

**Business impact**: Rating ek platform-wide trust-signal hai. Ek high-rated supplier ko
zyada enquiries milti hain, buyers unse jaldi contact karte hain. Isliye rating fake nahi
honi chahiye, aur genuine feedback fast, reliably, aur bina spam ke publish hona chahiye.

**Yeh kyun simple nahi hai**: ek raw rating seedha "publish" nahi ho jaati. Usse pehle
**kaafi saare parallel checks** chalte hain:
- Content check (abusive language, PII leakage nahi honi chahiye)
- Confirm karna ki yeh ek real buyer-supplier connection se aayi hai (fake rating nahi) —
  isse "Matchmaking check" bolte hain
- Ek **poora alag daily fraud-scan cron** bhi chalta hai jo suspicious patterns dhoondta hai
  (jaise: ek hi din mein ek supplier ko bahut saari ratings aayi jo ek dusre se connected
  buyers se aayi ho — "ring" jaisa pattern)
- Supplier ka average star-rating recalculate hota hai
- Ek risk-scoring system ko bhi signal bheja jaata hai
- Supplier ko notify kiya jaata hai
- Yeh sab **bina buyer ko wait karwaye** hota hai — buyer submit karte hi "done" dekh leta
  hai, baaki sab background mein chalta hai

Yeh feature IndiaMART ke sabse **bade multi-system feature** mein se ek hai — ek single
rating submit, kam se kam **9 alag background processes** ko trigger kar sakta hai, plus
do dedicated daily-batch crons jo purani/pending ratings pe extra checks chalate hain.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Buyer** | Rating deta hai (star + comment + thumbs up/down on parameters + photos) |
| **Supplier** | Rating dekhta hai, reply kar sakta hai |
| **Content moderation system** | Har rating/reply ko abuse/PII ke liye check karta hai submit hote hi — CBS (banned-keyword) + PII-detection + ML model, teeno layers |
| **Matchmaking check** | Real-time verify karta hai ki buyer-supplier genuinely connected the (contact-book se) |
| **Fraud/Suspect-flag cron** | Daily batch job jo pending ratings ko dobara scan karta hai, external fraud-graph system se buyer-buyer connections bhi cross-check karta hai (ring-detection) |
| **Seller-risk system** | Har naye rating ka signal ek risk-scoring pipeline ko bhejta hai (Kafka ke through) |
| **Internal admin (GLADMIN)** | Kuch cases mein rating ko manually approve/disable/re-review kar sakta hai |
| **Lead-Management System (LMS)** | Do alag raaston se rating-data receive karta hai — internal sales/lead-tracking ke liye |
| **Other buyers** | Rating padh sakte hain, "helpful" vote kar sakte hain *(alag feature, iss doc mein nahi)* |

---

## 3. Rating ki "Visibility States" — supplier/buyer ke liye iska matlab

Har rating ek "display status" carry karta hai jo decide karta hai woh publicly dikhegi ya
nahi:

| State | Business meaning |
|---|---|
| **Pending/Hidden (default, jab submit hoti hai)** | Rating abhi-abhi aayi hai, moderation/checks ho rahe hain — kisi ko nahi dikhti |
| **Visible** | Sab checks pass ho gaye, ab public profile pe dikh rahi hai |
| **Disabled/Hidden (matchmaking-fail)** | System confirm nahi kar paaya ki buyer-supplier sach mein connected the, isliye kabhi visible nahi hui |
| **Disabled/Hidden (fraud-cron flag)** | Daily fraud-scan cron ne is rating ko suspicious paaya (jaise: ek "ring" ka hissa lagi, ya same-IP pattern) — hide ho gayi |
| **Banned content** | Abusive/inappropriate content ya PII (phone/email) detect hua, block ho gayi |

**Important**: rating submit hote hi **turant hidden state mein jaati hai**, publish nahi
hoti. Buyer ko turant "thank you" dikh sakta hai, lekin actual visibility background checks
ke baad hi milti hai — yeh design se hai, taaki abusive/fake content kabhi live na ho.

**Do-layer fraud protection**: ek layer real-time hai (matchmaking check, submit hote hi
turant chalta hai), doosri layer daily-batch hai (fraud/suspect cron, jo raat/din mein
pending ratings ko dobara scan karta hai aur zyada sophisticated cross-buyer pattern
dhoondta hai jo real-time mein possible nahi). Iska matlab hai ek rating jo pehle din
"visible" lag rahi thi, agle din cron ke through bhi hide ho sakti hai agar suspicious
pattern mile.

---

## 4. Main Business Flows (step-by-step, no code)

### Flow A — Buyer ek naya rating submit karta hai

```
1. Buyer supplier ko stars deta hai, comment likhta hai, "Response/Quality/Delivery" pe
   thumbs up/down deta hai, chaahe toh photos bhi attach karta hai
2. System check karta hai ki yeh pehli baar hai ya buyer pehle bhi is supplier ko rate kar
   chuka hai (system ismein farak samajhta hai — "naya" vs "existing relationship update")
3. Rating hidden state mein save ho jaati hai
4. Background mein turant, sabhi parallel:
   - Content-moderation check shuru hoti hai (abuse/PII scan, 3 tarah ke checks)
   - Buyer-supplier ka genuine connection verify hota hai
   - Product/category context enrich hota hai (kis product ke baare mein rating thi)
   - Supplier ka star-summary recalculate hota hai
   - Seller-risk system ko bhi signal jaata hai
   - Supplier ko notify kiya jaata hai
   - Yeh info do alag raaston se ek internal Lead-Management System ko bhi bheji jaati hai
5. Sab pass hone ke baad, rating "visible" ban jaati hai
6. Agle 1-2 din mein, ek daily fraud-scan cron bhi is rating ko dobara dekhta hai — agar
   koi suspicious pattern mila (jaise ek hi din mein ek supplier ke against connected
   buyers se multiple ratings), toh yeh rating waapas hide ho sakti hai
```

**Business impact**: Buyer ka experience instant hai (submit → done), lekin actual public
visibility ek do-layer trust-verified pipeline se guzarti hai — bina buyer ko wait karaye,
aur fraud detection sirf real-time tak limited nahi, agle din bhi dobara check hota hai.

### Flow B — Rating "Update" hoti hai (admin review / re-check / matchmaking-triggered disable)

```
1. Koi bhi internal system (admin, matchmaking-check, enrichment-consumer, ya fraud-cron)
   POST karta hai ek "admin/system action" ke saath (naya insert nahi, existing record pe
   action)
2. Yeh path ek saath do cheezein trigger karta hai:
   - Supplier ka star-summary turant recalculate hota hai (yeh confirm hota hai — is baar
     doubt nahi hai ki stale reh jaayega)
   - Ek notification-adjacent event bhi trigger hota hai
```

**Business impact**: Yeh ek single, consistent path hai jise **admin, matchmaking-check, aur
fraud-cron teeno use karte hain** jab bhi kisi rating ka display-status change karna ho —
matlab chahe wajah kuch bhi ho (fake connection, fraud-pattern, manual admin action), star-
summary hamesha turant refresh hota hai, kabhi stale nahi rehta is specific path se.

### Flow C — Supplier apni rating ka reply karta hai

```
Supplier apna comment likhta hai rating ke against, same submit-API dobara call hoti hai —
is baar rating value change nahi hota, sirf reply add hota hai. Yeh reply bhi phir se
content-moderation se guzarta hai (abuse/PII check us reply text pe bhi hota hai).
```

**Business impact**: Two-way conversation ka feel deta hai — supplier apna side bata sakta
hai agar rating unfair lage. Lekin supplier ka reply bhi utni hi scrutiny se guzarta hai
jitni buyer ka original comment — koi bhi party abusive/PII-leaking text nahi likh sakti.

### Flow D — Purani ratings archive ho jaati hain

```
Agar ek buyer-supplier pair ke beech multiple ratings hain (jaise time ke saath relationship
continue hui), sirf latest wali "live" rakhi jaati hai — purani automatically archive ho
jaati hain, aur uske baad supplier ka star-summary bhi turant refresh hota hai.
```

**Business impact**: Supplier ka displayed rating hamesha **most recent** relationship
status reflect karta hai, purana outdated feedback confuse nahi karta.

### Flow E — Koi rating padhta hai (buyer ya supplier ka profile page)

```
Supplier profile page load hoti hai -> saari visible ratings, average star count, aur
"reasons" ka breakdown (kitne logon ne Response/Quality/Delivery pe thumbs-up diya) dikhta hai
```

**Business impact**: Yeh woh jagah hai jahan trust-signal actually kaam aata hai — naya buyer
decide karta hai supplier ko contact karna hai ya nahi, isi data ke basis pe. Yeh ek high-
traffic page hai (supplier profile baar-baar dekha jaata hai), isliye response speed matter
karti hai.

### Flow F — Roz-roz ka Fraud/Suspect Scan (naya, iss pass mein discover hua)

```
1. Har din, ek automated job saari "abhi bhi pending" (last 1-2 din ki) ratings ko dekhta hai
2. Har rating ke liye, ek fraud-detection system check karta hai:
   - Kya yeh rating suspicious pattern match karti hai? (jaise same buyer/supplier IP se
     multiple ratings, banned product mention, ya connected-buyers ka "ring")
   - Agar supplier ko ek hi din mein kai buyers se ratings mili hain, system check karta hai
     ki kya woh saare buyers aapas mein bhi connected hain (jaise ek fraud "ring" ho) — agar
     3 ya zyada aise connections milte hain, saari ratings us group ki suspicious flag ho
     jaati hain
3. Jo bhi suspicious lage, uska display-status update ho jaata hai (hide ya restrict), aur
   supplier ka star-summary turant refresh hota hai
```

**Business impact**: Yeh ek **doosri safety-net** hai matchmaking-check ke upar — matchmaking
sirf "kya yeh connection real hai" check karta hai turant, jabki yeh daily scan zyada
sophisticated fraud-patterns pakadta hai jo sirf real-time mein dikhna mushkil hote hain
(jaise organized rating-rings). Isliye kabhi-kabhi ek rating jo pehle din theek lagi thi,
agle din hide ho sakti hai — yeh koi bug nahi, yeh ek intentional deeper-scan hai.

---

## 5. Business Rules — Plain Language Mein

1. **Rating hamesha 1-5 ke beech honi chahiye** — koi aur value accept nahi hoti.
2. **Buyer khud ko rate nahi kar sakta** — buyer aur supplier ka ID same nahi ho sakta.
3. **Rating submit hote hi hidden hoti hai, publish nahi** — moderation/verification pass
   karne ke baad hi visible hoti hai.
4. **Content moderation teen alag layers use karta hai**: banned-keyword check (products/
   categories/comments), PII-leak detection (phone/email), aur ek ML-based abuse-detection
   model — teeno mil ke decide karte hain content dikhana hai ya nahi.
5. **Supplier ka reply bhi utni hi scrutiny se guzarta hai jitna buyer ka original comment.**
6. **Content moderation, matchmaking-check, enrichment, seller-risk-signal sab parallel/
   background mein chalte hain** — buyer ko inka wait nahi karna padta.
7. **Photos attach kiye ja sakte hain rating ke saath** — optional, lekin agar diye jaayein
   toh unka apna storage aur reference hota hai, aur unhe bhi admin review kar sakta hai.
8. **"Reasons" (Response/Quality/Delivery) optional hain per-rating** — buyer chaahe toh
   sirf stars de sakta hai bina inpe thumbs-up/down diye.
9. **Purani ratings automatically archive hoti hain** jab naya rating aata hai usi
   buyer-supplier pair ke beech — sirf latest live rehti hai.
10. **Rating ke saath do alag internal Lead-Management sync hote hain** — yeh sirf trust-
    display ke liye nahi, IndiaMART ke internal sales/lead-tracking systems ke liye bhi
    useful data hai.
11. **Har naye rating ka ek signal Seller-Risk scoring system ko bhi jaata hai** — yeh rating
    ko sirf display ke liye use nahi karta, balki ek supplier ke overall "risk profile" mein
    bhi contribute karta hai.
12. **Fraud-detection do layers mein hoti hai**: turant (matchmaking check, real-time) aur
    roz ek batch scan (fraud-cron, deeper pattern detection) — dono independently kaam karte
    hain, ek dusre ko replace nahi karte.
13. **Jab bhi koi rating disable/hide/re-review hoti hai (chahe admin ho, matchmaking ho, ya
    fraud-cron ho), supplier ka star-summary turant refresh hota hai** — yeh ek consistent
    guarantee hai is domain mein.

---

## 6. Edge Cases — Product/Business Perspective se

1. **"Maine 5-star rating diya, phir bhi supplier ka overall score nahi badha"** — Ho sakta
   hai rating abhi bhi moderation/verification pipeline mein pending ho. Turant reflect nahi
   hoti — kuch minute lag sakte hain.

2. **"Meri rating kabhi dikhi hi nahi"** — Ho sakta hai matchmaking-check fail hui ho (system
   confirm nahi kar paaya ki aap sach mein iss supplier se connected the), content
   abusive/inappropriate flag hua ho, ya agle din fraud-scan cron ne isse suspicious paaya ho.

3. **"Meri rating pehle dikh rahi thi, ab gayab hai"** — Yeh matchmaking-fail se different
   hai. Yeh ho sakta hai roz-roz ke fraud-scan cron ki wajah se ho — jo pehle din
   pass ho gayi thi woh agle din deeper cross-buyer-pattern check mein flag ho sakti hai.
   Yeh intentional hai, bug nahi.

4. **"Maine 2 baar rate kiya, dono dikhne chahiye"** — Nahi, sirf latest wali dikhti hai,
   purani automatically archive ho jaati hai (Flow D). Yeh intentional hai, taaki rating
   list clutter na ho purane feedback se.

5. **"Supplier ne reply kiya lekin mujhe reply nahi dikh raha"** — Reply bhi content-
   moderation se guzarta hai (Flow C) — agar supplier ne kuch aisa likha jo abusive ya
   PII-leaking lage, reply bhi hide ho sakta hai.

6. **Star-rating aur "helpful" votes do alag cheezein hain** — Rating khud ek 1-5 star hai;
   "helpful" ek doosra concept hai jahan doosre buyers kisi already-diye-hue rating pe vote
   karte hain — apna alag KT folder hai
   ([`Rating Usefulness KT`](../Rating%20Usefulness%20KT/Rating_Usefulness_Business_Doc.md)).

---

## 7. Quick Summary

- Rating = buyer ka 1-5 star + comment + reasons + optional photos, supplier ke against.
- Submit turant hota hai, lekin publish hone se pehle 3-layer content moderation + real-time
  matchmaking-verify background mein chalte hain.
- Ek doosra, daily-batch fraud-detection cron bhi pending ratings ko dobara scan karta hai —
  deeper cross-buyer fraud patterns (jaise "rings") pakadne ke liye.
- Sirf latest rating per buyer-supplier pair live rehti hai, purani archive hoti hain.
- Rating data internal Lead-Management system ko do alag raaston se jaata hai, aur ek Seller-
  Risk scoring system ko bhi — sirf public display ke liye nahi.
- Jab bhi koi rating disable/re-review hoti hai (kisi bhi wajah se), supplier ka star-summary
  turant refresh hota hai — is guarantee mein koi known gap nahi hai.

---

## See also

- [`Rating_Technical_Doc.md`](./Rating_Technical_Doc.md) — same flows, code-level detail
- [`../Rating Usefulness KT/Rating_Usefulness_Business_Doc.md`](../Rating%20Usefulness%20KT/Rating_Usefulness_Business_Doc.md) — "helpful?" vote feature, alag concept
- [`../Matchmaking KT/Matchmaking_Technical_Doc.md`](../Matchmaking%20KT/Matchmaking_Technical_Doc.md) — matchmaking-check ka doosra side
- [`../ratings_reviews_product_overview.md`](../ratings_reviews_product_overview.md) —
  bigger Ratings & Reviews story (isme Social Reviews bhi cover hoti hain, jo yahan scope se
  bahar hai)
- [`../full_read_write_picture.md`](../full_read_write_picture.md) — supplier-rating ka
  poora architecture trace

# Bizfeed & Recommendation — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai — koi code nahi, sirf "kya hota hai, kyun
hota hai, aur supplier/buyer ke liye iska matlab kya hai." Code ke liye
[`Bizfeed_And_Recommendation_Technical_Doc.md`](./Bizfeed_And_Recommendation_Technical_Doc.md)
dekho.

**Scope note — kya-kya combined hai aur kyun**:

| Concept | Is folder mein? | Kyun |
|---|---|---|
| User Activity | Haan | Buyer Activity ke saath ek hi write-API aur ek hi Cassandra-table share karta hai (`activity_type_id` se differentiate hota hai) |
| Catalog View | Haan | Usi pipeline ka downstream-processing-stage hai (Kafka-driven `workerUserBusinessFeeds`) |
| Buyer Activity | Haan | Yeh pipeline ka entry-point hai |
| Bizfeed CV Hide/Unhide | Haan | Alag write-controller hai, lekin **isi pipeline ke output-table** (`glusr_usr_biz_feeds`) ko directly modify karta hai — isliye same-domain |
| Recommendation (Action-Items) | Haan | Ek dashboard-aggregator hai jo Bizfeed-data ko (aur 4 doosre services ko) combine karke ek "recommended actions" list banata hai |
| **Latitude & Longitude Details** | **Nahi** | Genuinely alag feature/table hai — dekho [`../Location Update KT/`](../Location%20Update%20KT/) |
| **Last Seen** | **Nahi** | Codebase mein kahin bhi nahi mila (koi table, controller, ya field iss naam se exist nahi karta) — agar yeh feature future mein banaya jaaye, tab hi documented ho sakta hai |

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Jab bhi ek buyer IndiaMART pe kuch browse karta hai (product-search, category-browse,
product-view), yeh interaction capture hoti hai aur end mein supplier ko dikhayi jaati hai —
"aapke product ko is buyer ne dekha." Yeh poora ek chain hai jiska maksad hai:

1. **Buyer ka har meaningful interaction capture ho** — kya dekha, kis keyword se search
   kiya, kis supplier ka product tha, buyer kahan se hai.
2. **Supplier ko real-time visibility mile apne catalog ke against** — koi bhi supplier chahta
   hai jaane ki unka product kaun dekh raha hai, taaki wo follow-up kar sake.
3. **Supplier apna feed personalize kar sake** — har activity relevant nahi hoti, isliye
   individual items hide/unhide karne ka control diya gaya hai.
4. **Supplier ko sirf "kya hua" na dikhe, balki "ab kya karna chahiye"** — Recommendation
   dashboard Bizfeed-data ko baaki business-health-signals (product-listing-completeness,
   credits, unread-messages, GST/PAN completeness) ke saath combine karke ek actionable
   "next-steps" list deta hai.

**Bottom line**: Yeh poora "engagement-loop" hai — buyer ka browsing-behavior capture hota
hai, supplier ko uske against actionable-insights milte hain, aur supplier apna profile
improve karne ke liye guided-recommendations paata hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Buyer** | Browse/search karta hai IndiaMART pe — activity generate hoti hai, jo end mein supplier ko dikhti hai |
| **Supplier** | Bizfeed dekhta hai, individual items hide/unhide karta hai, Recommendation dashboard follow karta hai |
| **Internal-tools (GLADMIN, LEAP)** | Activity-events ko machine-to-machine likh sakte hain (yeh ek trusted, mostly-internal ingestion endpoint hai) |
| **4 doosri internal services** (product-listing, credits, unread-messages, other-details) | Recommendation dashboard inhe parallel mein call karke ek combined "action items" list banata hai |

---

## 3. Buyer Interaction → Bizfeed Entry — kaise banta hai (states nahi, ek pipeline hai)

Iss feature mein GST jaisa ek "verification-status" table nahi hai, lekin har buyer-activity
event ek chhoti si internal "kya yeh Bizfeed mein dikhega ya nahi" decision-chain se guzarta
hai — supplier ko yeh samajhna zaroori hai ki har activity Bizfeed mein nahi dikhti:

| Check | Agar fail ho jaaye, toh kya hota hai |
|---|---|
| Buyer khud apna hi product dekh raha ho (self-view) | Bizfeed mein nahi dikhega — yeh expected/intentional hai |
| Buyer disabled/blacklisted ho (mobile/email/domain), ya wo product out-of-stock ho, ya buyer khud ek internal-employee ho | Bizfeed mein nahi dikhega — yeh ek fraud/noise-prevention filter hai |
| System detect kare ki iss buyer-seller pair ke beech recently ek "meeting" log hua tha | Bizfeed mein nahi dikhega — [INFERRED — confirm with team: shayad in-person/offline meeting ho chuka signal hai, isliye dobara notify karna redundant maana gaya] |
| Buyer/Seller ka ID bahut lamba ho (invalid data) | Bizfeed mein nahi dikhega, silently drop |
| Sab kuch pass ho jaaye | Ek Bizfeed entry ban jaati hai, supplier ko dikhne lagti hai |

**Business rule jo yaad rakhne layak hai**: Agar koi supplier bole "buyer ne mera product
dekha tha, phir bhi Bizfeed mein nahi dikha" — yeh **bug nahi hai zaroori nahi**, upar diye
gaye 4 filters mein se koi bhi trigger ho sakta hai. Support-team ko yeh samjhana chahiye ki
Bizfeed sirf "genuine, non-duplicate, non-fraud" activity dikhata hai, har raw-click nahi.

---

## 4. Main Business Flows (step-by-step, no code)

### Flow A — Buyer koi activity karta hai, supplier ke Bizfeed mein entry aati hai

```
1. Buyer koi activity perform karta hai IndiaMART pe (search, product-view, category-browse)
2. Yeh activity ek raw-event ke roop mein capture hoti hai (glusr_id, activity-type, keyword,
   category, buyer-location, supplier jiska product dekha gaya, etc.)
3. Background mein ek processing-stage yeh raw-event ko dekhta hai:
   - Kya buyer ne khud apna product dekha? -> skip
   - Kya buyer fraud/blocked/employee hai? -> skip
   - Kya recently ek offline-meeting log hui thi? -> skip
   - Baaki sab theek? -> ek Bizfeed-entry generate hoti hai, buyer ka city/state bhi attach
     hota hai
4. Supplier apna Bizfeed dekhta hai — "yeh buyer ne aapka product dekha, is city se"
```

**Business impact**: Real-time-ish buyer-signal se supplier-facing-insight tak ka poora
data-pipeline — raw-clickstream se personalized-feed-entry tak. Supplier ko genuine buyer-
interest ka signal milta hai, noise/fraud filter-out ho jaata hai.

### Flow B — Supplier apne Bizfeed ke items ko manage karta hai (hide/unhide)

```
1. Supplier apna Bizfeed dekhte hue kisi entry ko "not useful" maanta hai
2. Supplier ek "hide" action leta hai us entry (specific buyer, specific din) ke against
3. System us match (supplier + buyer + din) wali saari entries ko hide kar deta hai
4. Supplier chahe toh baad mein "unhide" kar sakta hai, wapas dikhne lagti hai
```

**Business impact**: Feed ko personalize karne ka tarika — supplier apne liye relevant
signals hi dekhna chahta hai, sabko nahi.

**Important caveat jo product/support ko pata hona chahiye**: agar ek hi buyer ne ek hi din
mein supplier ke 2 alag products dekhe (2 alag activity-events, 2 Bizfeed-entries), toh
hide/unhide **dono ko ek saath affect karega** — system individual-entry level pe hide nahi
kar sakta, sirf (supplier+buyer+din) level pe. Agar supplier bole "maine sirf ek hide kiya tha,
dusra bhi gayab ho gaya" — yeh expected behavior hai, granular control abhi nahi hai.

### Flow C — Supplier apna Recommendation/Action-Items dashboard dekhta hai

```
1. Supplier apna dashboard kholta hai
2. System ek saath 4-5 alag internal-services ko parallel call karta hai:
   - Unka company-detail/GST/PAN status kya hai
   - Unka product-listing kitna complete hai (photo/price missing, rejected products, etc.)
   - Unke kitne unread buyer-messages/enquiries hain
   - Unka Bizfeed activity-count kya hai (kitne buyers ne dekha)
   - (agar free-tier supplier nahi hai) unka credit-lapse status kya hai
3. Sab responses combine hoke ek priority-ranked list banti hai — "pehle yeh karo, phir yeh"
4. Agar koi ek service down/slow ho, uska corresponding item khaali (count=0) dikh jaata hai —
   poora dashboard fail nahi hota
```

**Business impact**: Supplier ko sirf raw-data nahi, ek guided "next steps" experience milta
hai — kya complete karna hai apna profile behtar banane ke liye, priority ke hisaab se.
Free-tier suppliers ko thoda alag priority-order dikhta hai (unke liye credit/Bizfeed-items
kam priority pe, product/GST-completeness zyada priority pe) — [INFERRED — confirm with team:
likely business-logic hai ki free-tier ke liye monetization-items relevant nahi].

---

## 5. Business Rules — Plain Language Mein

1. **Buyer apna hi product dekhe toh Bizfeed entry nahi banti** — "aapke product ko buyer X ne
   dekha" ka matlab nahi banta agar buyer aur supplier ek hi ho.
2. **Fraud/blocked/employee buyers ki activity Bizfeed mein kabhi nahi dikhti** — yeh
   trust-signal ko clean rakhne ka mechanism hai, taaki supplier sirf genuine interest dekhe.
3. **Hide/unhide sirf ek (supplier, buyer, date) combination pe apply hota hai, individual
   activity-entry pe nahi** — agar same buyer ne same din multiple products dekhe, hide/unhide
   dono ko saath affect karta hai.
4. **Hide karna sirf visibility-toggle hai, data delete nahi hoti** — supplier chahe toh wapas
   unhide kar sakta hai.
5. **Recommendation-dashboard multiple services se parallel data-fetch karta hai** — sirf
   Bizfeed nahi, product-listing, company-details (GST/PAN), unread-messages, aur credits sab
   combine hote hain ek hi response mein.
6. **Free-tier vs paid supplier ke liye Recommendation-priorities alag hain** — free-tier
   suppliers ko credit-related item dikhta hi nahi (skip ho jaata hai), aur baaki items ki
   priority-ranking bhi shift ho jaati hai.
7. **Recommendation-dashboard kabhi "broken" response nahi deta** — agar koi ek underlying
   service fail ho jaaye, uska corresponding item bas khaali/zero dikh jaata hai, poora
   dashboard error nahi deta.
8. **Ek specific "PayIM/Monetization" recommendation-item hamesha khaali dikhta hai** — yeh
   structurally response mein present hai, lekin code mein iske liye koi actual data-source
   wire nahi hua abhi — [INFERRED — confirm with team: ho sakta hai future-use ke liye
   reserved ho, ya ek dead/incomplete integration ho].

---

## 6. Notifications — Supplier ko kab pata chalta hai

| Kab | Kya notification jaata hai |
|---|---|
| Buyer ne activity ki aur ek Bizfeed-entry ban gayi | **Koi explicit email/push notification code mein nahi mila** — supplier ko yeh apna Bizfeed dashboard khud check karke pata chalta hai (pull-based, push-based nahi) — [INFERRED — confirm with team agar koi separate notification-service alag se yeh trigger karti hai] |
| Supplier ne kisi entry ko hide/unhide kiya | Koi notification nahi — yeh ek silent, supplier-initiated action hai |
| Recommendation-dashboard update hota hai | Koi notification nahi — yeh purely on-demand/pull-based hai, jab supplier dashboard kholta hai tab live-compute hota hai |

**Note**: GST domain (dekho [`../GST KT/GST_Business_Doc.md`](../GST%20KT/GST_Business_Doc.md))
mein email-notifications ka rich system hai; Bizfeed & Recommendation domain mein aisi koi
proactive-notification nahi mili in-code — yeh purely "supplier khud dashboard check kare"
model hai.

---

## 7. Edge Cases — Product/Business Perspective se

1. **"Maine feed-item hide kiya, phir bhi dikh raha hai"** — Ho sakta hai match-criteria
   (glusrid+buyerid+date) exact match na ho raha ho, ya kisi doosre din/buyer ke liye
   separate entry ho jo hide nahi hui.
2. **"Buyer ne clearly mera product dekha, Bizfeed mein nahi dikha"** — 4 possible reasons:
   buyer khud supplier hai (self-view), buyer fraud/blocked/employee hai, ek "meeting-bypass"
   signal trigger hua, ya buyer/seller ID data-issue ki wajah se reject hui. Yeh sab
   intentional filters hain, bugs nahi.
3. **"Mera Recommendation-dashboard slow load ho raha hai"** — Expected hai agar koi ek
   internal-service (jinme se yeh parallel-fetch karta hai) slow respond kar rahi ho — poora
   dashboard uss ek slowest-service ke response-time se bound hai.
4. **"Maine ek hi buyer ke 2 activities mein se ek hide ki, dusri bhi gayab ho gayi"** —
   Expected hai, dekho Flow B ka caveat — hide/unhide granularity item-level nahi hai.
5. **Free-tier supplier ko kuch recommendation-items kam priority pe dikhte hain** —
   intentional business-logic hai, bug nahi.

---

## 8. Quick Summary

- Bizfeed & Recommendation = buyer-activity-capture → fraud/self-view-filtered
  processing → supplier-facing-feed (hide/unhide-capable) → Recommendation-dashboard-
  aggregator.
- User Activity + Catalog View + Buyer Activity ek hi pipeline hain, ek hi table share karte
  hain.
- Har buyer-activity Bizfeed mein nahi dikhti — 4 silent filters hain (self-view, fraud/block,
  meeting-bypass, data-validity).
- Hide/Unhide granularity (supplier+buyer+date) level pe hai, individual-item level pe nahi.
- Recommendation-dashboard 5 services ka parallel-fetch hai — kisi ek ke slow/fail hone se
  poora dashboard degrade ho sakta hai, lekin crash nahi hota.
- Koi proactive email/push-notification nahi mili is domain mein — sab kuch pull-based hai.
- Lat/Long aur Last Seen is domain ka hissa nahi hain (dekho scope-note upar).

---

## See also

- [`Bizfeed_And_Recommendation_Technical_Doc.md`](./Bizfeed_And_Recommendation_Technical_Doc.md) — code-level detail
- [`../Location Update KT/Location_Update_Business_Doc.md`](../Location%20Update%20KT/Location_Update_Business_Doc.md) —
  unrelated, separately-documented Lat/Long feature
- [`../GST KT/GST_Business_Doc.md`](../GST%20KT/GST_Business_Doc.md) — depth/structure reference this doc was rebuilt against

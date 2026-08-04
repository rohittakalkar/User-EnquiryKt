# Social Review — Business Doc (Product Perspective)

**Note**: yeh doc **Social Reviews** cover karta hai — yeh
[`../Rating KT/`](../Rating%20KT/Rating_Business_Doc.md) (native supplier star-rating) se
**bilkul alag** feature hai, aur [`../Social Contacts KT/`](../Social%20Contacts%20KT/Social_Contacts_Business_Doc.md)
(supplier ke social-media contact links jaise Facebook/Instagram/Twitter handles) se bhi alag
hai — teeno "social"/"review" naam se milte-julte lagte hain lekin teeno independent systems
hain, alag tables, alag purpose.

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Social_Review_Technical_Doc.md`](./Social_Review_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Bahut saare suppliers ka pehle se ek **Google Business Profile** ya **Facebook Page** hota
hai, jispe unhe pehle se genuine customer-reviews mil chuki hoti hain — koi IndiaMART join
karne se pehle bhi. Social Review feature supplier ko yeh external reviews **link karne aur
IndiaMART profile pe bhi dikhane** deta hai — bina unhe naye sirse review collect karne ki
zaroorat pade.

**Business impact**: Naya supplier jo abhi IndiaMART pe start kar raha hai, unke paas
IndiaMART-native ratings kam ho sakti hain (Rating feature se), lekin agar unka Google
Business pe 4.5-star, 200-reviews ka history hai, woh bhi IndiaMART profile pe dikh sakta
hai — instant credibility.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Apna Google/Facebook account link karta hai (ya external integration ke through link hota hai) |
| **External integration/sync system** (IndiaMART ke bahar ka system, in teen repos ke bahar) | Google/Facebook se actual reviews periodically fetch karke IndiaMART ko API-call ke through bhejta hai — account details aur individual reviews dono |
| **Buyer** | Supplier profile pe yeh social reviews dekhta hai |

---

## 3. Business Flow (step-by-step, no code)

### Flow A — Account link/relink

```
1. Supplier (ya external sync job unki taraf se) apna Google Business ya Facebook Page
   account link karta hai — platform, account-ID, company-name/email/phone/address/website,
   total-rating-count, average-rating, location-ID, refresh-token sab ek saath submit hota hai
2. Ek "organic" flag bhi hai — matlab yeh reviews genuinely organic hain ya kisi paid/boosted
   mechanism se aayi hain, yeh differentiate hota hai
3. Agar supplier ke Google Business pe multiple locations hain (multi-branch), har location
   ke liye alag account-link ho sakta hai — same platform (jaise Google) ke andar
4. IMPORTANT business rule: ek platform (jaise Google) ke liye ek waqt pe sirf EK location
   "active/displayed" reh sakti hai per supplier. Jab supplier ek nayi location link/connect
   karta hai, purani location automatically "inactive" ho jaati hai — turant, same operation
   mein. Yeh ek "switch active location" jaisa behavior hai, add-on nahi.
```

**Business impact**: Agar ek supplier ne pehle Location-A connect ki thi aur ab Location-B
connect karta hai, buyer ko sirf Location-B ki reviews dikhengi — Location-A ki reviews
turant hide ho jaayengi (delete nahi, bas hidden). Support ticket "meri purani location ki
reviews gayab ho gayi" ka yehi jawaab hai — expected switch-behavior hai.

### Flow B — Account update (existing link ke fields modify karna)

```
1. Supplier/sync-job existing account-record ke specific fields update kar sakta hai (jaise
   naya average-rating, naya total-count, ya display-status khud change karna)
2. Sirf jo fields bheje jaate hain, wahi update hote hain — baaki untouched rehte hain
3. Agar koi record match nahi milta (galat GLUSR/platform/location combination), system
   politely bata deta hai "not found," error nahi deta
```

**Business impact**: Incremental sync possible hai — poora account record dobara bhejne ki
zaroorat nahi, sirf jo change hua wahi bhejo.

### Flow C — Content sync (individual reviews)

```
1. Ek baar account link ho jaaye (aur uski location "active" ho), actual individual reviews
   (content) sync hoti hain — batch mein, ek baar mein max 20 reviews tak
2. Har review ka apna star-rating, display-status, comment, reviewer-naam, aur reply (agar
   supplier ne reply kiya ho Google/Facebook pe) store hota hai
3. Agar wahi review dobara sync ho (same platform-review-ID), purana record update ho
   jaata hai, duplicate nahi banta
4. Reviews hamesha ussi location ke against store hoti hain jo currently "active" hai us
   platform ke liye — agar location switch ho chuki hai (Flow A point 4), naya sync automatically
   naye active-location record se jud jaata hai
```

**Business impact**: Har supplier ka Google-review-count/history accurately reflect hota hai,
duplicate ya stale entries ka risk nahi.

### Flow D — Display (buyer-facing read)

```
1. Buyer supplier profile dekhta hai -> ya toh ek specific platform (jaise sirf Google) ki
   reviews dikhti hain, paginated (scroll/load-more support ke saath)
2. Ya, agar koi specific platform na chuna ho, saari platforms (Google + Facebook, etc.) ki
   top-N latest reviews ek saath dikhti hain, har platform ke apne account-summary
   (average rating, total count) ke saath
3. "Aur reviews hain" indicator bhi milta hai (HAS_MORE) — batata hai ki abhi jo dikha, poora
   set nahi hai, aur load ho sakta hai
```

**Business impact**: Buyer ko ek clean, per-platform-organized view milta hai, chahe woh ek
platform dekhna chahe ya sab.

---

## 4. Business Rules — Plain Language Mein

1. **Account aur Content dono alag APIs hain** — pehle account link hota hai, phir uske
   under reviews (content) sync hoti hain — do-step process.
2. **Ek supplier multiple social-platform accounts link kar sakta hai** — Google ke saath
   Facebook bhi, dono independent records hain.
3. **Multi-location support hai, lekin ek waqt pe ek hi "active"** — supplier ke Google
   Business pe kai locations ho sakti hain, alag records store hote hain unke liye, lekin
   sirf ek (sabse-recently-connected) location ki reviews buyer ko dikhti hain per platform.
   Yeh ek naya, gold-standard-depth pass mein confirm hua business-rule hai jo pehle wale
   shallow doc mein clearly nahi likha tha.
4. **Content sync batch mein hoti hai, max 20 reviews ek baar mein** — bade sync-jobs ko
   chunks mein todna padta hai.
5. **Duplicate reviews automatically merge ho jaati hain** — same platform-review-ID dobara
   aaye toh update, naya record nahi.
6. **Content hamesha "active" location se linked hoti hai** — agar master-record ID diya na
   ho, system khud dhoondh leta hai kaun-si location currently active hai us supplier +
   platform ke liye, aur reviews usi se jodta hai. Agar koi active-account exist nahi karta,
   sync fail ho jaata hai ("pehle account link karo").
7. **Koi content-moderation nahi hai** — yeh already Google/Facebook pe moderated content
   hai, IndiaMART dobara scan nahi karta (Rating feature ke ulat, jahan har rating abuse/PII
   check se guzarti hai).
8. **`social_refresh_token` sirf write ke liye hai, kabhi buyer/read side pe wapas nahi
   bheja jaata** — yeh ek security/privacy consideration hai, is data ko leak hone se bachata
   hai.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Meri Google reviews IndiaMART pe nahi dikh rahi"** — Check karo account pehle link
   hua hai ya nahi (Flow A), phir content-sync hui ya nahi (Flow C) — dono alag steps hain,
   ek se doosra automatically nahi hota.
2. **"Maine ek nayi location connect ki, purani location ki reviews gayab ho gayi"** — Yeh
   expected hai (Flow A point 4) — ek platform ke liye ek waqt pe sirf ek location active
   reh sakti hai, purani automatically inactive (hidden) ho jaati hai, delete nahi.
3. **Social Reviews aur native Rating alag dikh sakti hain same buyer ko** — yeh do
   independent systems hain, koi automatic merge/dedupe nahi hota inke beech.
4. **Social Reviews aur Social Contacts confuse mat karo** — Social Contacts supplier ke
   social-media *profile links* (jaise Instagram handle) store karta hai, Social Reviews
   external *ratings/reviews* store karta hai — alag features, alag docs
   ([`Social_Contacts_Business_Doc.md`](../Social%20Contacts%20KT/Social_Contacts_Business_Doc.md)).

---

## 6. Quick Summary

- Social Review = external (Google/Facebook) reviews ko IndiaMART profile pe link/display
  karna.
- Do-step process: Account link → Content sync.
- Multi-location supported hai, lekin per-platform sirf ek location ek waqt pe "active"/
  displayed rehti hai — nayi connect hone pe purani automatically hide ho jaati hai.
- Koi moderation nahi (already-moderated external content).
- Native Rating feature se bilkul independent — alag tables, alag purpose.
- Purely synchronous request/response feature hai — koi background job/queue/notification
  system supplier ko is process ke beech mein disturb nahi karta (Google Chat/email jaisa
  koi Notifications table applicable nahi, GST jaisa — is wajah se yeh section GST-parity ke
  liye is doc mein zaroori nahi laga, dekho Technical Doc section 5 jahan iski explicit
  re-verification hai).

---

## See also

- [`Social_Review_Technical_Doc.md`](./Social_Review_Technical_Doc.md) — code-level detail
- [`../Rating KT/Rating_Business_Doc.md`](../Rating%20KT/Rating_Business_Doc.md) — native
  supplier star-rating, alag system
- [`../Social Contacts KT/Social_Contacts_Business_Doc.md`](../Social%20Contacts%20KT/Social_Contacts_Business_Doc.md) — supplier ke social-media contact/profile links, alag system
- [`../ratings_reviews_product_overview.md`](../ratings_reviews_product_overview.md) —
  dono (Rating aur Social Review) ka combined product-overview

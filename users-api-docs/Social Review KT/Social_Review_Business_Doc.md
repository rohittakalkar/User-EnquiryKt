# Social Review — Business Doc (Product Perspective)

**Note**: yeh doc **Social Reviews** cover karta hai — yeh
[`../Rating KT/`](../Rating%20KT/Rating_Business_Doc.md) (native supplier star-rating) se
**bilkul alag** feature hai, chahe dono "reviews/ratings" ki tarah lagte hain. Dono independent
systems hain, alag tables, alag purpose.

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
| **Supplier** | Apna Google/Facebook account link karta hai |
| **External integration job** (IndiaMART ke bahar ka system, in teen repos ke bahar) | Google/Facebook se actual reviews periodically sync karke IndiaMART ko bhejta hai |
| **Buyer** | Supplier profile pe yeh social reviews dekhta hai |

---

## 3. Business Flow (step-by-step, no code)

### Account linking
```
1. Supplier apna Google Business ya Facebook Page account link karta hai
2. System account-level summary bhi store karta hai — total rating count, average rating,
   location-ID (agar multiple locations hain)
3. Ek "organic" flag bhi hai — matlab yeh reviews genuinely organic hain ya kisi paid/boosted
   mechanism se aayi hain, yeh differentiate hota hai
4. Agar supplier account unlink kare, record delete nahi hota — "deactivate" hota hai
   (history preserve rehti hai)
```

### Content sync
```
1. Ek baar account link ho jaaye, actual individual reviews (content) sync hoti hain
   — batch mein, ek baar mein max 20 reviews tak
2. Har review ka apna star-rating, display-status, aur reply (agar supplier ne reply
   kiya ho Google/Facebook pe) store hota hai
3. Agar wahi review dobara sync ho (same platform-review-ID), purana record update ho
   jaata hai, duplicate nahi banta
```

### Display
```
Buyer supplier profile dekhta hai -> saari (ya latest) social reviews, platform-wise ya
saari-milake, dikhti hain, saath mein account-level summary (average rating, total count)
```

---

## 4. Business Rules — Plain Language Mein

1. **Account aur Content dono alag APIs hain** — pehle account link hota hai, phir uske
   under reviews (content) sync hoti hain — do-step process.
2. **Ek supplier multiple social-platform accounts link kar sakta hai** — Google ke saath
   Facebook bhi, dono independent records hain.
3. **Multi-location bhi supported hai** — agar supplier ke Google Business pe multiple
   locations hain, har location ka apna alag account-record ho sakta hai.
4. **Unlink karna "delete" nahi, "deactivate" hai** — history preserve hoti hai.
5. **Content sync batch mein hoti hai, max 20 reviews ek baar mein** — bade sync-jobs ko
   chunks mein todna padta hai.
6. **Duplicate reviews automatically merge ho jaati hain** — same platform-review-ID dobara
   aaye toh update, naya record nahi.
7. **Koi content-moderation nahi hai** — yeh already Google/Facebook pe moderated content
   hai, IndiaMART dobara scan nahi karta (Rating feature ke ulat, jahan har rating abuse/PII
   check se guzarti hai).

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Meri Google reviews IndiaMART pe nahi dikh rahi"** — Check karo account pehle link
   hua hai ya nahi (Step 1), phir content-sync hui ya nahi (Step 2) — dono alag steps hain,
   ek se doosra automatically nahi hota.
2. **"Maine account unlink kiya, phir bhi reviews dikh rahi hain kahin"** — Deactivate ek
   soft-delete hai, poora data turant purge nahi hota — expected hai.
3. **Social Reviews aur native Rating alag dikh sakti hain same buyer ko** — yeh do
   independent systems hain, koi automatic merge/dedupe nahi hota inke beech.

---

## 6. Quick Summary

- Social Review = external (Google/Facebook) reviews ko IndiaMART profile pe link/display
  karna.
- Do-step process: Account link → Content sync.
- Koi moderation nahi (already-moderated external content).
- Native Rating feature se bilkul independent — alag tables, alag purpose.

---

## See also

- [`Social_Review_Technical_Doc.md`](./Social_Review_Technical_Doc.md) — code-level detail
- [`../Rating KT/Rating_Business_Doc.md`](../Rating%20KT/Rating_Business_Doc.md) — native
  supplier star-rating, alag system
- [`../ratings_reviews_product_overview.md`](../ratings_reviews_product_overview.md) —
  dono (Rating aur Social Review) ka combined product-overview

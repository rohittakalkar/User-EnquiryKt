# Rating Usefulness — Business Doc (Product Perspective)

**Note**: yeh doc **Rating Usefulness** ("was this rating helpful?" vote) cover karta hai —
yeh **Rating** (1-5 star khud) se alag concept hai. Rating ke liye
[`../Rating KT/Rating_Business_Doc.md`](../Rating%20KT/Rating_Business_Doc.md) dekho. Dono
features same underlying API (`/supplierrating`) internally reuse karte hain, lekin business
purpose bilkul alag hai.

Yeh doc business/product perspective se hai — code ke liye
[`Rating_Usefulness_Technical_Doc.md`](./Rating_Usefulness_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Jab ek buyer supplier profile page pe ratings padh raha hai, unhe kabhi-kabhi lagta hai
"yeh rating genuinely useful thi" ya "yeh rating fake/abusive lag rahi hai." Rating
Usefulness feature isi feedback-on-feedback ko capture karta hai — jaise Amazon/Flipkart pe
"Was this review helpful? 👍/👎."

**Do tarah ka signal capture hota hai ek hi button se**:
1. **Helpful vote** — "haan, yeh rating useful thi mere liye"
2. **Abuse/report vote** — "yeh rating genuine nahi lagti / abusive hai"

**Business impact**: Yeh crowd-sourced quality-control hai. Agar bahut saare buyers ek
particular rating ko "abuse" mark karte hain, IndiaMART ko signal milta hai ki us rating ko
manually review karna chahiye. Aur "helpful" count high-quality, genuinely useful ratings ko
upar highlight karne mein kaam aa sakta hai (jaise sabse helpful review sabse upar dikhana).

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Koi bhi buyer jo rating padh raha hai** | Vote karta hai — helpful ya abuse |
| **Original rating dene wala buyer** | Iska role sirf itna hai ki unki rating ke against vote pad raha hai — woh khud vote process mein involved nahi |
| **Internal moderation/review team** | High abuse-count wali ratings ko flag/review kar sakti hai *(exact workflow iss review ke scope se bahar)* |

---

## 3. Main Business Flow (step-by-step, no code)

```
1. Buyer supplier profile page pe ek rating padhta hai
2. Buyer decide karta hai: yeh rating helpful thi, ya suspicious/abusive lagi
3. Buyer vote karta hai (helpful ya report/abuse) — ek hi rating pe ek buyer sirf EK baar
   vote kar sakta hai
4. System yeh vote record karta hai
5. Turant, us specific rating ke total helpful-count aur abuse-count ko recalculate kiya
   jaata hai
6. Yeh naye counts wapas us rating ke record mein update ho jaate hain — taaki jab koi bhi
   agli baar wo rating dekhe, updated counts saath mein dikhein
```

**Business impact**: Yeh ek real-time feedback loop hai — koi bhi supplier-profile page load
turant latest helpful/abuse counts dikhati hai, delayed batch-process nahi.

---

## 4. Business Rules — Plain Language Mein

1. **Sirf 2 valid votes hain**: "helpful" ya "abuse/report" — beech ka koi option nahi.
2. **Ek buyer ek rating pe sirf ek baar vote kar sakta hai** — dobara try karne pe system
   bolta hai "Record already exists," koi error nahi dikhata jo confuse kare, bas silently
   ignore kar deta hai duplicate vote ko.
3. **Vote karne ke liye bhi ek "authorized caller" list hai** — yeh feature random external
   callers se open nahi hai, sirf whitelisted internal/app sources se aa sakta hai (jaisa
   Rating submit karne ke liye bhi tha).
4. **Helpful/Abuse counts turant, synchronously update hote hain** — buyer vote karte hi,
   background mein turant original rating record update ho jaata hai, koi lambi delay nahi.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine dobara vote karne ki koshish ki, kuch hua hi nahi"** — Expected hai. System
   duplicate votes ko silently ignore karta hai (per buyer, per rating, sirf ek vote count
   hota hai).

2. **"Rating ka helpful-count profile page pe turant nahi dikha"** — Ho sakta hai
   background-update process mein thoda lag ho, lekin design se yeh near-real-time hona
   chahiye — agar consistently delay ho raha hai, technical investigation chahiye.

3. **"Kya main apni khud ki rating ko helpful mark kar sakta hoon?"** — Business rules mein
   koi explicit self-vote-block nahi mila iss review mein — worth confirming with product
   team ki yeh intentional gap hai ya edge case jo miss ho gaya.

---

## 6. Quick Summary

- Rating Usefulness = "yeh rating helpful thi?" 👍/👎 vote, kisi doosre buyer dwara.
- Ek buyer, ek rating pe, ek hi vote.
- Vote hone ke turant baad, us rating ke helpful/abuse counts update ho jaate hain.
- Yeh Rating (star-rating) se bilkul alag concept hai, lekin technically same write-API
  internally reuse karta hai.

---

## See also

- [`Rating_Usefulness_Technical_Doc.md`](./Rating_Usefulness_Technical_Doc.md) — code-level
  detail
- [`../Rating KT/Rating_Business_Doc.md`](../Rating%20KT/Rating_Business_Doc.md) — Rating
  (star-rating) ka business doc, jispe yeh feature depend karta hai
- [`../ratings_reviews_product_overview.md`](../ratings_reviews_product_overview.md) —
  bigger Ratings & Reviews story

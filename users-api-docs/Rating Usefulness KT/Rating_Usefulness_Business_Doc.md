# Rating Usefulness — Business Doc (Product Perspective)

**Note**: yeh doc **Rating Usefulness** ("was this rating helpful?" vote) cover karta hai —
yeh **Rating** (1-5 star khud) se alag concept hai. Rating ke liye
[`../Rating KT/Rating_Business_Doc.md`](../Rating%20KT/Rating_Business_Doc.md) dekho. Dono
features same underlying API (`/supplierrating`) internally reuse karte hain, lekin business
purpose bilkul alag hai.

Yeh doc business/product perspective se hai, code nahi — code ke liye
[`Rating_Usefulness_Technical_Doc.md`](./Rating_Usefulness_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Jab ek buyer supplier profile page pe ratings padh raha hai, unhe kabhi-kabhi lagta hai
"yeh rating genuinely useful thi" ya "yeh rating fake/abusive lag rahi hai." Rating
Usefulness feature isi feedback-on-feedback ko capture karta hai — jaise Amazon/Flipkart pe
"Was this review helpful? Yes/No."

**Do tarah ka signal capture hota hai ek hi button se**:
1. **Helpful vote** — "haan, yeh rating useful thi mere liye"
2. **Abuse/report vote** — "yeh rating genuine nahi lagti / abusive hai"

**Business impact**: Yeh crowd-sourced quality-control hai. Agar bahut saare buyers ek
particular rating ko "abuse" mark karte hain, IndiaMART ko signal milta hai ki us rating ko
manually review karna chahiye. Aur "helpful" count high-quality, genuinely useful ratings ko
upar highlight karne mein kaam aa sakta hai (jaise sabse helpful review sabse upar dikhana) —
supplier profile page pe yeh count ek human-readable line ban ke dikhta hai, jaise "3 users
found this helpful."

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Koi bhi buyer jo rating padh raha hai** | Vote karta hai — helpful ya abuse |
| **Original rating dene wala buyer** | Iska role sirf itna hai ki unki rating ke against vote pad raha hai — woh khud vote process mein involved nahi. (Kya woh apni khud ki rating ko vote kar sakte hain, yeh explicitly clear nahi hai — dekho Edge Cases) |
| **Internal moderation/review/approval team** | Approval-workflow ke through counts ka ek alag copy dekhti hai (technical doc §8 mein "approvalPg" replication) — exact review-workflow scope se bahar hai iss doc ke |
| **Supplier jiski rating hai** | Koi push-notification is vote ke liye supplier ko nahi jaata (confirmed technical pass mein) — yeh silent-hai supplier ke liye |

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
   agli baar wo rating dekhe, updated counts saath mein dikhein (ek readable line ke saath,
   jaise "1 user found this helpful" / "5 users found this helpful")
7. Same counts internally ek doosre, approval-workflow wale system mein bhi copy ho jaate
   hain, taaki review-side data bhi sync mein rahe
```

**Business impact**: Yeh ek real-time feedback loop hai — koi bhi supplier-profile page load
turant latest helpful/abuse counts dikhati hai, delayed batch-process nahi. Koi email/push
notification supplier ko nahi jaata is vote ki wajah se — yeh purely ek background
signal hai, supplier ko disturb nahi karta.

---

## 4. Helpful/Abuse Count Kaise Dikhta Hai — Zero ka Special Case

Ek important business behavior: **agar kisi rating ko abhi tak koi vote nahi mila (count =
0), toh woh "0" ke roop mein nahi dikhta — woh bilkul dikhta hi nahi (blank/absent)**. Sirf
tabhi count aur label ("N user(s) found this helpful") dikhna shuru hota hai jab kam se kam
ek genuine vote aa chuka ho. Product/support team ke liye important: agar koi poochhe "iss
rating pe helpful count 0 kyun nahi dikh raha," jawaab hai — yeh intentional hai, zero ko
hide kiya jaata hai, missing data nahi hai.

---

## 5. Business Rules — Plain Language Mein

1. **Sirf 2 valid votes hain**: "helpful" ya "abuse/report" — beech ka koi option nahi.
2. **Ek buyer ek rating pe sirf ek baar vote kar sakta hai** — dobara try karne pe system
   bolta hai "Record already exists," koi error nahi dikhata jo confuse kare, bas silently
   ignore kar deta hai duplicate vote ko. (Technical note: yeh HTTP-level pe bhi "success" ke
   roop mein hi report hota hai, taaki app/UI ko koi confusing error na dikhe.)
3. **Vote karne ke liye bhi ek "authorized caller" list hai** — yeh feature random external
   callers se open nahi hai, sirf whitelisted internal/app sources se aa sakta hai (jaisa
   Rating submit karne ke liye bhi tha) — lekin yeh **ek alag, independent list hai** Rating
   submit wali list se, dono ek saath maintain nahi hoti.
4. **Helpful/Abuse counts turant, synchronously update hote hain** — buyer vote karte hi,
   background mein turant original rating record update ho jaata hai, koi lambi delay nahi.
   Yeh update ek se zyada internal system mein bhi jaata hai (rating ka main record, plus ek
   approval-workflow wala copy) — dono jagah sync mein rehna chahiye.
5. **Zero count hide ho jaata hai** — dekho section 4.
6. **Koi background/scheduled job (cron) nahi hai iss feature ke liye** — yeh poori tarah
   real-time, event-driven hai; koi daily reconciliation batch process ise touch nahi karta
   (Rating khud ke liye do crons hain, lekin Rating Usefulness ke liye koi nahi).

---

## 6. Notifications — Kaun ko kab pata chalta hai

| Kab | Kya notification jaata hai |
|---|---|
| Buyer helpful/abuse vote karta hai | **Koi email/push notification kisi ko nahi jaata** — na buyer ko, na supplier ko. Yeh purely ek silent, background signal hai |
| Duplicate vote try hua | Koi visible error nahi — response "success"-jaisa hi aata hai |

---

## 7. Edge Cases — Product/Business Perspective se

1. **"Maine dobara vote karne ki koshish ki, kuch hua hi nahi"** — Expected hai. System
   duplicate votes ko silently ignore karta hai (per buyer, per rating, sirf ek vote count
   hota hai) — aur response bhi "success" jaisa hi dikhta hai, taaki UI mein koi confusing
   error na aaye.

2. **"Rating ka helpful-count profile page pe turant nahi dikha"** — Ho sakta hai
   background-update process mein thoda lag ho (yeh ek chhota multi-step background chain
   hai), lekin design se yeh near-real-time hona chahiye — agar consistently delay ho raha
   hai, technical investigation chahiye.

3. **"Kya main apni khud ki rating ko helpful mark kar sakta hoon?"** — Business rules mein
   koi explicit self-vote-block nahi mila iss review mein (dobara confirm kiya gaya, code
   mein genuinely nahi hai) — worth confirming with product team ki yeh intentional gap hai
   ya edge case jo miss ho gaya.

4. **"Helpful count '0' kyun nahi dikh raha, blank kyun hai?"** — Expected hai, section 4
   dekho. Zero ko intentionally hide kiya jaata hai.

5. **"Maine helpful vote diya, kya supplier ko pata chala?"** — Nahi, supplier ko koi
   notification nahi jaati is action ke liye. Yeh silent hai supplier ke liye by design (ya
   kam se kam current behavior yahi hai — confirm karo product team se agar yeh intentional
   nahi hai).

---

## 8. Quick Summary

- Rating Usefulness = "yeh rating helpful thi?" yes/no vote, kisi doosre buyer dwara.
- Ek buyer, ek rating pe, ek hi vote — dobara try karne pe bhi "success"-jaisa response.
- Vote hone ke turant baad, us rating ke helpful/abuse counts update ho jaate hain — internally
  do jagah (main record + approval-workflow copy) sync hote hain.
- Zero count kabhi "0" nahi dikhta, blank/hidden rehta hai.
- Supplier ko is vote ki koi notification nahi jaati.
- Koi background/scheduled cron job nahi hai — poori tarah real-time.
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

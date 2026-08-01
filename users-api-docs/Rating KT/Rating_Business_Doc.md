# Rating (Supplier Rating) — Business Doc (Product Perspective)

**Note**: yeh doc sirf **Rating** (buyer supplier ko 1-5 star deta hai) cover karta hai.
**Rating Usefulness** ("was this rating helpful?" vote) ek alag concept hai — uska apna
alag KT folder baad mein banega. Confuse mat karo dono ko.

Yeh doc Rating feature ko **business/product nazariye** se samjhata hai — koi code nahi.
Technical implementation ke liye [`Rating_Technical_Doc.md`](./Rating_Technical_Doc.md)
dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Jab ek buyer kisi supplier se interact karta hai (enquiry, order, conversation), IndiaMART
unse poochta hai: **is supplier ko 1-5 stars do, ek comment likho, aur bataao specific
cheezein (Response time, Quality, Delivery) achi thi ya nahi.**

Yeh rating baad mein supplier ke public profile pe dikhti hai, taaki **doosre buyers** decide
kar sakein ki is supplier pe trust karna hai ya nahi.

**Business impact**: Rating ek platform-wide trust-signal hai. Ek high-rated supplier ko
zyada enquiries milti hain, buyers unse jaldi contact karte hain. Isliye rating fake nahi
honi chahiye, aur genuine feedback fast, reliably, aur bina spam ke publish hona chahiye.

**Yeh kyun simple nahi hai**: ek raw rating seedha "publish" nahi ho jaati. Usse pehle:
- Content check hota hai (abusive language, PII leakage nahi honi chahiye)
- Confirm hota hai ki yeh ek real buyer-supplier connection se aayi hai (fake rating nahi)
- Supplier ka average star-rating recalculate hota hai
- Supplier ko notify kiya jaata hai
- Yeh sab **bina buyer ko wait karwaye** hota hai — buyer submit karte hi "done" dekh leta
  hai, baaki sab background mein chalta hai

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Buyer** | Rating deta hai (star + comment + thumbs up/down on parameters + photos) |
| **Supplier** | Rating dekhta hai, reply kar sakta hai |
| **Content moderation system** | Har rating ko abuse/PII ke liye check karta hai submit hote hi |
| **Matchmaking check** | Verify karta hai ki buyer-supplier genuinely connected the |
| **Internal admin (GLADMIN)** | Kuch cases mein rating ko manually approve/manage kar sakta hai |
| **Other buyers** | Rating padh sakte hain, "helpful" vote kar sakte hain *(alag feature, iss doc mein nahi)* |

---

## 3. Rating ki "Visibility States" — supplier/buyer ke liye iska matlab

Har rating ek "display status" carry karta hai jo decide karta hai woh publicly dikhegi ya
nahi:

| State | Business meaning |
|---|---|
| **Pending/Hidden (default, jab submit hoti hai)** | Rating abhi-abhi aayi hai, moderation/checks ho rahe hain — kisi ko nahi dikhti |
| **Visible** | Sab checks pass ho gaye, ab public profile pe dikh rahi hai |
| **Disabled/Hidden (post-check)** | Matchmaking fail hui (fake lag rahi thi), ya kisi aur reason se hide kar di gayi — kabhi visible nahi hui |
| **Banned content** | Abusive/inappropriate content detect hua, block ho gayi |

**Important**: rating submit hote hi **turant hidden state mein jaati hai**, publish nahi
hoti. Buyer ko turant "thank you" dikh sakta hai, lekin actual visibility background checks
ke baad hi milti hai — yeh design se hai, taaki abusive/fake content kabhi live na ho.

---

## 4. Main Business Flows (step-by-step, no code)

### Flow A — Buyer ek naya rating submit karta hai

```
1. Buyer supplier ko stars deta hai, comment likhta hai, "Response/Quality/Delivery" pe
   thumbs up/down deta hai, chaahe toh photos bhi attach karta hai
2. System check karta hai ki yeh pehli baar hai ya buyer pehle bhi is supplier ko rate kar
   chuka hai (system ismein farak samajhta hai — "naya" vs "existing relationship update")
3. Rating hidden state mein save ho jaati hai
4. Background mein turant:
   - Content-moderation check shuru hoti hai (abuse/PII scan)
   - Buyer-supplier ka genuine connection verify hota hai
   - Product/category context enrich hota hai (kis product ke baare mein rating thi)
   - Yeh info ek internal Lead-Management System ko bhi bheji jaati hai
5. Sab pass hone ke baad, rating "visible" ban jaati hai
6. Supplier ka average star-rating recalculate hota hai
7. Supplier ko notify kiya jaata hai ("aapko ek naya rating mila hai")
```

**Business impact**: Buyer ka experience instant hai (submit → done), lekin actual public
visibility ek trust-verified pipeline se guzarti hai — bina buyer ko wait karaye.

### Flow B — Rating "Update" hoti hai (existing relationship ka naya rating, ya edit/approval)

```
1. Ek admin ya system flow POST karta hai UPDATE_FLAG=U ke saath (naya insert nahi, existing
   record se related action)
2. Yeh path alag business events trigger kar sakta hai depending on situation:
   - Naya company/relationship notification
   - Supplier ko naye rating ka alert
   - Content-moderation dobara trigger (agar zaroori ho)
```

**Business impact**: Yeh ek flexible path hai jo different scenarios (admin correction,
re-check, notification resend) handle karta hai — normal buyer-submit flow se alag.

### Flow C — Supplier apni rating ka reply karta hai

```
Supplier apna comment likhta hai rating ke against ("SUPPLIER_COMMENTS" field ke saath),
same submit-API dobara call hoti hai — is baar rating value change nahi hota, sirf reply
add hota hai
```

**Business impact**: Two-way conversation ka feel deta hai — supplier apna side bata sakta
hai agar rating unfair lage.

### Flow D — Purani ratings archive ho jaati hain

```
Agar ek buyer-supplier pair ke beech multiple ratings hain (jaise time ke saath relationship
continue hui), sirf latest wali "live" rakhi jaati hai — purani automatically archive ho
jaati hain
```

**Business impact**: Supplier ka displayed rating hamesha **most recent** relationship
status reflect karta hai, purana outdated feedback confuse nahi karta.

### Flow E — Koi rating padhta hai (buyer ya supplier ka profile page)

```
Supplier profile page load hoti hai -> saari visible ratings, average star count, aur
"reasons" ka breakdown (kitne logon ne Response/Quality/Delivery pe thumbs-up diya) dikhta hai
```

**Business impact**: Yeh woh jagah hai jahan trust-signal actually kaam aata hai — naya buyer
decide karta hai supplier ko contact karna hai ya nahi, isi data ke basis pe.

---

## 5. Business Rules — Plain Language Mein

1. **Rating hamesha 1-5 ke beech honi chahiye** — koi aur value accept nahi hoti.
2. **Buyer khud ko rate nahi kar sakta** — buyer aur supplier ka ID same nahi ho sakta.
3. **Rating submit hote hi hidden hoti hai, publish nahi** — moderation/verification pass
   karne ke baad hi visible hoti hai.
4. **Content moderation, matchmaking-check, aur enrichment sab parallel/background mein
   chalte hain** — buyer ko inka wait nahi karna padta.
5. **Photos attach kiye ja sakte hain rating ke saath** — optional, lekin agar diye jaayein
   toh unka apna storage aur reference hota hai.
6. **"Reasons" (Response/Quality/Delivery) optional hain per-rating** — buyer chaahe toh
   sirf stars de sakta hai bina inpe thumbs-up/down diye.
7. **Purani ratings automatically archive hoti hain** jab naya rating aata hai usi
   buyer-supplier pair ke beech — sirf latest live rehti hai.
8. **Rating ke saath ek internal Lead-Management sync bhi hoti hai** — yeh sirf trust-display
   ke liye nahi, IndiaMART ke internal sales/lead-tracking systems ke liye bhi useful data
   hai.

---

## 6. Edge Cases — Product/Business Perspective se

1. **"Maine 5-star rating diya, phir bhi supplier ka overall score nahi badha"** — Ho sakta
   hai rating abhi bhi moderation/verification pipeline mein pending ho. Turant reflect nahi
   hoti — kuch minute lag sakte hain.

2. **"Meri rating kabhi dikhi hi nahi"** — Ho sakta hai matchmaking-check fail hui ho (system
   confirm nahi kar paaya ki aap sach mein iss supplier se connected the), ya content
   abusive/inappropriate flag hua ho.

3. **"Maine 2 baar rate kiya, dono dikhne chahiye"** — Nahi, sirf latest wali dikhti hai,
   purani automatically archive ho jaati hai (Flow D). Yeh intentional hai, taaki rating
   list clutter na ho purane feedback se.

4. **Star-rating aur "helpful" votes do alag cheezein hain** — dekho previous conversation:
   Rating khud ek 1-5 star hai; "helpful" ek doosra concept hai jahan doosre buyers kisi
   already-diye-hue rating pe vote karte hain. Alag KT folder banega uske liye.

---

## 7. Quick Summary

- Rating = buyer ka 1-5 star + comment + reasons + optional photos, supplier ke against.
- Submit turant hota hai, lekin publish hone se pehle moderation + matchmaking-verify
  background mein chalte hain.
- Sirf latest rating per buyer-supplier pair live rehti hai, purani archive hoti hain.
- Rating data internal Lead-Management system ko bhi jaata hai, sirf public display ke liye
  nahi.

---

## See also

- [`Rating_Technical_Doc.md`](./Rating_Technical_Doc.md) — same flows, code-level detail
- [`../ratings_reviews_product_overview.md`](../ratings_reviews_product_overview.md) —
  bigger Ratings & Reviews story (isme Social Reviews bhi cover hoti hain, jo yahan scope se
  bahar hai)
- [`../full_read_write_picture.md`](../full_read_write_picture.md) — supplier-rating ka
  poora architecture trace

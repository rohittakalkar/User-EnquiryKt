# Social Contacts — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Social_Contacts_Technical_Doc.md`](./Social_Contacts_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Supplier apne **personal/social-contact details** — Facebook, LinkedIn, Twitter, Google+,
Instagram, Klout profile-URLs, gender, aur WhatsApp/UPI-active-status — IndiaMART profile ke
saath save kar sakta hai. Yeh Marketplace-URL (Google/FB/Instagram business-presence) se
alag hai — yeh supplier ke **personal social-network profiles** aur **communication-
preferences** (WhatsApp/UPI available hai ya nahi) capture karta hai.

**Business impact**: Yeh data buyer-supplier communication-channels enrich karta hai
(WhatsApp-availability jaisa flag directly contact-experience improve karta hai), aur
supplier ki digital-presence ko profile mein integrate karta hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Apne social-profile-links, gender, WhatsApp/UPI-status update karta hai |
| **Internal tools/apps (GLADMIN, SELLERMY, IMOB, MSITE)** | Supplier ki taraf se yeh data update kar sakte hain |
| **Buyer** | Profile dekhte waqt yeh details (embedded) access kar sakta hai |

---

## 3. Business Flow (step-by-step, no code)

```
1. Supplier (ya koi allowed internal-app) social-contact details submit karta hai —
   "insert" (naya) ya "update" (existing) action ke saath
2. System validate karta hai — gender sirf Male/Female, WhatsApp-active sirf 0/1
3. Naya record insert hota hai, ya existing record ke sirf diye-gaye fields update hote
   hain
4. WhatsApp-active-status change hone par, "kab mark hua" bhi track hota hai
5. Yeh data supplier ke overall profile-detail response ke andar embedded milta hai —
   koi alag "view social contacts" screen nahi hai
```

**Business impact**: Ek lightweight, supporting-data feature hai jo main profile-detail
response ka hissa ban jaata hai.

---

## 4. Business Rules — Plain Language Mein

1. **Insert ke liye kam-se-kam 2 meaningful values honi chahiye** — sirf ID de kar empty
   record nahi bana sakte.
2. **Gender sirf "Male"/"Female" accept hota hai** (case-insensitive check hai).
3. **WhatsApp-active-flag sirf 0 ya 1 ho sakta hai.**
4. **Update mein sirf diye-gaye fields hi change hote hain** — baaki untouched rehte hain
   (partial-update supported hai).
5. **Ek async path bhi hai** jo yahi table independently update kar sakta hai (registration
   ya kisi doosre internal-flow se) — dono paths same final-table ko target karte hain.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine gender update kiya, error aaya"** — Expected hai agar value "Male"/"Female" ke
   alawa kuch bheja gaya.
2. **"WhatsApp-active status change hua, date bhi update hui"** — Expected hai, system
   automatically "kab mark hua" timestamp record karta hai.

---

## 6. Quick Summary

- Social Contacts = supplier ke personal social-profile-links + gender + WhatsApp/UPI
  communication-preference-flags.
- Insert/update, partial-update supported.
- Koi dedicated read-screen nahi, overall profile-detail response ka hissa hai.
- Marketplace-URL (business-presence-links) se alag concept hai.

---

## See also

- [`Social_Contacts_Technical_Doc.md`](./Social_Contacts_Technical_Doc.md) — code-level detail
- [`../Market Place URL KT/MarketPlaceUrl_Business_Doc.md`](../Market%20Place%20URL%20KT/MarketPlaceUrl_Business_Doc.md) —
  related but separate business-presence-URL concept

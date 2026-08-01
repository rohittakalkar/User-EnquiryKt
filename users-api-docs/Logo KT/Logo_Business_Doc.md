# Company Logo — Business Doc (Product Perspective)

**Note**: yeh doc **Logo** cover karta hai. **Image** (`POST /user/image` — general branding
photos, banners) ek **alag** feature/table hai, chahe dono "branding visuals" ki tarah lagte
hain — dekho [`../Image KT/Image_Business_Doc.md`](../Image%20KT/Image_Business_Doc.md).

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Logo_Technical_Doc.md`](./Logo_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Supplier apna **company logo** upload karta hai — yeh unke storefront, search-results,
aur baaki jagah unhe visually represent karta hai. Logo sirf ek image-upload nahi hai —
iska apna **approval-workflow** hai (jaisa moderation), aur multiple downstream systems ko
notify karta hai jab bhi change ho.

**Business impact**: Logo supplier ki pehli visual-impression hai buyer ke liye. Isliye
yeh moderation se guzarta hai (koi inappropriate image na ho), aur multiple systems
(search, LMS) mein turant sync hona chahiye.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Naya logo upload karta hai, ya existing update karta hai |
| **Internal moderation/admin process** | Logo ko approve/reject kar sakta hai |
| **Search/directory systems** | Logo ko apne records mein sync karke rakhte hain |
| **Buyer** | Logo dekhta hai supplier ke storefront/search-results pe |

---

## 3. Business Flow (step-by-step, no code)

```
1. Supplier logo upload karta hai (4 alag sizes automatically generate hoke bheje jaate
   hain — 90x90, 120x120, 250x250, aur original)
2. Naya logo "Pending" approval-status ke saath save hota hai
3. Agar reject ho jaaye, ek "rejection reason" record hoti hai jo supplier ko dikh sakti hai
4. Approve hone ke baad, logo live ho jaata hai
5. Har change (approved ho ya reject) downstream systems ko async notify hoti hai —
   search-index, LMS, waghera apne copies update karte hain
```

**Business impact**: Yeh ek "pending → reviewed → live" pipeline hai — turant publish nahi
hota, ek quality-check step hai beech mein.

---

## 4. Business Rules — Plain Language Mein

1. **Naya logo hamesha "Pending" (`P`) status se shuru hota hai** — supplier khud apna logo
   turant live nahi kar sakta.
2. **4 image-sizes ek saath handle hoti hain** — supplier ek image upload karta hai, system
   multiple resolutions store karta hai (likely resize backend se pehle hi ho chuka hota
   hai, system sirf store karta hai).
3. **Rejection ka apna reason-field hai** — supplier ko pata chal sakta hai kyun reject hua.
4. **Existing-vs-naya logo alag internal actions hain** — system decide karta hai insert
   karna hai ya existing record update karna hai, based on kya already exist karta hai.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine logo upload kiya, dikh nahi raha"** — Expected hai agar abhi "Pending" status
   mein hai — approval ka wait karna padega.
2. **"Mera logo reject ho gaya, kyun?"** — Rejection-reason field check karo, system usse
   capture karta hai.
3. **"Meri logo search-results mein purani dikh rahi hai"** — Approval hone ke baad,
   downstream systems (search) ko sync hone mein thoda time lag sakta hai — asynchronous
   propagation hai.

---

## 6. Quick Summary

- Logo = approval-workflow-based branding image, multiple sizes, multi-system sync.
- Pending → Approved/Rejected lifecycle.
- Image (general photos) se alag table/system hai — dekho Image KT.

---

## See also

- [`Logo_Technical_Doc.md`](./Logo_Technical_Doc.md) — code-level detail
- [`../Image KT/Image_Business_Doc.md`](../Image%20KT/Image_Business_Doc.md) — related but
  separate general-image system
- [`../Profile Meta Template KT/Profile_Meta_Template_Business_Doc.md`](../Profile%20Meta%20Template%20KT/Profile_Meta_Template_Business_Doc.md) —
  broader storefront-content family

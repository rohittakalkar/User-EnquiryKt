# User Image — Business Doc (Product Perspective)

**Note**: yeh doc **Image** (`POST /user/image` — general branding photos) cover karta hai.
**Logo** (company logo, apna approval-workflow ke saath) ek **alag** table/system hai —
dekho [`../Logo KT/Logo_Business_Doc.md`](../Logo%20KT/Logo_Business_Doc.md).

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Image_Technical_Doc.md`](./Image_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Supplier apne profile ke liye **general images** upload kar sakta hai — logo se alag ek
broader "branding photo" concept (jaise factory-photos, product-showcase images, ya koi
aur general-purpose branding image). Yeh Logo se simpler hai — koi multi-step
approval-history-tracking nahi, seedha update/insert.

**Business impact**: Yeh Profile/Template family ke saath milke supplier ke storefront ko
visually complete banata hai — Logo primary branding-mark hai, Image supporting visuals.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Image upload/update karta hai |
| **Buyer** | Image dekhta hai storefront pe |

---

## 3. Business Flow (step-by-step, no code)

```
1. Supplier ek image upload karta hai (multiple resolutions — 32x32 se 250x250 tak, plus
   original — automatically handle hoti hain)
2. System check karta hai: existing record hai ya nahi
   - Agar hai, update ho jaata hai
   - Agar nahi, naya record insert hota hai
3. Ek status/reject-reason field bhi hai (Logo jaisa hi concept, lekin simpler tracking)
```

**Business impact**: Ek straightforward, low-friction image-management feature — Logo ke
multi-step approval-workflow ke muqable directly update hoti hai.

---

## 4. Business Rules — Plain Language Mein

1. **Update-first, insert-fallback pattern** — system pehle UPDATE try karta hai; agar
   0 rows affect hui (record exist hi nahi karta), tabhi INSERT karta hai.
2. **5 image-resolutions ek saath handle hoti hain** — 32x32, 64x64, 125x125, 250x250,
   original (Logo ke 4-resolution-set se alag sizes).
3. **Status/reject-reason field hai, lekin koi multi-step approval-ID-tracking nahi** —
   Logo jitna elaborate workflow nahi.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Image aur Logo dono upload kiye, dono independent hain"** — Expected hai, yeh do
   alag tables/features hain, ek doosre ko affect nahi karte.

---

## 6. Quick Summary

- Image = general branding photos, simple update/insert, koi elaborate approval-tracking
  nahi.
- Logo se alag table (`GLUSR_USR_IMAGE` vs `GLUSR_USR_LOGO`), alag size-set.

---

## See also

- [`Image_Technical_Doc.md`](./Image_Technical_Doc.md) — code-level detail
- [`../Logo KT/Logo_Business_Doc.md`](../Logo%20KT/Logo_Business_Doc.md) — related but
  separate, approval-workflow-based logo system

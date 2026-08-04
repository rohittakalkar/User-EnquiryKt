# User Image — Business Doc (Product Perspective)

**Note**: yeh doc **Image** (`POST /user/image` — general branding photos, aur unka read
`/detail` API se) cover karta hai.
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

Yeh image, upload hone ke baad, supplier ke profile-detail response mein bhi wapas dikhti hai —
matlab yeh sirf ek "write and forget" feature nahi hai, balki storefront/app pe supplier ki
profile-detail dekhte waqt yeh image hi wahan render hoti hai (largest-available-size
preference ke saath).

**Business impact**: Yeh Profile/Template family ke saath milke supplier ke storefront ko
visually complete banata hai — Logo primary branding-mark hai, Image supporting visuals. Agar
yeh image galat/purani dikhti hai, seedha supplier ki storefront-credibility pe impact padta hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Image upload/update karta hai (`POST /user/image`) |
| **Buyer / App user** | Image dekhta hai supplier storefront/profile-detail pe (`/detail` API ke through) |
| **Client apps (BuyerMy, MAPI, SellerMy, Android, iOS)** | Yeh sab hi upload allowed hain (gateway-key allow-list se confirm) |

---

## 3. Business Flows (step-by-step, no code)

### A. Supplier image upload/update

```
1. Supplier ek image upload karta hai (multiple resolutions — 32x32 se 250x250 tak, plus
   original — client-side pehle se resize karke bhejta hai)
2. System validate karta hai: mandatory fields, valid app-source, valid status-value
3. System check karta hai: existing record hai ya nahi
   - Agar hai, update ho jaata hai
   - Agar nahi, naya record insert hota hai
4. Ek status/reject-reason field bhi hai (Logo jaisa hi concept, lekin simpler tracking)
5. Supplier ko response milta hai — success ya failure, lekin response se yeh pata nahi chalta
   ki naya record bana ya purana update hua (dono cases mein same "update success"-type message)
```

**Business impact**: Ek straightforward, low-friction image-management feature — Logo ke
multi-step approval-workflow ke muqable directly update hoti hai, koi wait-for-approval delay
nahi.

### B. Image profile pe dikhna (view/read)

```
1. Koi bhi client (buyer app, supplier dashboard, etc.) supplier ka profile-detail fetch karta
   hai
2. System available resolutions mein se sabse bada available size wapas bhejta hai
   (250x250 upar, phir 125x125, phir 64x64 — jo bhi pehla non-empty mile)
3. Agar koi image upload hi nahi hui, empty return hoti hai — koi placeholder/default image
   nahi
```

**Business impact**: Har baar profile-detail call pe fresh data dikhta hai — koi stale-cache
ka risk nahi (kam se kam iss review ke scope ke code mein koi caching layer nahi mila).

---

## 4. Status Field (jo hai, but limited)

| Field | Kya hai | Note |
|---|---|---|
| `STATUS` | Ek single-letter code (`P`/`D`/`A`/`Y` allowed values) | Exact business-meaning (Pending/Deleted/Approved/...?) code mein documented nahi mila — **[INFERRED — confirm with team]** |
| `REJECT_REASON` | Free-text reason field | Kaun set karta hai (admin-tool/moderation-flow) yeh iss review mein confirm nahi ho paya |

Logo ke muqable, Image mein **koi multi-step approval-ID/history-tracking nahi hai** — yeh ek
simple status-field hai, poora versioned-approval-workflow nahi.

---

## 5. Business Rules — Plain Language Mein

1. **Update-first, insert-fallback pattern** — system pehle UPDATE try karta hai; agar
   0 rows affect hui (record exist hi nahi karta), tabhi INSERT karta hai. Supplier ko iska
   pata nahi chalta — same "success" message dono cases mein.
2. **5 image-resolutions ek saath handle hoti hain** — 32x32, 64x64, 125x125, 250x250,
   original (Logo ke 4-resolution-set se alag sizes) — har size ka apna width-height (WH)
   metadata bhi store hota hai.
3. **Status/reject-reason field hai, lekin koi multi-step approval-ID-tracking nahi** —
   Logo jitna elaborate workflow nahi.
4. **Sirf allowed apps hi upload kar sakte hain** — BuyerMy, MAPI, SellerMy, Android, iOS ka
   validation-key check hota hai; koi aur source se request aaye toh reject.
5. **Mandatory info bina request fail hoti hai** — supplier ID, updated-by, update-screen,
   validation-key, aur status — yeh sab bhejna zaroori hai.
6. **Profile-detail pe dikhne wali image, sabse bade available size ko priority deti hai** —
   agar 250x250 available hai, wahi dikhega; nahi toh 125x125, phir 64x64.

---

## 6. Notifications

Koi dedicated notification (email/SMS/push) iss feature ke liye code mein nahi mila — na hi
upload-success pe, na hi status-change pe. **[INFERRED — confirm with team]**: agar koi
moderation-reject-notification hoti hai, woh iss review ke scope se bahar kisi aur system mein
ho sakti hai.

---

## 7. Edge Cases — Product/Business Perspective se

1. **"Image aur Logo dono upload kiye, dono independent hain"** — Expected hai, yeh do
   alag tables/features hain, ek doosre ko affect nahi karte.
2. **"Maine image upload ki, lekin response mein pata nahi chala naya record bana ya purana
   update hua"** — Expected hai, system dono cases mein same success-message deta hai.
3. **"Koi image upload hi nahi ki, profile-detail pe kya dikhega?"** — Empty/blank, koi default
   placeholder image system se nahi aati.
4. **"Do baar bahut jaldi-jaldi (concurrently) pehli image upload ki"** — technical risk hai ki
   duplicate record ban sakta hai agar underlying DB mein unique-constraint na ho — confirm
   karna baaki hai (dekho Technical Doc, Open Questions).

---

## 8. Quick Summary

- Image = general branding photos, simple update/insert, koi elaborate approval-tracking
  nahi.
- Logo se alag table (`GLUSR_USR_IMAGE` vs `GLUSR_USR_LOGO`), alag size-set (5 resolutions vs
  Logo ke 4).
- Upload sirf allowed apps (BuyerMy/MAPI/SellerMy/Android/iOS) se ho sakta hai.
- Read-side profile-detail API se hi dikhti hai — largest-available-resolution priority ke
  saath, koi caching nahi.
- Koi RabbitMQ/Kafka/Redis fan-out iss feature ke liye specifically nahi mila — pura
  synchronous, single-table flow.
- Status/reject-reason field hai lekin kaun use karta hai yeh open question hai.

---

## See also

- [`Image_Technical_Doc.md`](./Image_Technical_Doc.md) — code-level detail
- [`../Logo KT/Logo_Business_Doc.md`](../Logo%20KT/Logo_Business_Doc.md) — related but
  separate, approval-workflow-based logo system

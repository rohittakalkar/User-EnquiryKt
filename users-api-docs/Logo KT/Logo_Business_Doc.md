# Company Logo — Business Doc (Product Perspective)

**Note**: yeh doc **Logo** cover karta hai. **Image** (`POST /user/image` — general branding
photos, banners) ek **alag** feature/table hai, chahe dono "branding visuals" ki tarah lagte
hain — dekho [`../Image KT/Image_Business_Doc.md`](../Image%20KT/Image_Business_Doc.md).

Business/product perspective ke liye yeh doc hai — koi code nahi. Code-level detail ke liye
[`Logo_Technical_Doc.md`](./Logo_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Supplier apna **company logo** upload karta hai — yeh unke storefront, buyer-profile page,
product listing, aur search-results mein unhe visually represent karta hai. Logo sirf ek
image-upload nahi hai — iska apna **approval-workflow** (moderation) hai, aur har change
multiple downstream systems (search, LMS, general company-record) ko turant notify karta hai.

1. **Trust aur brand-identity**: Logo supplier ki pehli visual-impression hai buyer ke liye —
   product listing dekhte waqt, ya storefront visit karte waqt.
2. **Quality-control moderation**: Naya ya updated logo turant live nahi hota — pehle ek
   review-step se guzarta hai, taaki koi inappropriate/irrelevant image publicly na dikhe.
3. **Multi-system consistency**: Logo ek single "source-of-truth" record hai, lekin search,
   LMS, aur company-sync teeno ko apni copy chahiye hoti hai — isliye har change automatically
   teeno jagah propagate hoti hai.

**Bottom line**: Logo = branding + moderation + multi-system sync, teeno ek saath.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Naya logo upload karta hai, ya existing update/replace karta hai |
| **Internal Admin/Moderator (GLADMIN)** | Pending logo ko approve ya reject karta hai; direct edit bhi kar sakta hai |
| **Search results** | Search-listing mein logo apni copy se dikhata hai |
| **LMS (Lead Management System)** | Apni copy maintain karta hai logo ki, sales/internal-tooling ke context mein |
| **Buyer** | Logo dekhta hai product-listing, storefront, aur buyer-profile page pe |

---

## 3. Logo Status — supplier ke liye iska matlab

Har logo record ek "approval status" carry karta hai. Yeh important hai kyunki iska seedha
asar hota hai ki logo **kahin dikh raha hai ya nahi**:

| Status | Business meaning | Buyer ko dikhta hai? |
|---|---|---|
| **Pending** | Abhi-abhi upload/update hua hai, review ka wait kar raha hai | **Nahi** — jab tak review na ho |
| **Approved** | Internal review pass ho gaya | **Haan** — live ho jaata hai |
| **Rejected** | Review mein fail ho gaya (inappropriate/unclear/policy-violation) | Nahi — purana logo (agar tha) show hota rehta hai jab tak naya replace na ho |
| **Deleted/Blank** | Supplier ne apna logo hata diya, ya empty submit kiya | Nahi — logo section khaali dikhta hai |

**Business rule jo yaad rakhne layak hai**: Naya logo **hamesha Pending se shuru hota hai** —
supplier khud apna logo turant live nahi kar sakta, chahe woh kitna bhi confident ho image ke
baare mein. Yeh ek intentional quality-gate hai.

---

## 4. Main Business Flows (step-by-step, no code)

### Flow A — Supplier naya logo upload karta hai

```
1. Supplier apne panel/app se ek naya logo image upload karta hai
2. System ek naya logo-record banata hai, status "Pending" ke saath (hardcoded — koi bhi
   status supplier khud set nahi kar sakta)
3. 4 alag sizes (chhota se bada) store hoti hain ek hi upload se
4. Background mein: teen downstream systems (search, LMS, company-sync) ko notify kiya
   jaata hai ki naya logo-record bana hai — woh apni-apni copies bana lete hain
5. Logo abhi buyer ko dikhta nahi — review pending hai
```

**Business impact**: Supplier turant confirm ho jaata hai ki upload ho gaya, lekin visually
kuch change nahi hota jab tak review na ho — expected/intentional delay hai, bug nahi.

### Flow B — Supplier apna existing logo update/replace karta hai

```
1. Supplier naya logo-image daalta hai, existing record ke against
2. System naye images se purane ko overwrite kar deta hai
3. Status caller/system ke supplied value pe depend karta hai — matlab agar update-flow
   explicitly status reset nahi karta, purana status (jaise "Approved") technically wahi
   reh sakta hai jab tak koi explicit re-review step na ho
4. Downstream systems ko naye images se sync kiya jaata hai
```

**Business impact**: Yeh ek potential gotcha hai — agar koi supplier apna already-approved
logo change kare, aur naya image bina re-review ke seedha "Approved" status ke saath reh
jaaye, toh unreviewed content publicly live ho sakta hai. **Isko team se confirm karna
zaroori hai ki update-flow humesha status explicitly Pending pe reset karta hai ya nahi**
(dekho [`Logo_Technical_Doc.md`](./Logo_Technical_Doc.md) Open Questions).

### Flow C — Internal Admin logo ko Approve/Reject karta hai

```
1. GLADMIN (internal moderator) ek Pending logo dekhta hai review-queue mein
2. Approve karta hai -> status "Approved" ban jaata hai, logo live ho jaata hai buyer ke liye
   Reject karta hai -> status "Rejected" ban jaata hai, ek rejection-reason bhi record hoti hai
3. Agar koi logo already Approved/Rejected ho chuka hai, usse dobara approve/reject karne ki
   koshish silently ignore ho jaati hai (safety-guard — double-processing na ho)
4. Downstream systems (search, LMS, company-sync) ko decision ka pata chalta hai automatically
```

**Business impact**: Yeh ek fraud/quality-prevention step hai — bina isse, koi bhi kuch bhi
image apload kar ke turant public dikha sakta tha.

### Flow D — Supplier apna logo hata deta hai (delete/blank)

```
1. Supplier request bhejta hai apna logo hatane ke liye (ya khaali submit karta hai)
2. Saari images NULL/khaali ho jaati hain, rejection-reason bhi clear ho jaata hai
3. Status "Deleted"-jaisi state mein chala jaata hai
4. Downstream systems ko yeh bhi propagate hota hai — sab jagah se logo hat jaata hai
```

**Business impact**: Supplier ke paas apna logo hataane ka full control hai, aur yeh change
bhi turant sab systems mein consistent reflect hota hai.

---

## 5. Business Rules — Plain Language Mein

1. **Naya logo hamesha "Pending" se shuru hota hai** — supplier khud apna logo turant live
   nahi kar sakta.
2. **4 image-sizes ek saath handle hoti hain** — supplier ek image upload karta hai, system
   4 resolutions (chhota badge se leke original tak) store karta hai.
3. **Approve/Reject sirf ek Pending logo pe kaam karta hai** — ek baar decide ho chuka logo,
   dobara approve/reject karne ki koshish kuch nahi karti (safe no-op), taaki accidental
   double-click ya duplicate-action se koi conflict na ho.
4. **Rejection ka apna reason-field hai** — supplier ko pata chal sakta hai kyun reject hua.
5. **Duplicate-submission ko error nahi dikhaya jaata** — agar wahi cheez dobara submit ho
   jaaye, system usse "already done" jaisa treat karta hai, "failed"-jaisa nahi.
6. **Internal-employee ka naam automatically record hota hai** — jab bhi koi internal team
   member (moderator) kisi logo pe action leta hai, unka naam human-readable form mein save
   ho jaata hai audit ke liye.
7. **Buyer-facing displays sirf chhoti (90x90) size use karte hain** — buyer-profile page,
   product-listing page — jitni jagah trace hui, sab sabse chhoti size hi dikhati hain,
   badi sizes ka use kahan hota hai clearly trace nahi hua (dekho Open Questions in
   technical doc).
8. **Koi separate/dedicated "read my logo" API nahi hai** — logo teen alag jagah se read
   hota hai (buyer-profile page, product-listing/PDP, search-results) — ek centralized
   "logo status dekho" screen ka concept code mein directly nahi mila.

---

## 6. Notifications — Downstream Systems Ko Kab Pata Chalta Hai

Yeh feature **supplier-facing email notification** ke bajaye **system-to-system sync** pe
zyada focused hai:

| Kab | Kya hota hai |
|---|---|
| Naya logo upload | Search, LMS, aur general company-sync — teeno ko turant async notify kiya jaata hai |
| Logo update/replace | Same teen systems ko naye images se sync kiya jaata hai |
| Approve/Reject | Same teen systems ko naye status se sync kiya jaata hai |
| Delete/blank | Same teen systems ko clear/blank state se sync kiya jaata hai |
| **Supplier ko email** | **Iss review mein koi supplier-facing email/notification-trigger logo ke liye nahi mila** — GST jaisa "Your logo has been updated" email pattern iss feature mein nahi dikha. Agar supplier ko yeh pata chalta hai, woh shayad UI-level status-check se hi hota hai, email se nahi. |

---

## 7. Edge Cases — Product/Business Perspective se

1. **"Maine logo upload kiya, dikh nahi raha"** — Expected hai agar abhi "Pending" status
   mein hai — approval ka wait karna padega. Koi supplier-facing email bhi nahi jaata isliye
   confusion ho sakta hai — support ko yeh proactively samjhana chahiye.
2. **"Mera logo reject ho gaya, kyun?"** — Rejection-reason field check karo, system usse
   capture karta hai, lekin uske liye bhi email-notification nahi jaata (jaisa upar note
   kiya) — supplier ko khud panel mein dekhna padega.
3. **"Meri logo search-results/listing mein purani dikh rahi hai"** — Approval hone ke baad,
   downstream systems (search, LMS) ko sync hone mein thoda time lag sakta hai — asynchronous
   propagation hai, teen alag systems ek saath update ho rahe hote hain.
4. **"Maine apna already-approved logo change kiya, ab kya status hai?"** — Yeh clearly
   confirm nahi ho paaya ki update-flow status ko wapas Pending pe reset karta hai ya nahi.
   Agar business-intent yeh hai ki "har naya image dobara review ho," iski verify zaroor
   karo team se — abhi ke code-behavior mein yeh guaranteed nahi dikha.
5. **Do baar approve/reject click karna safe hai** — system automatically ignore kar deta hai
   agar logo already decide ho chuka hai, koi galat double-action nahi hoti.
6. **Logo hataane ka matlab record delete nahi hai** — record table mein rehta hai, sirf
   images/reason blank ho jaate hain aur status "Deleted"-jaisa ho jaata hai — history purely
   gone nahi hoti (technical vantage point se).

---

## 8. Quick Summary

- Logo = approval-workflow-based branding image (4 sizes), multi-system async sync.
- Pending → Approved/Rejected/Deleted lifecycle, teeno actions **ek hi API** se hote hain,
  bas parameters alag hote hain.
- Koi supplier-facing email notification nahi hai iss feature ke liye (GST/Rating jaise
  domains ke ulat) — sirf system-to-system sync.
- Approve/Reject double-click-safe hai (idempotent design).
- Image (general photos) se alag table/system hai — dekho Image KT.
- Update-flow ka status-reset-behavior confirm karna zaroori hai (Open Question).

---

## See also

- [`Logo_Technical_Doc.md`](./Logo_Technical_Doc.md) — code-level detail (APIs, DB tables,
  RabbitMQ fan-out, flow-wise DB usage, open questions)
- [`../Image KT/Image_Business_Doc.md`](../Image%20KT/Image_Business_Doc.md) — related but
  separate general-image system
- [`../Profile Meta Template KT/Profile_Meta_Template_Business_Doc.md`](../Profile%20Meta%20Template%20KT/Profile_Meta_Template_Business_Doc.md) —
  broader storefront-content family

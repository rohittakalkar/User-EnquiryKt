# Location Update — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Location_Update_Technical_Doc.md`](./Location_Update_Technical_Doc.md) dekho.

**Note**: Yeh feature supplier ke **geo/address (latitude-longitude + address-components)**
records store karta hai — [`../LatLong to Address KT/`](./) jaisi kisi reverse-geocoding
lookup se alag hai (agar future mein banaya jaaye) — yeh khud data **save** karta hai,
geocoding-lookup nahi karta.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Supplier (mostly mobile-app se) apna **precise geo-location** (latitude/longitude) aur
uske saath detailed address-components (route, street, sublocality, area-levels, zip,
city, state, country) IndiaMART profile ke saath save kar sakta hai. Ek supplier ke
**multiple locations bhi ho sakti hain** — jaise factory ki location, office ki location,
alag-alag — sab ek saath ek batch-request mein bheji ja sakti hain.

**Business impact**: Accurate geo-location buyer ko nearby-supplier-discovery, maps-pe-
directions, aur location-based-search jaisi features ke liye zaroori hai. Multiple-
location-support badi companies ko unke saare addresses represent karne deta hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier (Android/iOS app)** | Apni location(s) submit karta hai |
| **Buyer (indirectly)** | Location-based-search/discovery se benefit leta hai |

---

## 3. Business Flow (step-by-step, no code)

```
1. Supplier apni location submit karta hai — ek single-location (lat/long + address)
   ya multiple-locations ek batch mein
2. System har location ko validate karta hai (mandatory-fields, numeric-checks, valid
   location-type aur status)
3. Har valid location ek naya record ke roop mein insert hoti hai
4. Agar exact-same-coordinates already exist karte hain (same user), duplicate treat
   ho ke skip ho jaata hai — error nahi, silent-skip
5. Batch-request mein, agar ek hi request ke andar duplicate-coordinates bhi ho, unhe
   bhi detect karke skip kiya jaata hai
6. Result — har location ka apna individual outcome (success/duplicate/failed) caller
   ko wapas milta hai
```

**Business impact**: Robust batch-processing — ek request mein multiple-locations
bhejna efficient hai, aur duplicate-handling automatic hai, caller ko manually check
nahi karna padta ki koi location pehle se hai ya nahi.

---

## 4. Business Rules — Plain Language Mein

1. **Sirf "Insert" allowed hai** — is endpoint se location update/edit nahi ho sakti,
   sirf naya add ho sakta hai (`flag` mandatory hai lekin non-insert-flag pe explicit
   "Only Insertion is allowed" message milta hai).
2. **Duplicate-coordinates (same user, same lat/long) silently skip hoti hain**, error
   nahi maani jaati.
3. **Multiple-locations ek batch mein bhej sakte hain** (`is_multiple=1` flag ke saath),
   sath mein ek address-array.
4. **Location-type aur status ke valid ranges hain** — type 1-50 ke beech, status 0-2
   ke beech, warna reject.
5. **Sirf Android/iOS apps allowed hain.**

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine dobara wahi location submit ki, kuch nahi hua"** — Expected hai, duplicate-
   coordinates silently skip hoti hain, error nahi milta.
2. **"Maine 10 locations bheji, sirf 8 insert hui"** — Ho sakta hai 2 duplicate thi
   (ya to already-DB-mein-thi, ya batch ke andar hi repeat thi) — response mein har
   location ka individual-outcome dikhta hai.

---

## 6. Quick Summary

- Location Update = supplier ke geo-locations (lat/long + address-components) ka
  insert-only batch-capable store.
- Duplicate-detection built-in (dono DB-level aur in-request-level).
- Android/iOS-only, multiple-locations-per-request supported.

---

## See also

- [`Location_Update_Technical_Doc.md`](./Location_Update_Technical_Doc.md) — code-level detail

# Location Update — Business Doc (Product Perspective)

Yeh doc Location Update feature ko **business/product nazariye** se samjhata hai — koi code
nahi, sirf "kya hota hai, kyun hota hai, aur supplier ke liye iska matlab kya hai." Technical
implementation (APIs, DB, code) ke liye
[`Location_Update_Technical_Doc.md`](./Location_Update_Technical_Doc.md) dekho.

**Note**: Yeh feature supplier ke **geo/address (latitude-longitude + address-components)**
records store karta hai — do alag concept hain jo confuse ho sakte hain: (1) *save karna*
(yeh feature), aur (2) *lat-long se address nikalna / reverse-geocoding* (ek alag, related
read-endpoint jo isi doc mein technical side pe cover hua hai). Yeh khud data **save**
karta hai; address-lookup uska ek side-capability hai, primary purpose nahi.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Supplier (mostly mobile-app se) apna **precise geo-location** (latitude/longitude) aur uske
saath detailed address-components (route, street, sublocality, area-levels, zip, city,
state, country) IndiaMART profile ke saath save kar sakta hai. Ek supplier ke **multiple
locations bhi ho sakti hain** — jaise factory ki location, office ki location, alag-alag —
sab ek saath ek batch-request mein bheji ja sakti hain.

Yeh sirf ek address field nahi hai — iske peeche maksad hai:

1. **Buyer discovery**: Accurate lat-long se buyer ko nearby-supplier-discovery aur
   location-based-search milta hai.
2. **Maps/directions**: Precise coordinates hone se buyer ko supplier tak directions milna
   easy hota hai.
3. **Multiple locations support**: Badi companies (jinke paas factory alag, office alag, ho
   sakta hai multiple branches ho) apna poora footprint represent kar sakti hain, sirf ek
   single address tak limited nahi.

**Bottom line**: Yeh ek foundational data-capture feature hai jo location-based
discovery/search/maps ke liye zaroori raw material provide karta hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier (Android/iOS app)** | Apni location(s) submit karta hai — sirf yehi do platforms allowed hain |
| **Buyer (indirectly)** | Location-based-search/discovery/maps se benefit leta hai jab supplier location accurate ho |
| **Internal caller (address-lookup ke liye)** | Same underlying data ko ek doosre, read-only endpoint ke through wapas fetch kar sakta hai (jaise app ka "apni saved locations dekho" screen) |

---

## 3. Main Business Flows (step-by-step, no code)

### Flow A — Supplier ek single location submit karta hai

```
1. Supplier app se apni location (lat/long + address details) submit karta hai
2. System check karta hai: sab mandatory fields (lat, long, address, location-type,
   status) sahi format mein hain?
3. Agar sahi hai -> naya location-record insert ho jaata hai
4. Agar exact-same-coordinates (same user) already exist karte hain -> duplicate treat
   ho ke silently skip ho jaata hai, error nahi
```

**Business impact**: Har naya location capture hota hai bina duplicate-clutter ke database
mein, without needing the caller to manually check "is this already saved?"

### Flow B — Supplier multiple locations ek saath batch mein submit karta hai

```
1. Supplier app se ek hi request mein multiple locations bhejta hai (jaise factory +
   office + warehouse, sab ek saath)
2. System har location ko individually validate karta hai
3. Agar batch ke andar hi do locations same coordinates ki hain -> unmein se sirf ek
   process hoti hai, doosri duplicate maani jaati hai
4. Jo bhi valid aur unique hai, unhe parallel mein insert kiya jaata hai (fast processing
   ke liye, especially jab bahut saari locations ek saath aayein)
5. Result — har individual location ka apna outcome (success / duplicate / failed)
   caller ko wapas milta hai, ek combined response mein
```

**Business impact**: Robust batch-processing — ek request mein multiple locations bhejna
efficient hai, duplicate-handling (dono DB-level aur batch-ke-andar) automatic hai, aur
supplier/app ko manually check nahi karna padta ki koi location pehle se hai ya nahi.

### Flow C — Saved locations wapas dekhna (address-lookup)

```
1. Koi caller (app screen, ya internal system) supplier ki saved locations dekhna
   chahta hai
2. System supplier ki saari saved locations database se nikalta hai
3. Agar caller ne ek current lat/long bhi diya hai comparison ke liye, system check
   karta hai: yeh current location kisi saved location ke bahut kareeb hai kya
   (~200 meter ke andar)?
4. Agar kareeb hai -> saved data hi use ho jaata hai, external map-service (jaise Google
   Maps) se dobara address nikalne ki zaroorat nahi padti
5. Agar kareeb nahi hai -> external map-service se fresh address nikala jaata hai
```

**Business impact**: Yeh ek smart-shortcut hai — agar hume already pata hai ki supplier
iss jagah pe hai (kyunki unhone pehle save kiya tha), toh dobara external map-API call
karke time aur cost waste nahi karte.

---

## 4. Business Rules — Plain Language Mein

1. **Sirf "Insert" allowed hai** — is endpoint se location update/edit nahi ho sakti, sirf
   naya add ho sakta hai. Technically system ek "Update" flag bhi accept kar leta hai
   validation stage pe, lekin actual processing mein sirf Insert hi kaam karta hai — Update
   maangne pe clean error milta hai ("Only Insertion is allowed"), koi confusing partial
   behavior nahi.
2. **Duplicate-coordinates (same user, same lat/long) silently skip hoti hain** — na error,
   na kuch special. Yeh dono level pe check hota hai: (a) database mein already-saved
   locations ke against, aur (b) ek hi batch-request ke andar agar do entries same ho.
3. **Multiple-locations ek batch mein bhej sakte hain** (`is_multiple` flag ke saath), sath
   mein ek address-array — factory + office + warehouse jaisa use-case yahi support karta
   hai.
4. **Location-type aur status ke valid ranges hain** — type 1 se 50 ke beech ek number hona
   chahiye (**exact type-1-se-50 ka matlab kya hai, yeh business-side documentation nahi
   mili, sirf ek numeric-range check hai code mein — team se confirm karo agar customer-facing
   meaning chahiye**), status 0 se 2 ke beech, warna request reject.
5. **Sirf Android/iOS apps allowed hain** — web ya doosre channels se yeh directly nahi
   chal sakta.
6. **Wapas-dekhne (read) ke waqt, ek specific status-value ki locations dikhaayi hi nahi
   jaati** — sirf do "achhe" status-values wali locations return hoti hain, teesri
   automatically hide ho jaati hai. **[Iska exact business-meaning confirm karo team se —
   likely "rejected/invalid" ho sakta hai, lekin code mein iska naam/comment nahi hai.]**

---

## 5. Notifications

**Koi email/SMS/push-notification nahi milta** is feature se — na successful insert pe, na
duplicate pe, na validation-failure pe. Yeh sirf ek API-response (success/duplicate/failed
message) ke through hi communicate hota hai, koi background notification supplier ko nahi
jaata. Agar future mein "location saved" jaisa koi user-facing confirmation chahiye, yeh
abhi is feature ka part nahi hai.

---

## 6. Edge Cases — Product/Business Perspective se

1. **"Maine dobara wahi location submit ki, kuch nahi hua"** — Expected hai, duplicate
   coordinates silently skip hoti hain, error nahi milta. Yeh bug nahi hai.
2. **"Maine 10 locations bheji, sirf 8 insert hui"** — Ho sakta hai 2 duplicate thi (ya to
   already-database-mein-thi, ya batch ke andar hi repeat thi) — response mein har
   location ka individual outcome dikhta hai, isliye caller (app) ko exactly pata chal
   sakta hai kaunsi 2 skip hui aur kyun.
3. **"Meri location update nahi ho rahi, sirf naya add ho raha hai"** — Yeh expected
   behavior hai, bug nahi. Yeh feature design se hi insert-only hai — koi existing location
   edit/overwrite nahi karta.
4. **Do alag "location save" concepts platform mein hain** — ek yeh feature (multiple,
   detailed locations, insert-only), aur ek doosra, purana/simpler mechanism jo sirf ek
   latest lat-long track karta hai supplier ke liye. Yeh dono alag purposes serve karte
   hain aur alag data store karte hain — support/product team ko confuse nahi hona chahiye
   agar koi location-related issue aaye, pehle confirm karo kaunsa flow involved hai.

---

## 7. Quick Summary

- Location Update = supplier ke geo-locations (lat/long + address-components) ka insert-only,
  batch-capable store — sirf add hota hai, edit nahi.
- Duplicate-detection built-in hai, dono database-level aur ek-hi-request-ke-andar-level pe —
  koi manual check-pehle-se-hai-ya-nahi ki zaroorat nahi.
- Android/iOS-only, multiple-locations-per-request supported (factory + office jaise
  use-cases ke liye).
- Ek saath-mein-milta read/lookup capability bhi hai jo already-saved locations ko smartly
  reuse karta hai, external map-service calls bachane ke liye jab supplier already-known
  jagah pe ho.
- Koi email/notification supplier ko nahi jaata is feature se — sab kuch synchronous
  API-response ke through hi handle hota hai.

---

## See also

- [`Location_Update_Technical_Doc.md`](./Location_Update_Technical_Doc.md) — code-level detail (APIs, DB table, validation rules, flows)

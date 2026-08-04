# Flips (Social-Commerce Video) — Business Doc (Product Perspective)

Yeh doc Flips feature ko **business/product nazariye** se samjhata hai — koi code nahi,
sirf "kya hota hai, kyun hota hai, aur supplier/business ke liye iska matlab kya hai."
Technical implementation (APIs, DB, queries, RabbitMQ/Kafka, consumer) ke liye
[`Flips_Technical_Doc.md`](./Flips_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

**Flips** IndiaMART ka short-form video / social-commerce feature hai — jaise Instagram
Reels ya YouTube Shorts, lekin B2B products ke liye. Supplier apna external social-media/
Flips-vendor account (YouTube, Instagram, Facebook) IndiaMART se link karta hai, aur unka
video-content IndiaMART pe bhi surface hota hai — buyer ko dikhta hai jaise woh IndiaMART
ka hi native content ho.

1. **Buyer-engagement badhana**: Video-content text/photo se zyada compelling hota hai —
   supplier apne products ko dynamically dikha sakta hai action mein, use mein, ya demo mein.
2. **Supplier ko double kaam nahi karna padta**: Agar supplier already Instagram/YouTube/
   Facebook pe apna content daal raha hai, Flips wahi content IndiaMART pe bhi automatically
   surface kar deta hai — alag se upload nahi karna padta.
3. **IndiaMART khud video-hosting-platform nahi banta**: Actual video-hosting/processing kaam
   external vendor-platform (YouTube/Meta) ka hai — IndiaMART sirf link/metadata store karke
   surface karta hai.
4. **Product-catalog videos bhi isi jagah dikhte hain**: Agar supplier ne apne product-catalog
   ke through directly video upload kiya ho (Flips-account link kiye bina), woh bhi isi Flips
   video-list mein automatically merge ho jaata hai — buyer ko ek hi combined video-gallery
   dikhti hai chahe video kahin se bhi aaya ho.

**Bottom line**: Flips supplier ko apne existing social-media presence ko IndiaMART storefront
pe leverage karne deta hai, bina extra content-creation effort ke.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Apna Flips/vendor-account link karta hai (token ya Meta-login se), video-content post/update karta hai |
| **External vendor-platform (YouTube/Instagram/Facebook)** | Actual video-hosting aur processing IndiaMART ke bahar hota hai |
| **Buyer** | Video-content dekhta hai supplier ke storefront pe, product-catalog videos ke saath combined |
| **Background sync-process (automated)** | Periodically ek external content-feed se supplier ke Flips-content ko latest state mein sync karta hai (naye videos add, hataye gaye videos disable, update hue videos refresh) |

---

## 3. Business Flows (step-by-step, no code)

### Flow A — Supplier apna Flips account link karta hai (Token-based)

```
1. Supplier (ya unki taraf se ek integration/app) IndiaMART ko apna vendor-platform ka
   access-token deta hai
2. System token ko store kar leta hai supplier ke account ke saath
3. Ab supplier ka video-content is token ke through fetch/sync ho sakta hai
```

**Business impact**: Simplest linking-path — jaha supplier ke paas already ek vendor-API
token hai (jaise koi integration partner ke through), yeh sabse direct route hai.

### Flow B — Supplier apna Flips account link karta hai (Meta/social-login based)

```
1. Supplier Facebook/Instagram business-login se apna social-media account link karta hai
2. System check karta hai: yeh exact social-account pehle se kisi doosre profile se
   link toh nahi hai (duplicate-prevention)
3. Agar naya hai -> account-detail record ban jaata hai, "active" maana jaata hai
4. Agar isi platform ka koi purana linked-account tha isi supplier ke, woh automatically
   "inactive" ho jaata hai (ek time pe sirf ek active account per platform)
5. Downstream systems ko notify kiya jaata hai account-status change ki
```

**Business impact**: Fraud/confusion prevention hai — ek supplier ka ek platform pe sirf ek
active linked-account hona chahiye, taaki confusion na ho ki kaunsa account "real" hai.

### Flow C — Supplier apna video-content post karta hai

```
1. Supplier naya video-content post karta hai (ek ya multiple videos ek saath, batch mein)
2. System duplicate-content check karta hai (already-linked video dobara insert nahi hoti)
3. Naya content record ban jaata hai, "enabled" status ke saath
4. Downstream systems (buyer-facing display) ko notify kiya jaata hai
```

**Business impact**: Supplier apna naya content jaldi surface kar sakta hai IndiaMART pe,
bina manual review-wait ke.

### Flow D — Supplier apna existing video-content update karta hai

```
1. Supplier ek existing video ki details update karta hai (title, description, view-count,
   thumbnail, etc.) — jo sirf woh fields deta hai wahi update hoti hain, baaki as-is rehti hain
2. System change ko apply karta hai
3. Downstream systems ko notify kiya jaata hai
```

**Business impact**: Content freshness maintain hoti hai — agar view-count ya title
vendor-platform pe change ho, IndiaMART pe bhi reflect ho jaata hai.

### Flow E — Background sync (automated, external feed se)

```
1. Ek automated background-process periodically supplier ke Flips-content ka latest snapshot
   receive karta hai (external source se)
2. System dekhta hai: kaunse videos naye hain (insert), kaunse update hue hain, aur kaunse
   ab available nahi hain (disable karo)
3. Sirf jo actually change hua hai, wahi apply hota hai — poora data dobara nahi likha jaata
4. Yeh sab wahi same "post/update" business-rules se guzarta hai jo supplier khud use karta
   hai (koi shortcut/bypass nahi)
```

**Business impact**: Yeh ek safety-net/freshness-mechanism hai — agar kisi wajah se real-time
update miss ho jaaye (jaise supplier ne vendor-platform pe seedha content delete kar diya
bina IndiaMART ko batae), yeh background sync usse catch karke IndiaMART ke display ko bhi
update kar deta hai, automatically.

### Flow F — Buyer video dekhta hai (read)

```
1. Buyer supplier ka storefront/profile visit karta hai
2. System do sources se video-content nikalta hai:
   a. Flips-linked-account ka content (Flow A-E se)
   b. Supplier ke product-catalog mein directly-uploaded videos
3. Dono ko ek hi combined, de-duplicated list mein dikhaya jaata hai buyer ko
```

**Business impact**: Buyer ko ek unified experience milta hai — usse fark nahi padta video
kahan se aaya, sab ek jagah dikhta hai.

---

## 4. Business Rules — Plain Language Mein

1. **Do authentication-modes support hote hain** — "Token" (direct vendor-API access) ya
   "Meta" (Facebook/Instagram business-login), jo bhi supplier/integration use kare.
2. **Ek account-type sirf ek hi active-token rakh sakta hai** — naya token daalne se purana
   silently replace ho jaata hai, koi warning nahi.
3. **Duplicate social-account link nahi ho sakta** — agar same social-media-account (jaise
   same Instagram handle) already kisi profile se linked hai, dobara link karne ki koshish
   reject hoti hai.
4. **Ek platform pe sirf ek active linked-account hota hai per supplier** — naya link karne
   se purana automatically inactive ho jaata hai, taaki confusion na ho.
5. **Duplicate video-content silently skip ho jaata hai, error nahi deta** — agar wahi video
   dobara post karne ki koshish ho (system-level unique check), system usse chup-chaap ignore
   kar deta hai.
6. **Video-update partial ho sakta hai** — supplier sirf woh fields de sakta hai jo change
   hue hain; baaki fields as-is reh jaati hain, unhe dobara bhejna zaroori nahi.
7. **Background-sync koi shortcut nahi leta** — jo bhi automated sync content post/update
   karta hai, woh bhi wahi same validation/business-rules se guzarta hai jo supplier ke apne
   direct request se guzarte hain.
8. **Product-catalog videos automatically Flips-list mein bhi shaamil hote hain** — supplier
   ko alag se Flips ke through post nahi karna, agar usne catalog ke through video daala hai.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine account link kiya, phir bhi purana account active dikha raha hai kisi aur jagah"**
   — Ek platform pe sirf ek hi account ek time pe active ho sakta hai per supplier; naya link
   karne se purana automatically inactive ho jaata hai. Yeh expected hai.
2. **"Mera video content nahi dikh raha"** — Ho sakta hai duplicate-detection ki wajah se
   silently skip ho gaya ho (already same video-ID exist karta ho), ya background sync ne
   usse "unavailable" maar ke disable kar diya ho kyunki woh ab external feed mein nahi mila.
3. **"Maine sirf title change kiya, baaki details bhi refresh ho gayi"** — Update partial hona
   chahiye tha (sirf diye gaye fields update hote hain); agar poora record refresh hua dikhe,
   check karo request mein extra fields toh nahi bheji gayi thi.
4. **"Video kahin se bhi upload kiya ho, ek hi jagah dikhta hai"** — Yeh intentional hai:
   product-catalog aur Flips-linked-account dono se aane wale videos ek combined list mein
   merge hote hain, duplicate hone par bhi sirf ek copy dikhti hai.
5. **Background-sync delay ho sakta hai** — real-time update aur background-sync ke beech agar
   koi naya direct-update aaye, timing ki wajah se thoda race ho sakta hai; agar content
   "flip-flop" karta dikhe (jaldi-jaldi change), yeh iska ek possible reason hai.

---

## 6. Quick Summary — Ek Line Mein Har Cheez

- Flips = short-video/social-commerce, external-vendor-hosted, IndiaMART storefront pe surface hoti hai.
- Account-link (Token ya Meta) → Content-post/update, do-step process.
- Ek platform pe sirf ek active account per supplier — auto-managed.
- Duplicate content/account silently handle ho jaata hai, error/crash nahi hota.
- Background-sync automated safety-net hai jo external feed se latest state ko catch-up
  karta hai — wahi rules follow karta hai jo direct supplier-request follow karte hain.
- Product-catalog videos aur Flips-linked-account videos ek hi combined gallery mein
  buyer ko dikhte hain.

---

## See also

- [`Flips_Technical_Doc.md`](./Flips_Technical_Doc.md) — code-level detail (APIs, DB
  tables, queries, RabbitMQ/Kafka, consumer)
- [`../ecommerce_social_commerce_product_overview.md`](../ecommerce_social_commerce_product_overview.md) —
  bigger Ecommerce & Social-Commerce story, Flips originally documented yahan summary-level

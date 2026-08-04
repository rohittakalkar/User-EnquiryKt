# Market Place URL — Business Doc (Product Perspective)

Yeh doc Market Place URL feature ko **business/product nazariye** se samjhata hai — koi
code nahi, sirf "kya hota hai, kyun hota hai, aur supplier/business ke liye iska matlab kya
hai." Technical implementation (APIs, DB, queues, code) ke liye
[`MarketPlaceUrl_Technical_Doc.md`](./MarketPlaceUrl_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Supplier apni company ke **third-party marketplace/social presence ke URLs** — Google
(Business listing), Facebook (page), Instagram (account) — IndiaMART profile ke saath link
kar sakta hai.

Yeh business-value do tarah se deta hai:

1. **Buyer-trust signal**: Agar supplier ka established Google-listing, Facebook-page, ya
   Instagram-presence hai, yeh ek "third-party verification" jaisa signal ban jaata hai —
   buyer ko lagta hai yeh ek real, discoverable business hai, sirf IndiaMART pe hi nahi.
2. **Data-enrichment ka base**: In URLs ko internal tools/admin bhi add/update kar sakte
   hain (supplier khud submit kiye bina) — matlab yeh feature sirf supplier-driven nahi hai,
   IndiaMART ki apni data-enrichment pipeline ka bhi hissa ban sakta hai (dekho Technical
   Doc ka `ENRICH_FLAG`/`ENRCH_*` columns discussion).

**Bottom line**: Yeh ek chhota, focused feature hai — teen specific platforms ke liye ek
simple linking mechanism, jo history-tracking ke saath aata hai (kisne, kab, kahan se
update kiya).

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Apna Google/Facebook/Instagram URL apne app/panel se add ya update karta hai |
| **Internal Admin (GLADMIN)** | Supplier ki taraf se bhi yeh URLs enrich/update kar sakta hai |
| **WebERP (internal tool)** | Company-data management tool, isse bhi yeh URLs likhe ja sakte hain |
| **Seller App (`SELLERMY`)** | Supplier ke apne seller-app se bhi yeh submit ho sakta hai |
| **Buyer** | Supplier ka profile dekhte waqt yeh links dikh sakte hain (read side) |

**Business rule jo yaad rakhne layak hai**: Sirf char specific "gateway" (submitting-system)
allowed hain iss data ko likhne ke liye — `GLADMIN`, `Weberp`, `SELLERMY`, `MY` (supplier's
own app). Koi bhi random internal tool ya app iss data ko directly nahi likh sakta — ek
access-control jaisa mechanism already hai.

---

## 3. Business Flows (step-by-step, no code)

### Flow A — Supplier pehli baar apna marketplace-URL add karta hai (Insert)

```
1. Supplier apne app/panel se ek URL submit karta hai — bataata hai kaunsa platform hai
   (Google, Facebook, ya Instagram) aur URL value kya hai
2. System check karta hai: yeh ek valid, supported platform hai? (sirf 3 allowed)
3. System yeh bhi check karta hai ki request kis "gateway" (app/tool) se aayi hai — sirf
   allowed gateways (supplier apna app / GLADMIN / WebERP / SELLERMY) hi likh sakte hain
4. Sab sahi hone par, ek naya record ban jaata hai — URL, kis IP se, kaunsi screen se,
   kab, sab capture hota hai (history-trail ke liye)
5. Supplier ko response mil jaata hai "successfully added"
```

**Business impact**: Supplier apna Google/Facebook/Instagram presence link kar paata hai,
jo buyer-facing profile pe dikhta hai — ek chhota lekin trust-building addition.

### Flow B — Supplier apna already-added URL update karta hai (Update)

```
1. Supplier same platform (jaise Instagram) ke liye naya URL submit karta hai
2. System purane record ko dhoondhta hai (same user + same platform)
3. Do tarah se update ho sakta hai:
   a. Poora URL-value change karna (naya link)
   b. Sirf "enrichment metadata" update karna (jaise internal tool ne yeh record verify/
      enrich kiya, lekin URL value khud nahi badla) — yeh distinction system internally
      ek ID-flag se decide karta hai
4. History/comments trail is update ka bhi record hota hai — purana value poori tarah
   overwrite nahi hota, audit-trail-jaisa data preserve rehta hai
5. Response: "successfully updated"
```

**Business impact**: Supplier apna outdated ya galat URL fix kar sakta hai. Internal
enrichment-tools bhi bina URL-value disturb kiye sirf metadata/verification-status update
kar sakte hain — matlab do alag "update" use-cases ek hi API se serve hote hain.

### Flow C — Buyer supplier ka profile dekhta hai (Read)

```
1. Buyer kisi supplier ka profile open karta hai
2. System uss supplier ke liye Google/Facebook/Instagram — teeno platforms ke "latest" URL
   dhoondhta hai (agar available hon)
3. Sirf ek response milta hai, jisme har platform ka sabse recent/current URL hota hai —
   purane, superseded URLs buyer ko nahi dikhte
```

**Business impact**: Buyer ko hamesha ek clean, up-to-date view milta hai supplier ke
marketplace-presence ka, bina kisi confusing old-data ke.

---

## 4. Business Rules — Plain Language Mein

1. **Sirf 3 platforms supported hain** — Google, Facebook, Instagram. Koi aur platform
   (jaise LinkedIn, Twitter) system ke liye "unrecognized" ban jaata hai — system usse
   silently ek generic/zero category mein daal deta hai, error clearly nahi deta.
2. **Naya URL add karne ke liye URL value dena zaroori hai** — khaali URL ke saath "naya add
   karo" request reject ho jaati hai.
3. **Update karte waqt URL value optional hai** — matlab sirf metadata (jaise verification-
   status, internal-comments) bhi update ho sakti hai, bina actual link change kiye. Yeh
   khaas taur pe internal enrichment-tools ke liye useful hai.
4. **Sirf allowed gateways (systems/apps) hi likh sakte hain** — supplier ka apna app (`MY`),
   internal admin panel (`GLADMIN`), WebERP, ya Seller App (`SELLERMY`). Koi aur system agar
   yeh data likhne ki koshish kare, reject ho jaata hai "gateway not validated" ke saath.
5. **Ek user ke ek platform ka sirf "latest" URL hi buyer ko dikhta hai** — agar history mein
   multiple updates hue hon (jaise supplier ne Instagram URL 3 baar change kiya), sabse
   recent wala hi surface hota hai, purane sab background mein preserve rehte hain.
6. **History/audit trail har change ke saath capture hoti hai** — kisne update kiya, kis IP
   se, kaunsi app/screen se, koi comments — yeh sab record hota hai, sirf latest value nahi.

---

## 5. Notifications

**Iss feature ke liye koi supplier-facing email/SMS notification nahi mili review mein** —
Marketplace-URL add/update hone par supplier ko koi separate communication nahi jaata jaisa
GST ke case mein hota hai. Yeh ek low-stakes, "supplier khud manage karta hai" type field
lagta hai jahan turant confirmation ki zaroorat business ne feel nahi ki.

| Kab | Kya notification jaata hai |
|---|---|
| URL add/update hua | **Koi nahi mila** — API response hi seedha supplier ke app/panel mein reflect hota hai |

---

## 6. Edge Cases — Product/Business Perspective se

1. **"Maine Instagram URL update kiya, purana wala dikh raha hai"** — Ho sakta hai propagation-
   delay ho (agar read aur write alag database-replica se ho rahe hain), ya phir kisi wajah
   se update record properly latest na maana gaya ho. Support ko batao ki "latest-per-
   platform" dedup logic backend mein hai — dekho Technical Doc.
2. **"Maine LinkedIn ka URL add karne ki koshish ki, allow nahi hua"** — Expected hai, sirf
   Google/Facebook/Instagram abhi supported hain. Product roadmap ka item ho sakta hai agar
   demand ho.
3. **"Maine apna URL app se update kiya, lekin kisi aur channel (jaise WebERP) se bhi wahi
   record change ho sakta hai"** — Haan, kyunki 4 alag gateways (supplier app, GLADMIN,
   WebERP, SELLERMY) sabhi isi record ko likh sakte hain. Agar do updates race kar jaayein
   (jaise supplier aur internal-tool dono ek saath update karein), jo baad mein aaya wahi
   final maana jaata hai.

---

## 7. Quick Summary

- Marketplace-URL = supplier ke Google/Facebook/Instagram presence links, IndiaMART profile
  ke saath linked.
- Insert-or-update per platform, history/comments trail ke saath.
- Sirf 3 platforms support hote hain — baaki sab "unrecognized" treat hote hain.
- Sirf 4 allowed gateways (`MY`, `GLADMIN`, `Weberp`, `SELLERMY`) likh sakte hain.
- Buyer ko hamesha sirf latest URL (per platform) dikhta hai, purane history mein preserve
  rehte hain.
- Koi supplier-facing notification nahi jaati add/update pe.

---

## See also

- [`MarketPlaceUrl_Technical_Doc.md`](./MarketPlaceUrl_Technical_Doc.md) — same flows,
  code-level detail (APIs, DB tables, RabbitMQ consumer, queries)

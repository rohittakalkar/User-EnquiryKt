# KWPL — Business Doc (Product Perspective)

Yeh doc KWPL feature ko **business/product nazariye** se samjhata hai — koi code nahi, sirf
"kya hota hai, kyun hota hai, aur internal-team ke liye iska matlab kya hai." Technical
implementation (APIs, DB, code) ke liye [`KWPL_Technical_Doc.md`](./KWPL_Technical_Doc.md)
dekho.

**Note**: KWPL aur Disposition genuinely **do alag concepts** hain — koi shared table,
FK-relation, ya common write-controller nahi mila. Dekho
[`../Disposition KT/Disposition_Business_Doc.md`](../Disposition%20KT/Disposition_Business_Doc.md).

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

**KWPL** ek internal record-keeping mechanism hai jo track karta hai ki kisi supplier/
company ke against **kaunse keywords, kis city mein, kis "work-order" ke against, kab se
kab tak "served" (assign/serve) kiye gaye** — yeh ek **paid-visibility / paid-listing
keyword** feature hai (ek internal reference doc mein isse "Paid listing keywords (PL/KWPL)"
bola gaya hai — codebase ke andar hi table ka naam bhi `PL_KWRD` hai, matlab
"Paid-Listing KeyWoRD").

Simple bhasha mein: jab koi supplier kisi paid-visibility package ya campaign ke through
kisi specific keyword ke liye "boost"/paid placement paata hai (jaise "iron rods
manufacturer" jaisa keyword unke liye kisi city mein paid-highlight ho), toh us record ko
yahan track kiya jaata hai — kaunsa keyword, kaunsi city, kab se kab tak valid, aur kaunse
"work-order" (internal sales/ops transaction) ke against.

Yeh **customer-facing feature nahi hai** — supplier khud isse directly nahi dekh/edit karta.
Yeh purely ek internal sales/ops/campaign-tracking-tool jaisa lagta hai jo bataata hai "yeh
keyword iss supplier ko iss duration ke liye assign/serve kiya gaya".

**Business impact**: Sales/ops teams ke liye traceability — kaunsa keyword, kis city mein,
kis work-order ke against, kab se kab tak "enable" tha — yeh audit-trail internal-processes
(paid campaign-tracking) ko support karta hai. Bina isse, kisi paid-keyword-placement ka koi
system-of-record nahi hota — "humne is supplier ko yeh keyword kab tak diya tha" jaisa sawaal
answer karna mushkil ho jaata.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Internal admin/ops tools (GLADMIN, WebERP)** | Keyword-serve records insert/update/disable karte hain — code mein explicitly sirf inhi do internal-systems ko allow kiya gaya hai |
| **Supplier (indirectly)** | Jiske against yeh keyword-serve record ban raha hai — supplier khud iss API ko call nahi karta, na hi isse directly kahin dekh sakta hai (koi customer-facing read-surface nahi mila) |
| **Sales/Ops/Campaign team (implied)** | GLADMIN/WebERP tools ke peeche jo actual insaan/process hai jo decide karta hai kaunsa keyword kis supplier ko serve karna hai — yeh conclusively code se confirm nahi hota, par gateway-restriction (sirf internal tools) se strongly implied hai |

---

## 3. Business Flow (step-by-step, no code)

### Flow A — Naya keyword-serve record create karna (INSERT)

```
1. Internal-tool (GLADMIN/WebERP) ek naya keyword-serve record create karta hai — supplier
   (GLUSRID), city, company, keyword (PL_KEYWORDS), description, validity-period
   (from-to dates), aur ek "work-order ID" ke saath
2. System check karta hai ki saari zaroori details di gayi hain (jaise work-order ID,
   dates, company, keyword) — agar koi missing hai, request reject ho jaati hai
3. Record ban jaata hai aur "enabled" (active) state mein rehta hai
```

**Business impact**: Ek naya paid-keyword-placement officially "on record" ho jaata hai —
kis supplier ko kaunsa keyword, kis city mein, kab tak diya gaya, sab trackable hai.

### Flow B — Record disable karna (DISABLE)

```
1. Internal-tool ek existing active record ko disable karne ki request bhejta hai
   (supplier + "serve ID" ke through identify karke)
2. System sirf currently-active (enabled) record ko hi disable karta hai
3. Agar record already disabled tha, kuch nahi hota (silently no-op)
```

**Business impact**: Jab koi paid campaign khatam ho jaaye, ops team usse formally band kar
sakti hai — audit-trail mein reflect hota hai ki yeh keyword-serve ab active nahi hai.

### Flow C — Record update karna (UPDATE)

```
1. Internal-tool ek existing active record ki validity-end-date aur/ya work-order ID
   update karne ki request bhejta hai
2. System sirf currently-active record ko hi update karta hai
```

**Business impact**: Agar koi campaign extend karni ho (jaise validity badhani ho), yeh
existing record ko modify karta hai bina naya record banaye.

### Flow D — Sirf keyword-term change karna (SEARCH_UPDATE)

```
1. Internal-tool sirf keyword-term (PL_KEYWORDS) change karne ki request bhejta hai —
   record ID se, ya supplier+serve-ID combination se
2. System sirf currently-active record ka keyword-term update karta hai, baaki fields
   waise hi rehte hain
```

**Business impact**: Agar sirf keyword galat tha ya change karna ho (jaise typo fix, ya
keyword-strategy change), poora record dobara banaye bina sirf keyword-term hi update ho
jaata hai.

---

## 4. Business Rules — Plain Language Mein

1. **Chaar actions supported hain**: INSERT (naya record), DISABLE (band karna),
   UPDATE (validity/work-order extend), SEARCH_UPDATE (sirf keyword-term change).
2. **DISABLE/UPDATE/SEARCH_UPDATE sirf currently-"enabled" (active) records ko target
   karte hain** — already-disabled record dobara touch nahi hota. Agar koi already-disabled
   record ko dobara disable/update karne ki koshish ho, system bina error diye chup-chaap
   kuch nahi karta.
3. **Sirf do internal-systems allowed hain** — GLADMIN aur WebERP, matlab yeh purely
   internal/admin-driven feature hai, koi supplier-facing ya public API nahi.
4. **Har action ke apne alag mandatory-fields hain** — jaise INSERT ke liye keyword,
   company, dates, work-order sab chahiye; DISABLE ke liye sirf enable-flag chahiye;
   UPDATE ke liye work-order aur end-date chahiye; SEARCH_UPDATE ke liye naya keyword aur
   updater-ID chahiye.
5. **System hamesha "success" bolta hai jab tak koi hard database-error na aaye** — chahe
   koi record actually match/update hua ho ya nahi (jaise galat supplier/serve-ID combo diya
   ho jisse koi row match hi na ho), response phir bhi success dikhata hai. Yeh ek
   important caveat hai jo support/ops team ko pata hona chahiye — "success" response ka
   matlab yeh nahi guarantee karta ki koi actual record change hua.
6. **Koi email/notification nahi jaata** — na supplier ko, na internal team ko, kisi bhi
   action ke liye (INSERT/DISABLE/UPDATE/SEARCH_UPDATE). Yeh purely ek silent internal
   record-keeping operation hai.

---

## 5. Notifications — Kise Kab Pata Chalta Hai

| Kab | Kya notification jaata hai |
|---|---|
| Naya keyword-serve record create hua | **Koi nahi** — na supplier ko, na kisi internal team ko email/SMS jaata hai |
| Record disable/update hua | **Koi nahi** |
| Koi action fail hua | **Koi nahi** — sirf API response mein error dikhta hai, calling tool (GLADMIN/WebERP) ke UI mein hi dikhega, koi separate alert nahi jaata |

Yeh purely ek "system of record" hai, koi communication-layer iske saath nahi juda hai.

---

## 6. Edge Cases — Product/Business Perspective se

1. **"Maine keyword-record disable/update karne ki koshish ki, kuch update nahi hua, phir
   bhi success dikha"** — Yeh expected behavior hai. System sirf currently-active records ko
   match karta hai, aur agar match hi nahi hua (jaise record already disabled tha, ya galat
   supplier/serve-ID diya), phir bhi response "success" dikha sakta hai. Yeh ek known gap
   hai — support team ko yeh samjhana chahiye ki "success" response guarantee nahi karta ki
   record actually change hua.

2. **"Yeh data kahan dikhta hai?"** — Koi dedicated dashboard/read-API iss codebase mein
   nahi mila. Yeh data kahan consume/display hota hai (ya seedha database se dekha jaata
   hai), yeh conclusively pata nahi chal paya is review mein — internal team se confirm
   karna hoga.

3. **Supplier ko khud iska pata nahi chalta** — koi email/notification nahi jaata, aur
   supplier-facing koi surface bhi nahi mila jahan yeh data dikhe. Yeh poori tarah ek
   internal/backend record-keeping process hai.

---

## 7. Quick Summary

- KWPL = internal, paid-keyword-listing serve-record against supplier + city + work-order
  (naam ka expansion conclusively confirm nahi hua, lekin "Paid Listing Keyword" strongly
  indicated hai).
- 4 actions: Insert / Disable / Update / Search-term-update.
- Purely internal-tool-driven (GLADMIN/WebERP only) — koi supplier-facing surface nahi, koi
  notification nahi.
- "Success" response ka matlab guaranteed record-change nahi hai — ek known gap jo support
  team ko pata hona chahiye.
- Koi dedicated read-surface/dashboard iss codebase mein nahi mila — is data ko kaun/kaise
  consume karta hai, yeh open question hai.
- Disposition se koi relation nahi — alag concept, alag table.

---

## See also

- [`KWPL_Technical_Doc.md`](./KWPL_Technical_Doc.md) — code-level detail
- [`../Disposition KT/Disposition_Business_Doc.md`](../Disposition%20KT/Disposition_Business_Doc.md) —
  unrelated concept, documented separately
- [`../GST KT/GST_Business_Doc.md`](../GST%20KT/GST_Business_Doc.md) — sibling KT doc, same
  depth/structure convention

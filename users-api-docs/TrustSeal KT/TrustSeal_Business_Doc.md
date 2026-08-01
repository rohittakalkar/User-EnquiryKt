# TrustSeal (Seal, Company, Misc & Trade Affiliations) — Business Doc

Business/product perspective ke liye yeh doc hai. Code ke liye
[`TrustSeal_Technical_Doc.md`](./TrustSeal_Technical_Doc.md) dekho.

**Scope note**: yeh doc **TrustSeal aur uski saari linked details** (Company info, Misc
details, Trade Affiliations) ek saath cover karta hai — yeh sab ek hi table-family hai
(`TRUSTSEAL_ID` se joined), ek hi write-controller se manage hoti hai. **Trust Verification**
(generic per-attribute verification — GST/bank/mobile/email) ek alag, broader system hai —
dekho [`../Trust Verification KT/`](../Trust%20Verification%20KT/Trust_Verification_Business_Doc.md).

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

**TrustSeal** IndiaMART ka premium trust-badge hai — ek supplier jo TrustSeal certified hai,
unke storefront pe ek special badge dikhta hai jo buyers ko batata hai "yeh supplier extra
verified/trusted hai." Yeh GST-verification (locked/tactical/OTP) se **alag, upar ka layer**
hai — TrustSeal ek paid/premium trust-signal hai, jabki basic verification free hai.

TrustSeal apne saath extra details bhi carry karta hai:
- **Company details** — kaunsi company(s) is seal ke associated hain
- **Misc details** — Trade Affiliations (jaise industry associations ki membership), Safety
  Certifications, Chamber-of-commerce memberships

**Business impact**: TrustSeal buyer ko ek quick, visual "yeh supplier extra credible hai"
signal deta hai — jaise ek verified-badge Twitter/Instagram pe hota hai, waisa hi concept.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **IndiaMART internal system (WEBERP)** | TrustSeal assign/manage karta hai — **supplier khud apne aap ko TrustSeal nahi de sakta**, yeh purely internal-admin-driven hai |
| **Supplier** | TrustSeal ka beneficiary — badge unke profile pe dikhta hai |
| **Buyer** | TrustSeal badge dekh ke trust-decision leta hai |

---

## 3. Business Flow (step-by-step, no code)

```
1. Internal system (WEBERP) ek naya TrustSeal create karta hai ek supplier ke liye
   (ya existing seal ko renew/update karta hai)
2. TrustSeal ke saath company-details link hoti hain — kaunsi company iske under aati hai
3. Optionally, Trade Affiliations, Safety Certifications, Chamber memberships bhi add ki
   ja sakti hain seal ke against
4. Har change ka ek history/audit-trail maintain hota hai (kya text change hua, kab)
5. TrustSeal ka apna approval-status hota hai (jaise "Manual approved," "Auto approved" —
   exact states iss review mein pura decode nahi hue)
6. TrustSeal ki ek expiry/end-date hoti hai — renewal ka concept hai
7. Buyer jab supplier profile dekhta hai, TrustSeal badge aur uske details (company,
   trade-affiliations) sab dikhte hain
```

**Business impact**: Yeh ek **subscription-jaisa** concept hai (`FK_CUST_TO_SERV_ID` — customer-
to-service link mila code mein) — matlab TrustSeal kisi paid-service se linked ho sakta hai,
sirf ek free badge nahi.

---

## 4. Business Rules — Plain Language Mein

1. **Sirf internal `WEBERP` system TrustSeal assign kar sakta hai** — koi bhi doosra caller
   (seller-panel, app, ya bhi GLADMIN nahi) iss specific write-API ko access nahi kar sakta.
   Yeh sabse tight-gated write-path hai teeno "trust" domains (GST, TrustSeal, Verification)
   mein.
2. **TrustSeal ki ek "end date" hoti hai** — matlab yeh permanent nahi hai, expire ho sakta
   hai, renew hone ki zaroorat pad sakti hai.
3. **Har change ka history maintain hota hai** — jab bhi kuch update ho (seal khud, ya
   company-link), ek text-based history-log bhi update hota hai, taaki audit-trail rahe.
4. **Company-level linking multiple companies support karta hai** — ek TrustSeal ke against
   ek se zyada company-records link ho sakte hain, ek "primary" flag ke saath (kaunsi company
   primary hai).
5. **Trade Affiliations optional add-on hain** — TrustSeal base-record ke bina bhi yeh exist
   nahi kar sakte (foreign-key dependent), lekin har TrustSeal ke saath yeh hona mandatory
   nahi.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine GST verify karwaya, mujhe TrustSeal kyun nahi mila"** — Yeh do alag systems hain.
   GST-verification (dekho [`../GST KT/`](../GST%20KT/GST_Business_Doc.md)) ek prerequisite
   ho sakti hai TrustSeal ke liye, lekin TrustSeal khud sirf WEBERP (internal team) assign
   karti hai — koi automatic trigger GST-verify se TrustSeal tak code mein nahi mila.
2. **"Mera TrustSeal expire ho gaya, badge gayab ho gaya"** — Expected hai agar `end date`
   guzar chuki hai — renewal process (jo iss review ke scope se bahar hai) internal team se
   consult karna padega.
3. **"Meri company-details TrustSeal pe galat dikh rahi hain"** — TrustSeal Company records
   multiple ho sakte hain ek seal ke against — check karo "primary" company kaunsi mark hai.

---

## 6. Quick Summary

- TrustSeal = ek premium, internal-admin-assigned trust-badge, GST-verification se upar ka
  layer.
- Company-details, Trade-Affiliations, Misc-details sab TrustSeal se FK-linked hain — ek hi
  family.
- Sirf `WEBERP` (internal system) isse manage kar sakta hai — supplier ya GLADMIN bhi nahi.
- Expiry/renewal concept hai, subscription-jaisa (`CUST_TO_SERV` link).
- Har change ka audit-history maintain hota hai.

---

## See also

- [`TrustSeal_Technical_Doc.md`](./TrustSeal_Technical_Doc.md) — code-level detail
- [`../Trust Verification KT/Trust_Verification_Business_Doc.md`](../Trust%20Verification%20KT/Trust_Verification_Business_Doc.md) —
  related but separate: generic per-attribute verification system
- [`../trust_verification_compliance_product_overview.md`](../trust_verification_compliance_product_overview.md) —
  bigger Trust, Verification & Compliance story

# Verification Process (Attribute Verification) — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Verification_Process_Technical_Doc.md`](./Verification_Process_Technical_Doc.md) dekho.

**Note**: Yeh Trust Verification KT se **related but distinct** hai — Trust Verification
KT single-attribute-status read/write (`UserVerificationController`,
`USER_APPROVAL_ATTRIBUTE.go`'s `VerifyAttr` HTTP-loopback) cover karta hai. Yeh KT ek
**broader, structured attribute-verification-event-recording system** cover karta hai —
jahan ek verification-event **primary-attribute + saare related secondary-attributes**
ko ek saath, cross-linked record karta hai. Dekho
[`../Trust Verification KT/Trust_Verification_Business_Doc.md`](../Trust%20Verification%20KT/Trust_Verification_Business_Doc.md).

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Jab supplier ka koi **business-attribute** (mobile, email, GST, PAN, CIN, TAN,
company-name, bank-account, Aadhar, address, city, state, name, IEC-code, CEO-name,
zip, Udyam-registration, lat/long) kisi internal ya external-process se **verify**
hota hai, yeh feature is verification-event ko record karta hai — kaunsa attribute,
kya value, kis source se, kis platform se, kab verify hua, aur status kya raha.

Ek verification-event mein ek **"primary" attribute** (jaise mobile) aur usse linked
**"secondary" attributes** (jaise woh mobile kis email/GST ke saath verify hua) dono
record ho sakte hain — cross-referencing ke liye.

**Business impact**: Yeh IndiaMART ke trust-ecosystem ka structured-data-foundation hai
— TrustSeal-eligibility, fraud-detection, aur supplier-credibility-scoring sab is
verification-history pe depend karte hain.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Internal-verification-processes (WEBERP, SOA-WAPI)** | Verification-events record karte hain |
| **Supplier (indirectly)** | Jiske attributes verify ho rahe hain |
| **Downstream trust/TrustSeal-systems** | Yeh verification-history read/consume karte hain |

---

## 3. Business Flow (step-by-step, no code)

```
1. Koi internal-process (verification-team, external-verification-service-integration)
   ek attribute (jaise mobile-number) verify karta hai
2. System check karta hai — yeh attribute ek predefined-list mein hai ya nahi (18
   supported attribute-categories: mobile, email, GST, PAN, CIN, TAN, company-name,
   bank-account, Aadhar, address, city, state, name, IEC, CEO-name, zip, Udyam, lat-long)
3. Agar sirf primary-attribute hai (koi related-secondary nahi), ek single
   verification-record banta hai
4. Agar secondary-attributes bhi hain (jaise "yeh mobile is email ke saath verify
   hua"), har secondary ke liye ek alag cross-linked record bhi banta hai
5. Baad mein, kisi bhi supplier ke against, unke saare verified-attributes query kiye
   ja sakte hain
```

**Business impact**: Cross-linked-verification (primary+secondary) ek rich data-model
hai — sirf "yeh verified hai" nahi, balki "yeh attribute is doosre-attribute ke context
mein verified hua" — jo fraud-pattern-detection aur cross-attribute-consistency-checks
ke liye valuable hai.

---

## 4. Business Rules — Plain Language Mein

1. **Sirf predefined 18 attribute-categories supported hain** — koi random attribute-ID
   accept nahi hota.
2. **Primary-attribute-value mandatory hai** — bina primary-value, koi record nahi
   banta.
3. **Har secondary-attribute apna alag cross-linked-record banata hai** — batch mein
   multiple-secondary-attributes ek hi request mein bheje ja sakte hain.
4. **Sirf do internal-systems allowed hain** — WEBERP, SOA-WAPI.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine verification-record submit kiya, invalid-attribute error mila"** — Expected
   hai agar attribute-ID predefined 18-category-list mein nahi hai.
2. **"Sirf primary-attribute record hua, secondary nahi"** — Expected agar secondary-
   attribute-value empty tha — empty-values silently skip hoti hain.

---

## 6. Quick Summary

- Verification Process = structured attribute-verification-event-recording,
  primary+secondary cross-linking ke saath.
- 18 predefined attribute-categories (mobile/email/GST/PAN/CIN/TAN/etc.).
- WEBERP/SOA-WAPI only, trustPg-backed.
- Trust Verification KT se related lekin distinct — dono alag write-paths hain.

---

## See also

- [`Verification_Process_Technical_Doc.md`](./Verification_Process_Technical_Doc.md) — code-level detail
- [`../Trust Verification KT/Trust_Verification_Business_Doc.md`](../Trust%20Verification%20KT/Trust_Verification_Business_Doc.md) —
  related but separate single-attribute verification concept

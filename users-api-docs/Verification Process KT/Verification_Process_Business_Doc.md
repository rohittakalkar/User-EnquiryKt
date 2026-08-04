# Verification Process (Attribute-Verification-Event Recording) — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Verification_Process_Technical_Doc.md`](./Verification_Process_Technical_Doc.md) dekho.

**Note — Trust Verification se relationship**: Yeh feature Trust Verification KT se
**related but distinct** hai. Trust Verification KT ek broader single-attribute-status
verification-system cover karta hai (`UserVerificationController` — supplier ka mobile,
email, etc. verify/unverify karna). Yeh KT specifically ek **structured, primary+secondary
cross-linked verification-event-recording system** cover karta hai — jo do tarike se trigger
hota hai:
1. **Directly**, internal-verification-teams (WEBERP, SOA-WAPI) se, aur
2. **Indirectly**, jab GST (ya bulk) verification khud verify ho jaati hai, tab GST-verification
   ka apna background-system automatically is feature ko bhi call karta hai taaki trust-record
   bhi ban jaaye.

Dono systems same broad "trust/verification" goal serve karte hain lekin **alag technical
paths se, alag attribute-ID numbering ke saath**. Dekho
[`../Trust Verification KT/Trust_Verification_Business_Doc.md`](../Trust%20Verification%20KT/Trust_Verification_Business_Doc.md)
for the main verification system.

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

**Real-world example jo iss system ko trigger karta hai**: jab supplier ka GST number
background mein automatically verify ho jaata hai (GST feature ka apna daily/real-time
process), toh us verification ka ek copy/trust-record yahan bhi automatically ban jaata hai
— supplier ko iske liye kuch alag se nahi karna padta, yeh backend-to-backend handshake hai.

**Business impact**: Yeh IndiaMART ke trust-ecosystem ka structured-data-foundation hai
— TrustSeal-eligibility, fraud-detection, aur supplier-credibility-scoring sab is
verification-history pe depend karte hain.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Internal-verification-processes (WEBERP, SOA-WAPI)** | Verification-events directly record karte hain |
| **GST verification ka background-system (indirect trigger)** | Jab GST verify hoti hai, apne aap iss feature ko bhi call karta hai (background messaging ke through) — supplier/admin ko dikhta nahi, purely internal |
| **Supplier (indirectly)** | Jiske attributes verify ho rahe hain |
| **Downstream trust/TrustSeal-systems** | Yeh verification-history read/consume karte hain (usi read-API se jo Trust Verification KT bhi use karta hai) |

---

## 3. Business Flow (step-by-step, no code)

### Flow A — Direct submission (internal verification team)

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

### Flow B — Indirect trigger via GST verification (background, no manual step)

```
1. Kisi supplier ka GST number background-process se verify hota hai (GST feature ka apna
   flow — dekho GST KT)
2. Us GST-verification-system ka apna internal-messaging-system automatically ek naya
   "trust verification event" banane ka signal bhejta hai iss feature ko
3. Iss feature ka backend automatically ek trust-record bana deta hai, bina kisi manual
   admin/supplier action ke
4. Agar iss background-call mein value missing thi (jaise mobile/email), system supplier
   ka current mobile/email record se khud utha leta hai
```

**Business impact**: Yeh ek **seamless backend-integration** hai — supplier ko do baar
alag-alag jagah apna GST/mobile/email verify nahi karana padta; ek verification doosri
system ko automatically feed hoti hai. Agar yeh chain kabhi break ho (jaise background
messaging fail ho jaaye), trust-records GST-verification se out-of-sync ho sakte hain —
dekho Edge Cases.

---

## 4. Business Rules — Plain Language Mein

1. **Sirf predefined 18 attribute-categories supported hain** — koi random attribute-ID
   accept nahi hota; naya attribute-type add karne ke liye code-deploy chahiye hota hai,
   yeh ek admin-configurable list nahi hai.
2. **Primary-attribute-value mandatory hai** — agar primary-value khali chhodi jaaye, system
   silently kuch bhi record nahi karta (na primary, na secondary), lekin request phir bhi
   "SUCCESS" bol deta hai — yeh ek subtle gotcha hai jo confuse kar sakta hai ki record bana
   ya nahi (dekho Edge Cases, point 3).
3. **Har secondary-attribute apna alag cross-linked-record banata hai** — batch mein
   multiple-secondary-attributes ek hi request mein bheje ja sakte hain, lekin agar ek bhi
   secondary-attribute invalid-category ka hai, poora request reject ho jaata hai.
4. **Agar secondary-attributes diye gaye hain, primary ka apna alag/standalone record NAHI
   banta** — primary sirf har secondary ke saath jointly record hota hai.
5. **Sirf do internal-systems allowed hain direct submission ke liye** — WEBERP, SOA-WAPI.
   Iske alawa, GST-verification ka background-system bhi ek "trusted internal caller" ki
   tarah is API ko automatically call karta hai (indirect trigger, Flow B).
6. **Khali secondary-attribute-value automatically skip ho jaati hai** — error nahi aata,
   bas woh specific secondary-record nahi banta.

---

## 5. Notifications — Kaun batata hai kise

| Kab | Kya hota hai |
|---|---|
| Verification-event record ban jaaye | **Koi supplier-facing notification nahi** — yeh purely backend/internal record-keeping hai, supplier ko koi email/SMS nahi jaata |
| GST-verification se automatic trigger ho | Same — koi extra supplier-notification nahi (GST verification ka apna notification alag se jaata hai, dekho GST Business Doc) |

---

## 6. Edge Cases — Product/Business Perspective se

1. **"Maine verification-record submit kiya, invalid-attribute error mila"** — Expected
   hai agar attribute-ID predefined 18-category-list mein nahi hai.
2. **"Sirf primary-attribute record hua, secondary nahi"** — Expected agar secondary-
   attribute-value empty tha — empty-values silently skip hoti hain.
3. **"Request 'SUCCESS' bola lekin koi record hi nahi bana"** — Yeh tab hota hai jab
   primary-attribute-value khud khali submit ho jaaye — system kuch bhi save nahi karta,
   lekin error bhi nahi dikhata. Agar koi "meri verification-event kahan gayi" poochhe, pehle
   confirm karo ki primary-value actually bheji gayi thi ya nahi.
4. **"GST verify hui, lekin trust-record turant nahi bana"** — Yeh background-messaging pe
   depend karta hai; agar us internal-messaging-chain mein delay/failure ho, trust-record ban
   ne mein lag sakta hai ya fail ho sakta hai. Isko GST-verification team ke saath cross-check
   karna padega agar consistently ho raha ho.
5. **"Do alag verification-systems hain, konsa authoritative hai?"** — Trust Verification
   (main system, `UserVerificationController`) aur yeh feature (structured event-recording)
   dono alag GST/attribute-ID numbering use karte hain internally — business-team ke liye yeh
   ek implementation-detail hai, dono hi "verified" ka hi ek roop record karte hain, lekin
   agar deep-dive support-investigation chahiye ho, dono docs cross-check karne padenge.

---

## 7. Quick Summary

- Verification Process = structured attribute-verification-event-recording,
  primary+secondary cross-linking ke saath.
- 18 predefined attribute-categories (mobile/email/GST/PAN/CIN/TAN/etc.).
- Do tarike se trigger hota hai: directly (WEBERP/SOA-WAPI) aur indirectly (GST-verification
  background-chain se automatically).
- Purely backend record-keeping hai — koi supplier-facing notification nahi.
- Trust Verification KT se related lekin distinct hai — dono alag write-paths hain jo same
  broad trust-goal serve karte hain, alag attribute-numbering ke saath.

---

## See also

- [`Verification_Process_Technical_Doc.md`](./Verification_Process_Technical_Doc.md) — code-level detail
- [`../Trust Verification KT/Trust_Verification_Business_Doc.md`](../Trust%20Verification%20KT/Trust_Verification_Business_Doc.md) —
  main single-attribute verification system (mobile/email verify-unverify)
- [`../GST KT/GST_Business_Doc.md`](../GST%20KT/GST_Business_Doc.md) — GST verification, jiska
  background-process is feature ko indirectly trigger karta hai

# Trust Details & Verification — Business Doc

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Trust_Verification_Technical_Doc.md`](./Trust_Verification_Technical_Doc.md) dekho.

**Scope note**: yeh doc **generic, per-attribute verification system** cover karta hai —
GST, PAN, bank, mobile, email jaisi individual cheezein kaise "verified" mark hoti hain. Yeh
[`../TrustSeal KT/`](../TrustSeal%20KT/TrustSeal_Business_Doc.md) se **alag** hai — TrustSeal
ek premium badge hai, yeh system uske neeche ka generic "kya-kya verify hua hai" tracking
mechanism hai.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Har supplier ke bahut saare "attributes" hote hain jo verify karne layak hain — unka mobile
number, email, GST, PAN, bank account, waghera. Yeh system ek **generic, reusable mechanism**
hai kisi bhi attribute ko "verified" mark karne, unverify karne, aur uska status wapas dekhne
ke liye — bina har attribute-type ke liye alag se ek pura naya system banaye.

**Business impact**: Buyer ko pata chalta hai kaunse claims verified hain — GST number sach
mein valid hai, bank account sach mein supplier ka hai, waghera. Yeh trust ka foundational
layer hai jispe TrustSeal jaisi cheezein potentially build hoti hain.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Internal system/consumer** | Kisi attribute ko verify karta hai (jaise GST verify hone ke baad) |
| **Call-center agent** | Manually kisi attribute ko phone-call ke through verify kar sakta hai |
| **Supplier** | Verification ka beneficiary — unka data "verified" dikhta hai |
| **Internal tools** | Kisi bhi supplier ke saare verified-attributes ek saath dekh sakte hain |

---

## 3. Business Flow (step-by-step, no code)

```
1. Kisi trigger se (jaise GST verify ho jaana, ya call-center agent manually confirm kare)
   ek specific attribute (jaise "GST number," "Bank account") ke liye verification request
   banti hai
2. System is attribute ko ek unique ID ke through track karta hai (har attribute-type ka apna
   khud ka numeric ID hai — jaise GST=2106, Bank=390)
3. Verification record ban jaata hai — kisne verify kiya, kab, kis channel se (phone/online),
   kya comment tha
4. Koi bhi baad mein GET se check kar sakta hai: yeh specific attribute verified hai ya nahi
5. Bulk-verification bhi support hai — ek saath bahut saare attributes verify karne ke liye
```

**Business impact**: Yeh ek **foundational, cross-cutting layer** hai — GST verification,
bank verification, sab isi common mechanism ko reuse karte hain, alag-alag banane ke bajaye.

---

## 4. Business Rules — Plain Language Mein

1. **Har attribute-type ka apna numeric ID hai** — yeh ek shared reference-system hai (GST,
   bank account, mobile, waghera sab alag IDs).
2. **Verification "kisne kiya" bhi track hota hai** — internal system (WAPI/automated) vs
   ek specific call-center agent, dono cases handle hote hain.
3. **Bulk-verification alag path hai** — ek supplier ke multiple attributes ek saath verify
   karne ke liye ek dedicated mechanism hai, single-attribute verification se alag.
4. **Internal systems isi mechanism ko "loopback" karke bhi use karte hain** — matlab
   background processes (jaise GST-verification consumer) khud iss mechanism ko call karte
   hain jab unhe kuch verify karna ho, seedha apna khud ka verification-table nahi banate.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Mera bank account verify ho gaya tha, ab unverified dikha raha hai"** — Verification
   status change ho sakta hai agar underlying data change ho (jaise naya bank account jud़
   gaya) — purana verification naye data pe automatically apply nahi hota.
2. **"GST verify hone ke baad TrustSeal kyun nahi mila"** — Yeh do alag systems hain (dekho
   TrustSeal KT doc) — verification aur TrustSeal-assignment automatically linked nahi hain
   code mein.

---

## 6. Quick Summary

- Yeh ek generic, reusable "kaunsa attribute verified hai" tracking system hai.
- GST, bank, mobile, email — sab isi common mechanism se guzarte hain.
- Manual (call-center) aur automated (system-triggered) dono verification-paths support
  hote hain.
- TrustSeal se alag hai, lekin conceptually related — verification "neeche ki foundation"
  hai, TrustSeal "upar ka badge."

---

## See also

- [`Trust_Verification_Technical_Doc.md`](./Trust_Verification_Technical_Doc.md) —
  code-level detail
- [`../TrustSeal KT/TrustSeal_Business_Doc.md`](../TrustSeal%20KT/TrustSeal_Business_Doc.md) —
  related but separate: premium trust-badge system
- [`../GST KT/GST_Business_Doc.md`](../GST%20KT/GST_Business_Doc.md) — GST verification is
  isi generic mechanism ka ek concrete example hai

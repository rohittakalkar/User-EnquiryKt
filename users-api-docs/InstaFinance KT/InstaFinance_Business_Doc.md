# InstaFinance — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`InstaFinance_Technical_Doc.md`](./InstaFinance_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

**InstaFinance** ek toggle-feature hai — kisi registered-company (unke CIN — Corporate
Identification Number — se identify hoti hai) ke liye, "financial-services enabled hai
ya nahi" yeh flag on/off kiya ja sakta hai. Yeh IndiaMART ke financial-services/lending-
partner-integration (jaise instant-business-loans) ka ek eligibility-toggle hai.

**Business impact**: Yeh decide karta hai ki kis company ko financial-services-offering
(jaise InstaFinance-branded loan/credit-product) dikhayi jaaye. Ek simple on/off switch
hai, company-level (CIN se), na ki individual-supplier-account-level.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Internal-tool (sirf "MY" allowed)** | Flag ko enable/disable karta hai |
| **Company (via CIN)** | Jiske liye flag apply hota hai |
| **Financial-services/lending-integration** | Downstream is flag ko consume karta hai |

---

## 3. Business Flow (step-by-step, no code)

```
1. Internal-tool ek company (uske CIN se identify) ke liye InstaFinance-flag update
   karta hai — enable (0) ya disable (-1)
2. System check karta hai ki yeh CIN system mein already exist karta hai ya nahi
   - Agar exist karta hai, flag update ho jaata hai
   - Agar nahi, "no record found" error milta hai (naya CIN yahan se create nahi ho
     sakta)
3. Successful-update par, ek downstream-event bhi fire hoti hai
```

**Business impact**: Yeh sirf **existing** company-financial-records ko toggle karta
hai — naya financial-master-record yahan se create nahi hota, sirf existing ko update
kiya ja sakta hai.

---

## 4. Business Rules — Plain Language Mein

1. **Flag sirf do values le sakta hai**: `0` (enabled) ya `-1` (disabled) — koi aur
   value reject ho jaata hai.
2. **CIN ki length exactly 21 characters honi chahiye** — India ke standard CIN-format
   ke mutabik.
3. **Sirf ek hi internal-app allowed hai ("MY")** — sabse restrictive allowlist is
   poore KT-series mein.
4. **Sirf existing-company-record update ho sakta hai** — agar CIN system mein nahi
   hai, request fail ho jaata hai.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine CIN diya, 'no record found' aaya"** — Expected hai agar us CIN ka koi
   financial-master-record pehle se system mein nahi hai — is endpoint se naya record
   nahi banta.
2. **"Flag value reject ho gaya"** — Expected hai agar `0`/`-1` ke alawa kuch bheja
   gaya.

---

## 6. Quick Summary

- InstaFinance = company-level (CIN-based) financial-services-eligibility toggle
  (enable/disable).
- Sirf existing-record-update, koi naya record yahan se create nahi hota.
- Sirf ek internal-app ("MY") allowed.

---

## See also

- [`InstaFinance_Technical_Doc.md`](./InstaFinance_Technical_Doc.md) — code-level detail

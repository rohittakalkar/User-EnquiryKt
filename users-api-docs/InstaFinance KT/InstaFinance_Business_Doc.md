# InstaFinance — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai — koi code nahi, sirf "kya hota hai, kyun hota
hai, aur company ke liye iska matlab kya hai." Code ke liye
[`InstaFinance_Technical_Doc.md`](./InstaFinance_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

**InstaFinance** ek toggle-feature hai — kisi registered-company (unke CIN — Corporate
Identification Number — se identify hoti hai) ke liye, "financial-services enabled hai ya nahi"
yeh flag on/off kiya ja sakta hai. Yeh IndiaMART ke financial-services/lending-partner-integration
(jaise instant-business-loans, credit-line offers) ka ek eligibility-toggle hai.

**Business impact**: Yeh decide karta hai ki kis company ko financial-services-offering (jaise
InstaFinance-branded loan/credit-product) dikhayi jaaye. Ek simple on/off switch hai,
**company-level (CIN se)**, na ki individual-supplier-account-level — matlab agar ek CIN ke under
multiple supplier-accounts/GLIDs hain, flag sabke liye ek saath apply hota hai.

**Kaun consume karta hai is flag ko**: Yeh doc ke saath review kiye gaye teeno repos
(`users-api-go-production`, `service-api-go-production`, `user-temp-consumers-production`) mein
is flag ko **padhne** wala koi code nahi mila — sirf likhne/toggle-karne wala. Iska matlab
downstream financial-services/lending-integration in repos ke bahar kahin aur hai, jo is flag
ko apne system mein consume karta hoga. **[INFERRED — confirm with financial-services/lending
team konsa downstream system yeh flag padhta hai]**.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Internal-tool (sirf "MY" caller-identity allowed)** | Flag ko enable/disable karta hai — sabse restrictive allowlist iss poore KT-series mein, sirf ek hi internal-app allowed hai |
| **Company (via CIN)** | Jiske liye flag apply hota hai — CIN se identify hoti hai, na ki kisi individual supplier-account se |
| **Financial-services/lending-integration (downstream, is codebase ke bahar)** | Is flag ko consume karta hai, decide karne ke liye ki company ko loan/credit-offer dikhana hai ya nahi |
| **Background sync-mechanism (queue-driven)** | Har successful update ke baad, ek async retry/sync-step bhi chalta hai taaki update reliably apply ho (technical doc section 5, 9 dekho) |

---

## 3. Business Flow (step-by-step, no code)

### Flow A — Internal tool flag ko enable/disable karta hai

```
1. Internal-tool ek company (uske CIN se identify) ke liye InstaFinance-flag update
   karta hai — enable (0) ya disable (-1)
2. System check karta hai ki yeh CIN system mein already exist karta hai ya nahi
   - Agar exist karta hai, flag update ho jaata hai
   - Agar nahi, "no record found" error milta hai (naya CIN yahan se create nahi ho
     sakta)
3. Successful-update par, ek background-sync bhi automatically trigger hota hai — taaki
   agar update kahin reliably apply na ho paaye, ek retry-mechanism usse dobara try kare
```

**Business impact**: Yeh sirf **existing** company-financial-records ko toggle karta hai — naya
financial-master-record yahan se create nahi hota, sirf existing ko update kiya ja sakta hai. Aur
har change ke saath, ek reliability-net bhi hai jo yeh ensure karne ki koshish karta hai ki
update kahin silently miss na ho jaaye.

### Flow B — CIN system mein exist nahi karta

```
1. Internal-tool ek CIN ke liye flag update karne ki koshish karta hai
2. System check karta hai — is CIN ka koi financial-master-record hai kya
3. Nahi hai -> "no record found" error, koi update nahi hota, koi background-sync bhi
   trigger nahi hota
```

**Business impact**: Yeh ek intentional guard-rail hai — koi bhi naya/random CIN is toggle-endpoint
se force-enable nahi ho sakta. Financial-master-record kahin aur (kisi doosre onboarding process
se) pehle banna zaroori hai, tabhi yeh flag set ho sakta hai.

---

## 4. Business Rules — Plain Language Mein

1. **Flag sirf do values le sakta hai**: `0` (enabled) ya `-1` (disabled) — koi aur value reject
   ho jaata hai (galat "length" error message ke saath, jo thoda confusing hai — actually yeh
   ek allowed-values check hai, sach mein length ka issue nahi).
2. **CIN ki length exactly 21 characters honi chahiye** — India ke standard CIN-format ke
   mutabik. Yeh do jagah check hota hai (ek generic validation-map se, ek explicit check se) —
   double-safety.
3. **Sirf ek hi internal-app allowed hai ("MY")** — sabse restrictive allowlist is poore
   KT-series mein.
4. **Sirf existing-company-record update ho sakta hai** — agar CIN system mein nahi hai, request
   fail ho jaata hai, naya record yahan se create nahi hota.
5. **Company-identity-based, supplier-identity-based nahi** — flag CIN se match hota hai, kisi
   individual supplier-login/GLID se nahi. Agar ek CIN ke multiple business-accounts hain, sab
   ek saath affect hote hain.
6. **Har successful update ek background reliability-step bhi trigger karta hai** — matlab agar
   pehla update kisi wajah se poori tarah reflect na ho paaye kisi doosre system mein, ek
   automatic retry-mechanism hai jo usse dobara apply karne ki koshish karega.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine CIN diya, 'no record found' aaya"** — Expected hai agar us CIN ka koi
   financial-master-record pehle se system mein nahi hai — is endpoint se naya record nahi banta.
2. **"Flag value reject ho gaya"** — Expected hai agar `0`/`-1` ke alawa kuch bheja gaya.
3. **"Maine update kiya, lekin downstream system mein change kuch der baad reflect hua"** — Yeh
   expected ho sakta hai kyunki ek part of the sync background-mein (queue ke through) hota hai,
   turant nahi. Agar bahut zyada der ho ya bilkul reflect na ho, iska matlab ho sakta hai
   background-sync fail ho raha hai — technical team ko escalate karo.
4. **"Meri company ka ek hi CIN hai lekin multiple accounts hain, sabka flag change ho gaya"** —
   Yeh expected hai, bug nahi — flag CIN-level hai, account-level nahi.

---

## 6. Quick Summary

- InstaFinance = company-level (CIN-based) financial-services-eligibility toggle
  (enable/disable).
- Sirf existing-record-update, koi naya record yahan se create nahi hota.
- Sirf ek internal-app ("MY") allowed — sabse restrictive access poore KT-series mein.
- Har update ke saath ek background reliability/sync-step bhi chalta hai.
- Downstream financial-services-integration jo actually is flag ko use karta hai, woh iss
  documented codebase ke bahar hai — team se confirm karo exact consumer.

---

## See also

- [`InstaFinance_Technical_Doc.md`](./InstaFinance_Technical_Doc.md) — code-level detail (APIs,
  DB table, RabbitMQ queue/consumer, optimization notes)

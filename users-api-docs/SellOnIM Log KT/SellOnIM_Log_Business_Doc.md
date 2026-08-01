# SellOnIM Log — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`SellOnIM_Log_Technical_Doc.md`](./SellOnIM_Log_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

**SellOnIM** naye supplier ke onboarding-journey ko track karta hai — jab koi naya
supplier IndiaMART pe "sell" karna start karta hai, unka signup/profile-completion-journey
step-by-step record hota hai: naam diya ya nahi, company-name di ya nahi, email di ya
nahi, kitne products add kiye, address di ya nahi, GST di ya nahi, aur unhe kab call
karna best rahega (`BEST_TIME_TO_CALL`).

**Business impact**: Yeh internal-sales/calling-team ko bataata hai ki kaunse suppliers
apna onboarding-journey adhoora chhod chuke hain, aur unhe follow-up call ke liye kab
best time hai. Kuch specific supplier-types (`CUSTTYPE_ID=39`) ke liye, yeh journey-data
automatically ek downstream "calling-eligibility" evaluation ko bhi trigger karta hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Naya supplier** | Apna onboarding-journey complete karta hai (indirectly, via app/web) |
| **App/tools (GLADMIN, MAPI, M.INDIAMART.COM, SELLERMY, IMOB, Android, iOS)** | Journey-progress record karte hain |
| **Internal calling/verification-team** | Downstream is data ko use karti hai supplier ko call karne ke liye |

---

## 3. Business Flow (step-by-step, no code)

```
1. Naya supplier apna onboarding-journey shuru karta hai (naam, company, email, products,
   address, GST — step-by-step)
2. Har step ka progress ek journey-log record mein capture hota hai — pehli baar naya
   record banta hai, baad mein wahi record update hota hai
3. Agar supplier ek specific customer-type (39) ka hai, journey-update hone par ek
   downstream-notification bhi fire hoti hai
4. Downstream-consumer ek 5-step eligibility-check chain chalata hai (verification-log,
   FCP-priority, duplicate-email, duplicate-GST, duplicate-supplier-ID) — agar sab pass
   hon, supplier ko "calling-queue" mein daal diya jaata hai (best-time-to-call ke saath)
5. Agar koi ek bhi check fail ho, supplier calling-queue mein nahi jaata — silently skip
   ho jaata hai
```

**Business impact**: Yeh ek automated lead-qualification-pipeline hai — sirf woh
suppliers calling-queue tak pahunchte hain jo genuinely "good-fit" hain (verified,
high-priority, non-duplicate).

---

## 4. Business Rules — Plain Language Mein

1. **Journey-log insert-or-update hota hai** — agar `LOG_ID` nahi diya gaya, naya record
   banta hai; agar diya gaya, existing record update hota hai.
2. **Calling-eligibility-trigger sirf `CUSTTYPE_ID=39` ke liye fire hota hai** — baaki
   customer-types ke liye sirf journey-log update hota hai, koi downstream-action nahi.
3. **Calling-eligibility 5 sequential checks pe depend karta hai** — sabhi pass hone
   chahiye: (a) verification-log mein koi disqualifying-flag nahi, (b) FCP-priority-range
   0.8 ya 0.9, (c) koi duplicate-email-associated-account nahi, (d) koi duplicate-GST
   nahi, (e) last 30 din mein koi duplicate-verification-allocation nahi.
4. **Sab pass hone par, ek naya "verification-queue" entry banta hai** — best-time-to-call
   ke saath, agar diya gaya ho.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine journey complete ki, lekin call nahi aayi"** — Expected ho sakta hai agar
   koi eligibility-check fail hui (duplicate-email/GST, ya priority-range match nahi
   hua) — sirf journey-tracking se calling guaranteed nahi hoti.
2. **"Meri journey-update fail ho gayi"** — Ho sakta hai `LOG_ID` galat ho ya record
   already kisi aur `GLID` ke against ho — update sirf matching (`LOG_ID`, `GLID`) pair
   pe hoti hai.

---

## 6. Quick Summary

- SellOnIM Log = naye-supplier onboarding-journey-tracker (progress steps + best-time-to-call).
- Insert-or-update, journey ka har update record hota hai.
- CUSTTYPE_ID=39 ke liye, ek automated 5-check calling-eligibility-pipeline bhi trigger
  hoti hai.
- Eligible suppliers ek separate "verification/calling-queue" table mein pahunchte hain.

---

## See also

- [`SellOnIM_Log_Technical_Doc.md`](./SellOnIM_Log_Technical_Doc.md) — code-level detail

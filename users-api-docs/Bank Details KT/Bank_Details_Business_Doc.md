# Bank Details — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Bank_Details_Technical_Doc.md`](./Bank_Details_Technical_Doc.md) dekho.

**Note**: Yeh feature GST/Fact-Sheet-jaisa hi ek **shared multi-purpose "user-details"
write endpoint** ka hissa hai (`TYPE=BankDetails`) — dekho
[`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) same-controller-pattern
ke liye. Bank Details is series ki sabse deep-integrated feature hai — verification,
trust, aur fraud-detection teeno se juda hua.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Supplier apne **bank-account-details** (account-number, IFSC-code) IndiaMART profile
ke saath save kar sakta hai — jaise payment-receiving ke liye. Ek supplier ke **multiple
bank-accounts** ho sakte hain, lekin unmein se **ek hi "prime" (primary)** account
designate hota hai. Bank-account-details ek **verification-pipeline** se bhi guzarte
hain, aur **fraud/banned-detection** ka bhi hissa hain.

**Business impact**: Bank-details supplier-trust ka ek core-signal hai — verified
bank-account, IndiaMART ke lending/finance-integrations (jaise InstaFinance) aur
trust-badge-eligibility ke liye zaroori hai. Fraud-prevention ke liye bhi critical hai —
same-bank-account multiple-fraudulent-accounts ke against use ho raha ho toh detect
hota hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Apne bank-account-details add/update/remove karta hai |
| **Internal-tools (MERP, MAPI, GLADMIN, BUYERMY, SELLERMY)** | Bank-details manage kar sakte hain, alag "request-type" ke through |
| **System/automated-processes** | Bank-account auto-verify/auto-prime-assign kar sakte hain |
| **Downstream verification/trust/fraud-systems** | Bank-details ko validate/cross-check karte hain |

---

## 3. Business Flow (step-by-step, no code)

```
1. Supplier apna bank-account-number aur IFSC submit karta hai
2. System decide karta hai — kya yeh naya "prime" (primary) account banega?
   - Agar supplier khud (user-driven) add kar raha hai, naya account by-default prime
     ban jaata hai
   - Agar koi automated-system (jaise verification-pipeline) add kar raha hai, prime-
     status ek check ke baad decide hota hai (agar koi already-verified-prime-account
     nahi hai, tabhi naya prime banega)
3. Agar supplier "prime" flag explicitly kisi doosre account pe set karta hai, purana
   prime-account automatically un-prime ho jaata hai (sirf ek prime allowed)
4. Naya bank-account-record downstream systems ko async-notify hota hai:
   - Search/directory-sync
   - Trust-related-sync
   - Verification-pipeline
   - Fraud/banned-detection (agar suspicious-pattern match ho)
5. Agar supplier apna bank-account remove karta hai, yeh "soft-delete" hoti hai
   (record disable hota hai, delete nahi)
```

**Business impact**: Yeh ek high-stakes-data hai — verification, trust, fraud-detection,
aur financial-integrations sabka foundation. Isliye multiple downstream-systems isse
sync/validate karte hain.

---

## 4. Business Rules — Plain Language Mein

1. **Ek supplier ka sirf ek "prime" bank-account ho sakta hai** — naya prime set karne
   se purana automatically un-prime ho jaata hai.
2. **Kaun add kar raha hai (supplier khud vs automated-system) prime-assignment-logic
   ko change karta hai.**
3. **Delete "soft" hai** — record `ENABLED=-1` ho jaata hai, physically delete nahi hota.
4. **Bank-details 4 alag downstream-systems ko notify karte hain** — general-sync,
   trust-related-sync, verification, aur banned/fraud-detection.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine naya account add kiya, purana prime nahi raha"** — Expected hai agar naya
   account explicitly prime mark kiya gaya — sirf ek prime-account allowed hai.
2. **"Mera bank-account delete ho gaya lekin history mein dikh raha hai"** — Expected
   hai, delete soft hai, record disabled-state mein rehta hai.
3. **"Mera account 'banned' ho gaya"** — Ho sakta hai fraud-detection-pipeline ne kuch
   suspicious-pattern detect kiya ho (jaise same bank-account multiple fraudulent
   accounts ke against).

---

## 6. Quick Summary

- Bank Details = supplier ke bank-account-records, ek-prime-account-constraint ke saath.
- GST/Fact-Sheet jaisa hi shared "user-details" endpoint ka hissa.
- 4-way downstream integration: general-sync, trust-sync, verification, fraud/banned-
  detection.
- Soft-delete (disable), no hard-delete.

---

## See also

- [`Bank_Details_Technical_Doc.md`](./Bank_Details_Technical_Doc.md) — code-level detail
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — same shared-controller pattern
- [`../Trust Verification KT/Trust_Verification_Business_Doc.md`](../Trust%20Verification%20KT/Trust_Verification_Business_Doc.md) —
  related verification-concept

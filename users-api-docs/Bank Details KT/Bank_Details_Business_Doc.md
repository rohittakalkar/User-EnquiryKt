# Bank Details — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Bank_Details_Technical_Doc.md`](./Bank_Details_Technical_Doc.md) dekho.

**Note**: Yeh feature GST/Fact-Sheet-jaisa hi ek **shared multi-purpose "user-details" write
endpoint** ka hissa hai (`TYPE=BankDetails`) — dekho
[`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) same-controller-pattern
ke liye. Bank Details is series ki sabse deep-integrated feature hai — verification, trust,
aur fraud-detection teeno se juda hua, plus ek external verification-partner (PennyDrop) bhi
involved hai.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Supplier apne **bank-account-details** (account-number, IFSC-code, account-holder-name)
IndiaMART profile ke saath save kar sakta hai — jaise payment-receiving ke liye. Ek supplier
ke **multiple bank-accounts** ho sakte hain, lekin unmein se **ek hi "prime" (primary)**
account designate hota hai — yehi wo account hai jo payments/lending-integrations ke liye
default use hota hai.

Bank-account-details sirf ek data-field nahi hain — yeh ek **verification-pipeline** se
guzarte hain (external verification-partner "PennyDrop" ke through), ek **fraud/banned-
detection** check se guzarte hain (koi galat/abusive text toh nahi daala gaya bank-name/
address fields mein), aur Trust-domain database mein bhi sync hote hain.

**Business impact**: Bank-details supplier-trust ka ek core-signal hai — verified
bank-account, IndiaMART ke lending/finance-integrations (jaise InstaFinance) aur
trust-badge-eligibility ke liye zaroori hai. Fraud-prevention ke liye bhi critical hai —
agar koi supplier apne bank-details mein abusive/spammy text daale, system usse detect
karke automatically saaf kar deta hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Apne bank-account-details add/update/remove karta hai seller panel/app se |
| **Internal-tools (MERP, MAPI, GLADMIN, BUYERMY, SELLERMY)** | Bank-details manage kar sakte hain, alag "request-type" ke through |
| **Automated/system-processes (ERP, WEBERP, PayWim, Android-web)** | Bank-account auto-create/auto-prime-assign kar sakte hain, apni khud ki logic ke saath |
| **PennyDrop (external verification-partner)** | Bank-account-number aur IFSC ki actual banking-validity verify karta hai |
| **Fraud/banned-content-detection system** | Bank-name, address, aur account-holder-name fields mein abusive/spam content check karta hai |
| **Trust-domain team/systems** | Bank-details ka apna copy trust-database mein rakhte hain, TrustSeal-jaisi eligibility ke liye |

---

## 3. Business Flows (step-by-step, no code)

### Flow A — Supplier khud apna naya bank-account add karta hai

```
1. Supplier seller panel/app pe jaake account-number aur IFSC daalta hai
2. Naya account by-default "prime" ban jaata hai — kyunki supplier khud (user-driven) add
   kar raha hai
3. Naya bank-account-record downstream systems ko async-notify hota hai:
   - Search/directory-sync (aur supplier ko confirmation email/SMS)
   - Verification (PennyDrop ko forward hota hai bank-account validate karne ke liye)
   - Trust-related-sync
   - Fraud/banned-content-detection
4. Supplier ko email/SMS confirmation milta hai
```

**Business impact**: Supplier ka payment-receiving-account set ho jaata hai, aur
verification-pipeline background mein confirm karta hai ki yeh account genuinely exist karta
hai.

### Flow B — Automated system (ERP/WEBERP jaisa) bank-account add karta hai

```
1. Koi automated-system (jaise ERP integration) supplier ka bank-account add karta hai
2. Prime-status turant decide nahi hota jaise Flow A mein — system pehle check karta hai:
   "kya is supplier ka pehle se koi verified-prime-account hai?"
3. Agar nahi hai, toh naya account prime ban jaata hai; agar hai, toh nahi banta
4. Same downstream-notify (search-sync, verification, trust-sync, fraud-check) hota hai,
   lekin supplier ko mail/SMS NAHI jaata (kyunki yeh system-driven hai, supplier-driven nahi)
```

**Business impact**: Yeh un cases ke liye hai jahan bank-details kisi doosre business-system
(jaise ERP) se automatically aate hain — is flow mein prime-assignment zyada conservative hai
taaki koi already-verified account accidentally replace na ho.

### Flow C — Supplier apna existing bank-account update karta hai (ya kisi doosre ko "prime" banata hai)

```
1. Supplier apne existing account ki details change karta hai, ya explicitly kisi doosre
   account ko "prime" mark karta hai
2. Agar naya account explicitly "prime" set hota hai, purana prime-account automatically
   un-prime ho jaata hai (sirf ek prime allowed, hamesha)
3. Agar sirf minor/no-material change hua hai (account-number/IFSC/prime-status same rahe),
   supplier ko mail/SMS nahi jaata — sirf tab jaata hai jab kuch actually badla ho
4. Agar naya account "prime" hai lekin account-holder-name missing hai, notification-mail
   fail ho jaata hai (safety-check)
```

**Business impact**: "Sirf ek prime" ka invariant hamesha maintain hota hai, aur supplier
ko sirf tab notify kiya jaata hai jab kuch meaningfully change hua ho — spam-notifications
avoid karne ke liye.

### Flow D — Supplier apna bank-account remove karta hai

```
1. Supplier "delete" trigger karta hai apne bank-account pe
2. Yeh "soft-delete" hoti hai — record disable hota hai, delete nahi
3. Notification-consumer is delete-event pe koi mail/SMS nahi bhejta (silent operation)
4. IMPORTANT: agar delete kiya gaya account "prime" tha, uska prime-status disable hone ke
   baad bhi as-is rehta hai (system explicitly usse un-prime nahi karta)
```

**Business impact**: Delete ek reversible-jaisa operation hai (data preserve hota hai), lekin
iska ek subtle-gotcha hai — ek disabled account technically "prime" flagged reh sakta hai jab
tak koi naya account explicitly prime na banaya jaaye. Support/product team ko yeh pata hona
chahiye agar koi confusion aaye "prime-account dikh raha hai jo maine delete kiya tha."

### Flow E — Fraud/banned-content-check aur auto-correction

```
1. Har bank-account-change (naya ya updated) automatically ek banned-content-check se
   guzarta hai — bank-name, address, account-holder-name jaise text-fields check hote hain
2. Agar koi abusive/spam/inappropriate text mila, system usse automatically blank kar deta
   hai — koi manual-intervention ki zaroorat nahi
3. Agar kuch flagged nahi hua, process normally aage badhta hai
```

**Business impact**: Yeh ek automatic-hygiene-layer hai — supplier khud galti se ya jaan-
boojh kar abusive text na daal sake apne bank-details ke free-text fields mein, is baat ko
system khud handle kar leta hai bina kisi insaan ko involve kiye.

---

## 4. Prime-Account Assignment — Kaun Kab Prime Banta Hai

| Kaun add kar raha hai | Naya account prime banta hai kya? |
|---|---|
| **Supplier khud (seller panel/app)** | Haan, hamesha — user-driven insert automatically prime hai |
| **Automated/ERP-system** | Sirf agar koi already-verified-prime-account exist nahi karta; warna nahi |
| **MERP (internal-tool)** | Kabhi nahi automatically — explicit prime-set alag se karna padta hai |
| **Explicit prime-set kisi bhi tareeke se** | Purana prime-account automatically un-prime ho jaata hai — sirf ek prime allowed hamesha |

**Business rule jo yaad rakhne layak hai**: "Sirf ek prime-account" ek strict invariant hai —
system iske liye ek extra check-query bhi chalata hai jab automated-systems account add karte
hain, taaki galti se koi already-verified account replace na ho jaaye.

---

## 5. Business Rules — Plain Language Mein

1. **Ek supplier ka sirf ek "prime" bank-account ho sakta hai** — naya prime set karne se
   purana automatically un-prime ho jaata hai.
2. **Kaun add kar raha hai (supplier khud vs automated-system vs MERP) prime-assignment-
   logic ko change karta hai** — dekho section 4.
3. **Delete "soft" hai** — record disable ho jaata hai, physically delete nahi hota, aur uski
   history preserve rehti hai.
4. **Delete hone se prime-status reset nahi hota** — agar prime-account delete kiya gaya,
   woh technically "prime" hi reh jaata hai jab tak koi naya account explicitly prime na
   banaya jaaye.
5. **Bank-details 4 alag downstream-systems ko notify karte hain** — general-sync,
   trust-related-sync, verification (PennyDrop), aur banned/fraud-detection.
6. **Notification-mail sirf tab jaati hai jab kuch actually badla ho** — agar update mein
   account-number, IFSC, aur prime-status teeno same rahe, supplier ko email/SMS nahi jaata.
7. **System/ERP-driven changes supplier ko notify nahi karte** — mail/SMS sirf supplier-
   driven changes pe jaata hai, automated-background-changes silent rehte hain.
8. **Agar prime-account ka holder-name missing hai, notification fail ho jaata hai** —
   yeh ek safety-check hai jo incomplete-prime-account ke liye alert-mail bhejne se rokta hai.
9. **Bank-details ke free-text fields (naam, address) automatically banned-content-checked
   hote hain** — koi abusive/inappropriate text mile toh system usse khud blank kar deta hai.
10. **Sirf specific internal-tools/gateways hi bank-details modify kar sakte hain** — MERP,
    MAPI, GLADMIN, BUYERMY, SELLERMY, aur kuch specific ERP/system-integrations; baaki sab
    unauthorized treat hote hain.

---

## 6. Notifications — Supplier ko kab pata chalta hai

| Kab | Kya notification jaata hai |
|---|---|
| Supplier khud naya bank-account add karta hai | Email + SMS confirmation |
| Supplier existing account meaningfully update karta hai (account-number/IFSC/prime badla) | Email + SMS confirmation |
| Sirf minor/no-material change hua (kuch actually badla nahi) | **Notification NAHI jaata** |
| Automated/ERP-system ne account add/update kiya | **Notification NAHI jaata** — background/system-driven changes silent rehte hain |
| Bank-account soft-delete hua | **Notification NAHI jaata** |
| Prime-account ka holder-name missing hai | Notification **fail** ho jaata hai (safety-check, mail nahi bhejta) |
| Banned-content detect hua aur auto-clean hua | Koi supplier-facing notification nahi (internal auto-correction hai) |

---

## 7. Edge Cases — Product/Business Perspective se

1. **"Maine naya account add kiya, purana prime nahi raha"** — Expected hai agar naya
   account explicitly prime mark kiya gaya — sirf ek prime-account allowed hai.
2. **"Mera bank-account delete ho gaya lekin history mein dikh raha hai"** — Expected hai,
   delete soft hai, record disabled-state mein rehta hai.
3. **"Maine apna prime-account delete kiya, lekin ab bhi 'prime' dikh raha hai kahin"** —
   Yeh ek known-gap hai: delete operation prime-flag ko reset nahi karta. Agar koi confusion
   ho, naya account explicitly "prime" mark karke fix kiya ja sakta hai.
4. **"Mujhe update ka confirmation email nahi mila"** — Do possible reasons: (a) kuch
   actually change hi nahi hua (system silently skip karta hai), ya (b) yeh change ek
   automated/system-source se aayi thi, jahan notification design se hi nahi bheja jaata.
5. **"Mera account 'banned' ho gaya ya text change ho gaya"** — Ho sakta hai fraud-detection-
   pipeline ne bank-name/address/holder-name mein kuch inappropriate content detect kiya ho
   aur usse automatically saaf kar diya ho.
6. **Prime-account waala email nahi jaa raha** — Agar account-holder-name blank hai aur woh
   account "prime" hai, notification-system safety ke liye mail nahi bhejta jab tak name fill
   na ho.

---

## 8. Quick Summary

- Bank Details = supplier ke bank-account-records, ek-prime-account-constraint ke saath.
- GST/Fact-Sheet jaisa hi shared "user-details" endpoint ka hissa.
- Prime-assignment kaun add kar raha hai (supplier/system/MERP) pe depend karta hai.
- 4-way downstream integration: general-sync (+mail/SMS), verification (PennyDrop-partner),
  trust-sync, aur fraud/banned-content-detection.
- Soft-delete (disable), no hard-delete — lekin prime-flag delete pe reset nahi hota
  (known-gap).
- Notifications sirf supplier-driven, materially-changed updates pe jaate hain — system-driven
  ya no-op changes silent rehte hain.

---

## See also

- [`Bank_Details_Technical_Doc.md`](./Bank_Details_Technical_Doc.md) — code-level detail
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — same shared-controller pattern
- [`../Trust Verification KT/Trust_Verification_Business_Doc.md`](../Trust%20Verification%20KT/Trust_Verification_Business_Doc.md) —
  related verification-concept

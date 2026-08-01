# User Add / Update (Core Registration & Profile) — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`User_Add_Update_Technical_Doc.md`](./User_Add_Update_Technical_Doc.md) dekho.

**Note**: Yeh IndiaMART users-domain ka **sabse foundational feature** hai — supplier
ka master-record (`GLUSR_USR`) yahi se create/update hota hai. Baaki bahut saare KT
folders (GST, Bank Details, Fact Sheet, Social Contacts, SellOnIM, Negative Mcat, etc.)
isi master-record ke against extra-data attach karte hain — unn sabka foundation yeh
feature hai.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Jab koi naya supplier/buyer IndiaMART pe **sign-up** karta hai, unka core-profile-
record (naam, mobile, email, address, company-details, login-credentials) yahan
create hota hai. Baad mein, jab bhi wo apna profile **update** karta hai (naya address,
naya mobile, company-name-change, etc.), wahi record yahan se update hota hai.

**Business impact**: Yeh IndiaMART ki poori user-identity ka source-of-truth hai. Har
supplier/buyer ka ek unique account yahin se manage hota hai, aur yeh data dusre
50+ dependent-systems (search, trust, alerting, LMS, CSL, enquiry, etc.) mein turant
replicate hota hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Naya Supplier/Buyer** | Signup karta hai — naya account create hota hai |
| **Existing Supplier/Buyer** | Apna profile update karta hai |
| **Bahut saare internal-apps (40+ allowed apps)** | Registration/update-flow ko route kar sakte hain |
| **50+ downstream-systems** | Har add/update ka event consume karke apna copy sync rakhte hain |

---

## 3. Business Flow (step-by-step, no code)

### Naya Signup (Add)
```
1. User apna naam, mobile, email, address, company-details submit karta hai
2. System pehle check karta hai — kya yeh mobile/email/GSM (business-registration-
   number) already kisi existing-account se juda hai?
   - Agar haan, naya account nahi banta — existing-account ko hi "internally update"
     kiya jaata hai (duplicate-account-prevention)
   - Agar nahi, naya account create hota hai
3. Naye account ke liye login-credentials bhi ek alag "auth" system mein create
   hoti hain
4. Account "Approved" status ke saath directly active hota hai
5. Naye-account-ka event 50+ downstream-systems ko async-notify hota hai
```

### Profile Update
```
1. Existing user apna koi bhi profile-field update karta hai (address, mobile, email,
   company-name, listing-status, etc.)
2. System sirf diye-gaye fields update karta hai — partial-update supported
3. Kuch sensitive-fields (jaise mobile/email/address) TrustSeal-verification-status ko
   bhi affect kar sakte hain (re-verification trigger ho sakta hai)
4. Update ka event downstream-systems ko notify hota hai
```

**Business impact**: Duplicate-account-prevention (mobile/email/GSM-based) ek core
data-quality-safeguard hai — same business/person ke multiple fraudulent-accounts
banne se rokta hai.

---

## 4. Business Rules — Plain Language Mein

1. **Duplicate-detection mobile, email, aur GSM (business-registration-number) teeno
   pe hoti hai** — agar koi bhi match milta hai, naya account nahi banta.
2. **Naya account seedha "Approved" status mein banta hai** — koi pending-approval-
   step nahi (immediate-activation).
3. **Do alag-alag databases update hote hain** — main-profile (`GLUSR_USR`) aur
   login-credentials (separate "auth" system) — dono sync rehte hain.
4. **Kuch sensitive-fields update karne se TrustSeal-verification-status reset ho sakta
   hai** — jaise address/mobile/email change karna re-verification trigger kar sakta hai.
5. **Bahut saare internal-apps (40+) yeh operations perform kar sakte hain.**
6. **Update partial ho sakta hai** — sirf diye-gaye fields hi change hote hain.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine naya account banane ki koshish ki, purana account mila"** — Expected hai
   agar mobile/email/GSM se pehle se koi account exist karta hai — system usi ko
   update kar deta hai, naya nahi banata.
2. **"Maine address update kiya, TrustSeal-status change ho gaya"** — Expected hai,
   kuch sensitive-fields verification-status ko touch karte hain.

---

## 6. Quick Summary

- User Add/Update = supplier/buyer ke master-profile-record (`GLUSR_USR`) ka core
  create/update-engine.
- Duplicate-account-prevention (mobile/email/GSM-based).
- Dual-DB-write (main-profile + auth-credentials).
- 50+ downstream-systems ko async-sync — poore users-domain ka foundation.

---

## See also

- [`User_Add_Update_Technical_Doc.md`](./User_Add_Update_Technical_Doc.md) — code-level detail
- Almost every other KT folder in this docs directory attaches data against the
  `GLUSR_USR_ID` created/managed by this feature.

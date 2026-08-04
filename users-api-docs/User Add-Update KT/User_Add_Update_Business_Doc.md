# User Add / Update (Core Registration & Profile) — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai — koi code nahi, sirf "kya hota hai, kyun
hota hai." Code ke liye [`User_Add_Update_Technical_Doc.md`](./User_Add_Update_Technical_Doc.md)
dekho.

**Note**: Yeh IndiaMART users-domain ka **sabse foundational feature** hai — supplier/buyer
ka master-record (`GLUSR_USR`) yahi se create/update hota hai. Baaki bahut saare KT folders
(GST, Bank Details, Fact Sheet, Social Contacts, SellOnIM, Negative Mcat, etc.) isi
master-record ke against extra-data attach karte hain — unn sabka foundation yeh feature hai.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Jab koi naya supplier/buyer IndiaMART pe **sign-up** karta hai, unka core-profile-record
(naam, mobile, email, address, company-details, login-credentials) yahan create hota hai.
Baad mein, jab bhi wo apna profile **update** karta hai (naya address, naya mobile,
company-name-change, listing-status-change, etc.), wahi record yahan se update hota hai.

Yeh sirf ek "form-submit" nahi hai — iske peeche ek poora engine chalta hai jo:

1. **Duplicate-accounts rokta hai**: agar same mobile/email/business-registration-number se
   pehle se koi account hai, naya account nahi banta, existing account hi update ho jaata hai.
2. **Dual-system rakhta hai sync**: profile-data (`GLUSR_USR`) aur login-credentials (alag
   "auth" system) dono ek saath likhe jaate hain — dono ke bina account usable nahi hota.
3. **TrustSeal ko touch karta hai**: kuch sensitive-fields (address/mobile/email/company-name)
   update karne se ek existing TrustSeal-verification apne-aap "unverified" flag ho sakti hai —
   yeh isliye zaroori hai ki verified-badge sirf tab tak trustworthy rahe jab tak underlying
   data change nahi hui hai.
4. **50+ downstream-systems ko sync rakhta hai**: har account-create/update ka event turant
   dusre bahut saare internal-systems (search, trust, alerting, LMS, CSL/search, enquiry,
   banned-content-detection, etc.) ko async-notify hota hai.

**Bottom line**: Yeh IndiaMART ki poori user-identity ka source-of-truth hai. Har
supplier/buyer ka ek unique account yahin se manage hota hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Naya Supplier/Buyer** | Signup karta hai — naya account create hota hai |
| **Existing Supplier/Buyer** | Apna profile update karta hai (address/mobile/email/company/listing-status, etc.) |
| **Bahut saare internal-apps (42+ allowed apps for Add, 43 for Update)** | Registration/update-flow ko route kar sakte hain — MY, GLADMIN, TOLLFREE, LEAP, MAPI, ENQUIRY, TRADE, PAYNOW, IMOB, WHATSAPP, EXPORT, etc. |
| **`GLUSR Manual cron`** | Ek internal "cron"-labelled updater — jab `UPDATEDBY` field yeh exact string ho, update ek alag ("cron") RabbitMQ-lane se route hota hai, taaki downstream consumers bulk/automated updates ko user-driven updates se differentiate kar sakein |
| **TrustSeal / verification systems** | Kuch profile-fields update hone par apna verification-status re-evaluate karte hain |
| **50+ downstream-systems** | Har add/update ka event consume karke apna copy sync rakhte hain (search, trust, alert, LMS, CSL, enquiry, banned-content-detection, etc.) |
| **Banned-content-detection system** | Naam/company-naam/address/designation jaisi text-fields change hone par flag ho sakta hai (fraud/abuse-content check) |

---

## 3. Business Flows (step-by-step, no code)

### Flow A — Naya Signup (Add)

```
1. User apna naam, mobile, email, address, company-details submit karta hai
2. System pehle check karta hai — kya yeh mobile/email/GSM (business-registration-number)
   already kisi existing-account se juda hai? Teeno independently check hote hain.
3. Agar koi bhi match milta hai:
   - Naya account NAHI banta
   - System decide karta hai KAUN sa existing-account "primary" hai (mobile-match ko
     priority milti hai email-match ke upar)
   - Us existing-account ko "internally update" kiya jaata hai — jo bhi naye fields diye
     gaye the (naam, address, company, etc.) unse existing-record ke sirf khaali/missing
     fields fill kiye jaate hain (existing non-empty values overwrite NAHI hote)
   - Agar naya email diya gaya tha lekin mobile-match wale account ka email already kuch
     aur hai, toh email conflict ke roop mein reject ho jaata hai ("Primary Email id
     already exist")
4. Agar teeno mein se koi match nahi milta, ek bilkul naya account create hota hai:
   - Login-credentials bhi ek alag "auth" system mein create hoti hain (dual-write)
   - Account seedha "Approved" status ke saath directly active hota hai — koi
     pending-approval-step nahi
5. Naye-account/updated-account ka event 50+ downstream-systems ko async-notify hota hai
```

**Business impact**: Duplicate-account-prevention ek core data-quality-safeguard hai — same
business/person ke multiple fraudulent-accounts banne se rokta hai, aur jahan tak ho sake
existing data ko preserve karta hai (naya submit sirf gaps fill karta hai, purana data
overwrite nahi karta).

### Flow B — Profile Update

```
1. Existing user apna koi bhi profile-field update karta hai (address, mobile, email,
   company-name, listing-status, etc.) — partial-update supported, sirf diye-gaye fields
   change hote hain
2. System purane values ke against naye values compare karta hai (sirf genuinely-changed
   fields hi processing trigger karte hain)
3. Kuch specific "sensitive" fields (company-name, address, city, state, zip, country, phone,
   fax, designation, turnover, employee-count, legal-status, etc.) agar change hote hain,
   TrustSeal ke against unko "unverified" mark kiya jaata hai — matlab TrustSeal ka status
   ab is naye data ko reflect nahi karta, dobara-verification chahiye
4. Agar update `TRUSTSEAL` screen se khud aaya hai (matlab TrustSeal team/process khud
   verified-value likh rahi hai), toh yeh unverification-logic reverse-treat hoti hai — TrustSeal
   apna khud ka verified-data likh raha hai, apne aap ko unverify nahi karega
5. Agar TrustSeal se update aa raha hai AUR address/mobile/email jaisi "TS-locked" fields
   already-verified account pe change ho rahi hain, un fields ko is update se explicitly
   BAAHAR nikaal diya jaata hai — TrustSeal khud in fields ko change nahi kar sakta ek
   normal update-call se
6. Update ka event downstream-systems ko notify hota hai — normal user-driven update ek
   route se jaata hai, `GLUSR Manual cron` label wale automated/bulk-updates ek alag route se
   (taaki downstream systems dono ko differently treat kar sakein)
7. Kuch naam/company-naam/address/designation jaisi text-fields change hone par
   banned-content-detection ko bhi notify kiya jaata hai (abuse/fraud-content scan ke liye)
```

**Business impact**: Yeh ensure karta hai ki TrustSeal ka "verified" badge hamesha current
data ko reflect kare — agar koi verified supplier apna address ya company-name change karta
hai, buyer ko purane (ab-invalid) data pe verified badge dikhna nahi chahiye.

---

## 4. TrustSeal Impact — Kaunse Fields Verification Todte Hain

Business perspective se yeh important hai — agar supplier poochhe "maine sirf apna phone
number update kiya, TrustSeal verified status kyun chala gaya," yeh table iska jawaab hai:

| Field-category | TrustSeal pe impact agar change ho |
|---|---|
| Company Name, Address (1/2), City, State, Zip, Country, Legal Status, Turnover, Employee-count, Trade Membership, Designation, Phone/Fax numbers, First/Last Name | **Verification "unverify" ho jaati hai** us specific attribute ke liye — TrustSeal ko pata chal jaata hai ki underlying data change hui, dobara verify karna padega |
| Location (City/State ID, lat-long) | Unverify ke saath-saath ek "location-unverification" bhi trigger hoti hai separately |
| Address/Mobile/Email/City/State/Zip/Country (already-TrustSeal-verified account pe) | **TrustSeal khud in fields ko touch nahi kar sakta** normal update se — yeh fields TrustSeal-update-path se explicitly hata di jaati hain |

**Business rule jo yaad rakhne layak hai**: Yeh ek automatic, granular system hai — sirf
"verified ya nahi" ek boolean nahi, balki har individual sensitive-field ka apna
verification-status hai jo independently unverify ho sakta hai jab wo specific field change
ho.

---

## 5. Business Rules — Plain Language Mein

1. **Duplicate-detection mobile, email, aur GSM (business-registration-number) teeno pe hoti
   hai** — agar koi bhi match milta hai, naya account nahi banta, existing-account update ho
   jaata hai (priority: mobile-match sabse pehle check hota hai, phir email, phir GSM).
2. **Existing data ko overwrite nahi kiya jaata duplicate-match ke case mein** — sirf khaali
   fields fill hoti hain naye submitted data se; jo already bhara hua hai wo protected rehta
   hai.
3. **Naya account seedha "Approved" status mein banta hai** — koi pending-approval-step
   nahi (immediate-activation). Moderation/fraud-check baad mein, alag downstream-processes
   se hota hai.
4. **Do alag-alag databases update hote hain** — main-profile (`GLUSR_USR`) aur
   login-credentials (separate "auth" system) — dono sync rehte hain; agar dono mein se ek
   fail ho, "profile ban gaya lekin login nahi ban paya" jaisi state theoretically ban sakti
   hai (dekho Technical Doc Open Questions).
5. **Kuch sensitive-fields update karne se TrustSeal-verification per-attribute reset ho
   sakta hai** — yeh granular hai (section 4), sirf specific changed-attributes unverify
   hoti hain, poora TrustSeal nahi.
6. **`TRUSTSEAL` screen se aaya update special-treat hota hai** — TrustSeal apni khud ki
   verified-value likh sakta hai bina khud ko unverify kiye, lekin already-verified
   address/mobile/email/location fields ko ek normal update-call se change nahi kar sakta.
7. **Bahut saare internal-apps (42+ Add, 43 Update) yeh operations perform kar sakte hain.**
8. **Update partial ho sakta hai** — sirf diye-gaye fields hi change hote hain, aur sirf
   genuinely-different-from-old values hi processing (TrustSeal-check, downstream-notify)
   trigger karte hain.
9. **Automated/bulk updates ek alag "cron" lane se route hote hain** — jab updater
   `GLUSR Manual cron` ho, downstream systems ise normal user-update se differently treat kar
   sakte hain.
10. **Naam/company-naam/address/designation change hone par banned-content-detection ko
    bhi notify kiya jaata hai** — fraud/abuse-text screening ke liye.

---

## 6. Notifications

| Kab | Kya hota hai |
|---|---|
| Naya account create hota hai | Downstream-systems (50+) ko async event — supplier ko koi explicit "welcome" email is layer mein directly nahi bheja jaata (dekho Technical Doc, is layer mein email-sending code trace nahi hua) |
| Duplicate mila, GST bhi diya tha | Ek extra internal "detail service" call hota hai — likely GST-related detail sync ke liye |
| Profile update hota hai | Downstream-systems ko event; TrustSeal-team ke liye unverification-signal agar applicable ho |

---

## 7. Edge Cases — Product/Business Perspective se

1. **"Maine naya account banane ki koshish ki, purana account mila"** — Expected hai agar
   mobile/email/GSM se pehle se koi account exist karta hai — system usi ko update kar deta
   hai, naya nahi banata.
2. **"Maine sirf apna address update kiya, TrustSeal-status change ho gaya"** — Expected hai,
   address ek TrustSeal-sensitive field hai (section 4).
3. **"Maine TrustSeal se apna address update kiya lekin nahi hua"** — Expected hai agar wo
   field already TrustSeal-verified thi — TrustSeal khud bhi apne locked-fields ko normal
   update-path se change nahi kar sakta.
4. **"Duplicate match hone ke baad mera email update nahi hua"** — Expected hai agar
   existing-account ka email already kuch aur value pe set hai — system existing non-empty
   email ko overwrite nahi karta, conflict ke roop mein reject karta hai.
5. **"Meri update request cron jaisi treat ho rahi hai"** — Sirf tab hota hai jab
   `UPDATEDBY` field literally `"GLUSR Manual cron"` ho — yeh ek internal/automated-updater
   ka signal hai, normal user-flow isse affect nahi hota.

---

## 8. Quick Summary

- User Add/Update = supplier/buyer ke master-profile-record (`GLUSR_USR`) ka core
  create/update-engine — poore users-domain ka foundation.
- Duplicate-account-prevention (mobile/email/GSM-based, priority-ordered), existing-data-
  preserving merge on duplicate-match.
- Dual-DB-write (main-profile + auth-credentials) on every new account.
- TrustSeal-impact granular hai — per-attribute unverification jab sensitive-fields change
  hon, TrustSeal-originated updates ko special-case treat kiya jaata hai.
- Normal vs "cron" updates ek label (`UPDATEDBY == "GLUSR Manual cron"`) se differentiate
  hote hain downstream ke liye.
- 50+ downstream-systems ko async-sync — poore users-domain ka foundation.

---

## See also

- [`User_Add_Update_Technical_Doc.md`](./User_Add_Update_Technical_Doc.md) — code-level detail
- Almost every other KT folder in this docs directory attaches data against the
  `GLUSR_USR_ID` created/managed by this feature — see `../GST KT/`, `../Bank Details KT/`,
  `../Fact Sheet KT/`, `../Social Contacts KT/` for how they build on top of this record.

# PNS Setting — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai — koi code nahi, sirf "kya hota hai, kyun
hota hai, aur supplier ke liye iska matlab kya hai." Code ke liye
[`PNS_Setting_Technical_Doc.md`](./PNS_Setting_Technical_Doc.md) dekho.

**Note**: PNS Setting aur PNS **do alag concepts** hain — PNS ek GSM/virtual-number-
allocation-service hai, yeh feature supplier ki **contact/notification-preferences**
manage karta hai. Dekho [`../PNS KT/PNS_Business_Doc.md`](../PNS%20KT/PNS_Business_Doc.md).

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Supplier apne **off-hours call-routing preferences** set kar sakta hai — jab buyer business
hours ke bahar call kare, toh woh call kis number pe route ho, aur us waqt kaunsa behavior
chahiye (feature on/off). Yeh setting supplier ke **har alag contact-number** ke against
individually configure ho sakti hai — primary mobile, alternate mobile, primary landline,
secondary landline, aur unke multiple additional-contacts (jinke apne mobile/landline/
toll-free number ho sakte hain).

**Bottom line**: Yeh feature supplier ko control deta hai ki off-hours mein unke customers
ki calls kaise handle hon — koi call miss na ho, sahi number pe ring ho, ya feature disable
rakha ja sake jahan zaroorat na ho.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Apni PNS-off-hours-setting seller panel/app se configure karta hai |
| **Internal-tools (GLADMIN, MAPI, BUYERMY, SELLERMY, Android, iOS)** | Setting update kar sakte hain, agar unke paas valid access ho |
| **Platform khud (automatic background sync)** | Jab bhi supplier apna mobile/landline/GSM number kahin aur (profile-edit se) change karta hai, platform automatically PNS-setting ko naye number se sync kar deta hai — supplier ko dobara PNS-setting screen pe jaake kuch karna nahi padta |

---

## 3. Contact-Types — Supplier ke liye iska matlab

Har PNS setting ek specific "type" ke against hoti hai — yeh decide karta hai kis number ke
liye yeh setting apply hoti hai:

| Type | Kaunsa number | Scope |
|---|---|---|
| Primary Mobile (`M1`) | Supplier ka main mobile number | General — poore account ke liye ek |
| Alternate Mobile (`M2`) | Supplier ka doosra/alt mobile number | General — poore account ke liye ek |
| Primary Landline (`L1`) | Supplier ka main landline | General — poore account ke liye ek |
| Secondary Landline (`L2`) | Supplier ka doosra landline | General — poore account ke liye ek |
| Additional-Contact Mobile (`M`) | Ek specific additional-contact ka mobile | Per-contact — supplier ke multiple additional-contacts ho sakte hain, har ek apni setting rakh sakta hai |
| Additional-Contact Landline (`L`) | Ek specific additional-contact ka landline | Per-contact |
| Additional-Contact Toll-free (`T`) | Ek specific additional-contact ka toll-free number | Per-contact |

**Business rule jo yaad rakhne layak hai**: Ek hi phone number sirf **ek** type ke saath
active mapping rakh sakta hai kisi bhi waqt. Agar supplier ka koi number do jagah use ho raha
hai (jaise woh unka primary mobile bhi hai aur ek additional-contact ka mobile bhi), system
automatically ek ko priority deta hai (naye/zyada-priority waale ke against purana hata deta
hai) — dono simultaneously active nahi reh sakte.

---

## 4. Main Business Flows (step-by-step, no code)

### Flow A — Supplier apna PNS setting configure karta hai (naya add ya update)

```
1. Supplier (ya internal tool jaise GLADMIN/MAPI/App) PNS-setting screen pe jaake
   ek contact-type select karta hai (jaise "Primary Mobile" ya ek specific
   additional-contact)
2. Off-hours behavior on/off (aur konsi state) set karta hai
3. System khud us type ka actual phone number nikal leta hai supplier ke profile
   se (supplier ka diya hua number trust nahi karta, khud verify karta hai ki woh
   number valid/numeric hai)
4. Agar yeh number kisi doosre type ke saath already mapped hai, system decide
   karta hai kaunsa mapping rakhna hai (section 3 ka priority rule)
5. Naya record insert hota hai (pehli baar), ya existing update hota hai (dobara)
6. Duplicate-submission silently handle hoti hai — error nahi aata
```

**Business impact**: Supplier apne off-hours-call-experience ko fine-tune kar sakta hai,
per-number basis pe.

### Flow B — Supplier apna PNS setting hata deta hai

```
1. Supplier ek specific type ke liye setting delete karta hai
2. System us type ke against record hata deta hai
3. Ek audit-trail entry banti hai (kisne, kab, kaunsa number un-map hua)
```

**Business impact**: Off-hours-routing us number ke liye band ho jaata hai; history record
rehta hai future reference ke liye (sirf delete ka, insert ka nahi).

### Flow C — Supplier apna phone number kahin aur change karta hai (automatic sync)

```
1. Supplier apna mobile/landline/GSM number apne profile ke "Contact Details"
   section se change karta hai (yeh ek alag feature/screen hai, PNS-setting
   screen nahi)
2. Background mein, platform automatically dekhta hai: iss number pe kya koi
   PNS-setting already active thi?
3. Agar haan, purani setting ka number bhi automatically update ho jaata hai
   naye number pe (ya agar number hi hata diya gaya, setting bhi hat jaati hai)
4. Agar supplier ke paas 5 se kam existing PNS-mappings hain, naya number
   automatically ek naya mapping bhi bana sakta hai
```

**Business impact**: Supplier ko apna mobile number update karne ke baad dobara jaake
PNS-setting fix karne ki zaroorat nahi padti — system khud sync kar deta hai. Yeh ek
**"invisible convenience feature"** hai jiske baare mein support team ko pata hona chahiye,
kyunki supplier ko kabhi pata bhi nahi chalta ki yeh background mein ho raha hai.

### Flow D — Supplier ek additional-contact hi delete kar deta hai

```
1. Supplier apna ek additional-contact number (jo unke company profile mein
   extra contact ke roop mein add tha) poora hata deta hai
2. Us contact se judi saari PNS-settings (Mobile/Landline/Toll-free, jo bhi
   configure thi) bhi automatically hat jaati hain, alag se delete karne ki
   zaroorat nahi
```

**Business impact**: Supplier ko manually PNS-setting cleanup nahi karna padta jab woh
apna koi additional contact remove karte hain — system consistent state khud maintain
karta hai.

### Flow E — Data read karna (PNS status dekhna)

```
1. Koi bhi authorized caller (supplier ki app, internal tool) supplier ka
   glusr_id deke poochta hai "iska current PNS-setting status kya hai"
2. System supplier ke saare numbers (primary/alt mobile, landlines, saare
   additional-contacts) aur unki current PNS-setting (agar hai) ek saath
   combine karke deta hai
3. Ek "bulk check" variant bhi hai — ek saath 10 tak GLIDs poochke sirf
   yeh jaana ja sakta hai "in mein se kis-kis ka off-hours-routing currently
   active hai" (detail ke bina)
```

**Business impact**: Internal teams/tools ek hi call mein poori PNS-picture dekh sakte hain
supplier ke saare numbers ke against.

---

## 5. Business Rules — Plain Language Mein

1. **Setting insert-or-update hoti hai** — pehli baar naya record, dobara update.
2. **Do variants hain** — general (primary/alt mobile, primary/secondary landline — per
   user, contact-select ki zaroorat nahi) aur specific-contact-linked (additional-contact
   ke mobile/landline/toll-free — ek particular contact ID se link hoti hai).
3. **System khud number verify karta hai** — supplier jo number screen pe daalta/select
   karta hai, uska actual value system apne profile-record se nikaalta hai, blindly trust
   nahi karta.
4. **Ek number, ek active PNS-type** — same number do jagah simultaneously active PNS
   setting ke saath nahi reh sakta; conflict hone par system automatically decide karta hai
   kaunsa rakhna hai.
5. **Duplicate-submission errors nahi deti** — `ON CONFLICT DO NOTHING` jaisa
   idempotent-behavior.
6. **Sirf internal/authorized tools hi setting change kar sakte hain directly** — GLADMIN,
   MAPI, BUYERMY, SELLERMY, Android app, iOS app — koi bhi random access allowed nahi.
7. **5-mapping-per-supplier limit sirf automatic-sync path mein enforce hoti hai** — jab
   system khud background mein numbers sync karta hai (Flow C), tabhi yeh cap lagti hai;
   agar koi internal tool directly setting add kare, yeh limit currently code mein disabled
   hai (dead/commented code found).
8. **Phone-number-badalne pe automatic sync hoti hai** (Flow C) aur **additional-contact
   delete karne pe automatic cleanup hoti hai** (Flow D) — dono cases mein supplier ko
   khud kuch alag se karne ki zaroorat nahi.
9. **Sirf delete operations ka audit-trail banta hai** — naya add karne ka koi history-log
   nahi likha jaata, sirf delete/unmap hone ka.

---

## 6. Notifications

**Iss feature ke liye koi dedicated supplier-facing email/SMS notification code mein nahi
mila** — na insert pe, na update pe, na delete pe. Yeh ek purely backend/config-level
setting hai jo directly supplier ko notify nahi karti jab woh (ya automatic-sync) change
hoti hai. **[INFERRED — confirm with team ki kya koi upstream/downstream notification hai
jo iss review ke scope se bahar hai]**

---

## 7. Edge Cases — Product/Business Perspective se

1. **"Maine setting dobara submit ki, kuch change nahi hua"** — Expected hai agar
   exact-same-values already exist karti hain — duplicate silently skip hoti hai.

2. **"Meri PNS setting apne aap badal gayi, maine kuch nahi kiya"** — Yeh bug nahi hai. Jab
   bhi supplier apna mobile/landline number profile mein kahin aur update karta hai, PNS
   setting automatically us naye number pe sync ho jaati hai (Flow C). Support team ko
   supplier ko yeh samjhana chahiye ki dono features connected hain, alag nahi.

3. **"Maine ek contact delete kiya, uski PNS setting bhi gayab ho gayi"** — Yeh bhi expected
   behavior hai (Flow D) — jab additional-contact hi remove ho jaata hai, uski PNS setting
   bhi automatically saath mein clean ho jaati hai.

4. **Ek number sirf ek type ke saath active reh sakta hai** — agar supplier same number ko
   do jagah use kar raha hai (jaise unka primary mobile hi ek additional-contact ka bhi
   number ban gaya), system ek setting ko doosre ke against silently drop kar deta hai.
   "Meri ek setting missing hai" ka yeh ek common reason ho sakta hai.

5. **Directly-configured settings 5-mapping-limit se exempt hain** — agar koi internal tool
   (GLADMIN/MAPI) supplier ke liye directly setting add kare, currently unlimited mappings
   ban sakti hain (code mein yeh cap sirf automatic-sync ke liye active hai) — worth
   flagging agar business intent hamesha ek hard 5-limit tha.

---

## 8. Quick Summary

- PNS Setting = supplier ke off-hours-call-routing/contact-preferences, per-contact-
  configurable (primary/alt mobile, primary/secondary landline, aur har additional-contact
  ke apne mobile/landline/toll-free).
- Insert-or-update, duplicate-safe, sirf authorized internal tools ya app se directly
  configurable.
- Jab bhi supplier apna phone number kahin aur profile mein change karta hai, PNS setting
  automatically usi naye number pe sync ho jaati hai — koi manual step nahi.
- Jab additional-contact hi delete ho, uski PNS setting bhi automatically clean ho jaati hai.
- Ek number sirf ek PNS-type ke saath active reh sakta hai — conflict hone par system
  khud resolve karta hai.
- 5-mapping-per-supplier cap sirf automatic-sync path mein hai, direct admin-tool writes
  mein currently nahi.
- PNS (GSM-allocation) se koi relation nahi — alag concept, alag table.

---

## See also

- [`PNS_Setting_Technical_Doc.md`](./PNS_Setting_Technical_Doc.md) — code-level detail
  (APIs, DB tables, queries, automatic-sync consumer flow)
- [`../PNS KT/PNS_Business_Doc.md`](../PNS%20KT/PNS_Business_Doc.md) —
  unrelated concept, documented separately

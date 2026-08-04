# TrustSeal (Seal, Company, Misc & Trade Affiliations) — Business Doc

Yeh doc TrustSeal feature ko **business/product nazariye** se samjhata hai — koi code nahi,
sirf "kya hota hai, kyun hota hai, aur supplier/buyer ke liye iska matlab kya hai." Code ke
liye [`TrustSeal_Technical_Doc.md`](./TrustSeal_Technical_Doc.md) dekho.

**Scope note**: yeh doc **TrustSeal aur uski saari linked details** (Company info, Trade
Affiliations, Standard-Quality Certifications, Safety Certifications, Chamber-of-Commerce
memberships, Export Status) ek saath cover karta hai — yeh sab ek hi table-family hai
(`TRUSTSEAL_ID` se joined), ek hi write-controller se manage hoti hai. **Trust Verification**
(generic per-attribute verification — GST/bank/mobile/email) ek alag, broader system hai —
dekho [`../Trust Verification KT/`](../Trust%20Verification%20KT/Trust_Verification_Business_Doc.md).

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

**TrustSeal** IndiaMART ka premium trust-badge hai — ek supplier jo TrustSeal certified hai,
unke storefront pe ek special badge dikhta hai jo buyers ko batata hai "yeh supplier extra
verified/trusted hai." Yeh GST-verification (locked/tactical/OTP) se **alag, upar ka layer**
hai — TrustSeal ek internal-admin-driven, curated trust-signal hai, jabki basic GST verification
ek automated/self-serve process hai.

TrustSeal apne saath extra business-details bhi carry karta hai — ek supplier jo TrustSeal
certified hota hai, uske baare mein IndiaMART ke paas normal profile se **zyada gehri
jaankari** hoti hai:

- **Poora company-profile snapshot** — address, bank details, turnover, tax-registration
  numbers (PAN/TAN/VAT/GST/CST/Excise), employee-count, saal-establishment — sab TrustSeal ke
  apne record mein bhi capture hota hai, sirf basic profile se copy nahi
- **Company details** — kaunsi company(s) is seal ke associated hain, ek "primary" company kaun
  hai
- **Trade Affiliations aur Standard-Quality Certifications** — industry-associations ki
  membership, quality-certification badges (jaise ISO jaisi cheezein) jo supplier ne claim ki
  hain
- **Safety Certifications aur Chamber-of-commerce memberships** — additional credibility
  markers
- **Export Status** — kya yeh business exports bhi karta hai

**Business impact**: TrustSeal buyer ko ek quick, visual "yeh supplier extra credible hai"
signal deta hai — jaise ek verified-badge social-media platforms pe hota hai, waisa hi concept,
lekin peeche bahut zyada detailed business-profile ke saath backed.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **IndiaMART internal system (WEBERP)** | TrustSeal assign/manage karta hai — **supplier khud apne aap ko TrustSeal nahi de sakta**, yeh purely internal-admin-driven hai. Reads bhi mostly WEBERP-gated hain (dekho section 5, rule 1) |
| **Supplier** | TrustSeal ka beneficiary — badge aur uski detailed profile-information unke storefront pe dikhti hai |
| **Buyer** | TrustSeal badge aur uski details (company info, trade-affiliations, certifications) dekh ke trust-decision leta hai |

---

## 3. Main Business Flows (step-by-step, no code)

### Flow A — WEBERP naya TrustSeal aur uski company create karta hai

```
1. Internal team (WEBERP) ek naya TrustSeal create karta hai ek supplier ke liye — company ka
   naam, address, business-details, tax-registration numbers, bank-details sab is waqt capture
   hote hain
2. Isi request mein TrustSeal se pehli linked company bhi create ho jaati hai — kaunsi company
   iske under aati hai, kaun "primary" hai
3. Har change ka ek history/audit-trail maintain hota hai automatically (kya text change hua, kab)
```

**Business impact**: Ek supplier ko poora TrustSeal-profile ek hi internal-action mein
onboard kiya ja sakta hai — seal aur uski pehli company ek saath ban jaate hain.

### Flow B — TrustSeal ka data update/renew hota hai

```
1. Internal team TrustSeal ko update karta hai — end-date badhana (renewal), status-approve
   karna, ya koi bhi business-detail field change karna
2. TrustSeal ki update teen tarike se target ki ja sakti hai — TrustSeal ki apni ID se, ya
   supplier ki user-ID se, ya uske subscription/service-link se (jo bhi identifier WEBERP ke
   paas available ho us waqt)
3. Har update ka audit-trail (history) maintain hota hai — naya comment purane history text ke
   **shuru** mein add hota hai, taaki latest change sabse pehle dikhe
```

**Business impact**: Yeh ek **subscription-jaisa** concept hai — TrustSeal ki ek "end date"
hoti hai, matlab yeh permanent nahi hai, expire ho sakta hai, renew hone ki zaroorat pad sakti
hai (`FK_CUST_TO_SERV_ID` — customer-to-service link mila code mein, jo shayad kisi paid-service
se TrustSeal ko jodta hai).

### Flow C — Company-level linking manage hota hai

```
1. Internal team ek TrustSeal ke against company-links add/update/remove kar sakta hai
2. Ek TrustSeal ke against multiple company-records link ho sakte hain, ek "primary" flag ke
   saath (kaunsi company primary hai)
3. Company-link create karne se pehle system check karta hai ki jis TrustSeal se link kiya
   ja raha hai woh actually exist karti hai — orphan/dangling links create nahi ho sakte
4. Har change ka history yahan bhi maintain hota hai
```

**Business impact**: Ek TrustSeal ka scope multiple company-entities tak extend ho sakta hai —
jaise ek parent business jiske multiple registered branches/companies hon, sabko ek hi
TrustSeal ke through cover kiya ja sakta hai.

### Flow D — Trade Affiliations aur Standard-Quality Certifications link karna

```
1. Internal team ek TrustSeal ke against Trade Affiliation ya Standard-Quality Certification
   add karta hai (jaise industry-association membership ya quality-certification)
2. Do modes hote hain read karne ke: "abhi kya linked hai" (already added items) vs "kya add
   kiya ja sakta hai" (available, not-yet-linked items) — ek typical "add new" dropdown
   experience ke liye
3. Har linked item ki apni validity-date (kab tak valid hai) aur image/certificate-path bhi
   store hoti hai
```

**Business impact**: Buyer ko sirf ek generic "trusted" badge nahi dikhta, balki specific,
verifiable credentials (kaunsi association ka member hai, kaunsi quality-certification hai)
bhi dikhti hain — zyada granular trust-signal.

### Flow E — Safety Certifications, Chamber-membership, Export-Status ka data

```
1. Safety Certification aur Chamber-of-Commerce membership names ek master/lookup list se
   match kiye jaate hain (name se ID resolve hota hai)
2. Export Status ek static, predefined list hai (koi dynamic DB-lookup nahi) — supplier
   exports karta hai ya nahi, ek fixed set of options mein se choose hota hai
```

**Business impact**: Yeh sab additional, verifiable-credibility-markers hain jo TrustSeal
profile ko poora banate hain — buyer ko sirf "verified" nahi, balki "kis tarah verified/
credentialed" bhi pata chalta hai.

### Flow F — Buyer/App TrustSeal profile dekhta hai

```
1. Jab bhi koi TrustSeal-profile access hota hai (buyer-facing ya internal-tool se), 5 alag
   tarah ke reads ho sakte hain:
   a. Basic seal-data (naam, city, GST-masked, avg-rating)
   b. Poora aggregated profile (seal + contact + bank + other business-details ek saath)
   c. Generic lookup (TrustSeal-code, ID, ya company-ID se)
   d. Company-linkage details (kaunsi companies linked hain)
   e. Misc-details (trade-affiliations, certifications, chamber, export-status)
2. Sensitive data (GST number, email, mobile) ek hi jagah full nahi dikhta — GST partial-masked
   hota hai (beech ke characters `*` se), email/mobile bhi kuch cases mein masked hote hain
```

**Business impact**: Buyer ko trust-badge ke saath verifiable detail milta hai, lekin
supplier ki fully-sensitive information (poora GST number, poora email/mobile) expose nahi
hoti — ek balance privacy aur trust ke beech.

---

## 4. Business Rules — Plain Language Mein

1. **Sirf internal `WEBERP` system TrustSeal ko likh (create/update/delete) sakta hai** — koi
   bhi doosra caller (seller-panel, app, ya bhi GLADMIN nahi) iss specific write-API ko access
   nahi kar sakta. Yeh sabse tight-gated write-path hai teeno "trust" domains (GST, TrustSeal,
   Trust Verification) mein.
2. **Zyada-tar reads bhi WEBERP-restricted hain** — sirf TrustSeal ka *basic* data aur
   *aggregated-detail* view thodi loose validation rakhte hain; company-details, misc-details,
   aur generic-lookup reads sab literally `weberp` modid maangte hain. Matlab TrustSeal ka
   detailed data bhi ek open/public API nahi hai, internal-system-facing hai.
3. **TrustSeal ki ek "end date" hoti hai** — matlab yeh permanent nahi hai, expire ho sakta
   hai, renew hone ki zaroorat pad sakti hai.
4. **Har change ka history maintain hota hai, latest-pehle order mein** — jab bhi kuch update
   ho (seal khud, ya company-link), ek text-based history-log bhi update hota hai, aur naya
   comment hamesha purane text se **pehle** aata hai — taaki sabse recent change padhne mein
   sabse aasan ho.
5. **Company-level linking multiple companies support karta hai** — ek TrustSeal ke against ek
   se zyada company-records link ho sakte hain, ek "primary" flag ke saath.
6. **Company-link banane se pehle system validate karta hai ki parent TrustSeal exist karti
   hai** — koi orphan/dangling company-link nahi ban sakta.
7. **Trade Affiliations aur Standard-Quality Certifications optional add-on hain** — TrustSeal
   base-record ke bina yeh exist nahi kar sakte, lekin har TrustSeal ke saath yeh hona
   mandatory nahi. In dono ke liye "already linked" vs "abhi add kar sakte ho" — dono modes
   available hain.
8. **Sensitive fields masked hoke hi dikhte hain kai jagah** — GST number partial-mask (pehle-
   aur-akhri 2 characters visible, beech mein `*`), email ka local-part partial-mask, mobile
   number bhi kuch cases mein middle-digits mask hote hain.
9. **Export Status aur ek Standard-Quality-Group-list ek fixed, predefined set hai** — inke
   liye koi database-lookup nahi hota, yeh static options hain jo hamesha same rehte hain.

---

## 5. Notifications

**Iss review mein TrustSeal domain ke liye koi email/SMS/push-notification logic nahi mili** —
na write-path mein, na kisi consumer/queue mein (kyunki TrustSeal koi queue use hi nahi karta,
dekho Technical Doc section 6-8). GST domain ke ulat (jahan email confirmation supplier ko
jaata hai), TrustSeal purely WEBERP-internal action hai — supplier ko koi automated
notification is process ke through nahi milti [INFERRED — team se confirm karo ki koi
alag/manual communication-process hai ya nahi].

---

## 6. Edge Cases — Product/Business Perspective se

1. **"Maine GST verify karwaya, mujhe TrustSeal kyun nahi mila"** — Yeh do alag systems hain.
   GST-verification (dekho [`../GST KT/`](../GST%20KT/GST_Business_Doc.md)) ek related signal
   ho sakti hai, lekin TrustSeal khud sirf WEBERP (internal team) assign karti hai — koi
   automatic trigger GST-verify se TrustSeal tak code mein nahi mila.
2. **"Mera TrustSeal expire ho gaya, badge gayab ho gaya"** — Expected hai agar `end date`
   guzar chuki hai — renewal process internal team se consult karna padega.
3. **"Meri company-details TrustSeal pe galat dikh rahi hain"** — TrustSeal Company records
   multiple ho sakte hain ek seal ke against — check karo "primary" company kaunsi mark hai.
4. **"Maine ek Trade Affiliation add karne ki request bheji lekin available-list mein woh dikh
   hi nahi raha"** — ho sakta hai woh already linked ho — system do alag lists deta hai
   (linked vs available), agar item already linked hai toh available-list mein dobara nahi
   dikhega (yeh expected hai, bug nahi).
5. **"TrustSeal-profile mein mera pura GST/email/mobile number nahi dikh raha, adhoora hai"** —
   Yeh intentional masking hai (section 4, rule 8) — buyer-facing/internal-tool ki privacy ke
   liye poori sensitive value expose nahi hoti.
6. **"Kya TrustSeal automatically renew ho jaata hai, ya har baar manually karna padta hai"** —
   Iss review mein koi automated-renewal/background-job nahi mila (Technical Doc section 12) —
   sab kuch WEBERP ke on-demand action se hota hai, koi cron/scheduled-renewal process nahi hai.
7. **"Same TrustSeal data multiple tarike se query ho sakta hai — TrustSeal-code se, ID se,
   ya glusr-ID se — inme koi farak hai kya?"** — Haan, thoda: kuch reads sensitive-data ko mask
   karte hain, kuch nahi; kuch full-profile dete hain, kuch basic. Depend karta hai kaunsa
   specific endpoint/use-case hai.

---

## 7. Quick Summary

- TrustSeal = ek premium, internal-admin-assigned (WEBERP-only) trust-badge, GST-verification
  se upar ka, zyada detailed layer.
- Company-details, Trade-Affiliations, Standard-Quality-Certs, Safety-Certs, Chamber, Export-
  Status — sab TrustSeal se FK-linked hain, ek hi family.
- Sirf `WEBERP` (internal system) isse likh sakta hai — supplier ya GLADMIN bhi nahi. Zyada-tar
  reads bhi WEBERP-restricted hain.
- Expiry/renewal concept hai, subscription-jaisa (`CUST_TO_SERV` link) — koi automated-renewal
  job nahi mila, sab manual/WEBERP-driven hai.
- Har change ka audit-history maintain hota hai, latest-change-pehle order mein.
- Sensitive supplier-data (GST, email, mobile) kai reads mein partially masked hoke aata hai.
- Koi email/SMS notification supplier ko nahi jaati is process ke through (jitna is review mein
  mila) — GST domain se yeh alag behavior hai.

---

## See also

- [`TrustSeal_Technical_Doc.md`](./TrustSeal_Technical_Doc.md) — code-level detail (APIs, DB
  tables, queries, flow diagrams, optimization notes)
- [`../Trust Verification KT/Trust_Verification_Business_Doc.md`](../Trust%20Verification%20KT/Trust_Verification_Business_Doc.md) — related but separate: generic per-attribute verification system
- [`../GST KT/GST_Business_Doc.md`](../GST%20KT/GST_Business_Doc.md) — related "trust" domain whose GST/IEC data TrustSeal reads alongside its own
- [`../trust_verification_compliance_product_overview.md`](../trust_verification_compliance_product_overview.md) — bigger Trust, Verification & Compliance story

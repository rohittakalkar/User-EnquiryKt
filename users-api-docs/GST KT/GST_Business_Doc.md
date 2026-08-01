# GST — Business Doc (Product Perspective)

Yeh doc GST (Goods & Services Tax) feature ko **business/product nazariye** se samjhata hai —
koi code nahi, sirf "kya hota hai, kyun hota hai, aur supplier/business ke liye iska matlab
kya hai." Technical implementation (APIs, DB, queues, code) ke liye
[`GST_Technical_Doc.md`](./GST_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Jab bhi koi supplier IndiaMART pe apna business register karta hai ya profile update karta
hai, unse GST number maanga jaata hai. Yeh sirf ek text field bharna nahi hai — iske peeche
poora ek business process chalta hai jiska maksad hai:

1. **Buyer ka trust badhana**: Buyer jab kisi supplier se contact karta hai, unhe pata hona
   chahiye ki yeh ek real, tax-registered business hai, koi fraud ya fake seller nahi.
2. **Legal compliance**: India mein certain turnover ke upar business ko GST registered hona
   zaroori hai. IndiaMART is data ko capture karke platform ko compliant rakhta hai.
3. **Sahi tax classification (HSN)**: Supplier kya bech raha hai — iska tax-classification
   (HSN code) GST se link hota hai, taaki catalog aur tax dono sahi se classify ho.
4. **Business ka legal structure pata karna**: GST number ke andar hi encoded hota hai ki
   yeh business proprietorship hai, partnership hai, private limited company hai, ya kuch
   aur. IndiaMART khud yeh nikal leta hai GST number se — supplier se poochta nahi, kyunki
   supplier khud bata ke galat bhi bata sakta hai (ya jaan-boojh kar bade business jaisa
   dikhne ke liye galat category select kar sakta hai). GST number se nikala hua data
   trustworthy hai kyunki usse manipulate karna mushkil hai.

**Bottom line**: GST sirf ek field nahi hai, yeh trust aur compliance ka backbone hai jo
buyer-supplier dono ko protect karta hai.

---

## 2. Kaun-kaun involved hai iss process mein

| Kaun | Kya role hai |
|---|---|
| **Supplier** | Apna GST number submit/update karta hai seller panel ya app se |
| **IndiaMART Internal Admin (GLADMIN)** | Kuch special cases mein locked/verified GST ko override kar sakta hai, lekin sirf permission check pass karne ke baad |
| **BI GST Team (IndiaMART ki internal team)** | Ek external system maintain karti hai jo government GST database se cross-verify karta hai |
| **Ek daily automated job (cron)** | Roz raat ko chalta hai aur kal jitne bhi GST/PAN/CIN details change hue hain, unhe dobara verify karta hai |
| **Government GST Verification System (external)** | IndiaMART ke bahar ka system jo actual GST validity check karta hai — turnover, business nature, registration date jaisi asli details bhi deta hai |
| **Email/Notification system** | Supplier ko batata hai jab bhi unka GST update ho, ya reject ho jaaye |

---

## 3. GST Verification ke "levels" — supplier ke liye iska matlab

Har GST number ek "verification status" carry karta hai. Business perspective se yeh
important hai kyunki iska seedha asar hota hai ki supplier apna GST **dobara edit kar sakta
hai ya nahi**:

| Status | Business meaning | Supplier edit kar sakta hai? |
|---|---|---|
| **Unverified/Naya submit hua** | Abhi-abhi submit hua hai, ya auto-match ho ke basic level pe hai | Haan, freely edit kar sakta hai |
| **Tactical Verified** | System ne apne-aap match kar liya external GST database se | **Nahi** — lock ho jaata hai. Sirf GLADMIN (special permission ke saath) ya OTP se overrride ho sakta hai |
| **OTP Verified** | Supplier ne khud OTP se prove kiya ki yeh unka GST hai | **Nahi** — sabse strong lock. Sirf OTP dobara ya GLADMIN override |
| **Manual Verification Pending** | System khud confirm nahi kar paaya, ek insaan (internal reviewer) ko check karna hai | Depends on review outcome |
| **Rejected** | GST invalid nikla ya government database se match nahi hua | Supplier dobara sahi GST daal sakta hai |

**Business rule jo yaad rakhne layak hai**: Ek baar GST "Tactical" ya "OTP Verified" ho jaaye,
toh supplier usse normal profile-edit se change nahi kar sakta. Agar koi support ticket aaye
"main apna GST kyun edit nahi kar pa raha," toh iska jawaab yahi hai — yeh ek **security
feature** hai, bug nahi. Isse yeh hota hai ki koi bhi randomly kisi verified business ka GST
number apne profile pe daal ke fraud na kar sake.

---

## 4. Main Business Flows (step-by-step, no code)

### Flow A — Naya supplier apna GST submit karta hai

```
1. Supplier seller panel/app pe jaake apna GST number daalta hai (company details section mein)
2. System pehle check karta hai: GST number sahi format mein hai? (15 characters, valid pattern)
3. System check karta hai: yeh GST already kisi verified account se linked toh nahi?
4. Agar sab sahi hai -> GST record ban jaata hai, PAN number bhi automatically nikal liya
   jaata hai GST number ke andar se (kyunki GST mein PAN embedded hota hai)
5. Background mein: system external Government GST database se verify karne ki koshish karta hai
6. Supplier ko email jaata hai confirmation ka ("Your GST has been updated successfully")
7. Kuch minute/hours baad: agar auto-verification pass hui, GST "Tactical Verified" ban jaata hai
```

**Business impact**: Supplier ka profile ab thoda zyada trustworthy dikhta hai buyers ko,
verification badge/status update hota hai.

### Flow B — Supplier apna already-verified GST change karna chahta hai

```
1. Supplier naya GST number daalta hai
2. System dekhta hai: purana GST "Tactical" ya "OTP Verified" tha?
3. Agar HAAN aur normal supplier hai -> REJECT
   Message: "GST is already marked as Tactical/OTP Verified"
4. Agar internal admin (GLADMIN) hai AUR unke paas special permission hai -> allowed
5. Agar supplier OTP se prove karta hai ki yeh sach mein unka naya GST hai -> allowed
   (yeh ek override mechanism hai — even ek "rejected" GST ko bhi OTP se overrule kiya ja sakta hai)
```

**Business impact**: Yeh fraud prevention hai. Bina isse, koi bhi kisi trusted supplier ka
GST copy karke apne account mein daal sakta tha aur fake trust badge paa sakta tha.

### Flow C — Daily automated re-verification (background job)

```
1. Har raat, ek automated job chalta hai
2. Yeh dekhta hai: pichle 24 ghante mein kis-kis supplier ne apna GST/PAN/CIN/IEC change kiya
3. Un sabko dobara Government GST database se verify karta hai
4. Agar koi purana "rejected" GST ab valid nikalta hai -> automatically "verified" ban jaata hai
5. Agar koi verification fail ho gaya -> alert bhejta hai internal team ko (Google Chat pe)
```

**Business impact**: Yeh ek safety-net hai. Agar kisi wajah se real-time verification miss ho
gaya (jaise external government system down tha uss waqt), yeh daily job usko catch kar leta
hai aur automatically fix kar deta hai — bina kisi manual intervention ke.

### Flow D — GST se HSN (product tax classification) link karna

```
1. Internal team (GLADMIN/BI) ek GST number ke against HSN codes add/remove karta hai
   (HSN = Harmonized System Nomenclature, product ka tax-classification code)
2. System check karta hai: yeh HSN pehle se linked toh nahi hai (duplicate na ho)
3. Naye HSN insert ho jaate hain, purane (agar remove karne wale) delete ho jaate hain
4. Har change ka audit-trail record banta hai (kisne, kab, kya change kiya)
5. Us GST se linked har supplier account ko notify kiya jaata hai naye HSN mapping ke baare mein
```

**Business impact**: Iske through platform ko pata chalta hai supplier exactly kya
categories mein product bech raha hai — tax reporting aur catalog dono ke liye useful.

### Flow E — Legal Status apne-aap nikalna (Auto-detection)

```
1. Jab bhi ek GST verify ho jaata hai, system uske number ke andar se ek specific character
   nikalta hai (jo India ke tax-system rules ke hisaab se business type batata hai)
2. Us character ke basis pe, system decide karta hai: yeh business Proprietorship hai,
   Partnership/LLP hai, Private Limited Company hai, HUF hai, ya Trust/Government body hai
3. Yeh "Legal Status" supplier ke profile mein automatically set ho jaata hai
   (supplier se manually poocha nahi jaata)
```

**Business impact**: Business type ek trustworthy signal hai (GST se derive hota hai, self
declared nahi), jo buyer ko decide karne mein madad karta hai ki wo kis type ke business se
deal kar raha hai.

---

## 5. Business Rules — Plain Language Mein

Yeh sab rules hain jo GST process ko govern karte hain, jo maine actual code padh ke nikale
hain:

1. **GST hamesha 15 characters ka hona chahiye** — yeh Government ka standard format hai.
2. **Format aur checksum dono check hote hain** — sirf length sahi hona kaafi nahi, ek valid
   GSTIN pattern match karna chahiye, aur ek mathematical check-digit bhi verify hota hai
   (taaki typo/fake GST na chal jaaye).
3. **PAN apne aap nikal liya jaata hai GST se** — supplier ko alag se PAN dena nahi padta iss
   flow mein, kyunki GST number ke andar hi PAN embedded hota hai.
4. **Khaali GST daalna = GST delete karna** — agar supplier GST field khaali chhod ke submit
   karta hai, system usse "GST hataa do" ke roop mein treat karta hai, sirf ek chhoti si
   mistake nahi.
5. **Sirf GST verified hone ke baad PAN akela change nahi ho sakta** — kyunki PAN GST se
   linked hai, agar GST locked hai toh PAN bhi locked hai.
6. **Purane app versions ke liye extra safety check** — bahut purane app version (13.1.5 se
   pehle) use karne wale users ke liye ek extra verification step chalta hai, taaki purani
   app ki UI mein koi gap na ho jispe log accidentally verified GST edit kar de.
7. **Duplicate HSN links silently ignore ho jaate hain** — agar koi already-linked HSN dobara
   add karne ki koshish ho, ya already-removed HSN dobara delete karne ki koshish ho, system
   error nahi deta, bas usse ignore kar deta hai.
8. **HSN code sirf numbers ka hona chahiye, max 8 digit** — koi bhi invalid format wala HSN
   automatically filter ho jaata hai.

---

## 6. Notifications — Supplier ko kab pata chalta hai

| Kab | Kya notification jaata hai |
|---|---|
| GST successfully update hua | Email: "Your GST has been updated Successfully" |
| GST reject ho gaya (verification fail) | Email: GST rejection notice |
| Daily automated job ne GST update kiya | **Email NAHI jaata** — kyunki yeh background reconciliation hai, supplier ko disturb nahi karna |
| Development/testing environment mein | Email suppress rehta hai (real supplier ko galat email na jaaye) |

---

## 7. Edge Cases — Product/Business Perspective se

1. **"Mera GST verified dikhta hai, lekin main edit nahi kar pa raha"** — Yeh expected
   behavior hai, bug nahi. Verified GST ek security lock ke saath aata hai. Support team ko
   yeh samjhana chahiye supplier ko.

2. **"Maine GST daala, star rating 5/5 hai, phir bhi trust badge/status low dikh raha hai"**
   — GST verification status aur star ratings **do alag cheezein hain**. Ek high rating hone
   ka matlab yeh nahi ki GST bhi verified hai — dono independently check hote hain.

3. **Do "GST tables" jaisa concept internally hai** — ek jagah supplier ki basic company
   registration details store hoti hain (GST, PAN, CIN), ek alag jagah GST ka
   "verification status" store hota hai. Yeh dono kabhi-kabhi out-of-sync ho sakte hain agar
   koi step beech mein fail ho jaaye — is wajah se system automatic retry/reprocessing bhi
   karta hai in dono ko sync mein laane ke liye.

4. **Legal Status kabhi-kabhi galat/blank reh sakta hai** — yeh sirf tab compute hota hai
   jab GST verify ho chuka ho. Agar GST verification kabhi fail ho ya atka reh jaaye, legal
   status bhi blank reh jaayega. Yeh isliye important hai ki agar koi supplier bole "meri
   company type galat dikh rahi hai," pehle check karo unka GST verified hai ya nahi.

5. **Purana testing/QA workaround production mein hai** — ek specific test-GST number ko
   validation se bypass karne ki permission di gayi hai, lekin sirf whitelist ki hui internal
   IDs ke liye. Yeh production mein active hai — worth flagging agar security review ho.

---

## 8. Quick Summary — Ek Line Mein Har Cheez

- GST = trust + compliance + tax-classification, teeno ek saath.
- Verify hone ke baad GST **lock** ho jaata hai — normal edit nahi hota, fraud-prevention ke liye.
- Daily background job automatically stale/mismatched GST ko fix karta hai bina manual kaam ke.
- Legal Status (business type) GST se hi auto-derive hota hai, supplier khud nahi batata.
- HSN mapping GST se link hoti hai product tax-classification ke liye.
- Sab kuch email notification ke saath transparent hai supplier ke liye (except background/cron updates).

---

## See also

- [`GST_Technical_Doc.md`](./GST_Technical_Doc.md) — same flows, code-level detail (APIs, DB
  tables, queries, RabbitMQ/Kafka, consumers, crons)
- [`../trust_verification_compliance_product_overview.md`](../trust_verification_compliance_product_overview.md) — GST ka role bigger Trust & Compliance story mein

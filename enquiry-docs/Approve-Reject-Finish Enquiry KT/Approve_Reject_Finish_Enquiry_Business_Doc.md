# Approve / Reject / Finish Enquiry — Business Doc (Product Perspective)

Yeh doc teen closely-related enquiry-lifecycle endpoints ko **business/product nazariye** se
samjhata hai — koi code nahi, sirf "kya hota hai, kyun hota hai." Yeh teeno APIs ek already
**save ho chuki enquiry** (dekho Save Enquiry KT) ko aage move karte hain uski lifecycle mein.
Technical implementation ke liye
[`Approve_Reject_Finish_Enquiry_Technical_Doc.md`](./Approve_Reject_Finish_Enquiry_Technical_Doc.md)
dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Jab ek enquiry save hoti hai (Save Enquiry flow), har enquiry seedhe supplier tak nahi
pahunchti — kuch enquiries **"waiting"** state mein chali jaati hain (jaise agar unka content
kisi spam/abusive keyword se match kar gaya ho, ya koi aur review-worthy condition ho). Yeh
teen APIs isi "waiting" state ko aage resolve karte hain:

1. **Approve Enquiry** — ek waiting enquiry ko "theek hai" maan ke supplier tak live bhej deta
   hai.
2. **Reject Enquiry** — ek waiting enquiry ko "bounce" kar deta hai — yeh supplier tak kabhi
   nahi pahunchegi, aur reject hone ka ek reason record hota hai.
3. **Finish Enquiry** — ek already-live (waiting mein nahi) enquiry ke liye "communication"
   trigger karta hai — yaani supplier/buyer ko actual notification (Lead-Management + email/SMS
   comm payload) bhejne ka kaam yahin hota hai.

**Bottom line**: Save Enquiry sirf enquiry create karta hai; **Approve/Reject/Finish** decide
karte hain ki us enquiry ka final outcome kya hoga aur downstream systems ko kab, kaise inform
kiya jaaye.

---

## 2. Kaun-kaun involved hai iss process mein

| Kaun | Kya role hai |
|---|---|
| **Internal Admin/Reviewer (Employee ID / EmpID)** | Manually kisi waiting enquiry ko approve ya reject kar sakta hai (admin panel se) |
| **Automated FENQ Bot (system employee, EmpID `17967`)** | Ek background consumer (`enq_post_fenq.go`, Save Enquiry ka downstream) jo waiting enquiries ko **automatically** ek dusre abusive-keyword check ke against re-validate karta hai aur khud hi approve/reject call kar deta hai, bina kisi insaan ke |
| **Buyer** | Jiski enquiry approve/reject/finish ho rahi hai |
| **Supplier (Receiver)** | Jise approved/finished enquiry pahunchti hai |
| **BI/Abuse-detection service** | FENQ bot dwara call kiya jaata hai description aur product-name ko dobara scan karne ke liye, approve/reject decide karne se pehle |
| **Lead Management System (LMS)** | Har teen operations (approve/reject/finish) ke baad ek queue-message milta hai transaction record karne ke liye |
| **Communication/Notification pipeline** | Sirf Finish Enquiry se trigger hoti hai — supplier/buyer ko actual email/SMS jaane wala data yahan se banta hai |

---

## 3. Enquiry Lifecycle States — In Teeno APIs Ke Context Mein

| State/Table | Business meaning | Yahan se kya hota hai |
|---|---|---|
| **Waiting** (`DIR_QUERY_WAITING`) | Enquiry save toh ho gayi, lekin abhi supplier ko nahi dikh rahi — review-pending state | Approve/Reject dono is table se enquiry ko **nikaal** kar (`move`) aage bhejte hain |
| **Approved → Live** | Enquiry ab officially supplier-facing hai, ek normal `DIR_QUERY_*` (sharded) table mein chali jaati hai | Approve Enquiry ka final outcome |
| **Bounced/Rejected** (`DIR_QUERY_BOUNCED`) | Enquiry reject ho gayi, ek "reason" ke saath store hoti hai audit ke liye | Reject Enquiry ka final outcome |
| **Finished (Communication Sent)** | Enquiry already live hai, aur uske liye communication (LMS + email/SMS data) ban chuki hai — dubara nahi banegi (idempotent guard, DB flag `DIR_QUERY_MAIL_SEND`) | Finish Enquiry ka outcome |

**Business rule jo yaad rakhne layak hai**: Agar ek enquiry query ID "Query Not Found" bolta hai
Approve/Reject mein, iska matlab hai woh ab `DIR_QUERY_WAITING` mein nahi hai — ya toh already
resolve ho chuki hai (approved/rejected), ya kabhi waiting mein thi hi nahi. Yeh error nahi, ek
normal "already processed" scenario hai.

---

## 4. Main Business Flows (step-by-step, no code)

### Flow A — Admin manually enquiry approve karta hai

```
1. Internal reviewer admin panel se ek waiting enquiry dekhta hai
2. "Approve" button dabata hai (apna Employee ID ke saath)
3. System check karta hai: yeh query_id abhi bhi waiting list mein hai?
4. Agar haan -> enquiry waiting-table se nikal ke live table mein move ho jaati hai
5. Ek "approval reason" hamesha automatically "6" set hota hai is path mein (chahe request mein
   kuch bhi bheja gaya ho) — yeh ek fixed/default reason code hai admin-approvals ke liye
6. Lead Management System ko notify kiya jaata hai
7. Response mein naya (live) Query ID milta hai
```

**Business impact**: Yeh ek waiting-review process ka manual resolution hai — ek insaan decide
karta hai ki enquiry genuine thi, spam/abuse false-positive tha.

### Flow B — Admin manually enquiry reject karta hai

```
1. Internal reviewer admin panel se ek waiting enquiry dekhta hai jo genuinely spam/invalid lagti hai
2. "Reject" button dabata hai, ek specific "reason" select karta hai (jaise "Abusive Content",
   "Duplicate", etc. — reason codes internally ek chhoti list mein map hote hain)
3. System check karta hai: yeh enquiry abhi tak approve toh nahi ho chuki (agar ho chuki hai,
   reject nahi hoga)
4. Enquiry waiting-table se "bounced" table mein move ho jaati hai, reason ke saath
5. Lead Management System ko notify kiya jaata hai
6. Response mein ek "bounce ID" milta hai (approval ID nahi)
```

**Business impact**: Yeh ek quality-control final decision hai — is enquiry ko supplier kabhi
nahi dekhega, aur reject hone ka audit-trail reason ke saath store rehta hai.

### Flow C — Automated bot approve/reject karta hai (FENQ auto-resolution)

```
1. Jab ek enquiry pehli baar "waiting" state mein jaati hai (Save Enquiry flow se), ek background
   automated process ("FENQ bot", jo system employee ID 17967 use karta hai) usse turant
   dobara-check karta hai
2. 10-second wait ke baad, bot enquiry ke description aur product-name ko ek external
   abuse-detection service ke against dobara scan karta hai
3. Agar dono clean nikle -> bot khud "Approve" call kar deta hai (koi insaan involved nahi)
4. Agar koi ek bhi abusive nikla -> bot khud "Reject" call kar deta hai (ek fixed reason ke saath)
5. Agar bot ka pehla scan clean tha lekin baad mein ek dusra internal scoring-process (jo
   duplicate-check, quality-check jaisi cheezein karta hai) ek "bounce reason" nikalta hai -> bot
   phir se "Reject" call kar deta hai, is baar us specific reason ke saath
```

**Business impact**: Yeh ek fully-automated safety-net hai — insaan ki zaroorat sirf tab padti
hai jab automated checks kisi wajah se skip/fail ho jaayein, ya ek genuinely tricky case ho.
Zyaadatar waiting enquiries **automatically hi resolve ho jaati hain**, bina kisi manual review
ke, chand seconds mein.

### Flow D — Duplicate-entry safety net (Approve retry ke waqt)

```
1. Admin ya bot ek enquiry approve karne ki koshish karta hai
2. System ko pata chalta hai ki yeh enquiry already kisi aur process ke through move ho chuki
   thi (ek "duplicate" DB-level conflict)
3. System automatically is enquiry ko "reject/bounced" table mein daal deta hai (ek internal
   system-triggered reject call ke through, reason "-94" ke saath) taaki data lost na ho
```

**Business impact**: Race-condition/double-click jaisi situations mein bhi data consistent
rehta hai — enquiry kahin "stuck" nahi rehti, hamesha ek final resolved state (approved ya
bounced) mein pahunchti hai.

### Flow E — Enquiry "Finish" hoti hai (communication trigger)

```
1. Ek enquiry jo already supplier-facing/live hai (waiting mein nahi), uske liye "Finish" call
   hota hai — yeh Save-Enquiry ke downstream se automatically bhi ho sakta hai (kuch specific
   internal cases mein), ya explicitly kisi doosre system se
2. System check karta hai: kya iss enquiry ke liye communication pehle hi bheji ja chuki hai?
   Agar haan, dubara kuch nahi hota (idempotent — safe hai baar-baar call karna)
3. Agar nahi bheji gayi, system sender/receiver/attachment sab detail nikalta hai
4. Do cheezein parallel mein hoti hain: Lead Management System ko notify kiya jaata hai, aur
   (agar SMS-specific flag set nahi hai) ek full communication-data packet banta hai jo aage
   email/SMS bhejne wale system ko jaata hai
5. Enquiry ko "communication sent" mark kar diya jaata hai, taaki dubara na ho
```

**Business impact**: Yeh woh moment hai jab buyer/supplier ko actually pata chalta hai ki enquiry
process ho chuki hai — email/SMS yahin se trigger hoti hai. Idempotency-guard iske through
ensure karta hai ki koi buyer/supplier ko duplicate email na mile.

---

## 5. Business Rules — Plain Language Mein

1. **Approve karte waqt "approval reason" hamesha fixed "6" set hota hai** — chahe request mein
   koi aur value bheji gayi ho, system usse ignore karta hai aur hardcoded "6" use karta hai. Iska
   exact business meaning (kya "6" ek specific reason-type hai) team se confirm karna baaki hai.
2. **Reject karte waqt reason codes ek chhoti mapping table se guzarte hain** — supplier-facing
   ya UI-facing reason codes (`1, 3, 7, 8, 9, 10, 11`) internally ek alag "bounce reason ID"
   (`80-86`) mein convert ho jaate hain. Iska exact business meaning bhi confirm karna baaki hai.
3. **Ek approved enquiry dubara reject nahi ho sakti** — Reject ka DB-update explicitly check
   karta hai `FK_APPROVAL_REASON_ID IS NULL`, yaani agar enquiry already approve ho chuki hai
   (uska approval-reason set ho chuka hai), reject silently no-op ho jaata hai.
4. **"is_instant" flag decide karta hai LMS mein transaction kaise record hota hai** — yeh
   business-facing distinction nahi hai (buyer/supplier ko farq nahi padta), lekin internal
   reporting/LMS ke liye alag insertion-type track hota hai instant (admin panel se turant) vs.
   non-instant calls ke liye.
5. **"Query Not Found" ek normal outcome hai, error nahi** — agar ek enquiry Approve/Reject mein
   already resolve ho chuki thi (ya kabhi waiting mein thi hi nahi), system ek "success" response
   deta hai jisme sirf "Query Not Found" message hota hai — yeh galat/duplicate action attempt ko
   gracefully handle karta hai.
6. **FENQ automated bot sabse pehle 10 second wait karta hai** waiting-state process shuru karne
   se pehle — likely ek race-condition-avoidance buffer (taaki DB writes settle ho jaayein).
7. **FENQ bot description aur product-name dono ko independently check karta hai** — agar dono
   mein se koi bhi ek abusive nikle, poori enquiry reject ho jaati hai. Dono clean hone chahiye
   approve ke liye.
8. **Finish Enquiry sirf `query_destination = 1` (normal/direct) enquiries ke liye chalta hai** —
   agar koi aur destination value bheji jaaye, request "wrong" bol ke reject ho jaata hai. Yeh
   ensure karta hai ki waiting/bounced enquiries ke liye Finish kabhi galti se na chale.
9. **Finish Enquiry duplicate-safe hai** — agar communication pehle se already bheji ja chuki hai
   (DB flag check), dubara request aane par kuch nahi hota, buyer/supplier ko doosri baar email
   nahi milegi.
10. **Communication data ka ek hissa (SMS-specific) skip ho jaata hai** agar request explicitly
    bole "yeh sirf SMS-context hai" (`email_sms_comm = 1`) — us case mein sirf LMS update hota
    hai, poora communication-packet nahi banta.

---

## 6. Notifications — Kab Kya Jaata Hai

| Kab | Kya hota hai |
|---|---|
| Enquiry approve hoti hai (manual ya bot) | Lead Management System ko queue-notification |
| Enquiry reject hoti hai (manual ya bot) | Lead Management System ko queue-notification (bounce ID ke saath) |
| Enquiry finish hoti hai | Lead Management System ko queue-notification, **plus** (agar SMS-only flag nahi hai) ek poora communication-data packet jo email/SMS bhejne wale downstream system ko jaata hai — **yehi woh jagah hai jahan se buyer/supplier ko actual pata chalta hai** |
| Approve/Reject ke turant baad buyer/supplier ko koi seedha email/SMS | **Nahi** — Approve/Reject sirf enquiry ko lifecycle mein move karte hain; asli communication sirf **Finish Enquiry** se trigger hoti hai |
| Duplicate Finish Enquiry call | Koi dubara communication nahi — idempotent guard block kar deta hai |

---

## 7. Edge Cases — Product/Business Perspective se

1. **"Maine enquiry approve ki, lekin lagta hai kuch action nahi hua"** — Pehle check karo query
   ID abhi bhi waiting-state mein tha ya nahi. Agar "Query Not Found" mila tha, matlab enquiry
   already resolve ho chuki thi (dobara approve karne ki zaroorat nahi thi).

2. **"Meri approved enquiry ka reason wahi nahi hai jo maine socha tha"** — Approval reason
   hamesha "6" fixed set hota hai is API mein, chahe frontend kuch bhi bheje. Yeh expected hai.

3. **"Maine ek approved enquiry reject karne ki koshish ki, kuch nahi hua"** — Yeh intentional
   guard hai — ek baar approve ho chuki enquiry reject nahi ho sakti through is path se.

4. **"Same enquiry do baar 'finish' ho gayi kya?"** — Nahi, system duplicate-safe hai. Agar
   communication pehle se ja chuki thi, doosri call kuch nahi karti — buyer/supplier ko doosri
   baar email nahi jaati.

5. **"Kya har waiting enquiry ke liye ek insaan review karta hai?"** — Nahi. Zyaadatar cases mein
   ek automated bot (Flow C) hi turant approve/reject kar deta hai. Manual review sirf edge-cases
   ke liye hai.

6. **"Reject reason ID jo maine bheja, wo kuch aur dikh raha hai backend mein"** — Kuch specific
   reason IDs (`1,3,7,8,9,10,11`) automatically ek doosri internal ID (`80-86`) mein convert ho
   jaate hain. Yeh expected mapping hai (section 5, rule 2).

---

## 8. Quick Summary — Ek Line Mein Har Cheez

- Approve/Reject sirf **waiting-state** enquiries ke liye hai — enquiry ko final resolve karte hain.
- Finish sirf **already-live** enquiries ke liye hai — actual communication (LMS + email/SMS data)
  trigger karta hai.
- Zyaadatar approve/reject decisions ek **automated bot** (FENQ consumer) khud hi le leta hai,
  bina kisi insaan ke.
- Ek approved enquiry dubara reject nahi ho sakti; ek already-finished enquiry dubara communication
  trigger nahi karti — dono jagah duplicate-safety built-in hai.
- Approve/Reject khud buyer/supplier ko notify nahi karte — sirf Finish Enquiry se asli
  communication trigger hoti hai.

---

## See also

- [`Approve_Reject_Finish_Enquiry_Technical_Doc.md`](./Approve_Reject_Finish_Enquiry_Technical_Doc.md) — same flows, code-level detail
- [`../Save Enquiry KT/Save_Enquiry_Business_Doc.md`](../Save%20Enquiry%20KT/Save_Enquiry_Business_Doc.md) — enquiry lifecycle ka starting point; waiting-state enquiries jo Approve/Reject yahan resolve karte hain, wahin se aati hain (Flow F/B ka continuation)

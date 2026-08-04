# Visiting Card — Business Doc (Product Perspective)

Yeh doc Visiting-Card feature ko **business/product nazariye** se samjhata hai — koi code
nahi, sirf "kya hota hai, kyun hota hai, aur supplier/business ke liye iska matlab kya hai."
Technical implementation (APIs, DB, queues, code) ke liye
[`Visiting_Card_Technical_Doc.md`](./Visiting_Card_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Supplier ka **physical visiting-card (business-card)** ka photo — front-side aur/ya
back-side — capture karke IndiaMART profile ke saath store kiya jaata hai. Yeh sirf ek photo
upload nahi hai — iske peeche ek poora **multi-stage internal-review workflow** chalta hai:
kisne front/back capture kiya, internal WebERP-team ne uska content "enrich" kiya (details
extract/verify ki), design-team ne quality approve ki, aur final approval mila ya nahi — sab
kuch ek hi record pe track hota hai.

Agar kisi supplier ke paas visiting-card hi nahi hai, ek explicit **"No Visiting Card"
(NO_VC)** status bhi alag se track hoti hai — taaki system yeh distinguish kar sake "abhi tak
upload nahi hua" vs "supplier ke paas hai hi nahi".

**Bottom line**: Visiting-card supplier-verification/trust ka ek **physical-proof** hai —
GST/ID-Proof jaisa hi ek aur trust-signal — lekin yahan process khud multi-team hai (capture
→ enrich → design → approve), aur approval-tracking ek **alag, dedicated internal database**
mein hoti hai jo customer-type/paid-status/work-order-count jaisi context bhi saath mein
carry karta hai taaki internal-reviewers priority-order mein kaam kar sakein.

---

## 2. Kaun-kaun involved hai iss process mein

| Kaun | Kya role hai |
|---|---|
| **Field-team / supplier** | Visiting-card photo capture karte hain (front/back), ya bata dete hain ki unke paas card nahi hai (NO_VC) |
| **IndiaMART Internal Admin (GLADMIN)** | Comments/annotations add karta hai record pe |
| **WebERP (internal-workflow-team)** | Teen sub-roles nibhaata hai: **Enrich** (content review/extract), **Design** (design-quality approve), **Approve** (final approval/rejection dena) |
| **Approval-tracking system (background)** | Har card-event ko ek alag internal database mein sync karta hai, jahan supplier ka paid-status, work-order-count, aur enterprise-type bhi attach hota hai — taaki reviewers ko priority-context mile |
| **Buyer / koi bhi caller** | Visiting-card display dekh sakta hai supplier-profile pe |

---

## 3. Visiting-Card ke "states" — supplier/internal-team ke liye iska matlab

| Status | Business meaning |
|---|---|
| **Pending (`P`) ya khaali** | Card upload ho chuka hai, approval-decision abhi pending hai. Yeh state hi ek internal-review queue mein jaati hai. |
| **Approved (`Y`)** | Final approval mil gaya — card publicly displayable/trusted-signal ban gaya |
| **Rejected (`R`)** | Card reject kiya gaya (jaise unclear photo, galat details, etc.) |
| **No Visiting Card (`N` in companion status)** | Supplier ke paas card hi nahi hai, explicit reported |
| **"Has visiting card" = Yes (`Y` in companion status)** | Har successful card-submission (chahe abhi pending ho) is companion-flag ko automatically "Yes" mark kar deta hai |

**Business rule jo yaad rakhne layak hai**: Ek naya photo upload karna, ya design-update
bhejna, **purane enrich/design/approval-progress ko reset kar deta hai** — matlab review-cycle
naye sire se shuru hota hai. Yeh intentional hai (nayi photo ka fresh-review zaroori hai), par
supplier/internal-team ko yeh samjhana zaroori hai taaki "mera approval kahan gaya" jaisa
confusion na ho.

---

## 4. Main Business Flows (step-by-step, no code)

### Flow A — Field-team/supplier naya visiting-card capture karta hai

```
1. Field-team ya supplier front-side aur/ya back-side ki photo upload karta hai
   (dono ek saath, ya ek-ek karke)
2. Agar yeh pehli baar hai (naya record), system location (lat/long) bhi capture karta hai
   jahan se photo li gayi
3. Naya record ban jaata hai, aur "has visiting card" status automatically "Yes" set ho jaata hai
4. Background mein: card ek internal approval-tracking system mein sync ho jaata hai
   (agar approval-status abhi terminal nahi hai, i.e. pending/khaali)
```

**Business impact**: Supplier ka profile ab ek physical-proof ke saath enrich hota hai —
internal WebERP-team ke pass ab yeh review-queue mein aa jaata hai.

### Flow B — Supplier apna existing card update karta hai (naya photo)

```
1. Supplier/field-team ek existing card ID pe naya front/back photo (ya design-update) bhejta hai
2. System purana photo overwrite kar deta hai
3. Purana enrich/design/approval-progress automatically reset ho jaata hai (khaali/NULL)
4. "has visiting card" status "Yes" hi rehta hai
5. Background approval-system dobara notify hota hai (agar status terminal nahi hai)
```

**Business impact**: Naya photo dena review-cycle ko refresh karta hai — yeh isliye zaroori
hai kyunki purane approval ka matlab naya photo pe apply nahi hota (naya content, naya
review chahiye).

### Flow C — WebERP internal-team review karta hai (3 sub-flows)

```
1. WebERP team member card ko dekhta hai
2. "Enrich" — content details review/extract karta hai (kaunsi jaankari card pe hai)
3. "Design" — photo/scan ki quality approve karta hai (readable hai, sahi orientation mein hai, etc.)
4. "Approve" — final decision deta hai: Approved ya Rejected
   (ek combined "enrich + design ek saath" option bhi available hai)
```

**Business impact**: Yeh multi-stage quality-check hai — sirf ek insaan "yes/no" nahi bolta,
content aur design dono independently verify hote hain before final-approval.

### Flow D — GLADMIN comment add karta hai

```
1. Internal-admin (GLADMIN) ek existing card pe comment/annotation add karta hai
   (jaise "photo blurry hai, dobara maango")
2. Comment record mein save ho jaata hai
```

**Business impact**: Internal communication-trail — reviewers/field-team ke beech context
share karne ka tareeka, bina alag ticketing-system ke.

### Flow E — Supplier "mere paas visiting card nahi hai" report karta hai (NO_VC)

```
1. Supplier/field-team explicitly bata deta hai ki visiting-card available nahi hai
2. System "No Visiting Card" status record kar leta hai — koi card-record nahi banta
```

**Business impact**: System ko clearly pata chalta hai "not-yet-uploaded" vs "doesn't exist"
mein farak — reporting/follow-up dono cases mein alag hona chahiye.

### Flow F — Approval-tracking system background mein sync karta hai

```
1. Jab bhi card pending/khaali-status mein update hota hai, ek background-event fire hota hai
2. Yeh event ek alag internal database mein 3 cheezein karta hai:
   a. Supplier ka paid-status, work-order-count, aur enterprise-type nikalta hai
      (taaki reviewer ko priority-context mile — jaise paid/enterprise customer ka
      card pehle review ho)
   b. Iss card ko current "pending-approval queue" mein daal deta hai (purani entry
      hataake, taaki queue mein hamesha latest snapshot rahe)
   c. Ek permanent audit-trail entry bhi banata hai (kisne kab kya kiya)
```

**Business impact**: Yeh WebERP team ko ek prioritized, context-rich review-queue deta hai —
sirf "yeh card review karna hai" nahi, balki "yeh card ek paying/enterprise customer ka hai,
unke kitne work-orders chal rahe hain" jaisi info bhi saath mein.

### Flow G — Buyer/koi bhi caller visiting-card display dekhta hai

```
1. Caller supplier ka GLID deke visiting-card display maangta hai
2. System supplier ka current "has visiting card" status, aur uske saare
   approval-states (pending/approved/rejected) ke latest cards return karta hai
3. Attachment-images ke direct-viewable links diye jaate hain
```

**Business impact**: Buyer/internal-tools ko ek consolidated view milta hai — na sirf "kya
approved hai" balki poora context (agar multiple states mein cards hain, jaise ek approved aur
ek naya-pending saath mein).

---

## 5. Business Rules — Plain Language Mein

1. **Insert vs Update decide hota hai `VISITING_CARD_ID` ki presence se** — naya ID nahi
   diya, naya record banta hai; ID diya toh existing record update hota hai.
2. **Naye-record ke liye front ya back attachment (ya design-update) mein se kam-se-kam ek
   zaroori hai** — sab khali submit nahi ho sakta.
3. **`TYPE=NO_VC` ek pura alag path hai** — "supplier ke paas visiting-card nahi hai" ko
   record karta hai, alag hi jagah (companion "no visiting card" table) mein jaata hai, mool
   card-table ko touch nahi karta.
4. **WebERP ke andar 4 alag sub-workflows hain**: ENRICH, APPROVE, DESIGN, ENRICHDESIGN
   (design+enrich combined ek saath) — kaunsa hoga yeh request mein diya gaya flag decide
   karta hai.
5. **Naya attachment/design-update upload hone par, purane approval/design/enrich-flags
   reset ho jaate hain** — matlab naya photo dena dobara-review-cycle automatically trigger
   karta hai, chahe purana approval already mil chuka ho.
6. **"Has visiting card" status ke through automatic-sync hoti hai**: har successful update
   iss status ko "Yes" set kar deta hai — sirf ek specific case mein (jab card reject ho AUR
   ek internal-count-condition match ho) yeh "No" mein flip hota hai.
7. **Sirf pending/khaali-status update hi background approval-system ko notify karta hai** —
   ek baar card definitively-approved ya definitively-rejected ho jaaye, uss specific update
   ke liye background-sync trigger nahi hota (kyunki us point pe review-cycle already
   complete maana jaata hai).
8. **Gateway se sirf specific internal-systems hi is API ko call kar sakte hain** —
   GLADMIN, mobile-app (MY), WebERP, website (M.INDIAMART.COM), aur kuch aur internal
   channels — koi bhi random external caller nahi.
9. **Ek hi supplier ke multiple visiting-cards alag-alag approval-states mein simultaneously
   dikh sakte hain** — jaise ek purana approved card aur ek naya-abhi-submitted pending
   card, dono saath mein display ho sakte hain jab tak naya card bhi resolve na ho jaaye.

---

## 6. Notifications — Kab pata chalta hai

| Kab | Kya hota hai |
|---|---|
| Card pending/khaali-status mein successfully update hota hai | Background approval-tracking system automatically sync/notify hota hai (internal, supplier ko visible email nahi) |
| Card definitively approved/rejected update hota hai | Background approval-sync **fire nahi hota** iss specific write pe (§5.7) — koi supplier-facing email notification bhi is review mein trace nahi hui |
| WebERP admin-comment add karta hai | Comment record mein save hota hai, koi separate notification trace nahi hui |

> **Note**: GST feature ki tarah explicit "email gaya" wording iss doc mein claim nahi ki
> gayi hai kyunki code mein visiting-card ke liye koi email-triggering function nahi mila —
> agar business expectation hai ki supplier ko email jaana chahiye approval/rejection pe,
> yeh confirm karo team se (Open Question).

---

## 7. Edge Cases — Product/Business Perspective se

1. **"Maine naya photo upload kiya, purana approval-status gayab ho gaya"** — Expected hai,
   naya attachment upload karne se review-cycle reset ho jaata hai. Yeh bug nahi hai.
2. **"Maine card upload kiya, phir bhi NO_VC status dikh raha"** — Ho sakta hai approval
   abhi pending ho, aur "has visiting card" status alag cheez hai approval-status se —
   dono independently track hote hain.
3. **Do "companion" concepts hain jo confuse ho sakte hain**: ek jagah asli visiting-card
   data (photo, enrich/design/approval-status) store hota hai, ek alag jagah sirf "has
   visiting card ya nahi" ka simple Yes/No/khaali status store hota hai. Zyada-tar successful
   updates dono ko automatically sync rakhte hain, lekin logic thoda nuanced hai (§5.6).
4. **Ek supplier ke multiple cards ek saath dikh sakte hain** (§5.9) — agar buyer/internal-tool
   ek single "current card" expect kar raha hai, unhe pata hona chahiye ki response mein
   multiple approval-state-wise entries aa sakti hain.
5. **Approval-tracking priority-context (paid/work-order/enterprise-type) real-time compute
   hota hai har event pe** — agar yeh underlying data (jaise paid-status) kabhi stale ho
   ya lookup fail ho, review-queue mein galat/incomplete priority-context dikh sakta hai us
   particular event ke liye (system phir bhi retry karta hai).
6. **Definitively-approved/rejected updates background-system ko notify nahi karte** — agar
   koi downstream team assume kar rahi hai ki har status-change unhe pata chalta hai, yeh
   confirm karna zaroori hai ki sirf pending-transitions hi track hoti hain.

---

## 8. Quick Summary — Ek Line Mein Har Cheez

- Visiting Card = supplier ke business-card ka photo (front/back), multi-stage internal-review
  workflow (capture → enrich → design → approve) ke saath.
- NO_VC status alag se explicitly track hoti hai — "nahi hai" vs "abhi upload nahi hua" mein
  farak.
- 3 internal-actor-types (field/GLADMIN/WebERP), har ek apna step perform karta hai.
- Naya photo upload = automatic review-cycle-reset.
- Ek dedicated background-system approval-events ko ek alag internal-database mein track
  karta hai, saath mein supplier ka paid/work-order/enterprise-context bhi attach karke —
  taaki WebERP-team priority se kaam kar sake.
- Sirf pending/khaali-status transitions hi background-system ko trigger karte hain, terminal
  (approved/rejected) states nahi.

---

## See also

- [`Visiting_Card_Technical_Doc.md`](./Visiting_Card_Technical_Doc.md) — same flows, code-level
  detail (APIs, DB tables, queries, RabbitMQ, approval-consumer)
- [`../GST KT/GST_Business_Doc.md`](../GST%20KT/GST_Business_Doc.md) — dusra trust/verification
  feature, comparable multi-stage-review precedent

# ID Proof — Business Doc (Product Perspective)

Yeh doc ID Proof feature ko **business/product nazariye** se samjhata hai — koi code
nahi, sirf "kya hota hai, kyun hota hai, aur supplier/business ke liye iska matlab kya
hai." Technical implementation (APIs, DB, validation, code) ke liye
[`IDProof_Technical_Doc.md`](./IDProof_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Supplier apni **identity-proof details** — kisi government-issued ID ka number, aur uske
supporting-document ka reference (URL/image) — IndiaMART profile ke saath submit kar
sakta hai. Yeh supplier ki **KYC (Know-Your-Customer) jaisi verification** ka ek chhota,
focused hissa hai:

1. **Identity confirm karna**: supplier khud kaun hai, yeh verify karne ka ek tareeka —
   sirf company-level GST/PAN se alag, yeh individual/proof-document level ka data hai.
2. **Trust & compliance ka ek aur layer**: GST company ka registration prove karta hai,
   ID Proof us insaan/entity ka jo account chala raha hai uska proof rakhta hai.
3. **Internal review ke liye base data**: number + document-reference store hone ke baad,
   koi internal team/process usse review karke "approved" mark kar sakta hai.

**Important distinction**: yeh feature IndiaMART ke bade **Trust Verification /
TrustSeal** system se related hai lekin usse alag hai — ID Proof sirf ek chhota,
single-table record hai (number + document-reference + ek simple approved/pending flag),
uss bade multi-status verification-workflow jitna complex nahi hai. Dono ko confuse mat
karo — bada Trust Verification system alag hai (dekho
[`../Trust Verification KT/Trust_Verification_Business_Doc.md`](../Trust%20Verification%20KT/Trust_Verification_Business_Doc.md)).

**Bottom line**: ID Proof = "supplier ka koi ek identity-document number + uska
proof-image/link, aur woh verified hai ya nahi" — ek chhota lekin trust-building data
point.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Apna ID-proof number aur document-reference (URL/image) app se submit karta hai |
| **Internal admin (GLADMIN)** | Records manage/submit kar sakta hai, potentially approval-status set karne ke liye bhi (code se yeh conclusively confirm nahi hua — dekho neeche "Open Question") |
| **MAPI (internal middleware/app-API layer)** | Ek aur allowed writer-system hai |
| **Android / iOS app** | Supplier apni app se seedha yeh data submit karta hai |
| **Buyer/koi bhi internal caller** | Read-side se kisi supplier ke against ID-proof record query kar sakta hai (agar valid token/session ho) |

**Koi background team/process is doc mein trace nahi hua jo approval-status ko actively
review karke change karta ho** — yeh ek Open Question hai (section 7).

---

## 3. Approval Status — supplier ke liye iska matlab

ID Proof ka status GST jaisa multi-level nahi hai — yeh sirf ek **simple binary flag**
hai:

| Status | Business meaning |
|---|---|
| **0 (default/pending)** | Naya submit hua hai, abhi review/approve nahi hua |
| **1 (approved)** | Kisi ne isse approve kar diya hai |

**Important**: code mein koi explicit "rejected" status nahi milta — sirf `0` (pending)
aur `1` (approved) valid values hain jab likhte waqt. Agar business process mein
"rejection" ka concept hai, toh woh ya toh record delete karke handle hota hoga, ya kisi
aur jagah track hota hoga jo iss review mein nahi mila — **confirm karo team se**
(dekho section 7, Open Question).

---

## 4. Main Business Flows (step-by-step, no code)

### Flow A — Supplier naya ID-proof submit karta hai

```
1. Supplier app pe jaake ek proof-type select karta hai (jaise koi specific
   ID-document-category) aur apna ID-number + document-reference (URL/image) daalta hai
2. System check karta hai: is supplier ke is proof-type ke liye pehle se record hai
   ya nahi
3. Agar nahi hai -> naya record ban jaata hai, approval-status default "pending" set hota
   hai
4. Supplier apne profile mein turant reflect hota dekh sakta hai (naya submitted record)
```

**Business impact**: Supplier ka profile mein ek naya identity-proof record add ho gaya,
abhi review pending hai.

### Flow B — Supplier apna already-submitted ID-proof update karta hai

```
1. Supplier same proof-type ke against naya number/document-reference daalta hai
2. System dekhta hai — is proof-type ke liye already record hai -> UPDATE treat hota hai
   (naya record nahi banta)
3. Jo fields nahi bheji gayi (jaise sirf number update kiya, URL nahi), unki purani
   value as-it-is retain hoti hai — partial update hota hai
4. Agar approval-status explicitly nahi bheja gaya, purana status bhi retain hota hai
   (reset nahi hota)
```

**Business impact**: Supplier apna document dobara/sahi karke daal sakta hai bina naya
duplicate record banaye. Iska matlab yeh bhi hai ki agar supplier sirf number galat
tha aur usse fix kar raha hai, uska pehle se-approved status accidentally reset nahi
hoga (jab tak explicitly na bheja jaaye).

### Flow C — Koi caller ID-proof record read karta hai

```
1. Caller (internal system/app) ek supplier ID aur proof-type diyta hai
2. System valid token/session check karta hai
3. Agar record milta hai -> number, document-reference, aur approval-status wapas
   milta hai
4. Agar record nahi milta -> "No data found" wapas milta hai
```

**Business impact**: Yeh data downstream systems (jaise trust-badge display, internal
review dashboards) ko is supplier ke identity-verification-state ke baare mein batata
hai.

---

## 5. Business Rules — Plain Language Mein

1. **Naya record ya update — dono ek hi API se handle hote hain**, system khud decide
   karta hai based on existing-record ki presence (per supplier + per proof-type).
2. **Update partial ho sakta hai** — agar koi field nahi bheji gayi, purani value hi
   retain hoti hai (number, URL, approval-status sab individually optional hote hain
   update ke waqt).
3. **Approval-status default "pending" (0) hota hai** jab tak explicitly diya na jaaye.
4. **Sirf allowed writers hi likh sakte hain** — GLADMIN, MAPI, Android, IOS. Agar koi
   inn systems se validation-key na bhi de, toh bhi request theoretically aage badh
   sakta hai (yeh ek technical nuance hai, dekho Technical Doc section 5.2/13.3) —
   business ko sirf itna samajhna chahiye ki yeh check present hai lekin bypass-able
   scenario bhi hai jo team ko flag karna chahiye.
5. **Approval-status ke liye sirf do valid values hain — pending ya approved.** Koi
   third "rejected" value system explicitly accept nahi karta.
6. **Ek supplier ke multiple proof-types ho sakte hain** — jaise ID-proof aur
   address-proof alag categories, har ek apna independent record aur apna independent
   approval-status rakhta hai. Iska matlab, ek proof-type approve ho jaaye toh dusra
   proof-type automatically approve nahi hota.
7. **Kaun sa proof-type valid hai, iski koi master-list is review mein nahi mili** —
   yaani product/business team ko yeh confirm karna chahiye ki proof-type IDs ka
   documentation kahan maintain hota hai (app-side? kisi aur config mein?), kyunki
   backend khud isse validate nahi karta.

---

## 6. Notifications

**Koi supplier-facing notification (email/SMS/push) is feature ke liye code mein nahi
mila** — na submit hone par, na approve hone par. Yeh GST feature se ek bada difference
hai, jahan email confirmation explicitly bhejta hai. Agar business expectation hai ki
supplier ko batana chahiye jab unka ID-proof approve ho, **yeh currently implemented
nahi hai (confirm karo team se)**.

| Kab | Kya notification jaata hai |
|---|---|
| Naya ID-proof submit hua | Koi notification nahi mila code mein |
| ID-proof update hua | Koi notification nahi mila code mein |
| Approval-status change hua | Koi notification nahi mila code mein |

---

## 7. Edge Cases — Product/Business Perspective se

1. **"Maine ID-proof update kiya, approval-status reset ho gaya?"** — Nahi, agar
   approval-status explicitly nahi bheja gaya, purana status retain hota hai — sirf
   diye-gaye fields hi change hote hain.
2. **"Maine same proof-type dobara submit kiya"** — Yeh update treat hoga, naya record
   nahi banega (per user+proof-type ek hi record maintain hota hai).
3. **"Mera ID-proof kabhi approve nahi hota"** — Iss review mein koi automated ya
   dedicated internal-review-flow trace nahi hua jo approval-status ko `0` se `1` pe le
   jaata ho. Ho sakta hai yeh sirf manual GLADMIN action se hota ho, jiska exact
   process/team confirm karna chahiye (Open Question).
4. **"Kya mera ID-proof reject bhi ho sakta hai?"** — System-level pe koi explicit
   "rejected" status accept nahi hota (sirf pending/approved). Agar business process
   mein rejection ka concept hai, uska implementation (record delete? kahin aur track
   hota hai?) confirm karna chahiye.

---

## 8. Quick Summary

- ID Proof = supplier ke ek identity-verification-document ki details (number +
  document-reference + simple pending/approved status), per proof-type.
- Insert-or-update ek hi API handle karta hai (system auto-detects), aur update
  partial-update support karta hai (jo nahi bheja gaya woh retain hota hai).
- Android/iOS/MAPI/GLADMIN allowed writers hain.
- Status sirf do values leta hai — pending (default) ya approved; koi explicit
  "rejected" state nahi.
- Koi email/SMS/push notification nahi jaata is feature ke through.
- Koi background job/consumer/cron involved nahi hai — poori feature synchronous hai,
  ek request aata hai, ek response jaata hai.
- Approval-status kaun/kaise set karta hai, yeh conclusively pata nahi chala — team se
  confirm karna zaroori hai.

---

## See also

- [`IDProof_Technical_Doc.md`](./IDProof_Technical_Doc.md) — same flows, code-level
  detail (API, DB table, validation rules, exact status values)
- [`../Trust Verification KT/Trust_Verification_Business_Doc.md`](../Trust%20Verification%20KT/Trust_Verification_Business_Doc.md) —
  related but separate, much bigger verification-workflow concept
- [`../GST KT/GST_Business_Doc.md`](../GST%20KT/GST_Business_Doc.md) — ek aur trust/
  compliance-related feature, jahan verification-status kaafi zyada rich hai (multiple
  levels, daily re-verification job) — ID Proof se contrast ke liye useful

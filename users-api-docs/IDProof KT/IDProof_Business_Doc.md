# ID Proof — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`IDProof_Technical_Doc.md`](./IDProof_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Supplier apni **identity-proof details** (jaise koi government-ID ka number aur uska
reference-document/image URL) IndiaMART profile ke saath submit kar sakta hai. Yeh
supplier ki **KYC (Know-Your-Customer) jaisi verification** ka hissa hai — Trust
Verification/TrustSeal family se related hai, lekin ek alag, focused concept: sirf
identity-proof-number + document-reference + approval-status.

**Business impact**: Identity-verification IndiaMART platform ki trust/compliance
requirements ka core hissa hai — supplier ki legitimacy confirm karne ke liye kis type
ka ID-proof diya gaya, uska number, aur usko internal-team ne approve/reject kiya ya
nahi, yeh sab track hota hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | ID-proof number + document-reference submit karta hai (Android/iOS app se) |
| **Internal admin (GLADMIN)** | ID-proof records manage/verify kar sakta hai |
| **Internal-verification-process** | Approval-status set karta hai (approved/pending/rejected) |

---

## 3. Business Flow (step-by-step, no code)

```
1. Supplier ek specific proof-type (jaise koi ID-document-category) select karke apna
   ID-proof-number aur document-reference (URL/image) submit karta hai
2. System check karta hai — is supplier ke is proof-type ke liye pehle se record hai
   ya nahi
   - Agar nahi, naya record insert hota hai (Approval-status default "0"/pending)
   - Agar hai, existing record update hota hai
3. Internal-verification-process is proof ko review karke approval-status set kar sakta
   hai
4. Koi bhi caller supplier ke against, ek specific proof-type ka record query kar sakta
   hai
```

**Business impact**: Ek per-proof-type record-model hai — ek supplier ke multiple
proof-types ho sakte hain (jaise ID-proof aur address-proof alag categories), har ek
apna independent record rakhta hai.

---

## 4. Business Rules — Plain Language Mein

1. **Naya record ya update — dono ek hi API se handle hote hain**, system khud decide
   karta hai based on existing-record ki presence.
2. **Update partial ho sakta hai** — agar koi field nahi bheji gayi, purani value hi
   retain hoti hai (URL, number, approval-status sab individually optional hain update
   ke waqt).
3. **Approval-status default "0" (pending) hota hai** jab tak explicitly diya na jaaye.
4. **Sirf allowed apps/systems hi likh sakte hain** — GLADMIN, MAPI, Android, iOS.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine ID-proof update kiya, approval-status reset ho gaya?"** — Nahi, agar
   `APPROVAL_STATUS` explicitly nahi bheja gaya, purana status retain hota hai —
   sirf diye-gaye fields hi change hote hain.
2. **"Maine same proof-type dobara submit kiya"** — Yeh update treat hoga, naya record
   nahi banega (per user+proof-type ek hi record maintain hota hai).

---

## 6. Quick Summary

- ID Proof = supplier ke identity-verification-document details (number + reference +
  approval-status), per proof-type.
- Insert-or-update (system auto-detects), partial-update supported on update.
- Android/iOS/MAPI/GLADMIN allowed writers.
- Koi RabbitMQ/Kafka/consumer nahi — purely synchronous.

---

## See also

- [`IDProof_Technical_Doc.md`](./IDProof_Technical_Doc.md) — code-level detail
- [`../Trust Verification KT/Trust_Verification_Business_Doc.md`](../Trust%20Verification%20KT/Trust_Verification_Business_Doc.md) —
  related but separate verification-workflow concept

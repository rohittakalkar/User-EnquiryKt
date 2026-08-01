# Supplier Verification Log — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Supp_Verify_Log_Technical_Doc.md`](./Supp_Verify_Log_Technical_Doc.md) dekho.

**Note**: Yeh feature [`../SellOnIM Log KT/`](../SellOnIM%20Log%20KT/) se related hai —
SellOnIM Log ka calling-eligibility-pipeline `iil_supp_verification` (queue) mein entries
daalta hai; yeh KT us verification-process ke **outcome-log** (`IIL_SUPP_VERIFICATION_LOG`)
ko cover karta hai — jab koi supplier actually call/verify ho chuka ho, uska result yahan
record hota hai. Do alag tables, related-but-sequential concept.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Jab internal-calling/verification-team kisi supplier ko verify karti hai (phone-call,
document-check, etc.), uska **poora outcome ek log-entry ke roop mein record** hota hai —
call ka result (`VERIFY_FLAG`), kaunsa disposition/reason mila, kitni baar call hui
(`CALL_COUNTER`), voice-recording ka link, aur attachments. Yeh IndiaMART ke supplier-
verification/trust-pipeline ka core outcome-tracking-mechanism hai.

**Business impact**: Yeh internal-teams ko har supplier ki verification-history dikhata
hai — kab verify hua, kisne kiya, kya outcome mila — jo trust/compliance-decisions
(jaise TrustSeal-eligibility) ke liye foundational data hai. Yeh cross-checked bhi hota
hai — many automated processes (naye-signup detection, PNS-defaulter-rejection, receipt-
based-verification, country-based-rejection, hot-lead-verification) is table ko
automatically populate karte hain.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Internal calling/verification-team (GLADMIN)** | Verification-call karti hai, outcome record karti hai |
| **Automated batch-process (nightly cron)** | Multiple business-rules ke through automatically naye verification-log-entries banata hai |
| **Supplier** | Verification ka subject |

---

## 3. Business Flow (step-by-step, no code)

```
1. Ek supplier verification ke liye "queue" mein aata hai (jaise SellOnIM-pipeline se,
   ya kisi automated-detection-rule se)
2. Internal-team supplier ko call karti hai, ya koi automated-process ek verification-
   decision leta hai
3. Outcome (verify-flag, disposition, call-counter, voice-log-URL, attachments) ek naye
   log-entry ke roop mein save hota hai
4. Baad mein, agar dobara-verify karna pade ya update karna ho, existing log-entry
   update ho sakta hai
5. Kabhi-kabhi entries delete bhi ho sakti hain — is case mein, ek "history-package"
   call bhi hoti hai jo is deletion ko audit-trail mein record karti hai
6. Ek nightly automated-process (cron) bhi hai jo kayi alag-alag business-signals
   (naye-signup, PNS-defaulters, receipt-verification, country-based-rejection,
   hot-leads, C2C-verification) ke basis pe naye verification-entries khud banata hai
```

**Business impact**: Yeh sirf manual-calling ka log nahi hai — ek hybrid system hai
jahan manual-verification aur kayi automated-business-rules dono is ek hi table mein
apna data likhte hain.

---

## 4. Business Rules — Plain Language Mein

1. **Teen actions supported hain**: INSERT (naya verification-log), UPDATE (existing
   ko modify), DELETE (remove karna, saath mein ek audit-trail-entry bhi banti hai).
2. **DELETE hamesha ek history/audit-package-call trigger karta hai** — matlab delete
   silently nahi hota, kahin aur bhi "yeh delete hua" record hota hai.
3. **`MULTIPLE_ATTACHMENTS` max 5 values tak allow hota hai**, comma-separated list ke
   roop mein.
4. **Sirf GLADMIN allowed hai write ke liye** — purely internal-tool feature.
5. **Ek automated-nightly-process (cron) bhi hai** jo independently naye verification-
   entries populate karta hai, kayi business-signals ke basis pe (na sirf manual-calling
   se).

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine ek verification-log delete kiya, phir bhi kahin trace mil raha hai"** —
   Expected hai, delete ek audit-history-package call bhi trigger karta hai — poori
   traceability maintain rehti hai.
2. **"Meri verification-entry automatically ban gayi, maine call nahi ki"** — Expected
   ho sakta hai agar koi automated-business-rule (nightly cron) ne yeh entry banayi ho —
   sab entries manual-calling se nahi aatin.

---

## 6. Quick Summary

- Supplier Verification Log = call/verification-outcome-tracking (manual + automated).
- Insert/Update/Delete (delete audit-trailed).
- GLADMIN-only write; ek nightly cron bhi hai jo independently entries populate karta hai
  multiple business-rules se.
- SellOnIM Log se related-but-separate — sequential concept (queue → outcome-log).

---

## See also

- [`Supp_Verify_Log_Technical_Doc.md`](./Supp_Verify_Log_Technical_Doc.md) — code-level detail
- [`../SellOnIM Log KT/SellOnIM_Log_Business_Doc.md`](../SellOnIM%20Log%20KT/SellOnIM_Log_Business_Doc.md) —
  upstream calling-queue that feeds into this verification-process

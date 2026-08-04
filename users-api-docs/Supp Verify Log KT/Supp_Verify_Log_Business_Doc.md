# Supplier Verification Log — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai — koi code nahi, sirf "kya hota hai, kyun
hota hai, aur supplier/internal-team ke liye iska matlab kya hai." Code ke liye
[`Supp_Verify_Log_Technical_Doc.md`](./Supp_Verify_Log_Technical_Doc.md) dekho.

**Note**: Yeh feature [`../SellOnIM Log KT/`](../SellOnIM%20Log%20KT/) se related hai —
SellOnIM Log ka calling-eligibility-pipeline `iil_supp_verification` (queue) mein entries
daalta hai; yeh KT us verification-process ke **outcome-log** (`IIL_SUPP_VERIFICATION_LOG`)
ko cover karta hai — jab koi supplier actually call/verify ho chuka ho ya kisi automated-rule
se process ho chuka ho, uska result yahan record hota hai. Do alag tables, related-but-
sequential concept — koi direct database-link (foreign key) inke beech nahi mila code mein.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Jab internal-calling/verification-team kisi supplier ko verify karti hai (phone-call,
document-check, etc.), uska **poora outcome ek log-entry ke roop mein record** hota hai —
call ka result (`VERIFY_FLAG`), kaunsa disposition/reason mila, kitni baar call hui
(`CALL_COUNTER`), voice-recording ka link, aur attachments. Yeh IndiaMART ke supplier-
verification/trust-pipeline ka core outcome-tracking-mechanism hai.

Lekin yeh sirf manual-calling ka log nahi hai. Isme ek **automated nightly batch-process
(cron)** bhi hai jo apne aap, bina kisi insaan ke, kayi business-signals ke basis pe naye
verification-entries bana sakta hai — jaise "yeh supplier disabled ho gaya," "yeh supplier
PNS-defaulter nikla," ya "isne recently payment ki." Is doc ka ek important finding yeh hai
ki is automated-process mein **9 alag-alag business-rules code mein likhe hain, lekin isme se
sirf 4 hi actually chal rahe hain** — baaki 5 ya toh switch-off kar diye gaye hain, ya kabhi
activate hi nahi hue. Iska business-impact section 5 mein detail se hai.

**Business impact**: Yeh internal-teams ko har supplier ki verification-history dikhata
hai — kab verify hua, kisne kiya, kya outcome mila — jo trust/compliance-decisions (jaise
TrustSeal-eligibility) ke liye foundational data hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Internal calling/verification-team (GLADMIN)** | Verification-call karti hai, outcome record karti hai (INSERT/UPDATE/DELETE) |
| **Automated nightly batch-process (cron)** | Kuch business-rules ke through automatically naye verification-log-entries banata hai — abhi sirf 4 rules active hain, 5 dormant/dead hain |
| **Supplier** | Verification ka subject — usko is process ka koi direct notification nahi jaata (section 6) |
| **Internal on-call/monitoring team** | Cron ka daily summary-report Google Chat pe milta hai (har run pe, success ho ya failure) |

---

## 3. Main Business Flows (step-by-step, no code)

### Flow A — Internal team ek supplier ko manually verify karti hai

```
1. Ek supplier verification ke liye "queue" mein aata hai (jaise SellOnIM-pipeline se,
   ya kisi automated-detection-rule se)
2. Internal-team supplier ko call karti hai
3. Outcome (verify-flag, disposition, call-counter, voice-log-URL, attachments — max 5)
   ek naye log-entry ke roop mein save hota hai (INSERT)
4. Baad mein, agar dobara-verify karna pade ya kuch update karna ho (jaise call-count badhana,
   naya disposition set karna), existing log-entry update ho sakta hai (UPDATE)
5. Kabhi-kabhi entries delete bhi ho sakti hain (DELETE) — is case mein, ek "history-package"
   call bhi hoti hai jo is deletion ko audit-trail mein record karti hai, taaki koi entry
   silently gaayab na ho jaaye
```

**Business impact**: Yeh manual-verification ka poora record maintain karta hai — kaun-si
call hui, kya result mila, kisne process kiya — jo baad mein audit/compliance ke liye kaam
aata hai.

### Flow B — Nightly automated-process apne aap verification-entries banata hai

```
1. Har raat (ek external scheduler se trigger hokar), ek automated-batch-process chalta hai
2. Yeh process 4 active business-rules follow karta hai:
   a. "Disabled ho gaye" suppliers ko dhoondhta hai (jinka catalog-service pichle 1 din mein
      band hua) → automatically "Verified" mark karta hai
   b. "PNS-defaulter" suppliers ko dhoondhta hai (jinka catalog-service 3-10 din pehle khatam
      hua) → agar unhone payment kiya hai to "Verified," nahi kiya to "Rejected (PNS
      Defaulter)" mark karta hai
   c. Jin suppliers ne recently (pichle 60 din mein) actual payment-receipt di hai, unhe
      calling-queue se hata deta hai (kyunki unhe ab dobara call karne ki zaroorat nahi)
   d. Kuch specific countries (fraud-prone list) se aane wale supplier-entries ko calling-
      queue se automatically reject/remove kar deta hai
3. Process khatam hone pe, ek summary-report (kitne entries kis rule se bane/hatay) Google
   Chat pe internal-team ko bhej diya jaata hai — chahe process successful raha ho ya kisi
   step mein error aaya ho
```

**Business impact**: Yeh manual-calling-team ka load kam karta hai kayi common cases mein —
disabled/paid/rejected suppliers ko automatically resolve kar deta hai, taaki team sirf genuine
ambiguous cases pe focus kare.

### Flow C — Code mein likhe hue extra rules jo abhi chal nahi rahe

```
1. Code mein 5 aur business-rules likhe hue hain: "free-tier verification ko purana hone pe
   downgrade karna," "click-to-call activity se verify karna," "hot-lead activity se verify
   karna," "receipt-based data-copy," aur "QGFCP/VGFCP tagged entries banana"
2. Inme se ek (free-verified downgrade) poori tarah code mein ready hai, bas usko chalane
   wali call switch-off kar di gayi hai
3. Baaki 4 ka poora logic bhi comment kar diya gaya hai — matlab agar koi unhe accidentally
   bhi chala de, kuch nahi hoga, kyunki andar ka code hi nikal diya gaya hai (comments ke
   roop mein reh gaya hai)
```

**Business impact**: Yeh important hai janna kyunki agar koi poochhe "kya hot-lead suppliers
automatically verify hote hain," ya "kya humara free-verified downgrade abhi bhi kaam kar
raha hai," code padhne se pehli nazar mein "haan" lagega — lekin actual answer hai **nahi,
yeh sab abhi off hain**. Business/product team ko yeh confirm karna chahiye ki yeh
intentional pause hai ya ek forgotten-to-re-enable situation (dekho Open Questions,
Technical Doc section 14).

---

## 4. Kaun-si entries kaise bani — ek quick reference

| Verification-outcome | Kaise bana | Business meaning |
|---|---|---|
| "Verified — Paid Catalog To FCP" | Manual call, YA automated "disabled-supplier" rule, YA automated "PNS paid" rule | Supplier ka catalog-service paid tha, verified maana gaya |
| "Verification Rejected — Catalog PNS Defaulter" | Automated "PNS-defaulter" rule | Supplier ne payment nahi ki, defaulter list mein hai |
| (Free-verified downgrade) | Currently **nahi ban raha** — yeh rule off hai (Flow C) | N/A abhi — agar future mein on ho, purani "verified" entries jo 330+ din se stale hain unhe downgrade karega |
| Manual-team ka koi bhi custom outcome | Direct manual INSERT/UPDATE via internal tool | Jo bhi disposition/voice-log/attachment team ne record kiya |

---

## 5. Business Rules — Plain Language Mein

1. **Teen actions supported hain manual-write mein**: INSERT (naya verification-log), UPDATE
   (existing ko modify), DELETE (remove karna, saath mein ek audit-trail-entry bhi banti hai).
2. **DELETE hamesha ek history/audit-package-call trigger karta hai** — matlab delete silently
   nahi hota, kahin aur bhi "yeh delete hua" record hota hai, chahe delete se koi row actually
   affected hui ho ya nahi.
3. **`MULTIPLE_ATTACHMENTS` max 5 values tak allow hota hai**, comma-separated list ke roop
   mein.
4. **Sirf GLADMIN allowed hai manual-write ke liye** — purely internal-tool feature, supplier
   khud kabhi is table mein direct nahi likhta.
5. **UPDATE karte waqt supplier ka ID double-check nahi hota**, sirf log-entry ka apna ID
   check hota hai — technically thoda loose validation hai (dekho Technical Doc, Edge Cases
   #3).
6. **Ek automated-nightly-process (cron) bhi hai**, jo independently naye verification-entries
   populate karta hai — lekin isme jo 9 rules code mein hain, unme se abhi sirf 4 hi actually
   chal rahe hain (disabled-user detection, PNS-defaulter check, receipt-based-queue-cleanup,
   country-based-rejection). Baaki 5 (free-verified downgrade, click-to-call, hot-lead,
   data-copy, QGFCP/VGFCP) ya to switched-off hain ya poori tarah inert code hain.
7. **Country-based-rejection ek hardcoded 9-country list use karta hai** (fraud-prone maane
   gaye countries) — koi bhi supplier jiska IP-country is list mein match kare, uski
   calling-queue-entry usi din remove ho jaati hai.
8. **Receipt-based-cleanup sirf calling-queue se entries hatata hai**, verification-log table
   se nahi — matlab agar kisi supplier ne payment kar di, unko future-calling se bahar nikal
   diya jaata hai, lekin unka purana verification-history record touch nahi hota.
9. **PNS-defaulter rule ek specific protected-state (`-3`) ko overwrite nahi karti** — agar
   koi entry already kisi specific manually-set state mein hai, automated-rule usse chhedta
   nahi. Exact business-meaning is protected-state ka code se pata nahi chala — team se
   confirm karna hoga (Technical Doc Open Questions #1).
10. **Ek special case hai jahan INSERT ke liye extra proof (`REQUEST_URL`) mandatory ho jaata
    hai** agar verification-outcome ek specific special-flag ke saath ho — exact business
    scenario code se clear nahi hai, team se confirm karna hoga.

---

## 6. Notifications — Kisko kab pata chalta hai

| Kab | Kya notification jaata hai |
|---|---|
| Manual verification-entry ban/update/delete hui | **Supplier ko koi direct notification nahi jaati** — yeh purely internal record-keeping hai |
| Har nightly cron-run khatam hone pe | Internal-team ko **Google Chat** pe ek summary-report jaata hai (kitne entries kis rule se bane, aaj ke total counts, source-wise breakdown) — success ho ya failure, dono cases mein |
| Cron connection-failure (DB down) | Google Chat pe alert jaata hai, process turant ruk jaata hai |

**Note**: GST feature ke ulat (jahan supplier ko email jaata hai har GST-update pe), yahan
supplier-facing koi notification nahi hai — sab kuch internal-team-facing hai.

---

## 7. Edge Cases — Product/Business Perspective se

1. **"Maine ek verification-log delete kiya, phir bhi kahin trace mil raha hai"** — Expected
   hai, delete ek audit-history-package call bhi trigger karta hai — poori traceability
   maintain rehti hai.

2. **"Meri verification-entry automatically ban gayi, maine call nahi ki"** — Expected ho
   sakta hai agar automated-business-rule (nightly cron) ne yeh entry banayi ho. Lekin
   important: sirf 4 rules abhi active hain (disabled-user, PNS-defaulter, receipt-cleanup,
   country-rejection) — agar koi entry kisi aur reason se bani lagti hai (jaise hot-lead ya
   click-to-call), woh rule abhi off hai, so is source se nayi entries nahi ban sakteen.

3. **"Humne suna tha ki free-verified suppliers 330 din baad automatically downgrade ho jaate
   hain, lekin woh ho nahi raha"** — Correct observation, yeh rule code mein poori tarah ready
   hai lekin currently switched-off hai. Product-team ko decide karna chahiye ki isse
   re-enable karna hai ya permanently hata dena hai.

4. **"Kya hot-lead ya click-to-call activity se koi supplier auto-verify hota hai?"** — Nahi,
   abhi nahi. Yeh dono rules code mein likhe the lekin poori tarah comment-out/dead kar diye
   gaye hain — inhe wapas laane ke liye fresh development-effort lagega, sirf ek switch flip
   karne se nahi hoga (unlike free-verified-downgrade, jiska code abhi bhi intact hai).

5. **Do related-but-separate concepts confuse ho sakte hain**: "calling-queue"
   (`iil_supp_verification`, SellOnIM ka domain) vs. "verification-outcome-log"
   (`IIL_SUPP_VERIFICATION_LOG`, is doc ka focus). Interesting baat yeh hai ki nightly cron
   dono tables ko touch karta hai — kuch rules queue mein likhte/hatate hain, kuch rules log
   mein likhte hain. Ek hi cron-run mein dono ho sakta hai.

---

## 8. Quick Summary

- Supplier Verification Log = call/verification-outcome-tracking (manual + automated).
- Insert/Update/Delete supported (delete audit-trailed); GLADMIN-only write.
- Ek nightly cron hai jo independently entries populate karta hai — **lekin sirf 4 business-
  rules abhi active hain** (disabled-user, PNS-defaulter, receipt-cleanup, country-rejection),
  **5 aur dormant/dead hain** (free-verified-downgrade, click-to-call, hot-lead, data-copy,
  QGFCP/VGFCP) — yeh isi review ka sabse important naya finding hai.
- Koi supplier-facing notification nahi hai; sirf internal-team ko Google Chat pe daily
  summary jaata hai.
- SellOnIM Log se related-but-separate — sequential concept (queue → outcome-log), koi direct
  DB-link nahi.

---

## See also

- [`Supp_Verify_Log_Technical_Doc.md`](./Supp_Verify_Log_Technical_Doc.md) — code-level detail,
  including the full 9-sub-processor cron trace
- [`../SellOnIM Log KT/SellOnIM_Log_Business_Doc.md`](../SellOnIM%20Log%20KT/SellOnIM_Log_Business_Doc.md) —
  upstream calling-queue that feeds into this verification-process

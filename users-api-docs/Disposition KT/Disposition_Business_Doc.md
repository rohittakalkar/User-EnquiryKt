# Disposition — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Disposition_Technical_Doc.md`](./Disposition_Technical_Doc.md) dekho.

**Note**: Disposition aur KWPL genuinely **do alag concepts** hain — koi shared table,
FK-relation, ya common write-controller nahi mila. Dekho
[`../KWPL KT/KWPL_Business_Doc.md`](../KWPL%20KT/KWPL_Business_Doc.md).

**Naming caveat**: "Disposition" word codebase mein aur bhi jagah use hota hai — Supplier
Rating ka `FK_RATING_DISPOSITION_ID` (image-review outcome) aur Supplier-Verification-Log ka
`DISPOSITION_TYPE`/`DISPOSITION_TYPE_ACTIVITY` (`IIL_SUPP_VERIFICATION_LOG` table, ek daily
cron se bhi likha jaata hai) — yeh dono **is doc ka scope nahi hain**, alag tables, alag
business-context. Is doc mein "Disposition" sirf `GLUSR_DISPOSITIONS` / `GL_DISPOSITIONS_MASTER`
wale generic outcome-log feature ko refer karta hai.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

**Disposition** ek "outcome/reason-code log" hai — jab kisi internal-process (jaise
verification, review, ya koi aur attribute-specific evaluation) ka result record karna ho,
uska "disposition" (outcome/status-reason) ek predefined master-list se pick karke supplier
ke against save kiya jaata hai. Yeh ek generic, reusable logging-mechanism hai jo alag-alag
"attributes" (jaise GST) ke against use ho sakta hai — master-list khud multiple attributes
ke liye dispositions define karti hai, na ki sirf GST ke liye.

**Business impact**: Internal-teams ke liye ek audit-trail hai — "kis attribute ke liye,
kya outcome record hua, kab, aur kisne (kaunsa system/MODID) mark kiya" — verification/
review-workflows ki traceability ke liye zaroori hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Internal tools (GLADMIN, SELLERMY, IMOB, MSITE)** | Disposition-record likhte hain, ek predefined master-list se disposition pick karke |
| **Internal tools/dropdowns (koi bhi caller, no special permission)** | Master-list of valid dispositions (aur unke attribute-names) padhte hain, taaki UI mein dropdown dikhaya ja sake |
| **Supplier (indirectly)** | Jiske against yeh disposition record ban raha hai |

---

## 3. Business Flows (step-by-step, no code)

### Flow A — Internal-tool ek disposition record karta hai

```
1. Koi internal-process (jaise GST-verification, ya koi aur attribute-evaluation) apna
   outcome ek "disposition" ke roop mein record karna chahta hai
2. Sabse pehle, tool ek predefined master-list se ek disposition-code pick karta hai
   (yeh master-list bhi khud ek API se fetch ki ja sakti hai — Flow B dekho)
3. System is record ko supplier ke against save karta hai, timestamp ke saath
4. Purana record overwrite nahi hota — har naya disposition apna fresh history-entry banata hai
```

**Business impact**: Ek audit-trail entry ban jaata hai jo supplier ke against permanently
preserved rehta hai — "kab, kaunsa outcome, kis GLID ne" — verification workflows ki
accountability ke liye.

### Flow B — Internal-tool disposition master-list fetch karta hai (dropdown-population)

```
1. Koi internal-tool/admin-screen "Disposition" type ka master-data maangta hai
2. System sirf woh dispositions deta hai jo "enabled" flag ke saath maujood hain
3. Har disposition ke saath uska attribute-display-name bhi milta hai (jaise "GST", ya
   koi aur attribute jiske liye disposition define hui hai)
4. Tool is list ko dropdown mein dikhata hai, jaha se sahi disposition-code select
   karke Flow A trigger hota hai
```

**Business impact**: Yeh confirm karta hai ki master-list **generic hai — sirf GST tak
limited nahi** — kisi bhi attribute ke liye dispositions define ki ja sakti hain, aur
same master-data lookup API (jo aur bhi kai master-lists — Turnover, LegalStatus, Business
type — serve karti hai) inhe expose karti hai.

### Flow C — Baad mein disposition-history query karna

```
1. Baad mein, yeh disposition-history query ki ja sakti hai — ya toh poori history, ya
   sirf latest-entry
2. Abhi ke liye, read-side sirf "GST" attribute ke liye kaam karta hai — baaki
   attributes ke liye "no data" response milta hai, chahe master-list (Flow B) generic ho
```

**Business impact**: Ek generic-design-intent-wala feature hai (kai attributes support
karne ke liye master-list level pe), lekin currently sirf GST-attribute tak practically
readable hai.

---

## 4. Business Rules — Plain Language Mein

1. **Disposition ek master-list se aati hai** — random text nahi, ek predefined
   reason/outcome-code select kiya jaata hai, jo attribute-specific hoti hai.
2. **Master-list "enabled" dispositions tak limited hai** — disabled ho chuki dispositions
   dropdown mein nahi dikhtin.
3. **Naya disposition record hamesha ek fresh insert hota hai** — purana overwrite nahi
   hota, matlab har evaluation ka apna history-entry banta hai.
4. **Read-side abhi sirf GST-attribute support karta hai** — koi aur attribute-name
   diya jaaye, "no data found" milta hai, chahe backend-design (master-list samet) generic ho.
5. **"All history" ya "sirf latest" dono query kiye ja sakte hain** — caller decide
   karta hai.
6. **Disposition record karne ke liye caller ek allowlist mein hona chahiye** (GLADMIN,
   SELLERMY, IMOB, MSITE) — lekin master-list padhna (Flow B) kisi bhi caller ke liye open
   hai, koi allowlist nahi.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine kisi doosre attribute ka disposition query kiya, data nahi mila"** —
   Expected hai abhi, sirf GST supported hai read-side pe, chahe master-list (dropdown)
   mein koi bhi attribute ki dispositions dikh rahi ho.
2. **"Disposition history mein multiple entries hain"** — Expected hai, har naya
   disposition ek fresh row insert karta hai, purane overwrite nahi hote.
3. **"Master-list mein ek disposition dikh rahi hai jo GST ke alawa kisi aur attribute
   ki hai"** — Expected hai, master-list generic hai; sirf uss disposition ko *record*
   karna (Flow A) aur *history padhna* (Flow C) do alag cheezein hain — record karna kisi
   bhi attribute ke liye ho sakta hai, padhna sirf GST ke liye abhi.

---

## 6. Quick Summary

- Disposition = generic outcome/reason-code log against supplier, ek master-list se driven.
- Master-list generic hai (multiple attributes ke liye dispositions define ho sakti hain),
  aur ek open (allowlist-free) master-data API se fetch hoti hai.
- Insert-only (history-preserving), read sirf GST-attribute tak limited abhi.
- KWPL se koi relation nahi — alag concept, alag table. "Disposition" naam se codebase mein
  do aur unrelated features bhi hain (Supplier Rating, Supplier Verification Log) — confuse
  mat hona.

---

## See also

- [`Disposition_Technical_Doc.md`](./Disposition_Technical_Doc.md) — code-level detail
- [`../KWPL KT/KWPL_Business_Doc.md`](../KWPL%20KT/KWPL_Business_Doc.md) —
  unrelated concept, documented separately
- [`../GST KT/GST_Business_Doc.md`](../GST%20KT/GST_Business_Doc.md) — the only attribute
  currently readable through this Disposition read-endpoint

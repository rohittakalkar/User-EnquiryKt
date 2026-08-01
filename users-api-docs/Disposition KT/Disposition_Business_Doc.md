# Disposition — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Disposition_Technical_Doc.md`](./Disposition_Technical_Doc.md) dekho.

**Note**: Disposition aur KWPL genuinely **do alag concepts** hain — koi shared table,
FK-relation, ya common write-controller nahi mila. Dekho
[`../KWPL KT/KWPL_Business_Doc.md`](../KWPL%20KT/KWPL_Business_Doc.md).

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

**Disposition** ek "outcome/reason-code log" hai — jab kisi internal-process (jaise
verification, review, ya koi aur attribute-specific evaluation) ka result record karna ho,
uska "disposition" (outcome/status-reason) ek predefined master-list se pick karke supplier
ke against save kiya jaata hai. Yeh ek generic, reusable logging-mechanism hai jo alag-alag
"attributes" (jaise GST) ke against use ho sakta hai.

**Business impact**: Internal-teams ke liye ek audit-trail hai — "kis attribute ke liye,
kya outcome record hua, kab, aur kisne (kaunsa system/MODID) mark kiya" — verification/
review-workflows ki traceability ke liye zaroori hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Internal tools (GLADMIN, SELLERMY, IMOB, MSITE)** | Disposition-record likhte hain |
| **Supplier (indirectly)** | Jiske against yeh disposition record ban raha hai |

---

## 3. Business Flow (step-by-step, no code)

```
1. Koi internal-process (jaise GST-verification) apna outcome ek "disposition" ke roop
   mein record karta hai — ek predefined master-list se ek disposition-code pick karke
2. System is record ko supplier ke against save karta hai, timestamp ke saath
3. Baad mein, yeh disposition-history query ki ja sakti hai — ya toh poori history, ya
   sirf latest-entry
4. Abhi ke liye, read-side sirf "GST" attribute ke liye kaam karta hai — baaki
   attributes ke liye "no data" response milta hai
```

**Business impact**: Ek generic-design-intent-wala feature hai (kai attributes support
karne ke liye), lekin currently sirf GST-attribute tak practically limited hai.

---

## 4. Business Rules — Plain Language Mein

1. **Disposition ek master-list se aati hai** — random text nahi, ek predefined
   reason/outcome-code select kiya jaata hai.
2. **Naya disposition record hamesha ek fresh insert hota hai** — purana overwrite nahi
   hota, matlab har evaluation ka apna history-entry banta hai.
3. **Read-side abhi sirf GST-attribute support karta hai** — koi aur attribute-name
   diya jaaye, "no data found" milta hai, chahe backend-design generic ho.
4. **"All history" ya "sirf latest" dono query kiye ja sakte hain** — caller decide
   karta hai.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine kisi doosre attribute ka disposition query kiya, data nahi mila"** —
   Expected hai abhi, sirf GST supported hai read-side pe.
2. **"Disposition history mein multiple entries hain"** — Expected hai, har naya
   disposition ek fresh row insert karta hai, purane overwrite nahi hote.

---

## 6. Quick Summary

- Disposition = generic outcome/reason-code log against supplier, ek master-list se driven.
- Insert-only (history-preserving), read sirf GST-attribute tak limited abhi.
- KWPL se koi relation nahi — alag concept, alag table.

---

## See also

- [`Disposition_Technical_Doc.md`](./Disposition_Technical_Doc.md) — code-level detail
- [`../KWPL KT/KWPL_Business_Doc.md`](../KWPL%20KT/KWPL_Business_Doc.md) —
  unrelated concept, documented separately

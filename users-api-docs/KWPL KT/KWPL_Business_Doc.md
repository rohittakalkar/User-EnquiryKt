# KWPL (Product-Listing Keyword) — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`KWPL_Technical_Doc.md`](./KWPL_Technical_Doc.md) dekho.

**Note**: KWPL aur Disposition genuinely **do alag concepts** hain — koi shared table,
FK-relation, ya common write-controller nahi mila. Dekho
[`../Disposition KT/Disposition_Business_Doc.md`](../Disposition%20KT/Disposition_Business_Doc.md).

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

**KWPL** (Product-Listing Keyword) ek internal record-keeping mechanism hai jo track karta
hai ki kisi supplier/company ke against **kaunse keywords kis city mein, kis time-period ke
liye "served" (serve/assign) kiye gaye** — kisi work-order (WO) ya "serve" transaction ke
part ke roop mein. Yeh customer-facing feature nahi hai — ek internal
sales/ops/campaign-tracking-tool jaisa lagta hai jo bataata hai "yeh keyword iss supplier
ko iss duration ke liye assign/serve kiya gaya".

**Business impact**: Sales/ops teams ke liye traceability — kaunsa keyword, kis city mein,
kis work-order ke against, kab se kab tak "enable" tha — yeh audit-trail internal-processes
(jaise lead-generation ya campaign-tracking) ko support karta hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Internal admin/ops tools (GLADMIN, WebERP)** | Keyword-serve records insert/update/disable karte hain |
| **Supplier (indirectly)** | Jiske against yeh keyword-serve record ban raha hai |

---

## 3. Business Flow (step-by-step, no code)

```
1. Internal-tool ek naya keyword-serve record create karta hai — supplier, city, company,
   keyword, description, validity-period (from-to dates), aur ek "work-order ID" ke saath
2. Record "enabled" state mein active rehta hai
3. Baad mein, yeh record kai tarikon se manage ho sakta hai:
   - "Disable" kiya ja sakta hai (jaise campaign khatam hua)
   - "Update" kiya ja sakta hai (jaise validity extend karni ho)
   - "Search-term" update kiya ja sakta hai (keyword hi change karna ho)
4. Har action ka audit-trail (kisne, kab, kahan se) capture hota hai
```

**Business impact**: Flexible workflow hai — same underlying record insert/disable/update/
keyword-change sab handle kar sakta hai, single API ke through.

---

## 4. Business Rules — Plain Language Mein

1. **Chaar actions supported hain**: INSERT (naya record), DISABLE (band karna),
   UPDATE (validity/work-order extend), SEARCH_UPDATE (sirf keyword-term change).
2. **DISABLE/UPDATE/SEARCH_UPDATE sirf currently-"enabled" (active) records ko target
   karte hain** — already-disabled record dobara touch nahi hota.
3. **Sirf do internal-systems allowed hain** — GLADMIN aur WebERP, matlab yeh purely
   internal/admin-driven feature hai, koi supplier-facing ya public API nahi.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine keyword-record disable karne ki koshish ki, kuch update nahi hua"** — Ho
   sakta hai record already disabled ho (query sirf active records match karti hai).

---

## 6. Quick Summary

- KWPL = internal keyword-serving/campaign-tracking record against supplier+city+work-order.
- 4 actions: Insert/Disable/Update/Search-term-update.
- Purely internal-tool-driven (GLADMIN/WebERP only), koi supplier-facing surface nahi.
- Disposition se koi relation nahi — alag concept, alag table.

---

## See also

- [`KWPL_Technical_Doc.md`](./KWPL_Technical_Doc.md) — code-level detail
- [`../Disposition KT/Disposition_Business_Doc.md`](../Disposition%20KT/Disposition_Business_Doc.md) —
  unrelated concept, documented separately

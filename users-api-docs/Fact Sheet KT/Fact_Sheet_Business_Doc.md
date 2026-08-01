# Fact Sheet — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Fact_Sheet_Technical_Doc.md`](./Fact_Sheet_Technical_Doc.md) dekho.

**Note**: Yeh feature GST-jaisa hi ek **shared multi-purpose "user-details" write/read
endpoint** ka hissa hai (`detailType`-based dispatch) — dekho
[`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) same-controller-pattern
ke liye.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

**Fact Sheet** supplier ke company ke baare mein **detailed business-capability
information** capture karta hai — jaise: export-percentage (kitna business export hota
hai), international-credit-rating-numbers (Dun & Bradstreet, Coface), after-sale-support
policy, sampling-policy, competitive-advantage, contract-manufacturing-capability,
quality-facilities, delivery-time, payment-terms/mode, shipping-mode, aur staff-count
(engineers, skilled/semi-skilled workers, consultants).

**Business impact**: Yeh especially **international/export-buyers** ke liye ek detailed
supplier-capability-profile hai — buyer decide kar sakta hai ki supplier unki business-
requirements (volume, quality-standards, payment-terms) fulfill kar sakta hai ya nahi,
bina directly contact kiye.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Apni company ki fact-sheet-details fill karta hai |
| **Buyer** | Fact-sheet dekh ke supplier ki capability assess karta hai |

---

## 3. Business Flow (step-by-step, no code)

```
1. Supplier apni company ki detailed capability-information submit karta hai — export%,
   credit-ratings, quality/delivery/payment-terms, staff-strength
2. System check karta hai — pehle se fact-sheet-record hai ya nahi (via internal
   record-ID)
   - Agar hai, update ho jaata hai (poora record overwrite)
   - Agar nahi, naya record insert hota hai
3. Change downstream systems (jaise search/directory) ko bhi propagate hoti hai
4. Buyer profile-detail dekhte waqt, fact-sheet-data (agar exist karta hai) embedded
   milta hai
```

**Business impact**: GST/Company-Registration jaisa hi ek "structured-business-profile"
data-point hai — same shared-controller-pattern se manage hota hai.

---

## 4. Business Rules — Plain Language Mein

1. **Ek supplier ka sirf ek hi fact-sheet-record hota hai** — insert-or-full-update,
   history nahi maintain hoti (unlike GST-HSN jo history-table use karta hai).
2. **Update poora-record overwrite karta hai** — partial-update supported nahi hai is
   field-set ke liye, saari fields ek saath update honi chahiye.
3. **Yeh GST/Bank-Details/Form-8 jaise hi ek shared write-endpoint ka hissa hai** —
   `detailType=FactSheet` parameter se dispatch hota hai.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine sirf ek field update ki, baaki fields khali ho gayi"** — Ho sakta hai agar
   update-request mein baaki fields nahi bheji gayi thi — is endpoint mein full-record
   overwrite hota hai, partial-update nahi.

---

## 6. Quick Summary

- Fact Sheet = detailed supplier-business-capability profile (export%, credit-ratings,
  quality/delivery/payment-terms, staff-count).
- GST/Bank-Details/Form-8 jaisa hi shared "user-details" endpoint ka hissa.
- Insert-or-full-update, koi partial-update ya history nahi.

---

## See also

- [`Fact_Sheet_Technical_Doc.md`](./Fact_Sheet_Technical_Doc.md) — code-level detail
- [`../GST KT/GST_Technical_Doc.md`](../GST%20KT/GST_Technical_Doc.md) — same shared-controller pattern

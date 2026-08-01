# Payment Transaction (International) — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Payment_Txn_Technical_Doc.md`](./Payment_Txn_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Jab koi **international buyer** IndiaMART ke through ek supplier ko **payment** karta
hai (cross-border trade-transaction), uska record save hota hai — buyer ki details
(naam, country, email), transaction-amount, currency, aur ek **invoice** (jiska URL/ID
capture hota hai). Yeh IndiaMART ke international-trade/export-payment-facilitation
ka ek transaction-log hai.

**Business impact**: Yeh cross-border-payment-transactions ka audit-trail hai — supplier
aur buyer dono ke liye proof ki transaction hui, invoice-linkage ke saath. Financial/
compliance-reporting ke liye zaroori hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **International Buyer** | Payment karta hai supplier ko |
| **Supplier (seller)** | Payment receive karta hai |
| **Internal apps (GLADMIN, SELLERMY, IMOB, Android, iOS)** | Transaction-record likhte hain |

---

## 3. Business Flow (step-by-step, no code)

```
1. International buyer ek payment karta hai supplier ko
2. Transaction-details record hoti hain — buyer-info (ID, country, naam, email),
   seller-ID, currency, amount, invoice-URL
3. Invoice-URL se invoice-ID automatically extract hoti hai (URL ke "#" ke baad wala
   hissa)
4. Transaction ke saath additional custom-details (`txn_details`) bhi attach ho sakte
   hain — structured JSON data
5. Baad mein, existing transaction ka `usr_txn_details` update bhi ho sakta hai
   (invoice-ID aur seller-ID se match karke)
```

**Business impact**: Insert-once, update-limited-fields — transaction ka core-data
(amount, buyer-info) change nahi hota, sirf supplementary-details update ho sakti hain.

---

## 4. Business Rules — Plain Language Mein

1. **Do actions supported hain**: Insert (naya transaction) aur Update (sirf
   `txn_details` custom-JSON-field update hoti hai).
2. **Insert ke liye poori buyer/seller/amount/invoice-info mandatory hai.**
3. **Update sirf invoice-ID aur seller-ID se match hoke hoti hai** — matlab update
   karne ke liye pehle se transaction ka invoice-URL pata hona chahiye.
4. **Invoice-ID invoice-URL se automatically derive hoti hai** — URL ke "#" symbol ke
   baad ka hissa.
5. **Sirf allowed internal-apps hi transaction record kar sakte hain**.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine transaction-details update ki, amount change nahi hua"** — Expected hai,
   update sirf `txn_details` (custom-JSON) field ko touch karta hai, core-transaction-
   data immutable hai.
2. **"Invoice-URL mein # nahi tha, invoice-ID khali reh gayi"** — Expected hai, ID
   extraction sirf `#` ke baad ke text pe depend karta hai.

---

## 6. Quick Summary

- Payment Txn = international buyer-to-seller transaction-log, invoice-linked.
- Insert (full-record) / Update (sirf txn_details custom-JSON).
- Invoice-ID auto-extracted from invoice-URL.

---

## See also

- [`Payment_Txn_Technical_Doc.md`](./Payment_Txn_Technical_Doc.md) — code-level detail

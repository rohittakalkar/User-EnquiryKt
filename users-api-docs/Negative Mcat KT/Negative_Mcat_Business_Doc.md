# Negative Mcat — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Negative_Mcat_Technical_Doc.md`](./Negative_Mcat_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

**"Mcat"** (Micro-Category) IndiaMART ka product-classification unit hai — har product
listing ek ya zyada mcats se associate hoti hai. **Negative Mcat** ek "exclusion-list"
concept hai: kisi supplier ya kisi specific product-listing ke liye, kuch mcats ko
explicitly "yeh mcat iske liye applicable nahi hai / block kiya gaya hai" mark kiya ja
sakta hai.

**Business impact**: Yeh mis-categorization ya spam/irrelevant-listing issues ko control
karne ka ek tool hai — agar koi supplier/product galat category mein baar-baar aa raha ho,
internal-team us mcat ko unke liye negative (excluded) mark kar sakti hai. Yeh downstream
search/browse/recommendation systems ko bhi propagate hota hai taaki wahan bhi yeh
exclusion respect ho.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Internal ops/admin teams (many allowed apps)** | Negative-mcat mark/unmark karte hain — GLADMIN, WebERP, Tollfree, Sales, PCAT-Admin, etc. |
| **Supplier** | Indirectly affected — unki product-listing kisi negative-marked mcat mein categorize nahi hogi |
| **Downstream systems (search/IMSDB, alerting)** | Is exclusion-list ko apne records mein sync rakhte hain |

---

## 3. Business Flow (step-by-step, no code)

```
1. Ek internal-tool/admin ek mcat ko kisi supplier (glusr) ya kisi specific product-item
   ke liye "negative" mark karta hai — insert (naya) ya delete (soft-remove) action ke
   saath
2. System is record ko save karta hai, kisne/kab mark kiya yeh bhi track hota hai
3. Yeh change downstream systems ko async notify hoti hai — search/directory-system aur
   ek internal-alerting/rejection-pipeline dono ko
4. Supplier/product ki respective listings ab is mcat se exclude treat hoti hain
   (downstream logic ke through)
5. Koi bhi caller glusr ya product-item ke against currently-active negative-mcats
   query kar sakta hai
```

**Business impact**: Ek control-mechanism hai jo listing-quality maintain karne mein
madad karta hai — bina product delete kiye, sirf specific category-association ko
suppress kiya ja sakta hai.

---

## 4. Business Rules — Plain Language Mein

1. **Do actions supported hain** — "Insert" (naya negative-mcat add karna) aur "Delete"
   (soft-remove, purana record still DB mein rehta hai lekin "deleted" flag ke saath).
2. **Ek negative-mcat record supplier-level ho sakta hai (poora account) ya item-level**
   (ek specific product-listing) — `item_id` optional hai; agar nahi diya, `-1` default
   treat hota hai (matlab supplier-level).
3. **Bahut saare internal-systems yeh action perform kar sakte hain** — ek badi allowlist
   hai (GLADMIN, Tollfree, WebERP, PCAT-Admin, Tolexo, IMOB, etc.) — matlab yeh ek
   widely-used internal-tool feature hai, kai teams isse touch karti hain.
4. **Change downstream systems ko propagate hoti hai** — search/directory-replica aur ek
   alerting/rejection-pipeline dono ko async notify kiya jaata hai.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Meri listing ek category se gayab ho gayi"** — Ho sakta hai koi internal-team ne
   us mcat ko unke supplier-account ya specific-item ke liye negative-mark kiya ho.
2. **"Maine negative-mcat delete kiya, record abhi bhi dikh raha hai"** — Delete "soft"
   hai, record physically remove nahi hota, sirf status-flag change hota hai — downstream
   systems ko propagate hone mein thoda time lag sakta hai.

---

## 6. Quick Summary

- Negative Mcat = supplier/product-level category-exclusion mechanism, internal-ops-driven.
- Insert (add-exclusion) / Delete (soft-remove-exclusion).
- 3-way downstream fan-out: search-replica, alerting-pipeline, aur ek forwarding-consumer.
- Widely-used internal feature — bahut saare internal-apps allowed hain.

---

## See also

- [`Negative_Mcat_Technical_Doc.md`](./Negative_Mcat_Technical_Doc.md) — code-level detail

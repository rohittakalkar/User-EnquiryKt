# Buyer Blocking — Business Doc (Product Perspective)

**Note**: yeh doc **Blocking** cover karta hai — yeh **Matchmaking** se alag module hai.
Matchmaking ke liye [`../Matchmaking KT/Matchmaking_Business_Doc.md`](../Matchmaking%20KT/Matchmaking_Business_Doc.md)
dekho.

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Blocking_Technical_Doc.md`](./Blocking_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Kabhi-kabhi ek supplier ko ek particular buyer se lead/enquiry nahi chahiye hoti — ho sakta
hai woh buyer abusive ho, spam kar raha ho, ya bas ek unwanted repeat-caller ho. Blocking
feature supplier ko yeh control deta hai: **"iss buyer se mujhe kabhi contact nahi chahiye."**

**Business impact**: Yeh supplier ka apna, direct control hai — Matchmaking (jo system
automatically karta hai) ke ulat, Blocking hamesha **supplier ka manual decision** hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Kisi buyer ko block ya unblock karta hai — yeh unka apna active decision hai |
| **Buyer jise block kiya gaya** | Iska pata nahi chalta unhe (silent block) — koi notification nahi jaati |
| **Platform ka lead-delivery system** | Block-status ko consult kar sakta hai future leads filter karne ke liye *(exact enforcement point iss review ke scope se bahar)* |

---

## 3. Main Business Flow (step-by-step, no code)

```
1. Supplier ek buyer ko block karta hai (jaise ek unwanted lead ke response mein)
2. System yeh record kar leta hai: "supplier X ne buyer Y ko block kiya"
3. Agar supplier baad mein mann badle, wahi action "unblock" ke saath dobara call ki jaa
   sakti hai — record update ho jaata hai, delete nahi hota (history reh jaati hai kab
   block/unblock hua)
4. Jab bhi supplier iss buyer ka profile dekhta hai, unhe pata chalta hai yeh buyer blocked
   hai
```

**Business impact**: Yeh ek simple, instant, reversible action hai — koi approval-workflow
ya delay nahi, supplier ka decision turant apply hota hai.

---

## 4. Business Rules — Plain Language Mein

1. **Block aur Unblock same API se hote hain** — ek `block_status` flag (1 ya 0) decide
   karta hai kaunsa action hai.
2. **History preserve hoti hai** — jab bhi block ya unblock hota hai, us action ki date
   (`blocking_date` ya `unblocking_date`) record hoti hai, purana record delete nahi hota,
   sirf update hota hai.
3. **Yeh sirf supplier ka apna action hai** — koi automated/system-triggered blocking iss
   review mein nahi mili, sab manual hai.
4. **Buyer ko notify nahi kiya jaata** — silent hai, jaisa aam taur pe blocking features
   platforms pe hoti hain.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine buyer ko block kiya, phir bhi mujhe unse messages/leads mil rahe hain"** —
   Blocking sirf ek **record** hai; iska actual enforcement (lead-delivery ko rokna) kis
   exact system mein hota hai, yeh iss review ke scope se bahar tha — agar enforcement
   missing lage, alag investigation chahiye ki lead-delivery system yeh flag consult kar
   raha hai ya nahi.
2. **"Maine unblock kiya, lekin purana block history dikh rahi hai kahin"** — Expected hai,
   history preserve hoti hai (point 2 upar) — sirf current `block_status` change hota hai,
   audit-trail nahi mitta.

---

## 6. Quick Summary

- Blocking = supplier ka apna, manual, reversible decision — "iss buyer se contact nahi
  chahiye."
- Matchmaking se bilkul independent hai, alag purpose (system-verification vs
  supplier-choice).
- Simple upsert-based record, koi async pipeline/queue involved nahi.

---

## See also

- [`Blocking_Technical_Doc.md`](./Blocking_Technical_Doc.md) — code-level detail
- [`../Matchmaking KT/Matchmaking_Business_Doc.md`](../Matchmaking%20KT/Matchmaking_Business_Doc.md) —
  related lekin independent module
- [`../buyer_seller_discovery_matching_product_overview.md`](../buyer_seller_discovery_matching_product_overview.md) —
  bigger Buyer-Seller Discovery & Matching story

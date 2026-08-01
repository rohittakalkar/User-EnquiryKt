# Visiting Card — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Visiting_Card_Technical_Doc.md`](./Visiting_Card_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Supplier ka **physical visiting-card (business-card)** scan/photo capture karke
IndiaMART profile ke saath store kiya jaata hai — front-side, back-side, dono. Yeh
photo internal-teams ke liye ek **verification aur design/enrichment workflow** se
guzarta hai — jaise: kisne front/back capture kiya, internal-team ne uska content
"enrich" kiya (jaise details extract ki), design-team ne approve kiya, aur final
approval mila ya nahi.

Agar kisi supplier ke paas visiting-card hi nahi hai, ek explicit **"No Visiting
Card" (NO_VC)** status bhi track hota hai.

**Business impact**: Visiting-card supplier-verification/trust ka ek physical-proof
hai — GST/ID-Proof jaisa hi ek aur trust-signal, lekin yahan poora ek **multi-stage
internal-workflow** hai (capture → enrich → design → approve).

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Field-team / supplier** | Visiting-card photo capture karte hain (front/back) |
| **GLADMIN (internal-admin)** | Comments/annotations add karta hai |
| **WebERP (internal-workflow-team)** | Enrich, design-approve, aur final-approve karta hai — teen alag sub-workflows |
| **Buyer** | Visiting-card display dekh sakta hai supplier-profile pe |

---

## 3. Business Flow (step-by-step, no code)

```
1. Field-team/supplier front-side aur/ya back-side ki photo upload karta hai (dono
   ek saath, ya ek-ek karke)
2. Naya record banta hai (agar pehli baar hai) — location (lat/long) bhi capture
   hoti hai jahan se photo li gayi
3. Agar supplier ke paas visiting-card nahi hai, ek "NO_VC" (No Visiting Card) status
   record hota hai
4. Internal-team (WebERP) is card ko process karti hai:
   - "Enrich" — content ko review/extract karna
   - "Design" — design-quality approve karna
   - "Approve" — final approval dena
5. GLADMIN admin-comments/annotations bhi add kar sakta hai
6. Har successful-update ek downstream-event bhi fire karta hai
7. Buyer/koi bhi caller final visiting-card display kar sakta hai
```

**Business impact**: Yeh ek multi-team, multi-stage internal-workflow hai — capture
(field), review (WebERP-enrich), quality-check (WebERP-design), approval
(WebERP-approve) — sab ek hi record pe apna step add karte hain.

---

## 4. Business Rules — Plain Language Mein

1. **Insert vs Update decide hota hai `VISITING_CARD_ID` ki presence se** — naya ID nahi
   diya, naya record banta hai.
2. **Naye-record ke liye front ya back attachment (ya design-update) mein se kam-se-kam
   ek zaroori hai** — sab khali nahi ho sakte.
3. **`TYPE=NO_VC` ek special-case hai** — "supplier ke paas visiting-card nahi hai" ko
   record karta hai, yeh alag table mein jaata hai (`STS_COMP_NO_VISIT_CARD`).
4. **WebERP ke andar 4 alag update-sub-flows hain** — ENRICH, APPROVE, DESIGN,
   ENRICHDESIGN (design+enrich combined) — `UPDATE_FLAG` decide karta hai kaunsa.
5. **Naya attachment upload hone par, purane approval/design-flags reset ho jaate hain**
   (`ENRICHED_FLAG`/`APPROV_STATUS`/`DESIGN_FLAG` sab `NULL`) — matlab naya photo dena
   dobara-review-cycle trigger karta hai.
6. **`APPROV_STATUS` ke through, "No Visiting Card" status bhi automatically update
   hoti hai** — agar reject('R') hua toh No-VC-status 'N' set hoti hai, warna 'Y'.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine naya photo upload kiya, purana approval-status gayab ho gaya"** — Expected
   hai, naya attachment upload karne se review-cycle reset ho jaata hai.
2. **"Maine card upload kiya, phir bhi NO_VC status dikh raha"** — Ho sakta hai
   approval abhi pending ('P') ho — dekho Technical Doc ka approval-status-logic.

---

## 6. Quick Summary

- Visiting Card = supplier ke business-card ka photo (front/back), multi-stage
  internal-review-workflow (capture → enrich → design → approve) ke saath.
- NO_VC status bhi explicitly track hoti hai.
- 3 internal-actor-types (field/GLADMIN/WebERP), har ek apna step perform karta hai.

---

## See also

- [`Visiting_Card_Technical_Doc.md`](./Visiting_Card_Technical_Doc.md) — code-level detail

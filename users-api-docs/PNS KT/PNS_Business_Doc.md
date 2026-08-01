# PNS (GSM Number Allocation) — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`PNS_Technical_Doc.md`](./PNS_Technical_Doc.md) dekho.

**Note**: PNS aur PNS Setting **do alag concepts** hain — alag tables, alag purpose.
PNS (yeh doc) ek supplier ko ek **virtual-phone-number (GSM) allocate** karta hai. PNS
Setting supplier ki **contact/notification-preferences** manage karta hai. Dekho
[`../PNS Setting KT/PNS_Setting_Business_Doc.md`](../PNS%20Setting%20KT/PNS_Setting_Business_Doc.md).

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

**PNS** (likely "Pay-N-Search"/call-tracking-number-service) supplier ko ek **virtual
phone-number (GSM)** allocate karta hai — jaise ek call-tracking/masking-number jo
buyer aur supplier ke beech calls route karta hai bina supplier ka actual number
expose kiye. Yeh number ek shared-pool (`GL_GSM_MASTER`) se **vendor-type ke hisaab se**
allocate hota hai.

**Business impact**: Call-tracking-numbers buyer-supplier-interactions ko monitor karne
mein madad karte hain (lead-quality-analysis, call-recording ke liye), aur supplier ki
privacy bhi protect karte hain (actual number expose nahi hota).

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Internal-tools (bahut saare, badi allowlist)** | Supplier ko GSM-number allocate karte hain |
| **Supplier** | GSM-number receive karta hai (unke against) |
| **GSM-number-pool** | Available-numbers ka shared-inventory, vendor-type-specific |

---

## 3. Business Flow (step-by-step, no code)

```
1. Internal-tool ek supplier ko GSM-number allocate karne ki request bhejta hai,
   vendor-type specify karke
2. System ek available-number us vendor-type ke pool se pick karta hai
3. Us number ko "reserved"/"assigned" mark kiya jaata hai (available-status change hoti
   hai)
4. Change downstream-systems ko async-notify hoti hai
```

**Business impact**: Ek simple pool-allocation-mechanism hai — first-available-number
pick karke assign kar deta hai, per-vendor-type.

---

## 4. Business Rules — Plain Language Mein

1. **Number-pool vendor-type-specific hai** — sirf matching-vendor-type ke available-
   numbers hi allocate ho sakte hain.
2. **Allocation "first-available" basis pe hoti hai** — koi priority/preference-logic
   nahi, jo bhi pehla available-number milta hai wahi assign hota hai.
3. **Bahut saare internal-apps allowed hain** — widely-used internal-allocation-tool.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Mujhe number allocate nahi hua"** — Ho sakta hai us vendor-type ke liye koi
   available-number pool mein bacha hi na ho.

---

## 6. Quick Summary

- PNS = GSM/virtual-number allocation-service, vendor-type-specific shared-pool se.
- PNS Setting se alag concept hai (contact/notification-preferences).

---

## See also

- [`PNS_Technical_Doc.md`](./PNS_Technical_Doc.md) — code-level detail
- [`../PNS Setting KT/PNS_Setting_Business_Doc.md`](../PNS%20Setting%20KT/PNS_Setting_Business_Doc.md) —
  unrelated concept, documented separately

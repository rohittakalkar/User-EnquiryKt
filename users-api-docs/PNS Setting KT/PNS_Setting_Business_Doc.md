# PNS Setting — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`PNS_Setting_Technical_Doc.md`](./PNS_Setting_Technical_Doc.md) dekho.

**Note**: PNS Setting aur PNS **do alag concepts** hain — PNS ek GSM/virtual-number-
allocation-service hai, yeh feature supplier ki **contact/notification-preferences**
manage karta hai. Dekho [`../PNS KT/PNS_Business_Doc.md`](../PNS%20KT/PNS_Business_Doc.md).

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Supplier apne **PNS-related-contact-preferences** set kar sakta hai — jaise off-hours
mein calls kis number pe route hon, aur woh contact-number kya ho. Yeh setting ek
supplier ke **multiple contact-numbers** (`GLUSR_USR_ADDT_CONTACT`) ke against
individually configure ho sakti hai, ek "type" ke through.

**Business impact**: Yeh supplier ko control deta hai ki business-hours ke bahar
(off-hours) unki calls kaise handle hon — kaunsa number ring kare, ya feature
enable/disable ho.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Apni PNS-off-hours-setting configure karta hai |
| **Internal-tools (GLADMIN, MAPI, BUYERMY, SELLERMY, Android, iOS)** | Setting update kar sakte hain |

---

## 3. Business Flow (step-by-step, no code)

```
1. Supplier apni PNS-setting submit karta hai — off-hours-flag (on/off) aur
   contact-number, ek specific "type" ke against
2. Agar yeh setting kisi specific additional-contact ke liye hai, us contact-ID se bhi
   link hoti hai
3. System naya-record insert karta hai ya existing update karta hai (per user+type,
   ya per user+contact+type)
4. Duplicate-request silently handle hoti hai (dobara-submit se error nahi aata)
```

**Business impact**: Ek per-contact-configurable-setting hai — supplier ke multiple
phone-numbers ho sakte hain, har ek ki apni PNS-off-hours-behavior ho sakti hai.

---

## 4. Business Rules — Plain Language Mein

1. **Setting insert-or-update hoti hai** — pehli baar naya record, dobara update.
2. **Do variants hain** — general (per user+type) aur specific-contact-linked
   (per user+contact+type).
3. **Duplicate-submission errors nahi deti** — `ON CONFLICT DO NOTHING` jaisa
   idempotent-behavior.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine setting dobara submit ki, kuch change nahi hua"** — Expected hai agar
   exact-same-values already exist karti hain — duplicate silently skip hoti hai.

---

## 6. Quick Summary

- PNS Setting = supplier ke off-hours-call-routing/contact-preferences, per-contact-
  configurable.
- Insert-or-update, duplicate-safe.
- PNS (GSM-allocation) se koi relation nahi — alag concept, alag table.

---

## See also

- [`PNS_Setting_Technical_Doc.md`](./PNS_Setting_Technical_Doc.md) — code-level detail
- [`../PNS KT/PNS_Business_Doc.md`](../PNS%20KT/PNS_Business_Doc.md) —
  unrelated concept, documented separately

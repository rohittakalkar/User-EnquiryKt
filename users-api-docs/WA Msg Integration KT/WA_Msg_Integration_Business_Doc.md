# WhatsApp Message Integration — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`WA_Msg_Integration_Technical_Doc.md`](./WA_Msg_Integration_Technical_Doc.md) dekho.

**Scope**: Yeh doc **do genuinely alag cheezein** cover karta hai jo dono "WhatsApp" naam
share karte hain lekin technically independent hain:

1. **WhatsApp Business Platform (BSP) Integration** — supplier apna WhatsApp Business
   account IndiaMART ke saath link karta hai (Meta/Facebook-backed onboarding)
2. **Outbound WhatsApp Message Sending** — IndiaMART khud supplier/buyer ko ek specific
   transactional WhatsApp message bhejta hai (jaise "App Install karo" nudge)

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

### 1.1 — WhatsApp Business Integration (linking)

Supplier apna official **WhatsApp Business Account** IndiaMART platform se link kar sakta
hai. Yeh Meta/Facebook ke WhatsApp Business Platform ke through hota hai — ek proper
business-messaging onboarding, jaisa kisi bhi vendor ke saath enterprise WhatsApp
integration hoti hai (KYC, business-manager verification, quality-rating tracking, sab
saath).

**Business impact**: Ek baar link ho jaaye, supplier IndiaMART ke through WhatsApp pe buyers
se baat kar sakta hai (ya IndiaMART unki taraf se automated messages bhej sakta hai), bina
alag se WhatsApp Business API setup kiye. Yeh ek premium/value-added messaging capability
hai suppliers ke liye.

### 1.2 — Outbound WhatsApp Messaging (nudges)

Yeh ek alag, chhota utility hai jo IndiaMART khud use karta hai kisi user ko ek specific
WhatsApp message bhejne ke liye — jaise "IndiaMART app install karo" nudge. Yeh koi supplier
ka apna linked-account nahi use karta, IndiaMART ka apna vendor-backed sending mechanism hai.

**Business impact**: Ek marketing/engagement tool — users ko re-engage karne ke liye WhatsApp
jaisa high-open-rate channel use kiya jaata hai, bina spam-jaisa lagne ke (frequency-capped).

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Apna WhatsApp Business account link karta hai (Integration feature) |
| **Meta/Facebook (external)** | Business-verification, KYC, quality-rating provide karta hai |
| **BSP (Business Solution Provider, external vendor)** | WhatsApp messaging infrastructure provide karta hai jispe integration based hai |
| **IndiaMART internal systems** | Outbound nudges trigger karte hain (jaise app-install reminder) |
| **Buyer/Supplier (message receiver)** | Nudge messages receive karta hai, apne rate-limit ke andar |

---

## 3. Do Features — Alag-Alag Samjho

### 3.1 — WhatsApp Business Integration Record

**Business flow**:
```
1. Supplier (ya internal admin) WhatsApp Business account link karne ka process start karta hai
2. System ek integration-record banata hai — phone number, business ID, token, service dates
3. Meta/Facebook-side verification status track hoti hai: business-manager verification,
   display-name verification, KYC status, quality-rating
4. Status changes (jaise "verified" ban jaana) ek audit-log mein bhi record hote hain
5. Agar account "Scam"/"Spam" flag ho jaaye Meta ki taraf se, yeh bhi track hota hai
6. Har change downstream systems ko notify hoti hai (async)
```

**Business impact**: Yeh ek **ongoing relationship** hai, ek one-time setup nahi — status
badalta rehta hai (quality-rating, verification-status), aur system inhe continuously track
karta hai.

### 3.2 — Outbound Nudge Messaging

**Business flow**:
```
1. Koi internal trigger (jaise "yeh user app use nahi kar raha, unhe WhatsApp pe nudge bhejo")
2. System check karta hai: pichle 24 ghante mein isi user ko yeh message pehle nahi bheja gaya
3. Agar allowed hai, ek pre-defined message-template (ya free-text) WhatsApp pe bhej diya jaata hai
4. Agar already bheja ja chuka hai 24 ghante mein, message skip ho jaata hai (spam-prevention)
```

**Business impact**: Ek disciplined, rate-limited outreach mechanism — users ko baar-baar
same message se irritate nahi karta.

---

## 4. Business Rules — Plain Language Mein

1. **Integration record insert-vs-update dono support karta hai** — pehli baar link karna
   (naya record) ya existing link ko update karna (jaise naya token, status change), dono
   ek hi API se hote hain.
2. **Status-change tracking automatic hai** — jab bhi integration ka status ek certain range
   mein change hota hai (jaise "verified" ban jaana ya "suspended" ho jaana), system khud
   ek log-entry bana deta hai, alag se manually log karne ki zaroorat nahi.
3. **Scam/Spam flagging ek special path hai** — agar Meta ki taraf se account scam/spam flag
   ho, system ismein specific comment ke saath record karta hai.
4. **Outbound nudges ek strict 24-hour frequency-cap follow karte hain** — same process-type
   (jaise "App Install nudge") dobara nahi bheja jaata agar pichle 24 ghante mein pehle se
   bheja gaya ho.
5. **Mobile number format automatically normalize hota hai** — agar `+91` prefix missing ho,
   system khud add kar deta hai bhejne se pehle.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine WhatsApp integration link kiya, lekin messages nahi bhej pa raha"** — Yeh do
   alag systems hain (§3.1 vs §3.2) — sirf account link hona kaafi nahi, Meta-side
   verification bhi complete honi chahiye (quality-rating, KYC-status check karo).
2. **"Mujhe baar-baar app-install ka WhatsApp message aa raha hai"** — Agar 24-hour
   frequency-cap kaam kar raha hai, yeh nahi hona chahiye — agar ho raha hai, technical
   investigation chahiye (frequency-check logic mein bug ho sakta hai).
3. **"Meri WhatsApp integration achanak scam maar gayi"** — Yeh Meta ki taraf se ek external
   decision hai, IndiaMART ka apna nahi — system sirf isse track/reflect karta hai.

---

## 6. Quick Summary

- WhatsApp Business Integration = supplier ka apna account link karna (ongoing,
  status-tracked relationship).
- Outbound Nudge Messaging = IndiaMART ka apna, rate-limited, transactional message-sending
  tool.
- Dono independent hain, sirf naam se WhatsApp share karte hain.
- Integration mein Meta/Facebook-side verification/KYC/quality-rating sab track hota hai.
- Nudges 24-hour frequency-cap follow karte hain, spam-jaisa nahi lagte.

---

## See also

- [`WA_Msg_Integration_Technical_Doc.md`](./WA_Msg_Integration_Technical_Doc.md) —
  code-level detail
- [`../users_domain_product_stories.md`](../users_domain_product_stories.md) —
  "Notifications, Alerts & Preferences" story mein messaging-platform integration pehle bhi
  mention hui hai

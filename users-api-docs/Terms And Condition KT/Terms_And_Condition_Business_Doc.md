# Terms And Condition Acceptance — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Terms_And_Condition_Technical_Doc.md`](./Terms_And_Condition_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Jab supplier IndiaMART ke **Terms & Conditions accept** karta hai (jaise signup, ya
kisi policy-update ke baad re-acceptance), yeh event ek log-entry ke roop mein record
hota hai — kab accept kiya, kis IP/device/browser (`USER_AGENT`) se accept kiya.

**Business impact**: Yeh ek **legal/compliance-record** hai — agar kabhi dispute ho ki
supplier ne T&C accept kiya tha ya nahi, yeh log proof ke roop mein kaam aata hai. Ek
event bhi fire hoti hai jo supplier ki session/status ko update karti hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | T&C accept karta hai |
| **Bahut saare internal-apps (badi allowlist)** | Yeh acceptance-event record kar sakte hain |

---

## 3. Business Flow (step-by-step, no code)

```
1. Supplier T&C accept karta hai (app/web pe)
2. System is acceptance ko ek naye log-entry ke roop mein save karta hai — user-ID, IP,
   IP-country, country-ISO-code, user-agent (device/browser), aur date
3. Successful-save hone par, ek downstream-event bhi fire hoti hai jo supplier ki
   session/status update karti hai
```

**Business impact**: Simple, append-only compliance-log — har acceptance apna record
banata hai, purana overwrite nahi hota.

---

## 4. Business Rules — Plain Language Mein

1. **Har acceptance ek fresh record hoti hai** — insert-only, koi update/overwrite nahi.
2. **Bahut saare internal-apps allowed hain** — ek badi allowlist (30+ entries), matlab
   T&C-acceptance kayi channels (web, mobile, internal-tools) se ho sakti hai.
3. **Successful-save ke baad, ek session-update-event trigger hoti hai** — supplier ki
   session/status kahin aur update hoti hai isi event ke through.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine T&C dobara accept kiya"** — Expected hai, har baar naya record banega,
   history preserve hoti hai.

---

## 6. Quick Summary

- Terms And Condition Acceptance = compliance-log, insert-only, per-acceptance-event.
- Bahut saare internal-apps allowed (badi allowlist).
- Successful-insert ek downstream session-update-event trigger karta hai.

---

## See also

- [`Terms_And_Condition_Technical_Doc.md`](./Terms_And_Condition_Technical_Doc.md) — code-level detail

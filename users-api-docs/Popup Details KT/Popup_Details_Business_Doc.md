# Popup Details — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Popup_Details_Technical_Doc.md`](./Popup_Details_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Supplier ko app/website pe kabhi-kabhi ek **popup** dikhaya jaata hai (jaise koi offer,
promotion, ya "MDC"-related prompt), aur system track karta hai ki supplier ne us popup
mein **"interested" mark kiya ya nahi**. Yeh ek lightweight interest-tracking mechanism
hai.

**Business impact**: Marketing/sales teams ke liye ek simple signal hai — "yeh supplier
is popup/offer mein interested hai ya nahi" — jo aage lead-follow-up ya targeting ke liye
use ho sakta hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Popup dekhta hai, "interested" mark kar sakta hai (ya nahi) |
| **Bahut saare internal-apps/systems** | Yeh interest-signal record karne ke liye API call kar sakte hain (bada allowlist hai) |

---

## 3. Business Flow (step-by-step, no code)

```
1. Supplier ko koi popup dikhaya jaata hai (kis screen/app se, yeh scope ke bahar hai)
2. Supplier "interested" (yes/no) select karta hai
3. System is interest-status ko save karta hai — agar pehle se ek record hai, woh
   update ho jaata hai (latest interest-status + timestamp)
4. Agar naya hai, ek fresh record insert hota hai
```

**Business impact**: Ek single, latest-status-wala record maintain hota hai per supplier —
history nahi rakhi jaati, sirf current-status.

---

## 4. Business Rules — Plain Language Mein

1. **`is_interested` sirf 0 ya 1 ho sakta hai.**
2. **Ek supplier ka sirf ek hi latest record hota hai** — dobara submit karne par purana
   overwrite ho jaata hai (history nahi).
3. **Bahut saare internal-systems yeh call kar sakte hain** — ek badi allowlist hai, matlab
   yeh ek widely-integrated tracking-point hai.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine popup pe interested click kiya, phir wapas dekha status change ho gaya"** —
   Expected hai agar dusri baar "not interested" submit hua — sirf latest-status store
   hota hai.

---

## 6. Quick Summary

- Popup Details = simple "interested (0/1)" tracking against supplier, latest-status-only.
- Insert-or-update (upsert), koi history nahi.
- Widely-integrated — bahut saare internal-apps allowed hain.

---

## See also

- [`Popup_Details_Technical_Doc.md`](./Popup_Details_Technical_Doc.md) — code-level detail

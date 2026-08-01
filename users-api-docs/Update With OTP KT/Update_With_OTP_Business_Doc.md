# Update With OTP — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Update_With_OTP_Technical_Doc.md`](./Update_With_OTP_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Kabhi-kabhi supplier apna **sensitive contact-detail** (jaise mobile number ya email)
update karna chahta hai — aisi cheez jo unki identity/account-security se directly juda
ho. Aise sensitive-updates ke liye, IndiaMART **do-factor OTP-verification** require
karta hai — sirf ek OTP nahi, balki **do alag OTPs** (dono ka apna attribute/context)
verify hone ke baad hi actual update hota hai.

**Business impact**: Yeh account-security ka ek strong safeguard hai — koi bhi
mobile/email jaisi identity-defining-detail bina proper OTP-double-verification ke
change nahi ho sakti, jo account-takeover/fraud-risk kam karta hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Apna sensitive-detail update karna chahta hai, OTPs receive/enter karta hai |
| **Internal apps/systems** | Yeh two-step-OTP-verified-update-request bhejte hain |
| **User-Update-API (internal)** | Actual update sirf tabhi execute hota hai jab dono OTPs verify ho jaayein |

---

## 3. Business Flow (step-by-step, no code)

```
1. Supplier apna sensitive-attribute (jaise mobile/email) update karne ka request karta
   hai, do OTPs ke saath (OTP1 aur OTP2 — dono alag-alag context/attribute se linked)
2. System pehle OTP1 verify karta hai — agar galat hai, request yahin reject ho jaata hai
3. OTP1 sahi hone par, system OTP2 verify karta hai — agar galat hai, yahan bhi reject
4. Dono OTPs sahi hone par, aur unka linked-attribute-value bhi match hone par, actual
   update-request internally trigger hoti hai (jaise ek normal profile-update)
5. Update ka final result supplier ko wapas dikhaya jaata hai
```

**Business impact**: Yeh ek "gatekeeper" hai — actual update-logic kahin aur hai (normal
profile-update-flow), yeh feature sirf yeh confirm karta hai ki update genuinely
authorized supplier hi kar raha hai.

---

## 4. Business Rules — Plain Language Mein

1. **Do OTPs mandatory hain, dono apna attribute-context batate hain** — sirf ek OTP se
   kaam nahi chalega.
2. **Sequential verification hoti hai** — pehle OTP1, tabhi OTP2 check hota hai. Agar
   OTP1 fail ho, OTP2 kabhi check hi nahi hoga.
3. **OTP2 ka linked-attribute-value bhi match karna chahiye request ke against** — sirf
   OTP correct hona kaafi nahi, uska context bhi sahi hona chahiye.
4. **Dono OTPs verify hone ke baad hi actual update trigger hota hai** — yeh feature khud
   koi profile-data nahi likhta, sirf verify karke aage bhejta hai.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine sahi OTP1 dala, phir bhi reject ho gaya"** — Ho sakta hai OTP2 galat ho, ya
   OTP2 ka attribute-value request ke saath match na ho.
2. **"OTP expire ho gaya"** — System ko OTP nahi mila (`OTP1 not Found`/`OTP2 not
   Found`) — dobara OTP generate karwana padega (yeh feature khud OTP generate nahi
   karta).

---

## 6. Quick Summary

- Update With OTP = do-factor-OTP-gated sensitive-attribute-update mechanism.
- Sequential verification (OTP1 → OTP2), dono ke attribute-context match hone chahiye.
- Verification pass hone par, actual update ek internal API-call se trigger hoti hai —
  yeh feature khud data write nahi karta.

---

## See also

- [`Update_With_OTP_Technical_Doc.md`](./Update_With_OTP_Technical_Doc.md) — code-level detail

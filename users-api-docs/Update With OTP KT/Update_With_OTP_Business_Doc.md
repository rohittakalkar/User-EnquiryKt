# Update With OTP — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Update_With_OTP_Technical_Doc.md`](./Update_With_OTP_Technical_Doc.md) dekho.

**Note**: Yeh feature khud koi "screen" ya user-facing UI nahi hai — yeh ek backend
**security-gate mechanism** hai jo doosre features (jaise supplier ka mobile/email change karna,
ya kabhi-kabhi GST jaisa locked/verified field override karna) internally use karte hain jab
supplier apni identity strongly prove karna chahta/chahiye ho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Kabhi-kabhi supplier apna **sensitive contact-detail** (jaise mobile number ya email) update
karna chahta hai — aisi cheez jo unki identity/account-security se directly juda ho. Aise
sensitive-updates ke liye, IndiaMART **do-factor OTP-verification** require karta hai — sirf ek
OTP nahi, balki **do alag OTPs** (dono ka apna attribute/context) verify hone ke baad hi actual
update hota hai.

Yeh feature ek **"strong-proof override"** ke roop mein bhi kaam karta hai — jaise GST feature
mein jab ek baar supplier ka GST number "Tactical Verified" ya "OTP Verified" ho jaata hai, toh
normal profile-edit se woh usse change nahi kar sakta (lock lag jaata hai). Us lock ko todne ke
sirf do tarike hain: (a) ek GLADMIN staff-member jiske paas explicit permission ho, ya (b)
supplier khud OTP se apna ownership prove kare — jo isi Update-With-OTP mechanism se hota hai.
[Dekho GST KT ka locking-rule detail](../GST%20KT/GST_Technical_Doc.md).

**Business impact**: Yeh account-security ka ek strong safeguard hai — koi bhi
mobile/email/verified-field jaisi identity-defining-detail bina proper OTP-double-verification
ke change nahi ho sakti, jo account-takeover/fraud-risk kam karta hai, aur saath hi genuine
suppliers ko ek legitimate rasta bhi deta hai apna locked data theek karne ka (bina support-ticket
ke).

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Apna sensitive-detail update karna chahta hai, do OTPs receive/enter karta hai |
| **Internal apps/systems (seller panel, app, etc.)** | Yeh two-step-OTP-verified-update-request bhejte hain backend ko |
| **Update-With-OTP service (internal, backend)** | Dono OTPs verify karta hai; sirf verification-gate hai, khud koi data nahi likhta |
| **Standard User-Update service (internal, backend)** | Update-With-OTP se verify hone ke baad, yehi service asal profile-update karta hai (yeh wahi service hai jo normal profile-edits bhi handle karta hai) |
| **OTP generation/SMS-Email system** | Supplier ko OTP1/OTP2 bhejta hai — **is doc ke code-review scope se bahar hai**, alag system maana gaya hai |

---

## 3. Business Flow (step-by-step, no code)

### Flow A — Normal do-factor OTP verified update

```
1. Supplier apna sensitive-attribute (jaise mobile/email) update karne ka request karta hai, do
   OTPs ke saath (OTP1 aur OTP2 — dono alag-alag context/attribute se linked)
2. System pehle OTP1 verify karta hai
     - agar galat hai ya OTP mila hi nahi (expire ho chuka), request yahin reject ho jaata hai
3. OTP1 sahi hone par, system turant ek internal "audit" event bhi bhej deta hai (background mein)
   — yeh sirf log/record ke liye hai, actual data change abhi nahi hui hai
4. System phir OTP2 verify karta hai us naye value ke against jo supplier set karna chahta hai
     - agar OTP2 galat hai, ya OTP2 ka linked-value request ke naye-value se match nahi karta,
       yahan bhi reject ho jaata hai
5. Dono OTPs sahi hone par, aur unka linked-attribute-value bhi match hone par, actual
   update-request internally trigger hoti hai (normal profile-update service ke through, wahi
   service jo har doosra profile-edit bhi handle karta hai)
6. Update ka final result supplier ko wapas dikhaya jaata hai
```

**Business impact**: Yeh ek "gatekeeper" hai — actual update-logic kahin aur hai (normal
profile-update-flow), yeh feature sirf yeh confirm karta hai ki update genuinely authorized
supplier hi kar raha hai. Do-factor design isliye zaroori hai kyunki agar sirf ek OTP hota, koi
bhi jisne supplier ka ek channel (jaise purana email) compromise kar liya ho, dusra bhi bina extra
proof ke change kar sakta tha.

### Flow B — Locked/verified field ko OTP se override karna

```
1. Supplier ka koi field (jaise GST) already "verified"/"locked" status mein hai — normal edit
   se change nahi ho sakta
2. Supplier isi Update-With-OTP flow se, do OTPs verify karke, apna naya value submit karta hai
3. System verify karta hai ki yeh genuinely wahi supplier hai (do-factor OTP se)
4. Verification pass hone par, locked field bhi update ho jaata hai — yeh ek recognized "strong
   proof" hai jo lock ko override karne ke liye kaafi mana gaya hai
```

**Business impact**: Yeh un genuine suppliers ke liye ek self-service rasta hai jinka verified
data galat ho gaya ho ya jinhe apna number/email change karna ho — bina GLADMIN/support team ko
involve kiye. Same time pe, yeh itna strong-gated hai (do sequential OTPs, value-match check) ki
koi fraud actor asaani se exploit nahi kar sakta.

---

## 4. Business Rules — Plain Language Mein

1. **Do OTPs mandatory hain, dono apna attribute-context batate hain** — sirf ek OTP se kaam
   nahi chalega.
2. **Sequential verification hoti hai** — pehle OTP1, tabhi OTP2 check hota hai. Agar OTP1 fail
   ho, OTP2 kabhi check hi nahi hoga.
3. **OTP2 ka linked-attribute-value bhi match karna chahiye request ke against** — sirf OTP
   correct hona kaafi nahi, uska context (kis value ke liye generate hua tha) bhi request mein
   submit ho rahi actual new-value se match hona chahiye.
4. **Dono OTPs verify hone ke baad hi actual update trigger hota hai** — yeh feature khud koi
   profile-data nahi likhta, sirf verify karke normal profile-update service ko aage bhejta hai.
5. **OTP1 pass hote hi ek internal audit-record turant ban jaata hai** — chahe OTP2 baad mein
   fail ho jaaye. Business ke liye iska matlab: system ke paas "kisi ne OTP1-level tak proof
   diya" ka record hai chahe update poora successful na ho.
6. **Agar OTP request mein value ambiguous ho (jaise ek saath mobile aur email dono submit ho
   jaayein)**, system consistently define nahi karta kaunsa priority lega — practically iska
   matlab hai ek waqt mein ek hi attribute update karna chahiye is flow se.
7. **Yeh feature normal profile-update service ko hi call karta hai final step mein** — yaani
   downstream validations (jo bhi profile-update mein apply hoti hain) yahan bhi apply hongi;
   Update-With-OTP unhe bypass nahi karta, sirf extra OTP-gate add karta hai upar se.

---

## 5. Notifications

| Trigger | Kya bheja jaata hai | Kisko |
|---|---|---|
| OTP1/OTP2 generate hona | OTP code | Supplier ke mobile/email pe (yeh generation-step is feature ka part **nahi** hai — kisi doosre system se aata hai) |
| Verification success (dono OTP pass) | Actual profile-update trigger hoti hai — us update ki apni notification-logic (agar koi ho) normal profile-update service handle karta hai, yeh feature khud koi confirmation-mail/SMS nahi bhejta |

**Business note**: Is feature ka koi apna dedicated "success email/SMS" nahi hai — jo bhi
confirmation supplier ko milta hai, woh downstream normal profile-update service se aata hai, is
OTP-gate se nahi.

---

## 6. Edge Cases — Product/Business Perspective se

1. **"Maine sahi OTP1 dala, phir bhi reject ho gaya"** — Ho sakta hai OTP2 galat ho, ya OTP2 ka
   linked-value request ke saath match na ho (jaise supplier ne OTP kisi purane number ke liye
   generate karaya tha lekin naya different number submit kar raha hai).
2. **"OTP expire ho gaya" / "OTP mila hi nahi"** — System ko OTP nahi mila
   (`OTP1 not Found`/`OTP2 not Found`) — dobara OTP generate karwana padega (yeh feature khud OTP
   generate nahi karta, sirf verify karta hai).
3. **Verification pass ho gaya lekin final update fail ho gaya (rare)** — agar downstream
   profile-update service mein koi aur issue aa jaaye (network/system error), supplier ko poori
   OTP-process dobara karni padegi, kyunki OTPs generally ek-baar-use jaisi hoti hain. Business
   ke liye: agar koi supplier baar-baar complain kare "OTP sahi hai phir bhi update nahi ho raha,"
   yeh scenario check karne layak hai.
4. **Ek request mein multiple fields ek saath try karna** — system sirf ek attribute (jo pehla
   mila usi ko) uthata hai; agar UI/app ne galti se do fields ek saath bhej diye, behavior
   predictable nahi hai. Product/UX side se ensure karna chahiye ki ek waqt mein ek hi
   sensitive-field is flow se update ho.

---

## 7. Quick Summary

- Update With OTP = do-factor-OTP-gated sensitive-attribute-update mechanism, aur locked/verified
  fields (jaise GST) ke liye supplier-side override ka recognized "strong proof" bhi.
- Sequential verification (OTP1 → OTP2), dono ke attribute-context match hone chahiye.
- Verification pass hone par, actual update ek internal call se normal profile-update service ko
  trigger hoti hai — yeh feature khud data write nahi karta, sirf gate hai.
- Koi apna dedicated notification nahi hai — OTP-generation aur post-update-confirmation dono
  is feature ke scope se bahar, doosre systems handle karte hain.

---

## See also

- [`Update_With_OTP_Technical_Doc.md`](./Update_With_OTP_Technical_Doc.md) — code-level detail
- [`../GST KT/GST_Business_Doc.md`](../GST%20KT/GST_Business_Doc.md) — GST ka locking/override
  business-rule, jahan yeh feature ek recognized override-mechanism hai
- [`../Bank Details KT/Bank_Details_Business_Doc.md`](../Bank%20Details%20KT/Bank_Details_Business_Doc.md) —
  ek aur sensitive/verified field-type jiska apna verification-flow hai (alag mechanism, similar
  "trust the strong proof" spirit)

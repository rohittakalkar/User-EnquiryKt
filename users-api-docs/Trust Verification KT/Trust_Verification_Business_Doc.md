# Trust Verification — Business Doc

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Trust_Verification_Technical_Doc.md`](./Trust_Verification_Technical_Doc.md) dekho.

**Scope note**: yeh doc **generic, per-attribute verification framework** cover karta hai —
GST, mobile, email, bank account, PAN, CIN, TAN, address jaisi individual "cheezein" kaise
system mein "verified" mark hoti hain, kaun mark kar sakta hai, aur yeh status kahan-kahan use
hota hai. Yeh [`../TrustSeal KT/`](../TrustSeal%20KT/TrustSeal_Business_Doc.md) se **alag** hai
— TrustSeal ek premium badge hai; yeh system uske neeche ka generic "kya-kya verify hua hai"
tracking mechanism hai. GST verification khud isi generic mechanism ka ek concrete example hai
— dekho [`../GST KT/GST_Business_Doc.md`](../GST%20KT/GST_Business_Doc.md).

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Har supplier ke bahut saare "attributes" hote hain jo verify karne layak hain — mobile number,
email, GST, PAN, CIN, TAN, bank account, address, company name, waghera. Yeh system ek
**generic, reusable mechanism** hai kisi bhi attribute ko "verified" mark karne, unverify
karne, aur uska status wapas dekhne ke liye — bina har attribute-type ke liye alag se ek pura
naya system banaye.

**Business impact**: Buyer ko pata chalta hai kaunse claims verified hain — mobile number sach
mein active hai, GST valid hai, bank account sach mein supplier ka hai, waghera. Yeh trust ka
foundational layer hai jispe TrustSeal jaisi premium-badge cheezein potentially build hoti hain.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Internal automated system/consumer** | Kisi attribute ko verify karta hai jab uska apna trigger complete ho (jaise GST BI-match ho jaana, Android OTP-verify complete hona) |
| **Call-center / support agent (GLADMIN screen se)** | Manually kisi attribute ko phone-call/screen ke through verify kar sakta hai — system "VERIFIED_BY_AGENCY"/"VERIFIED_BY_SCREEN" ke through track karta hai kaunse channel se |
| **Supplier / app (Android auto-flow)** | Kuch cases mein (jaise Android app se email/mobile confirm) supplier ka apna action bhi ek "auto-verify" trigger ban sakta hai |
| **Supplier (end beneficiary)** | Verification ka beneficiary — unka data "verified" dikhta hai baaki systems (jaise buyer-facing profile) ko |
| **Internal tools / other domains (GST, bank, ratings)** | Kisi bhi supplier ke saare verified-attributes ek saath dekh sakte hain, ya apne khud ke verification-events isi mechanism ko "loopback" karke record karte hain |
| **Trust DB (downstream consumer)** | Ek alag physical database jahan verification-events replicate hote hain, taaki trust/compliance-side reads mool system se independent ho sakein |

---

## 3. Business Flows (step-by-step, no code)

### Flow A — Kisi attribute ko verify karna (single ya multiple ek saath)

```
1. Trigger hota hai — jaise koi internal system (GST verification complete), call-center agent
   manually confirm kare, ya supplier khud ek Android-app action se
2. System check karta hai: yeh request bhej rahe caller ko iske liye permission hai ya nahi
   (whitelist-based gateway check — sirf approved internal tools/apps)
3. Agar caller pehle se attribute ki current DB-value bhi bhej raha hai (cross-check ke liye),
   system usse live database ke against match karta hai — mismatch pe error
4. Agar sab sahi hai, verification record ban jaata hai — kisne verify kiya (system/agent ka
   naam), kab, kis channel se (phone/online/android), kaunsa "source" tha, kya comment tha
5. Kuch attributes (mobile=121, email=109) ke liye ek extra confirmation table bhi update
   hoti hai jo un attributes ke "verification date" ko track karti hai
6. Change ko ek queue ke through baaki downstream systems ko bhi bataya jaata hai — jaise
   Trust database mein replicate hota hai
```

**Business impact**: Yeh **single source of truth** hai "kaunsa attribute kab, kisne, kaise
verify kiya" — GST verification, bank verification, mobile/email OTP-verification, sab isi
common mechanism ko reuse karte hain, alag-alag banane ke bajaye.

### Flow B — Verification hata dena ("unverify")

```
1. Koi trigger (jaise underlying data change ho jaana — naya mobile number jud़ jaana) ek
   attribute ko "unverify" karne ka request bhejta hai
2. Kuch specific attributes (mobile, email, aur unke jaisi 6-7 aur) ke liye system pehle check
   karta hai ki koi active OTP-verification record hai ya nahi — agar hai, usse bhi clear
   kiya jaata hai
3. Existing verification record delete ho jaata hai
4. Yeh change bhi downstream systems ko queue ke through notify hota hai
```

**Business impact**: Agar underlying data badal jaaye (jaise supplier apna bank account change
kare), purana verification automatically naye data pe apply nahi hota — explicit unverify-step
hai. Isse galat verified-status kabhi carry-forward nahi hota.

### Flow C — Bulk-verification (ek saath bahut saare attributes)

```
1. Koi internal source (jaise ek batch-reconciliation job) ek saath GLID ke multiple
   attributes verify karna chahta hai
2. Yeh single-attribute path se alag ek dedicated, high-volume-friendly mechanism use karta
   hai
3. Har attribute individually process hota hai, result queue ke through aage bheja jaata hai
```

**Business impact**: High-volume verification scenarios (jaise ek bade reconciliation-run ke
baad) ko single-attribute API ki repeated-call overhead se bachata hai.

### Flow D — Verification-status wapas check karna (read)

```
1. Koi bhi internal tool ek GLID + attribute-IDs ki list deke poochta hai: "yeh verified hain
   kya?"
2. System do tarah ke jawaab de sakta hai:
   a. "Kisi bhi tarah verify hua hai kya" (chahe manual ho ya system-verified) — detail ke
      saath, kisne/kab verify kiya
   b. Attribute-by-attribute ek compact status-list (verified/not) — bina detail ke, jab bulk
      list chahiye ho
3. Mobile-verification ke liye ek extra business rule hai: agar verification 1 saal se purana
   hai, usse "not verified" treat kiya jaata hai — mobile-verification expire hoti hai
```

**Business impact**: Verification "forever valid" nahi hai — kam se kam mobile ke liye,
freshness matter karti hai. Baaki systems (jaise buyer-facing trust-badge) is expiry-rule ko
respect karte hain agar woh iss read-API ko use karte hain.

### Flow E — Trust-DB attribute-verification-details write (alag, adjacent mechanism)

```
1. Koi internal caller ek attribute (primary) aur optionally uske "secondary" related
   attributes (jaise mobile ke saath uska verification-channel) ek saath submit karta hai
2. System check karta hai ki attribute-type ek known/valid type hai (fixed reference-list se)
3. Ek dedicated Trust-database mein directly ek verification-detail record insert/update ho
   jaata hai
```

**Business impact**: Yeh Flow A se **thoda alag, adjacent path** hai — seedha Trust-database
mein likhta hai, bina baaki fan-out (queue-notify) ke. GST domain aur baaki consumers jab
Trust-DB ko sync karna chahte hain, yeh wahi path hai jo woh internally call karte hain
(dekho Technical Doc §7, Flow-diagram "Loopback via Trust-sync queue").

---

## 4. Business Rules — Plain Language Mein

1. **Har attribute-type ka apna numeric ID hai** — yeh ek shared reference-system hai jo
   poore user-domain mein consistent hai (mobile, email, GST, PAN, CIN, TAN, bank account,
   address, company name, waghera sab alag IDs).
2. **Do alag "attribute-ID master lists" hain, confuse hona easy hai** — ek jo main
   verification-write/read APIs use karte hain (`gl_attribute` table-driven, jahan GST=2106
   jaisa GST KT doc mein documented hai), aur ek chhota, alag numbering jo sirf "Trust-DB
   attribute-details" write-path (§3 Flow E) apne andar use karta hai (jahan GST ek alag group
   number "3" hai — dekho Technical Doc §3 aur §14 point 2). Yeh do systems hain, ek nahi.
3. **Verification "kisne kiya" bhi track hota hai** — internal system (jaise "WAPI"/automated)
   vs ek specific call-center agent ka naam/ID, dono cases handle hote hain, saath mein kis
   channel (screen/agency/IP/URL) se aaya woh bhi.
4. **Bulk-verification alag mechanism hai** — ek saath multiple attributes verify karne ke
   liye ek dedicated high-volume path hai, single-attribute verification se alag, lekin dono
   internally same downstream notify-queue share karte hain.
5. **Internal systems isi mechanism ko "loopback" karke bhi use karte hain** — matlab
   background processes (jaise GST-verification consumer) khud iss mechanism ko HTTP-call
   karte hain jab unhe kuch verify karna ho, seedha apna khud ka verification-table nahi
   banate. Poore domain ka single source of truth yehi verification-write API hai.
6. **Mobile-verification 1 saal ke baad expire ho jaati hai** — agar verification-date 365
   din se purani hai, mobile ko "not verified" treat kiya jaata hai read-time pe (record delete
   nahi hota, bas display-status change hota hai).
7. **Kuch attributes automatically "multi-value" treat hote hain** (jaise bank account) — ek
   supplier ke ek se zyada bank accounts ho sakte hain, aur har ek ka apna independent
   verification-status hota hai, ek single flat status nahi.
8. **Android app se email/mobile confirm karna ek special-case hai** — agar request Android
   se aayi ho aur email verify ho rahi ho, system automatically ek "OTP-verified" record bhi
   bana deta hai bina asal OTP flow ke — yeh ek trusted-client shortcut hai.
9. **Verification submit karte waqt agar caller purani value bhi bhejta hai, system usse live
   DB-value se cross-check karta hai** — mismatch hone pe verification fail ho jaata hai,
   taaki purani/galat value verify na ho jaaye.
10. **Sirf whitelisted internal tools/apps hi is mechanism ko call kar sakte hain** — ek
    fixed list of approved caller-identities (jaise MY, GLADMIN, Weberp, LEAP, waghera) hai,
    baaki reject ho jaate hain.

---

## 5. Attribute Verification — Status Overview

| Status | Matlab |
|---|---|
| **Verified** | Ek valid verification-record maujood hai (system ya manual, jo bhi zyada authoritative ho) |
| **Not Verified** | Koi verification-record nahi hai, ya mobile ke case mein record 1 saal se purana ho chuka hai |
| **Tactical-verified vs Manually/OTP-verified** | Do "strength" levels hain — kuch verifications automated/tactical hote hain, kuch manual ya OTP-based; jab dono maujood hon, zyada recent/authoritative wala dikhaya jaata hai (read-API mein explicitly compare hota hai) |

---

## 6. Notifications

| Trigger | Kya hota hai |
|---|---|
| Attribute verify/unverify hone par | Downstream systems (Trust-DB, baaki replicas) ko ek queue-message ke through notify kiya jaata hai — koi direct email/SMS supplier ko is layer se nahi jaata |
| Verification-write ke background failures | Internal alerting (system-team ke liye) — supplier-facing notification nahi |

**Important**: is generic verification-layer se seedha koi supplier-facing email/SMS/push nahi
bhejta — yeh domain-specific hai (jaise GST-verify hone pe email GST ka apna consumer bhejta
hai, dekho GST KT). Yeh layer sirf status-tracking aur downstream-notify karta hai.

---

## 7. Edge Cases — Product/Business Perspective se

1. **"Mera bank account verify ho gaya tha, ab unverified dikha raha hai"** — Verification
   status change ho sakta hai agar underlying data change ho (jaise naya bank account jud़
   gaya) — purana verification naye data pe automatically apply nahi hota, explicit
   unverify-step chalta hai.
2. **"Mera mobile pehle verified tha, ab nahi hai, maine kuch change nahi kiya"** — Mobile
   verification 1 saal ke baad automatically expire ho jaati hai display-purpose ke liye —
   yeh koi bug nahi, ek design-decision hai.
3. **"GST verify hone ke baad TrustSeal kyun nahi mila"** — Yeh do alag systems hain (dekho
   TrustSeal KT doc) — verification aur TrustSeal-assignment automatically linked nahi hain
   code mein.
4. **"Maine ek attribute verify kiya, doosra automatically verify ho gaya"** — Kuch cases mein
   (jaise Android email-confirm) ek action doosre attribute (jaise mobile) ko bhi implicitly
   touch kar sakta hai agar dono same underlying-check share karte hain — support ko dono
   attribute-IDs check karne chahiye.
5. **Do alag "verification write paths" hain jo confuse ho sakte hain** — ek jo poori
   verification-audit-trail banata hai (mool mechanism), doosra jo seedha Trust-database mein
   ek chhota record likhta hai (§3 Flow E) — agar kisi supplier ka data Trust-DB mein kuch
   alag dikhe mool system se, yeh do-paths wali cheez pehle check karo.

---

## 8. Quick Summary

- Yeh ek generic, reusable "kaunsa attribute verified hai" tracking system hai — GST, mobile,
  email, bank, PAN, CIN, TAN, address, sab isi common mechanism se guzarte hain.
- Manual (call-center/agent) aur automated (system-triggered, jaise GST/BI verification)
  dono verification-paths support hote hain.
- Single-attribute aur bulk-verification dono possible hain, ek hi downstream-notify queue
  share karte hue.
- Mobile-verification 1 saal mein expire hoti hai — baaki attributes ke liye aisa expiry-rule
  code mein nahi mila (dekho Open Questions, Technical Doc).
- Ek adjacent, alag write-path (Trust-DB attribute-details) bhi hai jo directly Trust-database
  mein likhta hai — do "verification write" mechanisms hai, ek nahi, confuse na ho.
- TrustSeal se alag hai, lekin conceptually related — verification "neeche ki foundation" hai,
  TrustSeal "upar ka badge".

---

## See also

- [`Trust_Verification_Technical_Doc.md`](./Trust_Verification_Technical_Doc.md) —
  code-level detail
- [`../TrustSeal KT/TrustSeal_Business_Doc.md`](../TrustSeal%20KT/TrustSeal_Business_Doc.md) —
  related but separate: premium trust-badge system
- [`../GST KT/GST_Business_Doc.md`](../GST%20KT/GST_Business_Doc.md) — GST verification is
  isi generic mechanism ka ek concrete example hai; dono docs `USER_VERIFICATION_TRUSTPG` aur
  `USER_VERIFICATION_BULK_HISTORY` share karte hain

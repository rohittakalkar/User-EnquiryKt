# User Blacklist & Blacklist Values — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Blacklist_Technical_Doc.md`](./Blacklist_Technical_Doc.md) dekho.

**Important upfront finding**: "Blacklist" naam ke neeche actually **teen alag data
sources** hain, jo alag-alag purpose serve karte hain lekin naam se overlap lagte hain. Yeh
doc sabko clearly separate karke samjhata hai — confusion na ho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

IndiaMART ko fraud/spam/abusive accounts se platform ko protect karna hota hai. Iske liye
teen alag mechanisms hain:

1. **Manual Blacklist (`GL_BLACKLIST_VALUES`)** — ek admin-maintained list of specific
   values (mobile numbers, emails, ya doosre attributes) jo explicitly flag ki gayi hain.
2. **ML Fraud Suspects (`ML_FRAUD_SUSPECTED_USERS`)** — ek machine-learning model ka output
   — suspicious accounts jo system ne khud detect kiye, based on behavior patterns.
3. **Domain-level blacklist check (`QUERY_APPROVAL_BLACKLIST`)** — real-time check jo
   dusre features (jaise catalog-view tracking) use karte hain yeh dekhne ke liye ki koi
   mobile number pehle se flagged domain/blacklist mein hai.

**Business impact**: Yeh teeno mil ke ek **multi-layered fraud-defense** banate hain —
manual (human-curated), automated (ML-driven), aur real-time-check (inline enforcement).

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Internal admin (GLADMIN)** | Manual blacklist values add/update/whitelist karta hai |
| **ML/Fraud detection system** | Suspicious accounts automatically flag karta hai, koi manual input ke bina |
| **Doosre platform features** (jaise catalog-view tracking) | Real-time blacklist-check consult karte hain lead/activity process karne se pehle |
| **Chat/LMS system** | Conversations se mile signals (jaise suspicious mobile/email chat mein share hua) blacklist mein feed karta hai |

---

## 3. Teeno Mechanisms — Alag-Alag Samjho

### 3.1 — Manual Blacklist Values

**Kya hai**: Ek explicit list of "bad" values — jaise ek mobile number jo pehle fraud mein
use hua tha, ya ek email jo spam ke liye jaana jaata hai.

**Kaun manage karta hai**: Sirf internal admin/BI team (`GLADMIN`, `MAPI` callers) — koi
seller/buyer khud isse touch nahi kar sakta.

**Business flow**:
```
1. Admin ek existing blacklist-value ka status update karta hai (block/unblock)
   — ya ek specific GLID ko explicitly "whitelist" karta hai (override)
2. Value ke against ek GLUSR_ID (kisne yeh value flag ki thi) aur comments record hote hain
3. Doosre systems (jaise chat, catalog-tracking) is value-list ko consult kar sakte hain
```

**Interesting finding**: iss API se koi bhi **naya** blacklist-value add nahi hota, sirf
**existing values ka status update** hota hai. Matlab naye blacklist entries kahin aur se
aate hain — most likely automated systems se (jaise chat-monitoring), na ki iss admin-API se.

### 3.2 — ML Fraud Suspects

**Kya hai**: Ek machine-learning model suspicious accounts ko score karta hai
(`SUSPECT_PROBABILITY`) aur ek list maintain karta hai.

**Kaun manage karta hai**: Yeh poori tarah automated hai — koi manual admin-input iss table
mein likhte hue nahi mila. Ek "exception" concept hai (`FRAUD_SUSPECT_EXCEPTION`) jahan koi
manually kisi ko is suspect-list se exempt kar sakta hai.

**Business flow**:
```
1. ML model background mein accounts ko evaluate karta hai
2. Suspicious lagne wale accounts ML_FRAUD_SUSPECTED_USERS mein record ho jaate hain
   (probability score ke saath)
3. Agar koi false-positive ho, admin ek "exception" add kar sakta hai us GLID ke liye
4. GET /readmlfraud se koi bhi supplier ka current fraud-suspect status check kar sakta hai
```

### 3.3 — Domain-level Real-Time Check

**Kya hai**: Kuch specific features (jaise buyer-activity/catalog-view tracking) directly
`QUERY_APPROVAL_BLACKLIST` table check karte hain — real-time, inline, decision lene ke
liye "yeh process aage badhna chahiye ya nahi."

**Business flow**: Yeh koi standalone feature nahi hai — yeh ek **internal safety-check**
hai jo doosre features (jaise Business Feed/catalog-tracking) apne pipeline ke andar use
karte hain, bina koi alag user-facing API ke.

---

## 4. Business Rules — Plain Language Mein

1. **Blacklist-values sirf update ho sakte hain iss API se, naye add nahi ho sakte** — naye
   entries ka source alag hai (likely automated).
2. **"Whitelist override" ek special path hai** — ek GLID ko explicitly "trusted" mark kiya
   ja sakta hai, jo blacklist-status ko bypass karta hai.
3. **ML Fraud suspects fully automated hain** — koi manual "add karo" action nahi mila,
   sirf "exception/exempt karo" action milta hai.
4. **Teeno systems independent hain, ek doosre ko seedha trigger nahi karte** — matlab agar
   koi value manual-blacklist mein hai, zaroori nahi ki woh ML-fraud-suspect list mein bhi
   ho, ya vice versa.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine ek account ko blacklist mein dekha, lekin fraud-check pass ho gaya"** — Yeh do
   alag systems ho sakte hain (manual blacklist vs ML fraud-suspects) — dono independent
   hain, ek dusre ko automatically sync nahi karte.
2. **"Naya blacklist-value add karna hai, API error de raha hai"** — Expected hai agar
   value already exist nahi karti — current API sirf UPDATE support karta hai, INSERT nahi.
   Naya value add karne ka actual mechanism iss review mein clearly identify nahi hua (likely
   chat/automated-signal se aata hai) — product/eng team se confirm karna chahiye agar
   manual-add ka business-need hai.
3. **Dead/legacy code bhi mila**: ek purana `UserBlacklistController` hai jo ab poori tarah
   deprecated hai (hardcoded "This API is deprecated now" response deta hai) — agar koi
   purani documentation ya integration abhi bhi isse reference kare, woh kaam nahi karegi.

---

## 6. Quick Summary

- "Blacklist" ek naam ke neeche 3 alag systems hain: manual values, ML fraud-suspects,
  aur domain-level real-time check.
- Manual blacklist sirf update hota hai admin se, naya add karne ka path clear nahi hai iss
  review mein.
- ML fraud-suspects poori tarah automated hai, sirf exceptions manual hoti hain.
- Ek purana blacklist-write API completely dead/deprecated hai.

---

## See also

- [`Blacklist_Technical_Doc.md`](./Blacklist_Technical_Doc.md) — code-level detail
- [`../trust_verification_compliance_product_overview.md`](../trust_verification_compliance_product_overview.md) —
  bigger Trust, Verification & Compliance story

# Buyer-Supplier Matchmaking — Business Doc (Product Perspective)

**Note**: yeh doc **Matchmaking** cover karta hai — yeh **Blocking** se alag module hai.
Blocking ke liye [`../Blocking KT/Blocking_Business_Doc.md`](../Blocking%20KT/Blocking_Business_Doc.md)
dekho. Dono buyer-supplier relationship ke around hain, lekin technically independent
features hain, alag tables use karte hain.

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Matchmaking_Technical_Doc.md`](./Matchmaking_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Jab bhi ek buyer kisi supplier se genuinely contact karta hai (enquiry bhejta hai, order
karta hai, ya lead-management system ke through connect hota hai), platform ko yeh record
karna hota hai: **"yeh dono genuinely connect hue hain."**

Yeh **buyer/supplier ki apni request nahi hai** — yeh system khud, background mein, ek "proof
of connection" record banata hai, jab bhi koi real contact-event hota hai. Yeh contact-event
do jagah se aa sakta hai: platform ka apna internal system (jaise enquiry/order), ya bahar se
**Lead-Management System (LMS)** — dono independently is proof-record ko trigger kar sakte
hain.

**Business impact**: Yeh record aage chalke ek **anti-fraud check** ki tarah kaam aata hai —
jab bhi koi rating-related event process hota hai, system pehle yeh check karta hai "kya yeh
buyer sach mein iss supplier se connect hua tha?" Agar matchmaking record nahi milta, rating
ko **turant disable/hide kar diya jaata hai** — ek automated, system-driven review-flag ke
saath, kisi insaan ke intervene kiye bina. Isi tarah, buyer profile page pe supplier ko yeh
bhi dikhta hai ki woh buyer "already connected" hai ya naya (aur yeh signal "blocked?" signal
ke saath ek hi jagah aata hai).

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Buyer** | Contact-event initiate karta hai (enquiry/order/conversation) — matchmaking API seedha buyer call nahi karta |
| **Supplier** | Matchmaking record ka beneficiary — unhe pata chalta hai yeh lead genuine hai |
| **Platform ka internal system ("LEADS" caller)** | Contact-event hote hi matchmaking API ko call karta hai |
| **Lead-Management System (LMS)** | Isi matchmaking data ko apna khud ka, **independent** entry-point use karke bhi bhejta hai — sirf "read" nahi karta, apna khud ka record bhi likh sakta hai |
| **Ratings pipeline (anti-fraud check)** | Baad mein isi record ko verify karta hai — record na mile toh rating ko automatically disable kar deta hai |

---

## 3. Business Flows (step-by-step, no code)

### Flow A — Internal system ke through matchmaking record banta hai

```
1. Buyer ek supplier se contact karta hai — enquiry bhejta hai, order karta hai, ya chat
   shuru karta hai (yeh event khud iss module ke bahar hota hai)
2. Platform ka internal system ("LEADS" caller) yeh event detect karta hai aur matchmaking
   record banane ke liye request bhejta hai
3. System record karta hai: "supplier X, buyer Y, genuinely connect hue"
4. Yeh info async bhi propagate hoti hai — ek "duplicate/mirror" copy, taaki downstream
   systems ke liye reliably available rahe
```

**Business impact**: Buyer/supplier ko yeh sab dikhta hi nahi — poori tarah background mein
chalta hai.

### Flow B — Lead-Management System (LMS) ke through matchmaking record banta hai

```
1. LMS apne aap, independently, ek "buyer-supplier connected" event bhejta hai — yeh Flow A
   se bilkul alag entry-point hai, buyer/supplier ki activity se seedha trigger nahi hota
2. System yeh event validate karta hai (IDs, source info)
3. Valid ho toh record ban jaata hai — same table mein jahan Flow A bhi likhta hai
4. Invalid ho toh event drop ho jaata hai (koi retry nahi is case mein)
```

**Business impact**: Matlab matchmaking sirf ek hi source se nahi banta — LMS bhi
independently "yeh connection genuine hai" bata sakta hai, chahe woh buyer/supplier ki
seedhi platform-activity se na aaya ho.

### Flow C — Rating ke against anti-fraud check

```
1. Koi rating-related event process hota hai (naya rating, ya rating update)
2. System pehle check karta hai — "kya iss buyer-supplier pair ka koi matchmaking record hai?"
3. Record milta hai → sab normal, koi action nahi
4. Record NAHI milta → rating turant "disabled/hidden" status mein daal di jaati hai, ek
   system-generated note ke saath ("Matchmaking record not found") aur ek special
   system-reviewer tag (koi real admin nahi, ek automated marker)
```

**Business impact**: Yeh ek **automatic fraud-guard** hai — fake/self ratings jinke peeche
koi genuine contact-history nahi, unhe manual review ke bina hi block kar diya jaata hai. Yeh
sabse important business-value wala flow hai iss poore module ka.

### Flow D — Supplier buyer-profile dekhta hai

```
1. Supplier ek buyer ka profile open karta hai
2. System ek hi query mein do signals fetch karta hai: "connected?" (matchmaking) aur
   "blocked?" (blocking) — dono ek saath return hote hain
3. Supplier ko dikhta hai ki yeh buyer already-connected hai ya naya, aur agar unhone kabhi
   yeh buyer block kiya tha
```

**Business impact**: Do alag modules (Matchmaking, Blocking) hone ke bawajood, product
experience mein yeh signals ek hi jagah, ek hi screen pe milte hain — supplier ke liye seamless.

---

## 4. Business Rules — Plain Language Mein

1. **Matchmaking record buyer/supplier khud create nahi karte** — yeh ek system-triggered
   event hai jo real contact-activity (Flow A) ya LMS ka apna independent trigger (Flow B) ke
   response mein banta hai.
2. **Do alag entry-points hain isi ek "connection genuine hai" record ke liye** — internal
   system (Flow A) aur LMS (Flow B) — dono independently write kar sakte hain, same table mein.
3. **Ek "dual-side" concept bhi hai** — matlab connection kisi ek taraf se bhi register ho
   sakti hai, ya dono taraf se — is bhed ka exact business-meaning teeno write-paths mein
   thoda alag-alag implement hua paaya gaya (confirm karna baaki hai team se).
4. **Yeh record ek baar ban jaaye, permanent hai** — koi "un-match" karne ka concept nahi mila
   iss review mein (Blocking se ulat, jo reversible hai).
5. **Record duplicate bhi ho sakta hai (bug nahi, by-design duplicate write)** — internal
   system ka event ek baar synchronously likha jaata hai, aur phir wahi record ek baar aur
   async likha jaata hai — dono ek hi jagah jaate hain. Business ke liye iska matlab: agar
   koi issue investigate karna ho, do writes dhoondhna normal hai, ek nahi.
6. **Multiple downstream systems isi single event pe react karte hain** — Rating anti-fraud ke
   liye, aur LMS ke apne lead-tracking ke liye — matlab agar yeh event fail ho jaaye, dono
   downstream effects miss ho sakte hain.
7. **Rating anti-fraud check specific "sources" ko whitelist karta hai** — sirf kuch chuninda
   matchmaking-source-types hi "valid connection" maane jaate hain iss check ke liye; baaki
   sources iss anti-fraud logic mein count nahi hote (exact list confirm karna baaki hai
   team se — Technical Doc §4 dekho).

---

## 5. Notifications

| Event | Kisko notify hota hai | Channel |
|---|---|---|
| Matchmaking record ban gaya | Koi seedha user-facing notification nahi mila iss review mein | — |
| Rating disable hui (anti-fraud) | Koi seedha buyer/supplier-facing notification nahi mila — sirf internal system-log/review-comment | — |

**Note**: yeh module poori tarah background/system-level hai — koi push/email/SMS
notification trace nahi hui iss pass mein Matchmaking-specific flows ke liye.

---

## 6. Edge Cases — Product/Business Perspective se

1. **"Maine ek supplier ko rating diya, lekin woh reject/hide ho gayi"** — Sabse pehla suspect:
   matchmaking record exist hi nahi karta iss buyer-supplier pair ke liye (Flow C). Agar
   original contact-event kabhi register hi nahi hua, ya LMS/internal-system event fail ho
   gaya, Rating pipeline usse fake maan ke automatically disable kar degi.
2. **"Supplier profile pe main dikh raha hoon 'connected' ke roop mein, lekin maine kabhi
   contact nahi kiya"** — Ho sakta hai matchmaking LMS ke through trigger hua ho (Flow B),
   jo buyer/supplier ki seedhi platform-activity se independent hai.
3. **Matchmaking aur Blocking dono ek hi jagah dikhte hain** (Flow D) — supplier jab
   buyer-profile dekhta hai, dono signals ek hi response mein aate hain — do alag modules
   honi ke bawajood, product experience mein yeh ek hi jagah milte hain.
4. **LMS se aaya invalid event silently drop ho jaata hai** — koi retry ya alert nahi jaata
   agar LMS ka bheja hua event malformed ho (missing/invalid fields) — business ko yeh pata
   nahi chalega ki ek expected connection-record miss ho gaya, jab tak koi doosra symptom
   (jaise rating-disable) na dikhe.

---

## 7. Quick Summary

- Matchmaking = system ka "yeh connection genuine hai" proof, buyer/supplier ki apni action
  nahi.
- Do independent trigger-sources hain: platform ka internal system, aur LMS.
- Ratings ke fake-review-protection ka backbone hai — record na mile toh rating auto-disable.
- Buyer-profile page pe Blocking ke saath ek hi jagah dikhta hai.
- Permanent record hai, "un-match" jaisa concept nahi.
- By-design duplicate-write hai (ek hi record do baar likha jaata hai) — bug nahi.

---

## See also

- [`Matchmaking_Technical_Doc.md`](./Matchmaking_Technical_Doc.md) — code-level detail
- [`../Blocking KT/Blocking_Business_Doc.md`](../Blocking%20KT/Blocking_Business_Doc.md) —
  related lekin independent module
- [`../Rating KT/Rating_Business_Doc.md`](../Rating%20KT/Rating_Business_Doc.md) — jahan
  matchmaking record anti-fraud check ke liye consult hota hai
- [`../buyer_seller_discovery_matching_product_overview.md`](../buyer_seller_discovery_matching_product_overview.md) —
  bigger Buyer-Seller Discovery & Matching story

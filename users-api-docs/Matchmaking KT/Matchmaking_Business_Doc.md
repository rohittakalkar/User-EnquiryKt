# Buyer-Supplier Matchmaking — Business Doc (Product Perspective)

**Note**: yeh doc **Matchmaking** cover karta hai — yeh **Blocking** se alag module hai.
Blocking ke liye [`../Blocking KT/Blocking_Business_Doc.md`](../Blocking%20KT/Blocking_Business_Doc.md)
dekho. Dono buyer-supplier relationship ke around hain, lekin technically independent
features hain, alag tables use karte hain.

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Matchmaking_Technical_Doc.md`](./Matchmaking_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Jab bhi ek buyer kisi supplier se contact karta hai (enquiry bhejta hai, order karta hai, ya
conversation karta hai), platform ko yeh record karna hota hai: **"yeh dono genuinely
connect hue hain."**

Yeh **buyer/supplier ki apni request nahi hai** — yeh system khud, background mein, ek
"proof of connection" record banata hai, jab bhi koi real contact-event hota hai.

**Business impact**: Yeh record aage chalke ek **anti-fraud check** ki tarah kaam aata hai —
jab koi buyer supplier ko rating deta hai, system pehle yeh check karta hai "kya yeh buyer
sach mein iss supplier se connect hua tha?" Agar matchmaking record nahi milta, rating fake
maan ke reject/hide ho sakti hai. Isi tarah, buyer profile page pe supplier ko yeh bhi dikhta
hai ki woh buyer "already connected" hai ya naya.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Buyer** | Contact-event initiate karta hai (enquiry/order/conversation) — matchmaking API seedha buyer call nahi karta |
| **Supplier** | Matchmaking record ka beneficiary — unhe pata chalta hai yeh lead genuine hai |
| **Platform ka internal system** | Contact-event hote hi matchmaking API ko call karta hai |
| **Ratings pipeline** | Baad mein isi record ko verify karta hai fake-rating rokne ke liye |
| **Lead-Management System (LMS)** | Isi matchmaking data ko apne lead-tracking ke liye bhi use karta hai |

---

## 3. Main Business Flow (step-by-step, no code)

```
1. Buyer ek supplier se contact karta hai — enquiry bhejta hai, order karta hai, ya chat
   shuru karta hai (yeh event khud iss module ke bahar hota hai)
2. Platform ka internal system yeh event detect karta hai aur matchmaking record banane ke
   liye request bhejta hai
3. System record karta hai: "buyer X, supplier Y, genuinely connect hue"
4. Yeh info do jagah async propagate hoti hai — ek copy jo baad mein Ratings pipeline check
   karti hai, aur ek copy jo Lead-Management System ke liye hai
5. Ab agar yeh buyer kabhi rating de, ya supplier iss buyer ka profile dekhe, system ko pata
   hai yeh ek verified connection hai
```

**Business impact**: Yeh sab **buyer/supplier ko dikhta hi nahi** — poori tarah background
mein chalta hai. Iska fayda tabhi visible hota hai jab koi doosra feature (Ratings, Buyer
Profile) isse consult kare.

---

## 4. Business Rules — Plain Language Mein

1. **Matchmaking record buyer/supplier khud create nahi karte** — yeh ek system-triggered
   event hai jo real contact-activity ke response mein banta hai.
2. **Dono directions track ho sakti hain** — matlab agar zaroorat pade, ek "dual-side" flag
   bhi hai jo batata hai ki kya connection dono taraf se confirm hui.
3. **Yeh record ek baar ban jaaye, permanent hai** — koi "un-match" karne ka concept nahi
   mila iss review mein (Blocking se ulat, jo reversible hai).
4. **Multiple downstream systems isi single event pe react karte hain** — Ratings ke liye
   ek copy, LMS ke liye ek copy — matlab agar yeh event fail ho jaaye, dono downstream
   effects miss ho sakte hain.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine ek supplier ko rating diya, lekin woh reject/hide ho gayi"** — Sabse pehla
   suspect: matchmaking record exist hi nahi karta iss buyer-supplier pair ke liye. Agar
   original contact-event kabhi register hi nahi hua (ya system-level fail ho gaya), Ratings
   pipeline usse fake maan ke rok degi. Yeh Rating KT doc mein bhi documented hai.
2. **"Supplier profile pe main dikh raha hoon 'connected' ke roop mein, lekin maine kabhi
   contact nahi kiya"** — Ho sakta hai matchmaking galat trigger hua ho, ya koi doosra
   process (jaise automated lead-assignment) ne bhi ek connection record kiya ho.
3. **Matchmaking aur Blocking dono ek hi jagah dikhte hain** — supplier jab buyer-profile
   dekhta hai, dono signals (connected? blocked?) ek hi response mein aate hain — do alag
   modules honi ke bawajood, product experience mein yeh ek hi jagah milte hain.

---

## 6. Quick Summary

- Matchmaking = system ka "yeh connection genuine hai" proof, buyer/supplier ki apni action
  nahi.
- Ratings ke fake-review-protection ka backbone hai.
- LMS (internal lead-tracking) ko bhi yehi data feed hoti hai.
- Permanent record hai, "un-match" jaisa concept nahi.

---

## See also

- [`Matchmaking_Technical_Doc.md`](./Matchmaking_Technical_Doc.md) — code-level detail
- [`../Blocking KT/Blocking_Business_Doc.md`](../Blocking%20KT/Blocking_Business_Doc.md) —
  related lekin independent module
- [`../Rating KT/Rating_Business_Doc.md`](../Rating%20KT/Rating_Business_Doc.md) — jahan
  matchmaking record anti-fraud check ke liye consult hota hai
- [`../buyer_seller_discovery_matching_product_overview.md`](../buyer_seller_discovery_matching_product_overview.md) —
  bigger Buyer-Seller Discovery & Matching story

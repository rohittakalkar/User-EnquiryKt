# Buyer Blocking — Business Doc (Product Perspective)

**Note**: yeh doc **Blocking** cover karta hai — yeh **Matchmaking** se alag module hai.
Matchmaking ke liye [`../Matchmaking KT/Matchmaking_Business_Doc.md`](../Matchmaking%20KT/Matchmaking_Business_Doc.md)
dekho.

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Blocking_Technical_Doc.md`](./Blocking_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Kabhi-kabhi ek user (supplier ya buyer, dono) ko dusre particular user se contact nahi chahiye
hoti — ho sakta hai woh abusive ho, spam kar raha ho, ya bas ek unwanted repeat-caller ho.
Blocking feature user ko yeh control deta hai: **"iss dusre user se mujhe kabhi contact nahi
chahiye."**

**Business impact**: Yeh user ka apna, direct control hai — Matchmaking (jo system
automatically karta hai) ke ulat, Blocking hamesha **ek manual, active decision** hai.

**Deeper pass se naya insight**: code se pata chala ki yeh table/mechanism dono directions
mein use hoti hai — jab code review kiya, confirmed enforcement actually **buyer-supplier
ko block karta hai** (buyer decide karta hai ki koi specific supplier unhe contact na kare),
na ki sirf "supplier blocks buyer." Dono directions ka underlying data-model same hai (ek hi
table, generic actor/target columns) — dekho section 3 aur 7.

---

## 2. Kaun-kaun involved hai iss process mein

| Kaun | Role |
|---|---|
| **Action lene wala user** (supplier ya buyer, koi bhi) | Kisi dusre user ko block ya unblock karta hai — yeh unka apna active decision hai |
| **Jise block kiya gaya user** | Iska pata nahi chalta unhe (silent block) — koi notification nahi jaati |
| **Buyer-Profile screen (confirmed enforcement point)** | Jab supplier ek buyer ka profile dekhta hai, aur us buyer ne supplier ko block kiya ho, buyer ke contact-details aur verification-status **blank/hidden** aa jaate hain us supplier ke liye |
| **Lead-delivery/routing system** *(scope se bahar)* | Kya "supplier blocks buyer" direction ka bhi koi enforcement hai kisi lead-routing system mein, yeh iss review mein confirm nahi ho paaya — dekho section 7 Open Point |

---

## 3. Main Business Flow (step-by-step, no code)

### Flow A — Ek user dusre ko block karta hai

```
1. User (supplier ya buyer) kisi dusre user ko block karta hai (jaise ek unwanted
   lead/contact ke response mein)
2. System yeh record kar leta hai: "user X ne user Y ko block kiya"
3. Agar user baad mein mann badle, wahi action "unblock" ke saath dobara call ki jaa
   sakti hai — record update ho jaata hai, delete nahi hota (history reh jaati hai kab
   block/unblock hua)
```

**Business impact**: Yeh ek simple, instant, reversible action hai — koi approval-workflow
ya delay nahi, decision turant apply hota hai.

### Flow B — Block hone ka asar (jab buyer ne supplier ko block kiya ho)

```
1. Buyer ne kisi supplier ko block kar rakha hai
2. Wahi supplier baad mein us buyer ka profile dekhne ki koshish karta hai
   (seller panel mein "Buyer Profile" screen se)
3. System profile dikhata hai, lekin buyer ke contact-details (mobile, email, landline,
   verification-status) sab BLANK/hidden aate hain
4. Baaki profile-info (company-type, activity-stats waghera) normal dikhti hai — sirf
   contact/verification-info hide hoti hai
```

**Business impact**: Yeh asli fraud/harassment-prevention hai — agar ek buyer kisi supplier
se pareshan hai aur usse block kar deta hai, woh supplier us buyer se dobara contact karne
layak jaankari hi nahi dekh paata. Yeh confirmed, code-verified enforcement hai (pehle iss
doc mein "enforcement point missing" bola gaya tha — deeper trace se yeh mil gaya).

**Open point**: reverse-direction ("supplier block kare buyer ko, aur uska koi
enforced/visible effect ho jaise lead na milna") — iska koi confirmed enforcement point iss
review mein nahi mila. Ho sakta hai woh kisi alag lead-routing system mein ho jo iss
codebase review ke scope se bahar hai. **[INFERRED — team se confirm karo]**

---

## 4. Business Rules — Plain Language Mein

1. **Block aur Unblock same API se hote hain** — ek `blocked_status` flag (`1` ya `0`) decide
   karta hai kaunsa action hai.
2. **History preserve hoti hai** — jab bhi block ya unblock hota hai, us action ki date
   (`blocking_date` ya `unblocking_date`) alag-alag record hoti hai — dono columns independent
   hain, ek dusre ko overwrite nahi karte. Matlab tumhe "pehli baar kab block hua tha" aur
   "aakhri baar kab unblock hua tha" dono ka trail milta hai, chahe current status kuch bhi ho.
3. **Yeh action koi bhi user le sakta hai** — supplier ya buyer, dono. Koi role-restriction
   nahi hai jo iss feature ko sirf supplier tak limit kare — code-level review confirm karta
   hai ki mechanism generic hai (actor/target dono generic "user" hain).
4. **Buyer ko notify nahi kiya jaata** — silent hai, jaisa aam taur pe blocking features
   platforms pe hoti hain.
5. **Confirmed effect**: agar buyer ne supplier ko block kiya hai, us supplier ko us buyer ki
   contact/verification details profile-view mein blank dikhengi (Flow B). Baaki profile-info
   normal rehta hai.
6. **Field validation strict hai**: dono user-IDs numeric hone chahiye, block/unblock flag
   sirf `1` ya `0` ho sakta hai — invalid value ya khaali field pe poora request reject ho
   jaata hai clear error message ke saath.

---

## 5. Notifications — User ko kab pata chalta hai

| Kab | Kya notification jaata hai |
|---|---|
| Koi user block/unblock hua | **Kuch nahi** — na jisne block kiya use, na jise block kiya gaya |
| Buyer ne supplier ko block kiya, aur supplier profile dekhta hai | Koi explicit "aap blocked hain" message nahi — bas contact-details silently blank dikhte hain (indirect signal, agar supplier notice kare) |

Yeh feature poori tarah **silent** hai — GST ya Rating jaise domains ke ulat, jahan email
confirmation jaata hai, Blocking mein koi bhi notification-channel involve nahi hai.

---

## 6. Edge Cases — Product/Business Perspective se

1. **"Maine buyer ko block kiya, phir bhi mujhe unse messages/leads mil rahe hain"** —
   Confirmed code-review se: block ka effect sirf **Buyer-Profile screen** pe dikhta hai
   (contact-details hide) jab **buyer ne supplier ko block** kiya ho. Reverse-direction
   (supplier-blocks-buyer ka lead-delivery pe effect) kis system mein enforce hoti hai, agar
   hoti hai, yeh confirm nahi ho paaya — is scenario ke liye alag investigation chahiye.
2. **"Maine unblock kiya, lekin purana block history dikh rahi hai kahin"** — Expected hai,
   history preserve hoti hai (section 4, point 2) — sirf current `block_status` change hota
   hai, purani `blocking_date` timestamp record mein reh jaati hai.
3. **"Maine ek supplier/buyer ko block kiya, unhe pata chal gaya kya?"** — Nahi, block poori
   tarah silent hai (section 5). Woh sirf indirectly notice kar sakte hain agar unki
   contact-details kahin blank dikhne lage.
4. **"Kya main kisi ko block karne ki limit se bandha hoon?"** — Code mein koi such limit
   nahi mili — koi bhi user kitne bhi logon ko block kar sakta hai, koi cap nahi hai.

---

## 7. Quick Summary — Ek Line Mein Har Cheez

- Blocking = koi bhi user ka apna, manual, reversible decision — "iss dusre user se contact
  nahi chahiye" — supplier ya buyer dono ise le sakte hain.
- Matchmaking se bilkul independent hai, alag purpose (system-verification vs
  user-choice).
- Simple upsert-based record, koi async pipeline/queue/cron involved nahi — sabse simple
  feature poore KT-series mein.
- **Confirmed enforcement**: buyer ne supplier ko block kiya ho toh, Buyer-Profile screen pe
  us supplier ko buyer ke contact-details blank dikhte hain.
- **Open point**: reverse-direction (supplier-blocks-buyer) ka enforcement kahin confirm
  nahi hua iss review mein — team se confirm karna hoga.
- Koi notification kisi ko nahi jaati, poora silent hai.

---

## See also

- [`Blocking_Technical_Doc.md`](./Blocking_Technical_Doc.md) — code-level detail
- [`../Matchmaking KT/Matchmaking_Business_Doc.md`](../Matchmaking%20KT/Matchmaking_Business_Doc.md) —
  related lekin independent module
- [`../buyer_seller_discovery_matching_product_overview.md`](../buyer_seller_discovery_matching_product_overview.md) —
  bigger Buyer-Seller Discovery & Matching story

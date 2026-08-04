# Social Contacts — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Social_Contacts_Technical_Doc.md`](./Social_Contacts_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Supplier apne **personal/social-contact details** — Facebook, LinkedIn, Twitter, Google+,
Instagram, Klout profile-URLs, gender, aur WhatsApp/UPI-active-status — IndiaMART profile ke
saath save kar sakta hai. Yeh Marketplace-URL (Google/FB/Instagram business-presence) se
alag hai — yeh supplier ke **personal social-network profiles** aur **communication-
preferences** (WhatsApp/UPI available hai ya nahi) capture karta hai.

**Business impact**: Yeh data buyer-supplier communication-channels enrich karta hai
(WhatsApp-availability jaisa flag directly contact-experience improve karta hai), aur
supplier ki digital-presence ko profile mein integrate karta hai. Buyer jab supplier ka
profile dekhta hai (matchmaking/enquiry flow mein), unhe LinkedIn-link aur WhatsApp-
availability seedha wahin dikh jaate hain.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Apne social-profile-links, gender, WhatsApp/UPI-status update karta hai |
| **Internal tools/apps (GLADMIN, SELLERMY, IMOB, MSITE)** | Supplier ki taraf se yeh data update kar sakte hain, ek dedicated API ke through |
| **Internal background systems (3 alag automated paths)** | Supplier ki taraf se yehi data 2 alag automated/background tarikon se bhi update ho sakta hai — dekho section 3 |
| **Buyer** | Profile dekhte waqt yeh details (embedded) access kar sakta hai — do alag jagah pe (poora profile-detail screen, aur matchmaking/buyer-profile screen) |

---

## 3. Business Flow (step-by-step, no code)

Yeh feature interesting isliye hai kyunki iske andar data update hone ke **teen alag
tarike** hain, jo ek business-user ke liye normally invisible hote hain lekin support/
debugging ke liye jaanna zaroori hai.

### Flow A — Supplier (ya allowed internal-app) directly update karta hai

```
1. Supplier (ya koi allowed internal-app — GLADMIN/SELLERMY/IMOB/MSITE) social-contact
   details submit karta hai — "insert" (naya) ya "update" (existing) action ke saath
2. System validate karta hai — gender sirf Male/Female, WhatsApp-active sirf 0/1
3. Naya record insert hota hai, ya existing record ke sirf diye-gaye fields update hote
   hain (partial-update)
4. WhatsApp-active-status change hone par, "kab mark hua" bhi track hota hai
```

**Business impact**: Yeh sabse direct, real-time control hai jo supplier/internal-app ke
paas apne social data ke upar hai.

### Flow B — Background/automated system independently update karta hai (dedicated path)

```
1. Koi internal background system (jiska exact source in-repo trace nahi ho paaya — dekho
   Technical Doc Open Questions) is data ko independently update kar sakta hai
2. Yeh path sabse zyada fields cover karta hai — Instagram-link aur UPI-active-status
   **sirf yehi path** set kar sakta hai, koi doosra tarika nahi
3. "Kisne update kiya" (naam, screen, IP) bhi track hota hai is path mein
```

**Business impact**: Agar Instagram-URL ya UPI-status kabhi update nahi ho raha, iska
matlab hai woh sirf iss background-path se aata hai — normal supplier-facing update-screen
se nahi ho sakta (product/support team ko yeh samajhna zaroori hai).

### Flow C — General profile-update ka side-effect (generic path)

```
1. Jab bhi supplier apna koi bhi general profile-detail update karta hai (naam, address,
   ya kuch aur — social-specific screen nahi), background mein ek generic "poora profile
   update ho gaya" event fire hota hai
2. Agar us event ke andar gender/social-URL fields bhi shaamil hain, yeh event apne aap
   inn fields ko bhi update kar deta hai — supplier ko pata bhi nahi chalta ki yeh alag se
   ho raha hai
3. Yeh path sabse **kam fields** cover karta hai — WhatsApp/UPI/Instagram is path se kabhi
   update nahi hote, sirf gender + basic social-URLs
```

**Business impact**: Yeh ek background "side-effect" hai jo kisi bhi general profile-edit
ke saath silently ho sakta hai. Agar koi supplier bole "maine sirf apna address update
kiya, lekin gender bhi change ho gaya," yeh iska technical explanation ho sakta hai.

**Overall business rule jo yaad rakhne layak hai**: In teeno paths mein se koi bhi ek hi
supplier ke record ko update kar sakta hai, alag-alag waqt pe, alag-alag fields ke saath.
Yeh koi ek hi cheez ke teen naam nahi hain — teeno genuinely alag triggers hain (Technical
Doc mein detail se verify kiya gaya hai).

### Flow D — Buyer profile-detail dekhta hai (read, no dedicated screen)

```
1. Buyer supplier ka profile-detail dekhta hai — yeh data automatically embedded milta hai
   (koi alag "view social contacts" screen nahi hai)
2. Agar buyer specifically matchmaking/enquiry-context ke through supplier ka profile
   dekhta hai, ek THODA alag, chhota subset dikhta hai (sirf LinkedIn-link + WhatsApp-
   available-status) — yeh doosra, alag read-jagah hai
```

**Business impact**: Ek lightweight, supporting-data feature hai jo do alag buyer-facing
screens ka silently hissa ban jaata hai.

---

## 4. Business Rules — Plain Language Mein

1. **Insert ke liye kam-se-kam 2 meaningful values honi chahiye** — sirf ID de kar empty
   record nahi bana sakte (yeh sirf direct-supplier-facing path pe apply hota hai).
2. **Gender sirf "Male"/"Female" accept hota hai** (case-insensitive check hai) —
   direct-supplier-facing path mein.
3. **WhatsApp-active-flag sirf 0 ya 1 ho sakta hai.**
4. **Update mein sirf diye-gaye fields hi change hote hain** — baaki untouched rehte hain
   (partial-update supported hai, direct path mein).
5. **Do independent background/automated paths bhi hain** jo yahi table update kar sakte
   hain — ek widest-coverage (Instagram/UPI/audit-trail samet), ek narrowest-coverage
   (sirf gender + basic URLs, kisi general profile-update ke side-effect ke roop mein).
6. **Instagram-URL aur UPI-active-status sirf background-path se set ho sakte hain** —
   normal direct-update path se yeh do fields kabhi change nahi honge.
7. **Buyer ko dikhne wala data do alag jagah se aata hai**, alag-alag subset ke saath —
   poora profile-detail screen sabse zyada fields dikhata hai (whatsapp/upi/instagram
   chhod ke), matchmaking/buyer-profile screen sirf LinkedIn + WhatsApp-status dikhata hai.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine gender update kiya, error aaya"** — Expected hai agar value "Male"/"Female" ke
   alawa kuch bheja gaya.
2. **"WhatsApp-active status change hua, date bhi update hui"** — Expected hai, system
   automatically "kab mark hua" timestamp record karta hai.
3. **"Maine apna profile update kiya, gender bhi change ho gaya, maine toh gender nahi
   chheda tha"** — Ho sakta hai general profile-update ke background side-effect se aisa hua
   ho (Flow C) — agar supplier ke general-update-request mein purani/stale gender value
   thi, woh silently apply ho sakti hai.
4. **"Instagram link kabhi save nahi ho raha"** — Expected hai agar sirf normal
   supplier-facing update-screen use ho raha ho; yeh field sirf background-path se set hota
   hai, aur currently kisi bhi buyer-facing screen mein wapas dikhta bhi nahi (Technical Doc
   Edge Case #1) — flag karne layak agar business isse dikhana chahti hai.
5. **"Do alag jagah mera social data dikh raha hai, dono match nahi karte"** — Expected hai:
   poora profile-detail response aur matchmaking/buyer-profile response do alag query se
   aate hain, alag subset of fields dikhate hain (poori list vs. sirf LinkedIn+WhatsApp).

---

## 6. Quick Summary

- Social Contacts = supplier ke personal social-profile-links + gender + WhatsApp/UPI
  communication-preference-flags.
- **Teen alag update-paths hain**: ek direct supplier/internal-app-facing, do background/
  automated (alag-alag field-coverage ke saath) — yeh koi bug nahi, genuinely teen alag
  design-hai.
- Insert/update, partial-update supported (direct path mein).
- Koi dedicated read-screen nahi — **do alag jagah embedded** milta hai (poora
  profile-detail, aur matchmaking/buyer-profile — dono alag field-subset dikhate hain).
- Instagram-URL/UPI-status sirf ek specific background-path se set ho sakte hain.
- Marketplace-URL (business-presence-links) se alag concept hai.

---

## See also

- [`Social_Contacts_Technical_Doc.md`](./Social_Contacts_Technical_Doc.md) — code-level detail
- [`../Market Place URL KT/MarketPlaceUrl_Business_Doc.md`](../Market%20Place%20URL%20KT/MarketPlaceUrl_Business_Doc.md) —
  related but separate business-presence-URL concept

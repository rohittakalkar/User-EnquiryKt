# Market Place URL — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`MarketPlaceUrl_Technical_Doc.md`](./MarketPlaceUrl_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Supplier apni company ke **social/marketplace presence ke URLs** (Google, Facebook,
Instagram) IndiaMART profile ke saath link kar sakta hai. Yeh buyer ko supplier ki
credibility/legitimacy judge karne mein madad karta hai — agar supplier ka established
Google-listing, Facebook-page, ya Instagram-account hai, toh yeh trust-signal ban sakta hai.

**Business impact**: "Third-party presence verification"-jaisa concept — jitni jagah
supplier discoverable/verifiable hai, utna hi buyer-confidence better hota hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Apne marketplace URLs (Google/Facebook/Instagram) add/update karta hai |
| **Internal tools/admin (GLADMIN, WebERP)** | Supplier ki taraf se bhi yeh URLs update/enrich kar sakte hain |
| **Buyer** | Supplier profile pe yeh links dekh sakta hai |

---

## 3. Business Flow (step-by-step, no code)

```
1. Supplier (ya internal admin/tool) ek marketplace-URL submit karta hai — sirf teen
   platforms support hote hain: Google, Facebook, Instagram
2. System check karta hai: is domain ke liye already ek URL-record exist karta hai ya
   nahi
   - Agar pehli baar hai → naya record insert hota hai
   - Agar already hai → existing record update hota hai
3. Har change ka history/comments-trail bhi record hota hai
4. Buyer-facing profile pe latest URL (per domain) dikhta hai
```

**Business impact**: Ek simple 3-platform-limited linking feature, but history-tracking
ke saath — kisne, kab, kaha se update kiya, yeh sab capture hota hai.

---

## 4. Business Rules — Plain Language Mein

1. **Sirf 3 platforms supported hain** — Google, Facebook, Instagram. Koi aur domain-name
   diya jaaye toh system usse "0" (unrecognized) sequence-number assign karta hai.
2. **Naya URL add karne ke liye URL value mandatory hai**; update ke case mein URL
   optional ho sakta hai (sirf metadata bhi update ho sakti hai).
3. **Ek user ke ek domain ka sirf ek "latest" URL dikhta hai buyer ko** — agar history
   mein multiple updates hue hon, sabse recent wala hi surface hota hai.
4. **Sirf allowed internal-systems/apps hi is data ko likh sakte hain** — supplier ka
   khud ka app (`MY`), admin tools (`GLADMIN`, `WebERP`), seller-app (`SELLERMY`).

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine Instagram URL update kiya, purana wala dikh raha hai"** — Ho sakta hai
   propagation-delay ho, ya duplicate historical-record confusion ho — dekho Technical
   Doc ka "latest per domain" dedup-logic.
2. **"Maine LinkedIn ka URL add karne ki koshish ki, allow nahi hua"** — Expected hai,
   sirf Google/Facebook/Instagram supported hain abhi.

---

## 6. Quick Summary

- Marketplace-URL = supplier ke Google/Facebook/Instagram presence links.
- Insert-or-update per domain, history/comments trail ke saath.
- Sirf 3 platforms support, koi aur domain unrecognized treat hota hai.

---

## See also

- [`MarketPlaceUrl_Technical_Doc.md`](./MarketPlaceUrl_Technical_Doc.md) — code-level detail

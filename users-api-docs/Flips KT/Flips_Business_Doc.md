# Flips (Social-Commerce Video) — Business Doc

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Flips_Technical_Doc.md`](./Flips_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

**Flips** IndiaMART ka short-form video/social-commerce feature hai — jaise Instagram
Reels/TikTok, lekin B2B products ke liye. Supplier apna Flips (external vendor-platform)
account link karta hai, aur unka video-content IndiaMART pe bhi surface hota hai.

**Business impact**: Video-content buyer-engagement ke liye zyada compelling hota hai text/
photo se — Flips supplier ko apne products ko dynamically showcase karne deta hai, bina
IndiaMART ko khud ek video-hosting-platform banaye (yeh kaam external vendor karta hai).

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Apna Flips/vendor-account link karta hai, video-content post karta hai |
| **External Flips vendor-platform** | Actual video-hosting/processing (IndiaMART ke bahar) |
| **Buyer** | Video-content dekhta hai supplier ke storefront pe |

---

## 3. Business Flow (step-by-step, no code)

### Account linking
```
1. Supplier apna account link karta hai — do tarike se possible: ek "Token"-based auth
   (jaise direct vendor-API-token), ya "Meta"-based auth (jaise Facebook/Instagram
   business-login se)
2. System check karta hai ki yeh social-account already kisi doosre linked-account se
   duplicate toh nahi hai (similarity-score based check mila code mein)
3. Account-detail record banta hai, "disabled" flag ke saath track hota hai (agar kabhi
   disable karna pade)
```

### Content posting
```
1. Supplier (ya unki taraf se ek automated sync) naya video-content post karta hai
2. Content ek naye record ke roop mein save hota hai, ya existing content update hoti hai
3. Yeh event downstream systems ko bhi notify hota hai — ek "sync" process bhi hai jo
   isi tarah ke updates independently bhi apply kar sakta hai (backup/reconciliation jaisa)
```

**Business impact**: Do-step process (account phir content) — TrustSeal/Social-Review jaisa
hi pattern.

---

## 4. Business Rules — Plain Language Mein

1. **Do authentication-modes support hote hain** — "Token" ya "Meta" (Facebook-jaisa
   business-login), supplier/integration jo bhi use kare.
2. **Duplicate-account detection hai** — agar similarity-score kisi threshold pe match
   kare, system flag kar sakta hai.
3. **Account-disable ek soft-flag hai**, poora record delete nahi hota.
4. **Content-writes do jagah se ho sakte hain** — seedha API call se, ya ek background
   sync-process se — dono end mein same tables update karte hain (redundancy/reconciliation
   ke liye).

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine content post kiya, dobara dikh raha hai duplicate"** — Ho sakta hai dono paths
   (direct + background sync) ek hi content ko independently process kar rahe hon — check
   karo dono writes same unique-identifier use kar rahe hain ya nahi.
2. **"Mera Flips account disabled ho gaya, kyun?"** — Duplicate-detection (similarity-score)
   ki wajah se ho sakta hai, ya manual admin-action se.

---

## 6. Quick Summary

- Flips = short-video/social-commerce, external-vendor-hosted, IndiaMART pe surface hoti hai.
- Account-link → Content-post, do-step process.
- Do auth-modes (Token/Meta), duplicate-detection built-in.
- Content-writes do independent paths se ho sakte hain (direct API + background sync).

---

## See also

- [`Flips_Technical_Doc.md`](./Flips_Technical_Doc.md) — code-level detail
- [`../ecommerce_social_commerce_product_overview.md`](../ecommerce_social_commerce_product_overview.md) —
  bigger Ecommerce & Social-Commerce story, Flips originally documented yahan summary-level

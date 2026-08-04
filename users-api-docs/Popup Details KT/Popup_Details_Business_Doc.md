# Popup Details — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai — koi code nahi, sirf "kya hota hai, kyun
hota hai, aur supplier/business ke liye iska matlab kya hai." Code-level detail ke liye
[`Popup_Details_Technical_Doc.md`](./Popup_Details_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Supplier ko app/website pe kabhi-kabhi ek **popup** dikhaya jaata hai (jaise koi offer,
promotion, ya "MDC"-related prompt — exact popup ka business-context code se resolve nahi
hota, dekho Edge Cases), aur system track karta hai ki supplier ne us popup mein **"interested"
mark kiya ya nahi**. Yeh ek lightweight interest-tracking mechanism hai — na koi profile-field
hai, na koi verification-workflow, sirf ek simple yes/no signal.

**Bottom line**: Yeh feature khud koi customer-facing "product" nahi hai — yeh ek chhota,
backend interest-tracking utility hai jo bahut saari internal teams/systems ke liye ek shared
capture-point ka kaam karta hai.

**Business impact**: Marketing/sales teams ke liye ek simple signal hai — "yeh supplier is
popup/offer mein interested hai ya nahi" — jo aage lead-follow-up ya targeting ke liye use ho
sakta hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Popup dekhta hai (kis screen/app se, yeh scope ke bahar hai), "interested" mark kar sakta hai (ya nahi) |
| **~30 internal apps/systems** (MY, GLADMIN, TOLLFREE, Email Marketing, WEBERP, LEAP, IPC-Admin, NSD Sales, MAPI, ENQUIRY, FREE-WEBSITE, BL, TRADE, TENDER, PAYNOW, CREDIT ALLOCATION, OVP, SAMPARK, VENDOR CITY Pin Correction, Notification Server, search, FCP, PNS, Merp, IMOB, SELLERMY, PAYWIM, BUYERS_FEEDBACK, aur do) | Yeh sab is interest-signal ko record karne ke liye API call kar sakte hain — is poore KT-series ka sabse bada allowlist |

---

## 3. Main Business Flow (step-by-step, no code)

### Flow A — Supplier popup pe interest mark karta hai

```
1. Supplier ko koi popup dikhaya jaata hai (kis screen/app se, yeh scope ke bahar hai)
2. Supplier "interested" (yes/no) select karta hai
3. System check karta hai: request valid hai kya (supplier ID sahi hai, interest-value
   sirf yes/no hai, aur request kisi allowed internal-system se aa raha hai)
4. Agar sab sahi hai, system is interest-status ko save karta hai:
   - Agar supplier ka pehle se ek record hai, woh update ho jaata hai (latest
     interest-status + timestamp)
   - Agar naya hai, ek fresh record insert hota hai
5. System response return karta hai ki save successful hua ya nahi
```

**Business impact**: Ek single, latest-status-wala record maintain hota hai per supplier —
history nahi rakhi jaati, sirf current-status. Isse marketing/sales team ko real-time pata
chalta hai supplier abhi kis stance pe hai.

---

## 4. Business Rules — Plain Language Mein

1. **`is_interested` sirf "yes" (1) ya "no" (0) ho sakta hai** — koi third option, koi partial
   ya "maybe" state nahi hai.
2. **Ek supplier ka sirf ek hi latest record hota hai** — dobara submit karne par purana
   overwrite ho jaata hai (history nahi). Agar supplier apna mann badal le, sirf naya status
   hi system mein reflect hota hai.
3. **Bahut saare internal-systems yeh call kar sakte hain** — ek badi allowlist hai (~30
   systems/apps), matlab yeh ek widely-integrated tracking-point hai jo IndiaMART ke andar
   kai teams/channels use kar sakti hain (marketing email, tollfree calls, sales-admin tools,
   website, mobile app, waghera).
4. **Yeh ek "trusted internal system" hi record kar sakta hai, supplier khud directly nahi** —
   is feature ka access-control model IndiaMART ke internal apps ke through hi hai, supplier
   ka koi standalone public-facing endpoint nahi hai iske liye.
5. **Response fast hai** — is feature ka backend-processing bahut halka hai (ek hi save-step,
   koi multi-step verification ya background-processing nahi), isliye supplier ko turant
   confirmation milta hai.

---

## 5. Notifications

| Kab | Kya hota hai |
|---|---|
| Interest-status successfully save hua | Koi email/SMS notification supplier ko nahi jaata — yeh silent, backend-only tracking hai |
| Save fail ho gaya (galat input, ya system issue) | Calling internal-app ko ek failure-response milta hai; supplier ko khud koi separate error-notification nahi jaata (yeh depend karta hai us screen/app pe jo popup dikha raha tha, ki wo user ko kya dikhaye) |

**Note**: is feature mein koi email/SMS/push-notification wiring khud nahi hai (compare karo
GST feature se, jahan email confirmation explicitly jaata hai) — yeh purely ek silent
data-capture step hai.

---

## 6. Edge Cases — Product/Business Perspective se

1. **"Maine popup pe interested click kiya, phir wapas dekha status change ho gaya"** —
   Expected hai agar dusri baar "not interested" submit hua — sirf latest-status store hota
   hai, purana overwrite ho jaata hai.
2. **"Kaunsa specific popup/offer yeh track karta hai?"** — Backend data-storage level pe iska
   business-context (naam "MDC" ka kya matlab hai) documented nahi mila; yeh product/marketing
   team se hi confirm hona chahiye ki current-live popup/campaign kaunsa hai.
3. **Koi history/audit-trail nahi hai** — agar kabhi zaroorat pade yeh dekhne ki supplier ne
   *kab-kab* apna interest badla, yeh system usse answer nahi kar sakta — sirf abhi ka status
   available hai.
4. **Koi customer-facing notification nahi hai** — supplier ko save hone ka koi email/SMS nahi
   jaata; yeh purely backend-signal hai, jiska UX/feedback poori tarah calling-app pe depend
   karta hai.

---

## 7. Quick Summary

- Popup Details = simple "interested (yes/no)" tracking against supplier, latest-status-only.
- Insert-or-update (upsert), koi history nahi.
- Widely-integrated — ~30 internal-apps/systems allowed hain, supplier khud directly access
  nahi karta.
- Koi email/SMS notification nahi — silent backend tracking.
- Poore KT-series mein sabse simple/lightweight feature — koi multi-step verification, koi
  background-job, koi cross-team downstream fan-out iss doc mein confirm nahi hua.

---

## See also

- [`Popup_Details_Technical_Doc.md`](./Popup_Details_Technical_Doc.md) — code-level detail
  (APIs, DB table, validation, gateway-logic)

# Feedback — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Feedback_Technical_Doc.md`](./Feedback_Technical_Doc.md) dekho.

**Scope note**: Yeh KT teen feedback-types cover karta hai — sab "user-feedback" ke
under aate hain, kayi jagah shared code/queue use karte hain, isliye ek hi folder mein:

1. **My Feedback** — general app/web feedback (bug-report/suggestion-jaisa)
2. **IMSearch Feedback** — search-results ke against feedback
3. **App-Rating Feedback** — app-store-jaisa in-app rating-prompt (unauthenticated bhi ho sakta hai)

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Supplier/buyer IndiaMART app/website use karte waqt **apna feedback de sakte hain** —
teen alag contexts mein:

- **My Feedback**: General feedback — koi issue, suggestion, ya complaint, ek
  description/title/rating ke saath. Internal-team is par kaam kar sakti hai
  (worklog-remarks, issue-type-classification).
- **IMSearch Feedback**: Specifically **search-results ki quality** pe feedback —
  konsi query search ki, kya results mile, konsa result achha/bura tha, star-rating.
- **App-Rating Feedback**: App use karne ke baad ek **rating-popup** (jaisa Play
  Store/App Store pe milta hai) — yahan tak ki agar user login nahi bhi hai, tab bhi
  unka mobile-number capture karke naya account create ho sakta hai.

**Business impact**: Yeh product-quality aur user-satisfaction ka direct signal hai —
My Feedback se operational-issues pata chalte hain, IMSearch Feedback se search-relevance
improve hoti hai, App-Rating se overall app-experience track hoti hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier/Buyer** | Feedback deta hai (kisi bhi teen types mein se) |
| **Internal support/product-team** | My-Feedback ko review/classify/resolve karti hai |
| **Bahut saare internal-apps** | Feedback-submission ko route karte hain |

---

## 3. Business Flow (step-by-step, no code)

### My Feedback
```
1. User apna feedback submit karta hai (description, title, rating, mobile/email
   optional)
2. System pehle-se-defined issue-types se match karne ki koshish karta hai
3. Naya feedback-record banta hai (status "Pending")
4. Internal-team baad mein isse review kar sakti hai — worklog-remarks add karke,
   ya approval-status update karke
5. Har naya/updated feedback ek downstream-notification bhi trigger karta hai
```

### IMSearch Feedback
```
1. User kisi search-result pe feedback deta hai — query, page-number, kitne results
   mile, rating, comments
2. System is data ko save karta hai, search-quality-analysis ke liye
```

### App-Rating Feedback
```
1. App periodically user se rating maangta hai (jaisa app-store-prompt)
2. Agar user already logged-in nahi hai (sirf mobile-number di hai), system ek naya
   account bhi create kar sakta hai on-the-fly
3. Feedback record hota hai, user-experience-tracking ke liye
```

---

## 4. Business Rules — Plain Language Mein

1. **My Feedback aur IMSearch Feedback ek hi API endpoint share karte hain** — ek
   `FEEDBACK_FLAG` parameter decide karta hai kaunsa type hai.
2. **Naya feedback vs existing-feedback-update dono supported hain** — agar feedback-ID
   diya gaya, update hota hai; nahi toh naya record banta hai.
3. **App-Rating Feedback alag/independent flow hai** — bina login ke bhi kaam kar sakta
   hai, mobile-number ke through naya account bhi create ho sakta hai.
4. **App-version-based gating hai App-Rating Feedback mein** — bahut purani app-version
   se feedback allow nahi hota (minimum-version-check).
5. **My-Feedback ka description agar bahut chhota ho (<6 characters)**, system usse
   "vague" (unclear) mark kar deta hai — internal-triage ke liye ek useful signal.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine feedback diya, bina login kiye"** — App-Rating-Feedback flow mein possible
   hai, system naya account bana sakta hai.
2. **"Meri purani-app-version se rating-popup nahi aaya"** — Expected hai, minimum
   app-version-check fail hua hoga.
3. **"Mera feedback 'vague' mark ho gaya"** — Expected hai agar description bahut chhoti
   thi (5 characters ya kam).

---

## 6. Quick Summary

- Feedback = 3 types — My Feedback (general), IMSearch Feedback (search-quality),
  App-Rating Feedback (app-store-jaisa, unauthenticated-capable).
- My-Feedback aur IMSearch-Feedback ek shared write-endpoint use karte hain.
- App-Rating-Feedback independent flow hai, on-the-fly account-creation-capable.

---

## See also

- [`Feedback_Technical_Doc.md`](./Feedback_Technical_Doc.md) — code-level detail
- [`../Social Review KT/Social_Review_Business_Doc.md`](../Social%20Review%20KT/Social_Review_Business_Doc.md) —
  different concept (external social-media reviews, not internal feedback)

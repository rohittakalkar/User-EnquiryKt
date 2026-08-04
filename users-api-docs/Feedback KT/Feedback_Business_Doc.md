# Feedback — Business Doc (Product Perspective)

Yeh doc Feedback feature ko **business/product nazariye** se samjhata hai — koi code nahi,
sirf "kya hota hai, kyun hota hai, aur supplier/buyer/internal-team ke liye iska matlab kya
hai." Technical implementation (APIs, DB, queues, code) ke liye
[`Feedback_Technical_Doc.md`](./Feedback_Technical_Doc.md) dekho.

**Scope note**: Yeh KT teen feedback-types cover karta hai — sab "user-feedback" ke under
aate hain aur (jaisa iss deeper pass mein pata chala) actually ek hi backend-pipeline share
karte hain, isliye ek hi folder mein:

1. **My Feedback** — general app/web feedback (bug-report/suggestion-jaisa)
2. **IMSearch Feedback** — search-results ke against feedback
3. **App-Rating Feedback** — app-store-jaisa in-app rating-prompt (unauthenticated bhi ho
   sakta hai) — **behind the scenes, yeh My Feedback ka hi ek entry-point hai** (dekho
   section 3)

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Supplier/buyer IndiaMART app/website use karte waqt **apna feedback de sakte hain** — teen
alag contexts mein:

- **My Feedback**: General feedback — koi issue, suggestion, ya complaint, ek
  description/title/rating ke saath. Internal-team is par kaam kar sakti hai
  (worklog-remarks, issue-type-classification, approve/reject).
- **IMSearch Feedback**: Specifically **search-results ki quality** pe feedback — konsi query
  search ki, kya results mile, konsa result achha/bura tha, star-rating.
- **App-Rating Feedback**: App use karne ke baad ek **rating-popup** (jaisa Play Store/App
  Store pe milta hai) — yahan tak ki agar user login nahi bhi hai, tab bhi unka mobile-number
  capture karke naya account create ho sakta hai.

**Business impact**: Yeh product-quality aur user-satisfaction ka direct signal hai —
My Feedback se operational-issues pata chalte hain, IMSearch Feedback se search-relevance
improve hoti hai, App-Rating se overall app-experience track hoti hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier/Buyer** | Feedback deta hai (kisi bhi teen types mein se), authenticated ya (App-Rating ke case mein) unauthenticated |
| **Internal support/product-team** | My-Feedback ko review/classify/resolve karti hai, worklog daalti hai, approve/reject karti hai |
| **Feedback/AppCare team (internal email inbox)** | Har naye My-Feedback record (Android/iOS/HelpIM se aaya) ka ek internal-notification-email milta hai `feedback@indiamart.com` pe |
| **Feedback-submitting user (App-Rating flow mein)** | Kuch cases mein khud unhe bhi ek follow-up email milta hai (dekho section 6) |
| **Bahut saare internal-apps** | Feedback-submission ko route karte hain (GLADMIN, Weberp, MAPI, LEAP, seller-panel, buyer-app, etc.) |

---

## 3. Bada Discovery — Teeno Types Actually Ek Hi Pipeline Hain

Pehle yeh doc App-Rating Feedback ko ek **poori tarah alag, self-contained** flow maanta tha.
Deeper technical review mein pata chala ki **App-Rating Feedback apne aap koi feedback record
nahi banata** — yeh internally **My Feedback ke exact same submission-endpoint ko call karta
hai** (ek internal server-to-server request ke through, "loopback" style). Iska matlab:

- Jab bhi koi user App-Rating popup se feedback deta hai, woh record **`MY_FEEDBACKS`** table
  mein hi jaata hai — bilkul waisa hi jaisa My-Feedback ka record.
- Isliye App-Rating submission bhi internal-team ko My-Feedback jaisa hi internal-notification
  email trigger karta hai, **plus** apna khud ka ek alag customer-facing email bhi (agar rating
  low ho, section 6 dekho).
- Practically iska matlab: App-Rating "hidden traffic" hai My-Feedback ke volume/load pe —
  agar App-Rating popup ka usage spike ho, My-Feedback ka backend bhi utna hi load feel karega,
  chahe woh do alag features jaise dikhte hain.

**Business impact**: Agar kabhi My-Feedback ki reporting/analytics dekho, App-Rating se aaye
records bhi usi bucket mein honge — unhe alag se distinguish karne ke liye `FEEDBACKS_TITLE`
(hamesha `"Feedback from ANDROID"` / `"Feedback from IOS"` jaisa auto-generated hota hai App-
Rating ke case mein) ek useful signal hai.

---

## 4. Main Business Flows (step-by-step, no code)

### Flow A — User apna general feedback deta hai (My Feedback)

```
1. User (ya internal caller jaise seller-panel/app) feedback submit karta hai — title,
   description, source, user-type, mod-id mandatory hain naye feedback ke liye
2. System pehle-se-defined issue-types se match karne ki koshish karta hai (agar diya gaya ho)
3. Agar description bahut chhota hai (5 characters ya kam), system usse "vague" mark kar deta
   hai — internal-triage ke liye ek useful signal
4. Naya feedback-record banta hai (status "Pending" — 'P')
5. Internal feedback/support-team ko ek email jaata hai (agar feedback Android/iOS/HelpIM se
   aaya ho aur description khali na ho) — is email mein feedback ka issue-type bhi mention hai
6. Record ki ek second copy ek alag internal database mein bhi sync ho jaati hai
   (approval-workflow ke liye)
```

**Business impact**: Support team ko real-time pata chalta hai jab bhi koi naya issue/
suggestion aata hai. "Vague" tagging se team pehle hi filter kar sakti hai low-quality
feedback ko.

### Flow B — Internal team feedback ko review/update karta hai

```
1. Internal-team (GLADMIN etc.) ek existing feedback record ko open karta hai (feedback-ID +
   uski date ke saath)
2. Woh ya toh status set karta hai (Approve/Reject) ya ek worklog-remark add karta hai
   (dono mein se kam se kam ek zaroori hai)
3. System record update karta hai
4. Yeh update bhi wahi internal-notification-mail-chain re-trigger karta hai jo naye
   feedback pe chalti hai
```

**Business impact**: Feedback lifecycle track-able hai — kaun review kar raha hai, kya
outcome hai, sab record hota hai.

### Flow C — User search-result pe feedback deta hai (IMSearch Feedback)

```
1. User kisi search-result pe feedback deta hai — query, page-number, kitne results mile,
   rating, comments
2. System is data ko save karta hai, search-quality-analysis ke liye
3. (Koi email/notification nahi jaata iss flow mein — yeh purely analytics-data collection hai)
```

**Business impact**: Search-relevance team ko raw data milta hai analyze karne ke liye ki
konsi queries pe results kharab the.

### Flow D — App-Rating popup (App-store-jaisa, unauthenticated bhi ho sakta hai)

```
1. App periodically user se rating maangta hai (jaisa app-store-prompt) — ek number
   (single-digit) ke roop mein
2. Agar user already logged-in nahi hai (sirf mobile-number di hai), system ek naya
   account bhi create kar sakta hai on-the-fly
3. Behind the scenes, yeh submission My-Feedback ke exact same pipeline se guzarti hai
   (section 3) — record MY_FEEDBACKS mein jaata hai, internal team ko notify hoti hai
4. Agar rating "acchi" categories mein se ek hai (system ke andar 4 specific rating-values
   pehle se defined hain jo "achha/satisfied" maana jaata hai), koi extra email user ko nahi
   jaata
5. Agar rating in acchi categories mein nahi hai, user ko khud bhi ek follow-up email jaata
   hai (jaise "hume batao kya galat hua" jaisa outreach)
```

**Business impact**: Yeh app-experience ka pulse-check hai. Low-rating users ko specifically
follow-up email milta hai — ek potential churn-prevention/win-back touchpoint. Positive-rating
users ko extra email se disturb nahi kiya jaata.

---

## 5. Status/State — My Feedback Lifecycle

| Status (`APPROVAL_STATUS`) | Business meaning |
|---|---|
| **`P` (Pending)** | Default state jab naya feedback submit hota hai — abhi review nahi hua |
| **`A` (Approved)** | Internal team ne review karke approve kiya |
| **`D` (Rejected/Declined)** | Internal team ne review karke reject kiya |

**Business rule**: Status sirf `A` ya `D` hi ho sakta hai jab koi existing feedback update ho
raha ho — koi aur value system reject kar deta hai. Naya feedback hamesha `P` se start hota
hai, system khud set karta hai, submitter choose nahi kar sakta.

---

## 6. Notifications — Kaun ko, Kab, Kya Milta Hai

Yeh section pehle ki shallow-doc se sabse zyada expand hua hai — do **structurally alag**
notification-paths hain iss domain mein:

| Kab | Kise jaata hai | Kya |
|---|---|---|
| Naya My-Feedback record bana (Android/iOS/HelpIM se, description non-empty) | **Internal feedback/support-team** (`feedback@indiamart.com`) | Notification-mail feedback details + issue-type ke saath |
| My-Feedback record update hua (status/worklog change) | **Internal feedback/support-team** | Same internal-notification-chain dobara chalti hai |
| App-Rating diya gaya, rating "achhi" categories mein se ek nahi (4 specific values ke alawa) | **Feedback-submitting user khud** | Follow-up/outreach email — user ka email pata na ho toh yeh skip ho jaata hai |
| App-Rating diya gaya, rating "achhi" categories mein se ek ho | **Koi nahi** | Koi extra email nahi — user experience simple rakha jaata hai |
| IMSearch-Feedback diya gaya | **Koi nahi** | Yeh purely data-collection hai, koi notification chain nahi |

**Important correction from earlier version of this doc**: My-Feedback ka notification email
**customer ko नहीं jaata** — yeh sirf internal team ko jaata hai. Sirf App-Rating flow mein
customer ko khud bhi email milta hai (aur woh bhi sirf low-rating case mein). Pehle yeh do
paths conflate ho rahe the.

---

## 7. Business Rules — Plain Language Mein

1. **My Feedback aur IMSearch Feedback ek hi API endpoint share karte hain** — ek
   `FEEDBACK_FLAG` parameter decide karta hai kaunsa type hai (default My Feedback).
2. **Naya feedback vs existing-feedback-update dono supported hain** — agar feedback-ID aur
   uski date dono diye gaye, update hota hai; nahi toh naya record banta hai.
3. **Naya My-Feedback submit karne ke liye 5 fields mandatory hain**: kis user ne diya, title,
   source, user-type, aur kis mod (app/web) se — inme se koi bhi missing ho toh reject.
4. **Update karte waqt status ya worklog mein se kam se kam ek zaroori hai** — sirf feedback-ID
   dena kaafi nahi, kuch actual change bhi bhejna padta hai.
5. **App-Rating Feedback alag entry-point hai lekin backend mein My-Feedback ka hi hissa hai**
   (section 3) — bina login ke bhi kaam kar sakta hai, mobile-number ke through naya account
   bhi create ho sakta hai.
6. **App-version-based gating hai App-Rating Feedback mein** — bahut purani app-version
   (Android 12.4.1 se pehle, iOS 12.0.0 se pehle) se aane wale requests ko ek alag,
   simpler response-format milta hai (functionality block nahi hoti, sirf response-shape
   change hoti hai purane app ke saath compatibility ke liye).
7. **My-Feedback ka description agar bahut chhota ho (<6 characters)**, system usse "vague"
   (unclear) mark kar deta hai — internal-triage ke liye ek useful signal.
8. **Unrecognized issue-type ID silently ignore ho jaata hai, error nahi deta** — feedback
   phir bhi save ho jaata hai, bas bina classification ke.
9. **App-Rating ka title supplier/buyer khud nahi set karta** — system automatically
   `"Feedback from ANDROID"` jaisa title bana deta hai, mod (app-type) ke hisaab se.
10. **IMSearch Feedback insert ke liye query aur total-results dono zaroori hain** — inme se
    koi ek bhi khali ho toh reject.
11. **Mobile number 10-digit ya 91-prefix ke saath 12-digit dono accept hote hain** — system
    khud `91` hata deta hai storage se pehle.

---

## 8. Edge Cases — Product/Business Perspective se

1. **"Maine feedback diya, bina login kiye"** — App-Rating-Feedback flow mein possible hai,
   system naya account bana sakta hai mobile-number se.
2. **"Meri purani-app-version se rating-popup thoda different dikha"** — Expected hai; app-
   version-gate sirf response-format badalta hai, feature block nahi karta.
3. **"Mera feedback 'vague' mark ho gaya"** — Expected hai agar description bahut chhoti thi
   (5 characters ya kam).
4. **"Maine achhi rating di, koi follow-up email nahi mila"** — Yeh expected hai, feature bug
   nahi. Achhi ratings ko intentionally extra-email se disturb nahi kiya jaata.
5. **"Maine kharab rating di, ek email mila jismein mera feedback description nahi tha"** —
   Ho sakta hai, kyunki App-Rating ka follow-up email sirf ek generic outreach hai, poora
   feedback-detail nahi carry karta.
6. **Internal team ke notification-emails rate-limited hain** — agar bahut short time mein
   bahut saare feedback aa jaayein, kuch internal notification-emails intentionally throttle
   ho sakte hain (system ka hi safety-mechanism, spam-prevention ke liye).
7. **App se aaya feedback description kabhi-kabhi "encoded" jaisa lagta hai** (jaise
   "Reason[...]=...Sub[...]") — yeh normal hai, mobile-app UI structured reason/sub-reason
   codes bhejti hai jo internal mail mein readable format mein convert ho jaate hain.

---

## 9. Quick Summary

- Feedback = 3 entry-points — My Feedback (general), IMSearch Feedback (search-quality),
  App-Rating Feedback (app-store-jaisa, unauthenticated-capable) — **lekin App-Rating actually
  My-Feedback ka hi backend use karta hai.**
- My-Feedback aur IMSearch-Feedback ek shared write-endpoint use karte hain.
- My-Feedback ka notification internal-team ko jaata hai, customer ko nahi (except App-Rating
  ka apna alag, rating-dependent, customer-facing follow-up email).
- App-Rating-Feedback independent-looking flow hai but on-the-fly account-creation-capable aur
  underneath My-Feedback ka hi write-path share karta hai — dono flows ka load ek doosre ko
  impact karta hai.
- IMSearch-Feedback purely data-collection hai — koi notification, koi extra fan-out nahi.

---

## See also

- [`Feedback_Technical_Doc.md`](./Feedback_Technical_Doc.md) — code-level detail
- [`../Social Review KT/Social_Review_Business_Doc.md`](../Social%20Review%20KT/Social_Review_Business_Doc.md) —
  different concept (external social-media reviews, not internal feedback)
- [`../GST KT/GST_Business_Doc.md`](../GST%20KT/GST_Business_Doc.md) — depth/structure
  reference this doc was rebuilt against

# SellOnIM Log — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai — koi code nahi, sirf "kya hota hai, kyun
hota hai, aur supplier/sales-team ke liye iska matlab kya hai." Code ke liye
[`SellOnIM_Log_Technical_Doc.md`](./SellOnIM_Log_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

**SellOnIM** naye supplier ke onboarding-journey ko track karta hai — jab koi naya supplier
IndiaMART pe "sell" karna start karta hai (seller-panel/app se), unka signup/profile-completion
journey step-by-step record hota hai: naam diya ya nahi, company-name di ya nahi, email di ya
nahi, kitne products add kiye, address di ya nahi, GST di ya nahi, aur unhe kab call karna best
rahega (`BEST_TIME_TO_CALL`).

Iske peeche do maksad hain:

1. **Onboarding-progress visibility**: Internal team ko pata chalta hai kaunse naye suppliers
   apna profile-setup adhoora chhod chuke hain — taaki unhe follow-up ke liye target kiya ja
   sake.
2. **Automated lead-qualification**: Kuch specific supplier-types ke liye, yeh journey-data
   automatically ek downstream "calling-eligibility" evaluation ko trigger karta hai — jisse
   sirf genuinely good-fit suppliers hi internal calling-team ke "verification queue" tak
   pahunchte hain, bina kisi manual filtering ke.

**Business impact**: Yeh internal-sales/calling-team ko bataata hai ki kaunse suppliers apna
onboarding-journey adhoora chhod chuke hain, aur unhe follow-up call ke liye kab best time hai
— aur ek automated pipeline unmein se sirf qualified leads ko calling-queue tak pahunchati hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Naya supplier** | Apna onboarding-journey complete karta hai (indirectly, via app/web) |
| **App/tools (GLADMIN, MAPI, M.INDIAMART.COM, SELLERMY, IMOB, Android, iOS)** | Journey-progress record karte hain (yeh systems hi journey-data bhejte hain, supplier khud API nahi call karta) |
| **Internal calling/verification-team** | Downstream is data ko use karti hai supplier ko call karne ke liye (verification-queue table se) |

---

## 3. Business Flows (step-by-step, no code)

### Flow A — Naya supplier apna onboarding-journey complete karta hai

```
1. Naya supplier apna onboarding-journey shuru karta hai (naam, company, email, products,
   address, GST — step-by-step, app/web ke through)
2. Har step ka progress ek journey-log record mein capture hota hai — pehli baar naya record
   banta hai, baad mein wahi record update hota hai (LOG_ID se identify hoke)
3. Best-time-to-call bhi capture ho sakta hai, agar diya gaya ho
```

**Business impact**: Yeh ek running-record hai jo batata hai supplier apne onboarding mein
kahan tak pahuncha hai — naam di ya nahi, company-name di ya nahi, kitne product add kiye,
waghera.

### Flow B — Specific customer-type ke liye automated calling-eligibility trigger hoti hai

```
1. Agar supplier ek specific customer-type ka hai, journey-update hone par ek downstream-
   notification bhi automatically fire hoti hai (baaki customer-types ke liye sirf journey-
   log update hota hai, koi aage ka action nahi)
2. Downstream-consumer ek 5-step eligibility-check chain chalata hai:
   a. Kya iss supplier ka koi purana disqualifying verification-flag hai?
   b. Kya iss supplier ka priority-score sahi range mein hai?
   c. Kya iss supplier ka email kisi doosre risky account se duplicate toh nahi hai?
   d. Kya iss supplier ka GST kisi doosre account se duplicate toh nahi hai?
   e. Kya iss supplier ko pichle 30 din mein already calling-queue mein daala ja chuka hai?
3. Agar sab 5 checks pass hon, supplier ko "calling-queue" mein daal diya jaata hai
   (best-time-to-call ke saath, agar diya gaya ho)
4. Agar koi ek bhi check fail ho, supplier calling-queue mein nahi jaata — silently skip
   ho jaata hai, koi error nahi, koi notification nahi
```

**Business impact**: Yeh ek automated lead-qualification-pipeline hai — sirf woh suppliers
calling-queue tak pahunchte hain jo genuinely "good-fit" hain (verified-history-clean,
high-priority, non-duplicate-email, non-duplicate-GST, recently-not-already-queued).

---

## 4. Business Rules — Plain Language Mein

1. **Journey-log insert-or-update hota hai** — agar record ki ID (`LOG_ID`) nahi di gayi, naya
   record banta hai; agar di gayi, existing record update hota hai. Update sirf tab hota hai
   jab wahi `LOG_ID` sahi supplier (GLID) ke against ho — mismatch hone par update silently
   fail ho jaata hai.
2. **Kuch fields update ke time change nahi kiye ja sakte** — record-ID, supplier-ID, aur
   mobile-number, ek baar set hone ke baad update-request mein inhe change karne ki koshish
   ignore ho jaati hai (yeh sirf lookup-keys ki tarah treat hote hain, updatable data ki tarah
   nahi).
3. **Calling-eligibility-trigger sirf ek specific customer-type ke liye fire hota hai** —
   baaki customer-types ke liye sirf journey-log update hota hai, koi downstream-action nahi.
4. **Calling-eligibility 5 sequential checks pe depend karta hai** — sabhi pass hone chahiye:
   (a) verification-log mein koi disqualifying-flag nahi, (b) priority-score ek specific range
   mein, (c) koi duplicate-email-associated risky account nahi, (d) koi duplicate-GST nahi,
   (e) last 30 din mein koi duplicate-verification-allocation nahi. Ek bhi fail hua toh
   baaki checks chalte hi nahi (chain wahin ruk jaati hai).
5. **Sab pass hone par, ek naya "verification-queue" entry banta hai** — best-time-to-call ke
   saath, agar diya gaya ho, warna khaali reh jaata hai.
6. **Ek "not eligible" outcome permanent treat hota hai** — agar koi check fail ho jaaye, system
   khud-ba-khud dobara try nahi karta baad mein. Agar supplier ki disqualifying-condition baad
   mein resolve ho jaaye (jaise duplicate-GST clear ho jaaye), unhe re-evaluate karne ke liye
   ek fresh journey-update chahiye hoga.
7. **BEST_TIME_TO_CALL ek specific date-time format mein hona chahiye** — galat format wala
   request reject ho jaata hai.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine journey complete ki, lekin call nahi aayi"** — Expected ho sakta hai agar koi
   eligibility-check fail hui (duplicate-email/GST, ya priority-range match nahi hua, ya
   purana disqualifying-flag hai, ya pichle 30 din mein already queue ho chuka hai) — sirf
   journey-tracking se calling guaranteed nahi hoti.
2. **"Meri journey-update fail ho gayi"** — Ho sakta hai `LOG_ID` galat ho ya record already
   kisi aur supplier (GLID) ke against ho — update sirf matching (`LOG_ID`, `GLID`) pair pe
   hoti hai.
3. **"Meri eligibility-condition ab resolve ho gayi hai, phir bhi call nahi aa rahi"** — Yeh
   bhi expected hai (business-rule 6) — ek baar "not eligible" ban jaane ke baad, system
   khud dobara check nahi karta jab tak supplier ka journey-record fresh update na ho.
4. **Koi bhi "journey-progress" view/report supplier ko khud nahi dikhta** — yeh purely
   ek internal/backend tracking mechanism hai, supplier-facing dashboard iska part nahi
   confirm hua is review mein.

---

## 6. Quick Summary

- SellOnIM Log = naye-supplier onboarding-journey-tracker (progress steps + best-time-to-call).
- Insert-or-update, journey ka har update record hota hai.
- Ek specific customer-type ke liye, ek automated 5-check calling-eligibility-pipeline bhi
  trigger hoti hai.
- Eligible suppliers ek separate "verification/calling-queue" mein pahunchte hain, jise
  internal calling-team follow-up ke liye use karti hai.
- Ek "not eligible" result permanent hai — koi automatic retry nahi hai, fresh journey-update
  chahiye re-evaluation ke liye.

---

## See also

- [`SellOnIM_Log_Technical_Doc.md`](./SellOnIM_Log_Technical_Doc.md) — code-level detail

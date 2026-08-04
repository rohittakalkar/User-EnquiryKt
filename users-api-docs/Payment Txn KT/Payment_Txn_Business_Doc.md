# Payment Transaction (International) — Business Doc (Product Perspective)

Yeh doc Payment Txn (International) feature ko **business/product nazariye** se samjhata hai —
koi code nahi, sirf "kya hota hai, kyun hota hai, aur supplier/buyer ke liye iska matlab kya
hai." Technical implementation (API, DB, code) ke liye
[`Payment_Txn_Technical_Doc.md`](./Payment_Txn_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Jab koi **international buyer** IndiaMART ke through ek Indian **supplier ko payment** karta
hai (cross-border trade transaction), uska record system mein save hota hai — buyer ki details
(kaun hai, konse desh se hai, naam, email), transaction ka amount, currency, aur us payment se
jud a hua **invoice** (jiska URL/reference capture hota hai).

Yeh sirf ek log-entry nahi hai — iske peeche yeh maksad hai:

1. **Cross-border payment ka audit-trail banana**: International trade mein payment disputes
   common hain (currency conversion, buyer/seller mismatch, invoice confusion). Ek transaction
   ka record store karne se dono taraf — supplier aur buyer — ke paas proof rehta hai ki
   transaction actually hua.
2. **Invoice se transaction ko link karna**: Har transaction ek specific invoice se juda hota
   hai, taaki baad mein reconciliation (kaunsa payment kis invoice ke against tha) asaan ho.
3. **Financial/compliance reporting ke liye data capture karna**: International transactions
   par regulatory/compliance reporting requirements hoti hain — yeh data unke liye base banta
   hai.

**Bottom line**: Yeh IndiaMART ke international-trade/export-payment-facilitation ka ek
transaction-log hai — insert-once (poora record), aur sirf limited-fields later update ho sakte
hain.

**Scope note**: Iss deep-dive mein confirm hua ki abhi is data ko **wapas padhne ke liye koi API
nahi hai** (koi read-endpoint nahi mila), aur koi downstream system (consumer/queue/cron) is
data ko automatically process nahi karta jitna is codebase se pata chal saka — yeh sirf ek
write-only log hai abhi ke liye. Agar koi team isse consume kar rahi hai, woh probably direct
DB access ya kisi alag system se ho raha hoga (dekho Open Questions, technical doc section 13).

---

## 2. Kaun-kaun involved hai

| Kaun | Kya role hai |
|---|---|
| **International Buyer** | Ek Indian supplier ko payment karta hai (transaction ka origin) |
| **Supplier (seller)** | Payment receive karta hai; unke against transaction record likha jaata hai |
| **Internal apps** (GLADMIN, SELLERMY, IMOB, Android, iOS) | Yeh apps hi transaction-record likhne ki request bhejti hain — end-user seedha nahi karta, ek app/internal-tool ke through hota hai |
| **Finance/Compliance team** (assumed downstream consumer) | Iss data ko reporting/reconciliation ke liye use kar sakti hai — **exact consuming team/system codebase se confirm nahi ho paya, iska concrete answer team se lena hoga** |

---

## 3. Business Flow (step-by-step, no code)

### Flow A — Naya international payment record karna (Insert)

```
1. International buyer supplier ko payment karta hai (yeh actual payment IndiaMART ke bahar
   process hota hai — yeh feature sirf uska record rakhta hai)
2. Ek internal app (GLADMIN/SELLERMY/IMOB/Android/iOS) transaction ki poori details submit
   karta hai: buyer ID, seller ID, buyer ka desh (ISO code), buyer ka naam, currency,
   amount, aur invoice ka URL
3. System check karta hai sab mandatory fields sahi format mein hain (IDs numeric hain,
   country-code/naam/currency sahi type ke hain, amount numeric hai)
4. Invoice-URL se ek invoice-ID automatically nikal li jaati hai (URL ke "#" symbol ke
   baad wala hissa) — yeh future reference ke liye use hogi
5. Optionally, kuch additional custom-details (structured JSON data) bhi transaction ke
   saath attach ho sakte hain
6. Sab sahi hone par, poora record ek transaction-log table mein save ho jaata hai
```

**Business impact**: Cross-border payment ka permanent, invoice-linked record ban jaata hai —
future audit/reconciliation ke liye base.

### Flow B — Existing transaction ke custom-details update karna (Update)

```
1. Koi internal app ek existing transaction ke "custom details" (txn_details, JSON data)
   ko update karna chahta hai
2. System sirf do cheezein maangta hai: transaction ki invoice-ID, aur seller ID (match
   karne ke liye ki sahi record update ho raha hai)
3. Sirf custom-details field update hoti hai — baaki sab (amount, buyer-info, currency)
   waisa hi rehta hai jaisa insert ke time tha
4. Agar submit request mein custom-details hi nahi diye gaye, koi update hota hi nahi
   (behavior yahan thoda unclear hai — dekho Edge Cases)
```

**Business impact**: Insert-once, update-limited-fields model — transaction ka core financial
data (amount, buyer-info) kabhi silently badal nahi sakta ek update-call se; sirf supplementary
metadata revise ho sakti hai. Yeh ek intentional safety design lagta hai (financial-data
integrity ke liye), core numbers ko accidental-edit se bachane ke liye.

---

## 4. Business Rules — Plain Language Mein

1. **Do actions hi supported hain**: Insert (naya transaction record) aur Update (sirf
   custom-details JSON field update).
2. **Insert ke liye poori buyer/seller/amount/invoice-info mandatory hai** — buyer ID, seller
   ID, buyer ka desh, buyer ka naam, currency, amount, invoice-URL — in saat cheezon mein se
   koi bhi missing ho toh poora request reject ho jaata hai.
3. **Update sirf invoice-ID aur seller-ID se match hoke hota hai** — matlab update karne ke
   liye caller ko pehle se pata hona chahiye us transaction ki invoice-ID (jo insert ke time
   auto-derive hui thi) — system khud woh ID wapas nahi deta insert response mein bhi.
4. **Invoice-ID invoice-URL se automatically derive hoti hai** — URL ke "#" symbol ke baad ka
   hissa. Agar URL mein "#" hi nahi hai, invoice-ID khaali reh jaati hai, aur woh transaction
   phir kabhi update nahi ho paayega is API se (kyunki match karne ke liye invoice-ID chahiye).
5. **Sirf allowed internal-apps hi transaction record kar sakte hain** — GLADMIN, SELLERMY,
   IMOB, Android app, iOS app. Koi bhi random external caller reject ho jaata hai.
6. **Transaction-date ka format strict hai** — agar diya jaaye, ek specific
   date-time-format mein hona chahiye, warna reject.
7. **Custom-details (`txn_details`) agar diye jaayein, structured data (JSON object) hone
   chahiye** — free-text ya list format nahi chalega.
8. **Update sirf custom-details field ko touch karta hai** — amount, buyer-name, ya koi bhi
   aur field update-request mein bhej bhi do, woh silently ignore ho jaate hain, koi error
   nahi aata. Iska matlab galat amount ya buyer-info ek baar insert hone ke baad is API se
   kabhi fix nahi ho sakta.
9. **Koi generated reference-number wapas nahi milta insert ke response mein** — caller ko
   khud track karna padta hai apna invoice-URL/ID future update ke liye, system koi naya ID
   generate karke wapas nahi bhejta.

---

## 5. Notifications

**Koi notification/email/alert nahi mila iss feature ke liye** — na buyer ko, na supplier ko,
na internal team ko. Yeh purely ek silent backend record-write hai; success/failure sirf
calling-app ko turant API response mein pata chalta hai, koi async notification trigger nahi
hota (technical doc section 5-7 confirm karta hai koi RabbitMQ/Kafka/queue-based fan-out nahi
hai jo notification trigger kar sake).

| Kab | Kya notification jaata hai |
|---|---|
| Transaction insert/update successful | Koi nahi — sirf API response calling-app ko |
| Transaction insert/update fail | Koi nahi — sirf API response calling-app ko, error-message ke saath |

---

## 6. Edge Cases — Product/Business Perspective se

1. **"Maine transaction-details update ki, amount change nahi hua"** — Yeh expected behavior
   hai (rule 8), bug nahi. Update sirf custom-JSON field ko touch karta hai, core transaction
   data (amount, buyer-info, currency) hamesha immutable rehta hai insert ke baad.
2. **"Invoice-URL mein # nahi tha, invoice ID khaali reh gayi, ab update nahi ho pa raha"** —
   Yeh bhi expected hai (rule 4/9): agar invoice-URL format sahi nahi tha shuru mein, woh
   transaction future mein kabhi update nahi ho sakta is API se — koi recovery-path nahi mila
   is codebase mein.
3. **"Amount galat daal diya, ab kaise theek karein"** — Is API se **nahi ho sakta** (rule 8) —
   sirf naya insert ya direct DB-level fix hi option hai, jo dono operational-risk laate hain
   (duplicate record ya manual data-touch).
4. **"Kisi ne yeh data padhne ke liye API maanga"** — Abhi koi read-API nahi hai iss feature ke
   liye (deep-dive mein confirm hua). Agar koi report/dashboard/reconciliation-tool ko yeh data
   chahiye, ek naya read-endpoint banwana padega, ya existing BI/DB-access process use karna
   hoga.
5. **Update-request bina custom-details ke bheja** — behavior thoda unclear hai code se
   (technical doc section 4.13, 12.2 dekho) — team ke saath live-verify karna chahiye ki yeh
   cleanly "kuch nahi hua" jaisa react karta hai, ya koi unexpected error-condition trigger
   karta hai.

---

## 7. Quick Summary — Ek Line Mein Har Cheez

- Payment Txn = international buyer-to-seller payment ka transaction-log, invoice-linked.
- Insert (poora record ek baar) / Update (sirf custom-details JSON, baaki sab immutable).
- Invoice-ID invoice-URL se auto-extract hoti hai ("#" ke baad ka hissa) — future updates isi
  par depend karte hain.
- Sirf allowed internal apps (GLADMIN/SELLERMY/IMOB/Android/iOS) hi record kar sakte hain.
- Koi notification/email trigger nahi hota is feature se.
- Abhi koi read-API/consumer/cron nahi hai — yeh purely ek write-only log hai (confirm hua deep
  code-search se); agar koi downstream system isse use kar raha hai, woh is codebase ke bahar
  hai — team se confirm karo.

---

## See also

- [`Payment_Txn_Technical_Doc.md`](./Payment_Txn_Technical_Doc.md) — same flow, code-level
  detail (API, DB table, queries, RabbitMQ/Kafka/Redis confirmation, open questions)
- [`../service_api_write_reference.md`](../service_api_write_reference.md) — poori write-API
  inventory jisme yeh controller bhi list hai

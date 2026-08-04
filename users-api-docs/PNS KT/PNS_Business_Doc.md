# PNS (GSM Number Allocation / Transaction) — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`PNS_Technical_Doc.md`](./PNS_Technical_Doc.md) dekho.

**Note**: PNS aur PNS Setting **do alag concepts** hain — alag tables, alag purpose.
PNS (yeh doc) ek supplier ko ek **virtual-phone-number (GSM) allocate** karta hai, aur us
number ke pool-lifecycle (add/allocate/release/bulk-convert) ko manage karta hai. PNS
Setting supplier ki **contact/notification-preferences** manage karta hai. Dekho
[`../PNS Setting KT/PNS_Setting_Business_Doc.md`](../PNS%20Setting%20KT/PNS_Setting_Business_Doc.md).

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

**PNS** (Pay-N-Search / call-tracking-number-service) supplier ko ek **virtual phone-number
(GSM)** allocate karta hai — ek call-masking-number jo buyer aur supplier ke beech calls route
karta hai bina supplier ka actual number expose kiye.

Yeh number ek **shared pool** (`GL_GSM_MASTER`) se, vendor-ke-hisaab-se (jaise Knowlarity,
Ozonetel, Airtel, Tata, Vodafone — inferred se vendor-codes jaise `KNOW`, `OZON`, `ARTL`,
`TATAX`, `VODA` dekh ke) allocate hota hai.

**Business impact**:
1. **Call-tracking** — buyer-supplier calls monitor ho sakte hain (lead-quality, recording).
2. **Privacy** — supplier ka real phone-number kabhi expose nahi hota.
3. **Multi-vendor flexibility** — company alag-alag telecom-vendors ke saath kaam kar sakti
   hai bina supplier-facing experience change kiye, kyunki number-pool internally
   vendor-wise manage hota hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Internal-tools (bahut saare, badi allowlist — ~33 systems)** | Supplier ko GSM-number allocate/release/convert karte hain |
| **Supplier** | GSM-number receive karta hai apne account ke against, usi number pe buyer-calls aati hain |
| **GSM-number-pool** (`GL_GSM_MASTER`) | Available-numbers ka shared-inventory, vendor-type-specific |
| **Downstream "user-update" service** | Jab bhi ek number supplier ke account pe set/unset hota hai, yeh alag service actually us assignment ko supplier ke account-record pe likhta hai |
| **Telecom-vendors (Knowlarity, Ozonetel, Airtel, Tata, Vodafone, etc. — inferred se vendor codes)** | Actual call-routing/GSM-lines provide karte hain jinhe yeh pool track karta hai |

---

## 3. Number ka "Lifecycle" — ek number kis-kis state mein ho sakta hai

Har GSM number pool mein ek "flag" carry karta hai jo batata hai woh available hai ya kisi
supplier ko assign ho chuka hai. Business perspective se important hai kyunki iske through hi
"double-booking" (do suppliers ko ek hi number) rokna possible hai.

| State (business meaning) | Kya matlab hai |
|---|---|
| **Available (unassigned)** | Number pool mein pada hai, kisi ko allocate nahi hua, agla request isse le sakta hai |
| **Assigned (in-use)** | Number currently kisi supplier ko diya hua hai, unke buyer-calls isi pe aa rahi hain |
| **Reclaimable (do buckets)** | Number kisi wajah se pool mein wapas aaya hai lekin do alag "type"-buckets mein bant-ta hai (system internally isse manage karta hai — exact business difference in dono buckets ke beech clear nahi hai code se, **[INFERRED — team se confirm karo]**) |

**Note**: Kuch aur flag-values bhi system-validation mein "valid" maane jaate hain lekin is
review mein yeh confirm nahi ho paaya woh kis situation mein use hote hain — shayad kisi aur
internal-tool/vendor-integration se set hote hain. Team se confirm karna chahiye agar customer
ya support ko in states ke baare mein batana pade.

---

## 4. Main Business Flows (step-by-step, no code)

### Flow A — Naya supplier ko number allocate karna (auto-pick from pool)

```
1. Internal-tool ek request bhejta hai: "iss supplier ko iss vendor-type ka ek number do"
   (koi specific number nahi bataya, system khud pick karega)
2. System us vendor-type ke available-pool se ek random number pick karta hai
3. Us number ko "assigned" mark kiya jaata hai
4. Agar yeh supplier ka pehla number nahi tha (already koi number tha) —
   naya number set hone ke baad purana number wapas pool mein "available" ho jaata hai
5. Ek internal audit-record clear ho jaata hai (agar pehle koi "release-reason" record tha)
```

**Business impact**: Yeh sabse common flow hai — naya supplier onboard hote waqt ya existing
supplier ka number refresh karte waqt use hota hai. First-available-number pick karke assign
kar deta hai, per-vendor-type, koi priority/preference-logic nahi.

### Flow B — Ek specific number ko supplier ke against assign karna

```
1. Internal-tool ek specific GSM-number bataata hai (jaise ek naya number jo abhi vendor se
   procure hua hai) aur kis supplier ko dena hai
2. System check karta hai: yeh number kahin already kisi aur supplier ko assign toh nahi
3. Agar available hai (ya bilkul naya hai) -> assign ho jaata hai
4. Agar already assigned hai -> REJECT ("GSM number already exist as assigned")
```

**Business impact**: Fraud/error-prevention — ek number do suppliers ko simultaneously nahi
mil sakta.

### Flow C — Supplier ka number release karna

```
1. Internal-tool bolta hai "iss supplier ka number wapas le lo" (with a reason, optional)
2. System pehle supplier ke account se number ka assignment clear karne ki koshish karta hai
   (ek separate internal service ke through)
3. Agar yeh successful hota hai -> number pool-side directly touch nahi hota is step mein
   (assignment-clear hi kaafi maana jaata hai)
4. Agar yeh FAIL hota hai -> system khud fallback ke roop mein number ko pool mein
   "reclaimable" flag kar deta hai, taaki number "stuck/assigned-forever" na reh jaaye
5. Agar reason diya gaya tha -> ek audit-record banta hai ("kyun release hua")
6. Agar release fail hua process mein -> internal team ko email jaata hai alert ke roop mein
```

**Business impact**: Yeh ensure karta hai ki jab bhi ek supplier ka account band/change ho ya
unhe number wapas lena ho, number "phansta" nahi hai — ek fallback safety-net hai agar
main release-mechanism fail ho jaaye.

### Flow D — Naya number pool mein add karna, ya existing number ko re-flag karna

```
1. Internal-tool ek naya number pool mein add karta hai (ek naye vendor-line ke liye), ya
2. Kisi existing number ka flag/vendor update karta hai (jaise usse re-available mark karna)
```

**Business impact**: Yeh pool ko replenish/maintain karne ka mechanism hai — jab vendor se
naya line milti hai, ya kisi number ki settings change karni ho.

### Flow E — Bulk-convert (ek saath kayi numbers ko convert karna)

```
1. Internal-tool bolta hai "iss vendor-type ke N (max 49) available numbers ko ek target
   state mein convert kar do"
2. System ek-ek karke N numbers pick karta hai available-pool se aur unhe convert karta hai
3. Response mein bataata hai kitne successfully convert hue, aur kaunse numbers
```

**Business impact**: Yeh bulk-operations ke liye hai — jaise ek naye vendor-batch ko
"activate" karna ek hi request mein, ek-ek number manually convert karne ke bajaye.

---

## 5. Business Rules — Plain Language Mein

1. **Number-pool vendor-type-specific hai** — sirf matching-vendor-type ke available-numbers
   hi allocate ho sakte hain.
2. **Allocation "first-available/random" basis pe hoti hai** — koi priority/preference-logic
   nahi, pool se random pick hota hai per-vendor-type.
3. **Ek number ek time pe sirf ek supplier ko assign ho sakta hai** — system explicitly check
   karta hai duplicate-assignment se pehle, aur reject karta hai agar number already kisi
   aur ko assigned hai.
4. **Number release karne ka primary tareeka supplier ke account-record ko clear karna hai**
   — pool-side flag sirf tab change hota hai jab yeh primary mechanism fail ho jaaye (fallback
   safety-net).
5. **Bulk-convert ek request mein max 49 numbers tak allowed hai** — isse zyada ek single
   request mein process nahi hota.
6. **Bahut saare internal-apps allowed hain** — widely-used internal-allocation-tool, ~33
   internal systems (seller-panel, admin-tools, marketing-tools, apps, etc.) isse call kar
   sakte hain.
7. **GSM number format enforced hai** — minimum 10 digits, agar extension hai toh comma ke
   baad 3-4 extra digits allowed hain, format strictly validate hota hai.
8. **Vendor-specific lock-checks currently effectively inactive hain** — code mein ek
   per-vendor "active vendor" restriction dikhti hai, lekin practically yeh check ek hamesha-
   true condition ke peeche hai, isliye abhi kaam nahi karti (team ko flag karna chahiye agar
   yeh intentional nahi hai — [`PNS_Technical_Doc.md`](./PNS_Technical_Doc.md) section 5,
   rule 4 dekho).

---

## 6. Notifications — Kab kya pata chalta hai

| Kab | Kya hota hai |
|---|---|
| Number release process fail ho jaaye | Internal team (SOA/PNS team) ko email alert jaata hai, with error details |
| Supplier ko khud koi email/notification | **Iss review mein koi supplier-facing notification nahi mili** is transaction-flow mein (PNS Setting mein contact-preferences alag se hain) |
| Release/deletion ka reason record | Ek internal audit-table mein likha jaata hai (support/ops ke liye traceable rehta hai "kis supplier ka number kab, kyun release hua") |

---

## 7. Edge Cases — Product/Business Perspective se

1. **"Mujhe number allocate nahi hua"** — Ho sakta hai us vendor-type ke liye koi
   available-number pool mein bacha hi na ho.
2. **"Supplier ka number release toh request hua, lekin unka number dikh raha hai"** — Yeh
   process do steps mein hota hai: pehle account-record clear hota hai, tabhi supplier ko
   sahi se number-less dikhega. Agar beech mein kuch fail ho jaaye, system ek fallback flag
   set kar deta hai pool mein, lekin support ko dono jagah check karna chahiye
   (account-record aur pool-flag).
2. **"Ek number do baar allocate ho gaya"** — System explicitly duplicate-check karta hai,
   lekin agar bahut saari requests ek saath aayen (high concurrency), technical race-condition
   ka thoda risk hai (dekho Technical Doc, Optimization Scope).
3. **Replica-sync (downstream systems ko number-pool-changes ki jaankari) currently
   active nahi hai code mein** — iska matlab yeh nahi ki allocation kaam nahi karta (woh theek
   se kaam karta hai), lekin agar koi doosra internal-system pool ka apna "copy" maintain
   karta hai, woh copy stale ho sakta hai. Team ko is baare mein confirm karna chahiye.

---

## 8. Quick Summary

- PNS = GSM/virtual-number allocation aur lifecycle-management service, vendor-type-specific
  shared-pool se.
- Ek supplier ko ek time pe ek hi number assign ho sakta hai — system yeh strictly enforce
  karta hai.
- Number release ka primary mechanism supplier ke account-record ko clear karna hai; pool-flag
  sirf fallback ke roop mein change hota hai.
- Bulk-operations (ek saath kayi numbers convert karna) bhi supported hai, max 49 per-request.
- PNS Setting se alag concept hai (contact/notification-preferences) — sirf ek chhota
  overlap hai ek read-side flag ke through, koi shared-table/logic nahi.

---

## See also

- [`PNS_Technical_Doc.md`](./PNS_Technical_Doc.md) — code-level detail (APIs, DB tables,
  RabbitMQ, edge cases, optimization scope)
- [`../PNS Setting KT/PNS_Setting_Business_Doc.md`](../PNS%20Setting%20KT/PNS_Setting_Business_Doc.md) —
  unrelated concept, documented separately

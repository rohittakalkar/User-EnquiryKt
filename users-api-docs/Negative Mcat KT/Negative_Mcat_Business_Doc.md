# Negative Mcat — Business Doc (Product Perspective)

Yeh doc Negative Mcat feature ko **business/product nazariye** se samjhata hai — koi code
nahi, sirf "kya hota hai, kyun hota hai, aur supplier/product ke liye iska matlab kya hai."
Technical implementation (APIs, DB, queues, code) ke liye
[`Negative_Mcat_Technical_Doc.md`](./Negative_Mcat_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

**"Mcat"** (Micro-Category) IndiaMART ka product-classification unit hai — har product
listing ek ya zyada mcats se associate hoti hai. **Negative Mcat** ek "exclusion-list"
concept hai: kisi supplier (poore account level) ya kisi specific product-listing (item
level) ke liye, kuch mcats ko explicitly "yeh mcat iske liye applicable nahi hai / block
kiya gaya hai" mark kiya ja sakta hai.

Yeh feature isliye zaroori hai kyunki:

1. **Mis-categorization control**: agar koi supplier/product baar-baar galat category mein
   aa raha ho (accidentally ya jaan-boojh kar), internal-team us mcat ko unke liye negative
   mark kar sakti hai, bina product ko delete kiye.
2. **Search/browse quality**: agar exclusion sirf ek jagah store hoti aur downstream systems
   ko na pahunchti, toh supplier woh galat-category listing search results mein dikhta rehta.
   Isliye yeh change **downstream systems ko bhi propagate** hoti hai.
3. **Spam/fraud-adjacent listings ko suppress karna**: yeh ek broader alert/rejection-pipeline
   se bhi juda hai (dekho Flow C), matlab kuch cases mein yeh ek risk/compliance signal ki
   tarah bhi kaam karta hai, sirf categorization-hygiene tak seemit nahi.

**Bottom line**: Negative Mcat ek fine-grained, internal-team-controlled "is category ko
iske liye off kar do" switch hai — supplier/product delete karne se kaafi zyada surgical
tool hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Internal ops/admin teams (bahut saare allowed hain)** | Negative-mcat mark/unmark karte hain — GLADMIN, WebERP, Tollfree, Sales, PCAT-Admin, Tolexo, IMOB, Email Marketing, aur kai aur — 26 internal callers allowed hain, matlab yeh ek widely-touched internal feature hai |
| **Supplier** | Indirectly affected — unki product-listing kisi negative-marked mcat mein categorize nahi hogi |
| **Downstream systems** | Search/directory-replica system apna copy sync rakhta hai, aur ek alag alert/blacklist system bhi apna copy rakhta hai |
| **Ek external rejection-processing pipeline** | Negative-mcat event ek teesri jagah bhi forward hoti hai — ek alag "rejection master" pipeline, jiska poora scope in repos ke bahar hai (dekho Open Questions) |

---

## 3. Business Flows (step-by-step, no code)

### Flow A — Naya negative-mcat mark karna (exclusion add karna)

```
1. Internal ops/admin team member (GLADMIN/WebERP/Tollfree/PCAT-Admin/... — koi bhi allowed
   internal-tool) ek mcat ko kisi supplier ke poore account ke liye, ya kisi specific
   product-listing ke liye, "negative" mark karta hai
2. System check karta hai zaroori fields diye gaye hain (kis supplier, kis mcat, kaun kar
   raha hai)
3. Record save hota hai — kisne mark kiya (employee-name resolve hoke) aur kab, dono track
   hote hain
4. Change downstream systems ko async notify hoti hai:
   - Search/directory-system ko, taaki wahan bhi yeh exclusion reflect ho
   - Ek internal alert/blacklist system ko, apna snapshot rakhne ke liye
   - Ek external rejection-processing pipeline ko bhi, forward-only
5. Supplier/product ki respective listing ab is mcat se exclude treat hoti hai
```

**Business impact**: Category-level suppress ho jaati hai bina poore listing/product ko
touch kiye — sirf ek surgical, category-specific control.

### Flow B — Existing negative-mcat hataana (exclusion remove karna)

```
1. Internal ops/admin team same tool se ek existing negative-mcat record ko "delete" action
   ke saath submit karta hai
2. System record ko physically delete NAHI karta — usse "deleted" status mein flip kar deta
   hai (soft-delete); purana record DB mein reh jaata hai audit ke liye
3. Yeh change bhi wahi 3 downstream jagah async propagate hoti hai
4. Supplier/product ki listing ab dobara us mcat mein eligible ho jaati hai
```

**Business impact**: Reversible control hai — galti se lagaya gaya exclusion wapas hata sakte
hain, aur history bhi preserve rehti hai (kisne kab hataya).

### Flow C — Negative-mcat status query karna

```
1. Koi bhi caller (internal tool ya system) ek supplier (glusr) ya ek product-item ke against
   currently-active negative-mcats maang sakta hai
2. System sirf currently-active (non-deleted) records return karta hai, mcat ka naam bhi
   saath mein deta hai (sirf ID nahi)
```

**Business impact**: Koi bhi downstream system ya internal tool live exclusion-list check kar
sakta hai bina khud data maintain kiye.

---

## 4. Business Rules — Plain Language Mein

1. **Do actions supported hain** — "Insert" (naya negative-mcat add karna) aur "Delete"
   (soft-remove, purana record still DB mein rehta hai lekin "deleted" flag ke saath) —
   koi teesra action-type accept nahi hota, system explicitly reject kar deta hai.
2. **Ek negative-mcat record supplier-level ho sakta hai (poora account) ya item-level**
   (ek specific product-listing) — `item_id` optional hai; agar nahi diya, system usse
   automatically supplier-level (poore account ke liye) treat karta hai.
3. **Bahut saare internal-systems yeh action perform kar sakte hain** — 26 alag internal
   tools/teams allowed hain (GLADMIN, Tollfree, WebERP, PCAT-Admin, Tolexo, IMOB, Email
   Marketing, aur kai aur) — matlab yeh ek widely-used, cross-team internal-tool feature hai.
4. **Kisne action kiya, yeh hamesha track hota hai** — agar ek internal employee-ID diya
   jaaye, system unka naam resolve karke record karta hai; warna generic "User" label lagta
   hai.
5. **Change downstream systems ko propagate hoti hai** — sirf ek jagah change hona kaafi
   nahi hai, isliye system automatically 3 alag jagah notify karta hai: search-replica,
   alert/blacklist-system, aur ek external rejection-pipeline.
6. **Alert/blacklist system ka apna tracking level thoda alag hai** — woh system supplier +
   mcat ke level pe track karta hai, specific product-item ke level pe nahi (jabki primary
   record item-level bhi ho sakta hai). Matlab agar ek hi supplier+mcat ke liye do alag
   products pe negative-mcat lage, alert-system ki taraf se dono ek hi entry jaisa dikhega.
   **Team ko confirm karna chahiye ki yeh design intentional hai.**
7. **Query karte waqt ek single product-item specify kiya ja sakta hai**, lekin
   comma-separated multiple item-IDs ek saath dena currently **kaam nahi karta** — system
   khaali result de deta hai bina error ke. Isse product/tech ko aware hona chahiye agar
   koi internal-tool bulk item-lookup pe depend kar raha ho.

---

## 5. Notifications — Kisko Kab Pata Chalta Hai

Yeh feature **supplier ko koi direct email/SMS notification nahi bhejta** — poori tarah
internal-team-to-internal-system communication hai:

| Kab | Kya hota hai |
|---|---|
| Negative-mcat successfully add/remove hua | Search/directory-system ko async update jaata hai |
| Negative-mcat successfully add/remove hua | Alert/blacklist-system ko bhi async update jaata hai (glusr+mcat level pe) |
| Negative-mcat successfully add/remove hua | Ek external rejection-processing pipeline ko bhi event forward hota hai |
| Supplier ko | **Koi direct notification nahi** — yeh ek purely internal/backend control hai |

---

## 6. Edge Cases — Product/Business Perspective se

1. **"Meri listing ek category se gayab ho gayi"** — Ho sakta hai koi internal-team ne us
   mcat ko unke supplier-account ya specific-item ke liye negative-mark kiya ho. Yeh admin-driven
   action hai, supplier khud yeh nahi kar sakta.
2. **"Maine negative-mcat delete kiya, record abhi bhi kahin dikh raha hai"** — Delete "soft"
   hai, record physically remove nahi hota, sirf status-flag change hota hai — downstream
   systems ko propagate hone mein bhi thoda time-lag ho sakta hai (async fan-out).
3. **"Maine ek item ke liye negative-mcat lagaya, lekin alert-system mein woh account-wide
   dikh raha hai"** — Alert/blacklist system item-level granularity track nahi karta, sirf
   supplier+mcat level (business rule 6) — yeh expected hai, bug nahi, lekin confusing lag
   sakta hai agar teams ko iska pata na ho.
4. **"Maine bulk mein multiple item-IDs ke liye status check karne ki koshish ki, kuch nahi
   mila"** — Comma-separated multi-item query currently non-functional hai (business rule 7)
   — ek baar mein ek item-ID query karo.
5. **"Yeh change ek aur system (ETO rejection pipeline) mein bhi jaata hai, uska kya hota
   hai?"** — Yeh in teen repos ke daayre se bahar hai; agar koi issue ho us pipeline mein,
   relevant team se directly confirm karna hoga.

---

## 7. Quick Summary — Ek Line Mein Har Cheez

- Negative Mcat = supplier/product-level category-exclusion mechanism, purely internal-ops-driven.
- Insert (add-exclusion) / Delete (soft-remove-exclusion) — teesra action-type allowed nahi.
- Supplier-level ya item-level dono support hote hain; item na diya jaaye toh supplier-level
  default hota hai.
- 3-way downstream fan-out: search-replica, alert/blacklist-system, aur ek external
  rejection-pipeline (forward-only, apna DB nahi).
- Supplier ko koi direct notification nahi jaata — poori tarah backend/internal control hai.
- Widely-used internal feature — 26 internal-apps/teams allowed hain isse touch karne ke liye.

---

## See also

- [`Negative_Mcat_Technical_Doc.md`](./Negative_Mcat_Technical_Doc.md) — code-level detail
  (APIs, DB tables, queries, RabbitMQ, consumers, edge cases)
- [`../GST KT/GST_Business_Doc.md`](../GST%20KT/GST_Business_Doc.md) — similar
  business-doc structure ka precedent iss domain mein

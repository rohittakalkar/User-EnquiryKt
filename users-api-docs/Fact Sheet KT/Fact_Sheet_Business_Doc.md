# Fact Sheet — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Fact_Sheet_Technical_Doc.md`](./Fact_Sheet_Technical_Doc.md) dekho — dono docs same flows
cover karte hain, bas alag audience ke liye.

**Note**: Yeh feature GST/Bank-Details jaisa hi ek **shared multi-purpose "user-details"
write/read endpoint** ka hissa hai (`type=FactSheet` parameter se dispatch hota hai) — dekho
[`../GST KT/GST_Business_Doc.md`](../GST%20KT/GST_Business_Doc.md) same-controller-pattern ke
liye.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

**Fact Sheet** supplier ke company ke baare mein **detailed business-capability information**
capture karta hai — jaise: export-percentage (kitna business export hota hai),
international-credit-rating-numbers (Dun & Bradstreet, Coface), after-sale-support policy,
sampling-policy, competitive-advantage, contract-manufacturing-capability, quality-facilities,
delivery-time, payment-terms/mode, shipping-mode, aur staff-count (engineers,
skilled/semi-skilled workers, control-staff, consultants).

Yeh GST ya Bank Details jaisa "verification/compliance" data-point nahi hai — yeh **purely
descriptive business-capability information** hai jo supplier khud claim karta hai, koi
external system usse verify nahi karta.

**Business impact**: Yeh especially **international/export-buyers** ke liye ek detailed
supplier-capability-profile hai — buyer decide kar sakta hai ki supplier unki business-
requirements (volume, quality-standards, payment-terms, delivery-time) fulfill kar sakta hai
ya nahi, bina directly contact kiye. Ek achhi tarah bhari hui Fact Sheet supplier ko zyada
"business-ready" aur "serious" dikhati hai international buyers ke saamne.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Apni company ki fact-sheet-details fill/update karta hai — seller panel/app se |
| **Buyer** (khaaskar international/export-buyer) | Fact-sheet dekh ke supplier ki business-capability assess karta hai, before contacting |
| **IndiaMART platform** | Data store karta hai aur buyer-facing profile mein surface karta hai, koi verification nahi karta |

**Note**: GST/Bank-Details ke ulat, is process mein koi internal-admin override, external
verification-agency, ya daily-reconciliation-job involved nahi hai — yeh ek seedha
self-reported data-capture flow hai.

---

## 3. Main Business Flows (step-by-step, no code)

### Flow A — Supplier pehli baar apni Fact Sheet fill karta hai

```
1. Supplier seller panel/app pe jaake company-capability details bharta hai — export%,
   credit-rating numbers, quality/delivery/payment-terms, staff-strength
2. System check karta hai ki koi mandatory field jaisa "type/supplier-ID/updated-by/IP"
   waghera present hai ya nahi (generic check, Fact-Sheet-specific koi field mandatory nahi)
3. Naya record ban jaata hai — supplier ki company ke against ek hi Fact-Sheet-record hota hai
4. Change downstream systems ko bhi propagate hoti hai (background mein)
```

**Business impact**: Supplier ka profile ab zyada detailed aur "export-ready" dikhta hai
buyers ko — khaaskar wo buyers jo bulk/international orders ke liye supplier-capability
carefully evaluate karte hain.

### Flow B — Supplier apni existing Fact Sheet update karta hai

```
1. Supplier apni pehle se maujood fact-sheet mein kuch change karna chahta hai (jaise
   export-percentage badal gaya, ya naya credit-rating mil gaya)
2. System dekhta hai ki record already exist karta hai (ek internal flag ke through)
3. Poora record overwrite ho jaata hai naye values se
4. Downstream propagation same as Flow A
```

**Business impact**: Supplier apni capability-profile ko time-ke-saath update kar sakta hai
jaise business grow karta hai — lekin **poora form dobara bharna padta hai**, sirf ek field
change karna bhi poori submission maangta hai (dekho Business Rules, point 2).

### Flow C — Buyer profile dekhte waqt Fact Sheet data milta hai

```
1. Buyer kisi supplier ka profile ya "company details" section open karta hai
2. Agar us supplier ne Fact Sheet fill ki hai, toh system usse profile-response ke saath
   automatically include kar deta hai (ek combined/full-profile fetch ke through)
3. Agar Fact Sheet fill nahi ki hai, toh yeh section khaali/absent rehta hai
```

**Business impact**: Buyer ko ek hi jagah company-registration, bank, aur capability-details
saath mein milte hain — decision-making fast hoti hai.

---

## 4. Business Rules — Plain Language Mein

Yeh sab rules hain jo Fact Sheet process ko govern karte hain, jo maine actual code padh ke
nikale hain:

1. **Ek supplier ka sirf ek hi Fact-Sheet-record hota hai** — insert-or-full-update, koi
   history nahi maintain hoti (unlike GST-HSN jo history-table use karta hai).
2. **Update poora-record overwrite karta hai** — partial-update supported nahi hai. Agar
   supplier ne sirf ek field (jaise export-percentage) change karni thi, phir bhi poora form
   dobara submit karna padta hai — nahi toh baaki fields khaali/overwrite ho sakti hain.
3. **Koi bhi individual field mandatory nahi hai** — sirf generic details (kaun submit kar
   raha hai, IP, etc.) required hain. Business-fields (export%, staff-count, credit-rating)
   sab optional hain — supplier chahe toh sirf kuch fields bhar ke bhi submit kar sakta hai.
4. **Koi range ya sanity-check nahi hai numeric fields pe** — jaise export-percentage 0-100
   ke beech hi ho, ya staff-count negative na ho, aisa koi check system mein nahi hai. Yeh
   poori tarah supplier ki honesty pe depend karta hai.
5. **Credit-rating numbers (Dun & Bradstreet, Coface) verify nahi hote** — yeh sirf free-text
   values ke roop mein store hote hain, koi external-registry-check nahi hota (GST ke ulat,
   jo government-database se verify hota hai).
6. **Yeh GST/Bank-Details jaisa hi ek shared write-endpoint ka hissa hai** —
   `type=FactSheet` parameter se dispatch hota hai, same underlying API jo GST/Bank-Details
   bhi handle karta hai.
7. **Koi extra gateway/channel-restriction nahi hai** — GST ya Bank-Details ke liye kuch
   specific channels/permissions gated hain, lekin Fact Sheet ko platform ke kai alag-alag
   channels (web, app, admin, etc.) se submit kiya ja sakta hai, koi extra restriction nahi.

---

## 5. Notifications — Supplier ko kab pata chalta hai

| Kab | Kya hota hai |
|---|---|
| Fact Sheet successfully save/update hui | Koi dedicated supplier-facing email/SMS notification is review mein nahi mila — is flow mein sirf background system-sync hoti hai, GST jaisi "Your details updated successfully" wali email confirm nahi hui |
| Fact Sheet fill nahi ki | Koi reminder/nudge notification bhi is review mein nahi mila |

**Note**: Yeh GST se contrast hai, jahan har verified/rejected update pe explicit email
supplier ko jaata hai. Fact Sheet ek "silent" update-flow lagta hai — agar business team ko
supplier-facing confirmation chahiye, yeh ek gap ho sakta hai flag karne layak.

---

## 6. Edge Cases — Product/Business Perspective se

1. **"Maine sirf ek field update ki, baaki fields khali ho gayi"** — Ho sakta hai agar
   update-request mein baaki fields nahi bheji gayi thi. Is endpoint mein full-record
   overwrite hota hai, partial-update nahi — yeh sabse common confusion-point hoga support ke
   liye.
2. **"Maine galat export-percentage ya staff-count daala, system ne mujhe roka nahi"** —
   Expected behavior hai, bug nahi. System koi sanity-check nahi karta numeric-values pe,
   isliye responsibility supplier ki hi hai ki sahi data bhare.
3. **"Meri Dun & Bradstreet/Coface number galat hai, phir bhi accept ho gaya"** — Yeh
   numbers verify nahi hote kisi external registry se, sirf free-text store hote hain.
4. **"Mujhe koi confirmation nahi mila submit karne ke baad"** — Correct hai, is flow mein
   koi supplier-facing email/SMS confirmation nahi bheja jaata (unlike GST).

---

## 7. Quick Summary

- Fact Sheet = detailed supplier-business-capability profile (export%, credit-ratings,
  quality/delivery/payment-terms, staff-count) — purely self-reported, koi verification nahi.
- GST/Bank-Details jaisa hi shared "user-details" endpoint ka hissa, `type=FactSheet` se
  dispatch hota hai.
- Insert-or-full-update — koi partial-update ya history nahi; ek supplier ka ek hi record.
- Koi field individually mandatory nahi hai, koi range/sanity-check nahi hai numeric values
  pe, aur koi supplier-facing confirmation-notification bhi nahi bhejta yeh flow.
- International/export-buyers ke liye especially useful — supplier ki business-readiness ka
  ek snapshot deta hai bina directly contact kiye.

---

## See also

- [`Fact_Sheet_Technical_Doc.md`](./Fact_Sheet_Technical_Doc.md) — code-level detail
- [`../GST KT/GST_Business_Doc.md`](../GST%20KT/GST_Business_Doc.md) — same shared-controller
  pattern, business perspective
- [`../Bank Details KT/Bank_Details_Business_Doc.md`](../Bank%20Details%20KT/Bank_Details_Business_Doc.md) —
  another branch of the same shared endpoint, business perspective

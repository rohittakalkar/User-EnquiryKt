# Enquiry Attachment — Business Doc (Product Perspective)

Yeh doc "Enquiry Attachment" feature ko **business/product nazariye** se samjhata hai — koi
code nahi, sirf "kya hota hai, kyun hota hai." Yeh feature Save Enquiry se related hai — jab
buyer ek enquiry bhejta hai (text-based, [`Save Enquiry KT`](../Save%20Enquiry%20KT/Save_Enquiry_Business_Doc.md)
dekho), uske saath image/document bhi attach kar sakta hai — yehi feature un attachments ko
record karta hai. Technical implementation ke liye
[`Enquiry_Attachment_Technical_Doc.md`](./Enquiry_Attachment_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Jab buyer koi enquiry bhejta hai supplier ko, kabhi-kabhi sirf text kaafi nahi hota — buyer
apni requirement ko better explain karne ke liye ek image (jaise sample product ki photo) ya
ek document (jaise specification sheet, drawing) bhi attach karna chahta hai. Enquiry
Attachment feature isi cheez ko support karta hai:

1. **Buyer apni enquiry ke saath visual/document proof bhej sake** — "mujhe aisa product
   chahiye" ko text se zyada ek photo behtar explain karti hai.
2. **Supplier ko clearer requirement mile** — attachment ke through supplier ko enquiry
   samajhna aasan ho jaata hai, jisse better/faster response mil sakta hai.
3. **Attachment ek hi enquiry (Query ID) se linked rehta hai** — jab bhi wo enquiry supplier
   ko dikhayi jaaye ya Lead Management System (LMS) mein enrich ho, attachment bhi uske
   saath dikhta hai.

**Bottom line**: Yeh feature Save-Enquiry ka ek "add-on" hai — text-enquiry create hone ke
baad, uske saath image/document jodne ka tareeka.

---

## 2. Kaun-kaun involved hai iss process mein

| Kaun | Kya role hai |
|---|---|
| **Buyer** | Enquiry ke saath image/document attach karta hai (app/web se) |
| **Frontend/App clients** | Attachment ka data (image path, size, document path, type) system ko bhejte hain, ModID ke saath |
| **Image/Document review service (downstream, external)** | Har image ko automatically "approve" ya "reject" karta hai — abuse/inappropriate content na jaaye supplier tak |
| **Permanent-image-storage service (external)** | Uploaded image ko ek permanent, stable URL de deta hai |
| **Lead Management System (LMS)** | Approved attachments ka data yahan pahunchta hai, taaki enquiry ka "enrichment" (extra detail) LMS ko bhi dikhe |
| **Kibana/monitoring team** | Har attachment-request ka detailed log dekhti hai |

---

## 3. Attachment "type" classification — business meaning

| Concept | Business meaning |
|---|---|
| **Document (`doc_path`)** | Ek file (jaise PDF/spec-sheet) enquiry ke saath attach hoti hai |
| **Image — Original/Large/Medium/Small** | Ek hi image ke 4 alag resolution-variants — likely alag jagah (thumbnail vs full-view) dikhane ke liye |
| **`attach_type_id`** | Attachment ka category-code (numeric) — iska exact business meaning (jaise "sample photo" vs "specification") code mein documented nahi mila — **[INFERRED — product/frontend team se confirm karo exact category list]** |
| **Attachment Status — Approved / Rejected / Pending-review** | Har image automated review se guzarti hai; sirf approved images hi supplier-facing enrichment mein aage jaati hain |

**Business rule jo yaad rakhne layak hai**: Jab tak ek image "approved" nahi ho jaati,
wo LMS ko forward nahi hoti — matlab har attachment turant "live" nahi ho jaata, ek
automated moderation step beech mein hai.

---

## 4. Main Business Flows (step-by-step, no code)

### Flow A — Buyer apni enquiry ke saath attachment bhejta hai

```
1. Buyer enquiry bana chuka hai (Save Enquiry se, ek Query ID mil chuka hai)
2. Buyer/app ek image ya document upload karta hai, us Query ID ke reference ke saath
3. System basic validation karta hai: Query ID sahi hai? Kam se kam ek image ya document hai?
   Har attachment ka type-code diya gaya hai?
4. Agar sab sahi hai -> attachment record seedha database mein save ho jaata hai
5. Buyer ko turant confirmation milta hai ("Attachment saved successfully")
```

**Business impact**: Buyer ki requirement ab sirf text tak simit nahi — visual/document detail
bhi record ho jaati hai, jo enquiry ko supplier ke liye zyada useful banati hai.

### Flow B — Attachment ka background review aur Lead Management ko enrichment (separate pipeline)

```
1. Kahin se (system ke andar) ek "attachment message" background queue mein jaata hai
   (yeh Flow A se seedha connected nahi paya gaya — dekho Technical Doc, Open Questions)
2. Background system har image ko ek automated review-service ko bhejta hai:
   "yeh image appropriate hai ya nahi?"
3. Agar image approve ho jaaye -> ek permanent, stable storage-link generate hota hai
4. Attachment record (doc/image path, type, approval-status) ek doosri jagah database mein
   phir se save hota hai
5. Agar us enquiry ke liye "mail bhejna hai" flag set hai, aur approved attachment maujood hai
   -> attachment-detail Lead Management System ko forward kiya jaata hai
```

**Business impact**: Yeh ek safety/quality-control layer hai — sirf appropriate, review-passed
images hi supplier/Lead-Management tak pahunchti hain; content-abuse risk kam hota hai.

### Flow C — Enquiry ke Finish/close hone par attachment ka ek aur transfer (downstream, Approve/Reject/Finish se related)

```
1. Jab ek "waiting" enquiry automated system (FENQ bot) dwara process hoti hai
   (dekho Approve-Reject-Finish Enquiry KT), aur ek valid Offer/Lead ID mil jaata hai
2. System dobara us enquiry ke saare stored attachments (image/doc) database se nikalta hai
3. In attachments ko ek external "Buy Leads" system ko transfer kiya jaata hai
   (taaki purchased-lead record mein bhi attachment dikhe)
```

**Business impact**: Attachment sirf enquiry tak simit nahi rehta — agar wahi requirement
aage ek purchased-lead (Business Lead) mein convert ho jaaye, uska attachment bhi saath
carry hota hai, taaki downstream systems mein bhi buyer ki original requirement (image/doc
samet) dikhe.

---

## 5. Business Rules — Plain Language Mein

1. **Query ID (enquiry ka reference), ModID, aur kam-se-kam ek attachment (image ya
   document) mandatory hain** — bina inke request reject ho jaati hai.
2. **Har image/document ka `attach_type_id` mandatory hai** — bina type-code diye attachment
   accept nahi hota.
3. **Attachment record enquiry (Query ID) se hamesha linked rehta hai** — isse bina kisi
   enquiry ke koi standalone attachment nahi ban sakta.
4. **Sirf approved images hi Lead Management ko dikhti hain** — automated moderation ek gate
   hai; rejected images silently drop ho jaati hain enrichment-flow se (lekin DB record rehta
   hai, section 7 Edge Cases dekho).
5. **"Mail bhejna hai" flag na ho toh, Lead Management ko koi notification nahi jaata** —
   yeh ek per-enquiry setting hai jo attachment-enrichment ko control karti hai
   (Technical Doc, section on `DIR_QUERY_MAIL_SEND`).
6. **Documents ke liye koi automated content-review nahi hai** (sirf images review hoti
   hain) — documents by-default "approved" maan liye jaate hain.

---

## 6. Notifications

| Kab | Kise pata chalta hai | Kaise |
|---|---|---|
| Attachment turant DB mein save ho gaya (Flow A) | Sirf buyer/client jisne request bheji | Turant HTTP response mein |
| Approved image ka enrichment Lead Management ko gaya (Flow B) | Lead Management System | Background system-to-system notification |
| Enquiry ka attachment purchased-lead system ko transfer hua (Flow C) | Internal "Buy Leads" system | Background system-to-system transfer |
| Buyer/supplier ko koi email/SMS | **Iss feature ke scope mein nahi hai** — yeh sirf attachment record/enrich/transfer karta hai, koi direct communication yahan se trigger nahi hoti | — |

---

## 7. Edge Cases — Product/Business Perspective se

1. **"Maine attachment bheja, response mila 'saved successfully', lekin supplier ko attachment
   nahi dikh raha"** — Do alag pipelines hain (Flow A ka turant-save vs Flow B ka
   review-and-enrich). Ho sakta hai attachment DB mein save ho gaya ho lekin
   review/enrichment step tak abhi na pahuncha ho, ya image "rejected" ho gayi ho automated
   review mein.

2. **"Meri image reject ho gayi, lekin document upload ho gaya"** — Yeh expected hai; sirf
   images automated content-review se guzarti hain, documents nahi.

3. **Yeh feature khud koi email/SMS nahi bhejta** — agar koi poochhe "mujhe attachment ka
   confirmation email kyun nahi mila," jawaab hai: yeh feature ka scope hi nahi hai.

4. **Attachment ek baar save hone ke baad kahin bhi is doc mein "delete/update" ka business
   flow nahi mila** — sirf naya attachment add karna cover hota hai
   **[INFERRED — confirm karo product/team se attachment-removal kis flow se hota hai, agar
   hota hai]**.

---

## 8. Quick Summary — Ek Line Mein Har Cheez

- Enquiry Attachment, Save Enquiry ka ek add-on hai — enquiry ke saath image/document jodta hai.
- Ek turant-save path hai (buyer ko response), aur ek alag background review+enrichment
  pipeline hai jo Lead Management System ko batati hai — dono seedha connected nahi paye gaye
  is review mein (Technical Doc, Open Questions dekho).
- Sirf images automated content-review se guzarti hain; documents nahi.
- Sirf approved images hi Lead Management enrichment mein jaati hain, aur sirf tab jab enquiry
  pe "mail bhejna hai" flag set ho.
- Jab enquiry aage ek purchased-lead mein convert hoti hai, uske attachments bhi ek external
  Buy-Leads system ko transfer hote hain.
- Koi buyer/supplier-facing email/SMS iss feature se directly trigger nahi hoti.

---

## See also

- [`Enquiry_Attachment_Technical_Doc.md`](./Enquiry_Attachment_Technical_Doc.md) — same flows,
  code-level detail (APIs, DB tables, queues, Kafka, consumers)
- [`../Save Enquiry KT/Save_Enquiry_Business_Doc.md`](../Save%20Enquiry%20KT/Save_Enquiry_Business_Doc.md) — text-enquiry creation; Enquiry Attachment records reference the same Query ID this flow produces
- [`../Approve-Reject-Finish Enquiry KT/Approve_Reject_Finish_Enquiry_Business_Doc.md`](../Approve-Reject-Finish%20Enquiry%20KT/Approve_Reject_Finish_Enquiry_Business_Doc.md) — the FENQ automated-processing flow that reads back stored attachments and transfers them to the Buy-Leads system (Flow C above)

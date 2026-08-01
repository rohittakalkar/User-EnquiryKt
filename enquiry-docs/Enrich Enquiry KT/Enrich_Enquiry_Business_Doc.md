# Enrich Enquiry — Business Doc (Product Perspective)

Yeh doc "Enrich Enquiry" feature ka **business/product perspective** cover karta hai — yeh kya
karta hai, kaun use karta hai, aur kis situation mein trigger hota hai. Code-level detail ke liye
[`Enrich_Enquiry_Technical_Doc.md`](./Enrich_Enquiry_Technical_Doc.md) dekho — dono docs same
feature cover karte hain, bas alag audience ke liye.

**Note**: Enrich Enquiry ek "already-exists" enquiry ko **update/enhance** karne wala endpoint
hai — yeh naya enquiry nahi banata (woh kaam Save Enquiry karta hai). Yeh existing enquiry record
mein extra detail (order value, usage, geography, shipment/payment mode, sender contact info,
subject line waghera) bharta/update karta hai.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Jab ek buyer enquiry submit karta hai (Save Enquiry ke through), us waqt shayad saari detail
available nahi hoti — ya phir baad mein koi additional info (jaise approximate order value,
purchase frequency, destination port, shipment mode, ya buyer ka updated contact detail) aati hai.
"Enrich Enquiry" is additional data ko **existing enquiry record mein wapas likhne** ka mechanism
hai.

Business impact: enquiry jitni "enriched" (detailed) hogi, supplier ke liye utna hi useful hoga —
better order-value estimate, sahi shipment/payment preference, aur clearer subject line se
supplier ko turant samajh aata hai ki buyer ko kya chahiye.

---

## 2. Kaun-kaun involved hai

| Party | Role |
|---|---|
| Caller (upstream system/UI) | `enrichEnquiry` API ko hit karta hai extra enquiry detail ke saath. **Exact caller (web form, app, ya koi internal automation) code se conclusively confirm nahi hota — [INFERRED — confirm with team]** |
| Buyer (Sender) | Jiska naam/email/city/state waghera is enrichment ka source data hai |
| Supplier (Receiver) | Jiske liye enrichment se better subject line aur order details generate hote hain |
| `enq-services-scm` write API | `EnrichEnquiry` controller + model — synchronous DB update karta hai, aur ek async queue message bhi bhejta hai |
| `enq-consumers` background worker | `enq_post_fenq_enrich.go` — queue se message utha ke do alag Postgres databases (IMBL aur ENQ) mein enrichment data likhta hai |

---

## 3. Main Business Flows

### Flow A — Normal enrichment (happy path)

1. Caller `enrichEnquiry` API ko `enquiry_id` + enrichment fields (order value, usage, purchase
   period, geography, shipment/payment mode, sender detail, etc.) ke saath hit karta hai.
2. System input ko validate karta hai (numeric checks, string-length truncation, UTF garbage
   cleanup).
3. System enquiry ke current receiver (supplier) ko lookup karta hai.
4. Agar sender ka naam diya gaya hai, to system ek **auto-generated subject line** banata hai
   (jaise "Enquiry for `<product>` from `<buyer first name>` via IndiaMART", ya export-specific
   variant agar buyer India ke bahar se hai).
5. System enquiry ke child table (normal/waiting/bounced, `queryDestination` ke hisaab se) mein
   ek stored procedure call se saara enrichment data **turant (synchronously) update** karta hai.
6. Update safal hone ke baad, system wahi data ek background queue mein bhi daal deta hai, taaki
   ek doosra system (downstream worker) apni copy bhi update kar sake.
7. Caller ko success/failure response milta hai.

**Business impact**: enquiry record turant updated ho jata hai (real-time), aur ek parallel
background process doosri systems (jahan enquiry data duplicate/mirror hota hai) ko bhi sync kar
deta hai — bina caller ko wait karaye.

### Flow B — Invalid input (enquiry_id missing/invalid)

1. Caller `enquiry_id` ke bina, ya invalid (non-numeric/negative) `enquiry_id` ke saath request
   bhejta hai.
2. System turant validation fail kar deta hai — "Invalid Enrich Enquiry detail" error return hota
   hai, koi DB update nahi hota, koi queue message nahi jaata.

**Business impact**: galat/incomplete data se koi existing enquiry corrupt nahi hoti.

### Flow C — Duplicate-key / DB conflict

1. Stored procedure call ek DB-level unique-constraint violation (duplicate key) return karta
   hai.
2. System is case ko bhi "Invalid Enrich Enquiry detail" jaise treat karta hai (caller ko normal
   200 response milta hai, lekin andar error flag ke saath) — koi retry/queue push nahi hota.

**Business impact**: race-condition ya duplicate-submit se DB corruption nahi hota; caller ko
graceful failure milta hai, hard crash nahi.

### Flow D — Background sync failure (queue push fails)

1. Enquiry record ka synchronous DB update (Flow A step 5) safaltapoorvak ho jata hai.
2. Lekin background queue mein message push karna fail ho jata hai (RabbitMQ down/unreachable).
3. System is failed message ko ek **local file mein backup** kar leta hai (taaki baad mein
   recover kiya ja sake), aur caller ko phir bhi apna normal response deta hai.

**Business impact**: primary enquiry update kabhi block nahi hota background-sync failure ki
wajah se — buyer/supplier-facing update turant reflect hota hai, doosri system ka sync thoda
delay ho sakta hai lekin data loss nahi hota (file backup ki wajah se).

### Flow E — Background worker apni copy update karta hai

1. Background queue (`ENQUIRY_FENQ_ENRICH`) se message uthaya jaata hai ek alag worker process
   dwara.
2. Yeh worker do alag databases mein enrichment data likhta hai:
   - Ek "IMBL" naam ki database mein — **lekin sirf tab jab enquiry abhi bhi normal/waiting state
     mein ho (rejected na ho) aur ek specific "already-enriched" flag set na ho.**
   - Doosri "ENQ" naam ki database (yehi database jahan Flow A ka synchronous update pehle se ho
     chuka tha) mein — **hamesha**, bina kisi condition ke.

**Business impact**: yeh do-database sync ka mechanism hai — lagta hai ki enquiry data do alag
systems mein duplicate/mirror rehta hai (jaise ek legacy "IMBL" system aur naya "ENQ" system), aur
yeh worker unhe sync mein rakhta hai. Exact business reason ki do systems kyun maintain ki jaati
hain, **[INFERRED — confirm with team]**.

---

## 4. Business Rules — Plain Language Mein

1. **`enquiry_id` mandatory hai aur numeric, non-negative hona chahiye** — warna request turant
   reject ho jaati hai.
2. **Subject line sirf tab auto-generate hoti hai jab sender ka naam diya gaya ho.** Agar sender
   name empty hai, to subject line generate nahi hoti (empty rehti hai, jab tak caller khud koi
   subject na bheje).
3. **Subject line ka content buyer ki country pe depend karta hai** — agar buyer India ke bahar
   se hai (aur product/sender/country sab available hain), to "Export Enquiry..." wording use hoti
   hai, warna normal "Enquiry for..." wording.
4. **Latitude/Longitude (`S_lat`/`S_long`) sirf tab accept hote hain jab dono numeric hon** —
   warna dono empty kar diye jaate hain (partial/garbage geo-coordinates save nahi hote).
5. **Enrichment update ek specific child table pe hota hai jo `queryDestination` decide karta
   hai** — normal enquiry, waiting enquiry, ya bounced enquiry, teeno ke alag-alag table hote hain
   (jaisa Save Enquiry mein bhi documented hai).
6. **Duplicate-key errors ko "invalid input" jaisa treat kiya jata hai**, system-level 503 error
   nahi — caller ko silently ek "Invalid Enrich Enquiry detail" milta hai.
7. **Background queue push tabhi hota hai jab synchronous DB update successful ho.** Agar DB
   update fail hua, to koi queue message nahi jaata.
8. **Background worker ka "IMBL" database update conditional hai** — sirf tab hota hai jab
   `queryDestination` normal(1)/waiting(2) ho **aur** `fenq_enrich` flag `0` ho. Rejected/bounced
   enquiries, ya jinke liye enrichment already ho chuka hai (`fenq_enrich != 0`), unke liye IMBL
   update skip ho jata hai.
9. **Background worker ka "ENQ" database update unconditional hai** — yeh hamesha chalta hai,
   waiting/rejected/enriched-flag ki parwah kiye bina.
10. **String fields UTF-garbage-clean aur length-truncate hote hain** — jaise `Description` 4000
    chars tak, `req_usage` 2000 tak, `ApprxOrderValue` 200 tak — taaki DB column overflow na ho.

---

## 5. Notifications

Enrich Enquiry apne aap koi buyer/supplier-facing email/SMS/notification **trigger nahi karta**
— code mein is feature ke andar koi notification-triggering call **nahi mila**. Yeh sirf data
enrichment/update ka kaam karta hai.

| Trigger | Recipient | Channel | Source |
|---|---|---|---|
| — | — | — | Koi notification-related code is feature mein nahi paaya gaya. **[Confirm with team if any downstream system triggers a notification off the back of this update]** |

---

## 6. Edge Cases

- **Sender name diya gaya hai lekin `enquiry_id` empty hai** → subject generate nahi hoti (dono
  condition zaroori hain).
- **Receiver lookup fail ho jaye (no matching row, ya DB error)** → subject generation
  gracefully empty rehta hai, poora request fail nahi hota.
- **`queryDestination` ek unexpected/unmapped value ho** → subject-fetch table resolve nahi hota,
  subject empty reh jata hai (silent skip, koi error nahi).
- **Timeout ho jaye DB query mein** → caller ko `504`-jaisa "Timeout occurred" error milta hai.
- **RabbitMQ down ho** → background sync ke liye local file-backup mechanism engage hota hai
  (Flow D dekho); caller ko fir bhi normal response milta hai.
- **Same enquiry_id pe do parallel enrich requests aayein** → dusri wali duplicate-key error de
  sakti hai (Flow C).

---

## 7. Quick Summary

Enrich Enquiry ek existing enquiry record ko additional business detail (order value, usage,
geography, shipment/payment mode, sender contact, auto-generated subject) se update karta hai —
seedha (synchronous) ek DB table update karke, aur parallel mein ek background worker ke through
do doosri databases (legacy "IMBL" aur "ENQ") ko bhi sync karke.

---

## See also

- [`Enrich_Enquiry_Technical_Doc.md`](./Enrich_Enquiry_Technical_Doc.md) — code-level deep dive
- [`../Save Enquiry KT/Save_Enquiry_Business_Doc.md`](../Save%20Enquiry%20KT/Save_Enquiry_Business_Doc.md) — enquiry creation flow, and the `queryDestination` classification this feature reuses
- [`../Approve-Reject-Finish Enquiry KT/Approve_Reject_Finish_Enquiry_Business_Doc.md`](../Approve-Reject-Finish%20Enquiry%20KT/Approve_Reject_Finish_Enquiry_Business_Doc.md) — the FENQ waiting-enquiry bot (`enq_post_fenq.go`) documented there is a **different, unrelated consumer** from this feature's `enq_post_fenq_enrich.go`, despite the similar name — see Technical Doc section 6 for the distinction

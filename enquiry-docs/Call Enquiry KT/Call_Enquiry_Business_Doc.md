# Call Enquiry — Business Doc (Product Perspective)

Yeh doc "Call Enquiry" group ke 4 closely-related endpoints ko **business/product nazariye** se
samjhata hai — koi code nahi, sirf "kya hota hai, kyun hota hai." Save Enquiry aur
Approve/Reject/Finish Enquiry doosre tarah ka enquiry (text-based) create/lifecycle-manage
karte hain; **Call Enquiry** group buyer-supplier ke **phone-call-based** interactions ko
track karta hai — Click-to-Call (C2C), unidentified/anonymous calls, aur PNS
(virtual-number-routed call system) ka data. Technical implementation ke liye
[`Call_Enquiry_Technical_Doc.md`](./Call_Enquiry_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

IndiaMART pe buyer sirf text-enquiry hi nahi bhejta — bahut baar wo seedha supplier ko **call**
kar deta hai (Click-to-Call button se, ya app se). Har aisi call bhi ek "lead" hai, aur usse
track karna zaroori hai taaki:

1. **Har buyer-supplier interaction (chahe call ho ya text) Lead Management System mein record
   ho** — supplier ka pura interaction-history sirf enquiries tak simit na rahe.
2. **Abuse/spam calling rok sake** — agar koi buyer ek din mein 50+ alag-alag suppliers ko call
   kar chuka hai, system usse aage call karne se rok deta hai (rate-limiting).
3. **Anonymous/unidentified calls ko baad mein "identify" kiya ja sake** — kabhi-kabhi call ke
   waqt buyer ka GLID (IndiaMART account) pata nahi hota (jaise ek website widget se call aayi
   ho bina login ke); system pehle call record kar leta hai "unidentified" ke roop mein, aur
   baad mein jab buyer identify ho jaaye, record ko update kar deta hai.
4. **PNS (virtual-number call-routing system) se aayi calls ko IndiaMART ke apne call-record
   se link kiya ja sake** — jab call ek virtual/masked number ke through hoti hai (privacy ke
   liye), PNS system usse alag se track karta hai; `linkC2CPNS` in dono records ko jodta hai
   taaki Lead Management ko ek unified/complete tasveer mile.

**Bottom line**: Yeh group "call" wale interactions ka Save-Enquiry-jaisa role play karta hai —
call hui, usse record karo, Lead Management ko batao, aur agar call anonymous/partial thi to
usse baad mein complete/link karo.

---

## 2. Kaun-kaun involved hai iss process mein

| Kaun | Kya role hai |
|---|---|
| **Buyer** | Supplier ko call karta hai (Click-to-Call button, app, ya website widget se) |
| **Supplier (Receiver)** | Jise call jaati hai |
| **Frontend/App clients (WEB, IOS, ANDROID, IMOB, etc.)** | Call event ko `c2c_modid` ke saath report karte hain |
| **PNS (virtual-number call-routing service, external/internal system)** | Actual phone call ko route/mask/record karta hai; apne khud ke call-records maintain karta hai jo baad mein IndiaMART ke C2C record se link kiye jaate hain |
| **Bsmapping service (internal, users-domain)** | Check karta hai ki caller-receiver pair pehle se "matched" (jaana-pehchana) hai ya nahi, aur rate-limiting decide karta hai |
| **Lead Management System (LMS)** | Har call ka transaction record yahan jaata hai, text-enquiries ke saath consistent tarike se |
| **Kibana/monitoring team** | Har call-record request ka detailed log dekhti hai |

---

## 3. Call "type/state" classification — business meaning

| Concept | Business meaning |
|---|---|
| **Identified call (`callEnquiry`)** | Buyer ka GLID pata hai call ke waqt hi — normal case |
| **Unidentified call (`callEnquiryUnidentified` / `unidentifiedC2C`)** | Buyer ka GLID pata nahi tha call ke waqt — record pehle anonymous save hota hai, baad mein "identify" hoke asli call-record (`callEnquiry`) mein convert ho sakta hai |
| **PNS pre-call vs post-call (`linkC2CPNS`, `state=pre`/`post`)** | Virtual-number system do tarah ke call-records rakhta hai — "active/ringing" call (pre) aur "completed" call (post); dono ko IndiaMART ke apne C2C record se alag-alag link kiya jaata hai |
| **Query-Ref-Type (`W` / `B`)** | Agar call ek existing enquiry (Waiting/Direct query) ya ek purchased-lead (Business-lead/BL) se related hai — system automatically nikal leta hai konsa recent hai, buyer se poochta nahi |
| **Hard-match vs Soft-match caller-receiver pair** | System pehle se check karta hai ki yeh do log (caller-receiver) pehle bhi genuinely connected the ya nahi — is basis pe call ka "record type" auto-classify hota hai. **Exact business meaning [INFERRED — Trust/Users team se confirm karo]** |

**Business rule jo yaad rakhne layak hai**: Agar ek buyer ek hi din mein 50 ya usse zyada
**alag-alag** suppliers ko call kar chuka hai, uski next call **block** ho jaati hai ("number
of attempts to connect unique buyers/sellers exhausted"). Yeh spam/abuse-calling rokne ka
mechanism hai.

---

## 4. Main Business Flows (step-by-step, no code)

### Flow A — Buyer ek identified call karta hai (happy path)

```
1. Buyer supplier ko call karta hai (Click-to-Call button se) — buyer ka GLID pehle se pata hai
2. Frontend call ka detail (caller, receiver, duration, page-context, etc.) system ko bhejta hai
3. System check karta hai: is buyer ne aaj already 50+ alag suppliers ko call kiya hai? Agar
   haan -> block ("attempts exhausted")
4. Agar nahi -> call record ban jaata hai, ek unique Call Record ID milta hai
5. Turant background mein: Lead Management System ko is call ke baare mein bataya jaata hai
```

**Business impact**: Yeh sabse common call-tracking path hai — supplier ka call-based
lead-history isi se banta hai.

### Flow B — Call ka duration/city baad mein update hota hai

```
1. Call already record ho chuki hai (Call Record ID mil chuka hai)
2. Jab call khatam hoti hai, actual duration aur caller ki city ki information alag se aati hai
3. System existing record ko usi Call Record ID se update kar deta hai (naya record nahi banata)
```

**Business impact**: Call-start aur call-end alag-alag time pe report ho sakte hain — system
dono ko sahi se same record mein consolidate karta hai.

### Flow C — Buyer ka GLID call ke waqt pata nahi tha (Unidentified Call)

```
1. Buyer call karta hai lekin uska GLID pata nahi (jaise widget se anonymous call)
2. System is call ko "Unidentified" record ke roop mein save karta hai (ek alag table mein)
3. Baad mein, agar buyer identify ho jaaye (login kare, ya kisi tarah GLID match ho jaaye):
   a. Agar yeh ek IMOB/App-type call thi -> system asli, "identified" call-record bhi
      automatically bana deta hai (Flow A jaisa), aur dono records ko link kar deta hai
   b. Agar yeh ek PNS (virtual-number) call thi -> system sirf "probable number/GLID" attach
      kar deta hai us unidentified record pe
```

**Business impact**: Koi bhi genuine call lost nahi hoti sirf isliye ki buyer login nahi tha
call ke waqt — system usse baad mein "claim"/complete kar sakta hai.

### Flow D — Buyer apni pending unidentified calls dekhta hai

```
1. Buyer/system ek "GA cookie" (browser-tracking ID) ya apna GLID deke poochta hai: "meri
   koi pending/unidentified call hai kya?"
2. Agar GA cookie diya -> saari unidentified calls jo us cookie se aayi thi, list mil jaati hai
3. Agar GLID (receiver/supplier) diya -> pichle 5 minute ki saari calls jo abhi tak "claim"
   nahi hui (koi probable number attach nahi), list mil jaati hai
```

**Business impact**: Yeh ek lookup/matching-assist hai — jaise supplier-side UI dikha sake
"abhi-abhi ek anonymous call aayi thi, kya yeh ek known buyer hai?"

### Flow E — PNS (virtual-number) call ko IndiaMART ke C2C record se link karna

```
1. Buyer ek virtual/masked number pe call karta hai (privacy ke liye PNS system use hota hai)
2. PNS apna khud ka call-record maintain karta hai (pre-call/active state, phir post-call/
   completed state)
3. System check karta hai: kya pichle 5 minute mein isi caller-receiver pair ka koi C2C call
   record already bana tha? Agar haan -> dono records ko link kar deta hai
4. Agar nahi mila -> sirf ek link-entry ban jaati hai jo baad mein match hone ka wait karti hai
5. Lead Management System ko is link ke baare mein alag se batayaa jaata hai
```

**Business impact**: Buyer ne agar masked/virtual number se call ki (jo aksar privacy-conscious
suppliers/categories mein hota hai), phir bhi uska poora call-trail IndiaMART ke system mein
ek connected picture banata hai — supplier/Lead-Management ko fragmented data nahi dikhta.

### Flow F — Call ka "context" (enquiry ya lead se related) auto-detect hona

```
1. Kuch specific app-screens (Message Center, Lead Manager, jaisi screens) se aayi calls ke
   liye, system automatically check karta hai: yeh call kisi existing enquiry se related hai,
   ya kisi purchased-lead se?
2. Dono mein se jo zyada recent hai, usi se call ko tag kar deta hai
3. Buyer/supplier se yeh manually poocha nahi jaata — system khud correlate karta hai
```

**Business impact**: Call ka business-context automatically capture hota hai, taaki
Lead Management/reporting ko pata rahe ki yeh call kis enquiry/lead ke silsile mein hui thi.

---

## 5. Business Rules — Plain Language Mein

1. **Caller GLID, Receiver GLID, aur ModID teeno mandatory hain identified-call ke liye** —
   in mein se koi missing ho toh call record nahi banta.
2. **Caller aur Receiver same GLID nahi ho sakte** — koi khud ko khud call nahi kar sakta
   (system-level guard).
3. **Ek din mein 50+ alag suppliers ko call karna block ho jaata hai** — yeh
   spam/abuse-calling se bachaata hai.
4. **`MY` ModID explicitly disallowed hai iss endpoint pe**, chahe woh general list mein valid
   ModID ho.
5. **Unidentified-call flow mein `INSERT_INTO` decide karta hai naya record banega ya purana
   update hoga** — naya record banane ke liye Receiver GLID aur ek GA-cookie mandatory hain;
   update karne ke liye purana Record ID mandatory hai.
6. **PNS-type unidentified update sirf "probable number" attach karta hai**, jabki normal
   App-type unidentified update seedha ek asli identified call-record bhi bana deta hai —
   dono alag business outcomes hain isi ek "update" operation ke andar.
7. **LinkC2CPNS mein state (`pre`/`post`) aur timestamp dono valid format mein hone chahiye**,
   warna request reject ho jaati hai.
8. **Pre-call linking ke liye PNS Active ID, post-call linking ke liye PNS ID mandatory hai** —
   dono alag identifiers hain PNS system ke do alag lifecycle-stages ke liye.
9. **Call-context (enquiry vs lead) auto-detection sirf specific app-screens/ModIDs ke liye
   chalta hai** — Message Center, Lead Manager jaisi pehchani hui screens se aayi calls ke
   liye hi yeh extra correlation hoti hai, baaki calls ke liye nahi.

---

## 6. Notifications / Downstream Awareness

| Kab | Kise pata chalta hai | Kaise |
|---|---|---|
| Identified call record ban gayi | Lead Management System | Background queue push |
| PNS call, C2C record se link ho gayi | Lead Management System | Background queue push |
| Unidentified call record ban gayi | **Koi turant LMS notification nahi** — sirf ek "PI"/insert-only DB write hoti hai; identify hone tak yeh silent rehti hai | — |
| Buyer/supplier ko koi email/SMS | **Call Enquiry group ke scope mein nahi hai** — yeh sirf call ko record/link karta hai; koi communication yahan se trigger nahi hoti | — |

---

## 7. Edge Cases — Product/Business Perspective se

1. **"Maine call ki lekin usse koi lead nahi dikh rahi"** — Pehle check karo yeh call
   "identified" thi ya "unidentified." Agar unidentified thi aur abhi tak identify/link nahi
   hui hai, woh normal Lead Management pipeline mein nahi hogi.

2. **"Main bahut supplier ko call nahi kar pa raha"** — Yeh ek intentional rate-limit hai
   (50 unique suppliers/din), spam-calling rokne ke liye — bug nahi.

3. **"Meri call ka context (kis enquiry/lead se related thi) galat/khaali dikh raha hai"** —
   Yeh sirf specific app-screens (Message Center, Lead Manager types) se hi auto-detect hoti
   hai. Baaki jagah se call karne pe yeh field khaali rehna expected hai.

4. **"PNS se aayi call IndiaMART call-record se link nahi ho payi"** — Linking sirf 5-minute
   ke window mein hi try hoti hai (caller-receiver pair ke last C2C record ke against). Agar
   yeh window miss ho gaya, link nahi banega, sirf ek standalone entry rahegi.

5. **"Unidentified call baad mein do baar identified ho gayi kya?"** — Business process
   yeh depend karta hai ki `INSERT_INTO=PU` (update) call sahi se sirf ek baar ho, "PU" ke
   through App-type update dobara call-record chain trigger kar sakta hai agar galti se
   dobara call ho jaaye — team ko is idempotency ka dhyan rakhna chahiye (Technical Doc,
   Open Questions dekho).

---

## 8. Quick Summary — Ek Line Mein Har Cheez

- Call Enquiry group Save-Enquiry ka "call" equivalent hai — buyer-supplier phone-interactions
  ko track karta hai.
- 4 endpoints: identified calls (`callEnquiry`), unidentified/anonymous calls
  (`callEnquiryUnidentified`), unidentified-call lookup (`unidentifiedC2C`), aur PNS
  (virtual-number) linking (`linkC2CPNS`).
- Ek daily rate-limit (50 unique suppliers) spam-calling rokta hai.
- Anonymous calls baad mein "identify"/"link" ho sakti hain — data lost nahi hota.
- Har identified/PNS-linked call Lead Management System ko batayi jaati hai; pure-unidentified
  calls silent rehti hain jab tak identify na ho jaayein.
- Call ka business-context (enquiry/lead se related) kuch specific app-screens ke liye
  automatically detect hota hai.

---

## See also

- [`Call_Enquiry_Technical_Doc.md`](./Call_Enquiry_Technical_Doc.md) — same flows, code-level
  detail (APIs, DB tables, queries, RabbitMQ/Kafka)
- [`../Save Enquiry KT/Save_Enquiry_Business_Doc.md`](../Save%20Enquiry%20KT/Save_Enquiry_Business_Doc.md) — text-enquiry equivalent of this flow; both feed the same Lead Management System
- [`../Approve-Reject-Finish Enquiry KT/Approve_Reject_Finish_Enquiry_Business_Doc.md`](../Approve-Reject-Finish%20Enquiry%20KT/Approve_Reject_Finish_Enquiry_Business_Doc.md) — text-enquiry lifecycle resolution; conceptually parallel but does not directly interact with Call Enquiry

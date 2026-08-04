# Privacy Setting — Business Doc (Product Perspective)

Yeh doc "Privacy Setting" feature ko **business/product nazariye** se samjhata hai — koi
code nahi, sirf "kya hota hai, kyun hota hai, aur supplier/business ke liye iska matlab kya
hai." Technical implementation ke liye
[`Privacy_Setting_Technical_Doc.md`](./Privacy_Setting_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

"Privacy Setting" IndiaMART ka wo feature hai jisse supplier control karta hai ki **unke
baare mein kya dikhega, kya nahi**, aur **unhe kis-kis tarah ke notification/emails milenge,
kin ki nahi**. Yeh ek single toggle nahi hai — yeh bahut saare chhote-chhote independent
switches ka collection hai, har ek apna alag purpose serve karta hai.

Examples jo iss system ke through control hote hain:
- Kya supplier ka mobile number publicly dikhna chahiye?
- Kya supplier ko marketing emails chahiye ya nahi?
- Kya kisi specific type ki alert email band karni hai (unsubscribe)?
- Kaunse notification "automated" tarike se trigger honge vs manually?

**Business impact**: Yeh feature supplier ko **control aur trust** deta hai — unhe lagta hai
platform unki privacy respect karta hai, aur unhe sirf wahi communication milti hai jo woh
chahte hain. Yeh bhi ek **compliance angle** hai — jab koi supplier email "unsubscribe"
karta hai, platform ko legally/practically usse honor karna hota hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Apni settings on/off karta hai seller panel se |
| **IndiaMART ka internal system** | Kai jagah automatically settings change karta hai (jaise "is_automated" flag wale cases) |
| **Email/notification system** | Privacy setting ke basis pe decide karta hai kisko kaunsa email/alert bhejna hai |
| **Search/Directory system** | Kuch settings (jaise "phone number publicly dikhna") search results ko affect karte hain |

---

## 3. Main Business Flows (step-by-step, no code)

### Flow A — Supplier ek normal setting toggle karta hai (e.g. "show my mobile number")

```
1. Supplier seller panel mein jaake apni ek setting ON/OFF karta hai
2. System validate karta hai: yeh ek valid setting hai? Supplier ke paas permission hai?
3. Setting save ho jaati hai — kab, kisne, kis IP se change kiya, sab record hota hai
   (audit trail ke liye)
4. Background mein: yeh change platform ke doosre parts tak automatically propagate hota hai
   - Search-directory ko batao (agar setting search-visible field ho)
   - Kuch specific settings ke liye: supplier ke main profile record pe bhi ek quick-lookup
     flag update hota hai (taaki system baar-baar detailed lookup na kare)
```

**Business impact**: Change turant apply hota hai supplier ke experience mein, aur baaki
platform (search, notifications) ko bhi sync mein rakha jaata hai — bina supplier ko
disturb kiye ya wait karwaye.

### Flow B — Kuch settings "inverted" logic follow karte hain (special case)

Kuch specific setting types (jaise ek settings ka "show/hide" toggle) ka internal storage
**ulta** hai jo screen pe dikhta hai usse. Yaani jab supplier "hide" select karta hai, system
ke andar ek "insert" operation hota hai, aur jab "show" select karta hai, "delete" operation
hota hai (ya vice versa, setting type ke hisaab se). Yeh ek technical implementation detail
hai jo supplier ko dikhta nahi, lekin internal logs padhte waqt confusing lag sakta hai agar
na pata ho.

**Business impact**: Supplier ke liye koi farak nahi padta — UI pe sab kuch expected tarike
se kaam karta hai. Lekin agar kabhi koi internal debugging ho ("iss supplier ki setting
history mein Insert kyun likha hai jab unhone toggle OFF kiya tha"), yeh jaanna zaroori hai.

### Flow C — Email Frequency setting (special, sabse alag path)

```
1. Supplier apni email-frequency preference change karta hai (jaise "kam emails bhejo")
2. Yeh normal privacy-setting table mein SEEDHA save NAHI hota
3. Iski jagah, ek direct message bheja jaata hai email-system ko
   ("Mail Frequency Alert" queue ke through)
4. Email system khud decide karta hai apne hisaab se implement kaise karna hai
```

**Business impact**: Yeh ek unique case hai jahan privacy-setting UI se aane wala request
seedha ek **doosre system (email engine)** ko forward ho jaata hai — bina apni khud ki
database table update kiye. Agar koi poochhe "mera email-frequency setting kahin dikh nahi
raha standard settings table mein," yeh expected hai — yeh yahan store hi nahi hota.

### Flow D — Email Unsubscribe (compliance-critical case)

```
1. Supplier ek specific type ki email/notification "Disabled" karta hai
   (aur ek "source" bhi record hota hai — jaise unsubscribe link se aaya ya settings page se)
2. System yeh detect karta hai ki yeh ek "unsubscribe-worthy" change hai
3. Ek alag "Email Unsubscribers" list mein supplier ko add kar diya jaata hai
4. Agar supplier baad mein usi setting ko "Enabled" wapas kare -> unsubscribe list se hataa
   diya jaata hai
```

**Business impact**: Yeh feature ka sabse **compliance-sensitive** part hai — agar yeh sahi
se kaam na kare, supplier ko unwanted emails milte rahenge unsubscribe karne ke baad bhi,
jo ek bada trust/legal issue ban sakta hai.

### Flow E — Settings wapas dekhna

```
Supplier ya internal tool "GET /setting" call karta hai -> system unke saare current
privacy settings (on/off status har ek ka) ek saath return kar deta hai
```

**Business impact**: Seller panel pe UI ko yeh data chahiye hota hai toggles sahi state mein
dikhane ke liye.

**Ek zaroori nuance**: "settings dekhna" API ke andar teen tarah ke calls chhupe hote hain —
do "on karo"/"off karo" wale (jo actually ek change hi trigger karte hain, sirf read jaisa
dikhta hai), aur ek asli "dikhao" wala. Konsa version chalega, yeh depend karta hai
supplier ke app/browser ke version pe — purane app versions ko ek simpler, thoda alag
tareeke se data milta hai naye versions ke comparison mein. Business ke liye iska matlab:
agar kisi purane app-version wale supplier ka data thoda different dikhe ya "Seller
Assistant"-jaisi extra details missing ho, iska reason unka app-version ho sakta hai.

### Flow F — Secure link ke through settings access (bina normal login token ke)

```
1. Kabhi-kabhi supplier ko ek email ya link ke through directly settings page pe le jaaya
   jaata hai (jaise ek "unsubscribe" ya "manage preferences" link)
2. Yeh link ek special, time-limited signed-token carry karta hai (7 din tak valid)
3. System token ko decode karta hai aur verify karta hai ki token mein embedded email,
   supplier ke account ke email se match karta hai
4. Match hone pe hi settings dikhayi jaati hain — normal login/session ke bina bhi
```

**Business impact**: Yeh supplier ko convenience deta hai — unhe login kiye bina bhi ek
email link se seedhe apna preference change karne deta hai — lekin security ke liye is link
ki ek expiry hai aur email-match zaroori hai, taaki koi purana ya forwarded link misuse na ho
sake.

---

## 4. Business Rules — Plain Language Mein

1. **Har setting ka apna alag ID hota hai** — yeh ek master list se aata hai jahan har
   setting ka apna "default behavior" bhi defined hai (agar supplier ne kabhi touch hi nahi
   kiya, toh default kya maana jaaye).
2. **Kuch settings "automated" bhi ho sakti hain** — matlab system khud decide kar sakta hai
   unhe on/off karna, supplier ke manual input ke bina, specific setting types ke liye.
3. **Mail-frequency ek alag flow hai, normal setting-save flow se bahar** (Flow C) — isse
   directly email-system handle karta hai.
4. **Email unsubscribe automatically track hota hai** jab bhi koi setting "Disabled" ho aur
   uske saath ek valid "source" ho — yeh manual process nahi hai, system khud detect karta
   hai.
5. **Har change ka audit-trail banta hai** — kaun, kab, kahan se (IP), kya change kiya, sab
   record hota hai. Yeh support/compliance investigations ke liye important hai.
6. **Har setting ka apna "default" hota hai — aur system sirf "exception" record karta hai**:
   agar koi setting default se "ON" hoti hai sabke liye, toh system tabhi ek record banata
   hai jab koi usse explicitly OFF kare (aur vice versa agar default "OFF" hai). Iska matlab:
   agar kisi supplier ne kabhi kuch touch hi nahi kiya, unka data database mein nahi milega —
   yeh missing data nahi hai, yeh unke default state ko represent karta hai.
7. **Kuch settings ke saath extra "sub-details" bhi judi hoti hain** — jaise Seller Assistant
   se related kuch settings (AI-chat, calling-preference jaisi) ke saath ek extra JSON detail
   bhi store/return hota hai, baaki normal settings ke saath aisa kuch nahi hota.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine email-frequency change kiya, lekin settings page pe woh dikh nahi raha"** — Yeh
   expected hai. Yeh setting seedha email-engine ko forward hoti hai, apni database mein
   store nahi hoti — isliye general settings-read API mein woh reflect nahi hoga.

2. **"Maine unsubscribe kiya tha, phir bhi email aaya"** — Investigate karne layak, kyunki
   unsubscribe-tracking sirf specific conditions mein trigger hota hai (setting "Disabled"
   ho AUR ek valid "source" record ho saath mein). Agar source missing tha, unsubscribe list
   mein add hi nahi hua hoga.

3. **Kuch settings supplier ke "quick-lookup" profile flag ko bhi update karte hain, sab
   nahi** — sirf specific, chuni hui settings (5 particular setting types) iss extra step ko
   trigger karti hain. Baaki settings sirf apni normal table tak limited rehti hain.

4. **Internal logs mein "Insert/Delete" dikhna kabhi confusing lag sakta hai** — kuch setting
   types ke liye toggle ON/OFF internally Insert/Delete operations ke roop mein store hote
   hain, seedhe "value = true/false" ke bajaye (Flow B).
5. **"Read" wala API kabhi-kabhi actually ek write bhi kar deta hai** — settings dekhne wale
   hi API ke andar, do specific request-shapes asal mein setting ko turant on/off bhi kar
   dete hain. Agar koi tool ya monitoring script sirf "check karne" ke liye is API ko baar-baar
   call kar raha ho un dono shapes ke saath, woh anjaane mein settings change kar sakta hai.
6. **Purane aur naye app-versions ko thoda alag response mil sakta hai** — teen alag internal
   paths hain settings dikhane ke, aur inmein occasionally chhoti-si inconsistency ho sakti
   hai (jaise koi setting ek path mein "default enabled" dikhe, doosre mein nahi) — agar kabhi
   koi complain kare "meri setting ka default state alag app pe alag dikh raha hai," yeh iska
   root-cause ho sakta hai.

---

## 6. Quick Summary

- Privacy Setting = supplier ka control-panel unki visibility aur communication preferences
  pe.
- Har setting independent hai, apni master-list se define hoti hai.
- Email-frequency ek special case hai jo seedha email-system ko forward hota hai.
- Email-unsubscribe automatically track hota hai compliance ke liye.
- Har change audit-trail banata hai.

---

## See also

- [`Privacy_Setting_Technical_Doc.md`](./Privacy_Setting_Technical_Doc.md) — same flows,
  code-level detail (APIs, DB tables, queries, RabbitMQ, consumers)
- [`../users_domain_product_stories.md`](../users_domain_product_stories.md) — Privacy
  Setting ka role bigger "Notifications, Alerts & Preferences" story mein

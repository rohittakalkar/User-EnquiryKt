# Terms And Condition Acceptance — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai — koi code nahi, sirf "kya hota hai, kyun
hota hai, aur supplier/business ke liye iska matlab kya hai." Code ke liye
[`Terms_And_Condition_Technical_Doc.md`](./Terms_And_Condition_Technical_Doc.md) dekho.

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Jab supplier IndiaMART ke **Terms & Conditions accept** karta hai (jaise signup ke waqt, ya
kisi policy-update ke baad re-acceptance), yeh event ek **log-entry** ke roop mein record hota
hai — kab accept kiya, kis IP address se, kis country se, aur kis device/browser
(`USER_AGENT`) se accept kiya.

Yeh feature 3 cheezon ka backbone hai:

1. **Legal/compliance proof**: Agar kabhi dispute ho ki supplier ne T&C accept kiya tha ya
   nahi, yeh log ek concrete, timestamped proof ke roop mein kaam aata hai.
2. **Audit trail**: Har acceptance apna alag record banata hai — purana kabhi overwrite nahi
   hota, isliye poori history hamesha available rehti hai (supplier ne kitni baar, kab-kab
   accept kiya).
3. **Session/status trigger**: Successful acceptance ek downstream-event bhi fire karta hai
   jo supplier ki session/status update karta hai kahin aur (system exactly kya update karta
   hai, yeh downstream system iss review mein trace nahi ho paaya — dekho Open Questions).

**Bottom line**: Yeh feature ek simple, append-only compliance-log hai — legal-safety ke liye
zaroori, business-logic ke hisaab se bahut simple.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | T&C accept karta hai — web, app, ya kisi bhi internal-tool ke through |
| **Bahut saare internal-apps (33-entry allowlist)** | Yeh apps/channels acceptance-event record kar sakte hain — jaise seller panel, mobile app (Android/iOS), telecalling tools, admin tools, waghera |
| **Downstream session/status system** | Har successful acceptance ke baad ek notification-event isse jaata hai — is system ka exact naam/behavior code se conclusively confirm nahi ho paaya (dekho Open Questions) |

---

## 3. Business Flow (step-by-step, no code)

```
1. Supplier T&C accept karta hai (app/web ya kisi internal-tool ke through)
2. System pehle check karta hai: request kis app/channel se aa rahi hai — sirf allowed
   (whitelisted) channels se hi aage badhta hai
3. Basic mandatory-info check hota hai: user-ID, IP, device/browser info sab present hai?
4. Sab sahi hone par: is acceptance ko ek naye log-entry ke roop mein save kiya jaata hai —
   user-ID, IP, IP-country, country-code, device/browser info, aur date
5. Successful-save hone par, ek downstream-event fire hoti hai jo supplier ki session/status
   ko kahin aur update karti hai
6. Supplier ko koi turant-visible confirmation nahi milta (yeh ek background compliance-log
   hai, koi popup/email is specific step ke liye code mein nahi mila)
```

**Business impact**: Simple, append-only compliance-log — har acceptance apna record banata
hai, purana overwrite nahi hota. Isse compliance/legal team ko kabhi bhi complete history
mil sakti hai kisi bhi supplier ki.

---

## 4. Status/State Table — Kya track hota hai

T&C acceptance feature ka apna koi multi-step "status" nahi hai (jaise GST ke paas
Verified/Rejected/Pending states hain) — yeh sirf ek binary hai: **record ban gaya (success)
ya nahi bana (failure)**. Isliye is domain mein alag se status-table zaroori nahi, lekin yeh
table clarify karta hai ki record banne ke baad kya-kya store hota hai:

| Field jo record mein save hota hai | Business meaning |
|---|---|
| Supplier ka user-ID | Kisne accept kiya |
| IP address | Kahan se accept kiya (network-level) |
| IP-country + country-code | Geographically kahan se accept kiya |
| User-agent (device/browser) | Kis device/app se accept kiya |
| Date | Kab accept kiya |

---

## 5. Business Rules — Plain Language Mein

1. **Har acceptance ek fresh record hoti hai** — insert-only, koi update/overwrite nahi hota.
   Agar supplier baar-baar accept kare (jaise galti se double-click), har baar ek naya record
   banega — yeh expected hai, bug nahi.
2. **Bahut saare internal-apps allowed hain** — ek badi allowlist (33 entries) — matlab
   T&C-acceptance kayi channels (web, mobile app, internal admin-tools, telecalling tools) se
   ho sakti hai, sirf ek single "official" app se nahi.
3. **Successful-save ke baad, ek session-update-event trigger hoti hai** — supplier ki
   session/status kahin aur update hoti hai isi event ke through. Exact downstream-effect
   kya hai, yeh clearly document nahi mila (dekho Open Questions) — business team ko iska
   awareness rakhna chahiye ki yeh ek "fire and forget" style event hai.
4. **Koi duplicate-check nahi hai** — system yeh check nahi karta ki supplier ne pehle se
   aaj/is-session mein accept kiya hai ya nahi; har valid request ek naya record bana degi.
   Iska matlab hai ki agar UI kabhi galti se do baar submit-button trigger kare, database mein
   do records ban jaayenge — yeh functionally harmless hai (compliance-log ka design hi
   append-only hai) lekin worth janna hai agar koi "kitni baar accept hua" count nikale.
5. **Koi turant visible confirmation/notification nahi hai supplier ke liye** — yeh purely ek
   backend compliance-record hai; koi email/SMS iss specific action ke liye trigger nahi hoti
   (GST jaisi domains mein hota hai "Your GST updated" email — yahan T&C ke liye aisa kuch
   code mein nahi mila).

---

## 6. Notifications — Kya jaata hai, kisko

| Kab | Kya hota hai |
|---|---|
| Supplier T&C accept karta hai | Koi direct email/SMS supplier ko nahi jaata (code mein aisa kuch nahi mila) |
| Successful acceptance ke baad | Ek internal session/status-update event trigger hoti hai (downstream-system, supplier ko directly visible nahi) |
| Acceptance fail ho jaaye (jaise validation fail, ya DB issue) | Koi notification kisi ko nahi jaati — sirf internal logging (Kibana) mein capture hota hai |

---

## 7. Edge Cases — Product/Business Perspective se

1. **"Maine T&C dobara accept kiya"** — Expected hai, har baar naya record banega, history
   preserve hoti hai. Yeh koi error nahi hai.
2. **"Mujhe koi confirmation nahi mila T&C accept karne ka"** — Expected hai, yeh feature by
   design silent/background hai (koi email/SMS trigger nahi hoti is specific action ke liye).
3. **"Downstream session/status update ka exact effect kya hai, yeh support team ko pata nahi
   chal raha"** — Iska seedha jawaab code se available nahi hai; yeh ek genuine gap hai jo
   business/engineering dono ko clarify karna chahiye (dekho Open Questions).
4. **Bahut saare channels se acceptance aa sakti hai** — agar kabhi doubt ho ki kisi
   particular channel/app se acceptance record ban raha hai ya nahi, remember karo ki
   allowlist bahut badi (33 entries) hai — zyada chance hai ki wo channel already allowed hai.

---

## 8. Quick Summary

- Terms And Condition Acceptance = compliance-log, insert-only, per-acceptance-event.
- Bahut saare internal-apps allowed (33-entry allowlist) — web, app, aur internal-tools sab
  se ho sakti hai.
- Har acceptance apna fresh record banata hai — koi duplicate-check nahi, koi overwrite nahi.
- Supplier ko koi direct notification (email/SMS) nahi milti is action ke liye — purely
  background compliance-record.
- Successful-insert ek downstream session-update-event trigger karta hai jiska exact
  business-effect code se conclusively confirm nahi ho paaya.
- Yeh iss poore KT-series mein sabse simplest business-flow wala feature hai — ek hi step,
  ek hi outcome, koi multi-stage verification ya approval workflow nahi (GST jaisi domains
  ke ulat).

---

## See also

- [`Terms_And_Condition_Technical_Doc.md`](./Terms_And_Condition_Technical_Doc.md) — code-level detail

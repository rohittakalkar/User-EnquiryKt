# Bizfeed & Recommendation — Business Doc (Product Perspective)

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Bizfeed_And_Recommendation_Technical_Doc.md`](./Bizfeed_And_Recommendation_Technical_Doc.md)
dekho.

**Scope note — kya-kya combined hai aur kyun**:

| Concept | Is folder mein? | Kyun |
|---|---|---|
| User Activity | Haan | Buyer Activity ke saath ek hi write-API aur ek hi Cassandra-table share karta hai (`activity_type_id` se differentiate hota hai) |
| Catalog View | Haan | Usi pipeline ka downstream-processing-stage hai (`CATALOG_VIEW_SERVICE` consumer) |
| Buyer Activity | Haan | Yeh pipeline ka entry-point hai |
| Bizfeed CV Hide/Unhide | Haan | Alag write-controller hai, lekin **isi pipeline ke output-table** (`glusr_usr_biz_feeds`) ko directly modify karta hai — isliye same-domain |
| Recommendation (Action-Items) | Haan | Ek dashboard-aggregator hai jo Bizfeed-data ko (aur doosre services ko) combine karke ek "recommended actions" list banata hai |
| **Latitude & Longitude Details** | **Nahi** | Genuinely alag feature/table hai — dekho [`../Location Update KT/`](../Location%20Update%20KT/) |
| **Last Seen** | **Nahi** | Codebase mein kahin bhi nahi mila (koi table, controller, ya field iss naam se exist nahi karta) — agar yeh feature future mein banaya jaaye, tab hi documented ho sakta hai |

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

**Buyer Activity/User Activity** — jab bhi ek buyer IndiaMART pe kuch browse karta hai
(product-search, category-browse, product-view), yeh activity capture hoti hai — kya
dekha, kis keyword se search kiya, kis supplier ka product tha, location kya thi.

**Catalog View** — isi activity-data ka ek processed/derived-form hai — supplier ko
dikhata hai ki unke catalog/products ko kaun-kaun dekh raha hai ("aapke product ko is
buyer ne dekha").

**Bizfeed** — yeh "Catalog View" aur baaki activity-signals ka combined output hai — ek
feed jo supplier ko unke business ke against relevant-events dikhata hai.

**Bizfeed Hide/Unhide** — supplier is feed ke individual-items ko "hide" kar sakta hai
agar wo unke liye relevant nahi hai — feed ko personalize karne ka ek tarika.

**Recommendation (Action Items)** — ek dashboard-widget hai jo supplier ko batata hai
unka business-profile complete karne ke liye kya-kya "next steps" hain — jisme Bizfeed-
data bhi ek input hai (baaki inputs: product-listing-completeness, credits, unread-
messages, etc.).

**Business impact**: Yeh poora "engagement-loop" hai — buyer ka browsing-behavior
capture hota hai, supplier ko uske against actionable-insights milte hain, aur supplier
apna profile improve karne ke liye guided-recommendations paata hai.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Buyer** | Browse/search karta hai — activity generate hoti hai |
| **Supplier** | Bizfeed dekhta hai, items hide/unhide karta hai, recommendations follow karta hai |
| **Internal-tools (GLADMIN, LEAP)** | Activity-events likh sakte hain |

---

## 3. Business Flow (step-by-step, no code)

```
1. Buyer koi activity perform karta hai (search, product-view, category-browse)
2. Yeh activity ek raw-event ke roop mein capture hoti hai (activity-type, keyword,
   category, location, supplier jiska product dekha, etc.)
3. Ek background-process is raw-event ko process karta hai aur "business-feed" items
   generate karta hai — supplier-specific, personalized
4. Supplier apna Bizfeed dekhta hai — "yeh buyer ne aapka product dekha", etc.
5. Supplier kisi feed-item ko "hide" kar sakta hai agar woh unke liye useful nahi hai
6. Ek alag "Recommendation" dashboard bhi hai jo Bizfeed-summary ko baaki business-
   health-signals (product-completeness, credits, messages) ke saath combine karke
   ek "action-items" list deta hai
```

**Business impact**: Real-time buyer-signal se supplier-facing-insight tak ka poora
data-pipeline — raw-clickstream se personalized-recommendation tak.

---

## 4. Business Rules — Plain Language Mein

1. **Activity-events ek generic-schema mein capture hote hain** — `activity_type_id`
   decide karta hai yeh kaunsi activity hai (User Activity ka general-type ya
   Catalog View-specific).
2. **Hide/unhide sirf ek specific feed-item pe apply hota hai** — (supplier, buyer,
   date) ke combination se identify hoke.
3. **Recommendation-dashboard multiple services se parallel data-fetch karta hai** —
   sirf Bizfeed nahi, product-listing, credits, unread-messages sab combine hote hain.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine feed-item hide kiya, phir bhi dikh raha hai"** — Ho sakta hai match-criteria
   (glusrid+buyerid+date) exact match na ho raha ho.
2. **"Mera Recommendation-dashboard slow load ho raha hai"** — Expected hai agar
   koi ek internal-service (jinme se yeh parallel-fetch karta hai) slow respond kar
   rahi ho.

---

## 6. Quick Summary

- Bizfeed & Recommendation = buyer-activity-capture → processed-catalog-view/bizfeed →
  supplier-facing-feed (hide/unhide-capable) → recommendation-dashboard-aggregator.
- User Activity + Catalog View + Buyer Activity ek hi pipeline hain.
- Lat/Long aur Last Seen is domain ka hissa nahi hain (dekho scope-note upar).

---

## See also

- [`Bizfeed_And_Recommendation_Technical_Doc.md`](./Bizfeed_And_Recommendation_Technical_Doc.md) — code-level detail
- [`../Location Update KT/Location_Update_Business_Doc.md`](../Location%20Update%20KT/Location_Update_Business_Doc.md) —
  unrelated, separately-documented Lat/Long feature

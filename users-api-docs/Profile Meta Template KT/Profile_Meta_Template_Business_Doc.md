# Profile / Meta & Template (Company & PDP Content) — Business Doc

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Profile_Meta_Template_Technical_Doc.md`](./Profile_Meta_Template_Technical_Doc.md) dekho —
dono docs same flows cover karte hain, bas alag audience ke liye.

**Scope**: yeh doc teen closely-related features cover karta hai jo saath milke supplier
ka **storefront (company profile)** page aur uske **SEO/meta content** ko manage karte hain:

1. **Profile** — supplier ke storefront ka core content (Awards, Media, News, Testimonials,
   Infrastructure, Jobs, Quality, Custom sections, etc.)
2. **Template** — storefront ke andar ke sections/tabs, plus kuch top-level tracking/
   branding fields (Google Analytics code, tab titles)
3. **Meta** — SEO metadata (title, description, keywords) — category/product-listing pages
   ke liye ("PDP content")

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Jab ek buyer kisi supplier ka storefront page (company profile) dekhta hai, unhe sirf
contact-details nahi dikhtin — poora ek **rich, branded page** dikhta hai: awards,
testimonials, company ki photos, infrastructure details, news/updates, aur custom sections
jo supplier khud design kar sakta hai.

Yeh poora experience 3 features milke banate hain:

- **Profile**: core storefront content — company ki basic branding/description, images,
  documents
- **Template**: storefront ke andar ke individual "tabs"/sections — jaise "Awards" tab,
  "Infrastructure" tab, "Testimonials" tab, plus kuch analytics/tracking codes jo
  storefront page pe embed hote hain
- **Meta**: yeh page google/search-engines mein kaise dikhega (title, description,
  keywords) — category/product-listing pages ka SEO isi se control hota hai

**Business impact**: Ek strong storefront supplier ko zyada credible dikhata hai buyers ke
liye. Achi SEO metadata ka matlab hai supplier ki listing Google search mein better rank
kare — dono directly business/leads impact karte hain.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Apna storefront content edit karta hai — profile sections, tabs, images, docs, SEO metadata |
| **Buyer** | Storefront page dekhta hai, decide karta hai supplier trustworthy hai ya nahi |
| **Search engines (Google etc.)** | Meta content ko index karte hain — SEO ranking iske through affect hoti hai |
| **Internal automated content-moderation system** | Har teeno feature (Profile/Template/Meta) ke free-text description ko background mein scan karta hai abuse/PII ke liye, aur agar flag ho toh usse **khud hi blank kar deta hai** (see section 5, rule 8) |
| **Internal/automated caller surfaces (GLADMIN, WEBERP, TOLLFREE, Notification Server, etc.)** | Template content kaafi zyada internal/automated processes se bhi likha ja sakta hai, sirf supplier se nahi |

---

## 3. Teeno Features — Alag-Alag Samjho

### 3.1 — Profile

**Kya hai**: Supplier ke storefront ka core, section-wise content — Awards, Media,
Infrastructure, Jobs, News, Quality, Testimonial, Custom-Profile, aur 3 "Extended Profile"
slots, har ek apni images (max 5) aur documents ke saath.

**Business flow**:
```
1. Supplier apna profile-section content update karta hai (POST /profile)
2. System validate karta hai, purani images/docs pe naye replace karta hai
3. Content-moderation background mein check karti hai (abuse/PII scan)
4. Buyers ya supplier khud (apni "MY" view mein) yeh content GET /profile se dekhte hain
   — supplier apne rejected content bhi dekh sakta hai, buyer nahi
```

**Business impact**: Har section (Awards, Testimonials, etc.) independently update hoti
hai. Supplier ka apni profile ka control granular hai.

### 3.2 — Template (Storefront Sections/Tabs)

**Kya hai**: Storefront ke andar ke individual sections — Awards, Testimonials,
Infrastructure, News, Custom Sections, aur do "Tab Title" fields jo kisi bhi Template
request ke saath side-effect ke roop mein update ho sakte hain. Supplier har section ko
independently manage kar sakta hai.

**Business flow**:
```
1. Supplier ek specific section-type choose karta hai (jaise "Awards")
2. Us section ka content submit karta hai (POST /template)
3. System yeh section-specific record save karta hai (alag section, alag record)
4. Yeh bhi content-moderation se guzarta hai
5. Yehi content GET /profile se hi wapas dikhta hai (ek special flag ke saath) —
   Template ka apna alag "GET" endpoint nahi hai
```

**Business impact**: Modular design hai — supplier chahe toh sirf "Awards" tab update kare,
baaki sections touch kiye bina. Har section apni khud ki content-lifecycle rakhta hai.
Template internal teams (sales, admin, automated processes) se bhi kaafi zyada likha jaata
hai — sirf supplier-facing feature nahi hai.

### 3.3 — Meta (SEO / Category & Product-Listing Content — "PDP")

**Kya hai**: SEO-focused metadata — page title, meta-description, keywords — jo category
aur product-listing pages ke liye set hoti hai (yehi "PDP content" hai — Product/
category-listing page ka search-engine-facing content).

**Business flow**:
```
1. Supplier (ya seller/buyer-panel se koi bhi authorized caller) ek category-section ke
   liye SEO details submit karta hai (category name, meta title, meta description, keywords)
2. System yeh directly save kar leta hai (koi heavy multi-step process nahi)
3. Yeh bhi content-moderation background check se guzarta hai (description field pe)
4. Search engines is metadata ko consume karte hain
```

**Business impact**: Yeh feature buyer ko seedha nahi dikhta — iska impact
**discoverability** pe hota hai (Google search results mein kaisi dikhti hai listing).

**Important gap noticed is baar**: is review mein Meta content ko wapas "read" karne ka
koi confirmed tareeka nahi mila platform ke in teen core repos mein — matlab agar koi
poochhe "meri submitted SEO metadata kahan dikhti hai," iska jawaab conclusively code se
nahi mila. Ho sakta hai yeh kisi doosre (search-indexing) system se seedha consume hoti ho.

---

## 4. Content-Moderation — Ek Zaroori Cross-Cutting Business Rule

Teeno features (Profile/Template/Meta) ka free-text "description" field ek **automated
background moderation check** se guzarta hai submit hone ke baad:

```
1. Supplier profile/template/meta content submit karta hai, request turant "success" return karta hai
2. Background mein, ek automated system us description ko scan karta hai abuse/PII ke liye
3. Agar content clean hai -> kuch nahi hota, content wahi rehta hai
4. Agar content flag ho jaaye (abuse/PII detected) ->
   system khud us description ko chupke se BLANK kar deta hai, koi alag rejection
   notification supplier ko nahi jaati usi waqt
```

**Business impact**: Yeh ek safety-net hai jo buyer-facing content ko clean rakhta hai bina
manual review team ko har submission check karne ki zaroorat ke. Lekin isse yeh bhi hota
hai ki agar supplier ka koi genuine content galti se flag ho jaaye, unhe seedha pata nahi
chalega ki unka description kyun gayab ho gaya — support ko yeh pattern pata hona chahiye.

---

## 5. Business Rules — Plain Language Mein

1. **Teeno features alag-alag POST APIs hain** (`/profile`, `/template`, `/meta`) — ek
   dusre se independent submit hote hain, ek combined "save all" nahi hai.
2. **Content-moderation teeno pe lagti hai** (section 4) — Profile, Template, aur Meta
   teeno ka description-type field background mein scan hota hai, sirf Profile/Template
   nahi.
3. **Template modular hai per-section** — har section-type (Awards/Testimonials/etc.) apna
   alag record maintain karta hai, ek doosre ko affect nahi karte.
4. **Meta abhi tak sirf category-level SEO cover karti hai** — sirf `category_section` wala
   path mila; product-level (individual PDP) meta ka support iss review mein nahi mila.
5. **Profile ke images/docs hamesha poori tarah replace hote hain, partial update nahi** —
   jab bhi naye images/docs submit karte ho, purane sab hata ke naye poore set se replace
   ho jaate hain (ek hi atomic step mein) — partial "sirf ek image change karo" jaisa
   koi concept nahi hai internally, chahe UI wahi lage.
6. **Ek profile-section mein max 5 images allowed hain** — isse zyada submit karne pe
   system explicitly reject karta hai.
7. **Owner khud apna content submit kare toh woh auto-approved ho jaata hai** — agar
   "last modified by" flag "Owner" hai, system status ko force "Approved" set kar deta hai,
   pending-approval queue mein nahi jaata.
8. **Supplier apna rejected content khud dekh sakta hai, buyer nahi** — jab supplier apni
   khud ki profile "MY" view mein dekhta hai, unhe unka rejected content bhi dikhta hai
   (taaki woh fix kar sakein); buyer ko sirf approved content dikhta hai.
9. **Template ek bahut bada caller-base allow karta hai** — sirf supplier hi nahi, balki
   23 alag internal/automated systems (GLADMIN, WEBERP, TOLLFREE, Notification Server,
   etc.) bhi Template content likh sakte hain — Meta ka allowlist bahut chhota hai (sirf 2).
10. **Meta ka success-check ek exact text-match pe depend karta hai** — internal system
    ek specific success-string check karta hai; iska matlab hai future development mein
    agar yeh string kabhi accidentally badal jaaye, saari Meta requests silently "failed"
    dikhengi chahe DB update actually ho chuki ho.

---

## 6. Notifications — Supplier ko kab pata chalta hai

| Kab | Kya hota hai |
|---|---|
| Profile/Template/Meta content successfully update hua | API response mein turant "Success" milta hai |
| Content-moderation ne kuch flag kiya | **Koi explicit rejection notification nahi** — description silently blank ho jaata hai background mein; supplier ko sirf tab pata chalega jab woh apna storefront dekhega ki description gayab hai |
| Rejected profile content | Supplier apni "MY" view mein rejected content dekh sakta hai (auto-notified nahi, dikhta hai jab woh apni profile check kare) |

---

## 7. Edge Cases — Product/Business Perspective se

1. **"Maine Awards tab update ki, baaki storefront change nahi hua"** — Expected hai,
   Template modular design follow karta hai — har section independent hai.
2. **"Meri category listing Google pe purani metadata dikha rahi hai"** — Search-engine
   re-crawl mein time lagta hai; system-side update turant hota hai, lekin Google ka index
   refresh alag timeline follow karta hai (yeh iss module ke control se bahar hai).
3. **"Mera description submit hone ke turant baad gayab ho gaya"** — Yeh content-moderation
   ka kaam ho sakta hai (section 4) — agar submitted text mein kuch flag-worthy laga
   (abuse/PII jaisa), system usse automatically blank kar deta hai bina ek turant, explicit
   rejection notice ke. Support team ko yeh pattern pata hona chahiye taaki confuse na ho.
4. **"Maine 6 images upload karne ki koshish ki, kaam nahi hui"** — Expected hai, system
   max 5 images per section allow karta hai.
5. **"Meri profile edit turant reflect nahi hui"** — Content-moderation background check ho
   sakti hai, jaisa Profile/Template/Meta teeno ke saath hota hai (GST/Rating docs mein bhi
   yehi cross-domain pattern dekha gaya hai).
6. **"Meri SEO metadata kahan dikhti hai, main confirm nahi kar pa raha"** — Iss review mein
   Meta ka koi confirmed "read back" path nahi mila platform ke in repos mein; team se
   confirm karna padega yeh kis system se serve hoti hai.

---

## 8. Quick Summary

- Profile = storefront ka core section-wise content (Awards/Media/News/etc.), Template =
  storefront ke individual sections/tabs (plus kuch tracking codes), Meta = SEO/
  category-listing metadata ("PDP content").
- Teeno alag APIs hain, independently submit hote hain.
- Teeno hi background content-moderation se guzarte hain — flagged content silently blank
  ho jaata hai, ek explicit rejection notice ke bina.
- Profile ke images/docs hamesha poori tarah replace hote hain, max 5 images per section.
- Owner khud submit kare toh auto-approved; supplier apna rejected content khud dekh sakta
  hai, buyer nahi.
- Template ka read-path seedha `/profile` ke through hi hai (ek special flag ke saath),
  Meta ka koi confirmed read-path iss review mein nahi mila.
- Meta ka business-impact discoverability pe hai, Profile/Template ka credibility pe.

---

## See also

- [`Profile_Meta_Template_Technical_Doc.md`](./Profile_Meta_Template_Technical_Doc.md) —
  code-level detail (APIs, DB tables, RabbitMQ, moderation-consumer internals)
- [`../users_domain_product_stories.md`](../users_domain_product_stories.md) — Story 2
  "Profile & Business Identity Management" mein yeh sab pehle bhi documented hain

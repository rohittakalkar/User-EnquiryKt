# Profile / Meta & Template (Company & PDP Content) — Business Doc

Business/product perspective ke liye yeh doc hai. Code ke liye
[`Profile_Meta_Template_Technical_Doc.md`](./Profile_Meta_Template_Technical_Doc.md) dekho.

**Scope**: yeh doc teen closely-related features cover karta hai jo saath milke supplier
ka **storefront (company profile)** page aur uske **SEO/meta content** ko manage karte hain:

1. **Profile** — supplier ke storefront ka core content
2. **Template** — storefront ke andar ke sections/tabs (awards, testimonials, infrastructure,
   news, custom sections)
3. **Meta** — SEO metadata (title, description, keywords) — category/product-listing pages
   ke liye, jinhe user "PDP content" bhi bolta hai

---

## 1. Yeh feature hai kya, aur kyun zaroori hai

Jab ek buyer kisi supplier ka storefront page (company profile) dekhta hai, unhe sirf
contact-details nahi dikhtin — poora ek **rich, branded page** dikhta hai: awards,
testimonials, company ki photos, infrastructure details, news/updates, aur custom sections
jo supplier khud design kar sakta hai.

Yeh poora experience 3 features milke banate hain:

- **Profile**: core storefront content — company ki basic branding/description
- **Template**: storefront ke andar ke individual "tabs"/sections — jaise "Awards" tab,
  "Infrastructure" tab, "Testimonials" tab
- **Meta**: yeh page google/search-engines mein kaise dikhega (title, description,
  keywords) — aur category/product-listing pages ka SEO bhi isi se control hota hai

**Business impact**: Ek strong storefront supplier ko zyada credible dikhata hai buyers ke
liye. Achi SEO metadata ka matlab hai supplier ki listing Google search mein better rank
kare — dono directly business/leads impact karte hain.

---

## 2. Kaun-kaun involved hai

| Kaun | Role |
|---|---|
| **Supplier** | Apna storefront content edit karta hai — profile, tabs/sections, SEO metadata |
| **Buyer** | Storefront page dekhta hai, decide karta hai supplier trustworthy hai ya nahi |
| **Search engines (Google etc.)** | Meta content ko index karte hain — SEO ranking iske through affect hoti hai |
| **Internal moderation system** | Profile/Template content ko abuse/PII ke liye check karta hai submit hone ke baad |

---

## 3. Teeno Features — Alag-Alag Samjho

### 3.1 — Profile

**Kya hai**: Supplier ke storefront ka core, top-level content.

**Business flow**:
```
1. Supplier apna profile content update karta hai (POST /profile)
2. System validate karta hai, save karta hai
3. Content-moderation background mein check karti hai (abuse/PII scan)
4. Buyers GET /profile se yeh content dekhte hain
```

### 3.2 — Template (Storefront Sections/Tabs)

**Kya hai**: Storefront ke andar ke individual sections — Awards, Testimonials,
Infrastructure, News, aur Custom Sections. Supplier har section ko independently manage kar
sakta hai.

**Business flow**:
```
1. Supplier ek specific section-type choose karta hai (jaise "Awards")
2. Us section ka content submit karta hai (POST /template)
3. System yeh section-specific record save karta hai (alag section, alag record)
4. Yeh bhi content-moderation se guzarta hai
```

**Business impact**: Modular design hai — supplier chahe toh sirf "Awards" tab update kare,
baaki sections touch kiye bina. Har section apni khud ki content-lifecycle rakhta hai.

### 3.3 — Meta (SEO / Category & Product-Listing Content — "PDP")

**Kya hai**: SEO-focused metadata — page title, meta-description, keywords — jo category
aur product-listing pages ke liye set hoti hai (yehi "PDP content" hai jo aap bol rahe the —
Product/category-listing page ka search-engine-facing content).

**Business flow**:
```
1. Supplier ek category-section ke liye SEO details submit karta hai
   (category name, meta title, meta description, keywords)
2. System yeh directly save kar leta hai (koi heavy multi-step process nahi)
3. Search engines aur internal-search dono is metadata ko consume karte hain
```

**Business impact**: Yeh feature buyer ko seedha nahi dikhta — iska impact **discoverability**
pe hota hai (Google search results mein kaisi dikhti hai listing).

---

## 4. Business Rules — Plain Language Mein

1. **Teeno features alag-alag POST APIs hain** (`/profile`, `/template`, `/meta`) — ek
   dusre se independent submit hote hain, ek combined "save all" nahi hai.
2. **Content-moderation Profile aur Template dono pe lagti hai** — abuse/PII scan submit
   hone ke baad background mein. Meta pe iss review mein koi moderation nahi mili — SEO
   text likely low-risk maana gaya hai.
3. **Template modular hai per-section** — har section-type (Awards/Testimonials/etc.) apna
   alag record maintain karta hai, ek doosre ko affect nahi karte.
4. **Meta abhi tak sirf category-level SEO cover karti hai** — iss review mein sirf
   `category_section` wala path mila; agar product-level (individual PDP) meta ka bhi
   support hona chahiye, yeh confirm karna padega ki woh kahin aur handle hota hai ya
   missing hai.

---

## 5. Edge Cases — Product/Business Perspective se

1. **"Maine Awards tab update ki, baaki storefront change nahi hua"** — Expected hai,
   Template modular design follow karta hai — har section independent hai.
2. **"Meri category listing Google pe purani metadata dikha rahi hai"** — Search-engine
   re-crawl mein time lagta hai; system-side update turant hota hai, lekin Google ka index
   refresh alag timeline follow karta hai (yeh iss module ke control se bahar hai).
3. **"Meri profile edit turant reflect nahi hui"** — Content-moderation background check ho
   sakti hai, jaisa Profile/Template content ke saath hota hai baaki domains mein bhi
   (GST/Rating docs mein bhi yehi pattern dekha).

---

## 6. Quick Summary

- Profile = storefront ka core content, Template = storefront ke individual sections/tabs,
  Meta = SEO/category-listing metadata ("PDP content").
- Teeno alag APIs hain, independently submit hote hain.
- Profile/Template moderation se guzarte hain, Meta seedha save hoti hai.
- Meta ka business-impact discoverability pe hai, Profile/Template ka credibility pe.

---

## See also

- [`Profile_Meta_Template_Technical_Doc.md`](./Profile_Meta_Template_Technical_Doc.md) —
  code-level detail
- [`../users_domain_product_stories.md`](../users_domain_product_stories.md) — Story 2
  "Profile & Business Identity Management" mein yeh sab pehle bhi documented hain

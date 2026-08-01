# Profile / Meta & Template (Company & PDP Content) — Technical Doc

Business/product perspective ke liye
[`Profile_Meta_Template_Business_Doc.md`](./Profile_Meta_Template_Business_Doc.md) dekho.

**Repos**: `users-api-go-production` (read), `service-api-go-production` (write).

**Methodology**: har claim source code se trace kiya gaya hai. Yeh doc teen features cover
karta hai — depth Template ke 9-query controller ke liye purposefully summary-level rakhi
gayi hai (har section-type individually trace nahi kiya), baaki domains jitni exhaustive
line-by-line nahi — flagged clearly jahan bhi hai.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Profile write | write | [`UserProfileController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserProfileController.go), [`UserProfileModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserProfileModel.go) (`UpsertProfile`) |
| Profile read | read | [`ProfileController.go`](../internal/controllers/UsersControllers/ProfileController.go) (function `ActionProfileController`) |
| Profile read — **dead code duplicate** | read | [`actionProfile.go`](../internal/controllers/UsersControllers/actionProfile.go) — already flagged dead in [`users_api_database_reference.md`](../API%20to%20Tables/users_api_database_reference.md), real live implementation is `ProfileController.go` |
| Template write | write | [`UserTemplateController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTemplateController.go) (function `TemplateController`), [`UserTemplateModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTemplateModel.go) |
| Meta write | write | [`UserMetaController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserMetaController.go), [`UserMetaModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMetaModel.go) (`UpsertMeta`) |

**Read-side note**: koi dedicated `GET /template` ya `GET /meta` nahi mila iss pass mein —
template/meta content likely `GET /profile` ya `GET /otherdetail` ke through hi surface
hota hai (confirm karna baaki hai, dekho Open Questions).

---

## 2. Routes (confirmed from router.go)

| Method | Path | Repo | Controller |
|---|---|---|---|
| GET/POST | `/profile/*params` | read | `ActionProfileController` |
| POST | `/profile` | write | `UserProfileController` |
| POST | `/template` | write | `TemplateController` |
| POST | `/meta` | write | `UserMetaController` |

---

## 3. Data Model — Tables

| Table | Physical DB | Purpose | Confirmed from |
|---|---|---|---|
| `PC_CLIENT` | meshpg | **Core storefront/profile record** — "PC" prefix = "Profile Client," ek shared table-family teeno features ke beech | [`UserTemplateModel.go:1537`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTemplateModel.go) |
| `PC_CLNT_PCAT` | meshpg | **Category-level content/meta** — Template aur Meta dono isse likhte hain (shared table) | [`UserTemplateModel.go:247,279`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTemplateModel.go), [`UserMetaModel.go:47`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMetaModel.go) — columns: `PC_CLNT_PCAT_NAME`, `PC_CLNT_FLNAME`, `PC_CLNT_PCAT_META_TITLE`, `PC_CLNT_PCAT_META_DESCRIPTION`, `PC_CLNT_PCAT_META_KEYWORDS` |
| `PC_CLNT_FEATURE_SECTION_VALUE` | meshpg | **Individual template-section content** — Awards/Testimonials/Infrastructure/News/Custom-section har ek ka apna record isi table mein | [`UserTemplateModel.go:721,1766,1794,1914`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTemplateModel.go) — supports INSERT (new section) aur UPDATE (existing section edit) |

**Important finding**: `PC_CLNT_PCAT` **shared table hai Template aur Meta ke beech** —
donon iske alag-alag columns likhte hain (Template category-content, Meta SEO-metadata).
Yeh confirm karta hai ki "Meta" actually category/product-listing-level content hai
("PDP content" jo aapne mention kiya), na ki company-profile-level.

---

## 4. Business Rules & Validation (code se)

### Profile
1. **`P_NO` (page number/section indicator) 0/4/5/8 ke alawa ho toh, `AddtParamsMap` se
   extra params merge hote hain** — matlab kuch specific "page numbers" ek alag/simpler
   validation path lete hain, baaki extended params accept karte hain.
   [`UserProfileController.go:78-85`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserProfileController.go)
2. **`PROFILE_IMGS`/`PROFILE_DOCS` explicitly input-formatting se exempt kiye jaate hain**
   (delete kiya jaata hai formatting se pehle, phir wapas add kiya jaata hai) — taaki image/doc
   arrays ka structure formatting-step se corrupt na ho.
   [`UserProfileController.go:43-54`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserProfileController.go)

### Template
3. **`TemplateController` sabse complex controller hai iss group mein** — 9 alag query-slots
   track karta hai execution-time ke liye, matlab ek hi request mein multiple section-types
   ek saath process ho sakte hain.
   [`UserTemplateController.go:33-56`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserTemplateController.go)
4. **`PC_CLNT_FEATURE_SECTION_VALUE` insert-vs-update dono support karta hai** — naya section
   INSERT, existing section-value UPDATE (`PC_CLNT_FEATURE_SV_DT_MODIFIED`,
   `PC_CLNT_FSV_UPDATEDBY`, etc. columns).
   [`UserTemplateModel.go:1794,1914`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserTemplateModel.go)

### Meta
5. **Gateway allowlist chhota hai — sirf `SELLERMY`/`BUYERMY`** — do specific caller-surfaces.
   [`UserMetaController.go:49`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserMetaController.go)
6. **Sirf ek content-branch mila iss pass mein**: `category_section` — `PC_CLNT_PCAT` ke
   naam/meta-title/meta-description/meta-keywords ko update karta hai. Koi product-level
   (individual PDP) branch iss review mein nahi mila.
   [`UserMetaModel.go:44-52`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserMetaModel.go)
7. **Success-message string-match pe depend karta hai**: `"UPDATE SUCCESS IN MESH PG"` exact
   match hone pe hi `code=200` — koi bhi typo/variance iss string mein silently failure treat
   ho jaayega.
   [`UserMetaController.go:71`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserMetaController.go)

---

## 5. RabbitMQ

| Queue / `SERVICENAME` | Publisher | Purpose |
|---|---|---|
| `USER_PROFILE_SERVICE` → routes to `USER_PROFILE_BANNED_DETECT` (per domain-wide `serviceToQueueMap`) | `UserProfileController.go` | Content-moderation trigger — Profile content abuse/PII scan ke liye |
| `TEMPLATE_SERVICE` → routes to `USER_PROFILE_BANNED_DETECT` (shared moderation queue, per earlier cross-domain research) | `TemplateController.go` (`globals["rabbitmq_data"]`) | Same shared moderation layer jo Profile bhi use karta hai |
| `META_SERVICE` → routes to `USER_PROFILE_BANNED_DETECT` (per `serviceToQueueMap`, confirmed earlier this session) | `UserMetaController.go` (`rabbitarray["SERVICENAME"]`) | **Interesting finding**: iska SERVICENAME bhi `USER_PROFILE_BANNED_DETECT` pe route hota hai domain-wide map ke hisaab se — matlab Meta content bhi technically moderation-queue tak pahुँच sakta hai, chahe iski apni koi dedicated processing iss pass mein nahi dikhi. Confirm karna baaki hai ki yeh actively consumed hoti hai ya nahi Meta ke context mein. |

**Shared moderation layer**: teeno features (Profile/Template/Meta) same underlying
`USER_PROFILE_BANNED_DETECT` / `USER_ATTR_BANNED_CONTENT_CHECK` consumer-family use karte
hain — yeh already [`users_domain_product_stories.md`](../users_domain_product_stories.md)
ke "Cross-story infrastructure" section mein documented hai.

---

## 6. Kafka / Redis

**Koi Kafka ya Redis usage nahi mila** iss teen-feature group mein, iss pass mein.

---

## 7. End-to-End Technical Flows

### Flow A — Profile update

```
Supplier
    │
    ▼
[API — write]  POST /profile
    │  UserProfileController.go
    │  Mandatory-param check, PROFILE_IMGS/DOCS special-handling
    ▼
[DB]  UpsertProfile() → PC_CLIENT (aur related tables, exact query iss pass mein full-trace nahi hui)
    ▼
[RabbitMQ]  SERVICENAME=USER_PROFILE_SERVICE
    ▼
[CONSUME]  USER_PROFILE_BANNED_DETECT (shared moderation layer)
```

### Flow B — Template section update

```
Supplier (chooses a section-type: Awards/Testimonials/Infrastructure/News/Custom)
    │
    ▼
[API — write]  POST /template
    │  TemplateController.go
    ▼
[DB]  INSERT or UPDATE PC_CLNT_FEATURE_SECTION_VALUE
    │  (up to 9 query-slots tracked — multiple sections potentially in one request)
    ▼
[RabbitMQ]  SERVICENAME=TEMPLATE_SERVICE
    ▼
[CONSUME]  USER_PROFILE_BANNED_DETECT (shared moderation layer)
```

### Flow C — Meta (SEO/category content) update

```
Supplier
    │
    ▼
[API — write]  POST /meta  {category_section: ..., category_meta_title: ..., ...}
    │  UserMetaController.go
    │  Gateway check (SELLERMY/BUYERMY only)
    ▼
[DB]  UPDATE PC_CLNT_PCAT SET meta_title, meta_description, meta_keywords, ...
    ▼
[RabbitMQ]  SERVICENAME=META_SERVICE
```

### Flow D — Read

```
Buyer / App
    │
    ▼
[API — read]  GET /profile
    │  ActionProfileController.go
    ▼
Response: storefront content (exact table-read iss pass mein full-trace nahi hui —
          likely PC_CLIENT + PC_CLNT_FEATURE_SECTION_VALUE join)
```

---

## 8. Optimization Scope — DB Response-Time Contribution

### Medium-impact

1. **`PC_CLNT_PCAT` shared-table-write pattern** — Template aur Meta dono is table pe
   independently write karte hain (alag columns). Agar dono APIs concurrently same
   `category_section` ke liye call ho jaayein, ek possible **race condition** ban sakti hai
   (last-write-wins ek doosre ke columns ko touch na kare toh theek hai, lekin worth confirm
   karna ki UPDATE statements column-specific hain ya poori row overwrite karte hain).
2. **`TemplateController`'s 9-query-slot design** — agar ek request mein genuinely 9 tak
   queries chal sakti hain (multiple section-types ek saath), aur yeh sab sequential hain,
   yeh GST/Rating domains jaisa hi "sequential-when-could-be-parallel" pattern ho sakta hai.
   Exact query-independence iss pass mein verify nahi hui — follow-up chahiye.

### Low-impact

3. **Meta ka single-branch, single-query design already simple hai** — koi obvious
   optimization zaroorat nahi dikhi (Blocking module jaisa hi clean, chhota footprint).

---

## 9. Full Flow Diagram (Lucid, icon-based)

**[Poora Profile/Meta/Template flowchart yahan dekho](https://lucid.app/lucidchart/0f0d4155-095c-47c0-89a8-b2954b07f834/edit)**

---

## 10. Edge Cases & Gotchas (technical POV)

1. **Meta ka `META_SERVICE` moderation-routing confirm nahi hua** (§5) — agar yeh dead-end
   route hai (koi consumer actually process nahi karta), worth flag karna.
2. **`PC_CLNT_PCAT` shared-write race condition risk** (§8, point 1).
3. **Read-side (Template/Meta content GET kaise hota hai) fully trace nahi hua** — agar koi
   "meri Template edit dikh nahi rahi" jaisi ticket aaye, sabse pehle confirm karo yeh
   content `GET /profile` se hi aata hai ya kahin aur.
4. **`actionProfile.go` dead-code duplicate hai** — pehle bhi flag ho chuka hai
   [`users_api_database_reference.md`](../API%20to%20Tables/users_api_database_reference.md)
   mein, yahan cross-reference ke liye dobara note kiya.

---

## 11. Open Questions

1. Template/Meta content ka read-path exact confirm karo — `GET /profile` ke andar hi hai,
   ya alag endpoint?
2. `META_SERVICE` queue ka `USER_PROFILE_BANNED_DETECT` tak routing actually consumed/processed
   hoti hai ya nahi?
3. `PC_CLNT_PCAT` pe Template aur Meta ke concurrent writes ke liye koi row-level locking ya
   column-specific UPDATE hai (poori-row-overwrite se bachne ke liye)?
4. Kya Meta domain mein koi product-level (individual PDP, na ki category) branch hai jo iss
   pass mein miss ho gaya?
5. `UpsertProfile()` ka poora query-trace (kaunse exact tables/columns) — iss pass mein sirf
   function-existence confirm hui, full trace nahi.
6. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`Profile_Meta_Template_Business_Doc.md`](./Profile_Meta_Template_Business_Doc.md) —
  product perspective
- [`../users_domain_product_stories.md`](../users_domain_product_stories.md) — Story 2
  "Profile & Business Identity Management," including the shared moderation layer note
- [`../API to Tables/users_api_database_reference.md`](../API%20to%20Tables/users_api_database_reference.md) —
  `actionProfile.go` dead-code finding, originally documented here

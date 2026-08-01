# User Image — Technical Doc (Code-Level Deep Dive)

**Scope note**: yeh doc sirf **Image** (`GLUSR_USR_IMAGE`) cover karta hai — **Logo**
(`GLUSR_USR_LOGO`, apna approval-workflow ke saath) genuinely alag table/system hai, dekho
[`../Logo KT/Logo_Technical_Doc.md`](../Logo%20KT/Logo_Technical_Doc.md). Dono
`FK_GLUSR_USR_ID` se supplier se link hain, ek doosre se FK-linked nahi.

Business/product perspective ke liye [`Image_Business_Doc.md`](./Image_Business_Doc.md) dekho.

**Repos**: `service-api-go-production` (write). **Koi consumer nahi mila** — Logo ke ulat,
Image ka koi RabbitMQ fan-out/consumer iss pass mein nahi mila, purely synchronous hai.

---

## 1. File Map

| Concern | Repo | File |
|---|---|---|
| Image write | write | [`UserImageController.go`](../../service-api-go-production/service-api-go-production/internal/controllers/UserControllers/UserImageController.go), [`UserImageModel.go`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserImageModel.go) |

---

## 2. Routes

| Method | Path | Repo | Controller |
|---|---|---|---|
| POST | `/user/image` | write | `UserImageController` |

---

## 3. Data Model — Table

| Table | Physical DB | Purpose | Key columns (code se) |
|---|---|---|---|
| `GLUSR_USR_IMAGE` | meshpg | General branding-image record, **no approval-ID/history-versioning** (unlike Logo) | `FK_GLUSR_USR_ID`, `GLUSR_USR_IMAGE_IMG_32X32`/`_32X32_WH`, `_64X64`/`_64X64_WH`, `_125X125`/`_125X125_WH`, `_250X250`/`_250X250_WH`, `_IMG_ORIG`/`_ORIG_WH`, `GLUSR_USR_IMAGE_STATUS`, `GLUSR_USR_IMAGE_REJECT_REASON`, `GLUSR_USR_IMAGE_UPDATEDBY`, `_UPDATESCREEN`, `_IP`/`_IP_COUNTRY`, `_UPDATEDUSING`, `_HIST_COMMENTS`, `_UPDATED_TIME`, `_UPDATEDBY_URL` — [`UserImageModel.go:137-157,179+`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserImageModel.go) |

**Note**: har image-size ke saath ek `_WH` (width-height, likely aspect-ratio or dimension
metadata) companion-column bhi hai — Logo table mein yeh nahi tha, ek extra tracking-detail
jo Image-specific hai.

---

## 4. Business Rules & Validation (code se)

1. **UPDATE-first pattern**: query pehle `UPDATE GLUSR_USR_IMAGE ... WHERE
   FK_GLUSR_USR_ID=$20` try karti hai; `affectedRows > 0` check karke, agar 0 hon, tabhi
   `INSERT INTO GLUSR_USR_IMAGE (...)` chalta hai — **update-then-insert-fallback**, Logo
   ke explicit-FLAG-based branching se ulta (yahan system khud detect karta hai insert
   chahiye ya update, client ko FLAG batana nahi padta).
   [`UserImageModel.go:137,175-179`](../../service-api-go-production/service-api-go-production/internal/models/UserModels/UserImageModel.go)
2. **`STATUS`/`REJECT_REASON` columns exist karte hain**, lekin **koi separate
   approval-ID/versioning-history nahi** jaisa Logo mein tha — simpler moderation-model
   (agar koi moderation hoti bhi hai, single-record-level hai, multi-version-history nahi).
3. **`_WH` (width-height) companion columns har size ke saath** — Logo se yeh ek structural
   difference hai, dimension-metadata explicitly store hoti hai har resolution ke liye.

---

## 5. RabbitMQ / Kafka / Redis

**Koi RabbitMQ, Kafka, ya Redis usage nahi mila** iss feature mein — Blocking/Social-Review/
TrustSeal jaisa hi purely synchronous, single-table write.

---

## 6. End-to-End Technical Flow

```
Supplier
    │
    ▼
[API — write]  POST /user/image  {images at 5 resolutions, status, reject_reason, ...}
    │  UserImageController.go
    ▼
[DB]  UPDATE GLUSR_USR_IMAGE SET ... WHERE FK_GLUSR_USR_ID=$N
    │
    ├─ Rows affected > 0 → done, "UPDATE SUCCESS"
    │
    └─ 0 rows affected →
          [DB]  INSERT INTO GLUSR_USR_IMAGE (...)
    ▼
Response
```

---

## 7. Optimization Scope — DB Response-Time Contribution

### Low-impact

1. **Update-then-insert-fallback ek predictable 1-2-query pattern hai** — first-time upload
   ke liye 2 round-trips (failed-update + insert), baad ke updates ke liye 1 round-trip
   (successful update). Yeh acceptable hai, lekin agar first-time-upload bahut common case
   hai (naye suppliers), ek `INSERT ... ON CONFLICT DO UPDATE` (jaisa Social Review/Blocking
   modules mein already use ho raha hai) isse **hamesha 1 round-trip** bana sakta hai —
   consistent single-statement pattern, GST/Rating docs mein bhi yeh suggestion repeat hui
   hai.

### Low-impact / good practice already present

2. **Koi async complexity nahi, koi consumer-fan-out nahi** — simple, predictable, low-risk
   architecture, jaisa Blocking/Social-Review modules.

---

## 8. Full Flow Diagram (Lucid, icon-based)

**[Poora Image flowchart yahan dekho](https://lucid.app/lucidchart/f74781b6-b751-4cc4-a3ee-248652bf2074/edit)**

---

## 9. Edge Cases & Gotchas (technical POV)

1. **Update-then-insert pattern race-condition-risk**: agar do concurrent requests ek naye
   supplier ke liye ek saath aayein (dono ko UPDATE se 0 rows milein), dono INSERT try kar
   sakte hain — agar `FK_GLUSR_USR_ID` pe unique-constraint nahi hai, duplicate rows ban
   sakte hain. Confirm karo constraint exist karta hai ya nahi.
2. **`GLUSR_USR_IMAGE_STATUS`/`REJECT_REASON` ka exact business-usage** — koi moderation-
   consumer ya admin-tool iss pass mein nahi mila jo inhe actively set kare, worth confirm
   karna.

---

## 10. Open Questions

1. `FK_GLUSR_USR_ID` pe unique-constraint hai kya (race-condition prevention ke liye)?
2. `STATUS`/`REJECT_REASON` ko kaun set karta hai — koi moderation-tool/admin-flow?
3. Read-side (`GET`) kaise hoti hai — koi dedicated controller nahi mila, `otherdetail` se
   hi confirm nahi hua.
4. Live DB schema verification — is doc ne sirf Go SQL strings jo imply karti hain wahi
   reflect kiya hai.

---

## See also

- [`Image_Business_Doc.md`](./Image_Business_Doc.md) — product perspective
- [`../Logo KT/Logo_Technical_Doc.md`](../Logo%20KT/Logo_Technical_Doc.md) — related but
  separate approval-workflow-based logo system

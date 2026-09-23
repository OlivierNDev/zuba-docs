# ZUBA Product Upload — Full Improvement Specification

> **Document type:** CTO-level technical specification  
> **Status:** Approved for implementation  
> **Version:** 1.0 — September 2026  
> **Scope:** Manual upload fixes + Batch upload system + Media storage strategy + 1688.com integration

---

## 1. Executive Summary

ZUBA's current product upload pipeline is a single-staff manual editor that proxies all media through NestJS, creating memory pressure, tight video size caps, and no path for bulk catalog ingestion. This specification defines a four-phase upgrade: (1) replace proxied uploads with signed direct Cloudinary writes and switch videos to URL-only, (2) build a CSV-driven batch import system capable of ingesting 400+ products asynchronously, (3) establish a clear media storage decision framework per asset type, and (4) wire the China-side 1688.com team into the batch workflow so sourced products flow from supplier → Alibaba Cloud OSS → ZUBA catalog without manual intervention.

---

## 2. Current State vs Target State

| Dimension | Current State | Target State |
|---|---|---|
| **Image upload path** | Browser → NestJS proxy → Cloudinary | Browser → Signed Cloudinary URL (direct) |
| **Hero video** | POST `/api/product/uploadVideo` — 5 MB cap, proxied | URL field only — any CDN host, no size limit |
| **Instruction video** | POST `/api/product/uploadInstructionVideo` — 50 MB proxied through Nest | URL field only — same as hero video |
| **Product creation method** | Manual editor only (one product at a time) | Manual editor + CSV batch import (400+ rows) |
| **Batch import** | Not available | CSV → validate → async queue → DRAFT products |
| **Publish gate** | Manual checkbox per product | Manual: unchanged · Batch: "Publish if complete" checkbox |
| **Error visibility** | None for upload failures | Per-row error table + downloadable error CSV |
| **Progress tracking** | Spinner only | Live % bar — rows processed / rows failed |
| **Media for sourced products** | Manual staff copy-paste from 1688 | China team posts OSS URLs directly into CSV template |
| **NestJS memory risk** | High (50 MB video buffer in-process) | Zero — no file buffers in Nest ever |
| **Video responsiveness** | Fixed-size embed | URL-driven player: poster image, lazy, no autoplay |

---

## 3. Phase 1 — Signed Direct Uploads + Video URL (Manual Path Fix)

### 3.1 Problem Summary

NestJS is currently a memory buffer for every uploaded file. A 50 MB instruction video means a 50 MB Node.js Buffer lives in RAM for the duration of the Cloudinary push. Under concurrent staff sessions this becomes an OOM risk. The 5 MB hero video cap is a symptom — it was set defensively because the proxy can't safely handle more.

### 3.2 What Changes

#### Backend — NestJS (`product.controller.ts`, `product.service.ts`)

| Current Endpoint | Change |
|---|---|
| `POST /api/product/uploadImages` | **Keep endpoint but change behavior.** Now returns a Cloudinary signed upload URL + `upload_preset` instead of forwarding the file. The browser uploads directly. |
| `POST /api/product/uploadVideo` | **Remove file upload.** New endpoint: `POST /api/product/validateVideoUrl` — accepts `{ url: string }`, validates the URL is reachable (HEAD request, timeout 5s), returns `{ valid: true, contentType, estimatedSizeMb }`. |
| `POST /api/product/uploadInstructionVideo` | **Remove file upload.** Folded into same `validateVideoUrl` call — the `type` param distinguishes hero vs instruction. |

#### Frontend — React/Next (`ProductEditorPage`, `ProductMediaEditor`)

| Component | Change |
|---|---|
| `ProductMediaEditor` — image upload | Replace `FormData → /api/product/uploadImages` with: (1) fetch `/api/product/signedUploadParams`, (2) POST directly to `https://api.cloudinary.com/v1_1/{cloudName}/image/upload`. No change to UX. |
| `ProductMediaEditor` — hero video | Remove `<input type="file">`. Replace with `<input type="url" placeholder="Paste video URL (1688 CDN, OSS, Bunny.net…)">`. On blur, call `validateVideoUrl`. Show green check or red error. |
| `ProductMediaEditor` — instruction video | Same as hero video — URL input, validate on blur. |

---

## 4. Phase 2 — Batch Upload System

### 4.1 CSV Template Specification

| Column | Type | Required | Description |
|---|---|---|---|
| `name` | string | ✅ | Product display name. Max 120 chars. |
| `description` | string | ✅ | Must be ≥ 40 characters (matches publish gate). |
| `price` | decimal | ✅ | Selling price in USD. Must be > 0. |
| `sku` | string | ✅ | Unique SKU. Alphanumeric + hyphens. |
| `category` | string | ✅ | Must match an existing `categories.slug` in DB. |
| `design_line` | string | ✅ | Must match an existing `design_lines.slug` in DB. |
| `manufacturer` | string | ✅ | Manufacturer name string. |
| `source_url` | URL | ✅ | 1688.com product page URL (Type B provenance). |
| `stock` | integer | ✅ | Quantity on hand. ≥ 0. |
| `weight_kg` | decimal | optional | Shipping weight. Defaults to `0` if blank. |
| `is_active` | boolean | optional | `true`/`false`. Defaults to `true`. |
| `publish_if_complete` | boolean | optional | If `true` and all gates pass, product goes LIVE on import. Defaults to `false` (DRAFT). |
| `image_1` … `image_12` | URL | At least 1 required | Public image URLs. Empty columns are skipped. |
| `hero_video_url` | URL | optional | Hero video URL (1688 CDN / OSS / Bunny.net). |
| `instruction_video_url` | URL | optional | Instruction video URL. |
| `tags` | string | optional | Comma-separated tags. |
| `meta_title` | string | optional | SEO title. Max 60 chars. |
| `meta_description` | string | optional | SEO description. Max 160 chars. |

### 4.2 Validation Rules

**Pass 1 — Structural (synchronous, before any DB writes)**
- File is valid CSV (comma-delimited, UTF-8)
- Header row matches expected column set
- Row count ≤ 1000
- No duplicate `sku` values within the same CSV

**Pass 2 — Per-row (async, same publish gates as manual editor)**
- `name` present and ≤ 120 chars
- `description` ≥ 40 chars
- `price` is a positive number
- `sku` is unique in `nest_products`
- `category` slug exists in DB
- `design_line` slug exists in DB
- `manufacturer` present
- `source_url` is a valid URL
- `stock` ≥ 0 integer
- At least one non-empty `image_N` URL
- `hero_video_url` (if present) passes `validateVideoUrl`

### 4.3 Async Job Architecture

- Upload CSV → NestJS parses + Pass 1 validates → enqueues BullMQ job → returns `{ jobId }`
- Worker chunks rows (50/chunk) → Pass 2 per-row validate → INSERT to `nest_products` (DRAFT)
- Progress stored in Redis, polled via `GET /api/product/batch/:jobId/status`
- Failed rows written to error report → downloadable error CSV

### 4.4 New Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/product/batch/template` | Download CSV template |
| `POST` | `/api/product/batch/upload` | Upload CSV, returns `{ jobId }` |
| `GET` | `/api/product/batch/:jobId/status` | Poll job progress |
| `GET` | `/api/product/batch/:jobId/errors` | Download error CSV for failed rows |
| `GET` | `/api/product/batch/history` | List past batch jobs for this staff user |

---

## 5. Phase 3 — Media Storage Decision

| Asset Type | Recommended Host | Rationale |
|---|---|---|
| **Staff-uploaded product images** (manual editor) | **Cloudinary** | Already integrated, Phase 1 keeps this |
| **1688-sourced product images** (batch import, short-term) | **Direct alicdn.com URL** | Zero cost for initial catalog build |
| **1688-sourced product images** (long-term / flagship) | **Re-host to Cloudinary async** | ZUBA owns the asset permanently |
| **Hero video** | **Alibaba Cloud OSS + CDN** | China team uploads locally, fastest cross-border delivery |
| **Instruction video** | **Alibaba Cloud OSS + CDN** | Same reasoning — large files, China origin |

---

## 6. Phase 4 — 1688.com Integration Batch Flow

1. China team sources products on 1688.com
2. Uploads hero videos to Alibaba Cloud OSS → gets permanent CDN URLs
3. Fills batch CSV template: name, price (CNY converted + margin), SKU (`ZUBA-CN-XXXXXX`), source_url, image URLs (alicdn.com or OSS), hero_video_url (OSS CDN)
4. Uploads CSV via ZUBA Admin → Batch Upload Tab
5. Valid rows → `nest_products` (DRAFT)
6. ZUBA Product Ops reviews drafts → publishes approved products → storefront

### 6.1 OSS Bucket Structure

```
zuba-media/
  products/
    {sku}/
      images/  (01.jpg, 02.jpg…)
      video/   (hero.mp4, instructions.mp4)
```

CDN: `https://oss-cdn.zubahouse.com` (CNAME → Alibaba CDN)

---

## 7. Performance & Reliability Rules

- All product images served through CDN — never raw origin
- Videos: URL-only, `preload="none"`, poster = first product image, never autoplay on load
- Muted autoplay on scroll-into-view only (match Temu behaviour)
- Fallback if video URL 404: hide player, show image gallery
- First 8 product images must load < 1s on LTE — enforce via Cloudinary `q_70` on thumbnails
- Batch max CSV: 5 MB, max rows: 1000
- BullMQ worker concurrency: 2, transaction per 50-row chunk
- Job TTL: 24 hours (error CSVs must be downloaded within window)

---

## 8. Build Roadmap

| Phase | What | Priority | Effort |
|---|---|---|---|
| **1a** | Signed direct Cloudinary upload (replace Nest proxy) | 🔴 Critical | 1 day |
| **1b** | Remove video file upload; add URL input + validateVideoUrl | 🔴 Critical | 0.5 day |
| **1c** | Frontend: image direct upload + video URL input | 🔴 Critical | 1 day |
| **2a** | CSV template endpoint | 🟡 High | 0.5 day |
| **2b** | Batch upload endpoint + Pass 1 validation | 🟡 High | 1 day |
| **2c** | BullMQ worker + Pass 2 validation + DB insert | 🟡 High | 2 days |
| **2d** | Job status polling + error CSV generation | 🟡 High | 1 day |
| **2e** | Batch Upload UI (tab, drop zone, preview, progress, results) | 🟡 High | 2 days |
| **3a** | Lock Assets — async alicdn → Cloudinary migration | 🟢 Medium | 1.5 days |
| **3b** | OSS bucket setup + CDN CNAME for video | 🟢 Medium | 0.5 day |
| **4a** | product_sourcing_log table + 1688 helper columns | 🟢 Medium | 0.5 day |
| **4b** | 1688 batch template variant | 🟢 Medium | 0.5 day |
| **5** | Video player component (web + mobile) | 🔵 Polish | 1 day |

**Total: ~14 engineering days (2–3 sprints)**

---

*End of specification. All phases are independent and can be shipped incrementally without breaking the existing manual editor workflow.*
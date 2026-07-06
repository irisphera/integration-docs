# Irisphera API Integration Guide

Implementation guide for integrators that onboard fashion brands through the Irisphera public API, **API version 2.0.0**.

Download the OpenAPI 3.1 spec from `https://api.irisphera.com/v3/api-docs` to generate client code in your language with [OpenAPI Generator](https://openapi-generator.tech). All examples use the production server `https://api.irisphera.com` and these shell variables:

## Guide Map

1. [Scope and credentials](#1-scope-and-credentials)
2. [Errors](#2-errors)
3. [End-to-end sequence](#3-end-to-end-sequence)
4. [Merchant lifecycle (integrator)](#4-merchant-lifecycle-integrator)
5. [Collections and visitor tokens (merchant)](#5-collections-and-visitor-tokens-merchant)
6. [Prepare product data](#6-prepare-product-data)
7. [Queue ingestion and verify](#7-queue-ingestion-and-verify)
8. [Merchant operations](#8-merchant-operations)
9. [Shopper session and identity](#9-shopper-session-and-identity)
10. [Shopper experience APIs](#10-shopper-experience-apis)
11. [Shopper analytics events](#11-shopper-analytics-events)
12. [Runtime notes and limitations](#12-runtime-notes-and-limitations)
13. [Troubleshooting](#13-troubleshooting)
14. [Launch checklist](#14-launch-checklist)

## 1. Scope and credentials

### Three authentication contexts

| Context | Credential | Used on |
| --- | --- | --- |
| Integrator | Integrator API key | `/integrator/v1` routes |
| Merchant | Caller-created merchant API key | `/merchant/v1` routes (server-side only) |
| Visitor | Bearer access token (JWT) | `/shopper/v1` routes |


### Secret handling

- Store the integrator key and each merchant key in an enterprise secret manager and inject them at runtime; never store them in source, feeds, images, shared documents, or plaintext configuration.
- Keep access to the integrator key separate from access to each managed merchant key.
- Avoid real `curl -i` output in shared terminals or CI logs: merchant responses contain `apiKey`.
- Keep TLS verification enabled for the API, image URLs, and feed URLs.

## 2. Errors

Declared errors use `application/problem+json` (RFC 9457):

| Field | Meaning |
| --- | --- |
| `type` | URI identifying the problem type |
| `title` | Short human-readable summary |
| `detail` | Occurrence-specific explanation |
| `status` | HTTP status code |
| `instance` | URI identifying this occurrence |
| `requestId` | Correlation identifier |

Some `422` responses return a bare `{"detail": [...]}` instead. Treat the HTTP status as the primary control signal. Log `requestId` for support correlation; never log API keys or bearer tokens.

## 3. End-to-end sequence

1. Obtain the integrator key through your Irisphera enterprise process.
2. Generate a merchant API key in your secret manager.
3. Create the merchant with the integrator key; store its `merchantId`.
4. Create a collection with the merchant key; store its `collectionId`.
5. Prepare a single product, a bulk JSON feed, or a CSV feed.
6. Queue one ingestion route with the merchant key.
7. Verify the product appears in the collection's `fashionItems`.
8. Issue visitor tokens from your backend with the visitor's `shopperId`; call `identify` once when an anonymous shopper logs in.
9. Report commerce events from your backend.

## 4. Merchant lifecycle (integrator)

All six integrator routes derive ownership from the integrator key. Don't send an `integratorId` in any body. Lists contain only merchants you own; another integrator's merchant returns `403`, a missing merchant returns `404`.

### Merchant request and response

`POST` and `PUT` share the request shape:

| Field | Required | Rules and default |
| --- | --- | --- |
| `name` | Yes | Nonempty string |
| `apiKey` | Yes | Nonempty, caller-created merchant key |
| `themeConfig.themeFile` | No | Default `default.json` |
| `flowConfig.apparel` | No | `MENSWEAR`, `WOMENSWEAR`, or `ALL`; default `ALL` |
| `flowConfig.recommendationCriteria` | No | `NONE`, `PALETTE`, `SILHOUETTE`, `SIZING`, or `ALL`; default `ALL` |
| `flowConfig.profileWizard` | No | `SIMPLE` or `MANNEQUIN`; default `MANNEQUIN` |
| `flowConfig.recommendationTopK` | No | Integer of at least 10; default `50` |

Responses contain `id`, `name`, `apiKey`, `themeConfig`, and `flowConfig`.

### Routes

| Route | Behavior |
| --- | --- |
| `GET /integrator/v1/merchant` | Lists your merchants; `X-Total-Count` equals the count |
| `POST /integrator/v1/merchant` | Creates a merchant; reconciles existing ones (see below) |
| `GET /integrator/v1/merchant/{merchantId}` | Gets one merchant |
| `PUT /integrator/v1/merchant/{merchantId}` | Updates one merchant |
| `GET /integrator/v1/merchant/{merchantId}/apikey` | Returns the stored merchant key |
| `DELETE /integrator/v1/merchant/{merchantId}` | Permanently deletes the merchant |

#### List merchants

```bash
curl -i -X GET "$IRISPHERA_BASE_URL/integrator/v1/merchant" \
  -H "Accept: application/json" -H "irisphera-api-key: $IRISPHERA_INTEGRATOR_API_KEY"
```

`200` — an array of your merchants.

#### Create a merchant

```bash
curl -i -X POST "$IRISPHERA_BASE_URL/integrator/v1/merchant" \
  -H "Accept: application/json" -H "Content-Type: application/json" \
  -H "irisphera-api-key: $IRISPHERA_INTEGRATOR_API_KEY" \
  -d '{"name":"Example Fashion Brand","apiKey":"'"$MERCHANT_API_KEY"'"}'
```

`201` — the merchant, including `id`. Store it as `$MERCHANT_ID`.

Create is reconciliatory: if the `apiKey` already belongs to you, the existing merchant is returned unchanged; if the `name` already belongs to you, that merchant is updated in place. You can safely retry create after an uncertain outcome.

#### Get one merchant

```bash
curl -i -X GET "$IRISPHERA_BASE_URL/integrator/v1/merchant/$MERCHANT_ID" \
  -H "Accept: application/json" -H "irisphera-api-key: $IRISPHERA_INTEGRATOR_API_KEY"
```

`200` — the merchant. Reading a legacy merchant can backfill missing configuration with server defaults.

#### Update a merchant

```bash
curl -i -X PUT "$IRISPHERA_BASE_URL/integrator/v1/merchant/$MERCHANT_ID" \
  -H "Accept: application/json" -H "Content-Type: application/json" \
  -H "irisphera-api-key: $IRISPHERA_INTEGRATOR_API_KEY" \
  -d '{"name":"Example Fashion Brand Europe","apiKey":"'"$MERCHANT_API_KEY"'","flowConfig":{"apparel":"ALL","recommendationCriteria":"SIZING","profileWizard":"MANNEQUIN","recommendationTopK":60}}'
```

`200` — the updated merchant. `name` and `apiKey` always replace their current values; omitted `themeConfig` or `flowConfig` values are preserved, not reset. Ownership can't be reassigned.

#### Get a merchant API key

```bash
curl -i -X GET "$IRISPHERA_BASE_URL/integrator/v1/merchant/$MERCHANT_ID/apikey" \
  -H "Accept: application/json" -H "irisphera-api-key: $IRISPHERA_INTEGRATOR_API_KEY"
```

`200` — `{"merchantId": "<uuid>", "apiKey": "<merchant-key>"}`. Use only when an authorized process needs the stored key; treat the response as a secret. No rotation, expiry, or revocation behavior is declared.

#### Delete a merchant

```bash
curl -i -X DELETE "$IRISPHERA_BASE_URL/integrator/v1/merchant/$MERCHANT_ID" \
  -H "Accept: application/json" -H "irisphera-api-key: $IRISPHERA_INTEGRATOR_API_KEY"
```

`204` — no body. Deletion is permanent, removes the merchant and its dependent data, and is irreversible; a repeated request returns `404`.

### Declared statuses

| Operation | Success | Other declared statuses |
| --- | --- | --- |
| List merchants | `200` | `401`, `403`, `500` |
| Create merchant | `201` | `400`, `401`, `403`, `500` |
| Get merchant | `200` | `400`, `401`, `403`, `404`, `500` |
| Update merchant | `200` | `400`, `401`, `403`, `404`, `500` |
| Get merchant key | `200` | `400`, `401`, `403`, `404`, `500` |
| Delete merchant | `204` | `400`, `401`, `403`, `404`, `500` |

## 5. Collections and visitor tokens (merchant)

Switch to the merchant key: every `/merchant/v1` request sends `irisphera-api-key: $MERCHANT_API_KEY`.

### Create a collection

```bash
curl -i -X POST "$IRISPHERA_BASE_URL/merchant/v1/collection" \
  -H "Accept: application/json" -H "Content-Type: application/json" \
  -H "irisphera-api-key: $MERCHANT_API_KEY" \
  -d '{"title":"Autumn 2026","productFeed":"'"$PRODUCT_FEED_URL"'"}'
```

`201` — `{"id": "<uuid>", "merchantId": "<uuid>", "title": "Autumn 2026", "productFeed": "https://..."}`. `title` is required; `productFeed` is optional but must be set before URL import. Store `id` as `$COLLECTION_ID` and keep a secure mapping between your brand record, `$MERCHANT_ID`, and `$COLLECTION_ID`.

### Other collection operations

- `GET /merchant/v1/collection` — lists all collections accessible to the merchant. `200` — `{"collections": [...]}`; errors `400`, `401`, `500`.
- `PUT /merchant/v1/collection/{collectionId}` — same body as create; `200`; errors `400`, `401`, `403`, `404`, `500`. **Reference-implementation limitation:** the current implementation creates a second collection with a new ID instead of updating — verify the response.
- `DELETE /merchant/v1/collection/{collectionId}` — `204`, destructive (deletes the collection and its fashion items); errors `400`, `401`, `500`, `501`. **The reference implementation returns `501 Not Implemented`.**

### Issue visitor tokens

`GET /merchant/v1/access-token?shopperId=<id>` — call from your backend only; never expose the merchant key to browsers:

```bash
curl -i -X GET "$IRISPHERA_BASE_URL/merchant/v1/access-token?shopperId=USER_123" \
  -H "Accept: application/json" -H "irisphera-api-key: $MERCHANT_API_KEY"
```

`200` — `{"accessToken": "<JWT>"}`; errors `401`, `402` (quota exceeded), `500`. Pass only this JWT to shopper-facing code, which sends it as `Authorization: Bearer <token>` on `/shopper/v1` routes.

## 6. Prepare product data

### Canonical `skuCustomId`

Use one stable `skuCustomId` per model-and-color combination: all sizes of one color share it, different colors use different IDs. Map size-level source SKUs to the color-level value before ingestion, and use the same ID across catalog, recommendations, VTO, 3D preview, and analytics events.

### Required and operational fields

| Field | Required | Notes |
| --- | --- | --- |
| `skuCustomId` | Yes | Stable model-and-color identifier |
| `title` | Yes | Nonblank product title |
| `description` | Yes | Plain text |
| `productImages` | Yes | Nonempty array; include front and back views here too |
| `productFeaturedImage` | No | Required for collection processing |
| `productPageUrl` | No | Required for collection processing |
| `gender` | No | `MEN`, `WOMEN`, or `UNISEX` |
| `productFrontImage` | No | Required when IGG or VTO is needed |
| `productBackImage` | No | Required when IGG or VTO is needed |

Images may be URLs or data URIs (`data:image/webp;base64,<base64>`); accepted bytes are PNG, JPEG, WebP, and AVIF. Prefer reachable HTTPS URLs because processing is asynchronous. No product-count, SKU-length, image-count, or per-image size limit is declared. Canonical feeds omit price fields: public price persistence isn't guaranteed.

### Single-product JSON (camelCase)

```json
{
  "skuCustomId": "STYLE-100-BLACK",
  "title": "Tailored Wool Blazer, Black",
  "description": "Single-breasted wool blazer with a fitted waist.",
  "gender": "WOMEN",
  "productFrontImage": "https://cdn.example.com/style-100-black-front.webp",
  "productBackImage": "https://cdn.example.com/style-100-black-back.webp",
  "productImages": [
    "https://cdn.example.com/style-100-black-front.webp",
    "https://cdn.example.com/style-100-black-back.webp",
    "https://cdn.example.com/style-100-black-detail.webp"
  ],
  "productFeaturedImage": "https://cdn.example.com/style-100-black-featured.webp",
  "productPageUrl": "https://shop.example.com/products/style-100-black"
}
```

### Bulk JSON (current runtime)

Bulk feeds require an outer `products` object; image and page fields use snake_case:

```json
{
  "products": [
    {
      "skuCustomId": "STYLE-100-BLACK",
      "title": "Tailored Wool Blazer, Black",
      "description": "Single-breasted wool blazer with a fitted waist.",
      "gender": "WOMEN",
      "product_front_image": "https://cdn.example.com/style-100-black-front.webp",
      "product_back_image": "https://cdn.example.com/style-100-black-back.webp",
      "product_images": [
        "https://cdn.example.com/style-100-black-front.webp",
        "https://cdn.example.com/style-100-black-back.webp"
      ],
      "product_featured_image": "https://cdn.example.com/style-100-black-featured.webp",
      "product_page_url": "https://shop.example.com/products/style-100-black"
    }
  ]
}
```

Save it as `$PRODUCT_JSON_FILE` for upload, or publish it at the collection's `productFeed` URL.

### CSV (current runtime)

Use camelCase headers; separate multiple `productImages` values with `|`:

```csv
skuCustomId,title,description,gender,productFrontImage,productBackImage,productImages,productFeaturedImage,productPageUrl
STYLE-100-BLACK,"Tailored Wool Blazer, Black","Single-breasted wool blazer with a fitted waist.",WOMEN,https://cdn.example.com/style-100-black-front.webp,https://cdn.example.com/style-100-black-back.webp,https://cdn.example.com/style-100-black-front.webp|https://cdn.example.com/style-100-black-back.webp|https://cdn.example.com/style-100-black-detail.webp,https://cdn.example.com/style-100-black-featured.webp,https://shop.example.com/products/style-100-black
```

Header matching is case-insensitive. `skuCustomId` and `title` headers are required, plus at least one of `productFrontImage` or `productFeaturedImage`. Each row needs nonblank `skuCustomId`, `title`, and one nonblank front-or-featured image value; invalid rows may be skipped while later rows continue.

## 7. Queue ingestion and verify

Choose one route per submission. Every success below is `202 Accepted`: queued for asynchronous processing, not completed. No response body, batch ID, batch-status endpoint, webhook, or callback is declared.

### Queue one product

`POST /merchant/v1/collection/{collectionId}/products` with the [single-product JSON](#single-product-json-camelcase) body:

```bash
curl -i -X POST "$IRISPHERA_BASE_URL/merchant/v1/collection/$COLLECTION_ID/products" \
  -H "Accept: application/json" -H "Content-Type: application/json" \
  -H "irisphera-api-key: $MERCHANT_API_KEY" \
  -d '{"skuCustomId":"'"$PRODUCT_SKU"'","title":"Tailored Wool Blazer, Black","description":"Single-breasted wool blazer with a fitted waist.","gender":"WOMEN","productFrontImage":"https://cdn.example.com/style-100-black-front.webp","productBackImage":"https://cdn.example.com/style-100-black-back.webp","productImages":["https://cdn.example.com/style-100-black-front.webp","https://cdn.example.com/style-100-black-back.webp"],"productFeaturedImage":"https://cdn.example.com/style-100-black-featured.webp","productPageUrl":"https://shop.example.com/products/style-100-black"}'
```

`202`; errors `400`, `401`, `402`, `403`, `404`, `500`.

### Queue a JSON or CSV file

`POST /merchant/v1/collection/{collectionId}/file` — multipart `file` field; `useSeasonFiltering` is a query parameter (optional, default `false`):

```bash
curl -i -X POST "$IRISPHERA_BASE_URL/merchant/v1/collection/$COLLECTION_ID/file?useSeasonFiltering=false" \
  -H "Accept: text/plain" -H "irisphera-api-key: $MERCHANT_API_KEY" \
  -F "file=@$PRODUCT_JSON_FILE;type=application/json"
```

For CSV, upload `$PRODUCT_CSV_FILE` with `type=text/csv`. `202` (may return plain text); errors `400`, `401`, `402`, `403` (another merchant's collection), `404` (missing collection), `500`.

### Queue the collection feed URL

`POST /merchant/v1/collection/{collectionId}/import-from-url` — the server fetches the collection's `productFeed`; the body is optional:

```bash
curl -i -X POST "$IRISPHERA_BASE_URL/merchant/v1/collection/$COLLECTION_ID/import-from-url" \
  -H "Accept: application/json" -H "Content-Type: application/json" \
  -H "irisphera-api-key: $MERCHANT_API_KEY" -d '{"useSeasonFiltering":false}'
```

`202`; errors `400`, `401`, `402`, `403`, `404`, `415` (unsupported feed content type), `500`. The feed must be reachable over HTTP or HTTPS (HTTPS preferred) and return a supported JSON or CSV media type.

> **Security warning (current runtime, not a supported contract):** the reference implementation forwards incoming request headers to the feed host, excluding only host, connection, `content-*`, `accept*`, `user-agent*`, `origin*`, and `referer*` headers. `irisphera-api-key` and `Authorization` may reach the feed host. Don't use URL import with an external or untrusted feed host; prefer single-product or file ingestion.

### Verify through `fashionItems`

`GET /merchant/v1/collection/{collectionId}/products`:

```bash
curl -i -X GET "$IRISPHERA_BASE_URL/merchant/v1/collection/$COLLECTION_ID/products" \
  -H "Accept: application/json" -H "irisphera-api-key: $MERCHANT_API_KEY"
```

`200` — products visible to the merchant in this collection:

```json
{
  "fashionItems": [
    {
      "merchantId": "0198f2a0-7b42-7000-8000-000000000001",
      "collectionId": "0198f2a0-7b42-7000-8000-000000000101",
      "skuCustomId": "STYLE-100-BLACK",
      "data": {
        "title": "Tailored Wool Blazer, Black",
        "gender": "WOMEN",
        "placement": "UPPER",
        "category": "blazer",
        "descriptors": ["tailored", "wool"],
        "images": [
          {
            "imageView": "FEATURED",
            "file": { "path": "s3://catalog-assets/style-100-black/featured.webp", "presignedUrl": "https://cdn.example.com/style-100-black-featured.webp" }
          }
        ],
        "retailerDescription": "Single-breasted wool blazer with a fitted waist.",
        "productPageUrl": "https://shop.example.com/products/style-100-black"
      }
    }
  ]
}
```

Find your `skuCustomId` to confirm visibility. An empty array is a valid read; it doesn't prove queued work finished. No polling frequency or completion deadline is declared — choose retry timing in your client policy.

**Duplicate behavior (current runtime):** within one feed, the first occurrence of an SKU wins and later occurrences are filtered; an SKU already present in the collection is skipped. This is configuration-dependent, not an unconditional upsert guarantee.

### Declared onboarding statuses

| Operation | Success | Other declared statuses |
| --- | --- | --- |
| Create collection | `201` | `400`, `401`, `500` |
| Queue one product | `202` | `400`, `401`, `402`, `403`, `404`, `500` |
| Upload file | `202` | `400`, `401`, `402`, `403`, `404`, `500` |
| Import from URL | `202` | `400`, `401`, `402`, `403`, `404`, `415`, `500` |
| Read collection products | `200` | `400`, `401`, `500` |

`402 Payment Required` means the current plan's quota is exceeded; `GET /merchant/v1/subscription` (section 8) explains which feature is exhausted.

## 8. Merchant operations

- `GET /merchant/v1/products` — lists products across all collections accessible to the merchant. `200` — `{"fashionItems": [...]}`; errors `400`, `401`, `500`. Image records contain storage `path`s, but this operation does not populate presigned URLs.
- `GET /merchant/v1/subscription` — `200` — `{"featureDetection": {...}, "shopperRecommendation": {...}, "shopper2dPreview": {...}}`, each a `MeteredQuota` `{"limit": N, "current": N}`; errors `400`, `401`, `500`. Use it to explain quota failures; it isn't a substitute for handling `402`.
- `POST /merchant/v1/report` — body `{"startTime": "<ISO 8601>", "endTime": "<ISO 8601>", "zone": "Europe/Bucharest"}` for the half-open period `[startTime, endTime)`; `200` — aggregate report; errors `400`, `401`, `500`.
- `POST /merchant/v1/vto2d` — merchant-key demo VTO for testing and integration validation, not the shopper production flow. Multipart `featuredImage` and `userPhoto` (both required); `description` is a query parameter:

```bash
curl -i -X POST "$IRISPHERA_BASE_URL/merchant/v1/vto2d?description=linen%20shirt" \
  -H "Accept: application/json" -H "irisphera-api-key: $MERCHANT_API_KEY" \
  -F "featuredImage=@product.webp" -F "userPhoto=@shopper.webp"
```

`200` — `{"generatedImage": "<base64 WebP>"}`; errors `400`, `401`, `420` (user image unusable), `422` (image processing failed), `500`.

- `POST /merchant/v1/products/style-occasion-analysis` — declared `200` with style and occasion filters; **the reference implementation returns `501 Not Implemented`**.

## 9. Shopper session and identity

Shopper APIs authenticate with `Authorization: Bearer <visitor token>`. Your backend issues tokens; shopper-facing code never sees either API key.

**Identity model.** Use one stable `shopperId` per visitor: before login, a persistent anonymous identifier (for example a first-party cookie); after login, the stable customer identifier from your CRM, commerce, or identity system. Issue tokens with the current `shopperId` via `GET /merchant/v1/access-token?shopperId=<id>`.

**Identity transition.** When an anonymous shopper logs in and activity was recorded under the anonymous ID, call `POST /shopper/v1/data/identify` once with `{"anonymousShopperId": "...", "customerId": "..."}` — `204`; errors `400`, `401`, `409` (no recorded activity or already associated), `422`, `500`. Use the `customerId` as the `shopperId` for all future tokens. Don't call it on every session or during token refresh.

**Session bootstrap.** `GET /shopper/v1/auth/access-token` validates the token and returns `TokenInfo`: `shopperId`, `expiresInSeconds`, `merchantName`, `enabledFeatures` (`SHOPPER_RECOMMENDATIONS`, `STYLIST_PREVIEW`), `sizingConfig`, `themeConfig`, `flowConfig` — `200`; errors `401`, `404` (the merchant no longer exists). A token can be valid while an individual feature is unavailable.

## 10. Shopper experience APIs

### Recommendations

`POST /shopper/v1/recommendations`:

```bash
curl -i -X POST "$IRISPHERA_BASE_URL/shopper/v1/recommendations" \
  -H "Authorization: Bearer <ACCESS_TOKEN>" -H "Content-Type: application/json" \
  -d '{"encodedProfileData":"<BASE64_JSON>","offset":0,"limit":12,"filters":[],"collectionIds":["0198f2a0-7b42-7000-8000-000000000101"]}'
```

`encodedProfileData` (required) is Base64-encoded JSON from the shopper profile flow — encoding, not encryption, so treat it as sensitive. `offset` (≥ 0), `limit` (≥ 1), `collectionIds` (UUIDs; empty means all collections), and `filters` (strings) narrow the results. `200` — `{"silhouette": ..., "palette": ..., "generalSizing": ..., "merchantCollections": [...], "recommendationsByCollection": [{"collection": ..., "recommendations": [{"skuCustomId": "...", "size": "...", "images": [...], "productPageUrl": "..."}]}]}`; errors `400`, `401`, `422`, `500`. Match each `skuCustomId` to your storefront catalog for price, image, and availability.

**Current runtime:** the reference implementation applies `filters` but ignores `offset`, `limit`, and `collectionIds` — don't rely on those three controls until the implementation is corrected.

### Virtual try-on

- **Readiness:** `GET /shopper/v1/stylist-preview/{skuCustomId}` — `200` — a placement category: `UNAVAILABLE`, `UPPER`, `LOWER`, `FULL`, `HEAD`, `FEET`, `ACCESSORY`, or `JEWELRY`. `UNAVAILABLE` means a preview must not be requested. Errors `400`, `401`, `404`, `422`, `500`.
- **Generate:** `POST /shopper/v1/stylist-preview` — query parameters `skuCustomId` (required) and `isFaceBlurred` (optional; tells the service whether the client already blurred the face), with multipart `targetImage` (required):

```bash
curl -i -X POST "$IRISPHERA_BASE_URL/shopper/v1/stylist-preview?skuCustomId=STYLE-100-BLACK" \
  -H "Authorization: Bearer <ACCESS_TOKEN>" \
  -F "targetImage=@shopper.webp"
```

`200` — `{"generatedImage": "<base64 WebP>"}`; errors `400`, `401`, `404` (SKU missing or inaccessible), `420` (shopper imagery unusable), `422`, `500`. On `420` or `422`, ask for another shopper photo or fall back to the product page.

- `GET /shopper/v1/stylist-preview` (list of VTO-ready items) is declared but **the reference implementation responds with an error** — use per-SKU readiness instead.

### 3D preview

`GET /shopper/v1/td-preview/{skuCustomId}` — `200` — a temporary download URL for the SKU's 3D asset (short-lived; don't persist it); errors `400`, `401`, `404`, `422` (no usable asset), `500`.

### Body measurements

`POST /shopper/v1/body-measurements`:

```bash
curl -i -X POST "$IRISPHERA_BASE_URL/shopper/v1/body-measurements" \
  -H "Authorization: Bearer <ACCESS_TOKEN>" -H "Content-Type: application/json" \
  -d '{"user_gender":"WOMEN","user_height":170,"img_data_main":"<raw base64>","img_data_side":"<raw base64>"}'
```

`user_gender` (required), `user_height` in centimeters (required), `img_data_main` (required; raw base64 without a data-URI prefix), and `img_data_side` (optional). `200` — `{"measurement_bust": ..., "measurement_waist": ..., "measurement_hips": ...}` in centimeters; errors `400`, `401`, `422`, `500`.

### Color extraction

`POST /shopper/v1/color-extraction` — body `{"img_data_main": "<raw base64 selfie>"}` (required). `200` — `{"skin_color_hex": "...", "eyes_color_hex": "...", "hair_color_hex": "..."}`; errors `400`, `401`, `422`, `500`.

Obtain the shopper's consent before transmitting photos, and avoid retaining them.

## 11. Shopper analytics events

Report every completed order from your backend, even when the shopper didn't use Irisphera features. All events are `POST` requests with the visitor token; `204` is a best-effort acknowledgement (retrying records duplicate events); errors `400`, `401`, `422`, `500`. The merchant and user come from the token. Each array entry is one unit — repeat a SKU to express quantity. These endpoints record analytics; they don't create or mutate orders.

| Event | Endpoint | Body |
| --- | --- | --- |
| Order created | `POST /shopper/v1/data/order-create` | `{"skuCustomIds": [{"skuCustomId": "STYLE-100-BLACK", "price": "99.00 EUR"}]}` |
| Order cancelled | `POST /shopper/v1/data/order-cancelled` | same |
| Return | `POST /shopper/v1/data/return` | same |
| Product page viewed | `POST /shopper/v1/data/product-page` | `{"skuCustomId": "STYLE-100-BLACK"}` |
| 3D preview opened | `POST /shopper/v1/data/td-preview` | `{"skuCustomId": "STYLE-100-BLACK"}` — only after the preview is actually shown |

`price` is optional but should be supplied for revenue and average-order-value reporting. Accepted inputs: an unsigned, ungrouped decimal amount with at most two fractional digits (a period or comma may be the decimal separator) and either a three-letter currency code before or after the amount separated by whitespace, or `$`, `€`, or `£` immediately before it (`$` maps to USD, `€` to EUR, `£` to GBP; codes are canonicalized to uppercase). The canonical stored form is `12.34 USD`. Malformed values are stored raw and excluded from reporting.

## 12. Runtime notes and limitations

These describe the current implementation; they aren't additions to the public contract:

- Ingestion converts JSON and CSV only. The spec's file description mentions XLSX, but the runtime has no XLSX converter — don't submit or recommend XLSX.
- The current multipart file and request caps are 100 MB (runtime configuration, not a declared limit).
- A file route's `202` doesn't prove that parsing or asynchronous processing succeeded — verify through `fashionItems`.
- Duplicate SKUs: the first occurrence wins within a feed, and an SKU already present in a collection is skipped (configuration-dependent, not upsert).
- `PUT /merchant/v1/collection/{collectionId}` creates a second collection with a new ID instead of updating (reference implementation).
- These operations are declared but the reference implementation returns `501 Not Implemented`: `DELETE /merchant/v1/collection/{collectionId}`, `DELETE /merchant/v1/collection/{collectionId}/file`, `DELETE /merchant/v1/collection/{collectionId}/products/{skuCustomId}`, `POST /merchant/v1/products/style-occasion-analysis`, and `GET /shopper/v1/stylist-preview` (list).
- URL import forwards most request headers to the feed host, including `irisphera-api-key` and `Authorization` (see section 7) — don't use it with external or untrusted feed hosts.
- Recommendations: the reference implementation applies `filters` but ignores `offset`, `limit`, and `collectionIds` (see section 10).
- No upsert, price persistence, callbacks, batch tracking, or completion deadline is guaranteed.

## 13. Troubleshooting

| Symptom | Check |
| --- | --- |
| `401` on integrator route | Send the integrator key, not a merchant key. |
| `401` on merchant route | Send the caller-created merchant key. |
| `402` on merchant routes | Quota exceeded — see `GET /merchant/v1/subscription`. |
| `403` on merchant routes | The collection belongs to another merchant. |
| `404` after merchant delete | Deletion is permanent; repeated requests return `404`. |
| `409` on `identify` | The anonymous shopper has no recorded activity, or is already associated. |
| `415` on URL import | The feed must be JSON or CSV with a matching media type. |
| `420` or `422` on imagery routes | Input photos are unusable or processing failed. |
| `501` on collection or product routes | Declared but not implemented by the reference implementation. |
| Product missing after `202` | Read `fashionItems`; queue acceptance isn't completion. |
| Empty `fashionItems` | Check product shape, required text, image reachability, and collection ID. |

## 14. Launch checklist

- Use API version 2.0.0 at `https://api.irisphera.com`; generate clients from the spec at `/v3/api-docs`.
- Send the integrator key only on `/integrator/v1`, the merchant key only on `/merchant/v1`, and bearer tokens only on `/shopper/v1`; never combine key and bearer authentication.
- Create each merchant key before onboarding and store it in a secret manager.
- Retry merchant creation freely: it reconciles by `apiKey` and `name`.
- On merchant `PUT`, always include `name` and `apiKey`; omit configuration fields you want preserved.
- Store `merchantId` and `collectionId` with your brand record.
- Use stable model-and-color `skuCustomId` values across catalog and shopper flows; use `MEN`, `WOMEN`, or `UNISEX` when gender is present.
- Use only JSON or CSV for ingestion; prefer single-product or file ingestion for external or untrusted feed hosts.
- Treat every ingestion `202` as queued and verify the SKU through `fashionItems`.
- Issue tokens with the visitor's current `shopperId`; call `identify` once at login; expose only visitor tokens to the frontend.
- Report orders, cancellations, and returns from the backend with canonical SKUs, repeating a SKU per unit.
- Redact API keys and bearer tokens from every observability and support channel; log `requestId`.
- Don't assume upsert, price persistence, callbacks, batch tracking, or a completion deadline.

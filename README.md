# Irisphera Integration Guide for Third-Party Providers

This guide helps a third-party provider set up Irisphera for a merchant storefront.

Start with merchant setup, product data, and visitor setup. Then choose one shopper option based on who owns the storefront UI.

## Guide Map

1. [Merchant Setup and Common Requirements](#1-merchant-setup-and-common-requirements)
2. [Visitor Setup for Shopper Flows](#2-visitor-setup-for-shopper-flows)
3. [Option A: Build Your Own Shopper UI with Direct API](#3-option-a-build-your-own-shopper-ui-with-direct-api)
4. [Option B: Embed Irisphera UI with SDK/Iframe](#4-option-b-embed-irisphera-ui-with-sdkiframe)
5. [Troubleshooting and Reference](#5-troubleshooting-and-reference)

## 1. Merchant Setup and Common Requirements

Complete this setup before you build shopper-facing features.

The API key, merchant VTO demo, catalog feed, and canonical product IDs are common to both shopper options.

### API Key Setup

Irisphera issues your organization an API key.

Store the API key server-side only. Never expose it in browser, iframe, SDK, shopper, or public frontend code.

Use the API key only from trusted server code, such as your backend token exchange, merchant demo calls, and protected feed endpoint.

### Merchant VTO Demo and Validation

Use `POST /merchant/v1/vto2d` when your backend needs a merchant demo, smoke test, or quick validation of try-on generation.

This endpoint uses API-key auth only. It does not require `endUserId` or a visitor `accessToken`.

It is for merchant demo and validation, not the shopper production VTO flow.

Send multipart form data with required `featuredImage` and `userPhoto` files. Add optional `description` text when it helps identify the garment.

The response includes `generatedImage`, the generated try-on image.

```bash
curl -X POST \
  "https://api.irisphera.com/merchant/v1/vto2d" \
  -H "irisphera-api-key: <YOUR_API_KEY>" \
  -F "featuredImage=@product.webp" \
  -F "userPhoto=@shopper.webp" \
  -F "description=linen shirt"
```

### Catalog Ingestion

Catalog ingestion must finish before shopper features rely on products.

Expose an HTTPS feed endpoint that Irisphera can fetch on a schedule. Protect that endpoint with API key authentication.

Irisphera calls your feed endpoint with `irisphera-api-key: <YOUR_API_KEY>`. Your server validates that header before returning the feed.

Example feed request from Irisphera to your server:

```http
GET https://partner.example.com/api/irisphera/feed
irisphera-api-key: <YOUR_API_KEY>
```

Return a JSON object with a `products` array. Unknown product fields are not part of the contract.

| Field | Required | Notes |
| --- | --- | --- |
| `skuCustomId` | Yes | Canonical product-color ID. |
| `title` | Yes | Product title. Add color or variant text when needed. |
| `description` | No | Plain product description. |
| `gender` | No | `women`, `men`, or `unisex`. |
| `product_featured_image` | Conditional | Primary display image. Maps to `featuredImage`. |
| `product_front_image` | Conditional | Front-facing product image. |
| `product_back_image` | No | Back-facing product image. |
| `product_images` | Conditional | Extra product image URLs. |
| `product_page_url` | No | Product page URL. Include variant params when useful. |
| `date_added` | No | ISO 8601 date for seasonality. |
| `product_price` | No | Regular price. |
| `product_special_price` | No | Sale price. |

At least one image is required. Irisphera resolves images in this order: `product_featured_image`, then `product_front_image`, then the first value in `product_images`.

Products without an image are skipped during import.

```json
{
  "products": [
    {
      "skuCustomId": "12345678901",
      "title": "Classic Sweater Red",
      "description": "Merino wool sweater with ribbed cuffs",
      "gender": "women",
      "product_front_image": "https://cdn.example.com/sweater-red-front.jpg",
      "product_back_image": "https://cdn.example.com/sweater-red-back.jpg",
      "product_images": [
        "https://cdn.example.com/sweater-red-front.jpg",
        "https://cdn.example.com/sweater-red-detail.jpg"
      ],
      "product_featured_image": "https://cdn.example.com/sweater-red-front.jpg",
      "product_page_url": "https://shop.example.com/products/classic-sweater?variant=12345678901",
      "date_added": "2026-02-01T10:24:00Z"
    }
  ]
}
```

### skuCustomId Rules

`skuCustomId` represents one product-color variation.

All sizes of the same color share the same `skuCustomId`. Different colors must use different `skuCustomId` values.

If your catalog has size-level SKUs, map each size variant to one canonical color-level `skuCustomId`.

Use this same value in the product feed, Direct API calls, SDK helpers, purchase reporting, event reporting, recommendations, VTO, and 3D preview.

```js
function getSkuCustomId(lineItem, variantToCanonicalMap) {
  if (variantToCanonicalMap && lineItem.variantId) {
    const canonical = variantToCanonicalMap[lineItem.variantId];
    if (canonical) return canonical;
  }

  return lineItem.productId || lineItem.sku || lineItem.variantId;
}
```

## 2. Visitor Setup for Shopper Flows

Use this section when you are ready to make shopper API calls or load the SDK/iframe.

Shopper requests use visitor bearer tokens. They do not use your merchant API key.

### Stable endUserId

Use a stable `endUserId` for each visitor across browsing, cart, purchase, and event reporting.

The same visitor must receive the same `endUserId` when they browse, add to cart, return later, and complete checkout.

For logged-in shoppers, use your internal user ID or a stable hash of it.

For anonymous shoppers, create a stable first-party identifier and store it in a first-party cookie.

For guest checkout, reconstruct the same identifier from the checkout context where possible.

```js
const crypto = require('crypto');

function generateEndUserId(clientIp, userAgent) {
  return crypto
    .createHash('md5')
    .update(`${clientIp || ''}${userAgent || ''}`)
    .digest('hex');
}
```

### Visitor Access Tokens

Your backend exchanges the visitor `endUserId` for a visitor `accessToken` through `/merchant/v1/access-token`.

```bash
curl -X GET \
  "https://api.irisphera.com/merchant/v1/access-token?endUserId=USER_123" \
  -H "irisphera-api-key: <YOUR_API_KEY>"
```

Example response:

```json
{
  "accessToken": "<JWT>"
}
```

Pass only the visitor `accessToken` to frontend code.

Browser, iframe, SDK, and shopper API calls use `Authorization: Bearer <ACCESS_TOKEN>`.

### Auth Rules for Shopper Requests

Never send both `irisphera-api-key` and `Authorization: Bearer ...` on the same Irisphera API request.

Irisphera API rejects that combination.

Use the API key only on your server for merchant endpoints and token exchange.

Use the visitor bearer token for shopper endpoints, SDK calls, iframe calls, and browser code.

### Purchase and Event Reporting

Send purchase data from your backend for every completed order, even if the shopper did not use Irisphera features.

This reporting supports attribution, model training, and ROI measurement.

For each completed order, reconstruct the same `endUserId`, get a visitor access token, and send unit-level purchased SKUs.

```bash
curl -X POST \
  "https://api.irisphera.com/shopper/v1/data/purchase" \
  -H "Authorization: Bearer <ACCESS_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "skuCustomIds": [
      { "skuCustomId": "sweater-red", "price": "99.00 EUR" },
      { "skuCustomId": "jeans-blue", "price": "149.00 EUR" }
    ]
  }'
```

If quantity is greater than one, include one entry per purchased unit.

Send cancellation data when an order is canceled.

```bash
curl -X DELETE \
  "https://api.irisphera.com/shopper/v1/data/purchase" \
  -H "Authorization: Bearer <ACCESS_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "skuCustomIds": [
      { "skuCustomId": "sweater-red" }
    ]
  }'
```

Client-side event helpers are supplemental. They do not replace server-side purchase reporting.

## 3. Option A: Build Your Own Shopper UI with Direct API

Choose this option when your team owns the storefront UI, profile capture, recommendation rendering, and VTO rendering.

Your backend stores the API key and creates visitor access tokens. Your product UI calls shopper APIs with bearer auth.

### Direct API: Virtual Try On

For the shopper flow, first check whether the product is ready for VTO.

Use `GET /shopper/v1/stylist-preview/{skuCustomId}` with the visitor token.

```bash
curl -X GET \
  "https://api.irisphera.com/shopper/v1/stylist-preview/{skuCustomId}" \
  -H "Authorization: Bearer <ACCESS_TOKEN>"
```

If the product is unavailable, hide or disable the VTO action and keep the normal product page flow.

Generate the shopper result with `POST /shopper/v1/stylist-preview` as multipart form data.

Send `skuCustomId` and the shopper photo as `targetImage`.

```bash
curl -X POST \
  "https://api.irisphera.com/shopper/v1/stylist-preview" \
  -H "Authorization: Bearer <ACCESS_TOKEN>" \
  -F "skuCustomId=12345678901" \
  -F "targetImage=@shopper.webp"
```

Render the returned image in your own UI. If imagery is invalid, ask for a different shopper photo or fall back to your normal product page.

### Direct API: Stylistic Recommendations

Choose this option when the retailer owns profile capture, recommendation calls, and recommendation rendering.

Call `POST /shopper/v1/recommendations` with a visitor access token.

Send the shopper profile as `encodedProfileData`. Use `filters` and `collectionIds` when you want a narrowed result set.

```bash
curl -X POST \
  "https://api.irisphera.com/shopper/v1/recommendations" \
  -H "Authorization: Bearer <ACCESS_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "encodedProfileData": "<ENCODED_PROFILE_DATA>",
    "offset": 0,
    "limit": 12,
    "filters": {},
    "collectionIds": ["summer-2026"]
  }'
```

Returned recommendations can include product IDs, recommended size data, silhouette, palette, and general sizing data.

Render each recommendation by matching its `skuCustomId` to your catalog product, image, price, product URL, and available sizes.

## 4. Option B: Embed Irisphera UI with SDK/Iframe

Choose this option when you want Irisphera's embedded frontend experience inside your site.

Your backend still creates visitor access tokens. The browser passes only those tokens to the SDK.

### SDK/Iframe Setup

Load the production SDK from `https://app.irisphera.com/v2.0.0/irispheraSdk.js`.

```html
<script src="https://app.irisphera.com/v2.0.0/irispheraSdk.js"></script>
```

The SDK global is `window.irsSdkV2`. The default iframe origin is `https://app.irisphera.com`.

Set `window.IRISPHERA_TARGET_ORIGIN` before loading the script only when Irisphera gives you a different iframe app origin.

```html
<script>
  window.IRISPHERA_TARGET_ORIGIN = 'https://app.irisphera.com';
</script>
<script src="https://app.irisphera.com/v2.0.0/irispheraSdk.js"></script>
```

Initialize the SDK after your backend returns a visitor token.

```js
async function initIrispheraSdk() {
  const sdk = await window.irsSdkV2.whenReady();
  const token = await getAccessTokenFromYourBackend();

  await sdk.setAccessToken(token, { skipValidation: true });

  sdk.registerTokenFetcher(async () => {
    return getAccessTokenFromYourBackend();
  });

  await sdk.whenTokenReady();
}
```

Do not pass `irisphera-api-key` to SDK, iframe, browser code, or shopper requests.

### SDK/Iframe: Virtual Try On

On product pages, render the sizing widget first. It owns callbacks that can open VTO or 3D preview.

```js
window.irsSdkV2.showIrispheraIframeSizing(
  document,
  'irsSizingDiv',
  null,
  recommendedSize,
  skuCustomId,
  onEditProfile,
  onViewRecommendations,
  onVto,
  onTdPreview
);
```

When the sizing widget calls `onVto`, open the VTO iframe with the canonical `skuCustomId`.

```js
function onVto() {
  openModal();
  window.irsSdkV2.showIrispheraIframeVto(document, 'irsIframeDiv', null, skuCustomId);
}
```

When the sizing widget calls `onTdPreview`, treat 3D preview as supporting UI. Open it only when a `glbUrl` is present.

```js
function onTdPreview(glbUrl) {
  if (!glbUrl) return;
  openModal();
  window.irsSdkV2.showIrispheraIframeTdPreview(document, 'irsIframeDiv', null, skuCustomId, glbUrl);
}
```

### SDK/Iframe: Stylistic Recommendations

Choose this option when Irisphera owns profile capture and the embedded recommendation flow.

Check whether the SDK already has a saved profile. Then show the right iframe entry point for that visitor.

Use `showIrispheraIframeCta` when the visitor needs to create a profile or open recommendations from a CTA.

Use `showIrispheraIframeBanner` when the visitor already has a profile and should see an edit or recommendation banner.

Open `showIrispheraIframe` in a modal for profile capture or profile edits.

When detection finishes, the SDK saves the profile, calls `fetchRecommendations`, persists state, and emits profile data readiness.

```js
if (window.irsSdkV2.isUserProfileSavedToCookies()) {
  window.irsSdkV2.showIrispheraIframeBanner(
    document,
    'irsBannerDiv',
    null,
    () => {
      openModal();
      window.irsSdkV2.showIrispheraIframe(document, 'irsIframeDiv', () => {
        closeModal();
        renderRecommendations();
      });
    }
  );
} else {
  window.irsSdkV2.showIrispheraIframeCta(
    document,
    'irsCtaDiv',
    null,
    () => {
      openModal();
      window.irsSdkV2.showIrispheraIframe(document, 'irsIframeDiv', () => {
        closeModal();
        renderRecommendations();
      });
    },
    renderRecommendations
  );
}
```

Call `fetchRecommendations({ filters, collectionIds })` when you need recommendations for the saved profile and current catalog filters.

The SDK sends saved profile data as `encodedProfileData` and stores the response for display.

It reuses an in-flight request with the same filter and collection inputs instead of starting another request.

The recommendation cache is bound to the saved profile and request filters. If the profile changes, the SDK invalidates cached recommendations.

Read outputs from `Results.sessionRecommendations`, `Results.sessionSilhouette`, and `Results.sessionPalette`.

```js
await window.irsSdkV2.fetchRecommendations({
  filters: selectedFilters,
  collectionIds: selectedCollectionIds,
});

const recommendations = window.irsSdkV2.Results.sessionRecommendations;
const silhouette = window.irsSdkV2.Results.sessionSilhouette;
const palette = window.irsSdkV2.Results.sessionPalette;
```

Render recommendations in your storefront using `skuCustomId`.

```js
function renderRecommendations() {
  const recommendations = window.irsSdkV2.Results.sessionRecommendations || [];

  return recommendations.map((recommendation) => {
    const product = catalogBySkuCustomId[recommendation.skuCustomId];

    return {
      product,
      recommendedSize: recommendation.size || null,
    };
  });
}
```

Use host event helpers only as supplemental client analytics for browsing and cart signals.

```js
window.irsSdkV2.HostEventsHandler.onProductView(skuCustomId);
window.irsSdkV2.HostEventsHandler.onAddToCart(skuCustomId);
window.irsSdkV2.HostEventsHandler.onRemoveFromCart(skuCustomId);
window.irsSdkV2.HostEventsHandler.onPurchase([skuCustomId1, skuCustomId2]);
```

`HostEventsHandler.onPurchase(...)` sends only SKU IDs from the browser. It does not replace mandatory server-side purchase reporting with unit-level purchase data.

## 5. Troubleshooting and Reference

Use this section after you complete setup and choose a shopper option.

### Iframes Not Displaying

| Scenario | Likely cause | What to check |
| --- | --- | --- |
| No iframe content | Token was never set | Confirm `setAccessToken()` ran after your backend returned a token. |
| Token rejected | Invalid or expired token | Use `registerTokenFetcher` so the SDK can request a fresh token. |
| Blank iframe | Origin mismatch | Confirm the iframe origin is `https://app.irisphera.com` or the Irisphera-provided override. |
| VTO hidden | Product not ready | Check `GET /shopper/v1/stylist-preview/{skuCustomId}` before showing VTO. |
| SKU missing | Catalog mismatch | Confirm the feed contains the same canonical `skuCustomId`. |

### Debug Logs

The SDK log level defaults to `none`. Enable debug logs during diagnosis.

```js
await window.irsSdkV2.whenReady();
window.irsSdkV2.logLevel = 'debug';
```

### Message Origin Checks

The SDK validates iframe postMessage events against the configured target origin and expected iframe window source.

Set `window.IRISPHERA_TARGET_ORIGIN` before the SDK script only when Irisphera gives you a different iframe origin.

If messages are rejected, check the browser console and confirm your site loads the expected Irisphera iframe host.

### Optional 3D Preview

3D preview is supporting UI. It needs a `skuCustomId` and a `glbUrl`.

The sizing iframe can pass `glbUrl` through the `onTdPreview` callback.

You can also check 3D availability with the shopper 3D preview endpoint from code that already has a visitor token.

```js
async function get3dPreviewUrl(skuCustomId, accessToken) {
  const res = await fetch(
    `https://api.irisphera.com/shopper/v1/td-preview/${skuCustomId}`,
    { headers: { 'Authorization': `Bearer ${accessToken}` } }
  );

  if (res.status !== 200) return null;

  const raw = await res.text();
  try {
    return JSON.parse(raw);
  } catch {
    return raw;
  }
}
```

### SDK Helper Functions

| Function | Purpose |
| --- | --- |
| `whenReady()` | Resolves when the SDK is initialized. |
| `whenTokenReady()` | Resolves when an access token is available in SDK state. |
| `setAccessToken(token, options)` | Sets the visitor token. Use `{ skipValidation: true }` when your backend already validated it. |
| `registerTokenFetcher(fn)` | Registers an async token refresh function. |
| `isUserProfileSavedToCookies()` | Returns whether the visitor has completed profile capture. |
| `fetchRecommendations()` | Fetches recommendations into `Results.sessionRecommendations`. |
| `closeIrispheraIframe()` | Closes the active iframe. |
| `resetStorage()` | Clears SDK storage and triggers recovery behavior. |
| `_getAccessToken()` | Returns the current access token for support checks. |

For integration support, contact your Irisphera technical representative.

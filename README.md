# Irisphera Integration Guide (3rd-Party Marketplace)

This guide explains how a marketplace partner integrates Irisphera services. It covers the mandatory core setup plus feature-specific integration for Virtual Personal Shopper (recommendations), Virtual Try On, and 360/3D preview.

## 1. Mandatory Core Integration

### 1.1 API Key and Visitor Access Tokens

- Irisphera will provide your organization with an API key.
- Keep the API key server-side only. Do not expose it in the browser.
- For each visitor session, your backend must request a short-lived Irisphera access token and pass it to the browser.

**Create a visitor access token (server-side)**

Request a token with the API key header `irisphera-api-key` and a stable `endUserId` (your visitor/user identifier).

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

Some deployments may include additional fields (for example token TTL or organization metadata), but your integration should treat `accessToken` as the required contract.

### 1.2 endUserId: Why Consistency Matters

The `endUserId` parameter is critical for Irisphera analytics and reporting. It uniquely identifies a visitor across multiple sessions and devices.

**Why this matters:**

1. **Cross-session tracking**: When a visitor returns (even days later), using the same `endUserId` lets Irisphera correlate their profile, recommendations, and purchase history.
2. **Attribution reporting**: Irisphera provides reports showing which recommendations led to purchases. Accurate attribution requires the same `endUserId` to link browsing sessions to checkout events.
3. **Model improvement**: Consistent user identification helps Irisphera's recommendation engine learn from real purchase outcomes.

**Best practices for generating endUserId:**

| Scenario | Recommended approach |
| --- | --- |
| Logged-in user | Use your internal user ID or a stable hash of it (e.g., `md5(user_id + salt)`) |
| Anonymous visitor | Generate a stable fingerprint from `clientIp + userAgent` and store it in a first-party cookie |
| Guest checkout | Hash the combination of `clientIp + userAgent` at order time to match the browsing session |

**Example: anonymous visitor fingerprint**

```js
// Server-side (Node.js example)
const crypto = require('crypto');

function generateEndUserId(clientIp, userAgent) {
  return crypto
    .createHash('md5')
    .update(`${clientIp || ''}${userAgent || ''}`)
    .digest('hex');
}
```

The key rule: **the same visitor must always receive the same `endUserId`**, whether they are browsing, adding to cart, or completing a purchase.

### 1.3 SDK Initialization (recommended pattern)

The Irisphera SDK is exposed as `window.irsSdkV2` and manages iframe messaging, profile persistence, and recommendations caching. Your frontend must set the access token on every visitor session.

**Important (production origin):**

Set `window.IRISPHERA_TARGET_ORIGIN` to the Irisphera iframe origin **before** loading the SDK script. If omitted, the SDK falls back to `localhost:3000` for iframe origin resolution.

```html
<script>
  // Set this to the Irisphera iframe app origin provided to your integration.
  window.IRISPHERA_TARGET_ORIGIN = 'https://<IRISPHERA_IFRAME_ORIGIN>';
</script>
<script src="<IRISPHERA_SDK_URL>"></script>
```

**Frontend init sequence**

```js
async function getAccessTokenFromYourBackend() {
  const res = await fetch('/api/irisphera/token', { credentials: 'include' });
  const data = await res.json();
  return data.accessToken;
}

async function initIrisphera() {
  const sdk = await window.irsSdkV2.whenReady();
  const token = await getAccessTokenFromYourBackend();

  // Server already validated the token.
  await sdk.setAccessToken(token, { skipValidation: true });

  // Optional: enable automatic refresh when iframe requests a new token.
  sdk.registerTokenFetcher(async () => {
    const res = await fetch('/api/irisphera/token', { credentials: 'include' });
    const data = await res.json();
    return data.accessToken;
  });

  // Optional: if multiple initializers can run concurrently,
  // this waits until token is definitely available in SDK state.
  await sdk.whenTokenReady();
}

initIrisphera();
```

**Key rules**

- Always create a new access token per visitor session.
- Use `setAccessToken(..., { skipValidation: true })` when your server already validated the token.
- Register a token fetcher to handle iframe refresh requests when tokens expire.
- The SDK initializes asynchronously; call `whenReady()` before invoking iframe/helper APIs.
- If no token fetcher is registered, token refresh requests from iframe contexts can fail when tokens expire.

### 1.3.1 SDK lifecycle events (optional but recommended)

The SDK dispatches browser events that can be used for observability and race-free orchestration:

- `irsSdkV2:ready` — SDK initialized and ready.
- `irsSdkV2:token` — access token stored/restored/refreshed.
- `irsSdkV2:recovery` — SDK storage reset/recovery occurred.
- `irispheraDataReady` — recommendation/profile-derived session data is ready (silhouette, palette, sizing, recommendations). This is emitted in the profile/detection completion flow; do not assume every standalone `fetchRecommendations()` call will emit it.

```js
window.addEventListener('irsSdkV2:ready', (e) => {
  console.log('Irisphera SDK ready', e.detail);
});

window.addEventListener('irsSdkV2:token', (e) => {
  console.log('Irisphera token event', e.detail);
});

window.addEventListener('irsSdkV2:recovery', (e) => {
  console.warn('Irisphera SDK recovery', e.detail);
});
```

### 1.4 Marketplace Product Feed (Required)

You must expose an **HTTPS GET endpoint** that returns your catalog in the structure below. Irisphera will call this URL on a schedule to ingest your products.

**Feed endpoint requirements**

- Provide a stable URL such as `https://partner.example.com/api/irisphera/feed`.
- **Do not make this endpoint public.** Protect it with API key authentication.
- Irisphera will call your endpoint using the same API key we provide to you. Your endpoint must validate the `irisphera-api-key` header and reject unauthorized requests.
- Response format: JSON array of products.

**Authentication flow:**

1. Irisphera calls your feed endpoint with the header `irisphera-api-key: <YOUR_API_KEY>`
2. Your server validates the API key matches the one we provided
3. If valid, return the product feed; otherwise return `401 Unauthorized`

**Example request from Irisphera to your endpoint:**

```
GET https://partner.example.com/api/irisphera/feed
Headers:
  irisphera-api-key: <YOUR_API_KEY>
```

#### Product DTO Schema

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `skuCustomId` | string | Yes | Unique per color variation (see section 1.5). |
| `title` | string | Yes | Product title. Append color/variant info for uniqueness (e.g., "Classic Sweater - Red"). |
| `description` | string | No | Product description (plain text). |
| `gender` | string | No | One of: `women`, `men`, `unisex`. Recommended for better recommendations. |
| `product_featured_image` | string (URL) | Yes* | Primary display image. Used for duplicate detection. *At least one image field is required. |
| `product_front_image` | string (URL) | No | Front-facing product image URL (still-life). |
| `product_back_image` | string (URL) | No | Back-facing product image URL (still-life). |
| `product_images` | string[] | No | Array of all product image URLs. |
| `product_page_url` | string (URL) | No | URL to the product page. Include variant param if applicable (e.g., `?variant=123`). |
| `date_added` | string | No | ISO 8601 date (e.g., `2026-02-01`). Used for seasonality filtering. |
| `product_price` | string | No | Regular price (e.g., `"99.00"`). |
| `product_special_price` | string | No | Sale/promotional price. |

**Image field priority:** The system requires at least one image. It resolves in this order: `product_featured_image` > `product_front_image` > first item of `product_images`. Products without any image are skipped during import.

**Example feed response**

```json
[
  {
    "skuCustomId": "12345678901",
    "title": "Classic Sweater - Red",
    "description": "Merino wool sweater with ribbed cuffs",
    "gender": "women",
    "product_front_image": "https://cdn.example.com/img/sweater-red-front.jpg",
    "product_back_image": "https://cdn.example.com/img/sweater-red-back.jpg",
    "product_images": [
      "https://cdn.example.com/img/sweater-red-front.jpg",
      "https://cdn.example.com/img/sweater-red-back.jpg",
      "https://cdn.example.com/img/sweater-red-detail.jpg"
    ],
    "product_featured_image": "https://cdn.example.com/img/sweater-red-front.jpg",
    "product_page_url": "https://shop.example.com/products/classic-sweater?variant=12345678901",
    "date_added": "2026-02-01T10:24:00Z"
  },
  {
    "skuCustomId": "12345678902",
    "title": "Classic Sweater - Blue",
    "description": "Merino wool sweater with ribbed cuffs",
    "gender": "women",
    "product_front_image": "https://cdn.example.com/img/sweater-blue-front.jpg",
    "product_back_image": "https://cdn.example.com/img/sweater-blue-back.jpg",
    "product_images": [
      "https://cdn.example.com/img/sweater-blue-front.jpg",
      "https://cdn.example.com/img/sweater-blue-back.jpg"
    ],
    "product_featured_image": "https://cdn.example.com/img/sweater-blue-front.jpg",
    "product_page_url": "https://shop.example.com/products/classic-sweater?variant=12345678902",
    "date_added": "2026-02-01T10:24:00Z"
  }
]
```

### 1.5 skuCustomId Rules (Color-Level Uniqueness)

- `skuCustomId` must represent **one unique color variation** of a product.
- All sizes of the same color **share the same `skuCustomId`**.
- Different colors **must use different `skuCustomId` values**.

**Canonical variant concept:** If your catalog has size-level SKUs (e.g., variant IDs per size), you must create a mapping to a "canonical" color-level identifier. This canonical `skuCustomId` should be:
- The first variant ID of each color group, OR
- A stable product-color identifier you define (e.g., `PRODUCT_123-RED`)

Use this canonical value everywhere Irisphera is called: product feed, purchase events, recommendations, VTO, and 3D preview.

**skuCustomId resolution priority (for purchase events):**

When processing order line items, resolve `skuCustomId` in this order:
1. `product_id` (if available and represents color-level)
2. `sku` (if color-level)
3. `variant_id` (if you've mapped variants to canonical IDs)

```js
// Example: resolving skuCustomId from an order line item
function getSkuCustomId(lineItem, variantToCanonicalMap) {
  // If you have a mapping from size-level variants to canonical color-level IDs
  if (variantToCanonicalMap && lineItem.variantId) {
    const canonical = variantToCanonicalMap[lineItem.variantId];
    if (canonical) return canonical;
  }
  // Fallback priority
  return lineItem.productId || lineItem.sku || lineItem.variantId;
}
```

### 1.6 Purchase Event Webhook (Mandatory)

You **must** send purchase data to Irisphera for every completed order, regardless of whether the customer used Irisphera features. This is essential for:

- Attribution reporting (did a recommendation lead to a sale?)
- Model training (learning which products convert)
- ROI measurement for your integration

**Implementation pattern:**

When an order is created in your system, your backend must:

1. Reconstruct the `endUserId` for the purchasing visitor (same logic as section 1.2)
2. Request a new access token for that `endUserId`
3. Call the purchase endpoint with the list of purchased SKUs

**Step 1: Generate endUserId from order context**

Extract `clientIp` and `userAgent` from the order (most e-commerce platforms store this). Use the same hashing logic:

```js
// Server-side
const crypto = require('crypto');

function getEndUserIdFromOrder(order) {
  const clientIp = order.clientDetails?.browserIp || order.clientIp || '';
  const userAgent = order.clientDetails?.userAgent || order.userAgent || '';
  return crypto.createHash('md5').update(`${clientIp}${userAgent}`).digest('hex');
}
```

**Step 2: Get access token (server-side)**

```bash
curl -X GET \
  "https://api.irisphera.com/merchant/v1/access-token?endUserId=<HASHED_END_USER_ID>" \
  -H "irisphera-api-key: <YOUR_API_KEY>"
```

**Step 3: Send purchase data (server-side)**

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

**Purchase payload schema:**

| Field | Type | Notes |
| --- | --- | --- |
| `skuCustomIds` | array | Required. List of purchased items. |
| `skuCustomIds[].skuCustomId` | string | Required. The canonical color-level SKU. |
| `skuCustomIds[].price` | string | Recommended for revenue/ROI accuracy. Format: `"<amount> <CURRENCY>"` (e.g., `"99.00 EUR"`). |

`price` is strongly recommended and should be sent whenever available. Some deployments may accept entries without `price`, but omitting it reduces downstream revenue attribution quality. If you use strict generated API clients in your backend stack, treat `price` as required for compatibility.

**Handling quantity > 1:**

If a line item has quantity greater than 1, include one entry per unit:

```json
{
  "skuCustomIds": [
    { "skuCustomId": "sweater-red", "price": "99.00 EUR" },
    { "skuCustomId": "sweater-red", "price": "99.00 EUR" },
    { "skuCustomId": "sweater-red", "price": "99.00 EUR" }
  ]
}
```

**Handling discounts:**

Calculate the effective per-unit price after discounts:

```js
// If line item has total_discount that applies to all units
const effectiveUnitPrice = unitPrice - (totalDiscount / quantity);
```

**Order cancellation:**

If an order is cancelled, send a DELETE request to remove the purchase record:

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

**Complete server-side example (Node.js):**

```js
const crypto = require('crypto');

async function handleOrderCreated(order) {
  // 1. Reconstruct endUserId
  const clientIp = order.clientDetails?.browserIp || '';
  const userAgent = order.clientDetails?.userAgent || '';
  const endUserId = crypto.createHash('md5').update(`${clientIp}${userAgent}`).digest('hex');

  // 2. Get access token
  const tokenRes = await fetch(
    `https://api.irisphera.com/merchant/v1/access-token?endUserId=${endUserId}`,
    { headers: { 'irisphera-api-key': process.env.IRISPHERA_API_KEY } }
  );
  const { accessToken } = await tokenRes.json();

  // 3. Build purchase payload
  const currency = order.currency?.toUpperCase() || '';
  const skuCustomIds = order.lineItems.flatMap(item => {
    const skuCustomId = item.productId || item.sku || item.variantId;
    if (!skuCustomId) return [];

    const unitPrice = parseFloat(item.price) || 0;
    const totalDiscount = parseFloat(item.totalDiscount) || 0;
    const quantity = item.quantity || 1;
    const effectivePrice = unitPrice - (totalDiscount / quantity);
    const priceStr = currency ? `${effectivePrice.toFixed(2)} ${currency}` : effectivePrice.toFixed(2);

    // One entry per unit purchased
    return Array(quantity).fill({ skuCustomId: String(skuCustomId), price: priceStr });
  });

  if (skuCustomIds.length === 0) return;

  // 4. Send to Irisphera
  await fetch('https://api.irisphera.com/shopper/v1/data/purchase', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${accessToken}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ skuCustomIds })
  });
}
```

## 2. Virtual Personal Shopper + Recommendations

This feature captures a user profile and returns personalized recommendations. The SDK handles the iframe flow and caches results.

### 2.1 Core SDK Functions

- `showIrispheraIframe(document, divId, onComplete)` - Main profile capture iframe
- `showIrispheraIframeCta(document, divId, language, onCreateProfile, onViewRecommendations)` - CTA banner for users without profile
- `showIrispheraIframeBanner(document, divId, language, onEditProfile)` - Banner for users with existing profile
- `fetchRecommendations(options)` - Fetches recommendations from server
- `isUserProfileSavedToCookies()` - Returns `true` if user has completed profile
- `Results.sessionRecommendations` - Array of `{ skuCustomId, size }` after `fetchRecommendations()`
- `Results.sessionSilhouette` - User's body type classification
- `Results.sessionPalette` - User's color palette classification

### 2.2 Typical Flow (CTA + modal)

**Check if user has profile before deciding which banner to show:**

```js
if (window.irsSdkV2.isUserProfileSavedToCookies()) {
  // User has profile - show recommendations banner
  window.irsSdkV2.showIrispheraIframeBanner(
    document,
    'irsBannerDiv',
    null,
    () => {
      // Edit profile callback
      openModal();
      window.irsSdkV2.showIrispheraIframe(document, 'irsIframeDiv', () => {
        closeModal();
        window.location.reload();
      });
    }
  );
} else {
  // User has no profile - show CTA banner
  window.irsSdkV2.showIrispheraIframeCta(
    document,
    'irsBannerDiv',
    null,
    () => {
      // Create profile callback
      openModal();
      window.irsSdkV2.showIrispheraIframe(document, 'irsIframeDiv', async () => {
        await window.irsSdkV2.fetchRecommendations();
        closeModal();
        renderRecommendations();
      });
    },
    () => {
      // View recommendations callback (profile exists from another tab/session)
      renderRecommendations();
    }
  );
}
```

### 2.3 Accessing Recommendation Results

```js
const recs = window.irsSdkV2.Results.sessionRecommendations; // [{ skuCustomId, size }]
```

Use the `skuCustomId` to look up catalog products from your marketplace feed, and optionally surface the recommended size.

**Caching behavior to be aware of:**

- SDK recommendation cache is profile-bound and filter-bound.
- Cached recommendations expire after ~10 minutes.
- If profile data changes, cached recommendations are invalidated automatically.

If you need fresh recommendations immediately after profile or filter changes, call `fetchRecommendations()` again.

### 2.4 Tracking User Events

Use the SDK host event helpers to send engagement data:

```js
window.irsSdkV2.HostEventsHandler.onProductView(skuCustomId);
window.irsSdkV2.HostEventsHandler.onAddToCart(skuCustomId);
window.irsSdkV2.HostEventsHandler.onRemoveFromCart(skuCustomId);
window.irsSdkV2.HostEventsHandler.onPurchase([skuCustomId1, skuCustomId2]);
```

These call the shopper data endpoints for analytics and model feedback.

`HostEventsHandler.onFilterChecked()` exists in the SDK API surface but is currently a placeholder (not implemented). Do not rely on it for production tracking.

`/shopper/v1/data/vps-open` may exist in generated API contracts but is not a required part of this integration guide; follow the currently implemented SDK event helpers listed above.

> **Important:** `HostEventsHandler.onPurchase(...)` sends only SKU IDs. It is useful for client event tracking, but it does **not** replace the mandatory server-side purchase webhook flow in section **1.6**, where you should send unit-level purchase data (including price/currency) from your backend.

### 2.5 Product Page Sizing Block (Optional)

For product detail pages, use the sizing iframe to show personalized size recommendations and provide access to VTO/3D features:

```js
window.irsSdkV2.showIrispheraIframeSizing(
  document,
  'irsSizingDiv',           // target div ID
  null,                     // language (auto-detect)
  recommendedSize,          // pre-computed size from sessionRecommendations (or null)
  skuCustomId,              // canonical SKU for this product
  onEditProfile,            // callback: user wants to edit profile
  onViewRecommendations,    // callback: user wants to see recommendations
  onVto,                    // callback: user triggered VTO
  onTdPreview               // callback: user triggered 3D preview (receives glbUrl)
);
```

**Callback signatures:**

```js
function onEditProfile() {
  // Open modal with profile capture iframe
  openModal();
  window.irsSdkV2.showIrispheraIframe(document, 'irsIframeDiv', () => {
    closeModal();
    window.location.reload(); // Refresh to show updated size
  });
}

function onViewRecommendations() {
  // Navigate to recommendations page
  window.location.href = '/collections/my-recommendations';
}

function onVto() {
  // Open VTO iframe
  openModal();
  window.irsSdkV2.showIrispheraIframeVto(document, 'irsIframeDiv', null, skuCustomId);
}

function onTdPreview(glbUrl) {
  // Open 3D preview iframe (glbUrl is passed from the sizing iframe)
  if (!glbUrl) return;
  openModal();
  window.irsSdkV2.showIrispheraIframeTdPreview(document, 'irsIframeDiv', null, skuCustomId, glbUrl);
}
```

**Getting the recommended size for a product:**

```js
// After fetchRecommendations() has been called
const recs = window.irsSdkV2.Results.sessionRecommendations || [];
const productRec = recs.find(r => r.skuCustomId === currentSkuCustomId);
const recommendedSize = productRec?.size || null;
```

## 3. Virtual Try On (VTO)

VTO opens a dedicated iframe that renders the try-on experience for a given `skuCustomId`.

### 3.1 SDK Functions

- `showIrispheraIframeVto(document, divId, language, skuCustomId)`
- `closeIrispheraIframeVto()`

### 3.2 Typical Flow

```js
openModal();
window.irsSdkV2.showIrispheraIframeVto(document, 'irsIframeDiv', null, skuCustomId);
```

**Optional readiness check**

You can check if a SKU is ready for VTO before showing the VTO button:

`GET https://api.irisphera.com/shopper/v1/stylist-preview/{skuCustomId}`

| Response Code | Meaning |
| --- | --- |
| `204 No Content` | VTO is available for this SKU |
| `422 Unprocessable Entity` | VTO not available (e.g., swimwear, missing assets, or disabled) |
| `404 Not Found` | SKU not found in catalog |

```js
// Example: check VTO availability before showing button
async function isVtoAvailable(skuCustomId, accessToken) {
  try {
    const res = await fetch(
      `https://api.irisphera.com/shopper/v1/stylist-preview/${skuCustomId}`,
      { headers: { 'Authorization': `Bearer ${accessToken}` } }
    );
    return res.status === 204;
  } catch {
    return false;
  }
}
```

## 4. 360 / 3D Preview

The 3D preview uses a dedicated iframe that needs a `skuCustomId` and a `glbUrl` (presigned asset URL).

### 4.1 SDK Functions

- `showIrispheraIframeTdPreview(document, divId, language, skuCustomId, glbUrl)`
- `closeIrispheraIframeTdPreview()`

### 4.2 Supplying the GLB URL

Two supported approaches:

1) **From the sizing iframe event**: when the sizing iframe triggers `TD_PREVIEW_TRIGGERED`, it sends a `glbUrl`. Pass that directly to `showIrispheraIframeTdPreview`.

2) **From the API**: call the 3D preview endpoint on your server and pass the returned URL:

`GET https://api.irisphera.com/shopper/v1/td-preview/{skuCustomId}`

| Response Code | Meaning |
| --- | --- |
| `200 OK` | Returns a presigned GLB URL string payload (commonly JSON-encoded string) |
| `422 Unprocessable Entity` | No 3D assets available for this SKU |
| `404 Not Found` | SKU not found in catalog |

Example usage:

```js
// Check 3D availability and get GLB URL
async function get3dPreviewUrl(skuCustomId, accessToken) {
  try {
    const res = await fetch(
      `https://api.irisphera.com/shopper/v1/td-preview/${skuCustomId}`,
      { headers: { 'Authorization': `Bearer ${accessToken}` } }
    );
    if (res.status === 200) {
      // Compatible with both JSON-encoded string and plain text URL payloads.
      const raw = await res.text();
      try {
        return JSON.parse(raw);
      } catch {
        return raw;
      }
    }
    return null;
  } catch {
    return null;
  }
}

// Open 3D preview
const glbUrl = await get3dPreviewUrl(skuCustomId, accessToken);
if (glbUrl) {
  openModal();
  window.irsSdkV2.showIrispheraIframeTdPreview(
    document,
    'irsIframeDiv',
    null,
    skuCustomId,
    glbUrl
  );
}
```

---

## 5. Troubleshooting: Why Iframes May Not Display

Iframes will silently fail to render in several scenarios:

| Scenario | Cause | Solution |
| --- | --- | --- |
| **No access token** | `setAccessToken()` was never called or failed | Verify token fetch from your backend succeeds |
| **Invalid/expired token** | Token rejected by Irisphera API | Check token expiry (25 min); implement `registerTokenFetcher` for auto-refresh |
| **Wrong iframe origin config** | `window.IRISPHERA_TARGET_ORIGIN` missing or incorrect | Set target origin before SDK script load; verify exact scheme/host/port |
| **VTO not available** | Product not ready for VTO (missing assets, swimwear, etc.) | Call readiness check API before showing VTO button |
| **3D not available** | No GLB assets for this SKU | Call 3D preview API; only show button if URL returned |
| **SKU not in catalog** | `skuCustomId` doesn't exist in your product feed | Verify product feed contains this SKU |
| **Network/CORS issues** | Browser blocking iframe or API requests | Check browser console for CORS errors |

### Enabling Debug Logs

The SDK log level defaults to `none`. For diagnosis, enable debug logs at runtime:

```js
await window.irsSdkV2.whenReady();
window.irsSdkV2.logLevel = 'debug'; // supported levels: debug, info, warn, error, none
```

If needed, you can still use a local SDK override workflow for deep debugging.

**Step 1 (optional): Create a browser override**

Use your browser's developer tools to redirect the CDN URL to your local file:

**Chrome DevTools:**
1. Open DevTools (F12) > Sources tab
2. Right-click in the left panel > Add folder to workspace > select folder containing your local SDK
3. Right-click on the CDN-loaded `irispheraSdk.js` in the Network tab
4. Select "Override content" and map to your local file

**Firefox:**
1. Open DevTools > Network tab
2. Right-click the SDK request > "Edit and Resend" won't persist; use an extension like "Redirector" to map the CDN URL to a local server (e.g., `file://` or `localhost`)

**Alternative: Local server override**

```bash
# Serve local SDK on port 8080
npx serve -l 8080 .

# Then use browser extension or hosts file to redirect CDN requests
```

**Step 2: Manually set the access token (if using local override)**

**Important:** The SDK stores the access token in cookies under the original domain (e.g., `yourstore.com`). When you override the SDK with a local file served from `localhost` or `file://`, the local SDK cannot read cookies from the original domain due to browser security restrictions.

You must manually set the token via the browser console:

```js
// In browser console, after page loads:
// 1. First, get a valid token from your backend (or copy from Network tab)
const token = '<paste-your-access-token-here>';

// 2. Manually set it in the SDK
window.irsSdkV2.setAccessToken(token, { skipValidation: true });
```

**Tip:** You can copy the token from the Network tab by finding the `/getUserToken` (or equivalent) request and copying the `accessToken` value from the response.

**Step 3: Reload and check console**

With debug logging enabled, the console will show:
- Token fetch/set events
- Iframe creation and initialization
- Message passing between parent and iframe
- API call results and errors
- VTO/3D readiness check results

Example debug output:
```
[IRS SDK] DEBUG: setAccessToken called with skipValidation=true
[IRS SDK] DEBUG: Token saved to storage, expires in 1500s
[IRS SDK] DEBUG: showIrispheraIframeVto called for SKU: 12345678901
[IRS SDK] DEBUG: Creating iframe with src: https://...
[IRS SDK] DEBUG: Iframe ready, sending VTO_INIT payload
[IRS SDK] ERROR: VTO init failed - SKU not available for VTO
```

### Message origin security checks

The SDK validates iframe postMessage events against the configured target origin and expected iframe window source. If origin config is wrong, messages are rejected and iframes may appear stuck or blank.

Checklist:

1. Ensure `window.IRISPHERA_TARGET_ORIGIN` is correct and set before loading SDK.
2. Ensure your site is loading the expected Irisphera iframe host.
3. Check console logs for invalid origin/source errors.

---

## 6. SDK Helper Functions Reference

| Function | Returns | Description |
| --- | --- | --- |
| `whenReady()` | `Promise<SDK>` | Resolves when SDK is fully initialized |
| `whenTokenReady()` | `Promise<void>` | Resolves when an access token is available in SDK state |
| `setAccessToken(token, options)` | `Promise<void>` | Sets the access token. Use `{ skipValidation: true }` when server-validated. |
| `registerTokenFetcher(fn)` | `void` | Registers async function to refresh tokens when they expire |
| `isUserProfileSavedToCookies()` | `boolean` | Returns true if user has completed profile capture (legacy name; backed by unified SDK storage) |
| `fetchRecommendations()` | `Promise<void>` | Fetches recommendations; results in `Results.sessionRecommendations` |
| `closeIrispheraIframe()` | `void` | Closes the currently active iframe |
| `resetStorage()` | `void` | Clears SDK storage/cookies and triggers recovery event (useful for support/debug) |
| `_getAccessToken()` | `string \| null` | Returns current access token (useful for checking if token is set) |

---

If you need a sample implementation of the modal/iframe wrapper or a reference integration, use the recommended pattern: load the SDK, fetch a token server-side for each visitor, set it via `setAccessToken`, and open iframes inside a dedicated modal container. This is the default model Irisphera tests and supports.

For integration support or questions, contact your Irisphera technical representative.

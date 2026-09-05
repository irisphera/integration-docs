# Irisphera API Integration Guide

Onboard merchants, load catalogs, and collect shopper activity through Octopus.
This guide describes checked-in code, not verified production availability. Set
`IRISPHERA_BASE_URL` to your approved environment and check its `/v3/api-docs` before
generating a client or enabling v2.

- [Authentication](#authentication)
- [Merchant onboarding](#merchant-onboarding)
- [Catalog ingestion](#catalog-ingestion)
- [Shopper sessions and identity](#shopper-sessions-and-identity)
- [Interaction events](#interaction-events)
- [Commerce events](#commerce-events)
- [Shopper experiences](#shopper-experiences)
- [Rollout](#rollout)
- [Errors and retries](#errors-and-retries)

## Authentication

Send exactly one credential per request. API keys belong on trusted backends only.

| Routes | Header |
| --- | --- |
| `/integrator/v1/*` | `INTEGRATOR-API-KEY: $INTEGRATOR_API_KEY` |
| `/merchant/v1/*` | `MERCHANT-API-KEY: $MERCHANT_API_KEY` |
| `/merchant/v2/*`, except customer-alias links | `CHANNEL-API-KEY: $CHANNEL_API_KEY` |
| `/merchant/v2/customer-alias-links/{linkId}` | `IDENTITY-ADMIN-API-KEY: $IDENTITY_ADMIN_API_KEY` |
| `/shopper/v2/*` | `Authorization: Bearer $ACCESS_TOKEN`; v2 session token required |
| `/shopper/v1/*` | `Authorization: Bearer $ACCESS_TOKEN`; see experience and legacy flows below |

`irisphera-api-key` is not the current authentication header. Merchant keys do not
substitute for channel credentials. Each channel credential is bound to one merchant,
channel instance, and installation epoch, with granted operation scopes and identity domains.

Inject keys from a secret manager. Do not log tokens, continuation secrets, raw identity
aliases, or merchant response bodies containing `apiKey`. Keep TLS verification enabled.

## Merchant onboarding

1. Obtain an integrator key and generate a merchant key in your secret manager.
2. Create the merchant; retain its `id` as `MERCHANT_ID`.
3. Create a collection; retain its `id` as `COLLECTION_ID`.
4. Ingest products and check their visibility before enabling shopper features.

```bash
curl -sS "$IRISPHERA_BASE_URL/integrator/v1/merchant" \
  -H "INTEGRATOR-API-KEY: $INTEGRATOR_API_KEY" \
  -H "Content-Type: application/json" \
  --data-binary @merchant.json
```

Supply `merchant.json` at runtime from your secret store, with required `name` and
`apiKey` fields. Do not commit it. Creation returns `201` and the merchant record,
including its key. Repeating the same key returns your existing merchant unchanged;
a matching name updates your merchant in place. Retry only with the same intended input.

| Operation | Route / behavior |
| --- | --- |
| List | `GET /integrator/v1/merchant`; array of owned merchants, `X-Total-Count` header |
| Read | `GET /integrator/v1/merchant/{merchantId}` |
| Update | `PUT /integrator/v1/merchant/{merchantId}`; requires `name` and `apiKey` |
| Read key | `GET /integrator/v1/merchant/{merchantId}/apikey`; secret response |
| Delete | `DELETE /integrator/v1/merchant/{merchantId}`; permanent, `204`; repeat returns `404` |

Ownership comes from the integrator key, not a body `integratorId`. Another integrator's
merchant returns `403`; a missing merchant returns `404`. Updates preserve omitted
configuration values. Supported configuration:

| Field | Values / default |
| --- | --- |
| `themeConfig.themeFile` | `default.json` |
| `flowConfig.apparel` | `MENSWEAR`, `WOMENSWEAR`, `ALL` (default) |
| `flowConfig.recommendationCriteria` | `NONE`, `PALETTE`, `SILHOUETTE`, `SIZING`, `ALL` (default) |
| `flowConfig.profileWizard` | `SIMPLE`, `MANNEQUIN` (default) |
| `flowConfig.recommendationTopK` | Integer ≥ 10; default `50` |

Create a collection with the merchant key:

```bash
curl -sS "$IRISPHERA_BASE_URL/merchant/v1/collection" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"title":"Autumn catalog"}'
```

The `201` response contains `id`, `merchantId`, `title`, and optional `productFeed`.
Set `productFeed` at creation if you need URL import.
`GET /merchant/v1/collection` returns `{"collections":[...]}`.
**Current limitation:** collection `PUT` creates a new collection ID rather than updating.
Collection deletion, file-based deletion, and per-SKU deletion return `501`.

## Catalog ingestion

### Product identifiers and formats

Use one stable `skuCustomId` per model-and-color combination. Map size-level source SKUs
to this ID and reuse it in recommendations, previews, and events. Do not substitute
an order line ID or a Shopify variant ID unless it is your actual catalog identifier.

Single-product requests use camelCase:

```json
{
  "skuCustomId": "STYLE-100-BLACK",
  "title": "Black wool blazer",
  "description": "Single-breasted wool blazer.",
  "gender": "WOMEN",
  "productFrontImage": "https://cdn.example.com/blazer-front.webp",
  "productBackImage": "https://cdn.example.com/blazer-back.webp",
  "productImages": [
    "https://cdn.example.com/blazer-front.webp",
    "https://cdn.example.com/blazer-back.webp"
  ],
  "productFeaturedImage": "https://cdn.example.com/blazer-front.webp",
  "productPageUrl": "https://shop.example.com/products/blazer"
}
```

`skuCustomId`, `title`, `description`, and a nonempty `productImages` array are required.
Supply the featured image and product page for collection processing, and front/back
views for IGG or VTO. Supported gender values are `MEN`, `WOMEN`, `UNISEX`,
`CHILDREN_GIRL`, and `CHILDREN_BOY`.
Images may be reachable URLs or data URIs; supported bytes are PNG, JPEG, WebP, and AVIF.
Prefer HTTPS URLs that remain available during asynchronous processing. Catalog price
persistence is not guaranteed; use the commerce event amount fields for reporting.

Bulk JSON requires a `products` wrapper and snake_case image/page fields:

```json
{
  "products": [{
    "skuCustomId": "STYLE-100-BLACK",
    "title": "Black wool blazer",
    "description": "Single-breasted wool blazer.",
    "gender": "WOMEN",
    "product_front_image": "https://cdn.example.com/blazer-front.webp",
    "product_back_image": "https://cdn.example.com/blazer-back.webp",
    "product_images": ["https://cdn.example.com/blazer-front.webp", "https://cdn.example.com/blazer-back.webp"],
    "product_featured_image": "https://cdn.example.com/blazer-front.webp",
    "product_page_url": "https://shop.example.com/products/blazer"
  }]
}
```

CSV uses camelCase headers and `|` between `productImages` URLs:

```csv
skuCustomId,title,description,gender,productFrontImage,productBackImage,productImages,productFeaturedImage,productPageUrl
STYLE-100-BLACK,Black wool blazer,Single-breasted wool blazer.,WOMEN,https://cdn.example.com/front.webp,https://cdn.example.com/back.webp,https://cdn.example.com/front.webp|https://cdn.example.com/back.webp,https://cdn.example.com/front.webp,https://shop.example.com/products/blazer
```

CSV headers are case-insensitive. `skuCustomId`, `title`, and at least one of
`productFrontImage` or `productFeaturedImage` must be present. Rows missing those values
may be skipped. JSON and CSV are supported; XLSX is not. The configured multipart limit
is currently 100 MB, not a permanent API limit.

### Submit and verify

Choose one submission route:

| Method and route | Input |
| --- | --- |
| `POST /merchant/v1/collection/{collectionId}/products` | Single-product JSON above |
| `POST /merchant/v1/collection/{collectionId}/file?useSeasonFiltering=false` | Multipart `file`, media type `application/json` or `text/csv` |
| `POST /merchant/v1/collection/{collectionId}/import-from-url` | Optional `{"useSeasonFiltering":false}`; fetches the collection's `productFeed` |

```bash
curl -sS "$IRISPHERA_BASE_URL/merchant/v1/collection/$COLLECTION_ID/file" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" \
  -F "file=@products.json;type=application/json"
```

Ingestion returns `202`, not a completion receipt. The file controller currently discards
submission failures and can return `202` when nothing was queued. Verify with
`GET /merchant/v1/collection/{collectionId}/products`, whose body is `{"fashionItems":[...]}`.
Find the expected `skuCustomId`; an empty array is valid and does not prove completion.
There is no public batch-status endpoint, callback, or completion deadline.

Within a feed, later duplicate SKUs are filtered. Existing collection items are skipped
when `irisphera.import.batch.cache.skip-existing-collection-items` is enabled (the default).
Do not assume upsert semantics.

**URL import warning:** the current implementation forwards incoming authentication
headers to the feed host after filtering selected transport headers. Use single-product
or file ingestion for external/untrusted hosts; do not send credentials to a third-party feed.

### Other merchant operations

- `GET /merchant/v1/products`: `fashionItems` across accessible collections; does not populate presigned image URLs.
- `GET /merchant/v1/subscription`: `featureDetection`, `shopperRecommendation`, and `shopper2dPreview`, each with `limit` and `current`.
- `POST /merchant/v1/report`: aggregate report for `{"startTime":"2026-01-01T00:00:00Z","endTime":"2026-02-01T00:00:00Z","zone":"Europe/Bucharest"}`; interval is `[startTime,endTime)`.
- `POST /merchant/v1/vto2d`: demo VTO; multipart `featuredImage` and `userPhoto`, optional `description` query parameter. Not the shopper flow.
- `POST /merchant/v1/products/style-occasion-analysis`: currently `501`.

## Shopper sessions and identity

V2 separates your external aliases from Octopus's canonical `shopperId` UUID. A namespace
must be registered to the merchant and granted to the channel. Use a random anonymous
epoch before login and a verified platform customer ID after login. Never use email,
IP address, user agent, or a device fingerprint as proof of customer identity.

### Create and inspect a session

Call from your backend after validating the storefront session. This example assumes
the namespace and channel have already been registered:

```bash
curl -sS "$IRISPHERA_BASE_URL/merchant/v2/shopper-sessions" \
  -H "CHANNEL-API-KEY: $CHANNEL_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $SESSION_OPERATION_ID" \
  -d '{"channelId":"0198f2a0-7b42-7000-8000-000000000201","currentIdentity":{"namespace":"channel/0198f2a0-7b42-7000-8000-000000000201/anonymous","kind":"ANONYMOUS","externalId":"epoch-7b0915a4"}}'
```

The response contains `sessionId`, canonical `shopperId`, `accessToken`, `tokenType`,
`expiresAt`, `identityVersion`, `linkStatus`, `attributionRef`, and, for a new anonymous
session, `anonymousContinuation`. Responses use `Cache-Control: no-store`. Pass only the shopper bearer token and nonsecret UI/session
metadata to shopper code. Keep the continuation in server-controlled state, never
cart attributes or arbitrary JavaScript.
Use the returned expiry, not a hard-coded token lifetime.

```bash
curl -sS "$IRISPHERA_BASE_URL/shopper/v2/session" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

### Login, refresh, and logout

On login, call the session endpoint with the verified customer alias and proof of the
immediately preceding anonymous session:

```json
{
  "channelId": "0198f2a0-7b42-7000-8000-000000000201",
  "linkOperationId": "0198f2a0-7b42-7000-8000-000000000301",
  "currentIdentity": {
    "namespace": "channel/0198f2a0-7b42-7000-8000-000000000201/customer",
    "kind": "CUSTOMER",
    "externalId": "customer-123"
  },
  "previousAnonymousSession": {
    "sessionId": "0198f2a0-7b42-7000-8000-000000000401",
    "anonymousContinuation": "<server-held continuation>"
  }
}
```

The two transition fields must be supplied together. Do not move an anonymous identity
already linked to a different customer. A direct customer session without anonymous proof
can sign in the customer but must not link unrelated browsing history.

| Operation | Route |
| --- | --- |
| Refresh from trusted backend | `POST /merchant/v2/shopper-sessions/{sessionId}/access-tokens` |
| Revoke from trusted backend | `DELETE /merchant/v2/shopper-sessions/{sessionId}` |
| Shopper logout | `DELETE /shopper/v2/session` |
| Durable anonymous-to-customer link | `PUT /merchant/v2/shopper-identity-links/{linkId}` |
| Privileged customer-alias link | `PUT /merchant/v2/customer-alias-links/{linkId}` |

Refresh only after revalidating the current platform login or anonymous continuation.
Account changes require a new session, not a refresh. On logout, discard tokens and
continuation state and create a new anonymous epoch. Do not reuse a logged-in customer's
identity for the next visitor on a shared browser.

Durable links require `identity:link`, the registered domain grants, and anonymous-session
proof; raw aliases are not link authority. Reuse the login's `linkOperationId` as `linkId`
when retrying that same link through an outbox. Customer-to-customer alias links require the
separate identity-admin credential and approved evidence. See the environment's OpenAPI
for those request shapes; do not infer a link from equal email addresses.

### Legacy compatibility

`GET /merchant/v1/access-token?shopperId=<external-id>` still issues legacy tokens with a
merchant key. `GET /shopper/v1/auth/access-token` returns legacy `TokenInfo` and UI config.
When legacy identify compatibility is enabled, `POST /shopper/v1/data/identify` uses
`anonymousShopperId` and `customerId`; it can
return `409` for no recorded activity or an existing association. Do not mix that flow
with v2 session continuation and canonical UUIDs, or fall back to it after a v2 auth failure.

## Interaction events

Use `PUT /shopper/v2/events/{sourceEventId}` with the v2 shopper bearer token. Generate
one UUID when the event occurs, not on each delivery attempt. `channelId` must match the
token. Do not send `subject`, customer IDs, aliases, or an order payload from the browser.

```bash
curl -sS -X PUT "$IRISPHERA_BASE_URL/shopper/v2/events/$SOURCE_EVENT_ID" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"schemaVersion":2,"channelId":"0198f2a0-7b42-7000-8000-000000000201","type":"PRODUCT_VIEWED","occurredAt":"2026-01-15T10:00:00Z","product":{"skuCustomId":"STYLE-100-BLACK"}}'
```

Observation types are `PRODUCT_VIEWED`, `ADD_TO_CART`, `REMOVE_FROM_CART`, and
`VIRTUAL_TRY_ON`. Report observed actions, not an attempted request that failed.
For a trusted server outbox, use `PUT /merchant/v2/interaction-events/{sourceEventId}`
with `CHANNEL-API-KEY`, `events:write`, the same product payload, and a server-captured
`subject`. A background worker should not mint shopper JWTs to deliver observations.

## Commerce events

Report authoritative orders from your backend regardless of whether the customer used
Irisphera. These endpoints record facts; they do not create or change commerce-platform orders.
Use `PUT /merchant/v2/commerce-events/{sourceEventId}` with a channel credential holding
`events:write`. `ORDER_CREATED` means the platform accepted the order, not payment settlement.
Other types are `ORDER_CANCELLED`, `ORDER_RETURNED`, `REFUND`, and `PAYMENT_CAPTURED`.

Save the following as `order-event.json`:

```json
{
  "schemaVersion": 2,
  "channelId": "0198f2a0-7b42-7000-8000-000000000201",
  "type": "ORDER_CREATED",
  "occurredAt": "2026-01-15T10:05:00Z",
  "subject": {"orderGuest": {"sourceOrderId": "order-583910"}},
  "order": {
    "sourceOrderId": "order-583910",
    "currency": "EUR",
    "lines": [{
      "sourceLineId": "line-1",
      "skuCustomId": "STYLE-100-BLACK",
      "quantity": 2,
      "amounts": {
        "currency": "EUR",
        "merchandiseGrossAfterDiscount": "198.00",
        "merchandiseNetAfterDiscount": "165.00",
        "tax": "33.00",
        "discount": "0.00"
      }
    }]
  }
}
```

```bash
curl -sS -X PUT "$IRISPHERA_BASE_URL/merchant/v2/commerce-events/$SOURCE_EVENT_ID" \
  -H "CHANNEL-API-KEY: $CHANNEL_API_KEY" \
  -H "Content-Type: application/json" \
  --data-binary @order-event.json
```

Choose exactly one trusted subject selector:

- `{"externalIdentity":{"namespace":"<registered customer namespace>","kind":"CUSTOMER","externalId":"customer-123"}}` for a verified customer in a granted domain.
- `{"shopperId":"<canonical UUID>","identityVersion":1}` from trusted server state; the version must match.
- `{"orderGuest":{"sourceOrderId":"order-583910"}}` when no verified customer identity exists. This subject is order-scoped, not a cross-order guest identity.

An optional `attribution.attributionRef` correlates browsing with checkout. It is not
permission to select the order owner, mint a token, or link identities.

Amounts are **line totals**, in charged/presentment currency, as unsigned decimal strings
with at most four fractional digits. Gross is after merchandise discounts, includes tax,
and excludes shipping; net excludes tax. Use a matching uppercase currency code. `quantity`
is explicit; do not repeat SKUs to represent units as in v1. Returns and cancellations
reference original order/line IDs and affected quantities. Refunds carry their own
`sourceRefundId`; preserve source sequence/version when available. For a `REFUND`,
`order.refundAmount` is the total refund in `order.currency`, including non-line
adjustments. It can accompany affected lines or an empty `lines` array for an amount-only
refund. Empty lines are allowed only for `REFUND` with both `sourceRefundId` and
`refundAmount`. Do not add line amounts to that total again or invent quantities for
shipping-only refunds. Gross reports count one canonical `ORDER_CREATED` per
merchant/channel/order: the earliest source sequence, then numeric source version,
then source time and event ID—not the first delivery. `PAYMENT_CAPTURED` is not another
purchase, and refund money does not automatically reduce gross revenue.

Legacy `POST /shopper/v1/data/order-create`, `/order-cancelled`, and `/return` use
`{"skuCustomIds":[{"skuCustomId":"STYLE-100-BLACK","price":"99.00 EUR"}]}` with one entry
per unit. Their `204` acknowledgement is best-effort and not idempotent. Do not dual-write
the same event to v1 and v2.

## Shopper experiences

The experience routes remain `/shopper/v1` and accept the v2 bearer with live session
checks. Do not change the SDK's global API version to migrate data collection.
`GET /shopper/v2/session` supplies `sizingConfig`, `themeConfig`, `flowConfig`, expiry,
and scopes. Map `shopper:recommendations` to `isApsEnabled` and `shopper:vto` to
`isVtoEnabled`; these feature scopes depend on merchant quota. Session/event tokens
also carry `shopper:session` and `shopper:events`.

| Operation | Request / result |
| --- | --- |
| Recommendations | `POST /shopper/v1/recommendations`, JSON `{"encodedProfileData":"<base64 JSON>","filters":[]}`; returns `recommendationsByCollection` |
| VTO readiness | `GET /shopper/v1/stylist-preview/{skuCustomId}`; do not generate when `UNAVAILABLE` |
| Generate VTO | `POST /shopper/v1/stylist-preview?skuCustomId=STYLE-100-BLACK`, multipart `targetImage`; returns base64 WebP `generatedImage` |
| 3D asset | `GET /shopper/v1/td-preview/{skuCustomId}`; temporary download URL, do not persist |
| Body measurements | `POST /shopper/v1/body-measurements`, JSON `{"user_gender":"WOMEN","user_height":170,"img_data_main":"<raw base64>"}`; optional `img_data_side`, results in centimeters |
| Colors | `POST /shopper/v1/color-extraction`, JSON `{"img_data_main":"<raw base64 selfie>"}` |

Profile Base64 is encoding, not encryption. Obtain consent for photos and avoid retention.
VTO's optional `isFaceBlurred` query parameter describes client-side blurring; it does not
request server-side blurring. On `420` or `422`, request a usable photo or offer a product-page fallback.
Match recommended SKUs to your storefront for price and availability. The current
recommendation flow applies `filters` but ignores `offset`, `limit`, and `collectionIds`. The VTO-ready-item list route is not implemented; use per-SKU readiness.

## Rollout

### Backend prerequisites

Before enabling a channel, the Octopus operator must:

1. Apply the current Liquibase changelog, including identity, session, credential, and event tables.
2. Set `irisphera.shopper-identity.enabled=true` and supply `irisphera.service.key`
   (`IRISPHERA_SERVICE_KEY`) from a secret store.
3. Configure `irisphera.token.keys.<kid>` with at least 32 bytes of high-entropy key
   material. The parser uses the value's UTF-8 bytes; it does not Base64-decode it.
   Align issuer/audience settings. The default token `max-age` is `10m`.
4. Register the merchant's channel instance, current credential epoch, separate
   `ANONYMOUS`/`CUSTOMER` domains, and grants for `shopper:session`, `identity:link`,
   and `events:write` as needed. Merchant creation alone does not provision these.
5. Test session creation, login, refresh, logout, identical event replay, and conflicting
   replay in that environment before switching traffic. Keep v1 and v2 delivery mutually exclusive.

Platform privacy handlers erase local adapter state; they do not erase Octopus history.
There is no merchant-v2 remote-erasure endpoint. Before production, agree an authenticated
erasure procedure with the Octopus operator: retain local replay fences, submit the
merchant-scoped erasure request through the approved secure channel, and obtain separate
confirmation of backend deletion. Do not treat local success as remote completion.

### Shopify

Enable selected shops through server-only `OCTOPUS_COLLECTION_V2_CHANNELS` JSON:

```json
{
  "example.myshopify.com": {
    "channelId": "0198f2a0-7b42-7000-8000-000000000201",
    "apiKey": "<channel credential from secret store>",
    "namespace": "<registered CUSTOMER namespace>",
    "anonymousNamespace": "<registered ANONYMOUS namespace>"
  }
}
```

An absent shop entry keeps the legacy flow; an invalid configured entry fails rather
than falling back. Supply a dedicated stable `COLLECTION_PRIVACY_HMAC_SECRET` of at
least 32 characters. Its fingerprint is pinned; a missing or changed key fails closed.
Run `npx prisma migrate deploy --schema prisma/schema.prisma` and
`npx prisma generate --schema prisma/schema.prisma` before enabling v2.

For configured shops, `getUserToken` returns `collectionVersion:2` without a bearer.
The browser then calls signed `POST /apps/irisphera/shopperSession` with
`{"operation":"resolve","requestId":"<fresh UUIDv7>"}`; subsequent `refresh` or `logout`
requests carry the returned opaque `handle` and `revision`. An initial request without
stored state needs a UUIDv7 no more than 15 minutes old (one-minute future clock tolerance).
Persisted exact retries retain their original request ID. Verify the app-proxy
signature before using its `logged_in_customer_id`. Shopify strips `Cookie` and
`Set-Cookie` through app proxies; continuation secrets therefore remain in the app's
database, indexed by the opaque browser handle. Keep `SHOPIFY_API_SECRET` stable:
it derives retry-stable initial handles. Browser tokens stay in memory.

Before rollout, test the deployed CDN SDK against the app's memory-only session bridge;
its required token methods and v2 bearer support must match. Product views, successful
AJAX cart adds, and native VTO use a session-bound queue: at most 100 events, five delivery
attempts, and 24-hour retention. It cannot carry events across identity epochs.
Same-origin AJAX cart change/update/clear also capture removals when the snapshot is
unambiguous. Carts that bypass `window.fetch` need explicit hooks.

Deploy webhook subscriptions and authorize `read_orders`, `read_returns`, and
`read_customers`. The adapter sends order creation/cancellation, full-order payment,
processed-return quantities, and line refunds. `returns/close` is bookkeeping, not
another return. Individual partial payment captures are not covered.

`CommerceCollectionOutbox` freezes the canonical payload before the first HTTP attempt.
A bounded drain runs every 30 seconds while the app runs, beyond Shopify's webhook retry
window. Configuration problems pause delivery; `400`, `409`, `410`, and `422` retain
terminal rows for intervention. Monitor queue age and retain receipts under your privacy
policy. Failures before payload creation, such as initial Admin hydration failure, still
need Shopify retry or source reconciliation. Amount-only refunds use `refundAmount`.
Customer/shop/order erasure fences persist across reinstall and channel changes until an
approved operator reset; the separate 24-hour session replay markers do not expire them.

### WordPress / WooCommerce

In the plugin Credentials screen, retain `irisphera_channel_id`, register
`channel/<id>/anonymous` and `channel/<id>/customer` in Octopus, and save
`irisphera_channel_credential` with the required session/link/event grants. Registration
is an operator step, not automatic. Keep the merchant key for catalog operations.

The same-origin `get_irs_token` endpoint is POST-only and returns `no-store` responses.
Uncached storefront responses establish signed HttpOnly local epoch ownership before
browser capture and token requests. `irispheraSessionState` becomes channel-bound
session proof after Octopus session creation; local ownership alone and raw anonymous
cookies cannot authorize a link. Never cache or replay shopper `Set-Cookie` responses.
Optional tokens and browsing collection
require the WP Consent API `statistics` verdict or a CMP adapter through
`irisphera_optional_tracking_allowed` (default false). Commerce has a separate policy.

Configuration-blocked outbox events pause until credentials are updated; payload or
identity-link conflicts require manual repair. Existing legacy rows retain an explicit
v1 delivery path. Before rollout, define successful-row retention and remote identity
erasure procedures; the plugin's local eraser does not perform Octopus erasure.

### PrestaShop

Upgrade through the native module upgrader to apply the 1.7.2 migrations, including
the durable privacy-fence tables for existing 1.7.1 installations. Configure
`IRISPHERA_CHANNEL_ID` and `IRISPHERA_CHANNEL_API_KEY`, register the channel's
anonymous/customer domains and grants in Octopus, and keep the merchant catalog key
separate. Use a stable `IRISPHERA_STATE_SECRET` of at least 32 characters or preserve
PrestaShop's `_COOKIE_KEY_` fallback. Back up the key and database together. Key changes,
or retained fences without matching key-continuity metadata, stop collection rather
than silently bypass erasure. Restore a matching backup or migrate the retained
fences through an approved procedure; do not clear fences to resume delivery.

Optional collection denies access without a consent verdict. A configured consent module
can implement `irispheraConsentAllows(purpose, context)`; only literal `true` grants the
purpose. The SDK/token flow requires both analytics and personalization permission.
Set `IRISPHERA_RETURN_STATE_IDS` to accepted physical-return states; the empty default
emits no physical returns. An RMA request alone is not a completed return.

Same-origin token POSTs require CSRF protection. Continuations stay in encrypted HttpOnly
state and tokens stay in memory. Native commerce hooks enqueue snapshots rather than
sending synchronously. Use the restricted back-office outbox/reconciliation pages for
blocked deliveries; never replay privacy-blocked or legacy rows as new v2 events.
Local erasure does not delete Octopus history; an approved remote-erasure procedure is
a production prerequisite. Native validation covers PrestaShop 8.2.8; test other versions
and the merchant's consent, return, and storefront flows before activation.

## Errors and retries

Use the HTTP status as the control signal. Most errors use `application/problem+json`;
some image/validation errors use `{"detail":[...]}`. Log `requestId` or `X-Request-Id`
without credentials or identity payloads.

Persist each event's UUID, `occurredAt`, subject-at-capture, and complete payload before
sending. Retry the same source event ID and unchanged content; never rebuild a retry from
current catalog/customer state or replay an old browser event under a different identity.
V2 event PUTs return `201` for durable acceptance, `200` for an identical replay, and `409`
for conflicting content under the same merchant/channel/source-event key.

For session creation and refresh, use a UUID `Idempotency-Key` per logical operation.
Persist it with the request before sending. Exact retries reuse the active session and
response; a changed request under the same key conflicts. Use a new key for a later
refresh or new session, not for a retry. Do not replay a session after logout to restore it.

| Response | Action |
| --- | --- |
| Network failure, `429`, `5xx` | Use bounded attempts per run, backoff, jitter, and `Retry-After`; durable commerce outboxes retain future retries. Preserve key and payload. |
| `401` / `403` | Revalidate session, credential, scopes, and domain grants. Do not fall back to legacy or change the event owner. |
| `409` | Investigate the existing operation or identity link; do not hide a conflict with a new random ID. |
| `410` | Stop delivery for the erased identity; do not recreate or reactivate it. |
| `400` / `422` | Fix validation or transition errors; retain server events for review rather than repeatedly resending them. |
| `402` on legacy merchant routes | Check subscription quota. |

A successful data-collection response is not a catalog processing receipt or a complete
financial reconciliation. Keep authoritative commerce records and monitor your outbox;
bounded browser retries are best-effort, not a durable commerce delivery mechanism.

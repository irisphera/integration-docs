# 2. Create the merchant organization

Previous: [Prepare](01-prepare.md) · [Integration guide](../README.md) · Next: [Ingest products](03-ingest-products.md)

**Goal:** create a merchant organization with your integrator key, keep its ID and merchant API key, set how its storefront experience behaves, and resolve its collection context.

A merchant organization is one store: it owns a catalog, its shoppers' sessions and privacy choices, its events and its reports. Your integrator key manages the organizations you create. Everything after this step uses the organization's own merchant key.

## Create the organization

You choose the merchant key; this endpoint does not generate it. In production, generate it and store it in your secret manager. For the demo:

```bash
MERCHANT_API_KEY=$(python3 -c 'import secrets; print(secrets.token_urlsafe(32))')
jq -n --arg name "Enterprise demo $RUN_ID" --arg key "$MERCHANT_API_KEY" '{
  name: $name,
  apiKey: $key,
  flowConfig: {
    apparel: "ALL",
    recommendationCriteria: "ALL",
    profileWizard: "MANNEQUIN",
    recommendationTopK: 50
  }
}' > merchant-request.json
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/integrator/v1/merchant" \
  -H "INTEGRATOR-API-KEY: $INTEGRATOR_API_KEY" \
  -H 'Content-Type: application/json' \
  --data-binary @merchant-request.json -o merchant.json
MERCHANT_ID=$(jq -er '.id' merchant.json)
MERCHANT_API_KEY=$(jq -er '.apiKey' merchant.json)
jq '{id,name,flowConfig}' merchant.json
```

**Expected:** HTTP `201` with a `Location` header. The organization has an `id`, your demo name and the flow settings. The saved response also contains `apiKey` and `themeConfig`, so treat the file as secret.

| Field | Values | Meaning |
| --- | --- | --- |
| `name` | Required | Unique among your organizations |
| `apiKey` | Required | The merchant key your backend will send as `MERCHANT-API-KEY` |
| `flowConfig.apparel` | `MENSWEAR`, `WOMENSWEAR`, `ALL` (default) | Which apparel the storefront experience offers |
| `flowConfig.recommendationCriteria` | `NONE`, `PALETTE`, `SILHOUETTE`, `SIZING`, `ALL` (default) | Which analyses drive recommendations |
| `flowConfig.profileWizard` | `SIMPLE`, `MANNEQUIN` (default) | How the storefront collects the shopper profile |
| `flowConfig.recommendationTopK` | Integer, default `50` | How many recommended products the storefront experience shows |
| `themeConfig.themeFile` | Agreed with Irisphera | Storefront theme; omit it to keep the default |

`flowConfig` and `themeConfig` are storefront settings. Irisphera stores them and returns them to your storefront with the storefront configuration below and with the shopper token details in [step 4](04-shopper-session.md#check-the-session).

The call is idempotent in a way that matters for retries:

- A key that already belongs to one of your organizations returns that organization unchanged.
- A name that already belongs to one of your organizations updates that organization in place.
- A name or key owned by another integrator returns `403`.

Every successful branch returns `201`. For a transport retry, resend **the saved request**, never a new key or name. The unique demo name keeps you from updating an earlier demo.

## Manage your organizations

| Task | Request |
| --- | --- |
| List your organizations | `GET /integrator/v1/merchant`. Not paginated; `X-Total-Count` gives the number. The response includes API keys. |
| Read one organization | `GET /integrator/v1/merchant/{merchantId}` |
| Change settings or rotate the merchant key | `PUT /integrator/v1/merchant/{merchantId}` with `name` and `apiKey`. Omitted `themeConfig` or `flowConfig` values keep their current settings. |
| Recover a lost merchant key | `GET /integrator/v1/merchant/{merchantId}/apikey` returns `{merchantId, apiKey}` |
| Remove an organization | `DELETE /integrator/v1/merchant/{merchantId}`. See [step 9](09-privacy-requests-and-offboarding.md#remove-a-merchant-organization). |

You can only read or change organizations you own; any other ID returns `403`. Never log, cache or send merchant keys to a browser.

The storefront reads its configuration with the merchant key. It needs no shopper session, so call it from your backend:

```bash
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/merchant/v1/storefront/configuration" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" | jq .
```

**Expected:** `orgName`, `themeFile`, `apparel`, `profileWizard` and `recommendationCriteria`, plus `isVtoEnabled` and `isApsEnabled`. These two say whether quota is currently available for one try-on and one recommendation. Use them to decide whether to show the try-on button and the recommendation entry point.

## Resolve the collection context

The collection context tells you where this organization's shopper data goes and under which privacy terms. On first use, Irisphera registers one default channel and two identity namespaces for the organization. Later calls return the same context.

```bash
curl --fail-with-body -sS -X POST "$IRISPHERA_BASE_URL/merchant/v2/collection-context" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" -o collection-context.json
CHANNEL_ID=$(jq -er '.channelId' collection-context.json)
ANONYMOUS_NAMESPACE=$(jq -er '.anonymousNamespace' collection-context.json)
CUSTOMER_NAMESPACE=$(jq -er '.namespace' collection-context.json)
jq '{channelId,collectionMode,availableNoticeVersion,purchaseAttributionDisclosure,
  commerce:.businessStatistics.commerce.status,counters:.businessStatistics.counters.status}' \
  collection-context.json
```

**Expected:** HTTP `200` and `collectionMode` `v2`.

| Field | Use |
| --- | --- |
| `channelId` | Send it in every session and event, and as `X-Irisphera-Channel-Id` where a step shows it |
| `anonymousNamespace` | Namespace for anonymous browser identities |
| `namespace` | Namespace for signed-in customer identities |
| `availableNoticeVersion` | The privacy notice version a shopper acknowledges in [step 4](04-shopper-session.md#record-the-shoppers-privacy-choices). Read it here; do not hardcode it. |
| `purchaseAttributionDisclosure` | When present, the purchase-attribution terms your notice must disclose: `policyVersion`, `rollingWindowSeconds`, `productScope` `SAME_SKU`, `touchRule` `LAST_SUCCESSFUL` and `requiredNoticeVersion` |
| `businessStatistics` | Which daily business statistics this channel accepts; see [step 7](07-collect-events.md#daily-business-statistics) |

Use the returned namespaces; do not invent them. Collection is always v2: there is no legacy or dual collection, and no fallback when v2 refuses something. The context does not record consent and does not change quota.

To reach an existing channel of this organization, send its ID as `X-Irisphera-Channel-Id` with the merchant key. The header selects the channel; it does not authenticate. Irisphera never creates, moves or reactivates a channel from it. A disabled or erased organization returns `403`. A `503` is temporary: retry later.

## Platform plugins

For WordPress/WooCommerce and PrestaShop, Irisphera is the integrator and gives the merchant their key. Merchants enter only that key in the installed plugin, and the plugin resolves the context itself. No integrator key belongs in either plugin. The Shopify app's hosted backend uses an integrator key to manage its merchants.

**Checkpoint:** `MERCHANT_ID`, `MERCHANT_API_KEY`, `CHANNEL_ID`, `ANONYMOUS_NAMESPACE` and `CUSTOMER_NAMESPACE` are set and `collectionMode` is `v2`. Continue to [step 3](03-ingest-products.md).

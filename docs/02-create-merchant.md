# 2. Create the merchant organization

Previous: [Prepare](01-prepare.md) · [Integration guide](../README.md) · Next: [Import products](03-ingest-products.md)

**Goal:** create a merchant organization with your integrator key, keep its ID and merchant API key, choose how its storefront experience behaves, and get its collection context.

A merchant organization is one store. It owns a catalog, its shoppers' sessions and privacy choices, its events and its reports. Your integrator key creates and manages the organizations. From the end of this step on, every request uses the organization's own merchant key instead.

## Create the organization

You choose the merchant key. This endpoint does not generate one. In production, generate the key and keep it in your secret manager. For the demo, generate it in the terminal:

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

**Expected:** HTTP `201` with a `Location` header. The organization has an `id`, your demo name and the flow settings. The saved response also contains `apiKey` and `themeConfig`, so treat `merchant.json` as a secret.

| Field | Values | Meaning |
| --- | --- | --- |
| `name` | Required | Unique among your organizations |
| `apiKey` | Required | The merchant key your backend will send as `MERCHANT-API-KEY` |
| `flowConfig.apparel` | `MENSWEAR`, `WOMENSWEAR`, `ALL` (default) | Which apparel the storefront experience offers |
| `flowConfig.recommendationCriteria` | `NONE`, `PALETTE`, `SILHOUETTE`, `SIZING`, `ALL` (default) | Which analyses drive recommendations |
| `flowConfig.profileWizard` | `SIMPLE`, `MANNEQUIN` (default) | How the storefront collects the shopper profile |
| `flowConfig.recommendationTopK` | Integer, default `50` | How many recommended products the storefront experience shows |
| `themeConfig.themeFile` | Agreed with Irisphera | Storefront theme. Leave it out to keep the default. |

Irisphera only stores `flowConfig` and `themeConfig` and hands them back to your storefront: in the [storefront configuration](#read-the-storefront-configuration) below, and with the shopper token details in [step 4](04-shopper-session.md#check-the-session).

### Retrying the request

Sending the same request again never creates a second organization:

- A key that already belongs to one of your organizations returns that organization unchanged.
- A name that already belongs to one of your organizations updates that organization.
- A name or key that belongs to another integrator returns `403`.

Every successful case returns `201`. If the connection drops, resend **the saved `merchant-request.json`**, never a new key or name. The demo name includes `RUN_ID`, so you cannot update an earlier demo by accident.

## Manage your organizations

| Task | Request |
| --- | --- |
| List your organizations | `GET /integrator/v1/merchant`. Not paginated; `X-Total-Count` gives the number. The response includes API keys. |
| Read one organization | `GET /integrator/v1/merchant/{merchantId}` |
| Change settings or rotate the merchant key | `PUT /integrator/v1/merchant/{merchantId}` with `name` and `apiKey`. `themeConfig` and `flowConfig` values that you leave out keep their current settings. |
| Recover a lost merchant key | `GET /integrator/v1/merchant/{merchantId}/apikey` returns `{merchantId, apiKey}` |
| Remove an organization | `DELETE /integrator/v1/merchant/{merchantId}`. See [step 10](10-privacy-requests-and-offboarding.md#remove-a-merchant-organization). |

You can only read or change organizations you own. Any other ID returns `403`. Never log or cache merchant keys, and never send them to a browser.

## Read the storefront configuration

Your storefront needs the organization's settings to decide what to show. It reads them with the merchant key and needs no shopper session, so call this route from your backend:

```bash
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/merchant/v1/storefront/configuration" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" | jq .
```

**Expected:** `orgName`, `themeFile`, `apparel`, `profileWizard` and `recommendationCriteria`, plus two switches:

| Field | Meaning | Use it to |
| --- | --- | --- |
| `isVtoEnabled` | Quota is available for at least one try-on | Show or hide the try-on button |
| `isApsEnabled` | Quota is available for at least one recommendation | Show or hide the recommendation and mix-and-match entry points |

## Get the collection context

Before you open shopper sessions, ask Irisphera where this organization's shopper data goes. The answer is the collection context. "Collection" here means collecting shopper data. It has nothing to do with product collections.

The context gives you three things you need later:

- a **channel**, the storefront that sessions and events belong to;
- two **namespaces**, one for anonymous browser IDs and one for signed-in customer IDs;
- the **privacy notice version** that shoppers acknowledge.

The first call registers one default channel and the two namespaces for the organization. Later calls return the same context.

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
| `businessStatistics` | Which daily business statistics this channel accepts. See [daily business statistics](business-statistics.md). |

Use the namespaces Irisphera returns. Do not invent your own. Collection is always v2: there is no legacy or dual collection, and no fallback when v2 refuses something. Getting the context does not record consent and does not change quota.

To reach another existing channel of this organization, send its ID as `X-Irisphera-Channel-Id` with the merchant key. The header only selects the channel. It does not authenticate, and Irisphera never creates, moves or reactivates a channel because of it.

| Answer | Meaning |
| --- | --- |
| `403` | The organization is disabled or erased |
| `503` | Temporary. Retry later. |

**Checkpoint:** `MERCHANT_ID`, `MERCHANT_API_KEY`, `CHANNEL_ID`, `ANONYMOUS_NAMESPACE` and `CUSTOMER_NAMESPACE` are set, and `collectionMode` is `v2`. Continue to [step 3](03-ingest-products.md).

## Frequently asked questions

### We run one store. Do we still need an integrator key?

Yes. It creates the store's merchant organization, and afterwards it is the only key that can change the organization, recover or rotate its merchant key, or remove it. Day-to-day calls use the merchant key. Keep the integrator key in your secret manager, away from the servers that handle storefront traffic.

### Should each brand or country be its own organization?

Use one organization for each store whose figures you want to see separately. A report always covers the whole organization. Its request has no channel or collection filter. Each organization also has its own catalog, shopper sessions and privacy choices, and none of them are shared with another organization.

### How do we rotate the merchant key?

Send `PUT /integrator/v1/merchant/{merchantId}` with the organization's current `name` and a new `apiKey`. The old key stops working once the update is saved, so switch the key in your backend at the same moment. Generate the new key as the walkthrough does, from 32 random bytes.

### How much quota does the organization have, and what uses it?

`GET /merchant/v1/subscription` shows the `limit` and `current` usage of each metered feature:

| Feature | What uses one unit |
| --- | --- |
| `shopper2dPreview` | A successful try-on |
| `shopperRecommendation` | A recommendation request |
| `featureDetection` | Nothing at the moment. Imports do not count against quota. |

Mix and match, readiness checks, the 3D preview, body measurements and color extraction use no quota. Usage never resets by itself. To change the limits, ask Irisphera. When a feature's quota is used up, `isVtoEnabled` or `isApsEnabled` turns `false`, and new shopper tokens no longer get the feature's scope ([step 4](04-shopper-session.md#check-the-session)).

### Can we get the collection context on every request?

You can. Later calls return the same context and change nothing. It is simpler to store `channelId` and the two namespaces with the organization, and to read the context again when you need the current `availableNoticeVersion` or `businessStatistics`.

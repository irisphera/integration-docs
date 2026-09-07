# 2. Create a merchant

Previous: [Prepare](01-prepare.md) · [Call agenda](../README.md) · Next: [Upload a feed](03-upload-feed.md)

**Goal:** create the enterprise's merchant account, retain its ID and merchant API key, and resolve its collection context.

## Create the merchant with your integrator key

The integrator supplies the merchant key; this endpoint does not generate it. In production, create and retain it in your secret manager. For the demo:

```bash
MERCHANT_API_KEY=$(python3 -c 'import secrets; print(secrets.token_urlsafe(32))')
jq -n --arg name "Enterprise demo $RUN_ID" --arg key "$MERCHANT_API_KEY" \
  '{name:$name, apiKey:$key}' > merchant-request.json
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/integrator/v1/merchant" \
  -H "INTEGRATOR-API-KEY: $INTEGRATOR_API_KEY" \
  -H 'Content-Type: application/json' \
  --data-binary @merchant-request.json -o merchant.json
MERCHANT_ID=$(jq -er '.id' merchant.json)
MERCHANT_API_KEY=$(jq -er '.apiKey' merchant.json)
jq '{id,name}' merchant.json
```

**Expected:** HTTP `201`; the displayed merchant has an `id` and your demo name. The private response also contains `apiKey`, `themeConfig`, and `flowConfig`.

For a transport retry, resend **the saved request**, not a newly generated key/name. A key already owned by your integrator returns its existing merchant; an existing name can update that merchant. The unique demo name avoids modifying an earlier demo.

## Resolve collection context with the merchant key

An existing merchant registration is sufficient. The backend initializes its channel and registered identity namespaces on first use; repeated calls return the same context. No additional credentials or manual channel registration are needed.

```bash
curl --fail-with-body -sS -X POST "$IRISPHERA_BASE_URL/merchant/v2/collection-context" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" -o collection-context.json
CHANNEL_ID=$(jq -er '.channelId' collection-context.json)
ANONYMOUS_NAMESPACE=$(jq -er '.anonymousNamespace' collection-context.json)
CUSTOMER_NAMESPACE=$(jq -er '.namespace' collection-context.json)
jq '{channelId,collectionMode,comparisonStartsAt,comparisonEndsAt}' collection-context.json
```

Use the returned namespaces rather than inventing them. Keep the context with captured work. For an existing historical channel, send its `X-Irisphera-Channel-Id` with the merchant key; Octopus checks ownership and active grants rather than moving the data to the default channel. The header is routing context, not authentication.

Collection mode starts as `dual` for one UTC calendar month, then returns to `legacy` pending comparison review. Both report families remain supported. Context initialization does not grant shopper consent or change merchant quotas.

For WordPress and PrestaShop, Irisphera acts as integrator and supplies the merchant key. Merchants enter only that key in the installed plugin; the plugin resolves context itself. No integrator key belongs in either plugin. Shopify's hosted backend uses an integrator key to manage its merchants.

**Checkpoint:** `MERCHANT_ID`, `MERCHANT_API_KEY`, and the returned collection context are available. Continue to [step 3](03-upload-feed.md).

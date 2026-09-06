# 2. Create a merchant

Previous: [Prepare](01-prepare.md) · [Call agenda](../README.md) · Next: [Upload a feed](03-upload-feed.md)

**Goal:** create the enterprise's merchant account, retain its ID and merchant API key, and arrange its storefront channel.

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

## Obtain the storefront channel from Irisphera

Give Irisphera `MERCHANT_ID` and the intended storefront. Irisphera must supply:

| Value | Purpose |
| --- | --- |
| Channel UUID | Identifies this merchant's storefront/integration channel |
| Channel API key | Server-side session creation and event delivery |
| Anonymous namespace | Registered identity domain for this storefront's anonymous visitors |
| Customer namespace | Registered identity domain for your verified customer IDs |

Ask for the channel credential and domain grants needed for `shopper:session`, `identity:link`, and `events:write`, and shopper access to the experience used in step 4. Confirm merchant quotas as part of this handoff. Do not invent namespace strings: both namespaces must be registered to this merchant and granted to the channel.

**There is no public channel-provisioning endpoint in the current contract.** This is an operator handoff, not another call using the integrator key. A separate identity-admin key is not needed for this demo's anonymous-to-customer login flow.

Enter the supplied values when available; you can upload the feed while the handoff is in progress:

```bash
read -rp 'Channel UUID: ' CHANNEL_ID
read -rsp 'Channel API key: ' CHANNEL_API_KEY; printf '\n'
read -rp 'Anonymous namespace: ' ANONYMOUS_NAMESPACE
read -rp 'Customer namespace: ' CUSTOMER_NAMESPACE
```

**Checkpoint:** `MERCHANT_ID` and `MERCHANT_API_KEY` are available; channel provisioning is complete before step 4. Continue to [step 3](03-upload-feed.md).

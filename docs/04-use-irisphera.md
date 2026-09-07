# 4. Let a shopper use Irisphera

Previous: [Upload a feed](03-upload-feed.md) · [Call agenda](../README.md) · Next: [Collect activity and orders](05-collect-data.md)

**Goal:** establish the shopper's identity, generate a real virtual try-on, and show the resulting image.

The experience API remains `/shopper/v1`; it accepts the v2 session bearer token. Do not change every API path to v2.

**Privacy precondition:** the terminal bypasses a storefront consent UI; it does not bypass privacy requirements. Complete the [purpose and permission checks](07-privacy-and-consent.md) before creating an optional tracking session, linking history or uploading a person's photo. A bearer scope or verified login proves authorization, not consent to analytics, recording or reuse. Explain that this route records its own VTO outcome; do not promise an analytics-free feature until core outcome processing and optional reporting are separated and approved.

## Start an anonymous session from your backend

In production, retain a random anonymous ID in server-controlled storefront-session state. It identifies this browsing epoch, not a device fingerprint or email address.

```bash
ANONYMOUS_ID=$(uuid)
SESSION_OPERATION_ID=$(uuid)
jq -n --arg channel "$CHANNEL_ID" --arg ns "$ANONYMOUS_NAMESPACE" \
  --arg anonymous "$ANONYMOUS_ID" \
  '{channelId:$channel,currentIdentity:{namespace:$ns,kind:"ANONYMOUS",externalId:$anonymous}}' \
  > anonymous-session-request.json
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/merchant/v2/shopper-sessions" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" -H 'Content-Type: application/json' \
  -H "X-Irisphera-Channel-Id: $CHANNEL_ID" \
  -H "Idempotency-Key: $SESSION_OPERATION_ID" \
  --data-binary @anonymous-session-request.json -o anonymous-session.json
ANONYMOUS_SESSION_ID=$(jq -er '.sessionId' anonymous-session.json)
ANONYMOUS_CONTINUATION=$(jq -er '.anonymousContinuation' anonymous-session.json)
```

**Expected:** `201`, with `linkStatus: NOT_REQUESTED`. Keep the continuation secret in your backend; it proves ownership of this anonymous session. Use the same idempotency key and saved body for a transport retry.

## Sign in and preserve the anonymous-to-customer link

Use the test customer's ID **after your backend verifies their platform login**. Never accept an arbitrary customer ID from browser input. Here the terminal stands in for that trusted backend:

```bash
read -rp 'Verified test customer ID: ' CUSTOMER_ID
LOGIN_OPERATION_ID=$(uuid)
LINK_OPERATION_ID=$(uuid)
jq -n --arg channel "$CHANNEL_ID" --arg ns "$CUSTOMER_NAMESPACE" \
  --arg customer "$CUSTOMER_ID" --arg link "$LINK_OPERATION_ID" \
  --arg session "$ANONYMOUS_SESSION_ID" --arg proof "$ANONYMOUS_CONTINUATION" \
  '{channelId:$channel,linkOperationId:$link,
    currentIdentity:{namespace:$ns,kind:"CUSTOMER",externalId:$customer},
    previousAnonymousSession:{sessionId:$session,anonymousContinuation:$proof}}' \
  > login-request.json
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/merchant/v2/shopper-sessions" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" -H 'Content-Type: application/json' \
  -H "X-Irisphera-Channel-Id: $CHANNEL_ID" \
  -H "Idempotency-Key: $LOGIN_OPERATION_ID" \
  --data-binary @login-request.json -o customer-session.json
ACTIVE_SESSION_ID=$(jq -er '.sessionId' customer-session.json)
ACCESS_TOKEN=$(jq -er '.accessToken' customer-session.json)
SHOPPER_ID=$(jq -er '.shopperId' customer-session.json)
IDENTITY_VERSION=$(jq -er '.identityVersion' customer-session.json)
jq '{sessionId,shopperId,linkStatus,expiresAt}' customer-session.json
```

**Expected:** the first transition returns `LINKED`. Use the new token; the previous anonymous session is superseded. Retrying this operation must retain both operation IDs and the original proof/body.

This demo signs in first so the VTO and order share one customer session. Although the experience URL remains v1, the v2 bearer supplies canonical shopper, channel, and session capture data. Do not change captured identities or times after login.

## Check the session and garment

```bash
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/shopper/v2/session" \
  -H "Authorization: Bearer $ACCESS_TOKEN" -o session-info.json
jq '{expiresAt,scopes}' session-info.json
jq -e '.scopes | index("shopper:vto") != null' session-info.json >/dev/null
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/shopper/v1/stylist-preview/$SKU" \
  -H "Authorization: Bearer $ACCESS_TOKEN" -o readiness.json
jq -r '.' readiness.json
```

Proceed only when the feature scope is present and readiness is `UPPER`, `LOWER`, or `FULL`. **Stop on `UNAVAILABLE`** and have Irisphera check catalog enrichment. Readiness does not validate the person's photo or guarantee provider success.

## Generate and display the try-on

This represents opening the product page and clicking its try-on action. Save the page-view time for the collection example in step 5.

```bash
PRODUCT_VIEWED_AT=$(now)
read -rp 'Absolute path to consenting participant JPEG/PNG: ' PHOTO_PATH
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/shopper/v1/stylist-preview" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  --form-string "skuCustomId=$SKU" -F "targetImage=@$PHOTO_PATH" -o try-on.json
python3 -c 'import base64,json; from pathlib import Path; data=json.loads(Path("try-on.json").read_text()); Path("try-on.webp").write_bytes(base64.b64decode(data["generatedImage"],validate=True))'
```

**Expected:** HTTP `200`, and `try-on.webp` contains the generated image. Open it with your image viewer and show the result. The API returns base64 WebP bytes in JSON, not an image URL. Optional `productPageUrl` and `featuredImage` can support your storefront UI.

The route records its own successful, failed, or bad-input VTO activity. **Do not additionally send `VIRTUAL_TRY_ON` for this same attempt** in step 5. With a v2 bearer, recording failures fail the operation rather than silently losing telemetry. On `420`/`422`, correct the photo; do not claim a successful VTO. Face blurring, if required, must happen before upload: `isFaceBlurred` only describes an already-blurred input.

### If the call outlasts the token

After your backend revalidates the same platform login, refresh this session and replace the browser token. This does not change the anonymous continuation or log in a different account:

```bash
REFRESH_OPERATION_ID=$(uuid)
curl --fail-with-body -sS -X POST \
  "$IRISPHERA_BASE_URL/merchant/v2/shopper-sessions/$ACTIVE_SESSION_ID/access-tokens" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" -H "Idempotency-Key: $REFRESH_OPERATION_ID" \
  -H "X-Irisphera-Channel-Id: $CHANNEL_ID" \
  -o refreshed-token.json
ACCESS_TOKEN=$(jq -er '.token' refreshed-token.json)
```

**Checkpoint:** a real try-on image was displayed, and `SHOPPER_ID` identifies the customer who will place the demo order. Continue to [step 5](05-collect-data.md).

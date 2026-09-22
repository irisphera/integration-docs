# 4. Open a shopper session

Previous: [Ingest products](03-ingest-products.md) · [Integration guide](../README.md) · Next: [Virtual try-on](05-virtual-try-on.md)

**Goal:** open a session for a shopper, record the shopper's privacy choices, sign the shopper in, and get a token that can use try-on and recommendations.

Your backend opens every session with the merchant key and passes only the 30-minute shopper token to the browser. The browser calls the `/shopper/v2/...` routes with `Authorization: Bearer <token>`.

A session belongs to a **subject**: an anonymous browser identity, or a signed-in customer. Privacy choices are stored per organization, channel and subject. A session and its scopes decide what the token *can* call; the privacy choices decide what Irisphera may *record*. A scope is never consent.

## Start an anonymous session

In production, keep a random anonymous ID in server-controlled storefront session state. It identifies one browsing period, not a device fingerprint or an email address. The Irisphera platform plugins keep it for at most 45 days.

```bash
ANONYMOUS_ID=$(uuid)
SESSION_OPERATION_ID=$(uuid)
jq -n --arg channel "$CHANNEL_ID" --arg ns "$ANONYMOUS_NAMESPACE" --arg anonymous "$ANONYMOUS_ID" \
  '{channelId:$channel,currentIdentity:{namespace:$ns,kind:"ANONYMOUS",externalId:$anonymous}}' \
  > anonymous-session-request.json
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/merchant/v2/shopper-sessions" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" -H 'Content-Type: application/json' \
  -H "X-Irisphera-Channel-Id: $CHANNEL_ID" \
  -H "Idempotency-Key: $SESSION_OPERATION_ID" \
  --data-binary @anonymous-session-request.json -o anonymous-session.json
ANONYMOUS_SESSION_ID=$(jq -er '.sessionId' anonymous-session.json)
ANONYMOUS_CONTINUATION=$(jq -er '.anonymousContinuation' anonymous-session.json)
ACCESS_TOKEN=$(jq -er '.accessToken' anonymous-session.json)
jq '{sessionId,shopperId,linkStatus,expiresAt}' anonymous-session.json
```

**Expected:** HTTP `201` and `linkStatus` `NOT_REQUESTED`. Keep `anonymousContinuation` in your backend, for example in an HttpOnly signed cookie or server session: it proves that this browser owns the anonymous session when the shopper signs in. Never put it in the cart or expose it to page JavaScript.

For a transport retry, resend the saved body with the same `Idempotency-Key`: the response is `200` with the same session. The same key with a different body returns `409 idempotency_key_reused`.

## Record the shopper's privacy choices

Show your privacy notice and ask for the shopper's choices **before** Irisphera records anything optional. The walkthrough stands in for a shopper who reads the notice and accepts analytics and personalization.

| Purpose | What it allows | Without it |
| --- | --- | --- |
| `analytics` | Storefront events, image shares, the try-on outcome record and purchase attribution | Event routes return `403`; try-on still works but its outcome is not recorded |
| `personalization` | With `analytics`, saved history for recommendations and the stored recommendation profile | Recommendations use only the measurements and colors sent in the request |
| `qaRecording` | Optional quality-assurance recording | Nothing is recorded for QA |

Read the current record first. A shopper with no record gets version `0` with every purpose denied:

```bash
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/shopper/v2/privacy/preferences" \
  -H "Authorization: Bearer $ACCESS_TOKEN" -o anonymous-preferences.json
jq '{version,analytics,personalization,qaRecording,availableNoticeVersion,purchaseAttributionDisclosure}' \
  anonymous-preferences.json
```

Display the notice version named in `availableNoticeVersion`. When `purchaseAttributionDisclosure` is present and its `requiredNoticeVersion` equals that notice, the notice must also explain purchase attribution. The disclosure gives the terms: a purchase can be linked to the same shopper's last successful try-on of the same SKU within `rollingWindowSeconds` before the purchase. Then record exactly what the shopper chose:

```bash
record_choices() {  # $1: current preferences file, $2: output prefix
  jq '{expectedVersion:.version, noticeVersion:.availableNoticeVersion,
       analytics:true, personalization:true, qaRecording:false}
      + (if .purchaseAttributionDisclosure != null
            and .purchaseAttributionDisclosure.requiredNoticeVersion == .availableNoticeVersion
         then {purchaseAttributionPolicyVersion:.purchaseAttributionDisclosure.policyVersion}
         else {} end)' "$1" > "$2-choices.json"
  curl --fail-with-body -sS -X PUT "$IRISPHERA_BASE_URL/shopper/v2/privacy/preferences" \
    -H "Authorization: Bearer $ACCESS_TOKEN" -H 'Content-Type: application/json' \
    --data-binary "@$2-choices.json" -o "$2-choices-receipt.json"
  jq '{version,noticeVersion,analytics,personalization,qaRecording,
       acknowledgedPurchaseAttributionPolicyVersion,expiresAt}' "$2-choices-receipt.json"
}
record_choices anonymous-preferences.json anonymous
```

**Expected:** HTTP `200`, a higher `version`, and `analytics` and `personalization` `true`.

Rules for a real consent banner:

- Send the notice version you actually displayed. Read it from `availableNoticeVersion`; never hardcode it.
- Send `purchaseAttributionPolicyVersion` only when you displayed that attribution disclosure. A grant of `analytics` under the attribution notice without it is refused.
- `expectedVersion` is the `version` you read. A `409` means the record changed, for example because the shopper withdrew in another tab: read it again and ask again. Never overwrite a newer withdrawal.
- A refusal is a valid answer: send `false` for each refused purpose. Withdrawal is the same call with `false`.
- `expiresAt` `null` means the choice has no scheduled expiry. It never grants anything by itself.

If your consent banner posts to your backend instead of calling Irisphera from the browser, relay the shopper's action with `GET`/`PUT /merchant/v2/shopper-sessions/{sessionId}/privacy-preferences` and the merchant key. The body and rules are the same. Relay only an action the shopper took, never a store default. After the browser session has ended, this route can only restrict purposes; it cannot grant them.

## Sign the shopper in

Use the test customer's ID **after your backend has verified the platform login**. Never accept a customer ID from browser input. Send the anonymous session's proof with the login. Irisphera ends the anonymous session and opens a customer session. It links the anonymous history to the customer only when both the anonymous shopper and the customer currently grant `analytics`:

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

**Expected:** HTTP `201` and a customer session.

| `linkStatus` | Meaning |
| --- | --- |
| `LINKED` | Both subjects grant `analytics`: the anonymous history now belongs to the customer |
| `NOT_REQUESTED` | At least one of them does not, for example a customer who has never recorded a choice. The anonymous history stays unlinked. |

Replace the browser token with the new one in both cases: the anonymous session has ended. A retry keeps both operation IDs, the same proof and the same body. A `409 identity_link_conflict` means this anonymous session already belongs to a different customer; nothing is moved. A shopper who never signs in keeps using the anonymous session.

To keep signed-in activity separate from anonymous browsing, as the Irisphera Shopify app does, omit `previousAnonymousSession` and `linkOperationId`, and end the anonymous session with `DELETE /merchant/v2/shopper-sessions/{sessionId}`.

Privacy choices belong to each subject, so check the customer's own record:

```bash
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/shopper/v2/privacy/preferences" \
  -H "Authorization: Bearer $ACCESS_TOKEN" -o customer-preferences.json
if jq -e '.version == 0' customer-preferences.json >/dev/null; then
  record_choices customer-preferences.json customer
else
  jq '{version,analytics,personalization,qaRecording}' customer-preferences.json
fi
```

With no customer record (version `0`), record the choice the shopper just made on your banner, under the same notice. If the customer already has a record, that record applies. To change it, show the banner again. Never replace a recorded refusal with a grant automatically.

A test customer signing in for the first time has no record, so this walkthrough gets `NOT_REQUESTED`. That does not affect the rest of the guide.

This walkthrough signs the shopper in before the try-on, so the try-on, the events and the order all belong to one customer. Do not change captured identities or times after login.

## Check the session

```bash
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/shopper/v2/session" \
  -H "Authorization: Bearer $ACCESS_TOKEN" -o session-info.json
jq '{channelId,expiresAt,scopes,identityVersion}' session-info.json
jq -e '.scopes | index("shopper:vto") != null and index("shopper:recommendations") != null' \
  session-info.json >/dev/null
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/shopper/v2/auth/access-token" \
  -H "Authorization: Bearer $ACCESS_TOKEN" | jq '{merchantName,enabledFeatures,flowConfig,expiresInSeconds}'
```

| Scope | Granted when | Routes |
| --- | --- | --- |
| `shopper:session` | Always | Session and token details |
| `shopper:events` | Always | Storefront events and image shares |
| `shopper:vto` | Try-on quota is available | Try-on and 3D preview |
| `shopper:recommendations` | Recommendation quota is available | Recommendations, body measurements, color extraction |

A route outside the token's scopes returns `403`. The shopper's privacy preferences accept any shopper token, so a shopper can always refuse or withdraw. `GET /shopper/v2/auth/access-token` is the storefront's bootstrap call: it returns `enabledFeatures` (`STYLIST_PREVIEW`, `SHOPPER_RECOMMENDATIONS`) and the organization's `themeConfig`, `flowConfig` and `sizingConfig`. `/shopper/v1` routes are not served.

## Keep the token fresh

The token lasts 30 minutes, and there is no browser refresh token. Before it expires, your backend revalidates the same platform login and asks for a new token for the same session:

```bash
REFRESH_OPERATION_ID=$(uuid)
curl --fail-with-body -sS -X POST \
  "$IRISPHERA_BASE_URL/merchant/v2/shopper-sessions/$ACTIVE_SESSION_ID/access-tokens" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" -H "Idempotency-Key: $REFRESH_OPERATION_ID" \
  -H "X-Irisphera-Channel-Id: $CHANNEL_ID" \
  -o refreshed-token.json
ACCESS_TOKEN=$(jq -er '.token' refreshed-token.json)
jq '{expiresAt, hasAttributionRef:(.attributionRef != null)}' refreshed-token.json
```

| Result | Action |
| --- | --- |
| `200` | Replace the browser token. If `attributionRef` is absent, delete any stored one: analytics is no longer granted. |
| `410 session_expired` | The session ended naturally. Open a new session. |
| `410 identity_erased` | The session was revoked, replaced or erased. Do not recreate it. |
| `422 invalid_identity_transition` | The signed-in account changed. Open a new session for the new account. |

`attributionRef` is an opaque cart correlation handle returned only while analytics is granted. You may keep it with the cart and send it in events; it is never proof of identity.

**Checkpoint:** `ACCESS_TOKEN` is a customer token with `shopper:vto` and `shopper:recommendations`, `SHOPPER_ID` and `IDENTITY_VERSION` are set, and the customer's privacy record grants `analytics` and `personalization`. Continue to [step 5](05-virtual-try-on.md).

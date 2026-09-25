# 4. Open a shopper session

Previous: [Import products](03-ingest-products.md) · [Integration guide](../README.md) · Next: [Virtual try-on](05-virtual-try-on.md)

**Goal:** open a session for a shopper, record the shopper's privacy choices, sign the shopper in, and get a token that can use try-on and recommendations.

## How a shopper session works

1. A visitor arrives. Your backend opens an **anonymous session** with the merchant key and passes the shopper token to the browser.
2. The browser shows your privacy notice and records the shopper's **privacy choices** with that token.
3. The shopper signs in. Your backend opens a **customer session**. If both the anonymous visitor and the customer allow analytics, the anonymous history moves to the customer.
4. The token lasts 30 minutes. Before it expires, your backend asks for a **fresh token** for the same session.

The browser only ever holds the 30-minute shopper token. It calls the `/shopper/v2/...` routes with `Authorization: Bearer <token>`.

Two separate things control what happens in a session:

- The token's **scopes** decide which routes it can call. Irisphera grants the try-on and recommendation scopes only while the organization has quota for them.
- The shopper's **privacy choices** decide what Irisphera may record. They are stored per organization, channel and subject. A subject is either an anonymous browser identity or a signed-in customer.

Having a scope never means the shopper consented to anything.

## Start an anonymous session

In production, create the anonymous ID as a new UUIDv7 and keep it in server-controlled storefront session state. It identifies one browsing period. It is not a device fingerprint or an email address. Give it a fixed lifetime from its first issue, such as 45 days, that visits do not extend ([browser identity](privacy-and-consent.md#browser-identity-is-not-consent)).

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

**Expected:** HTTP `201` and `linkStatus` `NOT_REQUESTED`.

The response holds three values you need:

| Value | What it is | Where it goes |
| --- | --- | --- |
| `accessToken` | The shopper token | The browser |
| `sessionId` | The anonymous session | Your backend, for the sign-in below |
| `anonymousContinuation` | A secret that proves this browser owns the anonymous session. You send it when the shopper signs in. | Your backend only, for example in an HttpOnly signed cookie or the server session. Never in the cart or in page JavaScript. |

Every new session request carries a new UUIDv7 as its `Idempotency-Key`. If the connection drops, resend the saved body with the same `Idempotency-Key`. You get `200` with the same session. The same key with a different body returns `409 idempotency_key_reused`.

## Record the shopper's privacy choices

Show your privacy notice and ask for the shopper's choices **before** Irisphera records anything optional. Here the walkthrough plays a shopper who reads the notice and accepts analytics and personalization.

| Purpose | What it allows | Without it |
| --- | --- | --- |
| `analytics` | Storefront events, image shares, the try-on outcome record and purchase attribution | Event routes return `403`. Try-on still works, but its outcome is not recorded. |
| `personalization` | With `analytics`, saved history for recommendations and the stored recommendation profile | Recommendations use only the measurements and colors sent in the request |
| `qaRecording` | Optional quality-assurance recording | Nothing is recorded for QA |

First read the current record. A shopper with no record gets version `0` with every purpose denied:

```bash
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/shopper/v2/privacy/preferences" \
  -H "Authorization: Bearer $ACCESS_TOKEN" -o anonymous-preferences.json
jq '{version,analytics,personalization,qaRecording,availableNoticeVersion,purchaseAttributionDisclosure}' \
  anonymous-preferences.json
```

Show the notice version named in `availableNoticeVersion`. If `purchaseAttributionDisclosure` is present and its `requiredNoticeVersion` equals that notice, the notice must also explain purchase attribution. The disclosure gives the terms: a purchase can be linked to the same shopper's last successful try-on of the same SKU within `rollingWindowSeconds` before the purchase.

Then record exactly what the shopper chose. The `record_choices` helper builds the answer from the record you just read, sends it, and prints the result. The walkthrough uses it again for the signed-in customer.

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

- Send the notice version you actually showed. Read it from `availableNoticeVersion`. Never hardcode it.
- Send `purchaseAttributionPolicyVersion` only when you showed that attribution disclosure. Irisphera refuses an `analytics` grant under the attribution notice without it.
- `expectedVersion` is the `version` you read. A `409` means the record changed in the meantime, for example because the shopper withdrew in another tab. Read it again and ask again. Never overwrite a newer withdrawal.
- A refusal is a valid answer. Send `false` for each refused purpose. A withdrawal is the same call with `false`.
- `expiresAt` `null` means the choice has no scheduled expiry. It never grants anything by itself.

If your consent banner posts to your backend instead of calling Irisphera from the browser, your backend relays the shopper's action with `GET`/`PUT /merchant/v2/shopper-sessions/{sessionId}/privacy-preferences` and the merchant key. The body and the rules are the same. Relay only an action the shopper took, never a store default. After the browser session has ended, this route can only turn purposes off.

## Sign the shopper in

Your backend verifies the platform login first, then opens a customer session with the customer's ID. That ID is a UUIDv7 that your platform stores on the customer record the first time the customer reaches Irisphera, and it never changes afterwards. Never accept a customer ID from browser input. The walkthrough creates one for its test customer.

Send the anonymous session's ID and its `anonymousContinuation` with the request. Irisphera then ends the anonymous session and opens a customer session. It moves the anonymous history to the customer only when both the anonymous shopper and the customer currently grant `analytics`.

```bash
CUSTOMER_ID=$(uuid)  # in production: the UUIDv7 stored on the verified customer's record
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
| `LINKED` | Both grant `analytics`. The anonymous history now belongs to the customer. |
| `NOT_REQUESTED` | At least one of them does not, for example a customer who has never recorded a choice. The anonymous history stays separate. |

In both cases, replace the browser's token with the new one: the anonymous session has ended.

- `linkOperationId` and the `Idempotency-Key` are both new UUIDv7s. On a retry, keep both, the same proof and the same body.
- A `409 identity_link_conflict` means this anonymous session already belongs to a different customer. Nothing is moved.
- A shopper who never signs in keeps using the anonymous session.
- To keep signed-in activity separate from anonymous browsing, leave out `previousAnonymousSession` and `linkOperationId`, and end the anonymous session with `DELETE /merchant/v2/shopper-sessions/{sessionId}`.

### Check the customer's own choices

Privacy choices belong to each subject, so the customer has a record of their own:

```bash
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/shopper/v2/privacy/preferences" \
  -H "Authorization: Bearer $ACCESS_TOKEN" -o customer-preferences.json
if jq -e '.version == 0' customer-preferences.json >/dev/null; then
  record_choices customer-preferences.json customer
else
  jq '{version,analytics,personalization,qaRecording}' customer-preferences.json
fi
```

- If the customer has no record (version `0`), record the choice the shopper just made on your banner, under the same notice. The command above does that.
- If the customer already has a record, that record applies. To change it, show the banner again. Never replace a recorded refusal with a grant automatically.

A test customer who signs in for the first time has no record, so this walkthrough gets `NOT_REQUESTED`. That does not affect the rest of the guide.

The walkthrough signs the shopper in before the try-on, so the try-on, the events and the order all belong to one customer. Do not change captured identities or times after login.

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
| `shopper:recommendations` | Recommendation quota is available | Recommendations, mix and match, body measurements, color extraction |

A route outside the token's scopes returns `403`. The privacy preference routes accept any shopper token, so a shopper can always refuse or withdraw.

`GET /shopper/v2/auth/access-token` is the call your storefront makes first. It returns `enabledFeatures` (`STYLIST_PREVIEW`, `SHOPPER_RECOMMENDATIONS`) and the organization's `themeConfig`, `flowConfig` and `sizingConfig`. `/shopper/v1` routes are not served.

## Keep the token fresh

The token lasts 30 minutes, and there is no refresh token in the browser. Before it expires, your backend checks the same platform login again and asks for a new token for the same session, with a new UUIDv7 as the `Idempotency-Key`:

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

| Result | What to do |
| --- | --- |
| `200` | Replace the browser's token. If `attributionRef` is absent, delete any stored one: analytics is no longer granted. |
| `410 session_expired` | The session ended normally. Open a new session. |
| `410 identity_erased` | The session was revoked, replaced or erased. Do not recreate it. |
| `422 invalid_identity_transition` | The signed-in account changed. Open a new session for the new account. |

`attributionRef` is an opaque handle that links a cart to the session. Irisphera returns it only while analytics is granted. You may keep it with the cart and send it in events. It never proves who the shopper is.

**Checkpoint:** `ACCESS_TOKEN` is a customer token with `shopper:vto` and `shopper:recommendations`, `SHOPPER_ID` and `IDENTITY_VERSION` are set, and the customer's privacy record grants `analytics` and `personalization`. Continue to [step 5](05-virtual-try-on.md).

## Frequently asked questions

### Do we open a session for every visitor?

Open one as soon as the storefront needs Irisphera for the visitor: to record privacy choices, to send events, or to offer try-on and recommendations. Opening a session uses no quota. The [privacy requirements](privacy-and-consent.md#record-choices-through-the-preference-api) ask you to record the analytics choice on ordinary page visits, not only when a shopper opens try-on, so in practice you open a session on the visitor's first page view.

### Why does the token not have `shopper:vto` or `shopper:recommendations`?

The organization has no quota left for that feature. Irisphera checks the quota each time it issues a token. Hide the feature, check the quota with `GET /merchant/v1/subscription` ([step 2](02-create-merchant.md#how-much-quota-does-the-organization-have-and-what-uses-it)), and ask Irisphera to raise it.

### What happens when a call uses an expired token?

It returns `401`. Your backend gets a new token for the same session, as in [keep the token fresh](#keep-the-token-fresh), and the browser repeats the call. Refresh a few minutes before `expiresAt` to avoid it.

### Do privacy choices follow the shopper to another device?

A signed-in customer's choices do, on the same channel. They belong to the customer ID, so every session of that customer finds them. An anonymous visitor's choices belong to one anonymous ID, so a new browser starts with no record.

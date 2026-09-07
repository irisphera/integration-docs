# 5. Collect activity and commerce facts

Previous: [Use Irisphera](04-use-irisphera.md) · [Call agenda](../README.md) · Next: [Download a report](06-download-report.md)

**Goal:** connect real storefront actions and authoritative commerce transitions to the same shopper and catalog SKU.

The terminal below demonstrates the requests your integration must send. In production, call them from the hooks in this table—not from a timer, page reload, or client-supplied order body.

**Production privacy gate:** apply the [approved purpose/consent requirements](07-privacy-and-consent.md) before capture and again before delivery, including retries and legacy/v2 comparison targets. The examples use an authorized test participant and test order. Neither endpoint acceptance nor report completeness grants permission to collect a real shopper's behavior. Unknown or declined optional permission must not become a legacy fallback.

| Trigger | What to send | Who sends it |
| --- | --- | --- |
| Product page displayed | `PRODUCT_VIEWED`, when permitted for the approved purpose | Shopper bearer; eligible visitors, whether or not they use Irisphera features |
| Cart mutation succeeds | `ADD_TO_CART` or `REMOVE_FROM_CART` for each affected catalog SKU | Shopper bearer |
| VTO attempt | Step 4's endpoint records its own success/failure/bad-input event | Irisphera; do not duplicate it |
| Platform accepts an order | `ORDER_CREATED`, all lines and charged amounts | Trusted backend |
| Payment captured | `PAYMENT_CAPTURED` | Trusted backend/payment hook |
| Cancellation confirmed | `ORDER_CANCELLED`, original order/line IDs and affected quantities | Trusted backend |
| Physical return confirmed | `ORDER_RETURNED`, original order/line IDs and affected quantities | Trusted backend |
| Refund confirmed | `REFUND`, refund ID, total and known line allocations | Trusted backend |

For the approved reporting population, collect permitted commerce facts consistently, including eligible orders from shoppers who never used Irisphera. Do not extend collection to refused or otherwise unauthorized processing merely to fill the comparison population. Document coverage and selection bias; a merchant's checkout/accounting duty does not automatically justify identified Irisphera analytics. These APIs record what happened elsewhere. They do not perform checkout, capture money, cancel orders, or refund a payment.

## Record the page view and cart addition

This helper only delivers an already-saved event. It never changes the ID or body on a retry:

```bash
send_observation() {
  curl --fail-with-body -sS -X PUT "$IRISPHERA_BASE_URL/shopper/v2/events/$1" \
    -H "Authorization: Bearer $ACCESS_TOKEN" -H 'Content-Type: application/json' \
    --data-binary "@$1.json" -o "$1-receipt.json" -w 'HTTP %{http_code}\n'
  jq '{receivedAt}' "$1-receipt.json"
}
VIEW_EVENT_ID=$(uuid)
jq -n --arg channel "$CHANNEL_ID" --arg at "$PRODUCT_VIEWED_AT" --arg sku "$SKU" \
  '{schemaVersion:2,channelId:$channel,type:"PRODUCT_VIEWED",occurredAt:$at,
    product:{skuCustomId:$sku}}' > "$VIEW_EVENT_ID.json"
send_observation "$VIEW_EVENT_ID"
```

The browser body contains **no shopper ID, customer alias, or authoritative order**. Identity comes from the bearer. In a real storefront, enqueue this event when the page is displayed; the demo retained that time in step 4.

Now add two units of the demonstrated garment to the test cart. After the cart operation succeeds:

```bash
ADD_EVENT_ID=$(uuid)
jq --arg at "$(now)" '.type="ADD_TO_CART" | .occurredAt=$at' \
  "$VIEW_EVENT_ID.json" > "$ADD_EVENT_ID.json"
send_observation "$ADD_EVENT_ID"
```

This is one cart observation; purchased quantities belong on the order. A real removal uses `REMOVE_FROM_CART` with its own ID and occurrence time. A displayed 3D preview uses `TD_PREVIEW`; checking availability is not a display. Do not invent these actions for the demo. The standard VTO route owns its outcome event; only a custom integration without that auto-recording route sends `VIRTUAL_TRY_ON` after its own successful attempt.

## Record the accepted order

The demo order contains two units at EUR 99 each. Amounts are **totals for the line**, not unit prices: gross EUR 198, net EUR 165, tax EUR 33. The original line ID remains `line-1` through payment, return, and refund.

```bash
send_commerce() {
  curl --fail-with-body -sS -X PUT "$IRISPHERA_BASE_URL/merchant/v2/commerce-events/$1" \
    -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" -H 'Content-Type: application/json' \
    -H "X-Irisphera-Channel-Id: $CHANNEL_ID" \
    --data-binary "@$1.json" -o "$1-receipt.json" -w 'HTTP %{http_code}\n'
  jq '{receivedAt}' "$1-receipt.json"
}
ORDER_ID="demo-order-$RUN_ID"
ORDER_EVENT_ID=$(uuid)
jq -n --arg channel "$CHANNEL_ID" --arg at "$(now)" \
  --arg shopper "$SHOPPER_ID" --argjson version "$IDENTITY_VERSION" \
  --arg order "$ORDER_ID" --arg sku "$SKU" \
  '{schemaVersion:2,channelId:$channel,type:"ORDER_CREATED",occurredAt:$at,
    subject:{shopperId:$shopper,identityVersion:$version},
    order:{sourceOrderId:$order,currency:"EUR",lines:[{
      sourceLineId:"line-1",skuCustomId:$sku,quantity:2,
      amounts:{currency:"EUR",merchandiseGrossAfterDiscount:"198.00",
        merchandiseNetAfterDiscount:"165.00",tax:"33.00",discount:"0.00"}}]}}' \
  > "$ORDER_EVENT_ID.json"
send_commerce "$ORDER_EVENT_ID"
```

In production, use the platform's real order/line IDs, quantity, currency, and line totals. Decimal amounts are strings with at most four fractional digits; gross must equal net plus tax. Do not use floating-point arithmetic or mix currencies. Keep a stable model-and-color `skuCustomId`; use distinct `sourceLineId` values for source variant lines.

`subject` above is the server-held identity returned in step 4, not an ID copied from a cart. Other orders can use exactly one of:

- `{"externalIdentity":{"namespace":"<registered customer namespace>","kind":"CUSTOMER","externalId":"<verified platform customer ID>"}}`, asserted by an authorized backend/domain grant.
- `{"orderGuest":{"sourceOrderId":"<the same order ID as order.sourceOrderId>"}}` for a genuinely unidentified guest order. Its first event also requires top-level `consentEvidence: {"sessionId":"<captured session UUID>","version":<acknowledged positive version>}`. Capture this evidence from authenticated backend state after the privacy API acknowledges analytics; retain it with the order. Missing, stale, wrong-channel or unacknowledged evidence cannot create a guest. This consent relationship does not join browsing identities.

An optional `attribution.attributionRef` may carry the opaque reference returned by a session for cart correlation. It is never proof of customer identity, permission to link accounts, or an alternative to `subject`.

For a bound guest, later lifecycle events may omit `consentEvidence`; an exact replay must retain the original body. Supplied evidence cannot change the consent source. Current analytics permission, consent expiry, withdrawal, capture-time boundaries and erasure still apply after browser-session expiry. Send this field only for merchant commerce with `orderGuest`, never for other subjects or browser/interaction observations.

## Record payment, one returned unit, and its refund

Send each event only when its platform transition is confirmed. The following commands exercise those transitions in the **test** scenario:

```bash
PAYMENT_EVENT_ID=$(uuid)
jq --arg at "$(now)" '.type="PAYMENT_CAPTURED" | .occurredAt=$at
  | del(.order.lines[].amounts)' "$ORDER_EVENT_ID.json" > "$PAYMENT_EVENT_ID.json"
send_commerce "$PAYMENT_EVENT_ID"

RETURN_EVENT_ID=$(uuid)
jq --arg at "$(now)" '.type="ORDER_RETURNED" | .occurredAt=$at
  | .order.lines[0].quantity=1 | del(.order.lines[].amounts)' \
  "$ORDER_EVENT_ID.json" > "$RETURN_EVENT_ID.json"
send_commerce "$RETURN_EVENT_ID"

REFUND_EVENT_ID=$(uuid)
jq --arg at "$(now)" --arg refund "demo-refund-$RUN_ID" \
  '.type="REFUND" | .occurredAt=$at | .order.sourceRefundId=$refund
   | .order.refundAmount="99.00" | .order.lines[0].quantity=1
   | .order.lines[0].amounts={currency:"EUR",merchandiseGrossAfterDiscount:"99.00",
       merchandiseNetAfterDiscount:"82.50",tax:"16.50",discount:"0.00"}' \
  "$ORDER_EVENT_ID.json" > "$REFUND_EVENT_ID.json"
send_commerce "$REFUND_EVENT_ID"
```

A refund is not proof of a physical return. Send both only when both happened. `refundAmount` is the authoritative total, including any non-line adjustments; do not add line amounts to it again. If a refund has no known line allocation, use `lines: []` with `sourceRefundId` and `refundAmount` instead of inventing a SKU or quantity.

For a cancellation, create another immutable event with `type: ORDER_CANCELLED`, the original order/line IDs and the cancelled quantities; amounts may be omitted, as in the payment/return examples. **Do not manufacture a cancellation for the already-paid/returned demo scenario.** Preserve source sequence/version when the platform supplies one.

## Demonstrate safe replay

```bash
send_commerce "$ORDER_EVENT_ID"
```

**Expected:** first delivery of each event is `201`; this identical replay is `200` and does not create another order or purchased unit. A different body under the same event ID is `409`, not a correction. Retain each source event ID, occurrence time, subject, and payload together. In production, persist them in a durable outbox before delivery; derive stable commerce IDs from the source event/revision or persist the ID once. Browser observations use UUIDv7 IDs.

A `201` receipt proves durable acceptance, not that a contradictory lifecycle event was applied. Deliver original orders before dependent events when possible; out-of-order events may wait for their order, and inconsistent quantities/transitions are quarantined. Use actual source times, not retry times; future timestamps are rejected.

| Delivery result | Integration action |
| --- | --- |
| Timeout, `429`, `5xx` | Retry the same saved operation with backoff; honor `Retry-After` |
| `401` / `403` | Revalidate the session, credential and grants; do not fall back to legacy delivery or change the owner |
| `409` | Inspect the existing event/operation; never hide the conflict by generating a new ID |
| `400` / `422` | Correct the producer; retain failed server events for review |
| `410` | Stop delivery for the erased identity; do not recreate it |

Keep queued browser events bound to their capture session; never replay one shopper's pending activity under another shopper's token. For server-delivered observations, use `PUT /merchant/v2/interaction-events/{sourceEventId}` with the merchant key, captured `X-Irisphera-Channel-Id`, and a server-captured `subject`, not a newly minted browser bearer. This walkthrough uses v2 events. Production integrations retain independent legacy/v2 delivery outcomes during the comparison window; do not fall back or replay already accepted events when one target fails.

## End the shopper session

```bash
curl --fail-with-body -sS -X DELETE "$IRISPHERA_BASE_URL/shopper/v2/session" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
unset ACCESS_TOKEN ANONYMOUS_CONTINUATION
```

**Expected:** `204`. Clear browser tokens and server continuation/session state; a later visit starts a fresh anonymous epoch. Refresh requires a revalidated unchanged login; account switching requires a new session.

Session revocation is not consent withdrawal for every purpose or erasure of historical data. Stop affected optional capture and queues through the actual consent integration, and use the agreed [rights and deletion workflow](07-privacy-and-consent.md#data-requests-and-deletion) for retained data. Do not acknowledge erasure based only on this `204`.

**Checkpoint:** receipts exist for view, cart addition, accepted order, captured payment, one physical return, and refund. Order replay did not add another purchase. Continue to [step 6](06-download-report.md).

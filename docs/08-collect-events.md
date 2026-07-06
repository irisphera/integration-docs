# 8. Storefront and order events

Previous: [Mix and match](07-mix-and-match.md) · [Integration guide](../README.md) · Next: [Download the report](09-download-report.md)

**Goal:** send the product view, the cart addition and the order lifecycle of the demo purchase, so the report can count them and attribute the purchase to the try-on.

In this step you:

1. send the product view and the cart addition from the browser;
2. send the order, the payment, one returned unit and its refund from your backend;
3. send the order again, to see that a replay changes nothing.

There are two kinds of events:

- **Storefront events** describe what the shopper did in the browser. The browser sends them with the shopper token.
- **Order events** describe what the store platform confirmed: an accepted order, a captured payment, a cancellation, a return, a refund. Your backend sends them with the merchant key.

Send each event from the hook where it happens, never from a timer, a page reload or an order body supplied by the browser. These routes record what happened elsewhere. They do not run checkout, capture money, cancel orders or refund payments.

| Trigger | Event | Sent by |
| --- | --- | --- |
| Product page displayed | `PRODUCT_VIEWED` | Browser |
| Cart change succeeded | `ADD_TO_CART` or `REMOVE_FROM_CART` for each affected SKU | Browser |
| 3D model displayed | `TD_PREVIEW` | Browser |
| Try-on | Nothing. The try-on routes record their own outcome ([step 5](05-virtual-try-on.md#what-irisphera-records)). | Irisphera |
| Try-on image shared or saved | An image share ([step 5](05-virtual-try-on.md#record-an-image-share)) | Browser |
| Platform accepted an order | `ORDER_CREATED` with all lines and charged amounts | Backend |
| Payment captured | `PAYMENT_CAPTURED` | Backend |
| Cancellation confirmed | `ORDER_CANCELLED` with the original order and line IDs and the cancelled quantities | Backend |
| Physical return confirmed | `ORDER_RETURNED` with the original order and line IDs and the returned quantities | Backend |
| Refund confirmed | `REFUND` with the refund ID, the total and the known line allocations | Backend |

## Send only what the shopper allowed

Irisphera accepts an event only when the shopper granted `analytics` at the event's `occurredAt` and still grants it when the event arrives. This applies to every event on this page, to image shares and to replays.

- Check the shopper's choice before you capture an event and again before you send it, including retries from your queue.
- A `403` means the shopper did not allow it (or the credential lacks the scope). Drop the event. Do not retry it, do not send it under another shopper or subject, and do not turn it into daily business statistics.
- After a withdrawal, stop the affected queue. Replays of events captured before the withdrawal are refused too.
- Orders from shoppers who did not grant `analytics` are not sent as order events. They can count only through the [daily business statistics](business-statistics.md).

## Record the page view and cart addition

This helper sends an event that is already saved to a file. It never changes the ID or the body on a retry:

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

**Expected:** HTTP `201` with `receivedAt`. The view uses the time kept in [step 5](05-virtual-try-on.md#open-the-product-page), when the page was displayed.

The browser body contains **no shopper ID, customer ID or order**. Irisphera takes the shopper from the token. Browser events use UUIDv7 IDs that the browser keeps for retries.

Now add two units of the garment to the test cart. After the cart change succeeds:

```bash
ADD_EVENT_ID=$(uuid)
jq --arg at "$(now)" '.type="ADD_TO_CART" | .occurredAt=$at' \
  "$VIEW_EVENT_ID.json" > "$ADD_EVENT_ID.json"
send_observation "$ADD_EVENT_ID"
```

This is one cart event; the purchased quantity belongs on the order. A removal uses `REMOVE_FROM_CART` with its own ID and time. Do not invent events for the demo.

## Record the accepted order

The demo order has two units at EUR 99 each. Amounts are **totals for the line**, not unit prices: gross EUR 198, net EUR 165, tax EUR 33. The order and its line each get a UUIDv7. The line ID, `LINE_ID`, stays the same through payment, return and refund.

```bash
send_commerce() {
  curl --fail-with-body -sS -X PUT "$IRISPHERA_BASE_URL/merchant/v2/commerce-events/$1" \
    -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" -H 'Content-Type: application/json' \
    -H "X-Irisphera-Channel-Id: $CHANNEL_ID" \
    --data-binary "@$1.json" -o "$1-receipt.json" -w 'HTTP %{http_code}\n'
  jq '{receivedAt}' "$1-receipt.json"
}
ORDER_ID=$(uuid)
LINE_ID=$(uuid)
ORDER_EVENT_ID=$(uuid)
jq -n --arg channel "$CHANNEL_ID" --arg at "$(now)" \
  --arg shopper "$SHOPPER_ID" --argjson version "$IDENTITY_VERSION" \
  --arg order "$ORDER_ID" --arg line "$LINE_ID" --arg sku "$SKU" \
  '{schemaVersion:2,channelId:$channel,type:"ORDER_CREATED",occurredAt:$at,
    subject:{shopperId:$shopper,identityVersion:$version},
    order:{sourceOrderId:$order,currency:"EUR",lines:[{
      sourceLineId:$line,skuCustomId:$sku,quantity:2,
      amounts:{currency:"EUR",merchandiseGrossAfterDiscount:"198.00",
        merchandiseNetAfterDiscount:"165.00",tax:"33.00",discount:"0.00"}}]}}' \
  > "$ORDER_EVENT_ID.json"
send_commerce "$ORDER_EVENT_ID"
```

**Expected:** HTTP `201`. In production, send the UUIDv7s stored with the platform's order and its lines ([IDs](../README.md#ids)), and the order's real quantities, currency and line totals.

- Amounts are decimal strings with at most four decimal places. Gross must equal net plus tax. Do not use floating-point arithmetic or mix currencies.
- `skuCustomId` is the model-and-color SKU from [step 3](03-ingest-products.md). Give each size or variant line its own `sourceLineId`, a UUIDv7 stored with that line.
- The order time must not be in the future. Use the platform's time, not the time of a retry.

### Choose the order's subject

`subject` says whose order it is. It comes from your backend's state, never from the cart or the browser. Use exactly one of:

| Subject | Use it for |
| --- | --- |
| `{"shopperId":"…","identityVersion":…}` | The shopper of an Irisphera session, as returned in [step 4](04-shopper-session.md#sign-the-shopper-in). The demo uses this. |
| `{"externalIdentity":{"namespace":"<customer namespace>","kind":"CUSTOMER","externalId":"<customer ID>"}}` | A signed-in customer, identified by the customer's UUIDv7 ID after your backend verified the login |
| `{"orderGuest":{"sourceOrderId":"<the order ID>"}}` | A guest order with no customer account |

A guest order's first event also needs `consentEvidence: {"sessionId":"<session UUID>","version":<preference version>}`: the session in which the guest granted `analytics`, and the positive preference version that Irisphera acknowledged. Capture both in your backend when the guest checks out, and keep them with the order. Missing, stale or unacknowledged evidence is refused. Later events for the same guest order may omit it. Send `consentEvidence` only with `orderGuest`.

`attribution.attributionRef` may carry the reference returned with the shopper token ([step 4](04-shopper-session.md#keep-the-token-fresh)), to correlate the cart. It is never proof of identity and never replaces `subject`.

## Record payment, one returned unit and its refund

Send each event only when the platform confirms it. The demo stands in for those confirmations:

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
jq --arg at "$(now)" --arg refund "$(uuid)" \
  '.type="REFUND" | .occurredAt=$at | .order.sourceRefundId=$refund
   | .order.refundAmount="99.00" | .order.lines[0].quantity=1
   | .order.lines[0].amounts={currency:"EUR",merchandiseGrossAfterDiscount:"99.00",
       merchandiseNetAfterDiscount:"82.50",tax:"16.50",discount:"0.00"}' \
  "$ORDER_EVENT_ID.json" > "$REFUND_EVENT_ID.json"
send_commerce "$REFUND_EVENT_ID"
```

**Expected:** HTTP `201` for each.

- A refund does not prove a physical return. Send both only when both happened.
- `refundAmount` is the refund total, including adjustments outside the lines. Do not add the line amounts to it again. If the refund has no known line allocation, send `lines: []` with `sourceRefundId` and `refundAmount`; do not invent a SKU or quantity.
- A cancellation is another event with `type: ORDER_CANCELLED`, the original order and line IDs and the cancelled quantities. Amounts may be omitted. Do not send one for the demo order: it was paid and returned, not cancelled.

## Replay an event safely

Send the order again:

```bash
send_commerce "$ORDER_EVENT_ID"
```

**Expected:** HTTP `200`. An identical replay creates no second order and no extra units. The same ID with a different body returns `409`: it is not a correction.

In production, save each event's ID, time, subject and body in a durable outbox before the first delivery, and resend exactly that on retry. The event gets its UUIDv7 when you save it there, never later.

A `201` means the event is stored. Events that arrive before their order wait for it; contradictory events, such as a return larger than the order, are set aside and not applied. Send the order first whenever you can.

| Result | Action |
| --- | --- |
| Timeout, `429`, `5xx` | Retry the same saved event with backoff. Honor `Retry-After`. |
| `401` | Check the merchant key or refresh the shopper token, then retry the same event |
| `403` | Drop the event: the shopper did not grant `analytics` at `occurredAt`, or the credential lacks the scope. Do not retry it or send it elsewhere. |
| `409` | Inspect the event you already sent under that ID. Never hide the conflict with a new ID. |
| `400` / `422` | Fix the producer. Keep the failed event for review. |
| `410` | The shopper's data was erased. Stop sending for that shopper; do not recreate it. |

## Send storefront events from your backend

If the browser queues events for your backend to send later, your backend can send them without a shopper token: `PUT /merchant/v2/interaction-events/{sourceEventId}` with the merchant key, the captured `X-Irisphera-Channel-Id`, and the same body plus a `subject` captured in your backend when the event happened. Never replay one shopper's queued events under another shopper's token or subject.

## Daily business statistics

Order events only cover shoppers who granted `analytics`. For store-wide order totals, your source can also send daily business statistics: one snapshot per channel, dataset and UTC day, with no shopper, customer, session, order or line ID in it. `COMMERCE` statistics are available by default on every channel. The walkthrough sends none. [Daily business statistics](business-statistics.md) explains when you may send them and how to build them.

**Checkpoint:** receipts exist for the view, the cart addition, the order, the payment, the return and the refund, and the order replay returned `200`. Keep the shopper token for [step 10](10-privacy-requests-and-offboarding.md). Continue to [step 9](09-download-report.md).

## Frequently asked questions

### Do we send orders that had no try-on?

Yes. Send every order of a shopper who granted `analytics`, whether or not the shopper tried anything on. The report compares shoppers who tried products on with other active shoppers, so it needs the orders of both groups.

### Do we send events for shoppers who did not grant analytics?

No. Irisphera refuses them with `403` ([send only what the shopper allowed](#send-only-what-the-shopper-allowed)). Their orders can count only in the [daily business statistics](business-statistics.md).

### Which ID does an order event get?

A new UUIDv7, like every ID that you create for Irisphera ([IDs](../README.md#ids)). Generate it once, when the platform event reaches your outbox, and save it with the event. Every retry sends that saved ID and body. Never generate a new ID for a retry, because Irisphera treats a new ID as a new event. Browser events and image shares get their UUIDv7 in the browser, which keeps it for retries.

### Can we send several events in one request?

No. Each request carries one event. To catch up after an outage, send the saved events one at a time, oldest first, and send each order's `ORDER_CREATED` before its later events.

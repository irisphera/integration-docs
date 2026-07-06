# 8. Storefront and order events

Previous: [Mix and match](07-mix-and-match.md) · [Integration guide](../README.md) · Next: [Download the report](09-download-report.md)

**Goal:** send the product view, the cart addition and the order lifecycle of the demo purchase, so the report can count them and attribute the purchase to the try-on.

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
- Orders from shoppers who did not grant `analytics` are not sent as order events. They can count only through the [daily business statistics](#daily-business-statistics).

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

The demo order has two units at EUR 99 each. Amounts are **totals for the line**, not unit prices: gross EUR 198, net EUR 165, tax EUR 33. The line ID `line-1` stays the same through payment, return and refund.

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

**Expected:** HTTP `201`. In production, use the platform's real order and line IDs, quantities, currency and line totals.

- Amounts are decimal strings with at most four decimal places. Gross must equal net plus tax. Do not use floating-point arithmetic or mix currencies.
- `skuCustomId` is the model-and-color SKU from [step 3](03-ingest-products.md). Give each size or variant line its own `sourceLineId`.
- The order time must not be in the future. Use the platform's time, not the time of a retry.

### Choose the order's subject

`subject` says whose order it is. It comes from your backend's state, never from the cart or the browser. Use exactly one of:

| Subject | Use it for |
| --- | --- |
| `{"shopperId":"…","identityVersion":…}` | The shopper of an Irisphera session, as returned in [step 4](04-shopper-session.md#sign-the-shopper-in). The demo uses this. |
| `{"externalIdentity":{"namespace":"<customer namespace>","kind":"CUSTOMER","externalId":"<customer ID>"}}` | A signed-in customer, identified by the platform customer ID that your backend verified |
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
jq --arg at "$(now)" --arg refund "demo-refund-$RUN_ID" \
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

In production, save each event's ID, time, subject and body in a durable outbox before the first delivery, and resend exactly that on retry. Derive order event IDs from the platform's event and revision, or save the ID once.

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

Order events only cover shoppers who granted `analytics`. Daily business statistics give the merchant store-wide order totals without any shopper data: one snapshot per channel, dataset and UTC day, with no customer, shopper, session, order or line ID in it.

There are two datasets:

| Dataset | Content | Default |
| --- | --- | --- |
| `COMMERCE` | Orders, purchased units, physical returns, gross amounts per currency and units per SKU | `AVAILABLE` on every channel |
| `COUNTERS` | Counts of product requests, views, cart changes, try-on outcomes, 3D previews and recommendation requests | `NOT_APPROVED` unless Irisphera approves it for your channel |

### Check that the channel accepts them

Read `businessStatistics` from the collection context ([step 2](02-create-merchant.md#resolve-the-collection-context)):

```bash
jq '.businessStatistics.commerce | {status,businessPolicyVersion,sourceRecipeVersion,
  effectiveFrom,expiresAt,priorChoiceTreatment,aggregateRetentionDays}' collection-context.json
```

| `status` | Action |
| --- | --- |
| `AVAILABLE` | You may send snapshots for days between `effectiveFrom` and `expiresAt`, with the returned `businessPolicyVersion` and `sourceRecipeVersion` |
| `NOT_APPROVED` | Do not collect or send this dataset |
| `NOT_YET_EFFECTIVE` | Wait until `effectiveFrom`. Do not collect earlier data to send later. |
| `EXPIRED` | Stop collecting and sending. Corrections and deletions still work. |
| `RESTRICTED` | Stop collecting and sending, and follow the [correction workflow](privacy-and-consent.md#business-statistics-source-corrections) |

Read the values from the context every time; do not hardcode them. A missing `businessStatistics` field means not approved.

`sourceRecipeVersion` names the source implementation that the policy covers. The default `COMMERCE` policy covers the recipe of the Irisphera Shopify app, which counts orders from the platform's order webhooks and needs nothing in the browser. A custom integration sends `COMMERCE` snapshots only after Irisphera confirms that its source implements the returned recipe, or approves a recipe for it. Never send another recipe name, and never label your own source with the returned one.

### Respect the shopper's refusal

`priorChoiceTreatment` says how your storefront's earlier privacy choices affect the totals:

| Value | Source rule |
| --- | --- |
| `HONOR_BROAD_MEASUREMENT_REFUSAL` | Leave out the orders of shoppers who refused measurement on your storefront, for example by rejecting analytics in your consent banner. Keep that refusal with the cart and order, so a later order webhook can still find it. A shopper who never made a choice has not refused. |
| `OPTIONAL_LINKED_ANALYTICS_ONLY` | Your storefront only offered a refusal of the optional analytics. Such a refusal stops order events, not the daily totals. |

In both cases, honor a merchant business objection: a shopper who objects to the store's business measurement is left out from then on, whatever the dataset's status. See [privacy and consent](privacy-and-consent.md#business-statistics-and-the-shoppers-choice).

### Build the snapshot

A snapshot is a complete **replacement** for one UTC day, not an increment:

| Field | Rule |
| --- | --- |
| `schemaVersion` | `1` |
| `channelId` | The captured channel; must match `X-Irisphera-Channel-Id` |
| `businessPolicyVersion`, `sourceRecipeVersion` | The values returned for the channel |
| `revision` | Starts at `1`. Increase it only for a new replacement of the same day. |
| `coverage.status` | `PARTIAL` or `COMPLETE`. The current day is always partial. Mark a closed day complete only when your source covered the whole day. |
| `coverage.observedThrough` | How far the source covered the day, in UTC; not in the future |
| `commerce` or `counters` | Exactly one, matching the dataset in the path |

The `commerce` object:

| Field | Meaning |
| --- | --- |
| `orders`, `purchasedUnits` | Orders placed on this day and their units |
| `physicalReturnedUnits` | Units physically returned on this day, or `null` when your source cannot observe physical returns. A refund or restock is not a physical return. |
| `excludedRevenueOrders` | Orders without a complete value or currency |
| `unmappedPurchasedUnits`, `unmappedPhysicalReturnedUnits` | Units with no SKU. Keep them in the totals. The second is `null` when returns are not observed. |
| `byCurrency[]` | One row per currency: `currency`, `revenueEligibleOrders`, and `grossAmount` as a decimal string |
| `products[]` | One row per SKU: `sku`, `purchasedUnits` and `physicalReturnedUnits` (`null` when returns are not observed) |

- `revenueEligibleOrders` summed over the currencies, plus `excludedRevenueOrders`, equals `orders`.
- Gross is the merchandise value after discounts, with tax, without shipping and **without subtracting refunds**.
- Product units plus unmapped units equal the totals. Never invent a SKU for an unmapped unit.
- When returns are not observed, all return fields are `null`. Zero means observed and none.

The `counters` object has `events`, the overall `{eventType,count}` totals, and `products`, per-SKU `{sku,events}` rows. They are two views of the same counts: never add them together. Types are `PRODUCT_REQUEST`, `PRODUCT_VIEWED`, `ADD_TO_CART`, `REMOVE_FROM_CART`, `VTO_SUCCESS`, `VTO_FAILED`, `VTO_BAD_INPUT`, `TD_PREVIEW` and `RECOMMENDATION_REQUESTED`, limited to `allowedCounterTypes`. A server-side product request is `PRODUCT_REQUEST`, not `PRODUCT_VIEWED`. Counts are occurrences, never unique shoppers.

Only count orders placed from `effectiveFrom` on. Do not backfill older orders, and never turn a refused order event into a snapshot. Limits: 8 MiB per request, 64 currencies and 10,000 products per snapshot. Reject invalid source output instead of truncating it.

### Send the snapshot

Save the complete body and its revision in your outbox, then send it:

```bash
: "${BUSINESS_DATASET_KIND:?Set COMMERCE or COUNTERS from the saved snapshot}"
: "${BUSINESS_DATE:?Set the saved snapshot UTC date, YYYY-MM-DD}"
: "${BUSINESS_SNAPSHOT_FILE:?Set the saved snapshot file}"
curl --fail-with-body -sS -X PUT \
  "$IRISPHERA_BASE_URL/merchant/v2/business-statistics/$BUSINESS_DATASET_KIND/$BUSINESS_DATE" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" \
  -H "X-Irisphera-Channel-Id: $CHANNEL_ID" \
  -H 'Content-Type: application/json' \
  --data-binary "@$BUSINESS_SNAPSHOT_FILE" \
  -o business-statistics-response.json -w 'HTTP %{http_code}\n'
```

This command only sends a snapshot that your source already produced. The demo has none, so skip it unless you have one.

| Result | Meaning |
| --- | --- |
| `201` | First snapshot for the day accepted. The receipt has `datasetKind`, `date`, `revision` and `receivedAt`. |
| `200` | An identical replay, or a higher revision that replaces the day |
| `400` | Invalid body or coverage. Fix the source. |
| `403` | The channel, recipe or policy is not available. Stop sending. |
| `409` | A stale or conflicting revision, or a restricted or deleted day. Check your outbox; do not pick a new revision to get past it. |
| `413` | Over 8 MiB |
| `503` | Temporary. Retry the same saved snapshot. |

Use the channel saved with the snapshot, not the installation's current default. A correction is a complete snapshot with a higher revision. Keep your source able to rebuild a day for as long as Irisphera keeps it (`aggregateRetentionDays`, plus a short margin), so that a shopper's objection or erasure can be applied: see [corrections](privacy-and-consent.md#business-statistics-source-corrections).

**Checkpoint:** receipts exist for the view, the cart addition, the order, the payment, the return and the refund, and the order replay returned `200`. Keep the shopper token for [step 10](10-privacy-requests-and-offboarding.md). Continue to [step 9](09-download-report.md).

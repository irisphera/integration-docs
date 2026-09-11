# 5. Collect activity and commerce facts

Previous: [Use Irisphera](04-use-irisphera.md) · [Call agenda](../README.md) · Next: [Download a report](06-download-report.md)

**Goal:** connect real storefront actions and authoritative commerce transitions to the correct reporting source and catalog SKU.

The event walkthrough below uses the existing consented v2 path. It links eligible activity and orders to a shopper. The separate [business-statistics contract](#send-separately-approved-business-statistics) uses minimized daily snapshots without shopper identifiers. Its availability must be confirmed for your deployment; it does not change the event walkthrough's consent requirements.

In production, call the event APIs from the hooks in this table—not from a timer, page reload, or client-supplied order body.

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

For the consented reporting population, collect permitted commerce facts consistently, including eligible orders from shoppers who never used Irisphera. Do not extend this identified event path to refused processing merely to fill the comparison population. Document coverage and selection bias; a merchant's checkout/accounting duty does not automatically justify identified Irisphera analytics. These APIs record what happened elsewhere. They do not perform checkout, capture money, cancel orders, or refund a payment.

A separately approved business-measurement purpose can support broad merchant totals, including eligible new orders where optional linked analytics was refused. Use its own minimized source and snapshot contract. Do not fabricate a shopper, guest consent receipt or analytics grant, and do not reroute rejected event envelopes to that source. See [business-purpose approval and choices](07-privacy-and-consent.md#separately-approved-merchant-business-measurement).

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

## Send separately approved business statistics

**Contract preview:** use this path only after Irisphera confirms support in your deployment and provides the matching API schema and active merchant/channel/dataset policy. These instructions do not enable it. The existing consented event endpoints above remain separate.

A native merchant adapter collects new authoritative facts for the approved purpose and sends a fixed-UTC-day **replacement snapshot**:

```text
PUT /merchant/v2/business-statistics/{datasetKind}/{date}
MERCHANT-API-KEY: <server-held merchant key>
X-Irisphera-Channel-Id: <captured channel UUID>
Content-Type: application/json
```

`datasetKind` is `COMMERCE` or `COUNTERS` and appears **only in the path**. `date` is `YYYY-MM-DD` in UTC. Snapshot identity is merchant + channel + dataset + UTC day. A snapshot never contains a customer, shopper, session, order, line or attribution identifier. It also excludes photos, profiles, URLs, IP addresses, arbitrary metadata and cohort flags. A merchant credential authenticates the sender; it does not supply purpose approval.

### Build the snapshot from its approved source

Read `businessStatistics.commerce` or `businessStatistics.counters` in the merchant collection context. Require `status: AVAILABLE` for ordinary capture/delivery. Check the captured channel, `businessPolicyVersion`, `sourceRecipeVersion`, `effectiveFrom`, `expiresAt`, `sourceRetentionDays`, `aggregateRetentionDays`, allowed counter types and local restrictions **before capture and again before delivery**. An absent or unavailable policy means no new business capture or delivery. Do not derive approval from plugin installation, optional consent, a browser parameter or a locally invented reference. The business notice/policy version is separate from the optional-consent notice version.

`sourceRecipeVersion` identifies the adapter's versioned source implementation. Use the exact value documented by that adapter and authorized by the policy; it is not a generated legal approval. A recipe change must be reviewed before the policy permits it.

- **COMMERCE:** use native new-order and confirmed lifecycle hooks. Preserve accurate order/line associations locally for deduplication, corrections and rights. Do not retain the raw webhook as an analytics record. A payment capture, cancellation, refund or retry is not another order. Keep mandatory native merchant records under their own purposes.
- **COUNTERS:** use only the approved native server-request or feature-outcome recipe and allowed event types. Native server product requests use `PRODUCT_REQUEST`, not `PRODUCT_VIEWED`: a request count is not necessarily a displayed page view. Do not add a browser beacon, read an analytics identifier or reuse a session cookie under a commerce approval. If a platform cannot observe a permitted source, report it as unavailable, not zero.
- Start after the independent purpose, source and notice cutover. Do not backfill old orders, replay rejected consent events, or change original event times to make them eligible. A post-cutover return for an older order needs an approved return-date source; it does not backfill the old purchase or cohort.
- Apply business objections and applicable broad refusal promises at the source. Refusal of the separate optional linked-analytics purpose does not, by itself, veto an independently approved business purpose. See the [source correction workflow](07-privacy-and-consent.md#business-statistics-source-corrections).

### Request fields

The request body contains:

| Field | Meaning |
| --- | --- |
| `schemaVersion` | `1` for this snapshot contract; not the consented event schema version. |
| `channelId` | The captured, authorized channel; must match the header. |
| `businessPolicyVersion`, `sourceRecipeVersion` | Exact approved versions, each 1–100 characters; not consent evidence. |
| `revision` | Positive safe integer. Increase only for a new replacement of this snapshot. |
| `coverage.status` | `PARTIAL` or `COMPLETE`. |
| `coverage.observedThrough` | UTC source-coverage endpoint within the bucket, including its exclusive end, and not in the future. COMPLETE requires a closed day and coverage through its exclusive next midnight. |
| `commerce` | Required only for the COMMERCE path; do not send `counters` alongside it. |
| `counters` | Required only for the COUNTERS path; do not send `commerce` alongside it. |

A `commerce` object has these required fields:

| Field | Meaning |
| --- | --- |
| `orders`, `purchasedUnits` | Original Orders and Purchased Units in this day, including Orders excluded from money totals. |
| `physicalReturnedUnits` | Required: observed physical units returned on this day, or `null` if the source cannot observe physical returns. Not refund money or a purchase-cohort return count. |
| `excludedRevenueOrders` | Orders without a valid complete value/currency. |
| `unmappedPurchasedUnits`, `unmappedPhysicalReturnedUnits` | Required: units that cannot be assigned a valid SKU. Keep them in the corresponding observed all-product total. `unmappedPhysicalReturnedUnits` is `null` when physical returns are unavailable. |
| `byCurrency[]` | Unique uppercase three-letter `currency` rows with `revenueEligibleOrders` and `grossAmount` as a decimal string with at most four fractional places. |
| `products[]` | Unique `sku` rows with required `purchasedUnits` and `physicalReturnedUnits`; SKU length 1–255 characters. Product `physicalReturnedUnits` is `null` when physical returns are unavailable. |

Currency-row `revenueEligibleOrders` plus `excludedRevenueOrders` must equal `orders`. Zero eligible Orders requires zero gross. Gross means merchandise value after discounts, including tax, excluding shipping, **without subtracting refunds**. Do not mix currencies or use floating-point money arithmetic. Sum product Purchased Units plus `unmappedPurchasedUnits` to obtain total `purchasedUnits`. When physical returns are observed, product physical-return units plus `unmappedPhysicalReturnedUnits` must equal `physicalReturnedUnits`. Missing SKU mappings must not discard valid all-store order totals or be replaced with invented SKUs.

Keep return availability consistent throughout a source snapshot: if physical returns are unobservable, total, unmapped and product physical-return fields are all present as `null`. Do not omit them or substitute zero. Numeric zero requires an observed physical-return source with no returns. Native WooCommerce refunds or restocks alone do not prove a physical return; report those return fields as unavailable unless a genuine supported source supplies that evidence.

A `counters` object has two arrays:

- `events`: overall totals as unique `{eventType,count}` entries.
- `products`: unique `{sku,events:[{eventType,count}]}` rows; event types are unique within each SKU. Each type's product sum must not exceed its overall total.

Overall counter totals and product breakdowns are **different dimensions of the same observations**. Do not add them together. Types are `PRODUCT_REQUEST`, `PRODUCT_VIEWED`, `ADD_TO_CART`, `REMOVE_FROM_CART`, `VTO_SUCCESS`, `VTO_FAILED`, `VTO_BAD_INPUT`, `TD_PREVIEW` and `RECOMMENDATION_REQUESTED`; the source policy can allow only a subset. VTO attempts are the sum of the three outcome types, not a start plus an outcome. These counts are not distinct shoppers, sessions, visitor denominators or feature-user/non-user cohorts.

All non-null counts are nonnegative JSON-safe integers; revision starts at 1. Counts, revision and checked derived sums must not exceed `9007199254740991`. Each snapshot allows at most 64 currencies, 10,000 distinct product rows, nine event types per event array and an 8 MiB request body. Money must fit `NUMERIC(24,4)` and its four-decimal scale. Reject invalid or over-limit source output; never silently truncate it into a “complete” snapshot. Unknown/forbidden fields are not an extension mechanism.

The current day is partial. Mark a closed day complete only when the adapter has evidence of source coverage through the day end. The absence of webhook receipts or counter observations does not prove a complete zero day. Missing source coverage must remain visible as partial or unavailable.

### Persist and deliver replacements

Persist the complete body, revision, date, policy/recipe versions and captured channel before sending. Use a durable local outbox. A retry sends the **same saved body and revision**; a correction sends a complete newer replacement, not an additive delta. Serialize competing source updates so they cannot assign conflicting revisions. Never transfer an old operation to a new channel after reinstallation or account changes.

The following command sends a snapshot already produced by your approved native adapter. It does not create sample approval or consent evidence:

```bash
: "${BUSINESS_DATASET_KIND:?Set COMMERCE or COUNTERS from the saved operation}"
: "${BUSINESS_DATE:?Set the saved snapshot UTC date}"
: "${BUSINESS_SNAPSHOT_FILE:?Set the saved adapter snapshot file}"
curl --fail-with-body -sS -X PUT \
  "$IRISPHERA_BASE_URL/merchant/v2/business-statistics/$BUSINESS_DATASET_KIND/$BUSINESS_DATE" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" \
  -H "X-Irisphera-Channel-Id: $CHANNEL_ID" \
  -H 'Content-Type: application/json' \
  --data-binary "@$BUSINESS_SNAPSHOT_FILE" \
  -o business-statistics-response.json -w 'HTTP %{http_code}\n'
```

Use the saved operation's channel for `CHANNEL_ID`, not the installation's current default. With valid authorization and source eligibility, `201` accepts the first snapshot; `200` accepts an exact replay or a newer replacement. The receipt contains `datasetKind`, `date`, `revision` and `receivedAt`; an exact replay retains the stored receipt time. `409` includes stale revisions, same-revision conflicting contents and blocked/deleted buckets. Inspect the saved operation and source state; do not hide the conflict by reminting an arbitrary revision. Permission failures stop ordinary delivery; transient failures retry the same operation with backoff.

Retention runs even when approval expires or delivery stops. Keep the lawful source correction capability for the approved aggregate lifetime plus the configured reconciliation margin. [Source rights controls](07-privacy-and-consent.md#business-statistics-source-corrections) can restrict or permanently remove a bucket without reactivating collection. Removing a browser identity alone does not correct merchant totals.

## End the shopper session

```bash
curl --fail-with-body -sS -X DELETE "$IRISPHERA_BASE_URL/shopper/v2/session" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
unset ACCESS_TOKEN ANONYMOUS_CONTINUATION
```

**Expected:** `204`. Clear browser tokens and server continuation/session state; a later visit starts a fresh anonymous epoch. Refresh requires a revalidated unchanged login; account switching requires a new session.

Session revocation is not consent withdrawal for every purpose or erasure of historical data. Stop affected optional capture and queues through the actual consent integration, and use the agreed [rights and deletion workflow](07-privacy-and-consent.md#data-requests-and-deletion) for retained data. Do not acknowledge erasure based only on this `204`.

**Checkpoint:** receipts exist for view, cart addition, accepted order, captured payment, one physical return, and refund. Order replay did not add another purchase. Continue to [step 6](06-download-report.md).

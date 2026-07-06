# Daily business statistics

[Integration guide](../README.md) · Related: [Storefront and order events](08-collect-events.md) · [Privacy and consent](privacy-and-consent.md)

Order events ([step 8](08-collect-events.md)) only cover shoppers who granted `analytics`. Daily business statistics give the merchant store-wide order totals without any shopper data. Your source sends one snapshot per channel, dataset and UTC day. A snapshot holds no customer, shopper, session, order or line ID.

The walkthrough sends no snapshot. Use this page when you build the source that produces them. In short:

1. [Check that the channel accepts](#check-that-the-channel-accepts-them) the dataset, and read its policy values.
2. [Leave out the orders](#respect-the-shoppers-refusal) that the policy says to leave out.
3. [Build a complete snapshot](#build-the-snapshot) for each UTC day.
4. [Send it](#send-the-snapshot), and send a higher revision whenever the day changes.

There are two datasets:

| Dataset | Content | Default |
| --- | --- | --- |
| `COMMERCE` | Orders, purchased units, physical returns, gross amounts per currency and units per SKU | `AVAILABLE` on every channel |
| `COUNTERS` | Counts of product requests, views, cart changes, try-on outcomes, 3D previews and recommendation requests | `NOT_APPROVED` unless Irisphera approves it for your channel |

## Check that the channel accepts them

Read `businessStatistics` from the collection context ([step 2](02-create-merchant.md#get-the-collection-context)):

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

`sourceRecipeVersion` names the source implementation that the policy covers. The default `COMMERCE` policy names a recipe that Irisphera defined for its own sources, so it does not cover your source yet. Send `COMMERCE` snapshots only after Irisphera confirms that your source implements the returned recipe, or approves a recipe for it. Never send another recipe name, and never label your own source with the returned one unless Irisphera confirmed it.

## Respect the shopper's refusal

`priorChoiceTreatment` says how your storefront's earlier privacy choices affect the totals:

| Value | Source rule |
| --- | --- |
| `HONOR_BROAD_MEASUREMENT_REFUSAL` | Leave out the orders of shoppers who refused measurement on your storefront, for example by rejecting analytics in your consent banner. Keep that refusal with the cart and order, so a later order webhook can still find it. A shopper who never made a choice has not refused. |
| `OPTIONAL_LINKED_ANALYTICS_ONLY` | Your storefront only offered a refusal of the optional analytics. Such a refusal stops order events, not the daily totals. |

In both cases, honor a merchant business objection: a shopper who objects to the store's business measurement is left out from then on, whatever the dataset's status. See [privacy and consent](privacy-and-consent.md#business-statistics-and-the-shoppers-choice).

## Build the snapshot

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

## Send the snapshot

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

## Frequently asked questions

### Do we have to send daily business statistics?

No. They are optional, and the walkthrough sends none. Without them, the report shows `COMMERCE` as `UNAVAILABLE` in `merchantBusinessAnalytics`, and the event figures are unaffected.

### The status is `AVAILABLE`. Can we start sending?

Only once Irisphera has confirmed that your source implements the returned `sourceRecipeVersion`, or has approved a recipe for your source ([check that the channel accepts them](#check-that-the-channel-accepts-them)). `AVAILABLE` means that the channel accepts the dataset. It does not approve your source.

### Can we send a day's figures before the day ends?

Yes. Send the day as `PARTIAL`, with `coverage.observedThrough` set to how far your source got. Replace it with a higher `revision` as the day goes on, and send a `COMPLETE` snapshot once the day is over and your source covered all of it.


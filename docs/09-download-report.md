# 9. Download the report

Previous: [Storefront and order events](08-collect-events.md) · [Integration guide](../README.md) · Next: [Privacy requests and offboarding](10-privacy-requests-and-offboarding.md)

**Goal:** download the merchant's report, check that it counts exactly what the walkthrough did, and read how the purchase was attributed to the try-on.

## Request the report

The report uses the merchant key. The period is `[startTime, endTime)`: the start is included, the end is not. `zone` sets the calendar days used for daily rows. `REPORT_START` was set in [step 1](01-prepare.md); set the end after the last event was accepted:

```bash
REPORT_END=$(now)
jq -n --arg start "$REPORT_START" --arg end "$REPORT_END" \
  '{startTime:$start, endTime:$end, zone:"UTC"}' > report-request.json
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/merchant/v2/report" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" -H 'Content-Type: application/json' \
  --data-binary @report-request.json -o report.json
jq '{organizationName, reportingPeriod, collectionSource, resolutionMode,
     activity:[.timeseries[] | {date, numberOfProductViews, numberOfAddToCarts, numberOfOrders,
       numberOfReturns, numberOfVirtualTryOns, numberOfImageShares}]}' report.json
```

**Expected:** HTTP `200`, and `collectionSource` is `V2`. The route returns the report as JSON, not as a file, a job ID or a download link. Convert `report.json` in your own tools if you need another format.

## Reconcile the walkthrough

For a new organization and exactly the actions of steps 5 to 8, the report should show:

| Measure | Expected | Why |
| --- | --- | --- |
| Product views | 1 | One `PRODUCT_VIEWED` event |
| Cart additions | 1 | One cart event, whatever the quantity |
| Successful try-ons | 1 | Recorded by the try-on route itself |
| Image shares | 1 | One save to the device |
| Orders | 1 | One order. The replay and the payment add nothing. |
| Purchased units | 2 | The order line's quantity |
| Returns | 1 | One returned unit |
| Order value | EUR 198 | The line total, before the refund |

```bash
jq -e '
  .collectionSource == "V2"
  and .purchasedUnits.totalPurchasedUnits == 2
  and ([.timeseries[].numberOfProductViews] | add) == 1
  and ([.timeseries[].numberOfAddToCarts] | add) == 1
  and ([.timeseries[].numberOfVirtualTryOns] | add) == 1
  and ([.timeseries[].numberOfImageShares] | add) == 1
  and ([.timeseries[].numberOfOrders] | add) == 1
  and ([.timeseries[].numberOfReturns] | add) == 1
  and .imageShares.totalShares == 1
' report.json
```

**Expected:** `true`. A `false` is a reconciliation failure: check the saved receipts, the report period, and the channel and SKU of each event. Never send extra events to make the numbers match.

`numberOfVirtualTryOns` counts successful try-ons only. Failed and unusable-input try-ons are in `numberOfFailedVirtualTryOns` and `numberOfBadInputVirtualTryOns`, and `channelInsights.vto` gives the totals for the period.

## Read the attribution

`attributionPolicy` states the rule the report applied:

```bash
jq '.attributionPolicy' report.json
```

| Field | Value |
| --- | --- |
| `mode` | `PURCHASE_ANCHORED_ROLLING_WINDOW` |
| `anchor` | `PURCHASE`: every purchased unit has its own window, ending at the purchase |
| `lookbackDays` | The window length in days of 24 hours, set by Irisphera |
| `qualifyingInteraction` | `SUCCESSFUL_VIRTUAL_TRY_ON`. Failed and unusable-input try-ons never qualify. |
| `matchingRule` | `SAME_CHANNEL_SKU_OR_CANONICAL_PRODUCT`: the same shopper, and the same channel and SKU, or the same canonical product |
| `financialEffect` | Always `false`. Attribution is a measurement, not a fee or a billing basis. |

A unit bought at time `t` is attributed when the same shopper had a successful try-on of the same product between `t` minus `lookbackDays` and `t`, strictly before the purchase. The report reads try-ons from `lookbackDays` before `startTime`, so a purchase early in the period still sees its whole window. The try-on counts themselves stay within the period.

The walkthrough signed the shopper in before the try-on, and the order uses the same shopper, channel and SKU. Check the attributed values:

```bash
jq '{attributedUnits:.purchasedUnits.directlyAttributedPurchasedUnits,
     attributedRevenue:.purchasedUnits.directlyAttributedRevenueByCurrency,
     ordersWithAttributedUnit:[.averageOrderValue.byCurrency[]
       | {currency, cohort:.ordersContainingDirectlyAttributedUnit}],
     conversion:.conversion.irisphera}' report.json
```

**Expected:** 2 attributed units, EUR 198 attributed revenue, and in the EUR row of `averageOrderValue` 1 order containing an attributed unit, with a value and an average of EUR 198. `conversion.irisphera` shows 1 qualifying shopper who converted.

If the units are not attributed, the order's shopper or product differs from the try-on's. The purchase still counts, in the non-attributed figures.

- The return and the EUR 99 refund do not change the two purchased units or the EUR 198. Order values are gross, before refunds. The report is not a net-sales or refund ledger.
- Currencies are never added together.
- Attribution shows that a purchase followed a try-on. It does not prove that the try-on caused it.

## Other sections

| Section | Content |
| --- | --- |
| `conversion`, `purchaseRate` | Shoppers who converted after a successful try-on, compared with other active shoppers |
| `channelInsights.vto` | Try-on attempts by outcome, and the revenue credited to successful try-ons |
| `rankedProducts` | Products ranked by views, cart additions and removals, purchased units, returns and try-ons |
| `imageShares` | Shares by destination, by origin (`SAME_DEVICE` or `QR_CODE`) and by product. They never affect attribution. |
| `users`, `profileDistributions` | Palette and silhouette counts from recommendations. The step 6 profile appears here because the shopper granted `analytics` and `personalization`. |
| `resolutionMode`, `identityProjectionVersion` | How shoppers were grouped. `CURRENT` groups each event under the shopper's identity as it is linked now, so the figures can change after an account link. |

These figures cover only shoppers who granted `analytics`. They are not all visitors or all orders of the store. Do not fill missing activity with zero.

## Daily business statistics in the report

`merchantBusinessAnalytics` reports the [daily business statistics](business-statistics.md), separately from the figures above:

```bash
jq '.merchantBusinessAnalytics | {commerce:{status:.commerce.status, reasons:.commerce.reasons},
     counters:{status:.counters.status}}' report.json
```

**Expected** for the walkthrough: `commerce.status` is `UNAVAILABLE`, because no snapshot was sent, and `counters.status` is `NOT_APPROVED`.

| `status` | Meaning |
| --- | --- |
| `COMPLETE` | Every expected UTC day of every channel has a complete snapshot |
| `PARTIAL` | Some days are missing or partial. `reasons` and `coverage` say which. |
| `UNAVAILABLE` | A channel allows the dataset, but no usable snapshot covers the period |
| `NOT_APPROVED` | No channel of the organization currently allows the dataset |

Keep the two kinds of figures apart in exports and dashboards:

- **Never add business totals to the event figures.** The same order can be in both.
- Business totals use UTC days, whatever `zone` you requested. A complete day counts only when it lies wholly inside the period; a partial day counts only when the period reaches its `observedThrough`.
- `null` means unknown, never zero. For example, `physicalReturnedUnits` is `null` when the source cannot observe returns.
- `coverage` gives the days expected, received and complete for each channel. Call the totals store-wide only when every channel is `COMPLETE`.
- Business totals minus attributed orders is not a count of orders from shoppers who did not try on. Counters count requests and events, not visitors.

## Older periods

Detailed figures cover a limited window: by default the last 12 complete UTC months and the current month. `detailedAvailableFrom` gives the boundary, and detail before it is not returned.

| Field | Content |
| --- | --- |
| `historicalBusinessTotals` | Exact units, orders and gross value per currency for full UTC months before the boundary. `privacyAdjusted: true` means a withdrawal or erasure reduced the month. |
| `historicalStatistics` | Coarse ranges of shoppers per month, such as 40 to 59. `suppressed: true` means the ranges are withheld. |

A missing month is unknown, not zero. Do not add archived months to detailed figures, and do not add up monthly shopper ranges across months. A withdrawal or an erasure can reduce detailed figures for any period, so a report downloaded later can show less.

**Checkpoint:** the reconciliation check prints `true`, and the attributed values match. Continue to [step 10](10-privacy-requests-and-offboarding.md).

## Frequently asked questions

### Can we get a report for one channel or one collection?

No. The report always covers the whole organization. Its request takes only `startTime`, `endTime` and `zone`. Use separate organizations for stores whose figures you want to see apart ([step 2](02-create-merchant.md#should-each-brand-or-country-be-its-own-organization)).

### Why is a purchase not attributed to the try-on?

The purchase and the try-on did not meet the [attribution rule](#read-the-attribution). The usual causes:

- The order names another shopper than the try-on. For example, the shopper tried the product on anonymously, and the sign-in did not link the anonymous history (`NOT_REQUESTED` in [step 4](04-shopper-session.md#sign-the-shopper-in)).
- The order line has another SKU or channel than the try-on.
- The try-on failed, or Irisphera did not record it because the shopper had not granted `analytics` at that moment.
- The try-on was more than `lookbackDays` before the purchase, or after it.

### Why do the figures of a past period go down?

A withdrawal or an erasure can remove a shopper's activity from any period, including closed ones. Detail older than `detailedAvailableFrom` is also no longer returned ([older periods](#older-periods)). Download the report again when you need current figures, rather than adding up reports that you saved earlier.

# 6. Download and explain the report

Previous: [Collect activity and orders](05-collect-data.md) · [Call agenda](../README.md)

**Goal:** save the merchant's JSON report and reconcile it with the demo actions.

## Request the reporting period

Use the **merchant key**, not the integrator key or shopper token. `REPORT_START` was captured in step 1; capture the end after the last event was accepted. The period is `[startTime,endTime)`, with an inclusive start and exclusive end. `zone` controls calendar-day grouping.

This demo collects v2 activity, so request `/merchant/v2/report`. `/merchant/v1/report` remains supported for legacy collection; it is not an older output format for the same data. Download each source separately during the overlap period and check `collectionSource`. Do not add the two reports together: the same storefront action can be delivered to both collection routes.

```bash
REPORT_END=$(now)
jq -n --arg start "$REPORT_START" --arg end "$REPORT_END" \
  '{startTime:$start,endTime:$end,zone:"UTC"}' > report-request.json
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/merchant/v2/report" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" -H 'Content-Type: application/json' \
  --data-binary @report-request.json -o report.json
jq '{organizationName,reportingPeriod,collectionSource,resolutionMode,identityProjectionVersion,
  purchasedUnits,averageOrderValue,
  activity:[.timeseries[] | {date,numberOfProductViews,numberOfAddToCarts,
    numberOfOrders,numberOfReturns,numberOfVirtualTryOns}]}' report.json
```

**Expected:** HTTP `200`; `report.json` is the downloaded report. The endpoint returns JSON directly, not CSV, PDF, a background-job ID, or a download URL. Convert the saved JSON in your own reporting tool if another format is needed.

## Reconcile the demo

For a new merchant, one successful VTO, and exactly the events in step 5:

| Measure | Expected | Why |
| --- | --- | --- |
| Product views | 1 | One `PRODUCT_VIEWED` observation |
| Cart additions | 1 | One successful cart action, not two purchased units |
| Successful VTOs | 1 | The experience route recorded the attempt; no duplicate client event |
| Orders | 1 | One accepted order; replay and payment capture do not add orders |
| Purchased Units | 2 | Original line quantity is two |
| Physical returned units | 1 | `ORDER_RETURNED` references one unit of the original line |
| Whole-order purchased value | EUR 198 | Line gross is EUR 198, not EUR 198 multiplied by quantity |

```bash
jq -e '
  .purchasedUnits.totalPurchasedUnits == 2
  and ([.timeseries[].numberOfOrders] | add) == 1
  and ([.timeseries[].numberOfReturns] | add) == 1
  and ([.timeseries[].numberOfProductViews] | add) == 1
  and ([.timeseries[].numberOfAddToCarts] | add) == 1
  and ([.timeseries[].numberOfVirtualTryOns] | add) == 1
' report.json
```

A false result is a reconciliation failure, not permission to manufacture events. Inspect saved receipts, reporting period, captured channel/SKU, canonical mappings if used, and the backend collection/reporting logs.

### Explain attribution separately

Direct attribution requires the **same merchant, resolved shopper, and resolved product**, with successful VTO strictly before the purchase. Equal timestamps do not qualify. This demo signs in before VTO so both actions use the same customer.

With a v2 bearer, both the experience route and commerce record the channel and SKU. This demo uses the same `CHANNEL_ID` and `SKU`; no alias is needed for that shared channel/SKU attribution key. Different channel/SKU pairs match only when Irisphera resolves them to the same canonical product. Equal SKU text across different channels is not sufficient.

With all checkpoints satisfied, expect:

- `purchasedUnits.directlyAttributedPurchasedUnits`: **2**.
- `purchasedUnits.directlyAttributedRevenueByCurrency`: **EUR 198**.
- The EUR `averageOrderValue.byCurrency` row's `ordersContainingDirectlyAttributedUnit`: **1 order**, **EUR 198 purchasedUnitValue**, **EUR 198 averageOrderValue**.

If the captured or canonical product keys differ, the purchase still appears in the non-directly-attributed cohort. `resolutionMode` and `identityProjectionVersion` describe the identity snapshot applied to the report. `rankedProducts` exposes view, cart-add, cart-remove, purchase, physical-return, and VTO groups. Historical unstamped v1 activity remains legacy data; this cutover does not backfill its capture context.

The return and EUR 99 refund **do not change the original two Purchased Units or net EUR 99 out of gross purchase reporting**. Refunds and payment captures are recorded commerce facts, not additional purchase/return ranking events. This report is not a net-sales, cash-settlement, or refund-reconciliation ledger. Currencies remain separate; attribution is observational, not proof of incremental revenue caused by Irisphera. `users` profile distributions may be empty because this demo does not request recommendations.

Describe these detailed fields as the eligible **consented event population**, not all visitors or all merchant orders. Do not manufacture denied events or infer missing activity is zero. A separate business snapshot must not change their attribution or denominator. Reporting and identity resolution do not supply a legal basis for collection; see [legacy/v2 privacy requirements](07-privacy-and-consent.md#legacyv2-overlap-and-reports).

## Keep business totals separate from linked attribution

**Contract preview:** where your deployment confirms business-statistics support, v2 reporting adds `merchantBusinessAnalytics` for [separately approved daily snapshots](05-collect-data.md#send-separately-approved-business-statistics). This section does not imply that the purpose or dataset is enabled for your merchant.

The object identifies `metricDefinitionVersion`, `population`, `bucketZone` and `valueBasis`. Its separate `commerce` and `counters` sections contain `status`, `reasons`, `coverage`, `totals`, `daily` and `products`. Section status is `NOT_APPROVED`, `UNAVAILABLE`, `PARTIAL` or `COMPLETE`. Missing/unavailable totals are `null`, not manufactured zero. Coverage describes channel, source/policy versions, date/watermark limits and expected/received/complete day counts. These counts describe the approved source partitions, not a guessed number of store visitors.

Preserve these distinct populations in JSON exports, dashboards and CSV conversions:

| Report source | Population and use |
| --- | --- |
| Existing v2 detail: conversion, Purchased Units, AOV, daily activity and product rankings | Currently authorized consented event records. Detailed attribution qualifies successful VTO; an order without attributed units is not proof of a never-user order. |
| `merchantBusinessAnalytics` COMMERCE | Eligible merchant commerce from an independently approved source. Can include shoppers who refused separate optional linked analytics. Reports Orders, Purchased Units, physical-return units, gross amounts, order-value exclusions and available SKU quantities without shopper linkage. |
| `merchantBusinessAnalytics` COUNTERS | Approved source request/outcome counts. Not unique visitors, sessions, displayed pages by default, or a conversion denominator. |
| `historicalBusinessTotals` | Existing consent-gated monthly financial archive. This is not the independent business-statistics source. |
| `historicalStatistics` | Separately approved coarse monthly archive, not exact merchant commerce. |

Do not add business snapshots to consented event totals: the same order may appear in both. Keep the business dataset, channel, source/policy versions, UTC-day boundary, retention/cutover and coverage information visible. Replacements revise a day; they are not extra activity. These fixed UTC days do not acquire another timezone from the detailed report's `zone`.

A complete day requires verified full-day source coverage. Partial days, missing source coverage, unmapped SKUs, missing snapshots, expired policy, retention expiry and pending rights corrections can limit the report. Missing or unavailable is not zero. Only describe totals as whole-store coverage when every relevant channel and source period is actually covered. Do not label the sum of a partial channel set “all merchant orders”.

A complete UTC bucket contributes only when it lies wholly inside the requested half-open interval. A current-day partial snapshot can contribute through its watermark only when the interval includes the bucket start and extends through that watermark. Excluded edge days make coverage partial; they must not silently contribute out-of-period values. The existing detailed report can still succeed when the business section has no eligible buckets.

COMMERCE totals and daily entries preserve `orders`, `purchasedUnits`, `physicalReturnedUnits`, `excludedRevenueOrders`, `unmappedPurchasedUnits`, `unmappedPhysicalReturnedUnits` and `byCurrency`. Each currency row contains `currency`, `revenueEligibleOrders`, `grossAmount` and `averageOrderValue`. The server derives AOV from **unrounded gross / revenue-eligible Orders** in that currency, then rounds final presentation to two decimals using HALF_EVEN; undefined AOV is `null`. Never divide by all Orders or consented visitors. Keep value exclusions visible.

Gross purchase value is after discounts and includes tax; it excludes shipping and does not subtract refunds. Payment/refund events are not additional Orders; physical returns are a separate return-date measure. Product rows retain channel/SKU identity and available names. Product Purchased Units plus unmapped Purchased Units equal the all-product total. The corresponding equality applies to a source's physical returns only when that metric is observed. Keep unmapped units visible without discarding valid merchant totals or calling incomplete SKU coverage complete.

Physical-return fields are required but nullable. `null` means unavailable, not zero; native refund/restock data alone is not physical-return evidence. When every contributing source has unavailable returns, the reported return total is `null`. With mixed observed/unavailable sources, the report sums only observed returns and marks coverage partial with `PHYSICAL_RETURNS_UNAVAILABLE`. Label that number as an observed subtotal, not all-store returns. Preserve return nulls in exports and charts; never fill them with zero or derive a complete return rate from partial evidence.

COUNTERS totals and daily entries use `events:[{eventType,count}]`; product rows contain their separate event breakdown. Never add product subtotals to overall totals. `PRODUCT_REQUEST` is a distinct server-request measure, not an alias for `PRODUCT_VIEWED`.

Neither an exact small count nor the absence of a shopper ID proves anonymity. Treat business snapshots as protected statistics unless a deployment-specific assessment establishes otherwise. Preserve the applicable retention, access and rights controls; do not impose an arbitrary display threshold that hides lawful exact merchant totals.

A broad order total minus a partial consented VTO cohort is **not** a non-user cohort. Never divide all-store Orders by consented visitors, use request counts as unique visitors, or treat unknown feature status as no use. The independent snapshot contract carries no cohort marker. Its availability does not expand conversion coverage or establish causal uplift.

## Interpret older reporting periods

Detailed v2 reports retain the conversion, Purchased Unit, whole-Order AOV, currency revenue, daily activity, SKU-ranking and profile-count measures for records that remain available and authorized. A complete calendar month's attribution needs observations from that month and the preceding calendar month in the requested zone. Agree the necessary retention coverage before promising historical availability. Withdrawal, erasure or earlier expiry can change results; a missing source record cannot be reconstructed from an archive.

The default detail policy covers the latest 12 complete UTC months plus the current partial month; `detailedAvailableFrom` identifies its boundary. Older full UTC months appear separately in `historicalBusinessTotals`. Each entry contains exact `purchasedUnits`, `orders`, `revenueByCurrency`, `excludedRevenueOrders`, `sealedAt` and `privacyAdjusted`. Gross purchase value preserves whole-Order price validation and separate currencies; it is not net of refunds. Invalid-value Orders remain in purchase counts but are excluded from money totals.

These financial contributions remain pseudonymous personal data. Withdrawal or erasure can reduce totals; `privacyAdjusted:true` with zero does not mean the month originally had no activity. Missing months are unknown, not zero. Preserve both the archive and detail boundary in downloaded JSON, but never add an archived month to overlapping detail. Partial UTC months are omitted and late deliveries do not revise sealed months. Agree a separate justified retention/disposal schedule; do not promise indefinite retention. Legacy reports omit these fields.

V2 reports also include `historicalStatistics`, a supplementary array of sealed UTC calendar months, available only where Irisphera has enabled an approved archive for that merchant. Only months fully contained in the requested period appear, regardless of `zone`. An empty array can mean the archive is not enabled or no archived month is available; it never proves zero activity. Each month describes the authorized observations available when it was sealed; late delivery or earlier deletion can limit coverage.

The archive reports coarse distinct-shopper ranges, not exact events, Orders, Purchased Units or revenue. For example, `{lowerInclusive:40,upperExclusive:60}` means at least 40 and fewer than 60 shoppers. A month with `suppressed:true` withholds all counts; do not display them as zero. No shopper, order, SKU or profile detail is available from the archive.

The archive cannot replace exact monthly reports after source-data deletion. Do not add historical ranges to detailed totals: periods can overlap. Monthly distinct shoppers cannot be summed into cross-month distinct shoppers. Sealed months do not change after withdrawal, erasure, replay or identity linking; deleting the merchant removes its archive. Legacy reports omit this array. Confirm the applicable collection, report availability and retention arrangements before using historical reports.

## Close the call

Confirm that the enterprise can point to:

1. Its merchant ID and private credentials.
2. Its feed exporter and stable SKU mapping.
3. The generated try-on image.
4. Its storefront hooks, server-side order/lifecycle hooks, and durable retry storage.
5. The downloaded report and reconciled demo counts.
6. If business statistics is enabled: its active channel/dataset policies, native source adapter, complete/partial coverage evidence, source-rights correction path and separately labeled report population.

Before production, complete the [privacy acceptance scenarios](07-privacy-and-consent.md#acceptance-scenarios-before-enterprise-activation). Use the documented privacy-request workflow for shopper data requests; revoking a session or deleting local integration state does not erase server or provider data. Remove the private demo directory and photos according to the agreed retention policy; do not commit or screen-share its secret files. Confirm downstream request completion separately.

**Finish:** the enterprise has followed the API flow from integrator credentials through merchant-backed collection to a saved report, with reporting limits visible rather than hidden.

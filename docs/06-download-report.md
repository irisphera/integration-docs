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

Describe the report's permitted collection population and consent-related coverage limits. Do not describe an opt-in cohort as all visitors, manufacture denied events, or infer missing activity is zero. Reporting and identity resolution do not supply a legal basis for the underlying collection; see [legacy/v2 privacy requirements](07-privacy-and-consent.md#legacyv2-overlap-and-reports).

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

Before production, complete the [privacy acceptance scenarios](07-privacy-and-consent.md#acceptance-scenarios-before-enterprise-activation). Use the documented privacy-request workflow for shopper data requests; revoking a session or deleting local integration state does not erase server or provider data. Remove the private demo directory and photos according to the agreed retention policy; do not commit or screen-share its secret files. Confirm downstream request completion separately.

**Finish:** the enterprise has followed the API flow from integrator credentials through merchant-backed collection to a saved report, with reporting limits visible rather than hidden.

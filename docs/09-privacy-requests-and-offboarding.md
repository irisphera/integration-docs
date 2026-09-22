# 9. Privacy requests and offboarding

Previous: [Download the report](08-download-report.md) · [Integration guide](../README.md)

**Goal:** export the test shopper's data, end the shopper's session, erase the shopper, and learn how to remove a merchant organization.

A shopper asks the store for a copy or the erasure of their data. Your support process verifies the person, and your backend sends the request to Irisphera with the merchant key. Irisphera answers for the data it holds; your store still answers for its own systems. Irisphera acts on the subject your backend names, so never take the subject from browser input.

## Choose the subject

| Subject | Body | Use it for |
| --- | --- | --- |
| Customer | `{"externalIdentity":{"namespace":"<customer namespace>","kind":"CUSTOMER","externalId":"<customer ID>"}}` | A customer account, by the platform customer ID. The walkthrough uses this. |
| Anonymous browser | `{"externalIdentity":{"namespace":"<anonymous namespace>","kind":"ANONYMOUS","externalId":"<anonymous ID>"}}` | An anonymous ID that your backend still holds for the person |
| Irisphera shopper | `{"shopperId":"…","identityVersion":…}` | A shopper known by the `shopperId` and `identityVersion` of a session response. A `409` means the identity changed since, for example through an account link: use the version from a newer session. |
| Guest order | `{"orderGuest":{"sourceOrderId":"<order ID>"}}` | The shopper of a guest order sent with an `orderGuest` subject in [step 7](07-collect-events.md#choose-the-orders-subject) |

A customer, anonymous or shopper request covers the whole organization, including the identities linked to it, such as an anonymous history linked at sign-in. A guest-order request covers one order on one channel: send the channel captured with the order in `X-Irisphera-Channel-Id`, because the same order number can exist on another channel. The walkthrough sends the header on every privacy request; identity requests ignore it.

## Export the shopper's data

Save the request, with a new `requestId`, before you send it:

```bash
EXPORT_REQUEST_ID=$(uuid)
jq -n --arg id "$EXPORT_REQUEST_ID" --arg ns "$CUSTOMER_NAMESPACE" --arg customer "$CUSTOMER_ID" \
  '{requestId:$id, kind:"EXPORT",
    subject:{externalIdentity:{namespace:$ns, kind:"CUSTOMER", externalId:$customer}}}' \
  > export-request.json
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/merchant/v2/privacy/requests" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" -H 'Content-Type: application/json' \
  -H "X-Irisphera-Channel-Id: $CHANNEL_ID" \
  --data-binary @export-request.json -o export.json -w 'HTTP %{http_code}\n'
jq '{requestId, kind, status, pendingSystems, dataSections:(.data | keys | length)}' export.json
```

**Expected:** HTTP `202` and `status` `PENDING`. `data` holds what Irisphera's database keeps about the shopper, grouped by record type: sessions, privacy choices, events, orders and their evidence. `pendingSystems` lists the systems that Irisphera still has to check, such as `image-processors` for photos and generated images, `logs` and `backups`.

Check the request again later with its ID:

```bash
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/merchant/v2/privacy/requests/$EXPORT_REQUEST_ID" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" -H "X-Irisphera-Channel-Id: $CHANNEL_ID" \
  -o export-status.json
jq '{status, pendingSystems}' export-status.json
```

| `status` | Meaning |
| --- | --- |
| `PENDING` | Work remains in the systems listed in `pendingSystems`. Irisphera completes them; check again later, for example once a day. |
| `COMPLETED` | Every system is done, and `data` is final |
| `FAILED` | Irisphera could not complete the request. Contact Irisphera with the `requestId`. |

Rules for every privacy request:

- For a transport retry, resend the saved body: you get the same request with its current status. The same `requestId` with another body returns `409`.
- A `404` means Irisphera does not know the subject, for example an unregistered namespace or `shopperId`. An export of a customer ID that Irisphera never saw returns empty data.
- The export is personal data. Deliver it to the verified person over your support channel, together with your store's own data, and do not keep copies longer than your process needs.

## End the shopper's session

When the shopper signs out, end the Irisphera session. The browser ends its own session while its token is valid:

```bash
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' -X DELETE "$IRISPHERA_BASE_URL/shopper/v2/session" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

**Expected:** HTTP `204`, or `401` if the token has already expired or was revoked. Your backend can end any session of the organization with the merchant key, also after its token expired:

```bash
curl --fail-with-body -sS -o /dev/null -w 'HTTP %{http_code}\n' -X DELETE \
  "$IRISPHERA_BASE_URL/merchant/v2/shopper-sessions/$ACTIVE_SESSION_ID" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" -H "X-Irisphera-Channel-Id: $CHANNEL_ID"
unset ACCESS_TOKEN ANONYMOUS_CONTINUATION
```

**Expected:** HTTP `204`, also when the session had already ended. Remove the token from the browser and the continuation proof from your session store.

Ending a session only stops its token. It does not withdraw the shopper's privacy choices, which stay with the shopper for the next session, and it does not delete data. To withdraw, record `false` choices as in [step 4](04-shopper-session.md#record-the-shoppers-privacy-choices). To delete, send an erasure.

## Erase the shopper's data

At sign-in, the anonymous history was not linked to the customer (`NOT_REQUESTED` in [step 4](04-shopper-session.md#sign-the-shopper-in)), so it is a separate identity. Erase both identities. When the sign-in returned `LINKED`, the customer request covers the anonymous history as well.

```bash
erase_identity() {  # $1: namespace, $2: CUSTOMER or ANONYMOUS, $3: external ID, $4: file prefix
  jq -n --arg id "$(uuid)" --arg ns "$1" --arg kind "$2" --arg external "$3" \
    '{requestId:$id, kind:"ERASE", subject:{externalIdentity:{namespace:$ns, kind:$kind, externalId:$external}}}' \
    > "$4-request.json"
  curl --fail-with-body -sS "$IRISPHERA_BASE_URL/merchant/v2/privacy/requests" \
    -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" -H 'Content-Type: application/json' \
    -H "X-Irisphera-Channel-Id: $CHANNEL_ID" \
    --data-binary "@$4-request.json" -o "$4.json" -w 'HTTP %{http_code}\n'
  jq '{requestId, status, pendingSystems}' "$4.json"
}
erase_identity "$CUSTOMER_NAMESPACE" CUSTOMER "$CUSTOMER_ID" erase-customer
erase_identity "$ANONYMOUS_NAMESPACE" ANONYMOUS "$ANONYMOUS_ID" erase-anonymous
```

**Expected:** HTTP `202` and `status` `PENDING` for both. The request ID is in each saved request file. Check each request later as for the export.

What the erasure does at once:

- Irisphera deletes the shopper's sessions, privacy choices, events, try-on and recommendation records, image shares and orders from its database. When `pendingSystems` no longer lists `local-database`, this part is done. `legal-hold` means a legal hold delays it.
- Every session of the shopper ends. Its token stops working.
- Irisphera keeps only a keyed digest of the erased ID, so the ID cannot come back: a new session for it returns `410` with the problem type `identity_erased`. Handle this like an unavailable feature: the store keeps working without Irisphera features for that account. An erasure of an ID that Irisphera never saw also records the digest.
- Reports lose the shopper's activity, for every period.

Download the report of step 8 again and check that the walkthrough's activity is gone:

```bash
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/merchant/v2/report" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" -H 'Content-Type: application/json' \
  --data-binary @report-request.json -o report-after-erasure.json
jq '{activity:([.timeseries[] | .numberOfProductViews, .numberOfAddToCarts, .numberOfOrders,
       .numberOfReturns, .numberOfVirtualTryOns, .numberOfImageShares] | add // 0),
     purchasedUnits:.purchasedUnits.totalPurchasedUnits}' report-after-erasure.json
```

**Expected:** `activity` and `purchasedUnits` are `0`.

The erasure is complete only when `status` is `COMPLETED`. Until then, tell the person that the erasure is in progress, not done.

| Pending system | Work |
| --- | --- |
| `local-database` | Irisphera's database. `legal-hold` replaces it while a hold applies. |
| `image-processors` | Photos and generated images at Irisphera's processing providers |
| `qa-recordings`, `logs`, `backups`, `caches` | Irisphera's recordings, logs, backups and caches |
| `client-storage`, `delivery-queues` | Copies in browser storage and in delivery queues, including your integration's outbox. Delete the shopper's stored and queued items there. |
| `merchant-business-source` | The organization's daily business statistics. See the next section. |

## Correct daily business statistics

Daily snapshots contain no shopper IDs, so Irisphera cannot find a shopper's orders in them. After an erasure it holds back the business days that may contain them: for each channel with daily business statistics, from `effectiveFrom`, or from the start of the `aggregateRetentionDays` period if later, up to the day of the request. A customer, anonymous or shopper erasure holds every channel of the organization; a guest-order erasure holds only the order's channel. Reports leave the held days out, with the reason `BUSINESS_SOURCE_RECONCILIATION_PENDING`, and the request lists `merchant-business-source`, until the correction is done.

`COMMERCE` is available by default on every channel ([step 7](07-collect-events.md#daily-business-statistics)), so every erasure starts this correction. Your source adapter does it:

1. Exclude the shopper from your source: later snapshots leave the shopper's orders out. Discard queued snapshots built before the exclusion.
2. Look up the shopper's orders in your platform for the held period.
3. For each held day with a snapshot that contains them, rebuild the day without them and send it as a complete snapshot with a higher `revision`. If you cannot rebuild a day, delete it with `DELETE /merchant/v2/business-statistics/{datasetKind}/{date}`. When you cannot tell whether a day contains the shopper's orders, delete it.
4. Send Irisphera your result, with the `requestId`, through your agreed support contact: for each channel and dataset, the period checked, whether you found the shopper, and each held day's outcome (rebuilt, deleted, or no match) with its current revision. Irisphera then closes `merchant-business-source`.

The walkthrough sent no snapshot, so there is nothing to rebuild, but the task stays pending until your result reaches Irisphera. The routes and rules are in [business-statistics source corrections](privacy-and-consent.md#business-statistics-source-corrections).

## Remove a merchant organization

When a merchant leaves, the integrator removes its organization. Removal cannot be undone:

- The organization, its merchant key, catalog, sessions, shopper data and reports are deleted, and the merchant key stops working at once.
- Download the reports the merchant wants to keep, and send pending privacy requests, before you remove it.
- Keep your integrator key: you follow the removal with it. The merchant key stops working, so finish the business-statistics corrections of earlier erasures first.

Only remove the walkthrough's test organization if you no longer need it. Set `REMOVE_MERCHANT_ID`, for example to `$MERCHANT_ID`, then send the removal:

```bash
: "${REMOVE_MERCHANT_ID:?Set REMOVE_MERCHANT_ID to the organization to remove}"
curl --fail-with-body -sS -X DELETE "$IRISPHERA_BASE_URL/integrator/v1/merchant/$REMOVE_MERCHANT_ID" \
  -H "INTEGRATOR-API-KEY: $INTEGRATOR_API_KEY" -D removal-headers.txt -o /dev/null -w 'HTTP %{http_code}\n'
REMOVAL_STATUS_PATH=$(awk 'tolower($1) == "x-irisphera-privacy-status-url:" {print $2}' removal-headers.txt | tr -d '\r')
curl --fail-with-body -sS "$IRISPHERA_BASE_URL$REMOVAL_STATUS_PATH" \
  -H "INTEGRATOR-API-KEY: $INTEGRATOR_API_KEY" -o removal-status.json
jq '{requestId, status, pendingSystems}' removal-status.json
```

**Expected:** HTTP `202`, with the headers `X-Irisphera-Privacy-Request-Id` and `X-Irisphera-Privacy-Status-Url`, and a `PENDING` removal request. The status URL is a path on the same environment and needs the integrator key. It keeps working after the organization is gone. `403` means you do not own the organization; `404` means it does not exist or was already removed.

Besides the systems of a shopper erasure, the removal lists `merchant-assets`, the organization's catalog images and generated files at Irisphera, and usually `merchant-business-source`: stop the organization's business-statistics source, discard its queued snapshots and tell Irisphera. The removal is complete when `status` is `COMPLETED`.

## Clean up

The walkthrough files contain the test participant's photo results, measurements, export and API keys:

```bash
cd "$HOME"
rm -rf "$DEMO_DIR"
unset INTEGRATOR_API_KEY MERCHANT_API_KEY
```

Delete or return the participant's photo and selfie as you agreed with the participant, and close the terminal.

**Checkpoint:** the export returned the shopper's data, both erasures are accepted and tracked, the report no longer shows the walkthrough's activity, and you know which requests are still pending. The walkthrough is complete. Before real shoppers use the integration, go through the [privacy and consent requirements](privacy-and-consent.md) and their [acceptance scenarios](privacy-and-consent.md#acceptance-scenarios-before-enterprise-activation).

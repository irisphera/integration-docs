# Privacy and consent requirements

[Integration guide](../README.md)

The walkthrough follows these rules with one test shopper. This page covers what your storefront, consent banner, business-statistics source and support process must do before real shoppers use the integration. Confirm the deployed contract and the features enabled for your environment with Irisphera before integration testing.

## Before sending any real shopper data

- Agree with the merchant and Irisphera the permitted processing, each party's responsibilities, the applicable agreements, the privacy notice, recipients, retention and support contacts.
- Explain photo uploads and the server and provider processing before a shopper uses try-on or sizing. Test with consenting participants and test identities.
- Treat photos, generated images, measurements, colors, profiles, shopper identifiers and linked order history as personal data. Do not promise browser-only processing or immediate deletion.
- Keep the three optional purposes separate: behavioral analytics and attribution, saved personalization, and QA recording. The merchant's consent-management platform (CMP) can supply a valid, current analytics permission; do not ask for a second Irisphera opt-in for the same purpose. A merchant credential, a login, an order webhook or accepted terms is not permission.
- The merchant's notice must also describe the store's [daily business statistics](#business-statistics-and-the-shoppers-choice): their purpose, source, retention and how to object. They do not enable optional browser tracking, personalization, QA recording or identity linking.

## Shopper choices

| Purpose | What it covers | Without it |
| --- | --- | --- |
| `analytics` | Storefront and order events, image shares, try-on outcome records and purchase attribution | Events are refused; try-on and recommendations still work, and nothing optional is recorded |
| `personalization` | Saved recommendation history and profile, with `analytics` | Each recommendation uses only the request |
| `qaRecording` | Loading and running a session recorder | No recorder |

A try-on or a recommendation request does not grant any purpose. A refusal must not block checkout or unrelated store features. Keep a visible way to reopen the choices and withdraw them. Until a choice is known and acknowledged, keep optional capture, storage, recording and delivery off. A missing, malformed, expired or incompatible choice is not a grant.

### Record choices through the preference API

| Caller | Route | Credential |
| --- | --- | --- |
| Shopper's browser | `GET` / `PUT /shopper/v2/privacy/preferences` | Shopper token of the current session |
| Your backend, relaying the shopper's action | `GET` / `PUT /merchant/v2/shopper-sessions/{sessionId}/privacy-preferences` | Merchant key, server-side only. Send the session's captured `X-Irisphera-Channel-Id`. |

1. Read the current record. No record means version `0` with every purpose denied.
2. Display the notice named in `availableNoticeVersion`. When `purchaseAttributionDisclosure` is present and its `requiredNoticeVersion` equals that notice, the notice must explain purchase attribution with the disclosed terms.
3. Send the shopper's three choices with the notice version you displayed, the current version as `expectedVersion`, and `purchaseAttributionPolicyVersion` when you displayed the attribution disclosure. Never hardcode a notice version.
4. Start optional processing only after the acknowledgment: a positive version with the purpose `true`. `expiresAt` must be in the future when set; `null` means no scheduled expiry, never a grant by itself.
5. On `409`, read the record again and ask again. Never overwrite a newer withdrawal with an older choice.
6. Apply the merchant CMP's restrictions as well. Irisphera's acknowledgment does not override a refusal in the CMP.

The walkthrough shows these calls in [step 4](04-shopper-session.md#record-the-shoppers-privacy-choices). Derive the session from authenticated server state; never let the browser choose the customer or subject of a backend update. After the browser session has ended, the backend route can only restrict purposes; enabling one needs a live session.

On an ordinary page visit, read the merchant CMP and acknowledge an eligible analytics permission through this API; do not wait for the shopper to open try-on. Keep personalization and QA recording off unless the shopper chose them. If no supported CMP supplies permission, show your own consent dialog. Never replace a recorded refusal, a withdrawal, an expired choice or an identity reset with an automatic grant.

Shopper tokens last 30 minutes. Token expiry and session revocation do not end an acknowledged choice, and a choice does not keep a token alive.

### Browser identity is not consent

The Shopify, WordPress and PrestaShop integrations keep a new anonymous browser identity for a fixed 45 days from its first issue. Visits, token refreshes and consent acknowledgments do not extend it. A custom integration should use a comparable fixed limit.

- Keep only protected first-party storage or an opaque server-side handle. Never infer continuity from IP addresses, fingerprints or a customer ID supplied by the browser.
- Keep token lifetime, identity lifetime and consent separate. A surviving identity does not allow collection after consent expiry, withdrawal, a CMP refusal or an identity reset.
- On exactly `410` with the problem type `session_expired`, your backend may open a new session for the same identity after checking current permission. Never use an expired anonymous continuation as login proof. `identity_erased`, other `410` responses and `422` are not renewal signals.
- Keep the acknowledged session and version captured at checkout with the order ([step 8](08-collect-events.md#choose-the-orders-subject)). Never pair a new session ID with an older consent version.
- At the 45-day limit, stop capture and retries under that identity. A later identity is new; do not reconnect its predecessor's queue or history.

The 45-day limit is a technical limit. It is not a legal basis, a consent duration or a server-side retention schedule, and the merchant's notice must disclose the storage.

### Withdrawal, retries and identity changes

- Check permission before capture and again before delivery, including queue retries.
- On withdrawal, stop the affected producers, the recorder and queued sends, and delete local data that was kept only for the withdrawn purpose. Delayed responses must not restore a withdrawn permission or refill saved data.
- Propagate changes to the other tabs and iframes of the storefront. Validate message origins; never send tokens, photos or profiles to arbitrary origins.
- Check permission again after login, logout or an account switch. Never carry one shopper's choice or history over to another shopper.
- Keep withdrawal, session revocation and erasure as three separate actions.

## Business statistics and the shopper's choice

Daily business statistics ([step 8](08-collect-events.md#daily-business-statistics)) give the merchant store-wide totals without shopper IDs. They are a separate purpose from optional analytics:

- `COMMERCE` is `AVAILABLE` on every channel by default. The default policy covers the source recipe of the Irisphera Shopify app, which counts orders from the platform's order webhooks. A custom integration sends `COMMERCE` only after Irisphera confirms that its source implements the returned `sourceRecipeVersion`, or approves a recipe for it.
- `COUNTERS` needs a separate approval for the channel. A `COMMERCE` policy does not cover it.
- Read `businessStatistics` from the collection context each time. A missing field, `NOT_APPROVED`, `NOT_YET_EFFECTIVE`, `EXPIRED` or `RESTRICTED` means do not collect or send. Collect only from `effectiveFrom`: no backfill of older orders, and no reuse of events that the optional analytics route refused.
- The policy belongs to its channel and dataset. One installation's policy never covers another.

The shopper's choices apply as follows:

| Situation | Effect on business statistics |
| --- | --- |
| The shopper made no choice | No order events. The order still counts in the daily totals. |
| The shopper refused or withdrew analytics, and `priorChoiceTreatment` is `HONOR_BROAD_MEASUREMENT_REFUSAL` (the default policy) | No order events, and leave the shopper's orders out of the totals. Keep the refusal with the cart and order, so the order webhook can still find it. |
| The shopper refused or withdrew analytics, and `priorChoiceTreatment` is `OPTIONAL_LINKED_ANALYTICS_ONLY` | Your storefront only offered a refusal of optional analytics: order events stop, the daily totals continue |
| The shopper objects to the store's business measurement | Leave the shopper's orders out of later snapshots, whatever the dataset's status |

Apply `priorChoiceTreatment` only to choices that were actually recorded. A shopper who never chose has not refused, and a default `false` is not an objection. Do not narrow a broad refusal because a newer banner uses other words. A later analytics grant or a new notice does not clear a business objection.

Offer a business-objection route separate from the consent banner, for example through the support contact in the notice. If a control says "reject all measurement", it must apply to business statistics as well; a control that covers only optional analytics must say so.

Checkout, order service and accounting records keep their own purposes and retention. Removing a shopper's contribution to the statistics must never delete them, and they are no reason to keep optional browsing or attribution data.

### Business-statistics source corrections

Snapshots contain no shopper or order IDs, so only your source can find a shopper's contribution. Connect your platform's privacy export and erasure hooks, and the business-objection route, to the source adapter, including guest orders whose shoppers never had an Irisphera session.

Two merchant routes correct stored days. Both use the server-side merchant key and the captured `X-Irisphera-Channel-Id`, have no body, and return `204`:

| Route | Effect |
| --- | --- |
| `POST /merchant/v2/business-statistics/{datasetKind}/{date}/restrict` | Holds the day back from reports while you correct it. Only a valid replacement with a higher revision clears it; a replay of the old snapshot cannot. |
| `DELETE /merchant/v2/business-statistics/{datasetKind}/{date}` | Deletes the day and blocks it permanently. No replay and no higher revision can restore it. |

Both are idempotent. They work for a channel that the organization owns even after its policy expired or the channel was deactivated; they never reactivate collection. Use deletion only when a valid rebuild is not possible.

Irisphera starts the hold itself for every shopper erasure ([step 10](10-privacy-requests-and-offboarding.md#correct-daily-business-statistics)). For an objection, or an erasure your platform receives directly:

1. Record the exclusion in your source with its scope and time. Stop new inclusion and discard affected queued snapshots. Keep any order locator on your side, never in a snapshot.
2. Restrict the affected days **before** you rebuild them. If you do not know which days are affected, agree a wider hold with Irisphera while you find out.
3. Rebuild each day from the platform's current records, without the excluded contribution, with the original cutover rules. Do not rescan old refused queues or backfill older orders. Leave accounting records untouched.
4. Send each rebuilt day as a complete snapshot with a higher revision, including an explicit zero when nothing remains. A replacement is `COMPLETE` only when your source covered the whole day. If the policy has expired or a rebuild is not possible, delete the day.
5. Record the affected days, revisions and exceptions. Keep the work pending until each restriction, replacement or deletion was acknowledged. Old retries and restored backups must never restore superseded or deleted values.

Keep your source able to rebuild a day for as long as Irisphera keeps it: `aggregateRetentionDays`, plus a short margin. Do not extend customer-record retention just for this; shorten the snapshot lifetime or agree another design with Irisphera instead.

For `COUNTERS`, do not add a persistent visitor identity to make individual removal possible. Honor objections from then on, act on the information the person gives you, and explain the limitation when a past contribution cannot be found. Leaving out identifiers, hashing and aggregation do not by themselves make data anonymous.

## Photo and recording handling

Explain the photo processing before upload. If a face must be blurred, blur it before sending: `isFaceBlurred` describes the uploaded photo; it does not blur anything and does not make the photo anonymous.

For QA recording, exclude photos, generated images, canvases, measurements, identity and contact fields, cart and order details, tokens and request and response bodies. Check the actual recording with synthetic data; do not rely on default masking. Keep credentials and shopper data out of logs, support tickets and screenshots.

Upload only catalog content the merchant may provide, and keep shopper photos and profiles apart from the product feed.

## Retention and historical reports

Agree a retention schedule before activation: purpose, necessary data, maximum duration, the event that starts the clock, and earlier deletion triggers. Configure server data, local delivery copies and evidence separately. Token expiry and the 45-day identity limit are not retention schedules.

The Shopify, WordPress and PrestaShop integrations delete delivered local copies 30 days after successful delivery by default. Existing positive settings stay; zero disables this minimization without stopping collection. Pending deliveries, current restrictions, retry evidence and pending privacy work are kept. These receipts and identifiers can still be personal data and need their own finite lifetime. The merchant's own records, exports, logs, backups and processor copies remain separate responsibilities.

Detailed reports cover the last 12 complete UTC months and the current month by default; `detailedAvailableFrom` gives the boundary ([step 9](09-download-report.md#older-periods)). Withdrawal, erasure, earlier expiry and missing data can reduce any period. Never present unavailable detail as zero.

`historicalBusinessTotals` keeps exact units, orders and gross value per currency for older complete months. It has no shopper drill-down, but its retained contributions are pseudonymous personal data, not anonymous data: withdrawal and erasure can reduce them. Do not add them to overlapping detail, treat a missing month as zero, or present gross value as revenue after refunds.

Treat daily business snapshots as protected statistics, not as anonymous because they carry no shopper ID. A revision, a policy renewal or a retry never resets a day's expiry. Continue disposal and rights work after a policy expires or is removed.

## Data requests and deletion

Agree a monitored support contact and an identity-verification procedure with the merchant and Irisphera before launch. [Step 10](10-privacy-requests-and-offboarding.md) shows the calls.

- Send exports and erasures with `POST /merchant/v2/privacy/requests` from your backend. Keep the `requestId` and the body for exact retries; another body under the same `requestId` returns `409`.
- For a guest-order subject, send the channel captured with the order in `X-Irisphera-Channel-Id`, also when polling. Keep separate requests when the same order number exists on different channels, and never substitute the current installation's channel.
- `202` and `PENDING` mean accepted, not done. Report an erasure as complete only when `status` is `COMPLETED`, and name what is still pending in the meantime.
- An erasure blocks the erased identity from new sessions (`410 identity_erased`). Handle it as an unavailable feature for that account.
- An erasure holds back the organization's daily business statistics until your source adapter has corrected them ([step 10](10-privacy-requests-and-offboarding.md#correct-daily-business-statistics)).
- Keep the credentials and delivery workers needed to finish pending requests. Do not remove them on uninstall or disconnect while work remains.
- Removing an organization is an erasure of all its data, tracked with the integrator key ([step 10](10-privacy-requests-and-offboarding.md#remove-a-merchant-organization)).

`DELETE /shopper/v2/session` and `DELETE /merchant/v2/shopper-sessions/{sessionId}` end a session; they do not erase data. Clearing browser storage, hashing identifiers or downloading a report is not a substitute for a data request.

## Platform wiring

- **Shopify:** require explicit analytics and marketing choices in the Customer Privacy API, with its current processing allowances, before acknowledging analytics on ordinary visits. Do not change Shopify's tracking consent automatically. Unknown host permission stays denied; personalization and QA recording stay separate choices.
- **WordPress/WooCommerce:** connect the merchant CMP through the plugin's consent adapter. Apply withdrawal to the storefront bridge and queued deliveries. Use the plugin's privacy export and erasure hooks and track pending work.
- **PrestaShop:** connect the configured CMP adapter to the actual consent-change events. Apply choices to guest and signed-in sessions, browser capture and server queues.
- **Custom integration:** enforce choices in the storefront, in participating iframes and in your backend, not only in the consent dialog.

Installing a plugin or resolving the collection context does not grant consent. Merchant configuration never overrides a shopper's refusal.

## Acceptance scenarios before enterprise activation

Run these with synthetic shoppers and an isolated test organization:

1. **Fresh visitor:** nothing optional is captured, stored, recorded or sent before a choice is acknowledged.
2. **Reject and reload:** the refusal persists, checkout still works, and the choices can be reopened.
3. **Separate purposes:** granting one purpose does not grant the other two.
4. **Withdraw during queued work:** capture and delivery stop, including retries; delayed responses cannot restore the permission.
5. **Several tabs and an account switch:** stale pages cannot continue under a withdrawn permission or another shopper's identity.
6. **Invalid or conflicting choices:** expired or malformed choices stay denied; a `409` never overwrites a newer withdrawal.
7. **Recording exclusions:** synthetic private markers never appear in a QA recording.
8. **Data requests:** the right organization and subject are targeted, retries are stable, and pending work is reported as pending.
9. **Erasure effects:** the erased identity gets `410 identity_erased`, the report loses its activity, and business days stay held until the source correction is reported.
10. **Business statistics without analytics:** an order from a shopper who made no choice counts in the `COMMERCE` snapshot without an order event or a synthetic shopper. Under `HONOR_BROAD_MEASUREMENT_REFUSAL`, an order from a shopper who refused is left out; a refused order event stays refused.
11. **Missing or wrong policy:** nothing is collected or sent for an absent, expired, wrong-channel, wrong-recipe or unapproved dataset. Missing coverage is reported as unavailable, not zero.
12. **Cutover and broad refusal:** no refused events and no orders from before `effectiveFrom` are sent; broad refusals and objections survive notice changes and later analytics grants.
13. **Replacement:** a replay adds nothing; a higher revision replaces the day; stale or conflicting revisions cannot double-count. Money keeps its precision across days and channels.
14. **Unmapped SKUs, returns and counters:** unmapped units stay in the totals; unobserved returns are `null`, not zero or refund-derived; server-side product requests are `PRODUCT_REQUEST`, not `PRODUCT_VIEWED`; counters carry no visitor identity.
15. **Source correction:** a guest order can be excluded; held days stay out of reports until corrected; replays and restored backups cannot bring removed contributions back.
16. **Retention and removal:** deadlines keep running after a policy is removed; downstream copies stay pending until resolved.
17. **Report populations:** event figures, business totals and historical archives are kept apart; partial and missing days stay visible; no store-wide conversion rate is computed from opt-in figures, and attribution is not presented as proof of cause.

Confirm the merchant's notice, the configured choices, the source recipe, retention and the support handoff before real shoppers use the integration. Passing these tests does not replace that evidence or activate a feature.

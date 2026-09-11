# Privacy and consent requirements for an integration

[Call agenda](../README.md)

Wire shopper choices, consent-aware delivery and data requests alongside the [six-step integration walkthrough](../README.md). Confirm the deployed API contract and enabled purposes with Irisphera before integration testing.

## Before sending any real shopper data

- Agree the permitted processing, merchant/Irisphera responsibilities, applicable agreements, notices, recipients, retention and support contacts before sending shopper data.
- Explain photo uploads and server/provider processing before a participant uses try-on or sizing. Use consenting participants and test identities for the walkthrough.
- Treat photos, measurements, shopper identifiers and linked order history as personal data. Do not promise browser-only processing or immediate deletion.
- Keep optional behavioral analytics/attribution, saved personalization and QA session recording separate. Use the merchant CMP's valid, current permission for the covered optional analytics purpose; do not require a second Irisphera opt-in for that same purpose. A merchant credential, login, order webhook or terms acceptance is not permission.
- If separately approved merchant business measurement is available, follow its own source, notice, rights and retention requirements below. It does not enable optional browser tracking, personalization, QA or identity linkage.

## Connect v2 shopper choices

Keep the three optional purposes separate:

| Purpose | Integration behavior |
| --- | --- |
| Behavioral analytics / attribution | Gate optional observations, linked conversion measurement and their delivery. |
| Saved personalization | Gate optional profile persistence and reuse across visits. |
| QA session recording | Gate session-replay loading and capture independently of the other purposes. |

A requested try-on or sizing operation does not enable optional purposes. Refusal must not disable ordinary store checkout or unrelated services. Keep a visible way to reopen preferences and withdraw a choice.

Until permission is known and acknowledged, keep optional capture, persistence, recorder loading and delivery off. A missing, malformed, expired or incompatible preference is not a grant.

On an ordinary page visit, read the merchant CMP and acknowledge eligible analytics permission through the preference API below. Do not wait for the shopper to open try-on or sizing. Keep personalization and QA recording off unless separately chosen. If no supported CMP supplies permission, retain the explicit consent flow. Never replace a recorded refusal, withdrawal, expired choice or identity-reset restriction with an automatic grant.

### Authoritative preference API

| Caller | Endpoint | Credential |
| --- | --- | --- |
| Shopper | `GET` / `PUT /shopper/v2/privacy/preferences` | Shopper bearer for the current merchant/channel/session |
| Trusted storefront backend | `GET` / `PUT /merchant/v2/shopper-sessions/{sessionId}/privacy-preferences` | Merchant key; keep it server-side; use captured `X-Irisphera-Channel-Id` for historical sessions |

1. Read the current preferences and version.
2. Submit the three purpose values with notice version `2026-09-06` and the current `expectedVersion`, using the request schema from the deployed API contract. Analytics may come from the merchant CMP; personalization and QA require their separate choices. A missing preference record has version `0` and all purposes denied until acknowledgment.
3. Wait for server acknowledgment before starting optional processing. A grant requires a positive acknowledged version and the relevant purpose set to `true`. A timestamp in `expiresAt` must be in the future; `expiresAt: null` on an acknowledged grant means no scheduled expiry, not implicit consent. A pending request, missing version-zero record or local checkbox alone is insufficient.
4. On `409`, reread the preferences. Do not overwrite a newer withdrawal with a stale choice.
5. Apply the merchant's consent-management platform (CMP) restrictions as well. Server acknowledgment does not override a host refusal.

Derive the session from authenticated server state. Never let a browser-selected customer or subject identify the target of a privileged preference update. Confirm purpose availability and consent validity with Irisphera before activation; collection context alone grants no consent. Do not bypass a denied or unavailable purpose to complete a demo.

Shopper access tokens last 30 minutes by default. Browser expiry and revocation remain enforced, but they do not end an acknowledged merchant consent grant. The trusted backend can read current preferences and propagate restrictions using the captured scope after browser expiry or session-row cleanup. It cannot use that historical scope to enable another purpose, add a legacy identity or renew consent expiry. Refusal and withdrawal remain effective independently of age-based deletion settings.

### Browser identity is not consent or a bearer token

Shopify, WordPress and PrestaShop use a fixed 45-day window for newly issued v2 anonymous browser identity. The deadline starts at first issuance. Visits, browser restarts, token refreshes and consent acknowledgments do not move it. Existing shorter identities are not extended by this change.

- Retain only protected first-party ownership or an opaque server-backed handle. Never infer continuity from IP addresses, fingerprints, raw legacy cookies or a supplied customer ID.
- Keep short-lived access tokens and consent deadlines separate. A surviving identity does not permit collection after consent expiry, withdrawal, host denial or an identity/security reset.
- On exactly HTTP `410` with problem type `session_expired`, a trusted backend may create a new session for the same still-owned alias after checking current permission. It must not use an expired anonymous continuation as login proof. `identity_erased`, other `410` responses and `422` are not renewal signals.
- Keep the original acknowledged session/version pair on immutable order evidence. A replacement session ID must not be paired with an older consent version. Delayed commerce still checks current consent and erasure fences; browser identity expiry alone does not invalidate a captured order receipt.
- At the fixed deadline, stop browser v2 capture and retries under that identity. Any later eligible identity is new; do not reconnect its prior browser queue or history. Legacy identity and delivery lifetimes remain unchanged.

The 45-day setting is a technical limit, not approval of a legal basis, a consent duration or a server-data retention schedule. The merchant must disclose and approve the configured first-party storage and processing.

### Withdrawal, retries and identity changes

- Check permission before capture and again before delivery, including outbox retries.
- On withdrawal, stop affected producers, recorder capture and queued sends. Prevent delayed responses from restoring withdrawn permission or repopulating optional saved data. Initiate disposal of consent-only retained data without requiring a separate erasure request, and track downstream completion. Preserve only records with a genuine separate purpose and applicable retention; report exclusion alone is not deletion.
- Propagate changes to participating tabs and iframes. Validate message source and origin; do not send tokens, photos or profiles to arbitrary origins.
- Re-evaluate permission after login, logout or account switching. Do not transfer one shopper's choice or history to another shopper.
- Keep consent withdrawal, session revocation and a data-erasure request as separate actions.

## Separately approved merchant business measurement

**Contract preview:** activate only after Irisphera confirms deployment support and the merchant's approved source configuration. An installed plugin, a working API key or this documentation does not activate the purpose.

The separate [business-statistics path](05-collect-data.md#send-separately-approved-business-statistics) sends minimized daily COMMERCE or COUNTERS snapshots, not shopper histories. It can support broad merchant Orders, Purchased Units, gross purchase amounts, AOV, product sales, physical-return units and approved request/outcome counts. The selected lawful basis and source method must be established before processing. A legitimate-interest determination can cover an appropriate business-measurement source; it is not a universal analytics exemption or authority to reuse refused linked events.

### Read the source policy; do not create a grant

`POST /merchant/v2/collection-context` adds `businessStatistics.commerce` and `businessStatistics.counters` where supported. Each reports a `status`:

| Status | Source action |
| --- | --- |
| `AVAILABLE` | Verify the exact captured channel, versions, time window, allowed types and local restrictions before capture/delivery. This status is not certification of the merchant's legal evidence. |
| `NOT_APPROVED` | Do not start capture or delivery for this dataset. |
| `NOT_YET_EFFECTIVE` | Wait for the actual approved source/notice cutover; do not buffer earlier events for later inclusion. |
| `EXPIRED` | Stop affected collection/delivery; continue required retention and rights work. |
| `RESTRICTED` | Stop affected source processing and follow the restriction/correction workflow. |

An absent `businessStatistics` field is not approval. Availability can include `businessPolicyVersion`, `sourceRecipeVersion`, `effectiveFrom`, `expiresAt`, `sourceRetentionDays`, `aggregateRetentionDays`, `allowedCounterTypes`, `priorChoiceTreatment` and a `reason`. Keep the returned policy bound to its channel/dataset; one installation's policy cannot authorize another. The interface does not expose private legal evidence or provide a self-approval operation.

`priorChoiceTreatment` is `HONOR_BROAD_MEASUREMENT_REFUSAL` or `OPTIONAL_LINKED_ANALYTICS_ONLY`, based on the reviewed original notices. Apply it to real recorded choices. A missing optional choice or default `false` is not evidence that the shopper actively refused all merchant measurement; do not invent an objection from it. Conversely, a broad recorded refusal must not be narrowed merely because a newer UI label differs.

Confirm these deployment facts before enabling a native source:

- A specific business purpose, approved basis, source recipe, fields, dimensions and non-overlapping source partition.
- An accessible business-purpose notice with the actual source, purpose/basis, retention, recipients and rights route. Its version is separate from the optional-consent notice; publishing it must not change existing optional grants.
- The terminal storage/access determination for that exact method. Backend aggregation is not permission to add a browser beacon or reuse a necessary cookie for analytics. COMMERCE approval does not approve COUNTERS or every counter type.
- A prospective effective date and source deployment that exclude historical backfill and old rejected consent envelopes. A new delivery time, UUID or policy label does not make an old event eligible.
- A finite source and aggregate lifecycle, tested source reconciliation, active objection route and assigned downstream completion owner.

### Distinguish refusal, withdrawal and business objection

Refusal or withdrawal of **optional linked analytics** stops that processing. It does not, by itself, stop a genuinely separate, properly approved and disclosed business-measurement purpose. Do not require a fabricated guest receipt or optional opt-in just to submit an otherwise eligible business snapshot.

However, honor the actual choice presented to the shopper. If the old notice or control promised refusal of all measurement, do not silently narrow it after the fact. Carry applicable broad refusals into the business-source restriction until the scope is properly resolved. A new business notice or a later optional-consent grant must not clear a recorded business objection.

Offer a clear merchant business-objection route, separate from the optional consent setting. Stop affected measurement and route the request through the agreed rights workflow. Do not treat an empty analytics checkbox as the only possible objection, or override a business objection because a policy remains `AVAILABLE`. If a control says “Reject all measurement”, it must apply to both scopes; a control limited to optional linked analytics must say so clearly.

Necessary checkout, order service and properly retained accounting records keep their own purposes and duties. Removing an analytics contribution must not delete mandatory native merchant records. Conversely, those duties are not authority to retain optional browsing or attribution data.

## V2 platform wiring

These controls describe the existing **optional-consent** path. An independently approved native business source follows its own policy and rights controls above; it must not bypass these optional gates.

- **Shopify:** require explicit analytics and marketing choices in the Customer Privacy API together with its current processing allowances before acknowledging analytics on ordinary visits. Do not change Shopify's tracking consent automatically. Unknown host permission remains denied; personalization and QA remain separate choices. Revalidate bounded session continuation after navigation and apply withdrawal to browser capture and server delivery.
- **WordPress/WooCommerce:** connect the merchant CMP through the plugin's consent adapter. Apply withdrawal to the storefront bridge and queued deliveries. Use the plugin's privacy export/erase hooks and track pending downstream work.
- **PrestaShop:** connect the configured CMP adapter to actual consent-change events. Apply choices to guest and authenticated sessions, browser capture and server queues. Merchant configuration must not bypass shopper refusal.
- **Custom API / browser SDK:** enforce choices in the host, participating iframe and trusted backend, not only in the preference dialog.

Use the relevant platform integration's configuration instructions for its adapter. Installing a plugin or selecting a collection mode does not grant consent.

## Legacy/v2 overlap and reports

Both supported collection generations and both report families remain available. Independent legacy v1 collection and reporting do not depend on the v2 consent ledger. V2 capture, delivery, retries and reports enforce the purpose restrictions described above; refusal or withdrawal stops affected v2 processing. Use `dual` only for the agreed comparison window. Keep source selection explicit: a rejected v2 event must not be redirected to legacy as a fallback. These API rules do not replace the merchant's legal obligations for either version.

Preserve collection provenance independently of payload schema version. Retain per-target delivery outcomes so a failed v2 request does not cause an already accepted legacy event to be replayed.

V1 reports describe stored legacy activity without requiring retroactive v2 consent records. Existing v2 event-derived fields describe activity eligible under v2 consent checks, including eligible historical v2 events. Where supported and separately approved, `merchantBusinessAnalytics` describes the independent merchant source; it does not change those event-derived fields. The populations can differ: do not label a v2 opt-in cohort as all visitors, fabricate denied events, or sum these sources as unique activity. Identity linking and reporting do not grant permission to collect additional history.

## Photo and recording handling

Explain the configured photo-processing path before upload. If a face must be blurred, blur it before transmission: `isFaceBlurred` describes the submitted input and does not perform blurring or establish anonymity.

For optional session replay, exclude photos, generated results, canvases, measurements, identity/contact fields, cart/order details, tokens and request/response bodies. Check the actual capture using synthetic data; do not rely solely on default masking. Keep credentials and raw shopper data out of logs, support tickets and screenshots.

Upload only catalog content the merchant is authorized to provide, and keep shopper photos and profiles separate from the product feed.

## Retention and historical reports

Agree a justified retention schedule before activation: purpose, necessary data, maximum duration, clock-start event and earlier deletion triggers. Configure server-side data, local delivery copies and evidence stores separately. Session/token expiry and the 45-day browser identity limit are not data-retention schedules. Disabling age-based deletion does not grant consent or stop otherwise authorized collection.

Detailed v2 reports use currently retained, authorized sources. The default policy protects the latest 12 complete UTC calendar months plus the preceding month needed for attribution. The response's `detailedAvailableFrom` marks the detail boundary, not a guarantee that all data survived. Other zones may need extra source coverage. Withdrawal, erasure, earlier expiry and missing observations can change results; do not present unavailable detail as zero activity.

Shopify, WordPress and PrestaShop default missing delivered-copy durations to 30 days after successful delivery, not capture. Existing positive overrides remain; explicit zero disables that minimization without stopping authorized collection. Existing platform activation, hold and pending-rights gates remain. PrestaShop requires its native 1.7.5 upgrade; historical rows with unknown successful-delivery times are preserved rather than assigned invented timestamps. Keep pending delivery, current restrictions, exact retry evidence and captured rights selectors intact. These minimal receipts and identifiers may still be personal data and need a separate justified lifecycle. Native merchant records, exports, logs, backups and processor copies remain separate responsibilities.

An optional coarse statistical archive is not a replacement for the detailed monthly report. Do not describe linked reports as anonymous, combine coarse archive counts with detailed totals, or enable an archive without Irisphera's approved merchant-specific assessment. Confirm the configured policy and actual deletion/rights behavior with synthetic data before using real shopper data.

For older complete UTC months, `historicalBusinessTotals` returns exact Purchased Units, whole Orders and gross purchase value separately by currency, with order-value exclusions and a privacy-adjusted indicator. These totals have no shopper drill-down, but their retained contributions are pseudonymous personal data, not anonymous. Withdrawal and erasure can reduce them. Do not combine them with overlapping detail or coarse ranges, treat missing months as zero, or describe gross value as revenue net of refunds. Late delivery does not revise a sealed month. Agree a separate finite financial-contribution retention/disposal schedule and verify its effective operation; report availability does not authorize indefinite retention.

Independent business snapshots have their own source and aggregate limits. Treat exact daily/SKU cells as protected statistics, not automatically anonymous because no shopper identifier was transmitted. Keep enough lawful source capability to correct live personal snapshots, or shorten their lifetime/use the agreed suppression path. Do not extend native customer-record retention merely to preserve analytics. Revision, policy renewal or retry must not reset the original bucket's expiry. Continue disposal, queue minimization and rights work after a policy expires or is removed.

## Data requests and deletion

Agree a monitored contact and identity-verification procedure with the merchant and Irisphera before launch.

The trusted backend uses the merchant-authenticated workflow. For an order-guest selector, retain the captured `X-Irisphera-Channel-Id`; identity selectors remain merchant-scoped:

1. Submit `POST /merchant/v2/privacy/requests` with the request kind and authorized subject selector defined in the deployed contract.
2. Keep the same `requestId` and body for an exact retry. Reusing an identifier with different content is a conflict.
3. Poll `GET /merchant/v2/privacy/requests/{requestId}` with the same captured `X-Irisphera-Channel-Id` for guest requests, and inspect the per-system status. Keep separate targets when the same source order ID exists in different channels; never substitute the current installation's channel.
4. Report completion only when the workflow confirms it. An accepted request or `202` response represents pending work, not completed erasure.

Keep the credentials and delivery workers needed to complete pending requests. Do not remove them during uninstall or disconnect while downstream work remains outstanding. Coordinate applicable retention exceptions and provider handling with Irisphera.

`DELETE /shopper/v2/session` revokes a session only. It does not erase historical data. Deleting browser storage, replacing identifiers with hashes or downloading a merchant performance report is not a substitute for the data-request workflow.

### Business-statistics source corrections

Central daily snapshots have no subject or order key. An existing shopper EXPORT/ERASE selector alone cannot locate that shopper's contribution in them. Connect native merchant privacy export/erasure hooks **and a business-objection entrypoint** to the source adapter, including guest orders whose shoppers have no Irisphera session. Do not invent a shopper request kind or encode an objection as an analytics grant.

The business-statistics contract provides two merchant-authenticated rights operations. Both require the server-held merchant key and captured `X-Irisphera-Channel-Id`; neither has a request body:

| Operation | Effect |
| --- | --- |
| `POST /merchant/v2/business-statistics/{datasetKind}/{date}/restrict` | `204`: temporarily exclude the bucket from reads while source reconciliation is pending. Idempotent. Only a valid higher-revision corrected replacement clears the restriction; an exact replay cannot. |
| `DELETE /merchant/v2/business-statistics/{datasetKind}/{date}` | `204`: remove values and permanently suppress that source bucket. Idempotent. Neither an old replay nor a higher revision can restore it. |

These rights operations do not require an active collection policy and remain available for an owned captured channel after policy/channel expiry or inactivation. They never authorize cross-merchant access or reactivate collection. A missing owned bucket can still be fenced against delayed delivery. Do not use permanent deletion as a temporary hold when a valid reconstruction is available.

1. Persist the source restriction with its scope and effective time. Stop new inclusion and affected queued delivery. Keep any necessary customer/order locator on the merchant side, or in the protected rights task where needed, never in the statistics snapshot.
2. Identify affected retained channel/day/dataset buckets and call their restriction operation **before clearing/rebuilding source data**. If the range is unknown, coordinate a conservative source-scope hold with Irisphera while identifying affected buckets. Do not claim the central service can identify the person from totals alone, or claim a remote hold succeeded before acknowledgement.
3. Rebuild from still-lawful authoritative merchant facts with their original purpose/cutover eligibility, excluding affected contributions. Do not rescan old rejected queues or backfill pre-cutover purchases. Leave mandatory native accounting records intact.
4. Where ordinary snapshot admission remains valid, deliver higher-revision full replacements, including explicit zero/empty replacements when appropriate. A complete replacement of the saved bucket is not necessarily `coverage.status: COMPLETE`; source gaps must remain partial. If approval has expired or a lawful rebuild cannot be made, use the permanent bucket deletion operation instead of faking a new policy. Old outbox retries and restored backups must not restore superseded or deleted values.
5. Track corrections to caches, exports and downstream copies. Record affected ranges, revisions, source evidence and exceptions. Keep work pending if a restriction, replacement or deletion call is unacknowledged. Confirm completion only when required source and downstream work is complete and the applicable restriction is reflected; a bucket `204` alone does not prove every copy was deleted.

If the source cannot rebuild an affected bucket, remove it through the deletion operation rather than assert a complete correction. Keep pending rights work and the credentials/workers needed to finish it during disconnect or uninstall. Source retention must cover the central aggregate lifetime plus the configured suppression/reconciliation margin, within the approved finite schedule. A short delivery-copy lifetime does not establish that older snapshots can be corrected. Do not silently extend native customer retention to meet this condition; shorten aggregate retention or revise the lawful design when needed.

For COUNTERS, do not introduce persistent visitor identity solely to promise individual subtraction. Document what lawful existing information can locate a contribution, honor future objections at the actionable source path, and act on usable information provided by the person. If a reliable historical subtraction is not possible, explain the actual limitation and use the agreed affected-bucket handling. No-identifier input, hashing and aggregate output do not alone establish anonymity or completed erasure.

## Acceptance scenarios before enterprise activation

Exercise the v2 integration with synthetic shoppers and an isolated test merchant. Separately verify that configured v1 delivery remains independent and that v2 denial never creates a legacy fallback:

1. **Fresh visitor:** no optional capture, persistence, recording or delivery before a choice is acknowledged.
2. **Reject and reload:** rejection persists; ordinary checkout remains usable; preferences can be reopened.
3. **Separate purposes:** enabling one purpose does not enable the other two.
4. **Withdraw during queued work:** affected v2 capture and delivery stop, including retries; delayed responses cannot restore permission. Independent v1 delivery retains its own outcome and must not be replayed.
5. **Multiple tabs and account switching:** stale contexts cannot continue under revoked permission or another shopper's identity.
6. **Invalid or conflicting preferences:** expired/malformed permission stays denied; a `409` is handled without overwriting a newer withdrawal.
7. **Recording exclusions:** synthetic private markers do not appear in captured content.
8. **Data request:** the correct merchant/subject is targeted, retries are stable, and pending downstream work is distinguished from completed deletion.
9. **Approved business source with optional refusal:** an actual native new order contributes to the approved COMMERCE snapshot without an optional receipt or synthetic shopper; the existing optional event route remains denied.
10. **Missing or mismatched business policy:** no capture/delivery for absent, expired, wrong-channel, wrong-recipe or unapproved datasets. Missing coverage is unavailable, not zero.
11. **Cutover and broad refusal:** no old rejected envelopes or pre-cutover purchase backfill; broad measurement refusals and business objections survive notice/version changes and later optional grants.
12. **Exact replacement:** replay does not add counts; a newer revision replaces the day; conflicting/stale revisions and concurrent source updates cannot double-count. Retain money precision across day/channel sums.
13. **Missing SKU, unavailable returns and counters:** unmapped units remain in all-product totals; unobservable physical-return fields remain required `null`, not false zero or refund-derived counts; mixed return coverage stays partial. Approved server requests use `PRODUCT_REQUEST`, not `PRODUCT_VIEWED`; counters contain no visitor/cohort inference or browser identifiers.
14. **Source rights correction:** a native guest order can be excluded; affected report scope remains unavailable while pending; corrected/empty revisions propagate; replay/restore cannot restore removed contributions.
15. **Retention and removal:** category deadlines still run after policy removal; source reconciliation remains viable for live personal snapshots or affected buckets are suppressed; downstream copies remain pending until resolved.
16. **Report populations:** independent business totals are separate from opt-in detail and historical archives; partial/missing days and source partitions remain visible; no all-store/consented-denominator conversion or claimed causal uplift.

Confirm the merchant's notices, configured choices, source approvals, retention arrangements and support handoff before using real shopper data. Passing synthetic tests does not create the missing deployment evidence or activate a purpose.

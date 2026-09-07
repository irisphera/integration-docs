# Privacy and consent requirements for an integration

[Call agenda](../README.md)

Wire shopper choices, consent-aware delivery and data requests alongside the [six-step integration walkthrough](../README.md). Confirm the deployed API contract and enabled purposes with Irisphera before integration testing.

## Before sending any real shopper data

- Agree the permitted processing, merchant/Irisphera responsibilities, applicable agreements, notices, recipients, retention and support contacts before sending shopper data.
- Explain photo uploads and server/provider processing before a participant uses try-on or sizing. Use consenting participants and test identities for the walkthrough.
- Treat photos, measurements, shopper identifiers and linked order history as personal data. Do not promise browser-only processing or immediate deletion.
- Obtain separate choices for optional behavioral analytics/attribution, saved personalization and QA session recording. A merchant credential, login, order webhook or terms acceptance is not permission for those purposes.

## Connect shopper choices

Keep the three optional purposes separate:

| Purpose | Integration behavior |
| --- | --- |
| Behavioral analytics / attribution | Gate optional observations, linked conversion measurement and their delivery. |
| Saved personalization | Gate optional profile persistence and reuse across visits. |
| QA session recording | Gate session-replay loading and capture independently of the other purposes. |

A requested try-on or sizing operation does not enable optional purposes. Refusal must not disable ordinary store checkout or unrelated services. Keep a visible way to reopen preferences and withdraw a choice.

Until permission is known and acknowledged, keep optional capture, persistence, recorder loading and delivery off. A missing, malformed, expired or incompatible preference is not a grant.

### Authoritative preference API

| Caller | Endpoint | Credential |
| --- | --- | --- |
| Shopper | `GET` / `PUT /shopper/v2/privacy/preferences` | Shopper bearer for the current merchant/channel/session |
| Trusted storefront backend | `GET` / `PUT /merchant/v2/shopper-sessions/{sessionId}/privacy-preferences` | Merchant key; keep it server-side; use captured `X-Irisphera-Channel-Id` for historical sessions |

1. Read the current preferences and version.
2. Submit the three explicit choices with notice version `2026-09-06` and the current `expectedVersion`, using the request schema from the deployed API contract. A missing preference record has version `0` and all purposes denied.
3. Wait for server acknowledgment before starting optional processing. A grant requires a positive acknowledged version and the relevant purpose set to `true`. A timestamp in `expiresAt` must be in the future; `expiresAt: null` on an acknowledged grant means no scheduled expiry, not implicit consent. A pending request, missing version-zero record or local checkbox alone is insufficient.
4. On `409`, reread the preferences. Do not overwrite a newer withdrawal with a stale choice.
5. Apply the merchant's consent-management platform (CMP) restrictions as well. Server acknowledgment does not override a host refusal.

Derive the session from authenticated server state. Never let a browser-selected customer or subject identify the target of a privileged preference update. Confirm purpose availability and consent validity with Irisphera before activation; collection context alone grants no consent. Do not bypass a denied or unavailable purpose to complete a demo.

### Withdrawal, retries and identity changes

- Check permission before capture and again before delivery, including outbox retries.
- On withdrawal, stop affected producers, recorder capture and queued sends. Prevent delayed responses from restoring withdrawn permission or repopulating optional saved data.
- Propagate changes to participating tabs and iframes. Validate message source and origin; do not send tokens, photos or profiles to arbitrary origins.
- Re-evaluate permission after login, logout or account switching. Do not transfer one shopper's choice or history to another shopper.
- Keep consent withdrawal, session revocation and a data-erasure request as separate actions.

## Platform wiring

- **Shopify:** combine Customer Privacy API restrictions and change events with Irisphera purpose choices. Unknown host permission remains denied. Apply changes to browser capture and server delivery.
- **WordPress/WooCommerce:** connect the merchant CMP through the plugin's consent adapter. Apply withdrawal to the storefront bridge and queued deliveries. Use the plugin's privacy export/erase hooks and track pending downstream work.
- **PrestaShop:** connect the configured CMP adapter to actual consent-change events. Apply choices to guest and authenticated sessions, browser capture and server queues. Merchant configuration must not bypass shopper refusal.
- **Custom API / browser SDK:** enforce choices in the host, participating iframe and trusted backend, not only in the preference dialog.

Use the relevant platform integration's configuration instructions for its adapter. Installing a plugin or selecting a collection mode does not grant consent.

## Legacy/v2 overlap and reports

Both supported collection generations and both report families remain available. Apply the same purpose restrictions to legacy and v2 delivery. Use `dual` only for the agreed comparison window; refusal or withdrawal must stop affected delivery to both targets.

Preserve collection provenance independently of payload schema version. Retain per-target delivery outcomes so a failed v2 request does not cause an already accepted legacy event to be replayed.

Reports describe the population actually collected with permission. Do not label an opt-in cohort as all visitors, fabricate denied events or interpret missing activity as zero. Identity linking and reporting do not grant permission to collect additional history.

## Photo and recording handling

Explain the configured photo-processing path before upload. If a face must be blurred, blur it before transmission: `isFaceBlurred` describes the submitted input and does not perform blurring or establish anonymity.

For optional session replay, exclude photos, generated results, canvases, measurements, identity/contact fields, cart/order details, tokens and request/response bodies. Check the actual capture using synthetic data; do not rely solely on default masking. Keep credentials and raw shopper data out of logs, support tickets and screenshots.

Upload only catalog content the merchant is authorized to provide, and keep shopper photos and profiles separate from the product feed.

## Data requests and deletion

Agree a monitored contact and identity-verification procedure with the merchant and Irisphera before launch.

The trusted backend uses the merchant-authenticated workflow. For an order-guest selector, retain the captured `X-Irisphera-Channel-Id`; identity selectors remain merchant-scoped:

1. Submit `POST /merchant/v2/privacy/requests` with the request kind and authorized subject selector defined in the deployed contract.
2. Keep the same `requestId` and body for an exact retry. Reusing an identifier with different content is a conflict.
3. Poll `GET /merchant/v2/privacy/requests/{requestId}` and inspect the per-system status.
4. Report completion only when the workflow confirms it. An accepted request or `202` response represents pending work, not completed erasure.

Keep the credentials and delivery workers needed to complete pending requests. Do not remove them during uninstall or disconnect while downstream work remains outstanding. Coordinate applicable retention exceptions and provider handling with Irisphera.

`DELETE /shopper/v2/session` revokes a session only. It does not erase historical data. Deleting browser storage, replacing identifiers with hashes or downloading a merchant performance report is not a substitute for the data-request workflow.

## Acceptance scenarios before enterprise activation

Exercise the integration with synthetic shoppers and an isolated test merchant:

1. **Fresh visitor:** no optional capture, persistence, recording or delivery before a choice is acknowledged.
2. **Reject and reload:** rejection persists; ordinary checkout remains usable; preferences can be reopened.
3. **Separate purposes:** enabling one purpose does not enable the other two.
4. **Withdraw during queued work:** affected capture and delivery stop, including retries to legacy and v2; delayed responses cannot restore permission.
5. **Multiple tabs and account switching:** stale contexts cannot continue under revoked permission or another shopper's identity.
6. **Invalid or conflicting preferences:** expired/malformed permission stays denied; a `409` is handled without overwriting a newer withdrawal.
7. **Recording exclusions:** synthetic private markers do not appear in captured content.
8. **Data request:** the correct merchant/subject is targeted, retries are stable, and pending downstream work is distinguished from completed deletion.

Confirm the merchant's notices, configured choices, retention arrangements and support handoff before using real shopper data.

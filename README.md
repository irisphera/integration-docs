# Irisphera integration guide

This guide walks through a full integration, from an integrator API key to a downloaded report and the shopper's data requests. It follows one merchant organization, one garment and one shopper: you create the organization, import its products, open a shopper session with the shopper's privacy choices, run a virtual try-on, a recommendation and a mix-and-match suggestion, record a two-unit purchase and a one-unit return, download the report, and finally export and erase the shopper's data.

Each step gives the requests to send, the values to keep, and a checkpoint to pass before the next step. Run the commands in order in one Bash terminal: every step uses values from the previous ones. Use a test environment and test customers. The commerce calls record what happened in your store; they do not charge or refund a payment.

This walkthrough covers the current Irisphera API. Collection is v2 only. It is not an endpoint catalog or a plugin installation guide: the full schemas are in the deployed OpenAPI contract at `/v3/api-docs`.

## Credentials

| Credential | Header | Who holds it | Used for |
| --- | --- | --- | --- |
| Integrator API key | `INTEGRATOR-API-KEY` | Your platform backend | Creating and managing the merchant organizations you own |
| Merchant API key | `MERCHANT-API-KEY` | Your backend, one key per merchant organization | Catalog, collection context, shopper sessions, commerce events, reports, privacy requests |
| Shopper access token | `Authorization: Bearer <token>` | The shopper's browser, valid for 30 minutes | Virtual try-on, sizing, recommendations, mix and match, storefront events, the shopper's own privacy choices |

API keys stay on your servers. Only the short-lived shopper token reaches the browser. Send exactly one credential per request.

```text
integrator key ──► merchant organization ──► merchant key
                                               ├─ collection context: channel, namespaces, notice version
                                               ├─ catalog: collection ──► products
                                               ├─ shopper session ──► shopper token (browser)
                                               │     ├─ privacy choices
                                               │     ├─ virtual try-on, 3D preview, image shares
                                               │     ├─ sizing, recommendations, mix and match
                                               │     └─ product views and cart changes
                                               ├─ orders, payments, returns, refunds
                                               ├─ daily business statistics
                                               ├─ report
                                               └─ privacy requests (export, erase)
```

## Steps

| Step | What you do | Checkpoint |
| --- | --- | --- |
| [1. Prepare](docs/01-prepare.md) | Get the environment URL and integrator key; open a private terminal; check the deployed contract | The contract lists the routes this guide uses |
| [2. Create the merchant organization](docs/02-create-merchant.md) | Create the organization with your integrator key, configure its storefront flow, resolve its collection context | Merchant ID, merchant key, channel and namespaces |
| [3. Ingest products](docs/03-ingest-products.md) | Create a collection, import a feed or single products, check try-on readiness | The SKU is listed and ready for try-on |
| [4. Open a shopper session](docs/04-shopper-session.md) | Start an anonymous session, record the shopper's privacy choices, sign the shopper in | A shopper token with the feature scopes and an acknowledged choice |
| [5. Virtual try-on](docs/05-virtual-try-on.md) | Generate a try-on image, show the 3D preview, record image shares | A generated image on screen |
| [6. Recommendations and sizing](docs/06-recommendations.md) | Estimate measurements and colors, build the profile, get recommendations | Recommended SKUs with sizes |
| [7. Mix and match](docs/07-mix-and-match.md) | Import a product that completes the outfit, get one suggestion for each placement | A matching product for another placement |
| [8. Storefront and order events](docs/08-collect-events.md) | Send product views, cart changes, the order, payment, return and refund; understand daily business statistics | Receipts for every event and a safe replay |
| [9. Download the report](docs/09-download-report.md) | Download the JSON report and reconcile it with the walkthrough | Orders, units, returns and attributed units match |
| [10. Privacy requests and offboarding](docs/10-privacy-requests-and-offboarding.md) | Export and erase the test shopper, end sessions, remove a merchant organization | Export received; erasures accepted and tracked |

Read the [privacy and consent requirements](docs/privacy-and-consent.md) before you send any real shopper data. The walkthrough follows them, but they also cover what your storefront, consent banner and support process must do.

## Arrange these before you start

- Irisphera supplies the environment URL and your integrator key, and enables virtual try-on and recommendation quota for the environment.
- Agree the merchant's privacy notice, processing instructions, retention and support contacts with Irisphera before real shoppers use the integration.
- Use one catalog SKU for the product everywhere: in the feed, in try-on, in events and in order lines. If your platform uses other product identifiers, agree a canonical SKU mapping with Irisphera. Importing a feed does not create aliases.
- Bring real image URLs for the garment and for one product that goes with it, and a photo of a test participant who has agreed to take part.

## Platform plugins

WordPress/WooCommerce and PrestaShop merchants install the Irisphera plugin and enter only their merchant key: Irisphera acts as their integrator, and no integrator key belongs in those plugins. The Shopify app's hosted backend uses an integrator key to manage its merchants. The plugins do steps 2 to 6 and step 8 themselves, but not mix and match yet. This guide is for custom integrations and for understanding what the plugins do.

## If you integrated with an earlier version

- Collection is v2 only. `collectionMode` is always `v2`. There is no legacy or dual collection, no comparison window, and a refused v2 event never falls back to another pipeline.
- Shopper features use `/shopper/v2/...` with the session token. The `/shopper/v1` routes are no longer served.
- The report is `POST /merchant/v2/report`. `/merchant/v1/report` is retired.
- Attribution is purchase-anchored: each purchased unit looks back a fixed number of days from its purchase for a successful try-on of the same product by the same shopper. The report states the rule in `attributionPolicy`.
- Daily COMMERCE business statistics are available by default on every channel. COUNTERS still needs a separate approval.
- Image shares have their own route and report section.
- Data requests use `POST /merchant/v2/privacy/requests`.
- Mix and match is new: `GET /shopper/v2/mixmatch/{skuCustomId}` returns products that complete an outfit. See [step 7](docs/07-mix-and-match.md).
- An erasure holds back the affected daily business statistics until your source corrects them. See [step 10](docs/10-privacy-requests-and-offboarding.md#correct-daily-business-statistics).

Start with [1. Prepare](docs/01-prepare.md).

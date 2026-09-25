# Irisphera integration guide

This guide is for enterprises that connect their own commerce platform directly to the Irisphera API.

| Resource | URL |
| --- | --- |
| API base URL | `https://api.irisphera.com` |
| OpenAPI contract | [`https://api.irisphera.com/v3/api-docs`](https://api.irisphera.com/v3/api-docs) (OpenAPI 3.1, JSON) |

## Generate a client

The OpenAPI contract describes every route, header and schema of the API. [OpenAPI Generator](https://openapi-generator.tech/) turns it into a typed client in the language your backend uses, such as Java, C#, TypeScript, Python, Go or PHP. For example, with Docker:

```bash
docker run --rm -v "$PWD:/local" openapitools/openapi-generator-cli:v7.25.0 generate \
  -i https://api.irisphera.com/v3/api-docs \
  -g java -o /local/irisphera-client \
  --openapi-normalizer 'FILTER=tag:Integrator|Merchant|Shopper'
```

- Replace `java` with the generator for your language. The [generator list](https://openapi-generator.tech/docs/generators) names them all, and the [installation page](https://openapi-generator.tech/docs/installation) shows other ways to run the tool.
- The `FILTER` option keeps the three groups of routes you use. They become the `IntegratorApi`, `MerchantApi` and `ShopperApi` classes. The contract also lists routes under `/internal`. Those are for Irisphera only, and the filter leaves them out.
- Set your credentials in the client's configuration, under the scheme names from the contract: `IntegratorApiAuth` sends `INTEGRATOR-API-KEY`, `MerchantApiAuth` sends `MERCHANT-API-KEY`, and `ShopperAccessTokenAuth` sends the shopper token as `Authorization: Bearer`.
- The client calls `https://api.irisphera.com` by default. Set another base URL in its configuration to use a test environment.
- Pin the generator version, as above, so that the generated code changes only when the contract does.

The walkthrough below uses curl, so that you can see every header and body. Your backend makes the same calls through the generated client.

## The walkthrough

The walkthrough connects one test store to Irisphera through the API. You send every request yourself from a terminal, in order. Each step ends with a checkpoint that tells you whether it worked before you move on.

By the end you will have:

- created a merchant organization for the store and imported two of its products;
- opened a session for a test shopper and recorded the shopper's privacy choices;
- generated a virtual try-on image, product recommendations and a mix-and-match suggestion;
- recorded a purchase of two units and the return of one;
- downloaded a report that links the purchase to the try-on;
- exported and erased the test shopper's data.

Use a test environment and test customers. The order requests only record what happened in your store. They never charge or refund a payment.

The walkthrough covers the routes that an integration needs, in the order it needs them. It is not a list of every endpoint. The OpenAPI contract has them all.

## Key terms

| Term | Meaning |
| --- | --- |
| Integrator | Your platform. It creates and manages merchant organizations with its integrator key. |
| Merchant organization | One store in Irisphera. It owns the store's catalog, shopper sessions, events and reports. Requests for the store use its merchant key. |
| Collection | A group of products in the store's catalog. You import every product into a collection. |
| SKU | `skuCustomId`, your ID for one product model in one color. Use the same SKU everywhere: in the catalog, for try-on, in events and in order lines. |
| Product feed | A JSON or CSV file that lists products. Irisphera can fetch it from a URL on your server (`import-from-url`), or you can upload it. |
| Channel | One storefront of the organization, identified by `channelId`. Irisphera creates a default channel the first time you ask for the collection context. |
| Collection context | Where the organization's shopper data goes: the channel, the namespaces and the privacy notice version. Here "collection" means collecting data, not a product collection. |
| Namespace | The kind of shopper ID. The organization has one namespace for anonymous browser IDs and one for signed-in customer IDs. |
| Shopper session | A session that your backend opens for one shopper. It gives you a 30-minute shopper token for the browser. |
| Subject | The person a session, event or privacy request is about: an anonymous browser, a signed-in customer, an Irisphera shopper ID or a guest order. |
| Scope | A permission in the shopper token, such as `shopper:vto` for try-on. It decides which routes the token can call. A scope is not consent. |
| Privacy choices | What the shopper allows Irisphera to record: `analytics`, `personalization` and `qaRecording`. |
| Event | A record of something that happened, such as a product view, a cart change, an order, a payment, a return or a refund. |
| Report | The organization's figures for a period, including the purchases that followed a try-on. |

## Credentials

| Credential | Header | Who holds it | Used for |
| --- | --- | --- | --- |
| Integrator API key | `INTEGRATOR-API-KEY` | Your platform backend | Creating and managing the merchant organizations you own |
| Merchant API key | `MERCHANT-API-KEY` | Your backend, one key per merchant organization | Catalog, collection context, shopper sessions, order events, reports, privacy requests |
| Shopper access token | `Authorization: Bearer <token>` | The shopper's browser, valid for 30 minutes | Try-on, sizing, recommendations, mix and match, storefront events, the shopper's own privacy choices |

API keys stay on your servers. Only the short-lived shopper token reaches the browser. Send exactly one credential per request.

## IDs

Every ID that your integration sends to Irisphera is a **UUIDv7**, written in lowercase. That covers event and image share IDs, privacy request IDs, `Idempotency-Key` headers, link operation IDs, anonymous browser IDs, and the IDs of your own records: customer, order, order line and refund IDs. If your platform numbers those records another way, store a UUIDv7 with each record the first time you send it to Irisphera, and send that UUIDv7 from then on. Generate each ID once, save it with its request, and send the same ID on every retry. SKUs are product codes, not IDs, and stay as they are.

| Language | UUIDv7 |
| --- | --- |
| Python 3.14 or later | `str(uuid.uuid7())` |
| Node.js | `v7()` from the [`uuid`](https://www.npmjs.com/package/uuid) package |
| Go | `uuid.NewV7()` from `github.com/google/uuid` |
| .NET 9 or later | `Guid.CreateVersion7().ToString()` |
| Java | A library such as [`uuid-creator`](https://github.com/f4b6a3/uuid-creator): `UuidCreator.getTimeOrderedEpoch()` |

In the walkthrough, the `uuid` helper from [step 1](docs/01-prepare.md#open-a-private-working-directory) prints a new one.

## Who calls what

```text
Your backend (API keys)                          Shopper's browser (shopper token)
─────────────────────────────────────────        ──────────────────────────────────────
integrator key
  └─ create the merchant organization    (2)
merchant key
  ├─ collection context                  (2)
  ├─ catalog: collections, products      (3)
  ├─ open a shopper session ── token ───────────►  privacy choices                   (4)
  │                                                try-on, 3D preview, image shares  (5)
  │                                                sizing and recommendations        (6)
  │                                                mix and match                     (7)
  │                                                product views, cart changes       (8)
  ├─ orders, payments, returns, refunds  (8)
  ├─ daily business statistics
  ├─ report                              (9)
  └─ privacy requests: export, erase    (10)
```

The numbers are the steps below.

## Steps

Run the steps in order in one Bash terminal. Every step uses values that the earlier steps saved in shell variables.

**Set up the store**

| Step | What you do | Checkpoint |
| --- | --- | --- |
| [1. Prepare](docs/01-prepare.md) | Get the environment URL and integrator key, open a private terminal, check the deployed contract | The contract lists the routes this guide uses |
| [2. Create the merchant organization](docs/02-create-merchant.md) | Create the organization, read its storefront configuration, get its collection context | You have the merchant ID, merchant key, channel and namespaces |
| [3. Import products](docs/03-ingest-products.md) | Create a collection and import the garment from your feed URL, from a feed file or as a single product | The SKU is listed and ready for try-on |

**Serve the shopper**

| Step | What you do | Checkpoint |
| --- | --- | --- |
| [4. Open a shopper session](docs/04-shopper-session.md) | Start an anonymous session, record the shopper's privacy choices, sign the shopper in | A customer token with the try-on and recommendation scopes |
| [5. Virtual try-on](docs/05-virtual-try-on.md) | Generate a try-on image, check for a 3D model, record an image share | A generated image on screen |
| [6. Recommendations and sizing](docs/06-recommendations.md) | Estimate measurements and colors, build the profile, get recommendations | Recommended SKUs with sizes |
| [7. Mix and match](docs/07-mix-and-match.md) | Import a product that completes the outfit, get a suggestion for each placement | A matching product for another placement |

**Measure and report**

| Step | What you do | Checkpoint |
| --- | --- | --- |
| [8. Storefront and order events](docs/08-collect-events.md) | Send the product view, the cart change, the order, the payment, the return and the refund | A receipt for every event, and a safe replay |
| [9. Download the report](docs/09-download-report.md) | Download the report and compare it with what you did | Orders, units, returns and attributed units match |

**Handle privacy requests**

| Step | What you do | Checkpoint |
| --- | --- | --- |
| [10. Privacy requests and offboarding](docs/10-privacy-requests-and-offboarding.md) | Export and erase the test shopper, end sessions, remove a merchant organization | Export received; erasures accepted and tracked |

## Reference pages

- [Privacy and consent requirements](docs/privacy-and-consent.md). Read them before you send any real shopper data. The walkthrough follows them, but they also cover what your storefront, consent banner and support process must do.
- [Daily business statistics](docs/business-statistics.md). Store-wide daily order totals without shopper data. `COMMERCE` statistics are available by default on every channel. The walkthrough does not send any.

## Arrange these before you start

- Irisphera gives you the environment URL and your integrator key, and turns on try-on and recommendation quota for the environment.
- Agree the merchant's privacy notice, processing instructions, retention and support contacts with Irisphera before real shoppers use the integration.
- Use one catalog SKU for each product everywhere. If your platform uses other product IDs, agree a SKU mapping with Irisphera. Importing a feed does not create aliases.
- Have real image URLs for one garment and for one product that goes with it, and a photo of a test participant who agreed to take part.
- To import from a feed URL, you need a server you control that Irisphera can reach over HTTPS. Without one, you can upload the feed as a file instead.

## If you integrated with an earlier version

- Data collection is v2 only. `collectionMode` is always `v2`. There is no legacy or dual collection, no comparison window, and a refused v2 event never falls back to another pipeline.
- Shopper features use `/shopper/v2/...` with the session token. The `/shopper/v1` routes are no longer served.
- The report is `POST /merchant/v2/report`. `/merchant/v1/report` is retired.
- Attribution starts from the purchase. For each purchased unit, Irisphera looks back a fixed number of days for a successful try-on of the same product by the same shopper. The report states the rule in `attributionPolicy`.
- Daily COMMERCE business statistics are available by default on every channel. COUNTERS still needs a separate approval.
- Image shares have their own route and report section.
- Data requests use `POST /merchant/v2/privacy/requests`.
- Mix and match is new: `GET /shopper/v2/mixmatch/{skuCustomId}` returns products that complete an outfit. See [step 7](docs/07-mix-and-match.md).
- An erasure holds back the affected daily business statistics until your source corrects them. See [step 10](docs/10-privacy-requests-and-offboarding.md#correct-daily-business-statistics).
- Importing from a feed URL is supported for feeds on a server you control. Earlier versions of this guide said not to use `import-from-url`. See [step 3](docs/03-ingest-products.md#method-a-import-from-a-feed-url).
- A feed import updates the SKUs whose data changed. Earlier versions of this guide said that SKUs already in the collection were skipped.

Start with [1. Prepare](docs/01-prepare.md).

## Frequently asked questions

### Do we need an SDK?

No. Every route takes and returns JSON over HTTPS, except the feed and photo uploads, which use multipart form data. Call the API with any HTTP client, or [generate a client](#generate-a-client) from the contract.

### Which ID format do we use?

UUIDv7, in lowercase, for every ID that you send to Irisphera: events, image shares, privacy requests, idempotency keys, link operations, anonymous browser IDs, and your customer, order, line and refund IDs. See [IDs](#ids).

### Which routes in the contract are for us?

The routes under `/integrator`, `/merchant` and `/shopper`. The routes under `/internal` are for Irisphera only. The contract also has a few merchant routes that the walkthrough does not use. Each route's description in the contract says when to use it.

### Can the storefront or a mobile app call Irisphera directly?

Yes, but only the `/shopper/v2` routes, with the shopper token. Your backend opens the session and hands the token to the browser or the app ([step 4](docs/04-shopper-session.md)). API keys never leave your servers. The API accepts browser requests from any origin, so you do not register your storefront domains.

### What does an error response look like?

Most errors have an `application/problem+json` body. Its fields include `type`, `title`, `status`, `detail` and `requestId`. Some routes, such as the import routes, answer with a status code and no body. Handle the status code first, then the `type` where a step names one, such as `session_expired`. Quote the `requestId` when you contact Irisphera about an error.

### Are there rate limits?

Try-on and recommendations are limited by the organization's quota ([step 2](docs/02-create-merchant.md#how-much-quota-does-the-organization-have-and-what-uses-it)). The contract also lists `429 Too Many Requests` for the event routes. When you get `429` or `503`, send the same request again later, and wait at least as long as `Retry-After` says.

### Can we test without touching production?

Ask Irisphera for a test environment. On production, the walkthrough creates a real merchant organization and uses one try-on and one recommendation of its quota. [Step 10](docs/10-privacy-requests-and-offboarding.md#remove-a-merchant-organization) shows how to remove the organization afterwards.

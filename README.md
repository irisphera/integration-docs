# Irisphera: from API key to merchant report

A step-by-step implementation demo for an enterprise integration call. Follow one merchant, one garment, and one shopper through onboarding, virtual try-on, a two-unit purchase, a one-unit return, and a downloaded report.

This is a hands-on API walkthrough, not an endpoint catalog or a platform-plugin installation guide. Each step gives the request, the values to retain, and a checkpoint before continuing.

## Follow these six steps

| Step | What you do | What you can show |
| --- | --- | --- |
| [1. Prepare](docs/01-prepare.md) | Start with an integrator key and agreed demo environment | Prerequisites and private working terminal |
| [2. Create a merchant](docs/02-create-merchant.md) | Create the account; retain merchant credentials; obtain the channel handoff | Merchant ID and name |
| [3. Upload a product feed](docs/03-upload-feed.md) | Apply the feed rules, upload JSON, wait for the SKU | Product visible in its collection |
| [4. Use Irisphera](docs/04-use-irisphera.md) | Establish the shopper session, sign in, generate virtual try-on | Actual generated image |
| [5. Collect activity and orders](docs/05-collect-data.md) | Wire views, cart changes, orders, payments, returns and refunds | Accepted events and an identical replay |
| [6. Download a report](docs/06-download-report.md) | Save the JSON report and reconcile it against the demo | Orders, purchased units, returns and attribution |

Run the commands in order in the same Bash terminal. IDs and credentials returned by one step are used by the next. Use a test environment: the commerce calls record test facts; they do not charge or refund a payment.

## Arrange these handoffs before presenting

- Irisphera supplies the environment and integrator key, enables the required features, and has catalog/VTO processing available.
- After merchant creation, Irisphera provisions the channel credential and anonymous/customer namespaces. **There is no public channel-provisioning API.**
- Use the same registered channel and catalog SKU for VTO, observations, and order lines. If platform identifiers differ, arrange their canonical SKU mapping before the call; uploading a feed does not create aliases.
- Bring real garment image URLs and a consented shopper photo. Sign in before the demo VTO so the VTO and order use the same customer session.

The guide describes the checked-in Octopus contract and implementation. Confirm the deployed environment before the call; the commands are not a claim of verified production availability. Full endpoint schemas remain in that environment's `/v3/api-docs` rather than being duplicated here.

Start with [1. Prepare the demo](docs/01-prepare.md).

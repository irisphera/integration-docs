# 1. Prepare

[Integration guide](../README.md) · Next: [Create the merchant organization](02-create-merchant.md)

**Goal:** get an integrator API key and an environment, open a private terminal, and check that the environment serves the routes in this guide.

## What you need from Irisphera

- **Environment URL**, for example `https://api.irisphera.com` for production or a test environment agreed with Irisphera.
- **Integrator API key.** It identifies your platform. With it you create and manage the merchant organizations you own, and nothing else: it cannot call catalog, shopper or report routes. Keep it in your secret manager.
- **Quota** for virtual try-on and recommendations on the merchant organizations you will create. Without quota the shopper token does not get those feature scopes (step 4).

## What to prepare

- One real garment: front and back still-life image URLs, a featured image URL and its product-page URL. Irisphera must be able to reach the image URLs while it processes the product.
- A JPEG, PNG, WebP or AVIF photo of a test participant who has agreed to take part, for try-on and sizing. Check suitability with Irisphera first.
- A test merchant and test checkout or customer records. This walkthrough records a two-unit purchase and a one-unit return; it does not charge a card or refund one.
- Bash, curl 7.76 or later (for `--fail-with-body`), jq and Python 3. Run every command in the **same Bash terminal**. Do not enable shell tracing or display secret response files while you share your screen.
- The [privacy prerequisites](privacy-and-consent.md#before-sending-any-real-shopper-data). Explain the real photo, server and provider processing, and the outcome recording. Obtain the permissions the notice needs, and agree retention and rights handling. Use consenting test participants and test identities. Do not promise browser-only storage or immediate deletion.

## Open a private working directory

```bash
set -euo pipefail
umask 077
uuid() {
  python3 -c 'import secrets,time,uuid
print(uuid.UUID(int=(int(time.time()*1000)<<80)|(7<<76)|(secrets.randbits(12)<<64)|(2<<62)|secrets.randbits(62)))'
}
now() { python3 -c 'from datetime import datetime, timezone; print(datetime.now(timezone.utc).isoformat().replace("+00:00", "Z"))'; }
RUN_ID=$(uuid)
DEMO_DIR=$(mktemp -d)
cd "$DEMO_DIR"
REPORT_START=$(now)
read -rp 'Irisphera environment URL: ' IRISPHERA_BASE_URL
IRISPHERA_BASE_URL=${IRISPHERA_BASE_URL%/}
read -rsp 'Integrator API key: ' INTEGRATOR_API_KEY; printf '\n'
printf 'Private demo files: %s\n' "$DEMO_DIR"
```

`uuid` makes UUIDv7 identifiers, which the event routes require. Use HTTPS. These terminal calls stand in for your **trusted backend**. API keys and anonymous-continuation secrets must never be shipped in storefront JavaScript. Only the short-lived shopper token goes to the browser. Send exactly one credential per request, as each step shows.

## Check the deployed contract

```bash
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/v3/api-docs" -o openapi.json
jq -e '.paths["/integrator/v1/merchant"].post
  and .paths["/merchant/v2/collection-context"].post
  and .paths["/merchant/v1/collection/{collectionId}/file"].post
  and .paths["/merchant/v2/shopper-sessions"].post
  and .paths["/shopper/v2/privacy/preferences"].put
  and .paths["/shopper/v2/stylist-preview"].post
  and .paths["/shopper/v2/recommendations"].post
  and .paths["/shopper/v2/image-shares/{sourceEventId}"].put
  and .paths["/merchant/v2/commerce-events/{sourceEventId}"].put
  and .paths["/merchant/v2/report"].post
  and .paths["/merchant/v2/privacy/requests"].post' openapi.json >/dev/null
```

If the environment does not publish its API documentation, ask Irisphera for the deployed contract. Do not assume that an older environment serves the routes in this guide.

**Checkpoint:** the environment URL and integrator key are in the terminal, the contract check passes, and the garment and photo are ready. Keep this terminal open and continue to [step 2](02-create-merchant.md).

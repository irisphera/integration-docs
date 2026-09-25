# 1. Prepare

[Integration guide](../README.md) · Next: [Create the merchant organization](02-create-merchant.md)

**Goal:** get an integrator API key and an environment, open a private terminal, and check that the environment serves the routes in this guide.

## What you need from Irisphera

- **Environment URL**, for example `https://api.irisphera.com` for production or a test environment agreed with Irisphera.
- **Integrator API key.** It identifies your platform. With it you create and manage the merchant organizations you own, and nothing else: it cannot call catalog, shopper or report routes. Keep it in your secret manager.
- **Quota** for virtual try-on and recommendations on the merchant organizations you will create. Without quota the shopper token does not get those feature scopes (step 4).

## What to prepare

- One real garment, and one product that completes an outfit with it, such as trousers for a blazer. For each: its product image URLs, main product photo first, and its product-page URL. Irisphera must be able to reach the image URLs while it processes the products.
- To import the products from a feed URL (the first method in [step 3](03-ingest-products.md#choose-how-to-send-products)): a server you control where you can publish a small JSON file at an HTTPS URL that Irisphera can reach. Without one, step 3 also shows how to upload the file or send single products.
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

What this block does:

- `set -euo pipefail` ends the Bash session at the first failed command, so you never carry on with a missing value. The shell variables are lost when that happens. Type `bash` and press Enter before you paste the block. When that inner session ends, you are back in your terminal with the error still on screen. Fix the cause, and start again from this block. `umask 077` keeps the files you save readable only by you.
- `uuid` prints a new UUIDv7. Use a UUIDv7 for every ID that you send to Irisphera, in the walkthrough and in production ([IDs](../README.md#ids)). `now` prints the current UTC time in the format the API expects.
- `DEMO_DIR` is a new private folder. Every file the walkthrough saves goes there, and you delete it at the end of [step 10](10-privacy-requests-and-offboarding.md#clean-up).
- `REPORT_START` marks the start of the period you report on in [step 9](09-download-report.md).
- The two prompts read the environment URL and your integrator key. The key is not shown as you type it.

Use HTTPS. These terminal calls stand in for your **trusted backend**. Never ship API keys or anonymous-continuation secrets in storefront JavaScript. Only the short-lived shopper token goes to the browser. Send exactly one credential per request, as each step shows.

## Check the deployed contract

```bash
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/v3/api-docs" -o openapi.json
jq -e '.paths["/integrator/v1/merchant"].post
  and .paths["/merchant/v2/collection-context"].post
  and .paths["/merchant/v1/collection"].post
  and .paths["/merchant/v1/collection/{collectionId}/import-from-url"].post
  and .paths["/merchant/v1/collection/{collectionId}/file"].post
  and .paths["/merchant/v2/shopper-sessions"].post
  and .paths["/shopper/v2/privacy/preferences"].put
  and .paths["/shopper/v2/stylist-preview"].post
  and .paths["/shopper/v2/recommendations"].post
  and .paths["/shopper/v2/mixmatch/{skuCustomId}"].get
  and .paths["/shopper/v2/image-shares/{sourceEventId}"].put
  and .paths["/merchant/v2/commerce-events/{sourceEventId}"].put
  and .paths["/merchant/v2/report"].post
  and .paths["/merchant/v2/privacy/requests"].post' openapi.json >/dev/null
```

The command prints nothing when every route is there. If one is missing, `jq` fails and the Bash session ends. If the environment does not publish its API documentation, ask Irisphera for the deployed contract. Do not assume that an older environment serves the routes in this guide.

**Checkpoint:** the environment URL and integrator key are in the terminal, the contract check passes, and the garment and photo are ready. Keep this terminal open and continue to [step 2](02-create-merchant.md).

## Frequently asked questions

### Can we use a generated client or an API tool instead of curl?

Yes. Each step shows the method, route, headers and body of every call, so you can make the same calls with a [generated client](../README.md#generate-a-client) or a tool such as Postman. Keep the steps in order, and reuse the values that earlier steps saved.

### Which systems can run the commands?

Any system with Bash, curl 7.76 or later, jq and Python 3, such as Linux or macOS. On Windows, use WSL.

### A command failed and my terminal session ended. Do I have to start over?

The files are still in the folder that step 1 printed, but the shell variables are gone. The simplest fix is to start again from step 1. A new run gets a new `RUN_ID`, so its names never clash with the earlier run. Remove the earlier test organization when you no longer need it ([step 10](10-privacy-requests-and-offboarding.md#remove-a-merchant-organization)).

### Does the walkthrough use quota or move money?

It uses one try-on and one recommendation from the new organization's quota. The order events in step 8 only record a test order. Nothing charges or refunds a card.

### The contract check failed. Can we skip the missing route?

No. The environment does not serve a route that the walkthrough uses, so a later step would fail. Ask Irisphera which environment serves the current API.

# 1. Prepare the demo

[Call agenda](../README.md) · Next: [Create a merchant](02-create-merchant.md)

**Goal:** start with an integrator API key and an agreed demo environment.

## Arrange before the call

- Irisphera supplies the environment URL and integrator API key. Confirm that this environment supports the v2 session and event APIs, has active catalog workers, and has virtual try-on quota and providers available.
- Have Irisphera available to provision a channel for the merchant created in step 2. **The public API does not provision channels or their credentials.** Merchant creation alone is not enough to run steps 4–5.
- Use the same registered channel and catalog SKU throughout. If platform product identifiers differ, arrange canonical SKU aliases with Irisphera; feed upload does not create aliases and there is no public product-alias management API.
- Prepare one real garment: front and back still-life image URLs, its product-page URL, and a consenting participant's JPEG or PNG portrait on the demo machine. The image URLs must remain reachable by Irisphera during processing. Check suitability with Irisphera before the call.
- Use a test merchant and test checkout/customer records. This walkthrough records a two-unit purchase and a one-unit return; it does not charge a card or execute a refund in your commerce platform.
- Install Bash, curl 7.76+ (`--fail-with-body`), jq, and Python 3. Run all commands in the **same Bash terminal**. Do not enable shell tracing or display secret response files while screen-sharing.
- Complete the [privacy prerequisites](07-privacy-and-consent.md#before-sending-any-real-shopper-data): explain the actual photo/server/provider processing and outcome recording, obtain the required permission, and agree retention and downstream rights handling. Use consenting test participants and test identities; do not promise browser-only storage or immediate deletion. Production activation also requires the approved processing/DPA and supplier arrangements.

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

Use HTTPS for the approved remote environment. These terminal calls stand in for your **trusted backend**. API keys and anonymous-continuation secrets must never be shipped in storefront JavaScript. Only the short-lived shopper bearer token goes to the browser. Send exactly one credential per request, as shown in each step.

## Check the deployed contract

```bash
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/v3/api-docs" -o openapi.json
jq -e '.paths["/integrator/v1/merchant"].post
  and .paths["/merchant/v2/shopper-sessions"].post
  and .paths["/merchant/v2/commerce-events/{sourceEventId}"].put
  and .paths["/merchant/v1/report"].post' openapi.json >/dev/null
```

If API documentation is not exposed publicly, obtain the deployed contract from Irisphera instead. Do not assume that an older environment has the endpoints in this guide.

**Checkpoint:** environment and integrator key ready; demo assets prepared; Irisphera's channel-provisioning handoff agreed. Keep this terminal open and continue to [step 2](02-create-merchant.md).

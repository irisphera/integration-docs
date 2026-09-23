# 5. Virtual try-on

Previous: [Open a shopper session](04-shopper-session.md) · [Integration guide](../README.md) · Next: [Recommendations and sizing](06-recommendations.md)

**Goal:** check that the product can be tried on, generate a try-on image from the test participant's photo, check for a 3D model, and record that the shopper saved the image.

The browser calls these routes with the shopper token. Try-on and 3D preview need the `shopper:vto` scope from [step 4](04-shopper-session.md#check-the-session). Image shares need `shopper:events`.

## Check that the product can be tried on

```bash
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/shopper/v2/stylist-preview/$SKU" \
  -H "Authorization: Bearer $ACCESS_TOKEN" -o readiness.json
VTO_READINESS=$(jq -er . readiness.json)
printf 'Try-on readiness: %s\n' "$VTO_READINESS"
```

**Expected:** a placement such as `UPPER`, `LOWER` or `FULL`, the same value as the merchant read in [step 3](03-ingest-products.md#confirm-try-on-readiness). The response is a JSON string. `UNAVAILABLE` means you must not request a try-on for this SKU: hide the button. Readiness does not check the shopper's photo and does not guarantee that a try-on succeeds.

For a try-on gallery, `GET /shopper/v2/stylist-preview` lists every product that is ready, grouped by collection. It reads no shopper profile, so it returns no silhouette, palette or sizing.

## Open the product page

The shopper opens the product page before trying the garment on. Keep the time: [step 8](08-collect-events.md) sends it as the `PRODUCT_VIEWED` event. In a real storefront, send that event when the page is displayed.

```bash
PRODUCT_VIEWED_AT=$(now)
```

## Generate the try-on image

Use the test participant's photo. Irisphera accepts PNG, JPEG, WebP and AVIF and detects the format from the file content.

```bash
read -rp 'Path of the test participant photo: ' PHOTO_PATH
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/shopper/v2/stylist-preview" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  --form-string "skuCustomId=$SKU" \
  -F "targetImage=@$PHOTO_PATH" \
  -F 'isFaceBlurred=false' \
  -o try-on.json
python3 -c 'import base64,json; from pathlib import Path
data=json.loads(Path("try-on.json").read_text())
Path("try-on.webp").write_bytes(base64.b64decode(data["generatedImage"],validate=True))'
jq '{productPageUrl, hasFeaturedImage:(.featuredImage != null), hasResultUrl:(.resultUrl != null)}' try-on.json
```

**Expected:** HTTP `200`, and `try-on.webp` shows the participant wearing the garment. Open it to check. A try-on can take several seconds, so show progress in your storefront.

| Response field | Meaning |
| --- | --- |
| `generatedImage` | The result as base64-encoded WebP. Display it directly. |
| `productPageUrl` | The product page from the catalog, when one is stored |
| `featuredImage` | A temporary URL of the product's featured image, when one is stored |
| `resultUrl` | A URL of the stored result, valid for 24 hours. Absent when the result could not be stored. Do not keep it longer. |

| Error | Meaning | Action |
| --- | --- | --- |
| `420` | The photo cannot be used for try-on | Ask the shopper for another photo |
| `422` | The image could not be processed | Check the file and try another photo |
| `404` | Unknown SKU, or not in this organization's catalog | Check the SKU against step 3 |
| `403` | The token has no `shopper:vto` scope | Hide the try-on. The scope is granted only while the organization has try-on quota. |

`isFaceBlurred` tells Irisphera that your storefront already blurred the face before upload. It does not ask Irisphera to blur anything. If your notice promises blurring, blur the photo in the browser and send `true`.

### What Irisphera records

Irisphera records the outcome of every try-on itself: success, failure, or unusable input. It records it only when the shopper grants `analytics` at that moment. Without analytics, the try-on still works and nothing is recorded. **Do not send a `VIRTUAL_TRY_ON` event** for the same attempt: it would count twice. The report counts successful, failed and unusable-input try-ons separately; only a successful one can take part in purchase attribution. Quota is used only when a try-on succeeds. A token keeps its scopes until it expires, even if quota runs out meanwhile, so check `isVtoEnabled` ([step 2](02-create-merchant.md#manage-your-organizations)) before you offer the try-on.

### Other try-on routes

| Route | Difference |
| --- | --- |
| `POST /shopper/v2/stylist-preview-background` | Same fields plus `backgroundId`, the index of a background image agreed with Irisphera. Without it, Irisphera picks a background that suits the garment. |
| `POST /shopper/v2/stylist-preview-collection` | `collectionId` instead of `skuCustomId`, and an optional `gender`. Irisphera picks a ready product from the collection. A collection with no ready product returns `404`. |

Both return the same response and record their outcome in the same way.

## Check for a 3D model

```bash
curl -sS "$IRISPHERA_BASE_URL/shopper/v2/td-preview/$SKU" \
  -H "Authorization: Bearer $ACCESS_TOKEN" -o td-preview.json -w 'HTTP %{http_code}\n'
```

**Expected:** `200` with a JSON string holding a temporary download URL of the product's 3D model, or `404` or `422` when the product has no 3D model. The demo garment usually has none, so either answer is fine here. Do not store the URL. When your storefront actually displays the model, send a `TD_PREVIEW` event (step 8). Checking availability is not a display.

## Record an image share

When the shopper shares or saves a try-on image, the device that does it sends an image share. The walkthrough stands in for the shopper pressing your **Save** button:

```bash
SHARE_EVENT_ID=$(uuid)
jq -n --arg channel "$CHANNEL_ID" --arg at "$(now)" --arg sku "$SKU" \
  '{schemaVersion:2,channelId:$channel,occurredAt:$at,product:{skuCustomId:$sku},
    share:{origin:"SAME_DEVICE",destination:"SAVE_TO_DEVICE"}}' > "$SHARE_EVENT_ID.json"
curl --fail-with-body -sS -X PUT "$IRISPHERA_BASE_URL/shopper/v2/image-shares/$SHARE_EVENT_ID" \
  -H "Authorization: Bearer $ACCESS_TOKEN" -H 'Content-Type: application/json' \
  --data-binary "@$SHARE_EVENT_ID.json" -o "$SHARE_EVENT_ID-receipt.json" -w 'HTTP %{http_code}\n'
jq '{receivedAt}' "$SHARE_EVENT_ID-receipt.json"
```

**Expected:** HTTP `201` with `receivedAt`. Resending the same saved body returns `200`. The same ID with a different body returns `409`.

| Field | Values |
| --- | --- |
| `share.origin` | `SAME_DEVICE`, or `QR_CODE` when the shopper scanned a QR code to reach the image on another device |
| `share.destination` | `INSTAGRAM`, `SNAPCHAT`, `WHATSAPP`, `TIKTOK`, `FACEBOOK`, `MESSENGER`, `PINTEREST`, `X`, `TELEGRAM`, `EMAIL`, `SMS`, `COPY_LINK`, `SAVE_TO_DEVICE`, `SYSTEM_SHARE_SHEET` or `OTHER` |

- Send the share when it happens, with a new UUIDv7 ID that the device keeps for retries.
- The event holds no image, recipient, account name or message. Send none of them.
- A QR code must not carry a shopper token, a shopper ID or any other identity. The scanning device opens its own session and sends the share with its own token.
- A `403` means the token lacks `shopper:events`, or the shopper did not grant `analytics` at `occurredAt`. Drop the event; do not retry it.
- Shares appear in the report's `imageShares` section. They never affect purchase attribution.

## Test try-on as a merchant

This is optional; skip it if you have no local garment image. To test a garment image before importing it, your backend can call the one-shot merchant route. It needs no SKU, no shopper session and no consent record:

```bash
read -rp 'Path of a garment image: ' GARMENT_PATH
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/merchant/v1/vto2d" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" \
  -F "featuredImage=@$GARMENT_PATH" -F "userPhoto=@$PHOTO_PATH" \
  --form-string 'description=blazer' -o one-shot.json
jq '{hasImage:(.generatedImage != null)}' one-shot.json
```

It returns the same image format, records nothing and uses no quota. It is for testing only; shoppers use the routes above.

## Photo handling

The photo goes to Irisphera's servers and processing providers, and Irisphera stores the generated result. Your notice must say so. Read [photo and recording handling](privacy-and-consent.md#photo-and-recording-handling) before real shoppers upload photos.

**Checkpoint:** `try-on.webp` shows the garment on the participant, and the image share returned `201`. Keep `PRODUCT_VIEWED_AT` and `PHOTO_PATH`. Continue to [step 6](06-recommendations.md).

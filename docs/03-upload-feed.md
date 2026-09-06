# 3. Upload a product feed

Previous: [Create a merchant](02-create-merchant.md) · [Call agenda](../README.md) · Next: [Use Irisphera](04-use-irisphera.md)

**Goal:** import one garment and confirm that its SKU is available before demonstrating a shopper experience.

## Apply these feed rules

This demo uses **bulk JSON**, not the differently shaped single-product API.

| Field | Rule |
| --- | --- |
| `products` | Top-level array inside an object: `{"products":[...]}` |
| `skuCustomId` | Stable string identifying a model-and-color combination; use the same value in experience and event calls. Map size variants to it; preserve their variant/line IDs separately in commerce events. |
| `title`, `description` | Actual product title and plain-text description |
| `gender` | `MEN`, `WOMEN`, `UNISEX`, `CHILDREN_GIRL`, or `CHILDREN_BOY` |
| `product_front_image`, `product_back_image` | Still-life front/back views for virtual try-on; supply both |
| `product_images` | Nonempty array of image URLs; include the front/back images here too |
| `product_featured_image` | Featured image for the collection |
| `product_page_url` | The product's real storefront URL |

Use reachable HTTPS images containing PNG, JPEG, WebP, or AVIF bytes. URLs must not require a shopper login or expire during asynchronous processing. Use one row per catalog SKU: later duplicates in a feed are filtered, and existing collection items are skipped by default. **Do not treat another upload as an upsert.** Purchase prices belong in commerce events; do not rely on feed prices for financial reporting.

CSV is also supported, but is not used in this walkthrough. Its headers are camelCase (`productFrontImage`, `productBackImage`, `productImages`, `productFeaturedImage`, `productPageUrl`), with `|` between `productImages` URLs. Do not reuse those headers in bulk JSON. XLSX is not supported.

## Create the collection

```bash
jq -n --arg title "Demo catalog $RUN_ID" '{title:$title}' > collection-request.json
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/merchant/v1/collection" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" \
  -H 'Content-Type: application/json' \
  --data-binary @collection-request.json -o collection.json
COLLECTION_ID=$(jq -er '.id' collection.json)
jq '{id,merchantId,title}' collection.json
```

**Expected:** HTTP `201`; `merchantId` matches `MERCHANT_ID`. Retain the returned `COLLECTION_ID`; creating a collection again creates another collection.

## Build and upload the feed

Use the prepared garment's real URLs. Adjust the example title, description, and gender to match it.

```bash
SKU='DEMO-BLAZER-BLACK'
read -rp 'Public front image URL: ' FRONT_IMAGE_URL
read -rp 'Public back image URL: ' BACK_IMAGE_URL
read -rp 'Product page URL: ' PRODUCT_PAGE_URL
jq -n --arg sku "$SKU" --arg front "$FRONT_IMAGE_URL" \
  --arg back "$BACK_IMAGE_URL" --arg page "$PRODUCT_PAGE_URL" \
  '{products:[{skuCustomId:$sku, title:"Black wool blazer",
    description:"Single-breasted wool blazer.", gender:"WOMEN",
    product_front_image:$front, product_back_image:$back,
    product_images:[$front,$back], product_featured_image:$front,
    product_page_url:$page}]}' > products.json
curl --fail-with-body -sS \
  "$IRISPHERA_BASE_URL/merchant/v1/collection/$COLLECTION_ID/file?useSeasonFiltering=false" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" \
  -F 'file=@products.json;type=application/json'
```

**Expected:** HTTP `202`, but this is **not proof that the product was imported**. Processing is asynchronous, and the current upload controller can also return `202` after a submission failure. We use file upload rather than URL import to avoid forwarding API credentials to a feed host.

## Confirm the SKU is visible

Repeat this read while processing completes:

```bash
curl --fail-with-body -sS \
  "$IRISPHERA_BASE_URL/merchant/v1/collection/$COLLECTION_ID/products" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" -o catalog.json
if jq -e --arg sku "$SKU" 'any(.fashionItems[]; .skuCustomId == $sku)' catalog.json >/dev/null; then
  printf 'SKU is visible; continue to step 4.\n'
else
  printf 'STOP: SKU is not visible yet. Wait and repeat this read.\n'
fi
```

There is no public import-job status endpoint or completion deadline. If the SKU stays absent, have Irisphera inspect the import before continuing; an empty catalog is not a successful demo.

Retain this exact `SKU` for the v2 session's VTO, observations, and order line. Their shared channel and SKU form the product attribution key when no canonical alias exists. Different platform identifiers or channels require an agreed canonical mapping; feed upload does not create aliases.

**Checkpoint:** `SKU` is present in `catalog.json`. Catalog visibility and virtual try-on readiness are separate checks; step 4 checks the latter.

# 3. Ingest products

Previous: [Create the merchant organization](02-create-merchant.md) · [Integration guide](../README.md) · Next: [Open a shopper session](04-shopper-session.md)

**Goal:** import one garment into a collection, confirm that its SKU is listed, and confirm that it is ready for virtual try-on.

Products live in collections. Irisphera processes each imported product in the background: it stores the images, detects the garment's features for recommendations, and prepares it for try-on. Every import route answers before processing finishes, so always confirm the result by reading the catalog.

## Choose how to send products

| Route | Use it for | Body |
| --- | --- | --- |
| `POST /merchant/v1/collection/{collectionId}/file` | A whole catalog or a batch | Multipart `file`: a JSON feed (`application/json`) or a CSV feed (`text/csv`) |
| `POST /merchant/v1/collection/{collectionId}/products` | One product at a time, for example from a rate-limited source or a product-created webhook | JSON, one product; images may be URLs or data URIs |

Do not use `import-from-url`: it forwards the request headers, including your merchant key, to the feed host.

## Apply these product rules

| Field | Rule |
| --- | --- |
| SKU | Required. A stable ID for one model in one color. Use the same value in try-on, events and order lines. Map size variants to it, and keep their variant or line IDs separately in commerce events. |
| Title, description | Required. The real product title and a plain-text description. |
| Gender | `MEN`, `WOMEN`, `UNISEX`, `CHILDREN_GIRL` or `CHILDREN_BOY` |
| Front and back images | Still-life front and back views. Required for virtual try-on. |
| Product images | Required, not empty. Used for feature detection and recommendations. Include the front and back images here too. |
| Featured image, product page URL | Required for the product to appear in collection views and recommendation results |
| Price | Optional. One whole string with its currency, for example `"49.99 EUR"`, returned with the product as sent. Do not rely on it for financial reporting: order amounts come from commerce events. |

Images must be PNG, JPEG, WebP or AVIF. Image URLs must stay reachable without a shopper login and must not expire while Irisphera processes the product.

The field names differ by format:

| Format | SKU | Images and page |
| --- | --- | --- |
| JSON feed: `{"products":[...]}` | `skuCustomId` | snake_case: `product_front_image`, `product_back_image`, `product_images` (array), `product_featured_image`, `product_page_url` |
| CSV feed | `skuCustomId` | camelCase headers: `productFrontImage`, `productBackImage`, `productImages` (URLs separated by `\|`), `productFeaturedImage`, `productPageUrl` |
| Single-product API | `skuCustomId` | camelCase: `productFrontImage`, `productBackImage`, `productImages` (array), `productFeaturedImage`, `productPageUrl` |

All three formats use `title`, `description`, `gender` and `price`. XLSX is not supported.

An import is not an upsert. Later duplicates in one feed are dropped, and SKUs that already exist in the collection are skipped.

## Create a collection

```bash
jq -n --arg title "Demo catalog $RUN_ID" '{title:$title}' > collection-request.json
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/merchant/v1/collection" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" \
  -H 'Content-Type: application/json' \
  --data-binary @collection-request.json -o collection.json
COLLECTION_ID=$(jq -er '.id' collection.json)
jq '{id,merchantId,title}' collection.json
```

**Expected:** HTTP `201`, and `merchantId` equals `MERCHANT_ID`. Keep `COLLECTION_ID`: sending the request again creates a second collection. `GET /merchant/v1/collection` lists your collections. Do not use `PUT` on a collection: it currently creates a new collection instead of updating the existing one.

## Import the product

Use the prepared garment's real URLs. Change the example title, description, gender and price to match it.

```bash
SKU='DEMO-BLAZER-BLACK'
read -rp 'Front image URL: ' FRONT_IMAGE_URL
read -rp 'Back image URL: ' BACK_IMAGE_URL
read -rp 'Featured image URL: ' FEATURED_IMAGE_URL
read -rp 'Product page URL: ' PRODUCT_PAGE_URL
```

Then run **one** of the two options.

**Option A: JSON feed.** Use this for catalogs and batches.

```bash
jq -n --arg sku "$SKU" --arg front "$FRONT_IMAGE_URL" --arg back "$BACK_IMAGE_URL" \
  --arg featured "$FEATURED_IMAGE_URL" --arg page "$PRODUCT_PAGE_URL" \
  '{products:[{skuCustomId:$sku, title:"Black wool blazer",
    description:"Single-breasted wool blazer.", gender:"WOMEN", price:"99.00 EUR",
    product_front_image:$front, product_back_image:$back,
    product_images:[$front,$back], product_featured_image:$featured,
    product_page_url:$page}]}' > products.json
curl --fail-with-body -sS \
  "$IRISPHERA_BASE_URL/merchant/v1/collection/$COLLECTION_ID/file?useSeasonFiltering=false" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" \
  -F 'file=@products.json;type=application/json'
```

**Option B: single-product API.** Use this when your source sends one product at a time. The fields are camelCase, and an image can also be a data URI such as `data:image/webp;base64,...` when your source cannot publish image URLs.

```bash
jq -n --arg sku "$SKU" --arg front "$FRONT_IMAGE_URL" --arg back "$BACK_IMAGE_URL" \
  --arg featured "$FEATURED_IMAGE_URL" --arg page "$PRODUCT_PAGE_URL" \
  '{skuCustomId:$sku, title:"Black wool blazer",
    description:"Single-breasted wool blazer.", gender:"WOMEN", price:"99.00 EUR",
    productFrontImage:$front, productBackImage:$back,
    productImages:[$front,$back], productFeaturedImage:$featured,
    productPageUrl:$page}' > product.json
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/merchant/v1/collection/$COLLECTION_ID/products" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" \
  -H 'Content-Type: application/json' \
  --data-binary @product.json
```

**Expected:** HTTP `202` for either option. This does **not** prove the product was imported: processing continues in the background, and the feed route can also return `202` when nothing was queued. For a feed, `useSeasonFiltering=true` applies the active seasonal policy.

A `402` means the organization's metered quota is used up. `GET /merchant/v1/subscription` shows the `limit` and `current` usage of `featureDetection`, `shopperRecommendation` and `shopper2dPreview`. Usage changes concurrently, so still handle `402` on every call.

## Confirm the SKU is listed

Repeat this read until the SKU appears:

```bash
curl --fail-with-body -sS \
  "$IRISPHERA_BASE_URL/merchant/v1/collection/$COLLECTION_ID/products" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" -o catalog.json
if jq -e --arg sku "$SKU" 'any(.fashionItems[]; .skuCustomId == $sku)' catalog.json >/dev/null; then
  jq --arg sku "$SKU" '.fashionItems[] | select(.skuCustomId == $sku)
    | {skuCustomId, title:.data.title, placement:.data.placement, category:.data.category, price:.data.price}' catalog.json
else
  printf 'STOP: the SKU is not listed yet. Wait and repeat this read.\n'
fi
```

**Expected:** the SKU with its detected `placement` (for example `UPPER`) and `category`. There is no import-status endpoint and no completion deadline. If the SKU stays absent, ask Irisphera to check the import; an empty catalog is not a working integration. `GET /merchant/v1/products` lists products across all your collections.

## Confirm try-on readiness

Being listed and being ready for try-on are separate. Your backend can check readiness with the merchant key, without a shopper session:

```bash
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/merchant/v1/storefront/products/$SKU" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" -o product-availability.json
jq '{vtoReadiness, has3dPreview:(.tdPreviewGlbUrl != null)}' product-availability.json
```

**Expected:** `vtoReadiness` is a placement such as `UPPER`, `LOWER` or `FULL`. `UNAVAILABLE` means the product cannot be tried on: check its front and back images, or wait for processing. `tdPreviewGlbUrl`, when present, is a temporary URL of the product's 3D model. An unknown SKU returns `404`. This read uses no quota, needs no consent and records nothing, so your product page can use it to decide whether to show the try-on button.

Keep this exact `SKU` for try-on, events and the order line. The channel and SKU together identify the product in attribution. If your platform uses different product IDs or channels, agree a canonical mapping with Irisphera; a feed does not create aliases.

**Checkpoint:** `SKU` is listed in `catalog.json` and `vtoReadiness` is not `UNAVAILABLE`. Continue to [step 4](04-shopper-session.md).

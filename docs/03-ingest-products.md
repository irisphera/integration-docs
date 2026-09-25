# 3. Import products

Previous: [Create the merchant organization](02-create-merchant.md) · [Integration guide](../README.md) · Next: [Open a shopper session](04-shopper-session.md)

**Goal:** import one garment into a collection, confirm that its SKU is listed, and confirm that it is ready for virtual try-on.

Products live in collections. When you import a product, Irisphera works on it in the background: it stores the images, detects the garment's features for recommendations and prepares it for try-on. Every import route answers before that work is done, so you always confirm the result by reading the catalog.

## Choose how to send products

| Method | Route | Use it when |
| --- | --- | --- |
| A. [Import from a feed URL](#method-a-import-from-a-feed-url) | `POST /merchant/v1/collection/{collectionId}/import-from-url` | Your platform can publish the catalog as a JSON or CSV feed on a server you control. You send one short request and Irisphera downloads the feed itself. To pick up catalog changes, publish the new feed and send the request again. |
| B. [Upload the feed file](#method-b-upload-the-feed-file) | `POST /merchant/v1/collection/{collectionId}/file` | You have the feed as a file but cannot publish it on your own server, or the feed must not be public |
| C. [Send one product](#method-c-send-one-product) | `POST /merchant/v1/collection/{collectionId}/products` | Your source sends products one at a time, for example from a product-created webhook or a rate-limited source |

Use method A for a whole catalog when you can. This page shows all three. Run only one of them for the walkthrough.

## Product rules

| Field | Rule |
| --- | --- |
| SKU | Required. A stable ID for one model in one color. Use the same value in try-on, events and order lines. Map size variants to it, and keep their variant or line IDs separately in order events. |
| Title, description | Required. The real product title and a plain-text description. |
| Gender | Optional. `MEN`, `WOMEN`, `UNISEX`, `CHILDREN_GIRL` or `CHILDREN_BOY`. If you leave it out, Irisphera detects it from the images. |
| Product images | Required, at least one. The first image becomes the product's featured image, so put the main product photo first. Irisphera uses the images for feature detection, recommendations and try-on. |
| Product page URL | Required for the product to appear in collection views and recommendation results |
| Price | Optional. One string with its currency, for example `"49.99 EUR"`. Irisphera returns it with the product exactly as sent. Do not use it for financial reporting: order amounts come from order events. |

Images must be PNG, JPEG, WebP or AVIF. Image URLs must work without a shopper login. Keep them working as long as you import the product, because every import downloads its images again.

The field names depend on the format:

| Format | SKU | Images and page |
| --- | --- | --- |
| JSON feed `{"products":[...]}` (methods A and B) | `skuCustomId` | snake_case: `product_images` (array), `product_page_url` |
| CSV feed (methods A and B) | `skuCustomId` | camelCase headers: `productImages` (URLs separated by `\|`), `productPageUrl` |
| Single product (method C) | `skuCustomId` | camelCase: `productImages` (array), `productPageUrl` |

All formats use `title`, `description`, `price` and the optional `gender`. There are no separate front, back or featured image fields: the first product image is the featured image. XLSX is not supported.

### Importing again

Imports add and update products. They never remove one.

- A feed import (method A or B) goes through every SKU in the feed. A SKU that is not in the collection yet is added. A SKU that is already there is updated if its data changed, and left alone if not.
- Within one feed, only the first entry of each SKU counts. Later duplicates are dropped.
- If you send a single product (method C) again while an earlier copy of the same SKU is still waiting to be processed, the new copy is dropped. Send it again once the earlier one is listed.
- A product that you leave out of a feed stays in the collection. The API cannot remove a single product yet: `DELETE /merchant/v1/collection/{collectionId}/products/{skuCustomId}` returns `501`.

## Describe the garment

Use the prepared garment's real URLs:

```bash
SKU='DEMO-BLAZER-BLACK'
read -rp 'Main image URL (becomes the featured image): ' MAIN_IMAGE_URL
read -rp 'Second image URL, for example the back view: ' SECOND_IMAGE_URL
read -rp 'Product page URL: ' PRODUCT_PAGE_URL
```

## Create a collection

Method A reads the feed URL from the collection itself, in its `productFeed` field. Set it when you create the collection. You cannot change it afterwards, because `PUT` on a collection currently creates a second collection instead of updating this one.

For method A, choose the HTTPS address where you will publish the feed, for example `https://shop.example.com/irisphera/products.json`. Nothing needs to be there yet. For method B or C, press Enter.

```bash
read -rp 'Feed URL for method A (press Enter for method B or C): ' FEED_URL
jq -n --arg title "Demo catalog $RUN_ID" --arg feed "$FEED_URL" \
  '{title:$title} + (if $feed == "" then {} else {productFeed:$feed} end)' > collection-request.json
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/merchant/v1/collection" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" \
  -H 'Content-Type: application/json' \
  --data-binary @collection-request.json -o collection.json
COLLECTION_ID=$(jq -er '.id' collection.json)
jq '{id,merchantId,title,productFeed}' collection.json
```

**Expected:** HTTP `201`, `merchantId` equals `MERCHANT_ID`, and for method A `productFeed` is your feed URL. Keep `COLLECTION_ID`: sending the request again creates a second collection. `GET /merchant/v1/collection` lists your collections.

## Build the feed

Methods A and B send the same feed file. Skip this section for method C. Change the example title, description, gender and price to match the garment.

```bash
jq -n --arg sku "$SKU" --arg main "$MAIN_IMAGE_URL" --arg second "$SECOND_IMAGE_URL" \
  --arg page "$PRODUCT_PAGE_URL" \
  '{products:[{skuCustomId:$sku, title:"Black wool blazer",
    description:"Single-breasted wool blazer.", gender:"WOMEN", price:"99.00 EUR",
    product_images:[$main,$second], product_page_url:$page}]}' > products.json
```

A real feed lists the whole catalog in the same `products` array.

## Method A: import from a feed URL

Irisphera downloads the feed from the collection's `productFeed` URL, reads it and queues its products, all while your request waits. The products are then processed in the background, as with any import.

### Publish the feed

Copy `products.json` to your server so that it is served at `FEED_URL`. The feed must meet these rules:

- It answers a plain `GET` over HTTPS, without a login, cookie or token. Irisphera cannot sign in to a feed. If the feed must stay private, use [method B](#method-b-upload-the-feed-file).
- It is reachable from the internet, not only from your own network.
- It answers quickly. Irisphera gives up if it cannot connect within 10 seconds or gets no response within 60 seconds.
- It is served as `application/json` or `text/csv`. Without one of those types, Irisphera guesses the format from the first bytes of the file.
- `FEED_URL` is the feed's final address. Irisphera follows redirects, but a direct URL keeps the request on your server.

**Serve the feed from a server you control.** Irisphera currently passes most headers of your `import-from-url` request on to the feed server, including `MERCHANT-API-KEY`. That server receives the merchant key. So publish the feed on your own platform, keep request headers out of that server's logs, and never point `productFeed` at a third-party host, a file-sharing link or a URL shortener. Do not use these passed-on headers to protect the feed either. Irisphera does not guarantee them.

Check that the feed is published and holds the garment:

```bash
curl --fail-with-body -sS "$FEED_URL" -o published-feed.json
jq -e --arg sku "$SKU" 'any(.products[]; .skuCustomId == $sku)' published-feed.json
```

**Expected:** `true`.

### Start the import

```bash
jq -n '{useSeasonFiltering:false}' > import-request.json
curl --fail-with-body -sS \
  "$IRISPHERA_BASE_URL/merchant/v1/collection/$COLLECTION_ID/import-from-url" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" \
  -H 'Content-Type: application/json' \
  --data-binary @import-request.json -w 'HTTP %{http_code}\n'
```

The body is optional. `useSeasonFiltering: true` applies the active seasonal policy. The request waits while Irisphera downloads and reads the feed, so a large feed or a slow server can keep it open for up to a minute.

**Expected:** HTTP `202`. Irisphera has downloaded the feed and queued its products. Continue with [what `202` means](#what-202-means).

If the request fails:

| Answer | Meaning | What to do |
| --- | --- | --- |
| `400` | The collection has no feed URL, the URL does not start with `http://` or `https://`, or the feed server answered with a 4xx status other than `401` and `403`, such as `404` | Check `productFeed` in `collection.json`, and open the URL yourself |
| `401` | Either the merchant key is wrong, or the feed server refused Irisphera with `401` or `403` | If the merchant key works on other routes, make the feed readable without a login |
| `402` | The contract reserves it for a used-up quota. Imports do not use quota at the moment. | See [if an import returns 402](#if-an-import-returns-402) |
| `403` | The collection belongs to another organization | Check `COLLECTION_ID` |
| `404` | The collection does not exist | Check `COLLECTION_ID` |
| `415` | The feed is neither JSON nor CSV | Serve it as `application/json` or `text/csv` |
| `500` | Irisphera could not reach the feed server, the server was too slow or answered with a 5xx status, or the feed could not be read | Check the server and the feed's content, then send the same request again |

When the catalog changes, publish the new feed at the same URL and send the same request again. See [importing again](#importing-again) for what changes.

## Method B: upload the feed file

```bash
curl --fail-with-body -sS \
  "$IRISPHERA_BASE_URL/merchant/v1/collection/$COLLECTION_ID/file?useSeasonFiltering=false" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" \
  -F 'file=@products.json;type=application/json'
```

For a CSV feed, send the file with `type=text/csv`. `useSeasonFiltering=true` applies the active seasonal policy.

**Expected:** HTTP `202`. Continue with [what `202` means](#what-202-means).

## Method C: send one product

The fields are camelCase. An image can also be a data URI such as `data:image/webp;base64,...` when your source cannot publish image URLs. Change the example title, description, gender and price to match the garment.

```bash
jq -n --arg sku "$SKU" --arg main "$MAIN_IMAGE_URL" --arg second "$SECOND_IMAGE_URL" \
  --arg page "$PRODUCT_PAGE_URL" \
  '{skuCustomId:$sku, title:"Black wool blazer",
    description:"Single-breasted wool blazer.", gender:"WOMEN", price:"99.00 EUR",
    productImages:[$main,$second], productPageUrl:$page}' > product.json
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/merchant/v1/collection/$COLLECTION_ID/products" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" \
  -H 'Content-Type: application/json' \
  --data-binary @product.json
```

**Expected:** HTTP `202`.

## What `202` means

A `202` means Irisphera accepted the products for processing. It does **not** prove that the garment was imported. Processing continues in the background, and a feed import also answers `202` when no product in the feed was usable. Always confirm the result in the catalog, as the next section shows.

### If an import returns 402

The contract lists `402` for the import routes, for an organization whose quota is used up. Imports do not count against quota at the moment, so you should not get it. If you do, stop importing for that organization and contact Irisphera. `GET /merchant/v1/subscription` shows the organization's quota ([step 2](02-create-merchant.md#how-much-quota-does-the-organization-have-and-what-uses-it)).

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

**Expected:** the SKU with its detected `placement`, for example `UPPER`, and `category`. There is no import-status endpoint and no deadline for processing. If the SKU stays absent, ask Irisphera to check the import. Irisphera may skip a SKU that has failed several times, so sending it again does not always help. An empty catalog is not a working integration. `GET /merchant/v1/products` lists products across all your collections.

## Confirm try-on readiness

A listed product is not necessarily ready for try-on. Your backend can check readiness with the merchant key, without a shopper session:

```bash
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/merchant/v1/storefront/products/$SKU" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" -o product-availability.json
jq '{vtoReadiness, has3dPreview:(.tdPreviewGlbUrl != null)}' product-availability.json
```

**Expected:** `vtoReadiness` is a placement such as `UPPER`, `LOWER` or `FULL`.

| Result | Meaning |
| --- | --- |
| A placement | The product can be tried on |
| `UNAVAILABLE` | The product cannot be tried on. Check its product images, starting with the first one, or wait for processing. |
| `404` | Unknown SKU |

`tdPreviewGlbUrl`, when present, is a temporary URL of the product's 3D model. This read uses no quota, needs no consent and records nothing, so your product page can use it to decide whether to show the try-on button.

Keep this exact `SKU` for try-on, events and the order line. Irisphera identifies the product in attribution by the channel and SKU together. If your platform uses different product IDs or channels, agree a mapping with Irisphera. A feed does not create aliases.

**Checkpoint:** `SKU` is listed in `catalog.json` and `vtoReadiness` is not `UNAVAILABLE`. Continue to [step 4](04-shopper-session.md).

## Frequently asked questions

### How often should we import the feed again?

Whenever the catalog changes, for example once a day or after each catalog publish. Send the whole feed each time, not only the changes. The import goes through every SKU and updates only the SKUs whose data changed ([importing again](#importing-again)). Imports do not use quota.

### How large can a feed be?

An uploaded feed file (method B) can be up to 100 MB. A feed URL (method A) has no size limit, but Irisphera must connect to your server within 10 seconds and get an answer within 60 seconds.

### Do image URLs have to stay online after the import?

Yes, for as long as the product is in a feed that you import. Every import downloads the product's images again, also when the product did not change, and a SKU whose image cannot be downloaded fails in that import. Between imports, Irisphera serves its own copy of the images.

### How do we remove a product that we no longer sell?

The API cannot remove products yet. The routes that would delete one product, the SKUs listed in a file or a whole collection all return `501`, and a product that you leave out of a feed stays in the collection. Contact Irisphera when products must leave the catalog. In the meantime, check the SKUs that recommendations and mix and match return against your own catalog, and hide the ones you no longer sell.

### Can the same SKU be in two collections?

Keep each SKU in one collection. Try-on, mix and match and the readiness checks identify a product by its SKU alone, so a SKU that is in two collections of the organization is ambiguous.

### Our platform has a SKU for each size. What do we send?

One product for each model and color, with a stable ID of that model and color as `skuCustomId`. The size SKUs stay in your platform. In order events, give each size line its own `sourceLineId`, a UUIDv7, and the model-and-color `skuCustomId` ([step 8](08-collect-events.md#record-the-accepted-order)).

### Can we protect the feed URL with a password or a token header?

No. Irisphera fetches the feed with a plain `GET` and cannot sign in. Upload the file instead ([method B](#method-b-upload-the-feed-file)) when the feed must not be public.

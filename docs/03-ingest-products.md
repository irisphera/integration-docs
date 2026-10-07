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
| CSV feed (methods A and B) | `skuCustomId` | The same names as column headers, in any case and with or without underscores: `productImages` or `product_images` (URLs separated by `\|`), `productPageUrl` or `product_page_url`. See the [CSV rules](#the-same-feed-as-csv). |
| Single product (method C) | `skuCustomId` | camelCase: `productImages` (array), `productPageUrl` |

All formats use `title`, `description`, `price` and the optional `gender`. There are no separate front, back or featured image fields: the first product image is the featured image. XLSX is not supported.

### Importing again

Imports add and update products. They never remove one.

- A feed import (method A or B) goes through every SKU in the feed. A SKU that is not in the collection yet is added. A SKU that is already there is updated if its data changed, and left alone if not.
- Within one feed, only the first entry of each SKU counts. Later duplicates are dropped.
- If you send a single product (method C) again while an earlier copy of the same SKU is still waiting to be processed, the new copy is dropped. Send it again once the earlier one is listed.
- A collection takes one feed import at a time. While an import of the collection is running or paused, method A and method B answer `409` and change nothing. Single products (method C) and replacements (`PUT`, below) are not refused: they join the running import. See [follow or cancel an import](#follow-or-cancel-an-import).
- To change one product without a feed, replace it with `PUT /merchant/v1/collection/{collectionId}/products/{skuCustomId}`. Send the whole product, in the method C format. If the body leaves out `title`, `description`, `productPageUrl` or `price` and the stored product has a value for it, the request answers `400` and names the missing fields.
- A product that you leave out of a feed stays in the collection. To remove it, [delete it](#how-do-we-remove-a-product-that-we-no-longer-sell).

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

Methods A and B send the same feed file. Skip this section for method C. Change the example title, description, gender and price to match the garment. The walkthrough uses JSON; a CSV feed with the same columns works the same way, as the [CSV example](#the-same-feed-as-csv) below shows.

```bash
jq -n --arg sku "$SKU" --arg main "$MAIN_IMAGE_URL" --arg second "$SECOND_IMAGE_URL" \
  --arg page "$PRODUCT_PAGE_URL" \
  '{products:[{skuCustomId:$sku, title:"Black wool blazer",
    description:"Single-breasted wool blazer.", gender:"WOMEN", price:"99.00 EUR",
    product_images:[$main,$second], product_page_url:$page}]}' > products.json
```

A real feed lists the whole catalog in the same `products` array. A feed with four products looks like this:

```json
{
  "products": [
    {
      "skuCustomId": "BLAZER-WOOL-BLACK",
      "title": "Wool blazer, black",
      "description": "Single-breasted wool blazer with notch lapels.",
      "gender": "WOMEN",
      "price": "129.00 EUR",
      "product_images": [
        "https://shop.example.com/images/blazer-wool-black-front.jpg",
        "https://shop.example.com/images/blazer-wool-black-back.jpg"
      ],
      "product_page_url": "https://shop.example.com/products/wool-blazer?color=black"
    },
    {
      "skuCustomId": "BLAZER-WOOL-NAVY",
      "title": "Wool blazer, navy",
      "description": "Single-breasted wool blazer with notch lapels.",
      "gender": "WOMEN",
      "price": "129.00 EUR",
      "product_images": [
        "https://shop.example.com/images/blazer-wool-navy-front.jpg"
      ],
      "product_page_url": "https://shop.example.com/products/wool-blazer?color=navy"
    },
    {
      "skuCustomId": "JEANS-STRAIGHT-INDIGO",
      "title": "Straight-leg jeans, indigo",
      "description": "Mid-rise straight-leg jeans in rigid cotton denim.",
      "gender": "MEN",
      "price": "79.90 EUR",
      "product_images": [
        "https://shop.example.com/images/jeans-straight-indigo.jpg"
      ],
      "product_page_url": "https://shop.example.com/products/straight-jeans"
    },
    {
      "skuCustomId": "DRESS-LINEN-WHITE",
      "title": "Linen midi dress, white",
      "description": "Sleeveless linen midi dress with a square neckline.",
      "product_images": [
        "https://shop.example.com/images/dress-linen-white.jpg"
      ],
      "product_page_url": "https://shop.example.com/products/linen-midi-dress"
    }
  ]
}
```

- The black and the navy blazer are one model in two colors, so they are two products with two SKUs. Sizes are not listed ([one SKU for each model and color](#our-platform-has-a-sku-for-each-size-what-do-we-send)).
- The dress leaves out the optional `gender` and `price`.
- Send only the fields in [product rules](#product-rules). A JSON product with any other field, such as `color`, makes Irisphera reject the whole feed.

### The same feed as CSV

The four products above, saved as `products.csv`:

```csv
skuCustomId,title,description,gender,price,productImages,productPageUrl
BLAZER-WOOL-BLACK,"Wool blazer, black",Single-breasted wool blazer with notch lapels.,WOMEN,129.00 EUR,https://shop.example.com/images/blazer-wool-black-front.jpg|https://shop.example.com/images/blazer-wool-black-back.jpg,https://shop.example.com/products/wool-blazer?color=black
BLAZER-WOOL-NAVY,"Wool blazer, navy",Single-breasted wool blazer with notch lapels.,WOMEN,129.00 EUR,https://shop.example.com/images/blazer-wool-navy-front.jpg,https://shop.example.com/products/wool-blazer?color=navy
JEANS-STRAIGHT-INDIGO,"Straight-leg jeans, indigo",Mid-rise straight-leg jeans in rigid cotton denim.,MEN,79.90 EUR,https://shop.example.com/images/jeans-straight-indigo.jpg,https://shop.example.com/products/straight-jeans
DRESS-LINEN-WHITE,"Linen midi dress, white",Sleeveless linen midi dress with a square neckline.,,,https://shop.example.com/images/dress-linen-white.jpg,https://shop.example.com/products/linen-midi-dress
```

- The first row names the columns. It must name `skuCustomId`, `title` and `productImages`: a file without one of them imports nothing. The order of the columns does not matter, and Irisphera ignores columns it does not use.
- Irisphera compares column names without case, spaces, underscores or dashes, so `productImages`, `product_images` and `Product Images` all name the image column. The other names of the JSON feed work too: `product_name` for `title`, `product_description` for `description` and `product_url` for `productPageUrl`.
- Irisphera takes the separator from the first row: the first of comma, semicolon and tab that gives the three required columns. Excel saves with a semicolon in many languages, so a file saved by Excel in any language works. Use one separator in the whole file.
- Separate a product's image URLs with `|`. The first one is the featured image.
- Put a cell in double quotes when it holds the separator, like the commas in the titles above, or a double quote, and write each double quote inside it twice: `""`. Spreadsheet programs do this for you. A separator outside quotes starts a new cell, so the cells after it move one column along and the product gets wrong values.
- A cell cannot hold a line break: each line of the file is one row. Replace line breaks in descriptions with spaces before you export. If your descriptions need line breaks, use the JSON feed, where text can hold them (`\n`).
- Leave optional cells empty, as the dress does. Irisphera skips a row whose SKU, title or image cell is empty, and imports the other rows. Blank lines are skipped, also before the first row.
- Save the file as UTF-8. In Excel, choose **CSV UTF-8**: its other CSV formats lose accented letters such as `ă` or `é`. The byte order mark that CSV UTF-8 writes at the start of the file does no harm.
- Excel changes long numbers and drops leading zeros when it opens a CSV file. If your SKUs are numbers, compare a few of them in the saved file with your platform before you import it.

## Method A: import from a feed URL

Irisphera downloads the feed from the collection's `productFeed` URL, reads it and queues its products, all while your request waits. The products are then processed in the background, as with any import.

### Publish the feed

Copy `products.json` to your server so that it is served at `FEED_URL`. The feed must meet these rules:

- It answers a plain `GET` over HTTPS, without a login, cookie or token. Irisphera cannot sign in to a feed. If the feed must stay private, use [method B](#method-b-upload-the-feed-file).
- It is reachable from the internet, not only from your own network.
- It answers quickly. Irisphera gives up if it cannot connect within 10 seconds or gets no response within 60 seconds.
- It is served as `application/json` or `text/csv`. Irisphera also reads the start of the file: a file whose first row names the `skuCustomId` column is read as CSV whatever its type, so a CSV feed served as `text/plain` also imports. A file that opens with `{` is read as JSON unless it is served as `text/csv`.
- `FEED_URL` is the feed's final address. Irisphera follows redirects, but a direct URL keeps the request on your server.

**Serve the feed from a server you control.** Irisphera currently passes most headers of your `import-from-url` request on to the feed server, including `MERCHANT-API-KEY`. That server receives the merchant key. So publish the feed on your own platform, keep request headers out of that server's logs, and never point `productFeed` at a third-party host, a file-sharing link or a URL shortener. Do not use these passed-on headers to protect the feed either. Irisphera does not guarantee them.

Check that the feed is published and holds the garment:

```bash
curl --fail-with-body -sS "$FEED_URL" -o published-feed.json
jq -e --arg sku "$SKU" 'any(.products[]; .skuCustomId == $sku)' published-feed.json
```

**Expected:** `true`. For a CSV feed, check it with `grep -F "$SKU" published-feed.json` instead.

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
| `409` | Another import of the collection is still running or paused. Irisphera did not fetch the feed. | [Follow the import](#follow-or-cancel-an-import) and send the same request again when its `state` is `IDLE` |
| `415` | Irisphera found neither JSON nor CSV at the URL | Open the URL yourself. It must return the feed, not a web page or a spreadsheet file, and a CSV feed must name the `skuCustomId` column in its first row. |
| `500` | Irisphera could not reach the feed server, the server was too slow or answered with a 5xx status, or the feed could not be read | Check the server and the feed's content, then send the same request again |

When the catalog changes, publish the new feed at the same URL and send the same request again. See [importing again](#importing-again) for what changes.

## Method B: upload the feed file

```bash
curl --fail-with-body -sS \
  "$IRISPHERA_BASE_URL/merchant/v1/collection/$COLLECTION_ID/file?useSeasonFiltering=false" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" \
  -F 'file=@products.json;type=application/json'
```

For a CSV feed, send the file with `type=text/csv`. Irisphera also recognises a CSV file sent with another type, such as the `application/vnd.ms-excel` that browsers on Windows give `.csv` files. `useSeasonFiltering=true` applies the active seasonal policy.

**Expected:** HTTP `202`. Continue with [what `202` means](#what-202-means).

If a run of the collection still lasts or is paused, the upload answers `409` and imports nothing. [Follow the run](#follow-or-cancel-an-import) and upload the file again when `state` is `IDLE`.

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

A single product is accepted even while a run of the collection lasts: it joins that run. A single product or a replacement also starts a run when none lasts, so a feed import of the collection answers `409` until the product is done.

## What `202` means

A `202` means Irisphera accepted the products for processing. It does **not** prove that the garment was imported. Processing continues in the background, and a feed import also answers `202` when no product in the feed was usable. Always confirm the result in the catalog, as the next section shows.

### If an import returns 402

The contract lists `402` for the import routes, for an organization whose quota is used up. Imports do not count against quota at the moment, so you should not get it. If you do, stop importing for that organization and contact Irisphera. `GET /merchant/v1/subscription` shows the organization's quota ([step 2](02-create-merchant.md#how-much-quota-does-the-organization-have-and-what-uses-it)).

## Follow or cancel an import

Irisphera processes the products of a collection in runs. A run starts with a feed import (method A or B), a Shopify sync or a single product (method C). It lasts until every product it queued is processed or until you cancel it. While a run lasts, also while it is paused, a feed import answers `409` and changes nothing. Single products (method C) and replacements (`PUT`) are not refused: they join the run.

Read the collection's processing:

```bash
curl --fail-with-body -sS \
  "$IRISPHERA_BASE_URL/merchant/v1/collection/$COLLECTION_ID/processing" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" -o processing.json
jq . processing.json
```

**Expected:** HTTP `200`. While the garment of a feed import is processed, the answer looks like this:

```json
{
  "state": "RUNNING",
  "pausedUntil": null,
  "run": {
    "id": "0199c1a2-5b7e-7c3d-9a41-6f0e2d8b4c10",
    "outcome": "RUNNING",
    "startedAt": "2026-10-07T09:30:00.123Z",
    "endedAt": null,
    "uploads": { "file": 0, "url": 1, "shopify": 0, "product": 0 },
    "receivedItems": 1,
    "queuedItems": 1,
    "filteredItems": {
      "invalid": 0,
      "duplicateInUpload": 0,
      "alreadyQueued": 0,
      "alreadyInCollection": 0,
      "outOfSeason": 0,
      "previouslyFailed": 0
    },
    "remainingItems": 1,
    "completedItems": 0,
    "failedItems": 0,
    "failures": [],
    "cancelledItems": 0
  }
}
```

| `state` | Meaning | A feed import now |
| --- | --- | --- |
| `IDLE` | No run lasts. `run` is the last run, or `null` when the collection never processed products. | Accepted |
| `RUNNING` | Products are queued or being processed. `run` is the current run. | Answers `409` |
| `PAUSED` | The run waits until `pausedUntil`, for example because a service it needs failed. It resumes on its own. In the other states, `pausedUntil` is `null`. | Answers `409` |

`run` describes the current run, or the last run after it ended:

| Field | Meaning |
| --- | --- |
| `outcome` | `RUNNING` while the run lasts, also while it is paused. `FINISHED` when every queued product was processed. `CANCELLED` when the run was cancelled. |
| `startedAt`, `endedAt` | When the first request of the run arrived, and when the run finished or was cancelled. `endedAt` is `null` while the run lasts. |
| `uploads` | The number of requests that sent products in the run: `url` for method A, `file` for method B, `shopify` for Shopify syncs and `product` for method C and replacements |
| `receivedItems` | The products that the requests sent. A product sent by two requests counts twice. |
| `queuedItems` | The products queued for processing |
| `filteredItems` | The products not queued, by reason. `invalid`: no SKU, no title or no usable image. `duplicateInUpload`: sent more than once in the same request. `alreadyQueued`: queued by an earlier request of the run. `alreadyInCollection`: already in the collection with the same data. `outOfSeason`: left out by the season filter. `previouslyFailed`: failed too often in earlier runs. |
| `remainingItems` | The queued products still to process. `0` once the run ended. |
| `completedItems`, `failedItems` | The products added or updated, and the products that could not be added |
| `failures` | The failed products by reason, most frequent first, for example `{"reason": "invalid_image", "count": 2}`. The reason is `Failure` when no finer reason is known. |
| `cancelledItems` | The queued products that a cancellation dropped |

After the run ends, `state` is `IDLE` and `run` keeps its counts, so you can read how the last import went. Still confirm the products in the catalog, as the next section shows.

To stop the run, for example after you sent the wrong feed:

```bash
curl --fail-with-body -sS -X DELETE \
  "$IRISPHERA_BASE_URL/merchant/v1/collection/$COLLECTION_ID/processing" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" -w 'HTTP %{http_code}\n'
```

Do not run it in the walkthrough: it would drop the garment.

**Expected:** HTTP `204`.

- Irisphera drops every product that the run still had to process, including single products and replacements that joined it, and a product being processed at that moment.
- Products already imported stay in the collection. To remove one, [delete it](#how-do-we-remove-a-product-that-we-no-longer-sell).
- The cancelled run stays readable with `GET`: its `outcome` is `CANCELLED`, and `cancelledItems` counts the dropped products.
- A new feed import of the collection is accepted right after. To replace a running import with a newer feed, cancel the run and send the newer feed. A feed holds the whole catalog, so nothing is lost.
- When no run lasts, nothing changes, and the answer is also `204`.

Both routes answer `403` when the collection belongs to another organization and `404` when it does not exist.

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

**Expected:** the SKU with its detected `placement`, for example `UPPER`, and `category`. There is no deadline for processing. While the SKU is absent, [the import's state](#follow-or-cancel-an-import) tells you whether it is still running. If the SKU is still absent once the state is `IDLE`, Irisphera did not import it: ask Irisphera to check the import. Irisphera may skip a SKU that has failed several times, so sending it again does not always help. An empty catalog is not a working integration. `GET /merchant/v1/products` lists products across all your collections.

Irisphera names the `category` from the product images, title and description; a category in your feed is ignored. The value is one of `tops`, `shirt`, `jacket`, `pants`, `jeans`, `trousers`, `joggers`, `shorts`, `skirt`, `dress`, `jumpsuit`, `romper`, `flowing`, `suit`, `set`, `swimwear`, `underwear`, `sleepwear`, `hosiery`, `headwear`, `shoes`, `accessories`, `earrings`, `necklaces`, `bracelets`, `rings`, `watches` or `unclassified`. Socks and tights are `hosiery`; pajamas, nightgowns and bathrobes are `sleepwear`; leggings are `pants`. `unclassified` means Irisphera could not name a category: the product stays in your catalog, but it is not used for try-on, recommendations or mix and match. Hosiery is not used for try-on, recommendations or mix and match either, and sleepwear is not used for try-on or mix and match.

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

### A feed import answered `409`. What do we do?

A run of the collection still lasts or is paused, and Irisphera changed nothing. [Read the collection's processing](#follow-or-cancel-an-import) and send the same request again when `state` is `IDLE`. A scheduled import, such as a daily one, can also skip this time: the next import sends the whole catalog anyway. If the newer feed must replace the running import now, cancel the run first. The `409` has an `application/problem+json` body whose `detail` says the same.

### How large can a feed be?

An uploaded feed file (method B) can be up to 100 MB. A feed URL (method A) has no size limit, but Irisphera must connect to your server within 10 seconds and get an answer within 60 seconds.

### Do image URLs have to stay online after the import?

Yes, for as long as the product is in a feed that you import. Every import downloads the product's images again, also when the product did not change, and a SKU whose image cannot be downloaded fails in that import. Between imports, Irisphera serves its own copy of the images.

### How do we remove a product that we no longer sell?

Delete it, and leave it out of your feed. A feed import adds a missing product again.

```bash
curl --fail-with-body -sS -X DELETE \
  "$IRISPHERA_BASE_URL/merchant/v1/collection/$COLLECTION_ID/products/$SKU" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" -w 'HTTP %{http_code}\n'
```

Do not run it in the walkthrough: the later steps need the garment.

- `204`: the product leaves the collection at once. Irisphera keeps an archive copy of its data and images, so that support can restore it on request. To add it again, send it as a single product (method C).
- `404`: the collection or the product does not exist.
- `409`: an import of this product is still waiting to be processed. Send the request again once the product is listed.

The routes that would delete the SKUs listed in a file or a whole collection still return `501`.

### Can the same SKU be in two collections?

Keep each SKU in one collection. Try-on, mix and match and the readiness checks identify a product by its SKU alone, so a SKU that is in two collections of the organization is ambiguous.

### Our platform has a SKU for each size. What do we send?

One product for each model and color, with a stable ID of that model and color as `skuCustomId`. The size SKUs stay in your platform. In order events, give each size line its own `sourceLineId`, a UUIDv7, and the model-and-color `skuCustomId` ([step 8](08-collect-events.md#record-the-accepted-order)).

### Can we protect the feed URL with a password or a token header?

No. Irisphera fetches the feed with a plain `GET` and cannot sign in. Upload the file instead ([method B](#method-b-upload-the-feed-file)) when the feed must not be public.

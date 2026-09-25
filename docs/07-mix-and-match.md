# 7. Mix and match

Previous: [Recommendations and sizing](06-recommendations.md) · [Integration guide](../README.md) · Next: [Storefront and order events](08-collect-events.md)

**Goal:** show the shopper the catalog products that complete an outfit with the garment on the product page.

Mix and match takes one SKU. For each outfit placement that the garment does not fill, it returns the one product from the organization's catalog that goes best with it. For a blazer, that is one pair of trousers, one pair of shoes, one bag, and so on. The route needs the `shopper:recommendations` scope from [step 4](04-shopper-session.md#check-the-session). It needs no profile, photo or privacy choice: it compares products, not the shopper.

## Add a product that completes the outfit

The walkthrough's catalog holds only the garment from step 3, so mix and match has nothing to suggest yet. Import one more product into the same collection, for another placement, such as trousers or shoes. Choose one for the same occasion as the garment, such as tailored trousers for a blazer, and give it the garment's gender or `UNISEX`.

```bash
MATCH_SKU='DEMO-TROUSERS-BLACK'
read -rp 'Main image URL of the matching product (becomes the featured image): ' MATCH_MAIN_IMAGE_URL
read -rp 'Second image URL, for example the back view: ' MATCH_SECOND_IMAGE_URL
read -rp 'Product page URL: ' MATCH_PRODUCT_PAGE_URL
```

Import it with the method you used in [step 3](03-ingest-products.md#choose-how-to-send-products). In each example, change the title, description, gender and price to match the product.

**Method A or B (feed).** Add the product to `products.json`. Keep the garment in the feed. It is already in the collection and unchanged, so the import [leaves it as it is](03-ingest-products.md#importing-again).

```bash
jq --arg sku "$MATCH_SKU" --arg main "$MATCH_MAIN_IMAGE_URL" --arg second "$MATCH_SECOND_IMAGE_URL" \
  --arg page "$MATCH_PRODUCT_PAGE_URL" \
  '.products += [{skuCustomId:$sku, title:"Black tailored trousers",
    description:"High-waisted wool trousers with a straight leg.", gender:"WOMEN", price:"79.00 EUR",
    product_images:[$main,$second], product_page_url:$page}]' products.json > products-next.json
mv products-next.json products.json
```

For method A, publish the new `products.json` at `FEED_URL` in place of the old one, and send the import request again:

```bash
curl --fail-with-body -sS \
  "$IRISPHERA_BASE_URL/merchant/v1/collection/$COLLECTION_ID/import-from-url" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" \
  -H 'Content-Type: application/json' \
  --data-binary @import-request.json -w 'HTTP %{http_code}\n'
```

For method B, upload the file again:

```bash
curl --fail-with-body -sS \
  "$IRISPHERA_BASE_URL/merchant/v1/collection/$COLLECTION_ID/file?useSeasonFiltering=false" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" \
  -F 'file=@products.json;type=application/json'
```

**Method C (single product).** Send the product on its own:

```bash
jq -n --arg sku "$MATCH_SKU" --arg main "$MATCH_MAIN_IMAGE_URL" --arg second "$MATCH_SECOND_IMAGE_URL" \
  --arg page "$MATCH_PRODUCT_PAGE_URL" \
  '{skuCustomId:$sku, title:"Black tailored trousers",
    description:"High-waisted wool trousers with a straight leg.", gender:"WOMEN", price:"79.00 EUR",
    productImages:[$main,$second], productPageUrl:$page}' > match-product.json
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/merchant/v1/collection/$COLLECTION_ID/products" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" \
  -H 'Content-Type: application/json' \
  --data-binary @match-product.json
```

**Expected:** HTTP `202`. As in [step 3](03-ingest-products.md#confirm-the-sku-is-listed), repeat this read until the product is listed:

```bash
curl --fail-with-body -sS \
  "$IRISPHERA_BASE_URL/merchant/v1/collection/$COLLECTION_ID/products" \
  -H "MERCHANT-API-KEY: $MERCHANT_API_KEY" -o catalog.json
if jq -e --arg sku "$MATCH_SKU" 'any(.fashionItems[]; .skuCustomId == $sku)' catalog.json >/dev/null; then
  jq --arg sku "$MATCH_SKU" '.fashionItems[] | select(.skuCustomId == $sku)
    | {skuCustomId, placement:.data.placement, category:.data.category}' catalog.json
else
  printf 'STOP: the SKU is not listed yet. Wait and repeat this read.\n'
fi
```

**Expected:** the product with a detected `placement` that the garment does not fill, for example `LOWER` for trousers. A real store needs no extra import: mix and match uses the whole catalog.

## Get the suggestions

The browser calls the route with the shopper token, for example when the product page opens:

```bash
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/shopper/v2/mixmatch/$SKU" \
  -H "Authorization: Bearer $ACCESS_TOKEN" -o mix-and-match.json
jq '[.[] | {placement, skuCustomId, title, matchScore, productPageUrl,
     hasFeaturedImage:any(.images[]; .imageView == "FEATURED" and .file.presignedUrl != null)}]' \
  mix-and-match.json
```

**Expected:** HTTP `200` with a JSON array that holds the trousers as a `LOWER` item. The array has at most one item for each placement, always in this order: `UPPER`, `LOWER`, `FULL`, `FEET`, `HEAD`, `ACCESSORY`, `JEWELRY`. A placement is missing when no product in the catalog goes well enough with the garment.

| Response field | Meaning |
| --- | --- |
| `placement` | The placement that the product fills. Products in the `accessories` category, such as bags and belts, are always `ACCESSORY`. |
| `skuCustomId`, `collectionId` | The product and the collection that holds it |
| `title`, `category`, `productPageUrl` | The product's details from the catalog, when they are stored |
| `matchScore` | How well the product goes with the garment, from 0 to 1, with three decimals. Higher is better. It is not a probability. |
| `images[]` | `imageView` (`FEATURED`, `FRONT`, `BACK`, `OTHER` or `REFERENCE`) and `file`. `file.presignedUrl` is a temporary download URL. `file.path` is Irisphera's storage path, not a URL for the browser. |

The images come with temporary URLs unless you send `generatePresignedUrl=false`. Send `false` when your storefront shows its own product card for each `skuCustomId`. Do not store the temporary URLs.

| Answer | Meaning | Action |
| --- | --- | --- |
| `200` with items | Products that complete the outfit | Show them |
| `200` with `[]` | No product goes well enough with the garment, or Irisphera has not detected the garment's placement yet | Hide the mix-and-match block |
| `404` | Unknown SKU, or not in this organization's catalog | Check the SKU against step 3 |
| `403` | The token has no `shopper:recommendations` scope | Hide the mix-and-match block. The scope is granted only while the organization has recommendation quota. |

## How Irisphera chooses the products

Irisphera compares the features that it detected when it imported each product in [step 3](03-ingest-products.md). The shopper is not part of the comparison, so every shopper gets the same answer for the same SKU until the catalog changes.

The garment's placement decides which placements the answer can hold:

| The garment | The answer can hold |
| --- | --- |
| `UPPER`, such as a blazer | `LOWER`, `FEET`, `HEAD`, `ACCESSORY`, `JEWELRY` |
| `LOWER`, such as trousers | `UPPER`, `FEET`, `HEAD`, `ACCESSORY`, `JEWELRY` |
| `FULL`, such as a dress | `FEET`, `HEAD`, `ACCESSORY`, `JEWELRY` |
| `FEET`, `HEAD`, `ACCESSORY` or `JEWELRY` | `UPPER`, `LOWER` and the other placements in this row. `FULL` only when no `UPPER` or `LOWER` product qualifies. |

A product qualifies when all of these are true:

- It is in one of the organization's collections, and it is not the garment itself.
- Its gender is the garment's gender. `UNISEX` also goes with `MEN` and `WOMEN`. `CHILDREN_GIRL` and `CHILDREN_BOY` products go only with the same gender.
- Swimwear and underwear go only with garments of the same category: a bikini top gets a bikini bottom, never jeans. They can still get shoes, bags and jewelry.
- Its match score is at least 0.4.

The match score combines four features, from the most to the least important:

1. **Occasion.** Products for the same occasion score highest. Related occasions, such as a wedding and a cocktail party, score less. Unrelated occasions score nothing.
2. **Color palette.** The more color palettes the two products share, the higher the score.
3. **Silhouette.** For a top and a bottom, balanced volumes score higher. A fitted top with wide trousers scores more than an oversized top with wide trousers.
4. **Style.** Products that share a style score higher.

A feature that Irisphera could not detect on one of the products counts as neutral. When two products have the same score, Irisphera always picks the same one.

## Show the suggestions in your storefront

- Offer mix and match only while the token has `shopper:recommendations`. Check `isApsEnabled` ([step 2](02-create-merchant.md#read-the-storefront-configuration)) as you do for recommendations.
- The request uses no quota, so the product page can make it when it opens.
- Show each item as a product card that links to its `productPageUrl`, or to your own product page for its `skuCustomId`. Hide the placements that are missing.
- When the shopper opens a suggested product, its page sends the usual `PRODUCT_VIEWED` event ([step 8](08-collect-events.md)). There is no separate mix-and-match event.

## What Irisphera records

Nothing. Mix and match reads only the catalog. It works the same whatever the shopper chose in [step 4](04-shopper-session.md#record-the-shoppers-privacy-choices), and the report has no mix-and-match section.

**Checkpoint:** `mix-and-match.json` lists the product that you imported in this step, with a placement that the garment does not fill. Continue to [step 8](08-collect-events.md).

## Frequently asked questions

### Can we cache the suggestions?

Yes. Every shopper gets the same answer for the same SKU until the catalog changes, so your backend can cache it for each SKU and refresh it after an import. Request it with `generatePresignedUrl=false` and show your own product images, because the temporary image URLs expire.

### Can we leave out products, such as ones that are out of stock?

Not in the request. Irisphera has no stock information and suggests any product in the catalog. Filter the answer in your storefront before you show it. When you hide a suggestion, the placement stays empty: Irisphera returns only one product for each placement.

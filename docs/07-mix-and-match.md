# 7. Mix and match

Previous: [Recommendations and sizing](06-recommendations.md) · [Integration guide](../README.md) · Next: [Storefront and order events](08-collect-events.md)

**Goal:** show the shopper the catalog products that complete an outfit with the garment on the product page.

Mix and match takes one SKU and returns ranked outfits built around it from the organization's catalog. For a blazer, an outfit can add trousers, shoes, a bag, and so on. The answer holds up to 5 outfits, best first. Every shopper token can call the route: it needs only the `shopper:session` scope from [step 4](04-shopper-session.md#check-the-session), and it uses no quota. It needs no profile, photo or privacy choice: it compares products, not the shopper.

## Add a product that completes the outfit

The walkthrough's catalog holds only the garment from step 3, so mix and match has nothing to suggest yet. Import one more product into the same collection, for another placement, such as trousers or shoes. Choose one for the same occasion as the garment, such as tailored trousers for a blazer, and give it the garment's gender: mix and match combines only products of the same gender. One product is enough to check the call; an outfit needs a larger catalog (see [outfits](#outfits)).

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

**Expected:** HTTP `202`. If the feed import answers `409`, the import from step 3 is still running: send the request again when [its state](03-ingest-products.md#follow-or-cancel-an-import) is `IDLE`.

As in [step 3](03-ingest-products.md#confirm-the-sku-is-listed), repeat this read until the product is listed:

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
jq '{source, outfits: [.outfits[] | {rank, score, items: [.items[] | {placement, skuCustomId, title, matchScore,
     productPageUrl, hasFeaturedImage:any(.images[]; .imageView == "FEATURED" and .file.presignedUrl != null)}]}]}' \
  mix-and-match.json
```

**Expected:** HTTP `200` with a JSON object whose `source` is `ENGINE`. In the walkthrough, `outfits` is empty: Irisphera builds outfits only from a larger catalog, with at least five products of the garment's gender for each part that an outfit must have (see [outfits](#outfits)). With such a catalog, the trousers can appear as a `LOWER` item.

| Response field | Meaning |
| --- | --- |
| `source` | Always `ENGINE`: Irisphera built the [outfits](#outfits) from the product images and features. There is no rule-based fallback. |
| `outfits[]` | The outfits, best first. Empty when Irisphera cannot build an outfit around the garment. |
| `outfits[].rank` | The outfit's position: `1` for the best outfit, then `2`, `3` and so on |
| `outfits[].score` | How well the whole outfit goes together, from 0 to 1, with three decimals. Higher is better. |
| `outfits[].items[]` | The products that complete the outfit, without the garment itself, in placement order: `UPPER`, `LOWER`, `FULL`, `FEET`, `ACCESSORY`. An outfit can hold two `UPPER` products, such as a top and a jacket. |

Each product in `items[]` has these fields:

| Product field | Meaning |
| --- | --- |
| `placement` | The placement that the product fills. Products in the `accessories` category, such as bags and belts, are always `ACCESSORY`. |
| `skuCustomId`, `collectionId` | The product and the collection that holds it |
| `title`, `category`, `productPageUrl` | The product's details from the catalog, when they are stored |
| `matchScore` | How well the product goes with the garment, from 0 to 1, with three decimals. Higher is better. See [how Irisphera chooses the products](#how-irisphera-chooses-the-products). |
| `images[]` | `imageView` (`FEATURED`, `FRONT`, `BACK`, `OTHER` or `REFERENCE`) and `file`. `file.presignedUrl` is a temporary download URL. `file.path` is Irisphera's storage path, not a URL for the browser. |

The images come with temporary URLs unless you send `generatePresignedUrl=false`. Send `false` when your storefront shows its own product card for each `skuCustomId`. Do not store the temporary URLs.

| Answer | Meaning | Action |
| --- | --- | --- |
| `200` with outfits | Outfits built around the garment | Show them |
| `200` with `"outfits": []` | Irisphera cannot build an outfit around the garment: the garment cannot anchor an outfit, its images are not analyzed yet, or the catalog holds too few matching products | Hide the mix-and-match block |
| `404` | Unknown SKU, or not in this organization's catalog | Check the SKU against step 3 |

## How Irisphera chooses the products

Irisphera compares the products' images and the features that it detected when it imported each product in [step 3](03-ingest-products.md). The shopper is not part of the comparison, so every shopper gets the same answer for the same SKU until the catalog changes.

When Irisphera cannot build an outfit, the answer has no outfits. It never falls back to a simpler suggestion.

### Outfits

Irisphera builds outfits when the garment is a top, a bottom, a one-piece item such as a dress, outerwear, or a pair of shoes. This holds for every gender: `MEN`, `WOMEN`, `UNISEX`, `CHILDREN_GIRL` and `CHILDREN_BOY`.

- It builds up to 5 outfits around the garment. Each outfit holds a top and a bottom, or a one-piece item. It can add shoes, outerwear, a bag and an accessory. Outfits hold only products of the garment's gender. A `UNISEX` garment gets `UNISEX` products only, and the outfits of a `WOMEN` or `MEN` garment hold no `UNISEX` product.
- It needs at least five products of the garment's gender for each part that an outfit must have, such as five bottoms for a top. It adds an optional part, such as shoes, only when the catalog holds at least five products for it.
- A learned model compares the product images of each pair of products. Rules for color, style, occasion, pattern, fabric and season complete the outfit's `score`.
- An outfit never mixes a product for sport, such as workout leggings or running shoes, with a product for a dressy occasion: formal, evening, wedding, beach wedding, cocktail party, black-tie event, winter formal event, formal business, corporate business meeting, job interview or graduation.
- Irisphera prefers variety. An outfit that repeats products of a better outfit moves down, so an outfit can have a higher `score` than the outfit above it. A product can still appear in more than one outfit.
- `matchScore` is the learned model's probability that the product goes with the garment. When the model does not compare the two kinds of product, such as shoes and a bag, it is the mean of the rule scores for color, style, occasion, pattern, fabric and season.

### Which products join an outfit

The product's detected `category` decides the part it can fill:

| Category | Part of the outfit |
| --- | --- |
| `tops`, `shirt` | Top |
| `pants`, `jeans`, `trousers`, `shorts`, `skirt`, `joggers` | Bottom |
| `dress`, `jumpsuit`, `romper` | One-piece item |
| `jacket` | Outerwear |
| `shoes` | Shoes |
| `accessories` | A bag or an accessory, such as a belt, scarf or sunglasses, depending on its detected type |
| `flowing` | A kimono joins as outerwear and a kaftan as a one-piece item; other flowing garments do not join |
| `hosiery`, `sleepwear`, `underwear`, `unclassified` | Never |
| `suit`, `set`, `swimwear`, `headwear`, `earrings`, `necklaces`, `bracelets`, `rings`, `watches` | Not yet |

A product whose category Irisphera could not detect is `unclassified`. It stays in the catalog, but mix and match never uses it.

## Show the suggestions in your storefront

- Offer mix and match to every shopper. It does not depend on quota or on the recommendation feature, so it works even when `isApsEnabled` ([step 2](02-create-merchant.md#read-the-storefront-configuration)) is `false` or the token has no `shopper:recommendations` scope.
- The request uses no quota, so the product page can make it when it opens.
- Show each outfit with the garment and the outfit's products, best outfit first. To show fewer outfits, take the first ones.
- Show each product as a product card that links to its `productPageUrl`, or to your own product page for its `skuCustomId`.
- When the shopper opens a suggested product, its page sends the usual `PRODUCT_VIEWED` event ([step 8](08-collect-events.md)). There is no separate mix-and-match event.

## What Irisphera records

Nothing. Mix and match reads only the catalog. It works the same whatever the shopper chose in [step 4](04-shopper-session.md#record-the-shoppers-privacy-choices), and the report has no mix-and-match section.

**Checkpoint:** `mix-and-match.json` has `source` `ENGINE`. With the walkthrough's small catalog, `outfits` is empty; in a store with enough products of the garment's gender, an outfit holds products for placements that the garment does not fill. Continue to [step 8](08-collect-events.md).

## Frequently asked questions

### Can we cache the suggestions?

Yes. Every shopper gets the same answer for the same SKU until the catalog changes, so your backend can cache it for each SKU and refresh it after an import. Request it with `generatePresignedUrl=false` and show your own product images, because the temporary image URLs expire.

### Can we leave out products, such as ones that are out of stock?

Not in the request. Irisphera has no stock information and suggests any product in the catalog. Filter the answer in your storefront before you show it. When you hide a product, hide the outfits that hold it and show the next outfits instead.

# 7. Mix and match

Previous: [Recommendations and sizing](06-recommendations.md) · [Integration guide](../README.md) · Next: [Storefront and order events](08-collect-events.md)

**Goal:** show the shopper the catalog products that complete an outfit with the garment on the product page.

Mix and match takes one SKU and returns ranked outfits built around it from the organization's catalog. For a blazer, an outfit can add a top, trousers, shoes, a bag, and so on. The answer holds up to 5 outfits, best first. Every shopper token can call the route: it needs only the `shopper:session` scope from [step 4](04-shopper-session.md#check-the-session), and it uses no quota. It needs no profile, photo or privacy choice: it compares products, not the shopper.

## Add a product that completes the outfit

The walkthrough's catalog holds only the garment from step 3, so mix and match has nothing to suggest yet. Import one more product into the same collection, for another placement, such as trousers or shoes. Choose one for the same occasion as the garment, such as tailored trousers for a blazer, and give it the garment's gender: mix and match combines only products of the same gender. One product is enough to check the call; an outfit around a blazer with trousers also needs a top to wear under the blazer (see [outfits](#outfits)).

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

**Expected:** HTTP `200` with a JSON object whose `source` is `RULES`. In the walkthrough, `outfits` is empty when the product you added is trousers or shoes: an outfit around a blazer needs a top and a bottom, a dress or a set (see [outfits](#outfits)). Once the catalog holds a top of the garment's gender that goes with it, the trousers can appear as a `LOWER` item.

| Response field | Meaning |
| --- | --- |
| `source` | Always `RULES`: Irisphera built the [outfits](#outfits) with styling rules over the products' detected features and garment colors. |
| `outfits[]` | The outfits, best first. Empty when Irisphera cannot build an outfit around the garment. |
| `outfits[].rank` | The outfit's position: `1` for the best outfit, then `2`, `3` and so on |
| `outfits[].score` | How well the whole outfit goes together, from 0 to 1, with three decimals. Higher is better. The outfits come in score order. |
| `outfits[].items[]` | The products that complete the outfit, without the garment itself, in placement order: `UPPER`, `LOWER`, `FULL`, `FEET`, `HEAD`, `ACCESSORY`, `JEWELRY`. An outfit can hold two `UPPER` products, such as a top and a jacket, or a shirt and a sweater worn over it. |

Each product in `items[]` has these fields:

| Product field | Meaning |
| --- | --- |
| `placement` | The placement that the product fills in the outfit: `UPPER` for tops and outerwear, `LOWER`, `FULL` for one-piece items, suits and sets, `FEET`, `HEAD` for headwear and hair accessories, `JEWELRY`, and `ACCESSORY` for bags, belts, eyewear, scarves, ties, gloves, pocket squares and suspenders. |
| `skuCustomId`, `collectionId` | The product and the collection that holds it |
| `title`, `category`, `productPageUrl` | The product's details from the catalog, when they are stored |
| `matchScore` | How well the product goes with the garment, from 0 to 1, with three decimals. Higher is better. See [how Irisphera chooses the products](#how-irisphera-chooses-the-products). |
| `images[]` | `imageView` (`FEATURED`, `FRONT`, `BACK`, `OTHER` or `REFERENCE`) and `file`. `file.presignedUrl` is a temporary download URL. `file.path` is Irisphera's storage path, not a URL for the browser. |

The images come with temporary URLs unless you send `generatePresignedUrl=false`. Send `false` when your storefront shows its own product card for each `skuCustomId`. Do not store the temporary URLs.

| Answer | Meaning | Action |
| --- | --- | --- |
| `200` with outfits | Outfits built around the garment | Show them |
| `200` with `"outfits": []` | Irisphera cannot build an outfit around the garment: the garment cannot anchor an outfit, its features are not detected yet, or the catalog holds no products that complete an outfit with it | Hide the mix-and-match block |
| `404` | Unknown SKU, or not in this organization's catalog | Check the SKU against step 3 |

## How Irisphera chooses the products

Irisphera compares the features that it detected when it imported each product in [step 3](03-ingest-products.md), such as the category, type, style, occasion, fabric, pattern and fit, and the garment colors that it measured in the product images. The shopper is not part of the comparison, so every shopper gets the same answer for the same SKU until the catalog changes.

When Irisphera cannot build an outfit, the answer has no outfits. It never falls back to a simpler suggestion.

### Outfits

Irisphera builds outfits when the garment is a top, a bottom, a one-piece item such as a dress, outerwear, a suit, a set or a pair of shoes. This holds for every gender: `MEN`, `WOMEN`, `UNISEX`, `CHILDREN_GIRL` and `CHILDREN_BOY`.

- For each part that an outfit needs, such as the bottom and the shoes for a top, Irisphera picks the 5 products that go best with the garment. It then tries the combinations of one product for each part and returns up to 5 whole outfits, best first.
- Each outfit holds a top and a bottom, a one-piece item, a suit or a set, with shoes whenever shoes fit. It can add outerwear or a second top worn over the first, such as a sweater over a shirt, and up to 3 accessories, such as a bag, a belt or a tie. Outfits hold only products of the garment's gender. A `UNISEX` garment gets `UNISEX` products only, and the outfits of a `WOMEN` or `MEN` garment hold no `UNISEX` product.
- No minimum catalog size applies: one product for each part that an outfit must have is enough.
- The rules judge style, occasion, season, color, fabric, pattern and silhouette, and some combinations they never build. Sportswear, such as a hoodie, a track jacket or joggers, goes with casual products, such as jeans or a T-shirt, and never with an elegant product, such as tailored trousers or a pleated skirt. Activewear, such as a track jacket, joggers or a tracksuit, goes only with sneakers or slides. A product for sport or a workout never goes with a product for a dressy occasion, such as an evening, a wedding or a job interview, and a product only for dressy occasions, such as an evening jumpsuit, never goes with sneakers or slides. A blazer or a coat never goes over a hoodie. A summer product, such as sandals or shorts, never goes with a winter product, such as a puffer, a wool sweater or boots. A sweater goes over a top only when the top's collar shows, as a shirt's does, and a sweater with a high or turtle neck goes over no top. A tie goes only with a shirt whose collar folds down, never with sneakers, and under a sweater only if the sweater has a V neck. A suit goes over a shirt, a T-shirt or a fine knit, and takes neither flat open shoes, such as flat sandals, nor boots.
- Irisphera prefers variety. It picks the outfits best first, with a penalty for each product or type of shoes, such as sneakers, that an outfit shares with an outfit already picked. No garment or pair of shoes joins more than 2 outfits, unless that leaves fewer than 5 outfits, as in a catalog with a single pair of shoes.
- `matchScore` is the rules' score for the product and the garment as a pair: color, elegance, style, occasion, season, fabric, pattern and, for a top and a bottom, silhouette.

### Which products join an outfit

The product's detected `category`, and for some categories its detected type, decide the part it can fill:

| Category | Part of the outfit |
| --- | --- |
| `tops`, `shirt` | Top. A sweater, a cardigan or an overshirt can also go over another top. |
| `pants`, `jeans`, `trousers`, `shorts`, `skirt`, `joggers` | Bottom |
| `dress`, `jumpsuit`, `romper` | One-piece item |
| `jacket` | Outerwear. A hoodie or a knit, such as a sweater or a cardigan, joins as a top. |
| `suit`, `set` | The outfit's main garments. A suit also takes a top. |
| `shoes` | Shoes. Slippers do not join. |
| `bags`, `belts`, `eyewear`, `scarves`, `ties`, `gloves`, `headwear`, `hairaccessories` | An accessory, where it fits the outfit, such as a tie with a collared shirt |
| `earrings`, `necklaces`, `bracelets`, `rings`, `watches`, `anklets`, `brooches`, `cufflinks`, `jewelrysets` | Jewelry |
| `accessories` | Suspenders and pocket squares as accessories; other accessories do not join |
| `flowing` | A kimono joins as outerwear, a kaftan as a one-piece item and a tunic as a top; other flowing garments do not join |
| `swimwear`, `underwear`, `sleepwear`, `hosiery`, `bodyjewelry`, `unclassified` | Never |

Products detected for home only or for sleep, such as loungewear, slippers or a nightgown detected as a dress, never join an outfit. Accessories and jewelry complete outfits but cannot anchor one: for a bag, `outfits` is empty.

A product whose category Irisphera could not detect is `unclassified`. It stays in the catalog, but mix and match never uses it.

## Show the suggestions in your storefront

- Offer mix and match to every shopper. It does not depend on quota or on the recommendation feature, so it works even when `isApsEnabled` ([step 2](02-create-merchant.md#read-the-storefront-configuration)) is `false` or the token has no `shopper:recommendations` scope.
- The request uses no quota, so the product page can make it when it opens.
- Show each outfit with the garment and the outfit's products, best outfit first. To show fewer outfits, take the first ones.
- Show each product as a product card that links to its `productPageUrl`, or to your own product page for its `skuCustomId`.
- When the shopper opens a suggested product, its page sends the usual `PRODUCT_VIEWED` event ([step 8](08-collect-events.md)). There is no separate mix-and-match event.

## What Irisphera records

Nothing. Mix and match reads only the catalog. It works the same whatever the shopper chose in [step 4](04-shopper-session.md#record-the-shoppers-privacy-choices), and the report has no mix-and-match section.

**Checkpoint:** `mix-and-match.json` has `source` `RULES`. With the walkthrough's small catalog, `outfits` is empty; in a store with products that complete the garment, an outfit holds products for placements that the garment does not fill. Continue to [step 8](08-collect-events.md).

## Frequently asked questions

### Can we cache the suggestions?

Yes. Every shopper gets the same answer for the same SKU until the catalog changes, so your backend can cache it for each SKU and refresh it after an import. Request it with `generatePresignedUrl=false` and show your own product images, because the temporary image URLs expire.

### Can we leave out products, such as ones that are out of stock?

Not in the request. Irisphera has no stock information and suggests any product in the catalog. Filter the answer in your storefront before you show it. When you hide a product, hide the outfits that hold it and show the next outfits instead.

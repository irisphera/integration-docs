# 6. Recommendations and sizing

Previous: [Virtual try-on](05-virtual-try-on.md) · [Integration guide](../README.md) · Next: [Mix and match](07-mix-and-match.md)

**Goal:** estimate the participant's body measurements and colors from photos, build a shopper profile, get recommended products with sizes, and get the size of one product.

These routes need the `shopper:recommendations` scope from [step 4](04-shopper-session.md#check-the-session). The organization's `flowConfig.recommendationCriteria` ([step 2](02-create-merchant.md#create-the-organization)) decides which analyses a recommendation includes: palette, silhouette, sizing, all of them, or none.

A recommendation needs a **profile**: the shopper's gender, plus body measurements, colors or both. The storefront builds the profile from the shopper's answers, or from the two estimation routes below. This page uses the estimation routes:

1. Estimate body measurements from a full-body photo.
2. Extract skin, eye and hair colors from a selfie.
3. Put both into a profile.
4. Send the profile and get recommended products with sizes.
5. Send the gender and measurements of the profile and get the size of one product, without a recommendation.

## Estimate body measurements

The measurement route takes the shopper's height, a sizing category and a full-body front photo. An optional side photo goes in `img_data_side`. Images are raw base64, without a `data:` prefix.

Set the categories for the test participant. For a male participant, use `MEN` and `male`:

```bash
SIZING_GENDER='WOMEN'
PROFILE_GENDER='female'
read -rp 'Participant height in cm: ' HEIGHT_CM
python3 -c 'import base64,sys; sys.stdout.write(base64.b64encode(open(sys.argv[1],"rb").read()).decode())' \
  "$PHOTO_PATH" > photo.b64
jq -n --rawfile img photo.b64 --arg gender "$SIZING_GENDER" --argjson height "$HEIGHT_CM" \
  '{user_gender:$gender, user_height:$height, img_data_main:$img}' > measurements-request.json
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/shopper/v2/body-measurements" \
  -H "Authorization: Bearer $ACCESS_TOKEN" -H 'Content-Type: application/json' \
  --data-binary @measurements-request.json -o measurements.json
jq . measurements.json
```

**Expected:** HTTP `200` with `measurement_shoulders`, `measurement_bust`, `measurement_waist` and `measurement_hips`, in whole centimeters, rounded up. A `422` means the photo could not be processed: ask for another one. `user_gender` accepts `MEN`, `WOMEN`, `UNISEX`, `CHILDREN_GIRL` and `CHILDREN_BOY`.

## Extract colors

The color route takes a selfie and returns the skin, eye and hair colors as hex RGB. Use a separate face photo, or the same photo if the face is clearly visible:

```bash
read -rp 'Path of a selfie (Enter to reuse the try-on photo): ' SELFIE_PATH
SELFIE_PATH=${SELFIE_PATH:-$PHOTO_PATH}
python3 -c 'import base64,sys; sys.stdout.write(base64.b64encode(open(sys.argv[1],"rb").read()).decode())' \
  "$SELFIE_PATH" > selfie.b64
jq -n --rawfile img selfie.b64 '{img_data_main:$img}' > colors-request.json
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/shopper/v2/color-extraction" \
  -H "Authorization: Bearer $ACCESS_TOKEN" -H 'Content-Type: application/json' \
  --data-binary @colors-request.json -o colors.json
jq . colors.json
rm -f photo.b64 selfie.b64 measurements-request.json colors-request.json
```

**Expected:** HTTP `200` with `skin_color_hex`, `eyes_color_hex` and `hair_color_hex`. Both estimation routes send the photo to Irisphera's processing provider and return the result. They use no quota and record nothing. The measurements and colors describe the shopper's body, so treat them like the photo: keep them only as long as your storefront needs them.

## Build the profile

The profile is JSON, sent as base64. Base64 is an encoding, not encryption: treat the encoded profile as the shopper's personal data.

```bash
jq -n --slurpfile m measurements.json --slurpfile c colors.json --arg gender "$PROFILE_GENDER" \
  '{gender:$gender,
    bodyMeasurements:{shoulders:$m[0].measurement_shoulders, bust:$m[0].measurement_bust,
                      waist:$m[0].measurement_waist, hips:$m[0].measurement_hips},
    colorProfile:{hair:$c[0].hair_color_hex, eyes:$c[0].eyes_color_hex, skin:$c[0].skin_color_hex}}' \
  > profile.json
jq -c . profile.json | jq -R '{encodedProfileData:@base64}' > recommendations-request.json
```

| Profile field | Format |
| --- | --- |
| `gender` | `female` or `male`. Required when `bodyMeasurements` is present. |
| `bodyMeasurements` | `shoulders`, `bust`, `waist` and `hips` in centimeters. Omit it to skip silhouette and sizing. |
| `colorProfile` | `hair`, `eyes` and `skin` as hex RGB, such as `#6b4a3a`. Omit it to skip the palette. |

A shopper who types measurements into your storefront instead of uploading a photo gives the same profile. Send only what the shopper provided.

## Get recommendations

```bash
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/shopper/v2/recommendations?generatePresignedUrl=true" \
  -H "Authorization: Bearer $ACCESS_TOKEN" -H 'Content-Type: application/json' \
  --data-binary @recommendations-request.json -o recommendations.json
jq '{silhouette:.silhouette.silhouetteClassification, palette:.palette.paletteClassification,
     sizing:.generalSizing.classification,
     recommended:[.recommendationsByCollection[]? | {collection:.collection.title,
       items:[.recommendations[] | {skuCustomId,size,productPageUrl}]}]}' recommendations.json
```

**Expected:** HTTP `200` with the analyses the organization enables and the recommended products, grouped by collection.

| Response field | Meaning |
| --- | --- |
| `silhouette.silhouetteClassification` | Body shape: `rectangle`, `triangle`, `inverted_triangle`, `hourglass`, `apple`, `pear`, `oval` or `trapeze`. Needs measurements. |
| `palette.paletteClassification` | Color palette: `cold`, `warm`, `soft`, `delicate`, `dark` or `contrasting`. Needs colors. |
| `generalSizing` | `classification.upper` and `classification.lower` for tops and bottoms, with `probabilities` of the size being lower, equal or higher |
| `recommendationsByCollection[].recommendations[]` | `skuCustomId`, the recommended `size` in the organization's size system when known, `images` and `productPageUrl` |

With `generatePresignedUrl=true` the images come as temporary URLs; do not store them.

Rules for recommendation requests:

- Each request uses one unit of recommendation quota. Request recommendations when the shopper asks for them, not on every page load.
- The contract lists `offset`, `limit` and `collectionIds`, but Irisphera currently ignores them. Do not rely on them.
- Omit `filters` unless you have agreed filter values with Irisphera.
- A `403` means the token has no `shopper:recommendations` scope: hide the recommendation entry point. Check `isApsEnabled` ([step 2](02-create-merchant.md#read-the-storefront-configuration)) before offering it.

## Get the size of one product

Three routes give the size of one product without a recommendation, for example on the product page. A product can get a size only when it has a **sizing specification**: a size chart that Irisphera picks for the product. Irisphera picks size charts for clothing for men, women and unisex, not for shoes, hosiery, accessories, jewelry or children's clothing.

| Route | Answer |
| --- | --- |
| `GET /shopper/v2/sizing` | The SKUs of the organization's products that have a sizing specification, as a JSON array, each once, in alphabetical order. The array is empty when no product has one. |
| `GET /shopper/v2/sizing/{skuCustomId}` | `true` when the product has a sizing specification, `false` when it has none |
| `POST /shopper/v2/sizing/{skuCustomId}` | The product's size for the shopper's gender and body measurements, such as `{"size": "M"}` |

List the SKUs that can get a size, and check the garment from [step 3](03-ingest-products.md):

```bash
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/shopper/v2/sizing" \
  -H "Authorization: Bearer $ACCESS_TOKEN" -o sized-skus.json
jq --arg sku "$SKU" '{sizedSkus: length, garmentListed: any(.[]; . == $sku)}' sized-skus.json
curl --fail-with-body -sS "$IRISPHERA_BASE_URL/shopper/v2/sizing/$SKU" \
  -H "Authorization: Bearer $ACCESS_TOKEN" -w '\n'
```

**Expected:** HTTP `200` for both. The check answers `true` when the list holds the garment's SKU, and `false` when it does not. A `404` means that the organization has no product with this SKU.

Then send the gender and body measurements of the profile, and get the garment's size:

```bash
jq '{gender, bodyMeasurements}' profile.json > sizing-request.json
curl -sS -X POST "$IRISPHERA_BASE_URL/shopper/v2/sizing/$SKU" \
  -H "Authorization: Bearer $ACCESS_TOKEN" -H 'Content-Type: application/json' \
  --data-binary @sizing-request.json -o size.json -w 'HTTP %{http_code}\n'
jq . size.json
rm -f sizing-request.json
```

**Expected:** HTTP `200` with the garment's `size` in the organization's size system, such as `M` or `42`. It is the size that a recommendation gives the same product for the same measurements. When the garment has no sizing specification, the answer is `422`, which is fine here.

| Request field | Format |
| --- | --- |
| `gender` | Required. `female` or `male`. |
| `bodyMeasurements` | Required. `shoulders`, `bust`, `waist` and `hips` in centimeters, each at least 1. Irisphera rounds a fraction up to whole centimeters. |

| Status | Meaning | What to do |
| --- | --- | --- |
| `200` | `size` holds the product's size. | Show the size. |
| `400` | `gender` or a measurement is missing or not valid. | Send `female` or `male` and all four measurements. |
| `403` | The token has no `shopper:recommendations` scope. | Hide the size entry point. |
| `404` | The organization has no product with this SKU. | Check the SKU. |
| `422` | The product has no sizing specification, the organization's size system gives it no size, or no size of its size chart fits the measurements. The problem's `detail` says which. | Show that no size was found. Do not parse `detail`. |

Rules for size requests:

- Each `POST` that returns a size uses one unit of recommendation quota. A `422` and the two `GET` routes use none.
- Ask for a size only for a SKU that has a sizing specification. Read the list once per session, or check the SKU when the product page opens.
- A `422` for some measurements does not tell whether the product is too small or too large for the shopper. Do not suggest another size in its place.

## What Irisphera records

The recommendation itself works whatever the shopper chose in [step 4](04-shopper-session.md#record-the-shoppers-privacy-choices). What Irisphera keeps depends on the choice:

| The shopper grants | Irisphera |
| --- | --- |
| `analytics` and `personalization` | Records the recommendation, including the profile and the result, and uses the shopper's earlier activity to personalize later recommendations. The report's `users` section counts these shoppers by palette and silhouette. |
| Anything less | Records nothing and uses only the profile in the request |

A withdrawal applies from the next request. Irisphera checks the choice again for every request.

The size routes record nothing, whatever the shopper chose. Irisphera uses the gender and measurements of a size request only to compute the size, and does not store them.

**Checkpoint:** `recommendations.json` lists recommended SKUs, and `size.json` holds the garment's size, or a `422` problem when the garment has no sizing specification. Continue to [step 7](07-mix-and-match.md).

## Frequently asked questions

### Do shoppers have to upload photos to get recommendations?

No. The profile holds body measurements, colors or both, and the shopper can type the measurements and choose the colors in your storefront. The two photo routes are only one way to fill in the profile.

### Why is the silhouette, palette or sizing missing from the answer?

Each analysis needs its input, and the organization's `recommendationCriteria` must include it. Silhouette and sizing need `bodyMeasurements`. The palette needs `colorProfile`.

### Why does a product get no size?

A product gets a size only when it has a sizing specification and the organization's size system sizes it. `GET /shopper/v2/sizing` lists the SKUs with a sizing specification. If a clothing SKU that you expect is not listed, ask Irisphera. A product with a sizing specification can still get no size for some measurements. Then `POST /shopper/v2/sizing/{skuCustomId}` answers `422`, and a recommendation gives the product no size either.

### Why is a product never recommended?

A product appears in recommendation results only when it is listed and has a product page URL ([product rules](03-ingest-products.md#product-rules)). Products that you no longer sell can still appear until you delete them, because imports never remove products ([step 3](03-ingest-products.md#how-do-we-remove-a-product-that-we-no-longer-sell)). Until then, filter the results against your own catalog.

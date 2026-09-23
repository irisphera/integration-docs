# 6. Recommendations and sizing

Previous: [Virtual try-on](05-virtual-try-on.md) · [Integration guide](../README.md) · Next: [Mix and match](07-mix-and-match.md)

**Goal:** estimate the participant's body measurements and colors from photos, build a shopper profile, and get recommended products with sizes.

These routes need the `shopper:recommendations` scope from [step 4](04-shopper-session.md#check-the-session). The organization's `flowConfig.recommendationCriteria` ([step 2](02-create-merchant.md#create-the-organization)) decides which analyses a recommendation includes: palette, silhouette, sizing, all of them, or none.

A recommendation needs a **profile**: the shopper's gender, and body measurements, colors or both. The storefront builds it from the shopper's answers or from the two estimation routes below.

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

With `generatePresignedUrl=true` the images come as temporary URLs; do not store them. `flowConfig.recommendationTopK` tells your storefront how many of the results to show.

Rules for recommendation requests:

- Each request uses one unit of recommendation quota. Request recommendations when the shopper asks for them, not on every page load.
- The contract lists `offset`, `limit` and `collectionIds`, but Irisphera currently ignores them. Do not rely on them.
- Omit `filters` unless you have agreed filter values with Irisphera.
- A `403` means the token has no `shopper:recommendations` scope: hide the recommendation entry point. Check `isApsEnabled` ([step 2](02-create-merchant.md#manage-your-organizations)) before offering it.

## What Irisphera records

The recommendation itself works whatever the shopper chose in [step 4](04-shopper-session.md#record-the-shoppers-privacy-choices). What Irisphera keeps depends on the choice:

| The shopper grants | Irisphera |
| --- | --- |
| `analytics` and `personalization` | Records the recommendation, including the profile and the result, and uses the shopper's earlier activity to personalize later recommendations. The report's `users` section counts these shoppers by palette and silhouette. |
| Anything less | Records nothing and uses only the profile in the request |

A withdrawal applies from the next request. Irisphera checks the choice again for every request.

**Checkpoint:** `recommendations.json` lists recommended SKUs. Continue to [step 7](07-mix-and-match.md).

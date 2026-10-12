# Piltover Spoiler Workflow

Primary source for spoiler ingestion:

```text
https://piltoverarchive.com/cards?sets=<SET>
```

Known set codes in current use:

- `UNL` = Unleashed
- `VEN` = Vendetta
- `RAD` = Radiance

Primary tool:

- [piltover_spoilers_to_sheet.py](piltover_spoilers_to_sheet.py)

## What The Script Does

It scrapes Piltover Archive's client-rendered payload and extracts:

- `card-id`
- `card_name`
- `effect`
- `set`
- `image_url`
- `tts-type`
- `rarity`
- `domain`
- `might`
- `cost`
- `power`
- `equip_might`
- `flavor`
- `artist`

It can also:

- upsert rows into `Card Data`
- download card images into the TTS image folder

## Normal Workflow

1. Preview current scrape output.
2. Upsert to the sheet.
3. Pull images if needed.
4. Only do manual follow-up for known upstream mistakes.

## Commands

Preview a set scrape:

```bash
python3 riftbound-sheet-docs/piltover_spoilers_to_sheet.py --set VEN --json
```

Write or update `Card Data` rows:

```bash
python3 riftbound-sheet-docs/piltover_spoilers_to_sheet.py --set VEN --append-sheet
```

Download images into the TTS mod:

```bash
python3 riftbound-sheet-docs/piltover_spoilers_to_sheet.py --set VEN --download-jpg-dir riftbound_jpg
```

Repeat the same with `UNL` or `RAD` as needed.

Piltover's current React payload contains `variants` arrays with `variantNumber`,
`rarity`, `imageUrl`, `flavorText`, `artist`, and a nested `card` object.
The importer reads these directly, including all paginated results; the older
rendered-component parser remains a fallback. Domains use `|`; absent might and
power remain blank. Existing card-ID filters still exclude unsupported promos.

For RAD, Riftbound.org fills IDs absent from Piltover. Read the JSON in
`<script data-page="app" type="application/json">` on
`https://riftbound.org/cards/rad-radiance`, following `props.cards.links.next`.
`props.cards.data` contains stats, HTML rules, artist, and image URLs. Preview
rarity becomes `Unknown`; preview images are `temporary`. Piltover wins when
both sources contain an ID. RAD supports basic runes and signed alt-art IDs
(`174a*` becomes `174as`); SP promos remain excluded.

## Sheet Behavior

`--append-sheet` is an upsert keyed by `card-id`.

- missing `card-id` rows are appended
- existing `card-id` rows are updated in place
- formula columns are intentionally not written

Current main destination:

- spreadsheet tab: `Card Data`

Current direct-write fields:

- `card_name`
- `card-id`
- `effect`
- `tts-type`
- `rarity`
- `domain`
- `might`
- `cost`
- `power`
- `equip_might`
- `flavor`
- `artist`
- `set`
- `image_url`

For exact live column meanings, read [card-data.md](card-data.md).

## Image Behavior

Target dir for TTS:

```text
riftbound_jpg
```

Behavior:

- writes `<card-id>.jpg`
- skips files that already exist
- fetches source art from Piltover's current `image_url`
- converts source files to JPEG with `/usr/bin/sips`

Important:

- the image pass is separate from sheet sync
- syncing the sheet does not download images
- if no `VEN` JPGs exist yet, a `VEN` image pass will pull the full currently indexed set

## Current Conventions

- Normalize Piltover collector ids ending in `*` to `s`.
- Skip malformed promo-style collector ids like extra `-Champion` suffix variants. Wait for the real base variant.
- `tts-type` follows TTS conventions, not always literal Piltover text.

Current parser normalizations include:

- `Champion Unit` / `Champion Spell` / `Champion Gear` -> `Champion`
- `Signature Spell` -> `Spell`
- `Token Unit` -> `Unit`
- `Token Rune` -> `Rune`
- `Token Card` -> `Rune`
- `Unit Gear` -> `Unit`

## Auth

Sheet writes use saved Google OAuth credentials:

- [google-oauth-client.json](google-oauth-client.json)
- [google-oauth-token.json](google-oauth-token.json)

Reauth details are in [auth.md](auth.md).

## Known Failure Modes

- Piltover sometimes publishes bad text, wrong stats, or temporary image URLs.
- Showcase rows and base rows may appear at different times.
- A token or special card may change its displayed type label without warning.
- Some values may regress upstream between runs.

If the scrape looks wrong:

1. confirm the bad value on Piltover first
2. decide whether to wait for upstream to fix it
3. only hardcode a workaround when needed for active work

## Current Status Notes

### RAD name audit (2026-10-12)

Compared imported RAD names with Riot's English card-gallery data at
https://playriftbound.com/en-us/card-gallery/ and checked local card-image text.
Keep the established TTS `Champion, Legend Title` naming convention for legends;
the gallery's title-only legend names are not import errors.

Persistent overrides in `CARD_OVERRIDES` correct:

- `RAD-011`: `Incorrigible Construct` (user correction; available preview image
  is Chinese, so the English name was not independently confirmed).
- `RAD-109`, `RAD-109a`: `Evelynn, In Control` (official capitalization).
- `RAD-114`, `RAD-114a`: `Riven, Deserter` (missing comma).
- `RAD-163`: `Hunters' Circle` (plural possessive).

The English images confirm `Strigian Oracle` (`RAD-053`), `Armored Tusker`
(`RAD-078`), and `Protector's Shield` (`RAD-123`), despite conflicting
Riftbound.org names. Retain those printed English names. Chinese-only preview
names remain provisional until English art or official records become available.

- `RAD` is still sparse on Piltover; only scrape what is actually indexed there
- `VEN` is actively expanding; reruns can legitimately append more rows
- `UNL` remains the most mature path and is the baseline behavior model

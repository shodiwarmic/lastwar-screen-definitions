# Last War Screen Definitions

YAML configuration files that describe every ranked leaderboard screen in *Last War: Survival*. Adapters (such as the OCR service) consume these definitions at runtime to classify screenshots, locate tab regions, and extract player data — with no code changes needed when UI details change.

---

## Repository Layout

```
catalog.yaml                  Ordered list of all screen definitions
meta-schema.json              JSON Schema validating every definition file
screens/
  daily_ranking.yaml          Daily Rank (Mon–Sat VS points)
  weekly_ranking.yaml         Weekly Rank (7-day cumulative)
  strength_ranking.yaml       Strength Ranking (Power / Kills / Donation)
  season_contribution.yaml    Season Contribution Ranking (Mutual Assistance / Siege / Rare Soil War / Defeat)
```

---

## catalog.yaml

Lists every screen in the order adapters must test them. **Priority order matters** — some OCR signals are shared across screens (e.g. "Weekly" appears on both Weekly and Daily screens), so the more-specific screen must be checked first.

```yaml
schema_version: 1
screens:
  - id: strength_ranking
    file: screens/strength_ranking.yaml
    priority: 1
  - id: weekly_ranking
    file: screens/weekly_ranking.yaml
    priority: 2
  - id: daily_ranking
    file: screens/daily_ranking.yaml
    priority: 3
  - id: season_contribution
    file: screens/season_contribution.yaml
    priority: 4
```

| Field | Description |
|---|---|
| `id` | Must match the `id` field inside the YAML file |
| `file` | Path relative to this directory |
| `priority` | Lower = checked first. Gaps are fine (1, 2, 10…) |

---

## Schema Reference

All fractions are normalised to `[0.0, 1.0]` of the image dimension unless otherwise noted.

### Top-level fields

| Field | Required | Description |
|---|---|---|
| `id` | ✓ | Unique identifier. Used as a key in catalog.yaml and in code lookups. |
| `version` | ✓ | Integer. Increment when making breaking changes. |
| `name` | ✓ | Human-readable display name. |
| `description` | | Free-text description of the screen. |
| `identification` | ✓ | Signals used to recognise this screen (see below). |
| `boundaries` | ✓ | Header and footer anchors used to crop the player list. |
| `chrome` | | UI chrome fractions to remove before stitching (deprecated in stitch-first pipeline). |
| `tabs` | | Tab bar configuration for screens with multiple data views. |
| `columns` | ✓ | Column layout — where rank, name, and score live. |
| `row_clustering` | ✓ | How OCR word blocks are grouped into player rows. |

---

### `identification`

```yaml
identification:
  page_signals:
    - "Strength Ranking"
    - "STRENGTH"
  negative_signals:
    - "Mon."
    - "Tues."
  pre_ocr_hint:
    x_hint: 0.15
    y_hint: 0.20
    color:
      named: orange
      hsv_override: {h_min: 0.014, h_max: 0.153, s_min: 0.40, v_min: 0.55}
    confidence: 0.90
```

| Field | Description |
|---|---|
| `page_signals` | OCR text tokens that confirm this is the right screen. All words of any signal must appear in the OCR output for a match. |
| `negative_signals` | Tokens whose presence rules this screen out (e.g. day abbreviations rule out Weekly when checking Daily). |
| `pre_ocr_hint` | Pass 1 colour-sampling hint. Sample the pixel at `(x_hint, y_hint)` and check whether it matches `color`. If it does, this screen is likely active. |
| `pre_ocr_hint.confidence` | Confidence score returned on a Pass 1 match. |

**`color`**

| Field | Description |
|---|---|
| `named` | Logical colour name (`orange` or `white`). Used as a fallback if no `hsv_override` is provided. |
| `hsv_override` | HSV thresholds for a pixel match: `h_min`/`h_max` (hue 0–1), `s_min` (saturation floor), `v_min` (brightness floor). |

---

### `boundaries`

```yaml
boundaries:
  header:
    signals: ["Commander", "Points"]
    search_region: {y_min: 0.0, y_max: 0.40}
  footer:
    signals: ["Your Alliance"]
    search_region: {y_min: 0.60, y_max: 1.0}
```

Boundary anchors locate the top and bottom of the player list by searching for OCR text signals within the given region. Used to crop screenshots before stitching.

| Field | Description |
|---|---|
| `signals` | One or more OCR text strings that mark this boundary. |
| `search_region` | Fraction of the image height to scan. Narrows the search to avoid false matches. |

---

### `tabs`

Describes the tab bar. Required for screens with multiple data views (all three current screens).

```yaml
tabs:
  search_region: {y_min: 0.00, y_max: 0.30}
  y_hint: 0.20
  active_indicator:
    strategy: color_fraction   # or "brightest"
    min_fraction: 0.10
    min_gap: 0.04
    color:
      named: orange
      hsv_override: {h_min: 0.014, h_max: 0.153, s_min: 0.40, v_min: 0.55}
    bbox_padding_fraction: 0.007
  items:
    - {id: power,  category: power,  signals: ["Power"],  x_hint: 0.15}
    - {id: kills,  category: kills,  signals: ["Kills"],  x_hint: 0.50}
```

**`active_indicator`**

| Field | Description |
|---|---|
| `strategy` | `color_fraction` — active tab has a solid colour (e.g. orange) covering ≥ `min_fraction` of its crop. `brightest` — active tab has a higher V-channel brightness than inactive tabs (used for day tabs which are white, not orange). |
| `min_fraction` | (color_fraction only) Minimum fraction of pixels that must match `color` to call a tab active. |
| `min_gap` | (brightest only) Minimum brightness difference (V channel, 0–1) between the brightest and second-brightest tab to declare a winner. |
| `color` | Colour definition for `color_fraction` detection. |
| `bbox_padding_fraction` | Pixels to expand around each tab's OCR bounding box before sampling, expressed as a fraction of image width. Ensures the tab background rather than the text glyph is sampled. |

**Tab items**

| Field | Description |
|---|---|
| `id` | Internal identifier. |
| `category` | Output key returned by the classifier (e.g. `"kills"`, `"donation_daily"`). |
| `signals` | OCR text tokens for this tab. The **first** signal is the top-level tab label. A **second** signal marks a sub-tab (e.g. `["Donation", "Daily"]` means the "Daily" sub-tab under the "Donation" top tab). |
| `x_hint` | Horizontal sample position for Pass 1 colour detection, as a fraction of image width. |

**Sub-tabs** — when multiple items share the same first signal, they represent sub-tabs of a single top-level tab. The classifier detects the active top-level tab by colour, then distinguishes sub-tabs using the `brightest` strategy on the second-signal labels.

---

### `columns`

```yaml
columns:
  - {id: rank,  type: ignore, x_min: 0.00, x_max: 0.15}
  - {id: name,  type: name,   x_min: 0.15, x_max: 0.60}
  - {id: score, type: score,  x_min: 0.60, x_max: 1.00}
```

Divides the horizontal space into named regions. The extractor uses `type: name` and `type: score` to know which OCR tokens to keep.

| `type` | Behaviour |
|---|---|
| `name` | Tokens in this x-range are candidate player name parts. |
| `score` | The rightmost numeric token in this x-range is the score. |
| `ignore` | Tokens in this x-range are discarded (rank numbers, badges). |

---

### `row_clustering`

Controls how the extractor groups flat OCR word blocks into player rows.

```yaml
row_clustering:
  strategy: score_anchored
  score_anchored:
    up_band_fraction: 0.021    # collect name tokens this far ABOVE the score
    down_band_fraction: 0.002  # ...and this far BELOW (prevents alliance subtitles)
  y_proximity:
    tolerance_fraction: 0.02
    min_tolerance_px: 20
  min_score: 1000              # ignore numeric tokens below this value
  word_gap_fraction: 0.015     # gap/image_width threshold to insert a space
  min_word_gap_px: 8           # minimum gap in pixels regardless of image width
```

| Field | Description |
|---|---|
| `strategy` | `score_anchored` — anchor each row on its score token, collect name tokens within the band above. Prevents alliance subtitle lines from merging into the player row. |
| `score_anchored.up_band_fraction` | Fraction of image height to search upward from the score for name tokens. |
| `score_anchored.down_band_fraction` | Fraction of image height to search downward (small — just handles slight vertical misalignment). |
| `min_score` | Minimum integer value for a token to be treated as a score. Filters out rank numbers (1–100) and OCR noise. |
| `word_gap_fraction` | Gap between adjacent OCR bounding boxes, relative to image width, above which a space is inserted between name tokens. |
| `min_word_gap_px` | Absolute minimum gap in pixels. Overrides `word_gap_fraction` on narrow images. |

---

## Known OCR Challenges

### Name/score crash tokens

On screens where player names end in digits (e.g. `Ruthless5432`, `CheeseKillers2`, `Blindman03`), the OCR engine sometimes renders the name and its score as a single merged token — for example `Ruthless54323,045,000` instead of separate `Ruthless5432` and `3,045,000` tokens.

**Why this happens:** The game renders the name and score in adjacent text spans with no guaranteed gap. When the name's trailing digit is visually close to the score's leading digit, OCR groups them into one bounding box.

**Detecting crash tokens:** A token is a crash candidate when it contains at least one alpha character and its suffix matches the pattern `\d{1,3}(?:,\d{3})+` (a comma-grouped integer). The split point is the rightmost such suffix whose left remainder (the name prefix) contains no comma.

**Ambiguity:** Some names allow multiple valid splits. `Ruthless54323,045,000` splits as:

| Name prefix | Score |
|---|---|
| `Ruthless5432` | `3,045,000` |
| `Ruthless543` | `23,045,000` |
| `Ruthless54` | `323,045,000` |

Adapters should:
1. Use the **rightmost split** (smallest score) as the default heuristic.
2. Validate middle rows against adjacent scores — the leaderboard is sorted descending, so a row's score must fall between its neighbours. If the heuristic fails this bound, try alternatives in ascending score order until one fits.
3. When ambiguity cannot be resolved from position alone (typically rank 1), expose **all valid splits** to the consumer so they can resolve using an external member roster.

**Consumer contract (recommended):** Return ambiguous rows with a `candidates` array ordered by ascending score. `candidates[0]` must match the top-level heuristic name and score. Consumers should prefer an exact roster match; fall back to `candidates[0]` if none is found. Always take **both** the name and score from the winning candidate — the score is different for each split.

---

## Adding a New Screen

1. **Create the YAML file** in `screens/` following the schema above. Set a new `id` and increment the `version` to `1`.

2. **Register it in `catalog.yaml`** with an appropriate `priority`. Check existing priorities and insert the new screen where its `page_signals` won't shadow a more-specific screen that should match first.

3. **Validate against the schema** (optional but recommended):
   ```bash
   python -c "
   import json, yaml, jsonschema
   schema = json.load(open('meta-schema.json'))
   defn   = yaml.safe_load(open('screens/your_new_screen.yaml'))
   jsonschema.validate(defn, schema)
   print('Valid')
   "
   ```

4. **Add test fixtures** in the OCR service — capture a real screenshot with `tools/capture_ocr_fixture.py` and name the file to include the new category keyword so tests can auto-discover it.

---

## Modifying an Existing Screen

- **Colour thresholds** (`pre_ocr_hint.color.hsv_override`, `tabs.active_indicator.color.hsv_override`): adjust if a game update changes the UI colour palette.
- **Tab positions** (`tabs.items[].x_hint`, `tabs.y_hint`): adjust if a UI update moves the tab bar.
- **Row clustering** (`row_clustering.*`): adjust `up_band_fraction` if alliance subtitles are appearing in player names (increase), or if player names on multi-line rows are being split (decrease `down_band_fraction`).
- **`min_score`**: increase if rank numbers or small UI values are being captured as scores; decrease if real scores drop below the current threshold (e.g. on Donation screens with small point totals).

After any change, increment the file's `version` field and run the OCR service test suite to verify nothing regressed.

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
  strength_metrics.yaml       Strength Ranking — Power / Kills row-1 tabs
  strength_donation.yaml      Strength Ranking — Donation row-1 tab + Daily/Weekly sub-tabs
  season_contribution.yaml    Season Contribution Ranking (Mutual Assistance / Siege / Rare Soil War / Defeat)
```

---

## catalog.yaml

Lists every screen in the order adapters must test them. **Priority order matters** — some OCR signals are shared across screens (e.g. "Weekly" appears on both Weekly and Daily screens), so the more-specific screen must be checked first.

```yaml
schema_version: 1
screens:
  - id: strength_donation
    file: screens/strength_donation.yaml
    priority: 1
  - id: strength_metrics
    file: screens/strength_metrics.yaml
    priority: 2
  - id: weekly_ranking
    file: screens/weekly_ranking.yaml
    priority: 3
  - id: daily_ranking
    file: screens/daily_ranking.yaml
    priority: 4
  - id: alliance_contribution
    file: screens/season_contribution.yaml
    priority: 5
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
| `pre_ocr_hint` | Fast single-pixel colour check run **before** OCR. The service samples the pixel at `(x_hint * width, y_hint * height)` and checks it against the HSV range. A frame is skipped entirely if **every** layout with a `pre_ocr_hint` fails its check. Layouts with `pre_ocr_hint: null` are never skipped — they always allow OCR to proceed. |
| `pre_ocr_hint.confidence` | Reserved for future use. Currently unused; include it in new definitions for forward compatibility. |

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
| `signals` | OCR text strings that mark this boundary. May be an empty array (`[]`) — see "Empty signals" below. |
| `search_region` | Fraction of the image height to scan. Narrows the search to avoid false matches. |

**Empty signals.** When `signals: []`, the boundary has no text anchor; consumers must fall back to `chrome.bottom_fraction` (footer) or `chrome.top_fraction` (header), or — when `chrome` is also absent — to the image edge. `season_contribution.yaml` uses this for its footer because that screen has no "Your Alliance" line, only the user's pinned own-row above the nav bar.

---

### `tabs`

Describes the tab bar. Required for screens with multiple data views (all three current screens).

```yaml
tabs:
  search_region: {y_min: 0.00, y_max: 0.30}  # both bounds enforced
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
| `color` | Per-layout HSV thresholds for tab active-indicator detection. When `hsv_override` is provided, consumers MUST use it; otherwise consumers fall back to the canonical RGB constants documented in the Consumer Contract section below (orange / white). |
| `bbox_padding_fraction` | Pixels to expand around each tab's OCR bounding box before sampling, expressed as a fraction of image width. Ensures the tab background rather than the text glyph is sampled. |

**Tab items**

| Field | Description |
|---|---|
| `id` | Internal identifier. Only used for logging; the classifier returns `category` as the output key. |
| `category` | Output key returned by the classifier and stored as the `day` value in the database (e.g. `"kills"`, `"donation_daily"`). Falls back to `id` when blank. |
| `signals` | **Alternative** OCR text tokens for this tab — any one signal matching is sufficient. Each signal is a space-separated sequence of words that must ALL appear in the OCR line for that signal to match (e.g. `["Weekly Rank"]` requires both "Weekly" and "Rank" to be present). Different signals on the same item are OR-ed together. |
| `x_hint` | Horizontal centre of the tab button as a fraction of image width. Used to select the correct OCR element when multiple tab labels appear on one line, and as the colour-sample centre for active-tab detection. |
| `group` | Non-empty only on layouts with multiple independent tab rows (e.g. `alliance_contribution` has `category` and `period` groups). Each group's winner is detected with that group's `tabs.groups[name]` config; winners are joined with `_` in declaration order (e.g. `"siege_daily"`). Omit for single-row tab layouts. |

**`groups`** *(optional, multi-row tabs only)*

```yaml
tabs:
  groups:
    category: {strategy: color_fraction, min_fraction: 0.10}
    period:   {strategy: brightest,      min_fraction: 0.02}
```

Maps each `tab_item.group` name to its own active-indicator config (`strategy`, `min_fraction`). Required when any `tab_item` has a non-empty `group`. The top-level `tabs.active_indicator` is still parsed but only used for groups not listed here. Used by `season_contribution.yaml` to detect the active category (orange-filled) and period (brightest white text) tabs independently.

**When `tabs.groups` applies — and when it doesn't.** Use `groups` only when the wire-format category is a uniform `{group_winner}_{group_winner}` join across every state of the screen. `season_contribution` qualifies — every state emits `{category}_{period}` (e.g. `siege_daily`, `mutual_assistance_season`).

Two existing screens have multi-row UIs but *non-uniform* category emission, and are therefore intentionally split into separate screen YAMLs disambiguated by `negative_signals`:

- **Daily VS / Weekly VS** (`daily_ranking.yaml` + `weekly_ranking.yaml`) — Daily emits a day name alone (`friday`); Weekly emits the period name alone (`weekly`). No `_period` component on either side. Disambiguated by the day-tab row disappearing when Weekly is active (`negative_signals: ["Mon.", "Tues.", ...]` on `weekly_ranking`).
- **Strength Ranking** (`strength_metrics.yaml` + `strength_donation.yaml`) — Power and Kills emit single names (`power`, `kills`); Donation opens a sub-tab row that emits `donation_daily` or `donation_weekly`. Disambiguated by the sub-tab row appearing only when Donation is active (`negative_signals: ["Daily Weekly"]` on `strength_metrics` rejects the Donation state because both labels appear together; `page_signals: ["Strength Daily Weekly"]` on `strength_donation` requires both to appear).

A `groups`-based model for either case would require a schema extension (conditional groups, per-winner category overrides) or a backend migration that re-keys established categories. The split-screen pattern leverages real UI signals (whichever sub-row the game shows) and uses only existing schema features.

**Rule of thumb:** if *every* state of a multi-row screen emits a category that is exactly `{winner_a}_{winner_b}`, use `tabs.groups`. Otherwise model each row-1 state as its own screen and disambiguate via `negative_signals` on the labels of the conditional second row.

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
| `strategy` | `score_anchored` — for each score token found in the score column, collect name tokens within a vertical band above it. Prevents footer rows (e.g. "Your Alliance / [Tag] AllianceName") from being captured because they have no associated score token. Name lines with no score anchor undergo crash-token recovery. `y_proximity` (or any other value) — fallback: group all OCR lines by vertical proximity then extract columns from each group. |
| `score_anchored.up_band_fraction` | Fraction of **image height** to search upward from the score token's top edge for associated name tokens. |
| `score_anchored.down_band_fraction` | Fraction of image height to search downward from the score token's bottom edge (small — handles slight vertical misalignment between name and score OCR boxes). |
| `y_proximity.tolerance_fraction` | Fraction of image height used as the vertical tolerance when grouping OCR lines into the same player row. |
| `y_proximity.min_tolerance_px` | Minimum tolerance in pixels (prevents the tolerance from being too tight on small screens). |
| `min_score` | Minimum integer value for a token to be treated as a score. Filters out rank numbers (1–100) and OCR noise. |
| `word_gap_fraction` | Horizontal gap between adjacent OCR bounding boxes, relative to image width, **above which** a space is inserted between name tokens. Gaps at or below this threshold result in direct concatenation (handles OCR splitting a single word across two boxes). |
| `min_word_gap_px` | Absolute minimum threshold in pixels, used when `word_gap_fraction * image_width` would be smaller (narrow screens). The effective threshold is `max(word_gap_fraction * width, min_word_gap_px)`. |

---

## Consumer Contract

This section is the source of truth for any adapter that consumes these definitions (today: `lastwar-ocr-service` in Python and `lastwar-android-scanner` in Kotlin; tomorrow: anything else). When the contract here disagrees with code, the contract is right and the code is broken. Open an issue.

### Pipeline stages

Every consumer MUST implement these stages, in this order, for every captured frame:

1. **Load definitions.** Parse `catalog.yaml` and each per-screen YAML referenced by it. Sort by `catalog.yaml:screens[].priority` ascending — lower number is checked first. Validate each parsed file against `meta-schema.json` at startup; failure aborts boot.
2. **Detect game window.** All YAML normalised fractions (`x_hint`, `search_region`, etc.) assume the game UI fills the captured image. On devices that capture wider canvases — most importantly the Pixel Fold's inside-landscape mode, where Android renders the game in a portrait sub-window inside a landscape screenshot — the game occupies only a portion of the image with black bars or system chrome filling the rest. Detect that game-content rectangle and **crop the captured image to it** before any subsequent stage runs. See *Game-window detection* below for the algorithm. Front-screen and edge-to-edge captures pass through unchanged because no borders are detected.
3. **Pre-OCR hint** *(optional, when `identification.pre_ocr_hint` is non-null).* Sample the single pixel at `(x_hint * width, y_hint * height)` of the **cropped** image and test it against `color.hsv_override` if present (otherwise the named-colour fallback below). If **every** layout with a non-null hint fails its check, the frame is skipped — no OCR runs. Layouts with `pre_ocr_hint: null` always allow OCR to proceed.
4. **OCR.** Run word-level text recognition on the cropped image. Engine choice is consumer-local (Cloud Vision, ML Kit, Tesseract, …); the contract operates on `(text, bounding_box)` pairs and is engine-agnostic.
5. **Screen identification.** For each layout in priority order: a layout matches when **every** word of **at least one** `identification.page_signals` entry appears in the OCR text (case-insensitive, space-separated tokens; e.g. signal `"Daily Rank"` requires both `"daily"` and `"rank"` to appear). A layout is rejected if any of its `identification.negative_signals` is found by the same rule. First match wins.
6. **Boundary detection.** Locate `boundaries.header.signals` within `boundaries.header.search_region` (top boundary = bottom edge of matched line) and `boundaries.footer.signals` within `boundaries.footer.search_region` (bottom boundary = top edge of matched line). When `signals: []` (e.g. `season_contribution.yaml`'s footer), fall back to `chrome.bottom_fraction` / `chrome.top_fraction`, then to the image edge.
7. **Tab detection.** Within `tabs.search_region`, identify the active tab using either the top-level `tabs.active_indicator` or the per-group `tabs.groups[name]` config (when `tab_item.group` is non-empty on any item). For multi-row tabs, run detection independently per group and join winners with `_` in declaration order. The output category is the winning `tab_item.category` (falling back to `id` if `category` is empty).
8. **Column extraction.** Map every OCR token's `x_centre / image_width` to the column whose `[x_min, x_max]` contains it. `type: ignore` tokens are dropped; `type: name` tokens become name candidates; the rightmost numeric `type: score` token in each row is the score.
9. **Row clustering.** Per `row_clustering.strategy`. `score_anchored` (preferred): each numeric token in the score column ≥ `min_score` is an anchor; collect name tokens in `[anchor.top - up_band_fraction*H, anchor.bottom + down_band_fraction*H]`. `y_proximity`: group all tokens by vertical proximity then column-extract within each group.
10. **Crash-token recovery.** Detect tokens matching `[a-zA-Z].*\d{1,3}(?:,\d{3})+$` (alpha + comma-grouped trailing integer); generate every valid split where the name prefix contains no comma. See the algorithm spec in *Name/score crash tokens* below.
11. **Candidate disambiguation.** Pick one (name, score) per row using the priority order in *Name resolution* below. Emit the chosen pair (and optionally the candidates list — see *Output contract*).
12. **Output.** Emit one `(player_name, score, category)` triple per row, plus the active screen `id` and tab `category`.

### Game-window detection

The pipeline's stage 2 detects the rectangle inside the captured image where the game UI actually lives, and crops to it. This neutralises Pixel-Fold-style split-screen captures where the game is a portrait sub-window positioned at the left, centre, or right of a landscape canvas. Two strategies; consumers SHOULD implement both and prefer (a):

**(a) Black-border scan** *(preferred, runs pre-OCR).* Scan columns from the left edge of the image inward; the first column that is **not** a "border" is the left edge of the game window. Repeat from the right, top, and bottom edges. A column counts as a border when at least 95% of `SAMPLE_COUNT` (default 64) uniformly-spaced pixels in it have every channel ≤ 30 (near-black).

Reference values that work across all currently-shipped configurations:

| Constant | Value | Purpose |
|---|---|---|
| `NEAR_BLACK_MAX_CHANNEL` | 30 | A pixel is "near black" when every channel ≤ this. Wide enough to catch the slightly-grey Android letterbox; tight enough to reject UI chrome. |
| `BORDER_COVERAGE_THRESHOLD` | 0.95 | Fraction of sampled pixels in a column/row that must be near-black for it to count as a border. Allows a few stray noisy pixels. |
| `MIN_WINDOW_FRACTION` | 0.20 | Reject detected windows smaller than this fraction of either dimension — usually means the borders logic was confused (e.g. a near-black loading frame). |
| `SAMPLE_COUNT` | 64 | Pixels sampled per column/row. More = slower but more reliable. |

The function returns *no detection* (i.e. don't crop) when the detected window equals the full image (no letterbox to remove) **or** when it falls below `MIN_WINDOW_FRACTION`.

**(b) OCR-bbox union** *(post-OCR fallback).* When (a) returns no detection but the consumer has reason to suspect the image is still letterboxed (e.g. the system "Double-tap to move this app" panel in split-screen mode is dark grey, not black), re-run detection by taking the union of every text-block bounding box from the OCR pass and padding by `BBOX_PADDING_FRACTION` (0.03) of each dimension. Clamp to image bounds.

This strategy isn't free — it requires OCR to have already run on the un-cropped image. If a consumer relies on it as a fallback, the pre-OCR hint stage cannot be assumed correct (its sampled pixel may have landed in chrome, not the game). Production consumers should treat (b) as recovery from a (a) miss, not a primary strategy.

**When detection is unnecessary.** Captures where the game already fills the image — Pixel 10 Pro XL baseline, Pixel Fold front-screen, Pixel Fold inside-portrait — all return no-detection from (a) and pass through unchanged. Consumers can skip stage 2 entirely if they know their input source never letterboxes (e.g. a server-side import flow that only accepts edge-to-edge phone captures).

**Reference implementation.** `lastwar-ocr-service/app/utils/window_detect.py` is the canonical Python implementation; the Android scanner currently does not implement stage 2 because its capture source (`MediaProjection`) always produces edge-to-edge frames in portrait orientation. If that changes (e.g. the scanner adds Fold landscape support), it must mirror the same constants.

### Output contract

Consumers choose between two equivalent payload shapes — both are accepted by `lastwar-alliance-manager`:

**Shape A — server-resolved** (used by `lastwar-ocr-service`):

```json
{
  "category_key": [
    {"player_name": "ShodiWarmic", "score": 161528090},
    {
      "player_name": "Ruthless5432",
      "score": 1038000,
      "candidates": [
        {"player_name": "Ruthless5432", "score": 1038000},
        {"player_name": "Ruthless543",  "score": 21038000},
        {"player_name": "Ruthless54",   "score": 321038000}
      ]
    }
  ]
}
```

The consumer picks `candidates[0]` (rightmost split, smallest score) as the default and emits all valid splits for the backend to disambiguate against the full alias engine. `candidates` is omitted when the row had no crash-token ambiguity. **`category_key` is always the active tab's `category` field** (e.g. `"friday"`, `"weekly"`, `"power"`, `"donation_daily"`, `"siege_daily"`) — never the screen `id` itself, since every shipped screen has at least one tab.

**Shape B — client-resolved** (used by `lastwar-android-scanner`):

```json
{
  "name": "Ruthless5432",
  "score": 1038000,
  "category": "siege_daily"
}
```

The consumer has resolved candidates locally (using the *Name resolution* algorithm below against a cached roster) and ships only the winning `(name, score)`. The backend's name resolution becomes a final safety net rather than the primary disambiguator.

A consumer **must** pick a shape and stick with it for a given screen — mixing within one frame is a bug.

### Name resolution

The canonical algorithm both shapes implement. The backend's `resolveMemberAlias` (Go) and the scanner's `RosterAliasResolver` (Kotlin) MUST agree on this order. If they ever drift, this section is the tie-breaker.

For a candidate name, try in priority order, returning on first hit:

1. **Exact** — `LOWER(member.name) == LOWER(candidate)`.
2. **Personal alias** — `member_aliases` row where `LOWER(alias) = LOWER(candidate) AND user_id = $current_user`.
3. **Global alias** — `member_aliases` row where `LOWER(alias) = LOWER(candidate) AND user_id IS NULL AND category = 'global'`.
4. **OCR alias** — `member_aliases` row where `LOWER(alias) = LOWER(candidate) AND user_id IS NULL AND category = 'ocr'`.

When the row has multiple crash-token candidates: try each candidate through steps 1–4 in candidates-list order; the **first candidate** that resolves wins (its name AND its score travel together — the score differs per split). If none resolve, fall back to `candidates[0]`.

Personal aliases are user-scoped: a consumer that doesn't have the current user's identity (e.g. an offline batch tool) MUST skip step 2 — it MUST NOT see another user's personal aliases.

### Shared fallback constants

When a layout omits `hsv_override`, both the active-indicator and pre-OCR-hint stages fall back to these RGB checks. Both consumers MUST use these exact values to stay in lockstep. If a UI palette change makes them wrong, fix the values *here* and update both consumers in the same PR cycle.

| Colour | Test | HSV equivalent |
|---|---|---|
| Orange | `r > 200 AND 80 ≤ g ≤ 170 AND b < 90` | `h_min: 0.014, h_max: 0.153, s_min: 0.40, v_min: 0.55` |
| White | `r > 215 AND g > 215 AND b > 215` | `s_max: 0.10, v_min: 0.85` |

Default schema values (when a YAML omits the field) are authoritative in `meta-schema.json` — `min_score: 1000`, `word_gap_fraction: 0.015`, `min_word_gap_px: 8`, `up_band_fraction: 0.021`, `down_band_fraction: 0.002`, `tolerance_fraction: 0.02`, `min_tolerance_px: 20`, `min_fraction: 0.10`, `min_gap: 0.04`, `bbox_padding_fraction: 0.007`. Consumers should read defaults from the schema rather than hard-coding them.

### Versioning workflow

When changing the schema or adding a screen:

1. Bump the per-screen `version` in the YAML.
2. Bump `meta-schema.json`'s schema version comment if structurally changed.
3. Update both consumers' submodule SHA in the same merge cycle. Recommended order: `screen-definitions` → `lastwar-alliance-manager` (if backend contract changed) → `lastwar-ocr-service` → `lastwar-android-scanner`. Each downstream PR pins to the new submodule SHA.
4. Add a fixture screenshot to `lastwar-screenshots/` if the change affects parsing.

### Screen `id` is the backend category key

The `id` field in each screen YAML (e.g. `daily_ranking`, `alliance_contribution`) is the canonical category key the backend stores in `vs_points` / `power_history` / equivalent tables. Adding a new screen requires either matching an existing category or coordinating a new one with `lastwar-alliance-manager` in the same merge cycle.

For multi-tab screens, the wire-format category is the tab winner's `category` (e.g. `"friday"`, `"siege_daily"`), not the screen `id`. The backend uses the tab category to look up the column to upsert.

### Cross-pair classification audit

When adding a new screen, walk every existing screen and ask: *if the new screen's frame appears, can any of these other screens falsely match it via shared signals?* Add `negative_signals` until the answer is no. Conversely, ask: *can the new screen falsely match a frame from any of these other screens?* Add `negative_signals` to the new screen until the answer is no. Run the same audit in reverse for each existing screen against the new one.

The current state, with vulnerable cells marked **❌** (only the explicit `negative_signals` listed under the screen are considered — catalog priority alone is fragile because OCR noise can break the priority assumption):

| Checked ↓  /  Real screen → | Strength Power/Kills | Strength Donation | Daily VS | Weekly VS | Alliance Contribution |
|---|---|---|---|---|---|
| **strength_donation** (P1) | ✅ "Daily Weekly" combo not present | ✅ matches | ✅ day-name negs reject | ✅ "Weekly Rank" neg rejects | ✅ "Alliance Contribution" neg rejects |
| **strength_metrics** (P2) | ✅ matches | ✅ "Daily Weekly" combo neg rejects | ✅ day-name negs reject | ✅ "Weekly Rank" neg rejects | ✅ "Alliance Contribution" neg rejects |
| **weekly_ranking** (P3) | ✅ "Strength" neg rejects | ✅ "Strength" neg rejects (and "Weekly Rank" page_signal not present anyway) | ✅ day-name negs reject | ✅ matches | ✅ "Alliance Contribution" neg rejects |
| **daily_ranking** (P4) | ✅ "Daily Rank" not present (only "Daily" sub-tab) | ✅ "Daily Rank" not present | ✅ matches | ⚠ no `negative_signal` rejects, but `weekly_ranking` (P3) wins by priority | ✅ "Alliance Contribution" neg rejects |
| **alliance_contribution** (P5) | ✅ "Alliance Contribution" text not present | ✅ same | ✅ same | ✅ same | ✅ matches |

The single ⚠ cell (`daily_ranking` on a Weekly VS frame) is *covered by priority order, not by `negative_signals`*: weekly_ranking is checked first and matches cleanly on Weekly VS, so daily_ranking never gets evaluated. It would still match if weekly_ranking failed for some unrelated reason (e.g. OCR misread "Weekly Rank") — leaving the priority defence as a known soft spot worth recording rather than masking with a `negative_signal` that would also reject Daily VS (where "Weekly Rank" is also visible as the inactive period tab).

When you add a new screen, copy this table and add a row + column for it. Every cell in the new row and new column must end ✅ before merging.

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

Adapters MUST:
1. Use the **rightmost split** (smallest score) as the default heuristic — this becomes `candidates[0]`.
2. Validate against adjacent scores — the leaderboard is sorted descending, so a row's score must fall between its neighbours. If the default violates this bound, try alternatives in ascending score order until one fits.
3. After adjacency validation, run **Name resolution** (Consumer Contract section above) on each remaining candidate; the first to resolve wins (its name AND score travel together).
4. When ambiguity cannot be resolved (typically rank 1, where there's no upper neighbour), emit all valid splits via Output contract Shape A's `candidates` array, or pick `candidates[0]` and ship Shape B.

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

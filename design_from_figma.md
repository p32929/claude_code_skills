# /design_from_figma

You are a pixel-perfect design-to-code agent. Your job: make the running app look **identical** to the Figma design. You combine code auditing, asset downloading, and live screenshot comparison in a continuous loop until zero differences remain — and then you verify five more times.

---

## Core principles

**1. Both code AND screenshot — always, together**
Code audit catches exact values (color, font, spacing). Screenshots catch rendering reality (correct values that render wrong due to platform behavior, clipping, overflow). Run both every iteration.

**2. Scope follows the design, not the user's file list**
If the user names one file but the pixel-perfect result requires changing other files (shared components, navigators, theme/font registries, build configs), change those too. Tell the user every file you touched.

**3. Use the assets from Figma, not what already exists in the project**
Designers chose specific icons and images. The project's existing icon library is irrelevant — if the design shows a Vuesax/Material/custom icon and the project uses Lucide, you download the Figma asset and use it. Do NOT default to whatever icon library is already installed.

**4. Format/transform raw values for display**
Database values (snake_case, lowercase, IDs) are NOT what the design shows. The design shows the formatted human-readable form. Add format helpers (snake→Title Case, hex→rgba, etc.) at the display boundary.

**5. Cache lives at a unique path per design — do NOT wipe by default**
Every Figma fetch is cached at `/tmp/figma_cache/<fileKey>_<nodeId>/`. The cache persists across invocations so multiple designs can coexist. Only wipe when the user explicitly says "wipe / clear / fresh / re-fetch / re-download".

**6. Update these instructions when you learn something new**
When you hit a problem not covered here, add it to the architecture pitfalls section. Keep notes generic — never bake project-specific names (font names, color hexes, file paths from one project) into the skill. The skill must remain framework-agnostic and project-agnostic.

**7. Ask when ambiguous**
If a design label, field name, or behavior is unclear, ask before guessing.

**8. Termination is earned, not declared**
Pixel-perfect is not "looks good to me." It is a quantitative claim that survives **five consecutive verifications** after the diff table first goes clean — where each verification is backed by measurement-tool output, not visual judgment.

**9. You cannot trust your own eye on a downsampled screenshot**
The screenshots you Read are visually compressed. Font weights, 1-2px line thicknesses, and small position shifts are NOT reliably visible to you. The default failure mode is calling things ✓ because the screenshot "looks fine" at the resolution you can read it at. Defend against this with `magick`, `sips`, `compare`, OCR — anything that produces a numeric or pixel-level result. NEVER mark a row ✓ without a command output to back it up.

**10. Generate the diff image before claiming anything matches**
Before any iteration completes, run `magick compare -metric SSIM` between the live screenshot (resized to match) and the Figma reference, and Read the resulting diff image. The magenta highlights show pixel-level disagreements that your eye missed. Continue iterating until the diff image is clean inside the design's content area.

**11. Figma API data is the source of truth — NEVER substitute screenshot pixel sampling**
`get_design_context` already returns exact values: colors, sizes, radii, spacing, offsets, font names. Use these numbers DIRECTLY in code. Do NOT re-derive values you already have by sampling screenshots or measuring pixels from images.
- `rounded-[8px]` → `borderRadius: 8`
- `size-[31px]` → `width: 31, height: 31`
- `ml-[5.96px] mt-[5.96px]` → `marginLeft: 5.96, marginTop: 5.96` (or equivalent padding on the container)
- `gap-[14px]` → `gap: 14`
If the exact value is unclear (e.g., a clip-path mask with no numeric radius), THEN use `get_screenshot` on the specific node at native scale to measure from pixel data. This is the fallback, not the default.

---

## PHASE 0 — Understand intent

**Scope:** "fix only X" → start with X, but follow the design impact across other files. Figma URL alone → full implementation.

**Target:** Named or obvious from context: note it. Otherwise ask one question.

**Platform:** Detect from project (React Native, Flutter, Web, native iOS/Android). This determines how live screenshots are captured.

**Cache key:** Compute it now. From the Figma URL, extract `<fileKey>` and `<nodeId>`. The cache path for this run is:
```
CACHE=/tmp/figma_cache/<fileKey>_<nodeId>/
```
Sanitize by replacing `:` and `-` in the nodeId with `_` so the path is filesystem-safe.

State the plan in one line:
> "Full implementation — `<primary_file>` (+ any others as needed) — <platform> — cache at <CACHE>"

---

## PHASE 1 — Download everything once

```bash
CACHE=/tmp/figma_cache/<fileKey>_<sanitized_nodeId>
```

**If the cache exists and the user has NOT asked for a fresh fetch:**
- Verify required files exist: `design_context.txt`, `figma_reference.png`, `assets/` directory
- If complete, reuse it and skip to Phase 2
- If incomplete (missing files), top it up by fetching what's missing — do not wipe what's already there

**If the user explicitly asked to wipe / clear / re-download:**
```bash
rm -rf "$CACHE"
```

**Initialize:**
```bash
mkdir -p "$CACHE/assets"
```

### 1a — Design data (in parallel)
1. `get_design_context(fileKey, nodeId)` → save to `$CACHE/design_context.txt`
2. `get_screenshot(fileKey, nodeId)` → save the URL response, then `curl -s "<url>" -o "$CACHE/figma_reference.png"`

### 1b — If response is large/truncated, fetch sections
1. `get_metadata(fileKey, nodeId)` → save to `$CACHE/metadata.txt`, enumerate top-level children
2. Identify section nodes (Header, Tabs, Cards, Bottom Nav, Modals, etc.)
3. `get_design_context` on each section in parallel → save `$CACHE/section_<nodeId>.txt`
4. `get_screenshot` on each section → `$CACHE/section_<nodeId>.png`

### 1c — Identify ALL assets used in the design
Scan the design context for asset references (image-fill, vector, svg, icon nodes).

For each, get its node ID. Build an asset list and save it as `$CACHE/asset_list.md`.

### 1d — Download every asset
- **Icons: ALWAYS SVG.** Never PNG/JPG for icons. SVGs scale to any DPI and accept color overrides at runtime.
- **Photos / illustrations / raster art:** PNG at 3x for high-DPI.
- **Logos:** SVG if available, PNG 3x fallback.

```bash
curl -s "<svg_url>" -o "$CACHE/assets/icon_<name>.svg"
curl -s "<png_url>" -o "$CACHE/assets/photo_<name>.png"
```

For Figma MCP: parent wrapper nodes often return EMPTY SVGs — the actual path lives in a child Vector node. If a downloaded SVG has no `<path d="...">`, fetch the child Vector's URL instead.

Verify each downloaded file with `file <path>` — should report "SVG Scalable Vector Graphics image" (or PNG image data). If wrong format, the wrong URL was used; fetch the right one.

**The Figma API is now closed for this session.**

---

## PHASE 2 — Read all relevant files

Read every file that may need to change:
- The named target file
- Component files it imports/renders
- Navigators (if the design includes nav bars, tab bars, headers)
- Shared style/theme/constants files
- The font registry (where typefaces are declared) — needed for Phase 4 font verification
- The asset transformer config (e.g., metro.config.js for SVG, webpack for web)
- Any wrapper component (Typography, Button, Input) — read the source to understand prop→render mapping; never assume

Don't dismiss a file as irrelevant until you've checked what it controls.

---

## PHASE 3 — Extract design values

Save to `$CACHE/design_values.md`. Source of truth for everything that follows.

For every visible element, extract:
- Position (x, y from Figma) — use to calculate padding/margin/gap
- Size (width, height)
- Background color (exact hex/rgba — resolve any token references)
- Border (width per side, radius per corner, color)
- Shadow (x, y, blur, spread, color, opacity)
- Opacity
- Font: family, size, weight (numeric), lineHeight, letterSpacing, color, align
- Layout: flexDirection, alignItems, justifyContent, gap, padding, margin
- For icons: file path in `$CACHE/assets/`, size, color, strokeWidth
- For interactive elements (tabs, buttons, switches): extract BOTH/ALL states

Position math for spacing:
- Padding from container edge = element.x − container.x
- Gap between siblings = next.x − (prev.x + prev.width)
- Section gap = element.y − (previous.y + previous.height)

Resolve every design token to its concrete value. No `var(--x)` allowed.

**Special attention for vector lines:** If a vector node has `height=0` (or `width=0`), download its actual SVG and inspect — the rendered thickness comes from `stroke-width` in the SVG, not from the metadata. Common case: separator and active indicator both at `height=0` in metadata, but the separator's SVG has `stroke-width=1` and the indicator's has `stroke-width=2`.

---

## PHASE 4 — Implement

For each section:

**Read `design_values.md`. Use ONLY exact values from it.**

### Asset installation rules

**ALWAYS copy SVG/PNG files from `$CACHE/assets/` into the project's asset directory.** Reference from disk — NEVER inline SVG markup or path data into JSX components.

For SVG rendering, the project must support importing SVG as components. If it doesn't:
1. Install the appropriate transformer for the platform (e.g., `react-native-svg-transformer`)
2. Configure the bundler / build tool
3. Add type declarations for SVG modules

For Flutter / native iOS/Android / Web — follow the equivalent pattern (asset registration + render-from-file).

### Icon state intelligence — when to download multiple SVGs vs handle in code

Before downloading every variant, ask: does the state difference change the **shape**, or just **color/stroke/fill**?

**Same shape, different color/fill/stroke → handle in code (download ONE SVG):**
- Active vs inactive icon when only the color changes
- Pre-process the SVG: replace literal hex with `currentColor` so React props can override at runtime

**Different shape (truly different visual asset) → download BOTH variants:**
- Filled vs outlined icons where the path data differs
- Toggled icons where the variant adds/removes path segments

**Verification:** download one SVG and inspect. If `d="..."` is identical across states and only fill/stroke attributes differ, it's a code-toggle case. If `d="..."` differs, download both.

### Typography & font registration check

For every text element, verify the font family the design specifies is actually registered in the project. The Figma design will name fonts like `AirbnbCereal_W_XBd` or `SF Pro Display Bold`. Check:

1. The project's font registry (constants/theme file mapping logical names to font file names)
2. The native font registration (Info.plist on iOS, fonts directory on Android, @font-face on web)

If the Figma-specified font is **NOT registered**:
- Option A: register the font (add font file + native config) — preferred for true pixel parity
- Option B: pick the registered font with the closest weight + family — note that this is a fallback and the result will not be 100% identical

Do not silently assume an unregistered font name will work. If you don't register it, the text will fall back to system default and lose the intended weight.

When a wrapper text component (`Typography`, `Text`, etc.) maps its own preset names (`bold`, `regular`, `semiBold`) to project fonts, verify those mappings actually correspond to the right Figma weights — they are often inverted or named misleadingly. Read the wrapper's source.

### Other implementation rules
- Match project conventions (file structure, component patterns, state management)
- Read every wrapper component before using it — verify what each prop renders
- Add format helpers for raw data (snake_case → "Title Case", numeric IDs → labels) wherever the design shows formatted text but the API returns raw values
- Preserve existing logic, state, API calls, navigation handlers
- It's OK to add a build/transformer dependency if needed for asset loading — that's infrastructure, not application code
- Use **literal pixel values** from Figma (12, 16, 20, 24, etc.) — do NOT wrap them in `moderateScale` or other responsive helpers unless the design system explicitly defines them as relative

---

## PHASE 5 — The pixel-perfect loop

This is the hardest phase. The default failure mode is declaring "done" too early. Defend against this with structure, not optimism.

### Mindset

You are NOT looking for "is it close enough?" You are looking for "what is different?" — and there is ALWAYS something different until you've measured every numeric attribute. Each iteration produces a structured diff table that names every checked attribute. You do not stop iterating until that table is entirely ✓ AND you've passed five consecutive verification passes.

### Step A — Run the app yourself

Do NOT ask the user to run the app, navigate, or relaunch unless absolutely necessary (e.g., needing them to log in with credentials you don't have). For everything else, do it yourself with the platform's CLI.

**iOS:** `npx react-native run-ios --simulator="<device>"` or `xcodebuild` then `xcrun simctl launch`. Confirm the booted device with `xcrun simctl list devices | grep Booted`.

**Android:** `npx react-native run-android` after `adb devices` confirms a target.

**Web:** `npm run dev` (or equivalent) + Playwright to drive navigation.

**Flutter:** `flutter run -d <device-id>`.

Wait for the build/bundle to finish. Verify the app launched without crashes (read logs).

### Step B — Confirm the live screen is the target

Take a screenshot:
- iOS: `xcrun simctl io <UDID> screenshot $CACHE/live.png`
- Android: `adb -s <id> exec-out screencap -p > $CACHE/live.png`
- Web: Playwright `browser_take_screenshot`

Read the screenshot with the Read tool. Verify:
- It shows the target screen (header, content, nav match the kind of screen you're implementing)
- No build / Metro / packager error overlays covering the UI
- No loading spinners blocking visible content
- Hot-reload has applied your latest code (visible changes from your last edit are present)

If any of those fail, fix the cause and re-screenshot. Common causes:
- Metro disconnected → restart the bundler
- App crashed / red-box → read logs, fix, rebuild
- Stale cache → clear bundler cache (--reset-cache for Metro, equivalents elsewhere)
- App on wrong screen → drive navigation via the platform's automation

Do NOT proceed to diffing a stale or wrong screen.

### Step C — Build the structured diff table — WITH MEASUREMENTS, NOT EYE-BALLING

You cannot reliably tell pixel-level differences from a screenshot via the Read tool. The screenshot is downsampled; font weights, 1px line thicknesses, and small position deltas all look "close enough" to your eye even when wrong. **Stop trusting your eye. Use measurement commands.**

For every row in the diff table, the **Live** value must come from a CONCRETE measurement, not visual estimation. The table must show the **command** used and its **output**.

#### Mandatory measurement tools

Before starting the diff table, ensure these are available (install if missing):
- `magick` (ImageMagick 7) — pixel sampling, color extraction, image diff, SSIM
- `sips` (macOS built-in) — crop, resize
- `tesseract` (optional) — OCR for text-position extraction
- `compare` (ImageMagick) — generate visual diff images

#### Per-attribute measurement playbook

For each attribute being checked, use the corresponding command. Save the output. Quote it in the table.

**Color at a coordinate:**
```bash
magick "$LIVE" -format "%[pixel:p{<X>,<Y>}]" info:
```
Returns `srgb(R,G,B)` or `srgba(R,G,B,A)`. Compare to design hex (convert as needed). A ✓ requires the values match within ±2 per channel (tiny rendering tolerance).

**Background color of a region** (sample several points and confirm consistency):
```bash
for xy in "X1,Y1" "X2,Y2" "X3,Y3"; do magick "$LIVE" -format "%[pixel:p{$xy}]" info:; done
```

**Line / separator thickness** — count colored pixels in a vertical strip at the line's known x:
```bash
magick "$LIVE" -crop 1x100+<X>+<Y_START> -threshold 50% -format "%[fx:mean*100]" info:
```
Or even simpler — crop the strip and `magick identify` to see what colors are present. For a 1px line, you should see exactly 1 row of darker pixels in the strip.

**Element x position** — sample horizontally across a known y to find where colored content starts:
```bash
magick "$LIVE" -crop <W>x1+0+<Y> -depth 8 txt:- | head -200 | grep -v "white\|255,255,255" | head -5
```
Returns first non-white columns; tells you where the element's left edge actually is.

**Image dimensions / aspect:**
```bash
magick identify -format "%w x %h" "$LIVE"
```

**Visual diff — entire screen vs expected:**
After aligning live and reference (they may differ in size — resize the smaller to match), run:
```bash
magick compare -metric SSIM "$LIVE_RESIZED" "$REF" "$CACHE/diff.png"
```
SSIM returns 0 (different) to 1 (identical). Read the diff image — the highlighted areas show where pixels disagree. This is a SEEING tool: you can actually inspect WHICH parts are wrong, instead of guessing.

For fairer comparison, also try AE (absolute error count):
```bash
magick compare -metric AE -fuzz 5% "$LIVE_RESIZED" "$REF" /tmp/diff_ae.png 2>&1
```

**Font-weight visual check** (cannot be fully automated — bold detection is hard):
1. Crop the text region from both live and reference (`sips -c <h> <w> --cropOffset <y> <x>`)
2. Resize both to identical pixel dimensions
3. Run `magick compare -metric SSIM` — if SSIM < 0.95, weights/families differ
4. Read both crops with the Read tool side by side — if at zoomed-in resolution one looks visibly heavier than the other, the font isn't matching
5. Confirm by checking the project's font registry: is the design's font actually registered? (See Step F.)

**Bounding box check — element not just present but at the RIGHT position:**
For a known-color element (e.g., a black active indicator at (24, 141, 52, 2)), use:
```bash
magick "$LIVE" -fuzz 10% -fill none -opaque white -trim -format "%@" info:
```
This finds the bounding box of non-white content. Compare to expected.

#### Format of the diff table — every row carries evidence

```
DIFF TABLE — iteration N
═════════════════════════════════════════════════════════════════════
ELEMENT              ATTR             DESIGN     CODE       LIVE-MEASURED              CMD                                   STATUS
─────────────────────────────────────────────────────────────────────
Header               paddingTop       16         16         pixel-sample says greeting top y=<Y>; expected y=insets.top+16  magick ... -format ...   ✓
Greeting             color            #000000    #000000    p{50,80}=srgb(0,0,0)       magick "$LIVE" -format ... info:   ✓
Tab indicator        thickness        2px        height:2   strip at x=24..76: 2 black rows then white  magick ... -crop 53x4+24+141 ... ✓
Tab indicator        x position       24         left:24    first black col at y=141: x=<measured>      magick ... txt:- ...                ✓
Tab indicator        width            52         width:52   bbox of black at y=140..142: <w>            magick ... bbox                     ✓
"All" text           font heaviness   800/XBd    XBd ref'd  cropped text SSIM vs ref text: <score>      magick compare -metric SSIM         ✗ (SSIM 0.71 — too low)
...
```

The table must enumerate EVERY element from `design_values.md`. The `LIVE-MEASURED` column must contain a concrete value (a number, color tuple, or named characteristic from a tool), NEVER a vague claim like "looks right" or "✓".

If a row has no command output, the row is incomplete and you must measure it before moving on.

### Step D — Generate a visual diff image and inspect it

Don't try to spot-the-difference between two full-screen screenshots by eye. Compute the diff image and look at THAT.

**Align dimensions first.** Live screenshot and Figma reference will differ in pixel size. Resize live to match the reference's dimensions:
```bash
REF_W=$(magick identify -format "%w" "$REF")
REF_H=$(magick identify -format "%h" "$REF")
magick "$LIVE" -resize "${REF_W}x${REF_H}!" "$CACHE/live_aligned.png"
```

**Generate the diff image:**
```bash
magick compare -metric SSIM -highlight-color "magenta" -lowlight-color "white" \
  "$CACHE/live_aligned.png" "$REF" "$CACHE/diff.png" 2> "$CACHE/ssim.txt"
cat "$CACHE/ssim.txt"   # SSIM score (0=different, 1=identical)
```

**Read the diff image with the Read tool.** Magenta pixels are differences. White pixels match. The pattern of magenta tells you WHERE the design and live disagree. Examples:
- Magenta rectangle around a tab text → font / weight / size is off
- Magenta band across a line → thickness or position differs
- Magenta blob over an icon → wrong shape / size / color
- Magenta everywhere proportionally → screens are misaligned (wrong navigation, status bar overlap, etc.)

**Reading rule:** if the diff image has any magenta inside the design's content area (excluding mobile native chrome — status bar / home indicator on iOS), there are diffs to fix. Do NOT call it done while magenta exists.

**Close-up crops** as a follow-up: for each magenta region in the diff image, crop both the live and reference at that region and Read them at high zoom. This tells you the specific nature of the difference (e.g., "live text is lighter than reference text" → font weight issue).

### Step E — Anchored-coordinate check (deliberate-design rule)

When Figma metadata gives an element explicit `x`, `y`, `width`, `height`, those numbers are deliberate. If `Line A` has `x=24, width=52` and `Line B` has `x=24, width=351`, the designer chose those numbers. They are not artifacts and they MUST be reproduced literally:

- A `height=0` vector in Figma metadata is a hairline — but inspect the actual SVG for `stroke-width` (often >1)
- NEVER use `position: 'absolute'` for layout in React Native. Absolute positioning causes clipping, overflow, and spacing bugs that are impossible to predict. Always use flexbox (flexDirection, gap, padding, margin) to position elements in normal flow.
- An indicator/marker whose width differs from the natural item width is a fixed-width design element. Set explicit width; do not infer from `onLayout` measurements.

For every absolute-coordinate element, write:
```
ANCHORED-COORDINATE CHECK — <element name>
  Figma:   x=<x>, y=<y>, width=<w>, height=<h>
  Code:    <how it's positioned in code>
  Verdict: ✓ / ✗
  Fix:     <if ✗>
```

### Step F — Font weight verification check

For EVERY text element where the design specifies a non-default weight (anything that's not 400 / regular):

1. Note the Figma font family + numeric weight (e.g., `AirbnbCereal_W_XBd`, weight 800)
2. Open the project font registry. Is this font registered?
3. If yes — confirm code uses it correctly
4. If no — code is rendering the fallback (default system font at the requested weight). On iOS the fallback rarely produces the same visual weight; bold can render as regular. Either register the font or pick a registered close-weight equivalent and DOCUMENT this is a fallback.

For the live screenshot: zoom into the text. Compare its visual heaviness to the Figma reference text. If the Figma text is clearly **heavier** (denser strokes) and live is **lighter**, the font isn't rendering — flag this in the diff table.

### Step G — Fix every ✗

For each failed row:
1. Identify root cause: wrong value? wrong architecture? wrong asset? wrong field? unregistered font? missing format?
2. Apply the fix in the right file (which may be a different file than the named target — including native config files for fonts)
3. If a fix needs new architecture, do it — don't paper over with hacks

### Step H — Re-screenshot and re-table

Wait for hot reload (or rebuild if a native change was needed). Take a fresh screenshot. Verify it isn't stale.

Then **rebuild the diff table from scratch**. Do NOT carry forward last iteration's ✓s without re-checking — fixing one thing can break another.

### Step I — Termination criteria (5 consecutive clean verifications, with measurement evidence)

You may declare the loop complete only when ALL of the following are true:

1. The diff table has zero ✗ rows AND every row carries a measurement-tool output as evidence (no rows marked ✓ purely on visual judgment)
2. The full-image SSIM score (live_aligned vs reference, content area only) is ≥ 0.97
3. The diff image (Step D output) shows NO magenta inside the design's content area
4. **Five consecutive verification passes after first achieving the above** — for each pass:
   - Take a fresh screenshot (no code edits between passes)
   - Regenerate the diff image (`magick compare`) and verify SSIM ≥ 0.97 + clean diff
   - Re-run the per-attribute measurement commands — confirm outputs still match
   - Crop close-ups of at least 3 different zones; for each, run SSIM against the same crop of the reference and confirm ≥ 0.95
   - If any pass shows numeric ✗ OR magenta in the diff image, the streak resets to 0 and you go back to Step G
5. The five passes must include at least two screenshots taken with > 30 seconds between them
6. Final close-up SSIM check on every previously-failed zone — all ≥ 0.95

If all six pass, only then write the final report. Otherwise, iterate again.

**You may NOT skip steps to "save time".** The measurement output is the ONLY trustworthy signal that you've matched the design — your eye has already failed multiple times in this session and will fail again. Trust the tools.

### Anti-patterns that signal premature "done"

If you find yourself thinking any of these, you are wrong and must keep iterating:
- "It looks close to the design, that's good enough"
- "The remaining differences are minor"
- "This is a platform rendering quirk, can't be fixed"
- "The user will probably accept this"
- "I've done a lot of iterations already, time to wrap up"
- "I don't see anything obviously wrong"

The correct thoughts at the termination point are:
- "I have measured every value in design_values.md against the code and the screenshot"
- "Every measurement matches"
- "Five consecutive verification passes confirm this"
- "I cropped close-ups of every tight zone and they match too"
- "I verified every non-default font weight is actually registered, not just specified"

---

## Architecture pitfalls

Generic patterns to watch for. Update this list when you discover new ones — keep entries generic, never bake in project-specific names.

**Tab separator / active indicator:**
- `borderBottomWidth` on items leaves gaps where `marginRight` between items has no border
- `borderBottomWidth` on a container may not align with item borders (box model: border outside padding)
- Both separator and indicator should be dedicated absolutely-positioned `View` elements, NOT borders
- Look at the Figma metadata for the indicator. If its `x`/`width` differ from the natural tab item position/width, the indicator is anchored to fixed coordinates — DO NOT use `onLayout` measurement to position it. Use the literal coordinates from Figma. Only use `onLayout` when the indicator's coordinates match the tab item's natural geometry.
- A `height=0` vector in Figma metadata is a flat line — but inspect the SVG's `stroke-width` to know the actual thickness. Don't assume "1px hairline".
- Separator and indicator at the same `bottom: 0` must be sized so they visually align (often the same height, or the indicator centered on the separator y-line).

**Bottom navigation bar in framework navigators:**
- Often this is NOT a hand-coded View — it's a navigator's tabBarStyle / tabBarLabelStyle / tabBarActiveTintColor / tabBarInactiveTintColor
- Match: design height → tabBarStyle.height (with safe-area considerations for iPhone home indicator); design shadow → platform shadow props; design icon size → icon component's size prop; design label font → tabBarLabelStyle.fontFamily; design active color → tabBarActiveTintColor
- Remove default top border if the design shows none

**Typography wrappers:**
- Read the actual switch/mapping. Names like `regular`, `bold`, `medium` often DO NOT correspond to typical CSS weights. Always verify before using.
- **Critical pitfall — preset overrides user style:** Many wrapper components do `let s = { ...userStyle }; switch(textType) { case 'regular': s.fontFamily = X; }` — the switch runs AFTER spreading user style, so user's `style.fontFamily` and `style.fontWeight` are silently overwritten. Read the wrapper's source to confirm the order. If the preset overrides user style, you cannot use the wrapper for off-preset weights — bypass it with a raw `<Text>` instead.
- Verify the override took effect via measurement: crop the rendered text and compare its black-pixel density against a known-regular text in the same screenshot. A bold word should have ~3× the dark pixel density of a regular word at the same size. If density doesn't differ, the font override didn't apply.

**FlatList / ScrollView clipping:**
- `overflow: hidden` clips child borders and absolute elements
- Horizontal FlatList items can't overflow vertically beyond the FlatList bounds

**Absolute element stacking:**
- In RN, JSX render order determines stacking (later = on top), unless explicit `zIndex` overrides it
- Use deliberately: render a separator View before a FlatList to place it behind tab items

**Data field mapping:**
- Never guess API field → design label. Check the form/create screen for what's submitted and any values/options file for dropdowns
- If ambiguous, ASK

**Raw vs displayed values:**
- Raw DB values (snake_case, lowercase, numeric IDs) ≠ what the design shows
- Add format helpers: `snake_case_thing` → `Snake Case Thing`, status IDs → status labels, timestamps → human-readable dates
- Apply formatters at the display boundary, not in the data layer

**Icons from Figma vs project library:**
- The design's icon set (Vuesax, Phosphor, custom) is rarely the same as the library already installed
- Don't substitute the project's existing icons — download from Figma in Phase 1d, copy into project assets, render from file
- An icon that's the right size and color but the wrong shape is still wrong

**Unregistered fonts rendering as fallback:**
- Specifying `fontFamily: 'SomeFontName'` in code does NOT register that font. The font file must be in the project's asset/resource directory AND declared in the platform's font registry.
- If a Figma weight (e.g., 800 / ExtraBold) requires a specific font file the project doesn't bundle, the visual result will be the system fallback — usually noticeably lighter than the design.
- Always verify font registration during code audit, not just font references in styles.

**Responsive scaling helpers vs literal Figma values:**
- Project utility functions like `moderateScale`, `normalize`, `verticalScale` produce platform-dependent values that DRIFT from literal Figma pixels
- Use literal Figma values (12, 16, 20, etc.) directly — do not pass them through scaling helpers unless the design system explicitly says values should scale
- The design is designed at one density; reproducing it requires literal numbers

**SVG icon aspect ratio distortion:**
- A Figma SVG with a non-square viewBox (e.g., `viewBox="0 0 17.14 13.71"`) + `preserveAspectRatio="none"` rendered at equal width/height (e.g., 21×21) will be STRETCHED — the icon appears squished or elongated, noticeably ugly
- Rule: ALWAYS check the SVG viewBox W:H ratio before setting the rendered dimensions. If the ratio is not 1:1, either:
  - a) Render at natural proportional dimensions (e.g., width=21, height=17 for a ~5:4 viewBox), OR
  - b) Rewrite the SVG with a square viewBox that adds padding: viewBox=`"-pad_x -pad_y square_size square_size"` where pad_x/pad_y center the path inside
- The Figma "container" for icons is often a square with `overflow-clip` — the icon path itself has padding. Reproduce this by computing: pad_x = (container_size - path_w) / 2, pad_y = (container_size - path_h) / 2 and setting viewBox=`"-pad_x -pad_y container_size container_size"`
- Remove `preserveAspectRatio="none"` from any SVG you rewrite with a padded square viewBox

**SVG compound path fill rule:**
- Figma icons with multiple sub-paths (M...Z M...Z ...) in one `<path>` rely on winding rules to create holes (outlined shapes)
- React Native's SVG renderer may not handle the default nonzero winding correctly for all compound paths
- Always add `fillRule="evenodd"` on compound-path icons to ensure inner sub-paths create transparent holes instead of filled overlaps
- After downloading and saving a Figma SVG icon, ALWAYS render the app and Read the screenshot to visually verify the icon shape before declaring it correct — a compound path that "should work" can render as a solid black blob

**SVG icon visual verification — mandatory before proceeding:**
- After every new icon is added/replaced, take a screenshot, crop the icon region, and Read it
- Compare the cropped icon against the Figma reference icon (crop from `figma_reference.png`)
- Do NOT assume a path that "looks right in code" will render correctly — inspect the actual pixel output
- Icon verification commands:
  ```bash
  magick "$LIVE" -crop <W>x<H>+<X>+<Y> /tmp/live_icon_crop.png
  magick "$REF" -crop <W>x<H>+<X>+<Y> /tmp/ref_icon_crop.png
  magick compare -metric SSIM /tmp/live_icon_crop.png /tmp/ref_icon_crop.png /tmp/icon_diff.png 2>&1
  ```
  Read both crops. If SSIM < 0.85 or the shapes visually differ, the icon is wrong — diagnose and fix.

**Header background: transparent vs opaque at scroll=0:**
- When a screen has a fixed/floating header over a cover image, ALWAYS pixel-sample the header background area in the Figma reference (at ~y=header_top+5, at several x positions) to determine if it's transparent or opaque at scroll=0
- White pixels in the header area = opaque header (cover image is BELOW the header, not behind it)
- Photo/content pixels in the header area = transparent header (cover image bleeds through behind the header)
- Do NOT assume transparency — this is a common source of wrong cover-image height bugs
- If the header is opaque: the cover image must start BELOW the header. Use `paddingTop: header_height` on the ScrollView's `contentContainerStyle` to push cover below the fixed header
- If the header is transparent: use an animated opacity on the header background (0 at scroll=0, 1 at scroll threshold)

**Per-button styling — never assume all buttons share the same style:**
- In a header with multiple buttons (back, share, close, etc.), check EACH button individually in the Figma pixel data
- A button with a gray background box ≠ a button without one — they are different design choices, not a uniform pattern
- Sample pixels around each button's bounding box: if `background = rgba(86,86,86,0.2)` renders as #DDDDDD gray, surrounding pixels are #DDDDDD; if no background, surrounding pixels are white
- Apply the correct style per button, not the same style to all

**Multiple simulators — always use UDID:**
- If more than one simulator is running (`xcrun simctl list devices | grep Booted` shows multiple), `xcrun simctl io booted screenshot` picks an arbitrary device
- Always identify the TARGET simulator's UDID and use it explicitly: `xcrun simctl io <UDID> screenshot`
- Confirm the screenshot is from the right device by checking screen dimensions and content

---

## PHASE 6 — Final report

Only after Step I's five consecutive clean passes:

```
Done. Pixel-perfect after N iterations + 5 consecutive verification passes.
─────────────────────────────────────────────────────────────────────
Cache:         <CACHE>
Files changed: <list>
Assets added:  <list>
Code audit:    CLEAN
Visual audit:  CLEAN  (5 consecutive passes)
Iterations:    N
```

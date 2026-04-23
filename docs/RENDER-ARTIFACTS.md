# RENDER-ARTIFACTS

Chrome headless + PIL crop recipe for the PDF + PNG half of every component's
artifact quartet. The orchestrator runs this after a subagent emits README.md
and {name}.html. Subagents never run Chrome.

## REQUIREMENTS

- `/Applications/Google Chrome.app/Contents/MacOS/Google Chrome` on macOS.
  If missing, exit with an install prompt — no fallback.
- Python 3 with PIL (`Pillow`) available via `python3`.

## RENDER A PDF

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new --disable-gpu --hide-scrollbars \
  --virtual-time-budget=5000 \
  --no-pdf-header-footer \
  --print-to-pdf="<path>/{name}.pdf" \
  "file://<absolute-path>/{name}.html"
```

**Flags**:
- `--headless=new` — modern headless mode (better CSS support than old headless)
- `--virtual-time-budget=5000` — wait 5s of virtual time for fonts + animations to settle
- `--no-pdf-header-footer` — strip the default page header/footer from the PDF
- `--print-to-pdf=<path>` — output file
- `--disable-gpu` — stability on headless
- `--hide-scrollbars` — don't include scrollbar in the print

**Positional arg**: absolute `file://` URL to the HTML.

## RENDER A PNG (full-page snapshot)

Two-step: shoot at max height, then trim with PIL to the actual content.

### Step 1 — snapshot

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new --disable-gpu --hide-scrollbars \
  --virtual-time-budget=5000 \
  --window-size=1320,3200 \
  --screenshot="<path>/{name}.png" \
  "file://<absolute-path>/{name}.html"
```

Width `1320` matches the `.page` `max-width` used by `_preview.css`.
Height `3200` is generous — the screenshot includes trailing canvas that we
crop in step 2. For very tall previews (skeleton, button), bump to 6000–9000.

### Step 2 — PIL trim to content

```python
from PIL import Image
import os

def trim_to_content(png_path, canvas_rgb=(247, 246, 243), tol=3, pad=40):
    """Trim trailing canvas so the PNG ends where content ends."""
    im = Image.open(png_path)
    w, h = im.size
    px = im.load()
    bottom = h - 1
    for y in range(h - 1, -1, -1):
        xs = [40, w // 4, w // 2, 3 * w // 4, w - 40]
        if any(
            abs(px[x, y][0] - canvas_rgb[0]) > tol or
            abs(px[x, y][1] - canvas_rgb[1]) > tol or
            abs(px[x, y][2] - canvas_rgb[2]) > tol
            for x in xs
        ):
            bottom = y
            break
    crop_h = min(bottom + pad, h)
    im.crop((0, 0, w, crop_h)).save(png_path, optimize=True)
```

**Canvas RGB** defaults to `#f7f6f3` (oyster — the light-theme surface-canvas
value). If the target design uses a different canvas color, pass it via
`canvas_rgb=`. Read the value from `tokens/theme.light.json` at runtime:
`canvas = hex_to_rgb(theme_light['surface']['canvas'])`.

## BATCH RENDER (orchestrator helper)

```bash
# After a phase completes, render every component in that phase.
LAYER=atoms   # or molecules | organisms | views
for d in "<output>/$LAYER"/*/; do
  name="$(basename "$d")"
  [ "$name" = "_preview.css" ] && continue
  html="$d$name.html"
  [ -f "$html" ] || continue

  # PDF
  "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
    --headless=new --disable-gpu --hide-scrollbars \
    --virtual-time-budget=5000 --no-pdf-header-footer \
    --print-to-pdf="$d$name.pdf" \
    "file://$(realpath "$html")"

  # PNG
  "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
    --headless=new --disable-gpu --hide-scrollbars \
    --virtual-time-budget=5000 --window-size=1320,3200 \
    --screenshot="$d$name.png" \
    "file://$(realpath "$html")"
done

# Then trim PNGs with the Python snippet above.
```

## PER-CATEGORY PRIMITIVES PNG (Phase 1 only)

Phase 1 produces `tokens.html` with identified sections per category
(typography, surface, text, border, accent, status, elevation, spacing,
radius, motion, layout). After rendering the full `tokens.html`, crop each
section into its own PNG at `<output>/primitives/{category}.png` for LLM
reference.

Measure each section's bounding box via headless Chrome + a wrapper HTML that
iframes `tokens.html` and reports `getBoundingClientRect()` JSON into a
`<pre id="out">` node. Read the JSON via `--dump-dom` + regex extract.

See the reference implementation in
`~/Projects/experience-surface/.spec/design/system/primitives/README.md` — the
measurement script is portable.

## VIEW RENDERING (Phase 5 only)

Views render at two viewport sizes to capture responsive layout.

### Desktop render

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new --disable-gpu --hide-scrollbars \
  --virtual-time-budget=5000 \
  --window-size=1320,6400 \
  --screenshot="<path>/{name}.png" \
  "file://<absolute-path>/{name}.html"
```

Produces `{name}.png` — same as the standard render but with taller window
(6400px) because full-page views are taller than component previews.

### Mobile render

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new --disable-gpu --hide-scrollbars \
  --virtual-time-budget=5000 \
  --window-size=375,6400 \
  --screenshot="<path>/{name}-mobile.png" \
  "file://<absolute-path>/{name}.html"
```

Produces `{name}-mobile.png` at 375px width (iPhone viewport). The view HTML
must include `@media (max-width: 375px)` rules for this to show the mobile
layout — if it doesn't, the PNG will show the desktop layout squeezed, which
is still useful for identifying the responsive gap.

### Batch for views

```bash
for d in "<output>/views"/*/; do
  name="$(basename "$d")"
  html="$d$name.html"
  [ -f "$html" ] || continue

  # Desktop PNG
  "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
    --headless=new --disable-gpu --hide-scrollbars \
    --virtual-time-budget=5000 --window-size=1320,6400 \
    --screenshot="$d$name.png" \
    "file://$(realpath "$html")"

  # Mobile PNG
  "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
    --headless=new --disable-gpu --hide-scrollbars \
    --virtual-time-budget=5000 --window-size=375,6400 \
    --screenshot="$d$name-mobile.png" \
    "file://$(realpath "$html")"
done

# Then trim both desktop + mobile PNGs with PIL trim_to_content.
```

Non-view components (atoms, molecules, organisms) use the single desktop
render defined in the BATCH RENDER section above.

## FAILURES

- **Chrome crashes with GPU errors** — keep `--disable-gpu`; the errors are
  harmless on macOS headless.
- **Empty PDF** — fonts haven't loaded; bump `--virtual-time-budget` to `8000`.
- **Wrong height** — the page is longer than the window-size; bump to 6000 or
  9000 and re-run trim.
- **Black PNG** — dark-theme pane at the end; canvas_rgb match failed. Pass
  the explicit canvas color or switch the detection to light-theme pane only.

## REPRODUCIBILITY

The render is deterministic modulo:
- font substitution if `@font-face` fonts fail to download (add `--offline`
  Chrome won't try, but macOS Helvetica fallback kicks in — check the PDF for
  rendering weirdness).
- the `virtual-time-budget` clock — animations pause at that point; a longer
  budget catches more animation frames but doesn't change the final layout.

For identical byte-for-byte output across runs, keep `--virtual-time-budget`
constant and avoid ambient timestamp injection (no `new Date()` in previews).

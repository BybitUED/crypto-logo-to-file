---
name: crypto-logo-to-file
description: |
  Finds an official crypto token logo from a CMC or CoinGecko URL and saves 4 files to ~/Downloads/:
  {SYMBOL}-light.svg, {SYMBOL}-dark.svg, {SYMBOL}-light.png, {SYMBOL}-dark.png (192×192px each).

  Use this skill whenever the user:
  - Provides a CoinMarketCap or CoinGecko URL and wants the logo saved locally
  - Provides a contract address (0x...) and wants a token logo as file
  - Says phrases like "新币LOGO", "token logo", "帮我存一下这个币的logo", "coin icon"
  - Wants Dark + Light versions of a crypto logo as SVG or PNG

  The skill handles the full pipeline: logo fetch → background detect → save 4 files (SVG + PNG × Dark/Light).
---

# Crypto Logo → File

Given a CMC or CoinGecko URL (plus an optional symbol), fetch the official logo and save **4 files** to `~/Downloads/`:

```
{SYMBOL}-light.svg   {SYMBOL}-dark.svg
{SYMBOL}-light.png   {SYMBOL}-dark.png
```

All files are 192×192px. The symbol is always **UPPERCASE**. If the user doesn't provide a symbol, parse the token's ticker from the source page/API and use it.

---

## Inputs

- **CMC or CoinGecko URL** — e.g. `https://coinmarketcap.com/currencies/bitcoin/`
- **Symbol/code** (optional) — e.g. `BTC`. If omitted, parse from the source. Always UPPERCASE.

Do **not** verify the token name or look up a contract address unless the user provides one instead of a URL.

---

## Step 1 — Get the logo

### Source: CoinMarketCap URL

Use Jina Reader to scrape the CMC page (single call, cache the output):

```bash
PAGE=$(curl -s "https://r.jina.ai/{cmcUrl}")
```

Find the CMC image ID. CMC uses different paths for coins vs RWA (stocks):

```bash
# Match both /coins/ and /rwa/ image paths
echo "$PAGE" | grep -o 's2\.coinmarketcap\.com/static/img/[a-z]*/64x64/[0-9a-f]*\.png' | head -1
```

The ID is the part after `64x64/` (numeric for coins, hex for RWA). For RWA pages, verify the match is the token's own logo by checking the alt text (e.g., `![Image N: TSLA]`), not a sidebar coin.

Parse the symbol from the cached page if not provided by the user:

```bash
echo "$PAGE" | head -3
# Title line typically contains: "TokenName (SYMBOL) ..."
```

Download the PNG using Python (curl exits with code 56 on CMC CDN). Use the image type (`coins` or `rwa`) from the grep match:

```python
import urllib.request, os

img_type = '{TYPE}'  # 'coins' or 'rwa' — from the grep match path
cmc_id   = '{ID}'    # numeric (coins) or hex (rwa)
symbol   = '{SYMBOL}'

url = f'https://s2.coinmarketcap.com/static/img/{img_type}/200x200/{cmc_id}.png'
req = urllib.request.Request(url, headers={'User-Agent': 'Mozilla/5.0', 'Referer': 'https://coinmarketcap.com/'})
with urllib.request.urlopen(req) as r:
    data = r.read()

src_path = os.path.expanduser(f'~/{symbol.lower()}_source.png')
with open(src_path, 'wb') as f: f.write(data)

from PIL import Image
from io import BytesIO
img = Image.open(BytesIO(data))
print(f'Downloaded {len(data)} bytes | {img.size[0]}x{img.size[1]}')
```

> **Tip**: CMC source is only 200px. Cross-reference on CoinGecko using the same slug and fetch via `/original/` path for much higher resolution.

### Source: CoinGecko URL

Extract the coin ID from the URL (e.g. `.../coins/bitcoin` -> `bitcoin`).

**Step A — Try the API first** (may fail for very new tokens):

```bash
curl -s "https://api.coingecko.com/api/v3/coins/{coinGeckoId}" | python3 -c "
import json, sys
d = json.load(sys.stdin)
print('Symbol:', d['symbol'].upper())
print('Image:', d['image']['large'])
"
```

**Step B — If API returns `coin not found`**, fall back to Jina Reader to scrape the page:

```bash
curl -s "https://r.jina.ai/https://www.coingecko.com/en/coins/{coinGeckoId}" | grep -o 'https://assets\.coingecko\.com/coins/images/[^)]*' | sort -u
```

Find the token's own logo URL (not BTC/ETH/other coins) and parse the symbol from the page title.

**Step C — Download the highest resolution source**:

CoinGecko image URLs contain a size segment: `/thumb/`, `/small/`, `/standard/`, `/large/`, `/original/`. **Always replace the size segment with `/original/`** to get the full-resolution source (often 2000px+). This is critical — `/large/` is only 250px which produces blurry output at 192px.

```python
import urllib.request, os, re

image_url = '{imageUrl}'  # from API or Jina scrape
# Force /original/ for max resolution
image_url = re.sub(r'/(thumb|small|standard|large)/', '/original/', image_url)

symbol = '{SYMBOL}'  # UPPERCASE
req = urllib.request.Request(image_url, headers={'User-Agent': 'Mozilla/5.0'})
with urllib.request.urlopen(req) as r:
    data = r.read()

src_path = os.path.expanduser(f'~/{symbol.lower()}_source.png')
with open(src_path, 'wb') as f: f.write(data)

from PIL import Image
from io import BytesIO
img = Image.open(BytesIO(data))
print(f'Downloaded {len(data)} bytes | {img.size[0]}x{img.size[1]}')
```

### Source: Native SVG (official brand site)

If a native SVG is available (official docs, brand kit), download and verify:

```python
import urllib.request, os

url    = '{svgUrl}'
symbol = '{SYMBOL}'
req = urllib.request.Request(url, headers={'User-Agent': 'Mozilla/5.0', 'Referer': '{referer}'})
with urllib.request.urlopen(req) as r:
    data = r.read()

if b'<svg' not in data[:500]:
    print('ERROR: Not a valid SVG')
else:
    src_path = os.path.expanduser(f'~/{symbol.lower()}_source.svg')
    with open(src_path, 'wb') as f: f.write(data)
    print(f'Valid SVG saved -> {src_path}')
```

If native SVG obtained: **go to Step 3b**.
If only PNG available: continue to Step 2.

---

## Step 2 — Background detection (PNG source only)

**Dual-radius sampling**: sample both the interior (85%) and the edge (97%) of the logo. Many logos have a built-in contrasting border (e.g., gold ring on black background). Single-radius sampling at the interior misdetects these as "black" and adds an unwanted stroke. By comparing inner vs edge luminance, we detect self-bordered logos and skip the stroke.

```python
from PIL import Image
import numpy as np, math, os

symbol   = '{SYMBOL}'
src_path = os.path.expanduser(f'~/{symbol.lower()}_source.png')

img = Image.open(src_path).convert('RGBA')
arr = np.array(img)
h, w = arr.shape[:2]
cx, cy = w // 2, h // 2

def sample_ring(radius_pct):
    r = int(w * radius_pct / 2)
    pts = [(cy + int(r * math.sin(a)), cx + int(r * math.cos(a)))
           for a in [i * math.pi / 4 for i in range(8)]]
    samples = [arr[y, x] for y, x in pts if 0 <= y < h and 0 <= x < w]
    a_avg = np.mean([s[3] for s in samples])
    r_avg = np.mean([s[0] for s in samples])
    g_avg = np.mean([s[1] for s in samples])
    b_avg = np.mean([s[2] for s in samples])
    lum = 0.299 * r_avg + 0.587 * g_avg + 0.114 * b_avg
    return {'r': r_avg, 'g': g_avg, 'b': b_avg, 'a': a_avg, 'lum': lum}

inner = sample_ring(0.85)  # logo interior
edge  = sample_ring(0.97)  # logo edge (just inside the circle boundary)

# Decision logic:
# 1. Edge transparent → no stroke
# 2. Edge lum differs from inner by >40 → logo has its own border → no stroke
# 3. Edge lum < 30 → truly dark edge → dark mode needs white stroke
# 4. Edge lum > 220 → truly light edge → light mode needs black stroke
# 5. Otherwise → mid-tone edge → no stroke
if edge['a'] < 30:
    bg = 'transparent'
elif abs(edge['lum'] - inner['lum']) > 40:
    bg = 'has_border'
elif edge['lum'] < 30:
    bg = 'black'
elif edge['lum'] > 220:
    bg = 'white'
else:
    bg = 'other'

print(f'Inner lum={inner["lum"]:.0f} | Edge lum={edge["lum"]:.0f} | Diff={abs(edge["lum"]-inner["lum"]):.0f} | bg={bg}')
```

---

## Step 3a — Save 4 files (PNG source)

**Border rules** — stroke baked into both SVG and PNG:

| Classification | Light stroke | Dark stroke |
|---------------|-------------|------------|
| white         | `#000000` 4px | none |
| black         | none | `#ffffff` 4px |
| has_border / transparent / other | none | none |

Run this single script to produce all 4 output files.

**CRITICAL — Anti-aliased edges via 4× supersampling**: PIL's `ellipse` has no anti-aliasing. Drawing the circular mask and stroke at 192px produces jagged/blurry edges. The fix: render everything at 4× (768px), then downscale with LANCZOS. This gives pixel-perfect smooth circle edges.

```python
import base64, os
from PIL import Image, ImageDraw

symbol   = '{SYMBOL}'   # UPPERCASE
src_path = os.path.expanduser(f'~/{symbol.lower()}_source.png')
out_dir  = os.path.expanduser('~/Downloads')
# bg = 'white' / 'black' / 'transparent' / 'other'  (from Step 2)

light_stroke = '#000000' if bg == 'white' else None
dark_stroke  = '#ffffff' if bg == 'black' else None

SIZE  = 192
SCALE = 4
BIG   = SIZE * SCALE  # 768 — supersample resolution

# Load source
with open(src_path, 'rb') as f:
    raw_bytes = f.read()
b64     = base64.b64encode(raw_bytes).decode()
src_img = Image.open(src_path).convert('RGBA')

# SVG builder — PNG embedded as base64, circular clip + optional stroke baked in
# Uses xlink:href (SVG 1.1) — required by Figma's SVG parser
def make_svg(stroke_color=None):
    # Stroke at r="96" sw="8" clipped by same clipPath — SVG stroke is centered,
    # outer 4px clipped away, inner 4px visible. Matches PNG flush-edge behavior.
    stroke_el = (
        f'<circle cx="96" cy="96" r="96" fill="none" stroke="{stroke_color}" stroke-width="8" clip-path="url(#c)"/>'
        if stroke_color else ''
    )
    return f'''<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" width="192" height="192" viewBox="0 0 192 192">
  <defs>
    <clipPath id="c">
      <circle cx="96" cy="96" r="96"/>
    </clipPath>
  </defs>
  <image width="192" height="192" xlink:href="data:image/png;base64,{b64}" clip-path="url(#c)" preserveAspectRatio="xMidYMid meet"/>
  {stroke_el}
</svg>'''

# PNG builder — 4× supersampled circular clip + stroke for crisp anti-aliased edges
def make_png(stroke_color_hex=None):
    big_img = src_img.resize((BIG, BIG), Image.LANCZOS)
    out  = Image.new('RGBA', (BIG, BIG), (0, 0, 0, 0))
    mask = Image.new('L', (BIG, BIG), 0)
    ImageDraw.Draw(mask).ellipse([0, 0, BIG - 1, BIG - 1], fill=255)
    out.paste(big_img, (0, 0), mask)
    if stroke_color_hex:
        rv = int(stroke_color_hex[1:3], 16)
        gv = int(stroke_color_hex[3:5], 16)
        bv = int(stroke_color_hex[5:7], 16)
        sw = 4 * SCALE  # stroke width scaled up
        # Flush with mask edge — PIL draws stroke inward from bbox,
        # so [0,0,BIG-1,BIG-1] puts the stroke outer edge at the circle boundary.
        # Using an offset creates a dark gap between stroke and mask edge.
        ImageDraw.Draw(out).ellipse(
            [0, 0, BIG - 1, BIG - 1],
            outline=(rv, gv, bv, 255), width=sw
        )
    return out.resize((SIZE, SIZE), Image.LANCZOS)

# Save all 4 files
for mode, stroke in [('light', light_stroke), ('dark', dark_stroke)]:
    base = f'{symbol}-{mode}'
    with open(os.path.join(out_dir, f'{base}.svg'), 'w') as f:
        f.write(make_svg(stroke))
    make_png(stroke).save(os.path.join(out_dir, f'{base}.png'))
    print(f'{mode}: stroke={stroke or "none"} -> {base}.svg + {base}.png')

# Cleanup source file
os.remove(src_path)
```

---

## Step 3b — Save 4 files (native SVG source)

**Background detection for SVG**: inspect the SVG source for a background `<rect fill>` or `<circle fill>`. No background shape found -> assume `transparent`.

```python
import re, os

symbol   = '{SYMBOL}'
src_path = os.path.expanduser(f'~/{symbol.lower()}_source.svg')

with open(src_path, 'r', errors='ignore') as f:
    svg_text = f.read()

bg_match = re.search(r'<(?:rect|circle)[^>]+fill=["\']([^"\']+)["\']', svg_text)
if bg_match:
    fill = bg_match.group(1).lower().strip()
    if fill in ('#fff', '#ffffff', 'white'):   bg = 'white'
    elif fill in ('#000', '#000000', 'black'): bg = 'black'
    else:                                       bg = 'other'
else:
    bg = 'transparent'

print(f'SVG background: {bg}')

# Ensure explicit width/height — required for Figma import
if 'width=' not in svg_text:
    svg_text = re.sub(r'(<svg\b)', r'\1 width="192" height="192"', svg_text, count=1)
else:
    svg_text = re.sub(r'width="[^"]+"',  'width="192"',  svg_text, count=1)
    svg_text = re.sub(r'height="[^"]+"', 'height="192"', svg_text, count=1)
```

**Add stroke and save SVG files**:

```python
out_dir      = os.path.expanduser('~/Downloads')
light_stroke = '#000000' if bg == 'white' else None
dark_stroke  = '#ffffff' if bg == 'black' else None

def patch_svg(text, stroke_color=None):
    stroke_el = (
        f'\n  <defs><clipPath id="sc"><circle cx="96" cy="96" r="96"/></clipPath></defs>\n  <circle cx="96" cy="96" r="96" fill="none" stroke="{stroke_color}" stroke-width="8" clip-path="url(#sc)"/>'
        if stroke_color else ''
    )
    return text.replace('</svg>', f'{stroke_el}\n</svg>')

for mode, stroke in [('light', light_stroke), ('dark', dark_stroke)]:
    base = f'{symbol}-{mode}'
    with open(os.path.join(out_dir, f'{base}.svg'), 'w') as f:
        f.write(patch_svg(svg_text, stroke))
    print(f'{mode}: stroke={stroke or "none"} -> {base}.svg')
```

**PNG export for native SVG** — requires `cairosvg`. Render at 4× then downscale for crisp edges:

```python
try:
    import cairosvg
    from PIL import Image
    from io import BytesIO

    SIZE  = 192
    SCALE = 4
    BIG   = SIZE * SCALE

    for mode in ['light', 'dark']:
        base     = f'{symbol}-{mode}'
        svg_path = os.path.join(out_dir, f'{base}.svg')
        png_path = os.path.join(out_dir, f'{base}.png')
        png_data = cairosvg.svg2png(url=svg_path, output_width=BIG, output_height=BIG)
        img = Image.open(BytesIO(png_data)).resize((SIZE, SIZE), Image.LANCZOS)
        img.save(png_path)
        print(f'{mode}: PNG -> {base}.png')
except ImportError:
    print('cairosvg not installed — PNG skipped. Install with: pip3 install cairosvg')
```

---

## Step 4 — Report result

```
✅ {SYMBOL} logo saved to ~/Downloads/

  {SYMBOL}-light.svg   stroke=#000000
  {SYMBOL}-dark.svg    stroke=none
  {SYMBOL}-light.png   192x192px
  {SYMBOL}-dark.png    192x192px

Source: CMC PNG / CoinGecko PNG / Native SVG
Background detected: white / black / transparent / other
```

---

## Tips & gotchas

**Source resolution is king**: Output quality at 192px depends entirely on source resolution. Always get the highest res available:
- CoinGecko: replace `/standard/` or `/large/` with **`/original/`** in image URL (often 2000px+)
- CMC: max is 200px — cross-reference on CoinGecko `/original/` for better quality
- A 2000px source downscaled to 192px is dramatically sharper than 200px→192px

**curl exits with code 56**: Use Python `urllib.request` for all logo downloads. Save to `~/`, not `/tmp/`.

**Jina Reader**: Use `https://r.jina.ai/{url}` to scrape JS-heavy pages (CMC, CoinGecko, Bybit, project sites).

**CoinGecko API may fail for new tokens**: If API returns `coin not found`, fall back to Jina Reader scrape.

**CMC coin ID & RWA**: The coin ID is in the image URL on the CMC page. CMC uses `/coins/` (numeric ID) for crypto and `/rwa/` (hex ID) for stocks/real-world assets. The Step 1 grep handles both paths automatically. For RWA pages, verify the match is the token's own logo via alt text — sidebar images are unrelated coins.

**Symbol fallback**: If the user provides no symbol, parse the ticker from the page. Always store as **UPPERCASE**.

**SVG width/height missing**: Figma requires explicit `width` and `height` on `<svg>`. If only `viewBox` is present, add them manually.

**xlink:href**: Use `xlink:href` (not `href`) on `<image>` elements — Figma's SVG parser requires it.

**Non-ETH chains**: Adjust TrustWallet path (`blockchains/smartchain/` for BSC, etc.) if falling back to TrustWallet assets.

---

## Runtime requirements

- **Bash** — curl + Jina scraping
- **Python 3** with `Pillow` and `numpy` — background detection + PNG output
- **`cairosvg`** (optional) — only needed for PNG export when source is a native SVG (`pip3 install cairosvg`)

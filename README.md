# crypto-logo-to-file

A Claude Code skill that fetches official crypto token logos from CoinMarketCap or CoinGecko and saves production-ready files to `~/Downloads/`.

Given a URL, it outputs 4 files: `{SYMBOL}-light.svg`, `{SYMBOL}-dark.svg`, `{SYMBOL}-light.png`, `{SYMBOL}-dark.png` (192x192px each).

## Features

- **Auto source resolution**: Fetches the highest available resolution (CoinGecko `/original/` up to 2000px+, CMC 200px). Supports both `/coins/` and `/rwa/` (stocks/real-world assets) paths.
- **Smart background detection**: Dual-radius sampling compares logo interior vs edge luminance. Detects self-bordered logos (e.g., gold ring on black bg) and skips unnecessary strokes.
- **Anti-aliased edges**: 4x supersampled circular clipping and stroke rendering via PIL, then LANCZOS downscale for pixel-perfect smooth circles.
- **Light/Dark mode**: Adds a subtle border stroke only when the logo edge would blend into the target background (black edge on dark bg, white edge on light bg).

## Requirements

- Python 3 with `Pillow` and `numpy`
- `cairosvg` (optional, for native SVG to PNG conversion)
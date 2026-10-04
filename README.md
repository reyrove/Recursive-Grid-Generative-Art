# Recursive-Grid-Generative-Art

A seed-based generative art catalogue for **Recursive Grid**, a computational recursive-subdivision system. Built as a single-file HTML page with p5.js, styled as a gallery-ready portfolio and licensing surface.

**Live:** `https://reyrove.github.io/Recursive-Grid-Generative-Art/`

---

## Overview

**Recursive Grid** renders a recursively subdivided canvas from numeric seeds. Each composition starts with the full canvas, then splits it into 2–4 × 2–4 rectangles. Each cell either draws itself as a flat coloured rectangle or recurses further — three levels deep. Positions are jittered by up to 10%, sizes vary by 0.8–1.1, and every rectangle is drawn in a fresh saturated HSB colour at 80% alpha on a pure black ground.

The page functions as a **catalogue and licensing surface** for fashion houses, textile studios, and surface designers. Every composition is reproducible by seed, meaning the same number always produces the identical field.

---

## Features

### Generative Engine (p5.js)

- **Recursive subdivision** — three levels deep
- **Per-level split** — each rectangle divides into 2–4 columns and 2–4 rows
- **70/30 split rule** — 70% of cells recurse further, 30% draw immediately
- **Base case** — a cell stops recursing if depth is exhausted or if it becomes smaller than 5% of the canvas
- **Jitter** — every cell's position is offset by up to 10% of its width or height
- **Size variance** — every cell's size is scaled by 0.8–1.1
- **Flat colour** — every rectangle picks a fresh random HSB colour at 80% alpha:
  - Hue: 0 – 360 (full spectrum)
  - Saturation: 50 – 100
  - Brightness: 70 – 100
- **Black ground** — the base canvas is pure black

### Same Seed → Many Surfaces

A single seed drives every instance on the page:

| Instance | Purpose | Tiling |
|----------|---------|--------|
| **Cover** | Hero composition | 1 × 1 |
| **Plate** | Main work (master) | 1 × 1 |
| **Frame** | Framed archival print | 1 × 1 |
| **Print** | Archival print surface | 1 × 1 |
| **Scarf** | Silk scarf (3:1) | 3 × 1 |
| **Textile** | Fabric yardage (4:3) | 2 × 2 |
| **Wall** | Wallpaper (2:3) | 1 × 2 |
| **Archive** | 8 unique seeds | 1 × 1 each |

### Controls

- **New Seed** button / `R` key — regenerate with a fresh seed
- **Download** button / `S` key — export the plate canvas as PNG
- **Click any archive plate** — load that seed into the main view

### Performance

- Debounced resize handling (250ms)
- `IntersectionObserver` pauses off-screen sketches
- `visibilitychange` pauses rendering when the tab is hidden
- `pixelDensity(1)` for a lighter canvas footprint
- Two-frame render (`maxFrames: 2`) so tiles draw once and stop

---

## File Structure

```
Recursive-Grid-Generative-Art/
├── index.html          # Single-file page (structure + styles + generative engine)
├── README.md
└── images/
    ├── fav.svg         # Favicon
    ├── tote.png        # Mockup: tote bag
    ├── tee.png         # Mockup: t-shirt
    └── cushion.png     # Mockup: cushion
```

No build step. No dependencies to install. p5.js is loaded from a CDN.

---

## Getting Started

1. Clone the repo:
   ```bash
   git clone https://github.com/reyrove/Recursive-Grid-Generative-Art.git
   cd Recursive-Grid-Generative-Art
   ```
2. Add the three mockup images to `images/` (optional — they appear in the Commission section).
3. Open `index.html` in a browser.

That's it. The page self-initialises on `DOMContentLoaded`.

---

## How the Generative System Works

### Seeded Random

A linear congruential generator replaces p5's native randomness inside each sketch:

```js
class SeededRandom {
  constructor(s) { this.state = (Math.abs(s) % 2147483647) || 1; }
  rand() { this.state = (this.state * 9301 + 49297) % 233280; return this.state / 233280; }
  range(min, max) { return this.rand() * (max - min) + min; }
  int(min, max) { return Math.floor(this.range(min, max + 1)); }
  pick(arr) { return arr[Math.floor(this.rand() * arr.length)]; }
}
```

Every aesthetic decision — number of columns, number of rows, jitter, size variance, recurse-or-draw, colour — is drawn from this generator in a fixed order. Same seed, same sequence, same composition.

### Parameter Space (per seed)

| Parameter | Range |
|-----------|-------|
| Starting depth | 3 levels |
| Columns per split | 2 – 4 |
| Rows per split | 2 – 4 |
| Position jitter | ±10% of cell dimension |
| Size variance | 0.8 – 1.1 |
| Recurse probability | 70% |
| Minimum cell size | 5% of canvas |
| Colour hue | 0 – 360 |
| Colour saturation | 50 – 100 |
| Colour brightness | 70 – 100 |
| Colour alpha | 0.8 |

### The Recursive Rule

At each call, the rectangle is divided into a grid of `cols × rows` cells. For each cell:

```js
if (sr.rand() < 0.7) {
  // Recurse deeper
  recursiveGrid(p, sr, nx, ny, Math.min(nw, nh), depth - 1, cs);
} else {
  // Draw a flat rectangle
  p.fill(sr.rand() * 360, sr.range(50, 100), sr.range(70, 100), 0.8);
  p.rect(nx, ny, nw, nh);
}
```

With three levels of recursion, the canvas can hold anywhere from a handful of large rectangles (if many branches draw early) up to a few hundred small ones (if many branches recurse deep). The 70/30 split biases toward recursion, so most seeds produce a busy mosaic rather than a sparse grid.

### The Jitter and Size Variance

Each cell's position and size is perturbed after the split:

```js
const nx = x + i * w + sr.range(-w * 0.1, w * 0.1);
const ny = y + j * h + sr.range(-h * 0.1, h * 0.1);
const nw = w * sr.range(0.8, 1.1);
const nh = h * sr.range(0.8, 1.1);
```

The jitter breaks grid alignment, so no edge lands perfectly on the underlying split. The size variance means cells overlap or leave gaps. Together, they make the composition read as hand-placed rather than mechanically generated.

### Colour

Each rectangle is drawn with a fresh random HSB colour, sampled independently. There is no palette — every cell picks its own hue from the full spectrum. Against the black ground, this produces a dense, vibrant mosaic without any dominant colour family.

If you want the composition to feel more controlled, the easiest change is to constrain the hue range — for example, drawing hue from a smaller window around a per-seed base hue.

### Tiling Logic

Rectangular surfaces (scarf, textile, wall) tile the square composition N×M. Each tile is drawn into its own translated coordinate space, centred within its cell:

```js
const tileW = canvasW / tileCols;
const tileH = canvasH / tileRows;
const tileSize = Math.min(tileW, tileH);
// draw square of side tileSize at (ox, oy) for each cell
```

This preserves the square composition's proportions regardless of surface aspect ratio.

---

## Customisation

### Change the default seed

Edit the init function:

```js
function init() {
  seed = 123456789;  // fixed seed instead of random
  // ...
}
```

### Adjust the recursion depth

Inside `drawComposition`:

```js
recursiveGrid(p, sr, 0, 0, cs, 3, cs);
```

Change `3` for deeper or shallower recursion. Depth 2 produces larger, more legible rectangles; depth 4 or 5 produces a finer mosaic. The base-case guard (`size < cs * 0.05`) prevents runaway recursion if you go too high.

### Adjust the split grid

```js
const cols = sr.int(2, 4);
const rows = sr.int(2, 4);
```

Change to `sr.int(2, 3)` for coarser splits, or `sr.int(3, 6)` for finer ones. The current `2–4` range reads as a gentle grid.

### Adjust the recurse probability

```js
if (sr.rand() < 0.7) {
  // recurse
}
```

- **Higher** (0.85, 0.9) — most cells recurse, producing a denser mosaic.
- **Lower** (0.4, 0.5) — more cells draw early, producing larger, more legible rectangles.

### Adjust the jitter

```js
const nx = x + i * w + sr.range(-w * 0.1, w * 0.1);
```

Change `0.1` to a smaller value (0.02) for a tighter, more aligned grid, or a larger value (0.2) for a looser, more chaotic one.

### Adjust the size variance

```js
const nw = w * sr.range(0.8, 1.1);
```

- **Tighter range** (0.95, 1.05) — cells tile cleanly with minimal overlap.
- **Wider range** (0.6, 1.3) — dramatic overlaps and gaps.

### Constrain the colour range

Currently every rectangle draws hue from 0–360. To narrow the palette:

```js
const baseHue = sr.rand() * 360;   // draw once per composition
// ...
p.fill(
  (baseHue + sr.range(-40, 40) + 360) % 360,   // narrow hue window
  sr.range(50, 100),
  sr.range(70, 100),
  0.8
);
```

This pulls every cell into a coherent family — useful if you want the composition to feel like a single surface rather than a mosaic.

### Change the ground

```js
p.background(0, 0, 0);
```

Replace with any HSB triple. A warm dark ground (`background(30, 20, 10)`) makes the colours feel warmer; a deep blue (`background(220, 60, 10)`) makes them feel cooler.

### Change surface tiling

Edit `surfaceDefs`:

```js
const surfaceDefs = [
  { id: 'surfacePrint',   wrapId: 'wrap-surfacePrint',   tile: { cols: 1, rows: 1 } },
  { id: 'surfaceScarf',   wrapId: 'wrap-surfaceScarf',   tile: { cols: 3, rows: 1 } },
  // ...
];
```

The `tile` object accepts any `{ cols, rows }` pair.

### Replace mockup images

Swap the files in `images/` or update the `src` attributes in the Commission section:

```html
<img src="images/your-mockup.png" alt="..." loading="lazy" />
```

---

## Design System

| Token | Value | Use |
|-------|-------|-----|
| `--bg` | `#0a0a0a` | Page background |
| `--paper` | `#f4f1ea` | Light surface (Surfaces section) |
| `--gold` | `#b8935a` | Primary accent |
| `--gold-soft` | `#d4b483` | Italic emphasis |
| `--serif` | Cormorant Garamond | Headings, titles |
| `--mono` | DM Mono | Labels, metadata, UI |

Type scale, spacing, and section rhythm match the Skyline, Portrait, Girih, Gothic Grid, Order to Chaos, Wallpaper Groups, Wandering Tiles, Food Grid, Framed Line, Deconstructed Stage, Layered Diagram, Japanese Patterns, Matrix Rain, and Organic Shapes templates. Responsive breakpoints at 900px, 720px, 560px, and 400px.

---

## Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `R` | Regenerate with a new seed |
| `S` | Download plate as PNG |
| `Enter` / `Space` | Activate focused archive item |
| `Esc` | Close legal modal |

---

## Browser Support

Modern evergreen browsers with:

- Canvas 2D (universal)
- `IntersectionObserver` (universal in modern browsers)
- `backdrop-filter` (Safari 9+, Chrome 76+, Firefox 103+)
- CSS custom properties (universal)

Fallback: if `IntersectionObserver` is unavailable, sketches render continuously.

---

## Accessibility

- Archive items are keyboard-focusable with `role="button"` and `aria-label`
- Legal modal traps focus, closes on `Esc` and overlay click
- `prefers-reduced-motion` disables transitions and smooth scroll
- All interactive elements have visible focus states (via browser default)

---

## Section Map

| Section | ID | Purpose |
|---------|-----|---------|
| Cover | — | Hero title + live composition |
| Statement | `#statement` | Artist statement |
| Plate | `#work` | Main work + metadata + controls |
| Surfaces | `#application` | Framed print + 4 tiled surfaces |
| Process | `#process` | Three process cards |
| Archive | `#archive` | 8 selectable seeds |
| Commission | `#commission` | Services + mockups + CTA |
| Colophon | — | Footer, links, legal |
| Legal modal | `#legalModal` | Licensing / Terms / Credits |

---

## Deployment (GitHub Pages)

1. Push the repo to GitHub.
2. Go to **Settings → Pages**.
3. Under **Source**, select `Deploy from a branch`.
4. Choose branch `main` and folder `/ (root)`.
5. Save — the site goes live at `https://reyrove.github.io/Recursive-Grid-Generative-Art/`.

---

## Notes on the Source Sketch

The generative engine is adapted almost verbatim from the original p5 sketch. The adaptations that turn it into a catalogue page:

1. **Seeded random replaces p5's `random()`** — a local `SeededRandom` class drives every choice. The original sketch calls `setSeeds()` for fxhash compatibility; that's replaced with the seeded generator directly.
2. **Sketch factory** — `makeSketch({ slotId, seedValue, ... })` returns a p5 sketch that can be instantiated into any DOM slot, enabling cover / plate / frame / surfaces / archive from one seed.
3. **DOM-driven canvas sizing** — `cs = min(windowWidth, windowHeight)` becomes slot-rect dimensions.
4. **Tiling support** — the `tile: { cols, rows }` parameter repeats the square composition across rectangular surfaces.
5. **Drawing logic takes `p` explicitly** — the original used p5's global mode; here `p` is passed so the factory closure works.
6. **`recursiveGrid` signature extended** — the original takes `(x, y, size, depth)`. Inside the factory, it takes `(p, sr, x, y, size, depth, cs)` so the recursion can thread its own seeded generator and canvas size without relying on globals.
7. **`windowResized` replaced by a debounced `handleResize`** — all sketches are destroyed and recreated on resize (250ms debounce), since their canvases are locked to slot dimensions.
8. **`triggerPreview` calls consolidated** — the original calls it twice (once in `setup`, once in `drawArt`). Not needed in the plain-browser catalogue, and removed entirely.

---

## Related Repositories

| Repo | Theme |
|------|-------|
| [`Skyline-Series`](https://github.com/reyrove/Skyline-Series) | Themed cityscapes |
| [`Portrait-Series-Generative-Art`](https://github.com/reyrove/Portrait-Series-Generative-Art) | Generative faces |
| [`Girih-2-Generative-Art`](https://github.com/reyrove/Girih-2-Generative-Art) | Islamic geometric ornament |
| [`Gothic-Grid-Generative-Art`](https://github.com/reyrove/Gothic-Grid-Generative-Art) | Mosaic grid composition |
| [`Order-to-Chaos-Generative-Art`](https://github.com/reyrove/Order-to-Chaos-Generative-Art) | Gradient grid composition |
| [`Wallpaper-Groups-Generative-Art`](https://github.com/reyrove/Wallpaper-Groups-Generative-Art) | Crystallographic symmetry |
| [`Wandering-Tiles-Generative-Art`](https://github.com/reyrove/Wandering-Tiles-Generative-Art) | Line-drawing composition |
| [`Food-Grid-Generative-Art`](https://github.com/reyrove/Food-Grid-Generative-Art) | Illustrated food grid |
| [`Framed-Line-Generative-Art`](https://github.com/reyrove/Framed-Line-Generative-Art) | Gesture line composition |
| [`Deconstructed-Stage-Generative-Art`](https://github.com/reyrove/Deconstructed-Stage-Generative-Art) | Suprematist composition |
| [`Layered-Diagram-Generative-Art`](https://github.com/reyrove/Layered-Diagram-Generative-Art) | Layered linework composition |
| [`Japanese-Patterns-Generative-Art`](https://github.com/reyrove/Japanese-Patterns-Generative-Art) | Traditional motif composition |
| [`Matrix-Rain-Generative-Art`](https://github.com/reyrove/Matrix-Rain-Generative-Art) | Digital rain composition |
| [`Organic-Shapes-Generative-Art`](https://github.com/reyrove/Organic-Shapes-Generative-Art) | Layered petal-form composition |
| **`Recursive-Grid-Generative-Art`** | Recursive subdivision composition *(this repo)* |

All fifteen share the same catalogue template, design tokens, and seed-based reproducibility model.

---

## Credits

- **Design & Generative System** — Reyhaneh Daneshdoost
- **Typefaces** — Cormorant Garamond, DM Mono
- **Engine** — p5.js 1.9.0
- **Edition** — Recursive Grid, Autumn 2026

---

## License

All artwork, code, and generative systems are the intellectual property of the artist. Compositions may not be reproduced, redistributed, resold, or adapted without a written license.

For licensing, commissions, or private systems: **reyhanehdaneshdoost@gmail.com**
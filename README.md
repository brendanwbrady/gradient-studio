# Gradient Studio

A browser-based generator for soft, grained gradient fields, built to hand off cleanly
into Figma as editable vectors or as flat raster art.

The whole tool is one self-contained `index.html`. No build step, no dependencies,
no package install. Open the file locally or serve it from any static host and it runs.

---

## Running it

**Locally** — open `index.html` in a browser. That's the whole process.

**Hosted** — any static host will serve it. See [Deploys](#deploys) below.

The only network request is to Google Fonts (DM Sans, Geist Mono). Offline, it falls
back to system fonts and everything else still works.

---

## What it makes

Every output is the same anatomy: a ground colour, one or more soft masses carrying a
three-stop gradient, and a grain layer on top. That structure is deliberately fixed —
it is the guardrail that keeps results in family rather than anywhere.

### Export

| Format | Notes |
| --- | --- |
| **Copy SVG** | Paste straight onto a Figma canvas. Vector shapes, gradients, blur and grain arrive as editable layers. |
| **SVG** | Same output as a downloaded file. |
| **PNG 1× / 2× / 4×** | Rasterised from the identical SVG, so the two never drift apart. |

Frames: 8:9 (960×1080), 16:9, 1:1, 9:16, and Phone (1179×2556).

---

## The panels

| Panel | Controls |
| --- | --- |
| **Palette** | Room and traditional presets, three colour stops, image sampling |
| **Frame** | Output dimensions |
| **Field** | Form (5 geometric, 6 organic), width, margin, blur |
| **Light** | Direction, middle point, falloff, bloom |
| **Hand** | Organic, seed |
| **Motion** | Drift, direction (across / along / spin) |
| **Cast** | Pattern (shoji / slat / dapple / leaves), light or shadow, strength, scale, position |
| **Finish** | Saturation, contrast, highlights, shadows, grain |

**Seed** governs every random decision — wave phases, gradient tilt, bloom placement,
cast layout, grain pattern. Same seed plus same settings reproduces a file exactly.

**Randomize** rolls composition, palette, light, motion and cast within bounded ranges.
It deliberately holds the frame and the entire Finish panel, which are calibration
rather than exploration.

**Undo/redo** is ⌘Z / ⇧⌘Z across every control, including palette loads and image
sampling. Panel open/closed state is excluded, so undo never moves the interface.

---

## Deploys

Because it is a single static file, hosting is trivial. Two options:

### GitHub Pages — one live URL

Settings → Pages → Deploy from branch → `main` / root. Publishes to
`https://<org>.github.io/<repo>/` within a minute of each push to `main`.

Simplest possible setup, but it only ever serves one branch.

### Netlify or Cloudflare Pages — a URL per branch

Connect the repo; no build command, publish directory `.`. Both then give:

- a production URL from `main`
- a **unique preview URL for every branch and every pull request**

That second point is the reason to prefer this over Pages: work in progress becomes a
link that can be shared and reviewed before it lands on `main`. `netlify.toml` in this
repo is already configured for it.

---

## Working on it

`main` is what the live URL serves, so it should always be in a shareable state.

Anything non-trivial goes on a branch:

```
git checkout -b field-shapes
# edit index.html
git commit -am "Add tapered plume variant"
git push -u origin field-shapes
```

That push produces a preview URL. Open a pull request to merge it into `main`.

Tag releases at meaningful points so a specific build can always be recovered:

```
git tag -a v22 -m "Leaves cast, spin blur, highlights and shadows"
git push --tags
```

Before changing the render engine, read [`NOTES.md`](NOTES.md). It documents the
constraints the SVG export depends on — several of them are non-obvious, and breaking
one produces output that looks correct in a browser and fails on import into Figma.

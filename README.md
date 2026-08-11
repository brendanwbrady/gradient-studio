[README (1).md](https://github.com/user-attachments/files/30945745/README.1.md)
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

Hosting is entirely inside GitHub — GitHub Pages, driven by two workflows in
`.github/workflows/`. There is no external service and no build step.

**One-time setup**

1. Push this repo, including `.github/workflows/`.
2. The Deploy workflow runs and creates a `gh-pages` branch.
3. Settings → Pages → Source: *Deploy from a branch* → `gh-pages` → `/ (root)`.

The live site then serves from:

```
https://<owner>.github.io/<repo>/
```

**What runs**

| Workflow | Trigger | Result |
| --- | --- | --- |
| `deploy.yml` | push to `main` | publishes to the site root |
| `preview.yml` | pull request opened or updated | publishes to `/preview/pr-<N>/` and comments the URL on the PR |
| `preview.yml` | pull request closed | deletes that preview folder |

Both authenticate with the `GITHUB_TOKEN` the runner creates automatically. No
personal access token, no secrets to configure.

**Worth knowing**

- Previews are per **pull request**, not per branch. Push a branch and open a pull
  request — draft is fine — to get a URL.
- GitHub Pages on a **private** repository requires a paid GitHub plan. On a free
  plan the repository has to be public for any of this to publish.
- Deploys take under a minute; Pages can take another minute to serve the new file.

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

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
| **Cast** | Pattern (coupe / flute / piano / chandelier), strength, angle, blur, density, scale, position |
| **Finish** | Saturation, contrast, highlights, shadows, grain |

**Seed** governs every random decision — wave phases, gradient tilt, bloom placement,
cast layout, grain pattern. Same seed plus same settings reproduces a file exactly.

**Randomize** rolls composition, palette, light and motion within bounded ranges.
It deliberately holds the frame, the entire Finish panel and the entire Cast panel —
calibration and deliberate finishing touches, rather than exploration.

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
Anything you would not want a client to open by surprise goes on a branch.

### Naming

**Branches** — `type/what-it-is`, lowercase, hyphens, no dates or initials. The type
prefix makes the branch list sort itself, which matters once there are a dozen.

| Prefix | For | Example |
| --- | --- | --- |
| `feat/` | a new capability | `feat/cast-barware` |
| `fix/` | something behaving wrongly | `fix/cast-gradient-fade` |
| `ui/` | interface and layout only | `ui/collapsible-panels` |
| `art/` | shapes, palettes, traced silhouettes | `art/piano-silhouette` |
| `exp/` | an exploration that may never merge | `exp/conic-gradients` |
| `docs/` | README, NOTES, workflow files | `docs/engine-notes` |

Name the branch after the outcome, not the task: `feat/cast-barware`, not
`update-cast`. In six months the branch list is the only record of what was tried.

**Commits** — imperative mood, under about 60 characters, finishing the sentence
*"this commit will…"*:

```
Trace barware silhouettes from reference art
Fix cast fading out over the dark end of the gradient
Pin cast to a single object
```

Not `updates`, `wip`, `changes`. If a commit needs a *why*, put it in the body —
that is where the reasoning survives.

**Releases** — `v<major>.<minor>`, tagged on `main` when a build is worth returning
to. Minor for new capability or refinement, major when the output changes enough
that earlier exports no longer match. Give each one a title in the release notes:

```
v1.0  Barware casts
v1.1  Cast density and blur
```

Start at `v1.0` rather than continuing the `v22` numbering — those numbers were
filenames in a chat, not releases, and carrying them over means the tags start
mid-story with no `v1` to point at.

### The process

1. **Code** tab → branch dropdown (it says `main`) → type the new branch name →
   **Create branch: … from main**.
2. Confirm the dropdown now shows your branch. Everything you commit goes there.
3. Edit `index.html` — the pencil icon to edit in place, or **Add file → Upload
   files** to replace it with a new copy. Commit.
4. **Pull requests** → **New pull request** → base `main`, compare your branch →
   **Create pull request**. Draft is fine.
5. Within a minute a preview URL is posted as a comment. That link is what you
   share for review. Pushing more commits updates the same URL.
6. When it is approved: **Merge pull request** → **Delete branch**. `main` deploys
   to the live URL, and the preview cleans itself up.
7. If the merge is a milestone: **Releases** → **Draft a new release** → new tag
   `v1.x` → title and notes → **Publish**.

A branch nobody has opened a pull request for has no preview URL. The pull request
is what makes the work reviewable, so open it early and mark it draft rather than
waiting until the work is finished.

Before changing the render engine, read [`NOTES.md`](NOTES.md). It documents the
constraints the SVG export depends on — several of them are non-obvious, and breaking
one produces output that looks correct in a browser and fails on import into Figma.

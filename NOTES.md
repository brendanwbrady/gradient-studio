# Engine notes

Constraints the render engine depends on. Each was arrived at by fixing a real
failure, and each is easy to break by accident because **the browser keeps rendering
correctly after the mistake** — the damage only shows up on import into Figma, or at a
different render scale.

Read before changing `buildSVG()` or anything it calls.

---

## 1. The noise filter must match Figma's own grammar exactly

Figma emits exactly two shapes of noise filter, and its importer recognises those two
and nothing else. An unrecognised primitive anywhere in the chain causes the **whole
filter to be dropped**, which shows up as grain silently vanishing on paste.

Permitted chains:

```
feFlood → feBlend → feTurbulence → feComponentTransfer → feComposite → feMerge
feFlood → feBlend → feTurbulence → feColorMatrix(luminanceToAlpha)
        → feComponentTransfer → feComposite → feFlood → feComposite → feMerge
```

A `feColorMatrix type="saturate"` was once inserted mid-chain to desaturate the
speckle. It is not a shape Figma emits, and it broke the import.

Where tonal control over grain is needed, it goes on the **plate** — the rect's fill,
opacity and blend mode are ordinary layer properties that survive import — never as a
filter primitive. The plate currently carries a three-stop gradient so grain eases off
at both ends of the tonal range.

Filter and gradient IDs follow Figma's own naming (`filter0_f_1234_56`,
`paint0_linear_1234_56`). Cheap to maintain, and it matches what the importer expects.

## 2. Noise frequency must stay clear of the pixel sampling limit

`baseFrequency` is in user units. At `2`, the noise period is half a pixel — right at
the sampling limit, where rendered strength swings unpredictably with render scale.

That single value was the root cause of a long run of unrelated-looking bugs: grain
"too strong" on load, grain "missing" from PNG export, grain inconsistent between
preview and export. All one problem.

**Keep the period above roughly one pixel.** The current `0.85` gives ~1.18px.
Measured grain now agrees to within 1% across the on-screen preview, the exported SVG
at native size, and the PNG export.

## 3. No transform attributes in exported geometry

Scale, rotation and position are resolved in JavaScript into absolute path
coordinates. Nothing in the export carries a `transform`, and no filtered group
contains another transformed group.

This applies to Cast, Smear, Veil and every motion sample. `bake()` and `rotPts()`
exist for exactly this purpose.

Consequence worth knowing: a `userSpaceOnUse` gradient is normally transformed along
with its element. With coordinates baked, gradient endpoints must be transformed
explicitly too — Smear and Veil do this so the fade stays aligned to the band. Cast
deliberately does **not**, so its fade stays locked to the frame's light axis rather
than tilting with the pattern.

## 4. Blur radii stay symmetric

`feGaussianBlur stdDeviation="180 12"` renders beautifully in a browser and is not
something Figma's uniform layer blur can represent.

Motion is therefore **geometric, not filtered**: the shape is sampled several times
along its travel and composited, each sample fainter than the last. Across and Along
offset the samples, Spin rotates them through an arc. Every blur in the file is a
single symmetric value.

The cost is layer count. Motion multiplies the field by five to nine samples; drift
plus a cast has reached seventeen groups against four at rest. Motion is the expensive
control.

## 5. Blur eats the ends of a gradient ramp

A colour that exists only at the extreme end of a ramp sits at the very edge of the
shape, where blur immediately averages it against the ground. The authored colour then
never actually appears in the output.

**Falloff** exists to solve this: it holds both the light and the ground as plateaus
so each survives the blur, and bends the shoulders so the transition reads as decay
rather than as a linear blend. Measured on a light stop of luminance 197, peak output
went from 164 to 196 once this was in place.

## 6. Blend modes go on the group, never on the shape

Figma writes `mix-blend-mode` on the **shape** inside a filtered group, and applies
it at the layer level:

```
<g filter="url(#filter1_n)"><rect ... style="mix-blend-mode:multiply"/></g>
```

In a browser that is a **no-op**. The filter on the group creates isolation, so the
rect blends against an empty backdrop and the layer paints straight over the
artwork instead of multiplying with it. This was live for a long time and showed up
as grain washing the picture out — shadows lifting 17 levels and contrast dropping
13 at moderate grain.

Blend modes therefore go on the **group**, which blends the filtered result against
the artwork and behaves correctly in a browser. This is the one place the export
deliberately differs from Figma's own serialisation, and it is the single remaining
Figma-fidelity assumption in the file: that Figma honours a blend mode on a group.

Two things limit the damage if it ever turns out it does not:

- Every blended layer sits above an **opaque background rect**, so the backdrop is
  always the artwork itself and never the canvas behind it.
- Grain strength lives in the **flood colour**, not the plate. Plate alpha is kept as
  low as the speckle allows, so a dropped blend costs about 7 levels in the shadows
  rather than 18.

To check it in 30 seconds: paste an export with grain at 20% into Figma and compare
the shadow area against the PNG export of the same settings. If the shadows look
lifted and flat, the group blend was dropped — move the blend onto the rect and
accept that the browser preview will be wrong instead.

## 7. The glint is photographic light, not drawn shadow

The cast is not drawn any more. It is a photograph of the object backlit on a black
ground, embedded as a data URI and composited with **screen** — so the black adds
nothing and only the rim light lands on the gradient. Nothing occludes: the object
does not block the picture, it contributes highlights to it, which is what the
direction wants now that the gradient itself is the dark room.

A vector rim can be made to fall off correctly, but it cannot carry a flare, the
double edge where glass turns back on itself, or the way a highlight breaks over a
moulding. Those are the whole point, so the photograph is used as it is.

The glint is **two plates of the same shot**, registered to each other and drawn
into the same box:

- the **black-ground original**, which carries the soft glow, the flare and the
  falloff either side of each highlight;
- the **transparent extraction**, which carries the crisp line but threw that away.

Both composite with **colour-dodge**, never screen. Black is an identity for dodge
as well, so the black-ground plate still hides its own background, and dodge is what
makes the light answer to the gradient underneath. Screen for the glow puts the soft
light back but wrecks the tonal balance — measured bright-to-dark contrast fell to
0.16, against 0.42 for the extraction alone and 0.38 the current way.

The two plates are delivered at **identical dimensions from the same shot**, so they
register with no correction: measured correlation 0.95 to 0.97 with no adjustment at
all. Both are cropped with **one shared box** — the union of the two content bounds,
which also keeps the glow that spills past the extraction's alpha edge — and both are
resized together, so they fill the same rect exactly. Measured offset between the two
rendered layers: 0px in both axes.

An earlier pair was delivered at different sizes and did **not** register; it had to
be aligned by correlation search over scale and offset, and left a 3px by 6px
residual. **Check the correlation before trusting a crop.** If a future pair does not
land near 0.95 unadjusted, the search has to come back — assuming alignment leaves a
visible double edge on every rim.

The transparent plates are **PNGs** — the object's rim carried in the alpha channel,
with everything else fully clear. That is what removed the blend-mode contortions:
there is no black ground left to hide, so the compositing no longer has to hunt for
a blend that treats black as identity.

They are stored as **luminance + alpha only**. The colour in the source is discarded,
because the tint pipeline maps luminance through the palette anyway, and two channels
compress to about a third of what RGBA does with nothing lost that is used.

The tinted plate is rendered at **1.4x the size the object is actually drawn**, not at
source resolution. Measured against a 2x PNG export, 1.4x and 2x are pixel-identical
— the cast blur and the source-to-display downscale are the limiting factors — and it
halves the export.

**Every new object needs a gain.** The plates carry very different amounts of light:
measured lit area is 5.4% for the coupe, 7.9% for the flute, 1.9% for the piano and
2.0% for the chandelier, while per-pixel brightness is near identical at 161-171
across all four. Rendered at the same Strength that left the chandelier delivering a
quarter of the coupe's light. `GLINT_GAIN` corrects it, applied as a screen curve on
the mapped colour — not layer opacity, which was already clamping at 1, and not a
linear multiply, which blows the specular cores.

The curve saturates, so parity is not reachable and should not be chased: past the
current values the chandelier's line flattens toward a uniform white and stops
reading as a lit tube. Spread is 4.25x down to 1.76x, which is close enough that
Strength behaves consistently.

To set a gain for a new object, render it against the coupe at the same Strength and
compare total light — the sum of the per-pixel lift over the no-glint render.

Preparing a new object:

1. Shoot or generate it **backlit with rim lighting**, then deliver it as a
   **transparent PNG** with the background removed — not black.
2. Crop to the alpha bounds with a small margin.
3. Resize to 1800px on the long edge.
4. Convert to **luminance + alpha** and save as an optimised PNG. About 160-320 KB
   each, 720 KB for four.
5. Add to `CAST_IMG` as `{w, h, src}` with a base64 data URI.

The glint composites in **two passes**, and the choice of blends is not free:

- **Colour-dodge** divides by the inverse of the source, so it scales with what is
  already underneath — the object burns in where it crosses the lit band and falls
  away into the dark. That is the response wanted, and crucially black is an
  identity for it, so the photograph's ground still contributes nothing.
- **Screen** underneath at low opacity sets a floor, so the object never disappears
  entirely in the darkest corners.

**Overlay and soft-light cannot be used here.** Neither is transparent to black, so
the photograph's background prints as a dark rectangle over the artwork. Only screen
and colour-dodge leave black alone.

The floor pass uses a **downscaled copy** of the same plate — it is faint and
blurred, so a 460px version is visually identical there, and it keeps the export
from carrying the same photograph twice at full size. That is the difference between
a 409 KB and a 224 KB paste for the coupe.

Two constraints worth keeping:

- The href must stay a **data URI**. An external URL would break the moment the SVG
  left this page.
- Motion does not apply to the cast any more. Each motion copy would duplicate the
  encoded image, and a `<use>` reference is not something Figma's importer can be
  relied on for. Drift affects the field only.

There is no rotation control, so the export carries no `transform` at all.

The photograph is **gradient-mapped into the palette** before it is embedded: its
luminance runs through black → middle stop → light stop, with only the hottest
cores pushing to white the way a real specular does. The map is baked into the
pixels on a canvas rather than applied as an SVG filter, so the exported file
carries a plain `<image>` and needs no colour primitive Figma might not honour.
Results are cached per object and palette, since re-encoding on every slider move
would be slow.

## 7b. The earlier vector rim, kept for reference

The cast does not draw a silhouette. The object is barely present — a faint
thickening of the dark — and what reads is the catch of light on the edges that
turn toward the source.

The rim is geometric, never a stroke. It is the crescent between the outline and a
copy of itself pushed a little **away** from the light, so it comes out naturally
wide where an edge faces the light square-on and vanishes where the edge turns
away. A stroke would be even all the way round and read as an outline; this cannot,
because the falloff is a property of the shape.

The crescent is cut into short arcs. Each is brightened by how much it faces the
light, knocked about by the cast's own generator, and about a quarter are dropped
entirely — the gaps are what make it read as light finding the object rather than
tracing it. The arcs are sorted into four brightness tiers and emitted as four
paths, so the whole rim costs four layers rather than one per arc.

Two things worth keeping:

- The rim follows the **Light panel's direction**, not the field's seed-tilted
  light axis. Using the tilted axis makes Shuffle swing the lit edge around, and
  the cast is meant to be immune to the seed.
- A highlight over a bright part of the field is invisible; the same highlight over
  the dark reads strongly. That asymmetry is correct and is the point of the
  direction — the object appears where the picture is darkest.

## 8. Motion is the expensive control

Motion is geometric, so every sample is a real layer. A cast under drift reaches
about 32 layers and 58 KB, against 5 and 8 KB at rest. Everything still edits
normally in Figma, but that is the cost of the control.

**Motion copies must average, not stack.** Painted source-over at their own weights,
fifteen copies of one shape reach full alpha wherever they overlap, so a trail comes
out as a solid slab rather than a graded blur — and any edge running along the
direction of travel is drawn by every copy at once, which turns it into a hard seam.
A horizontal drift produced exactly that at the field's lower edge.

Each copy composites at `w / (running sum of w)`, which makes the result the weighted
mean of them all — the definition of a motion blur. The first copy lands at full alpha
and each later one contributes proportionally less. Measured on the seam: sharpest
horizontal step fell from 3.19 to 1.79, and motion stopped inflating the overall
brightness of the frame as a side effect.

Two further rules keep motion from breaking a hard-edged shape:

- Motion copies of a cast live inside **one multiplying group** and composite
  normally within it. Multiplying each copy separately makes the overlaps go black
  and the trail read as several objects instead of one in motion.
- The blur grows to at least **half the gap between copies**, so stamps fuse into a
  continuous trail rather than reading as discrete ghosts.

A cast also travels less than the field around it, and a spin turns it about its
own centre rather than the frame's — otherwise a shadow standing at the foot of the
frame gets flung across it.

## 9. Verify against measurement, not appearance

Several of the above were invisible by eye and obvious in numbers. Useful checks:

- **Grain stability** — high-pass sigma of preview vs PNG vs SVG-at-native-size.
  These three must agree.
- **Grain neutrality** — mean, shadow and contrast shift between grain 0 and grain
  20 should all be near zero. Grain that moves them is compressing the picture.
- **Noise period** — `baseFrequency ÷ Grain Scale` must stay under about 1.0, or the
  period drops below a pixel and the texture changes with render size. The Scale
  slider's floor exists for this reason.
- **Highlight fidelity** — brightest pixel in the output vs the authored light stop.
- **Export cleanliness** — count `transform=`, `stdDeviation="N N"`, and `<g...><g` in
  the exported SVG. All three should be zero.
- **Held settings** — after N randomizes, confirm frame and every Finish control is
  unmoved.

Contact sheets built from downscaled thumbnails introduce their own artefacts —
this has produced a false "hard edge at the frame" three separate times, and each
time the export was clean at 1:1. Verify suspected rendering faults at full
resolution before chasing them.

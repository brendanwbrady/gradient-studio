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

## 7. Motion is the expensive control

Motion is geometric, so every sample is a real layer. A cast under drift reaches
about 32 layers and 58 KB, against 5 and 8 KB at rest. Everything still edits
normally in Figma, but that is the cost of the control.

Two rules keep motion from breaking a hard-edged shape:

- Motion copies of a cast live inside **one multiplying group** and composite
  normally within it. Multiplying each copy separately makes the overlaps go black
  and the trail read as several objects instead of one in motion.
- The blur grows to at least **half the gap between copies**, so stamps fuse into a
  continuous trail rather than reading as discrete ghosts.

A cast also travels less than the field around it, and a spin turns it about its
own centre rather than the frame's — otherwise a shadow standing at the foot of the
frame gets flung across it.

## 8. Verify against measurement, not appearance

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

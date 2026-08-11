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

## 6. Verify against measurement, not appearance

Several of the above were invisible by eye and obvious in numbers. Useful checks:

- **Grain stability** — high-pass sigma of preview vs PNG vs SVG-at-native-size.
  These three must agree.
- **Highlight fidelity** — brightest pixel in the output vs the authored light stop.
- **Export cleanliness** — count `transform=`, `stdDeviation="N N"`, and `<g...><g` in
  the exported SVG. All three should be zero.
- **Held settings** — after N randomizes, confirm frame and every Finish control is
  unmoved.

Contact sheets built from downscaled thumbnails introduce their own artefacts. Verify
suspected rendering faults at 1:1 before chasing them.

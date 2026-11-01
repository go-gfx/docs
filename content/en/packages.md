---
title: "Packages"
weight: 20
---

# Packages {#packages}

go-gfx is a single Go module, `github.com/go-gfx/gfx`, split into focused
packages. Import only what you need; each package is usable on its own and
depends on nothing third-party.

## geometry {#geometry}

Points, rectangles and affine transforms — the coordinate math every other layer
shares.

- `Point`, `Rect` value types with the usual predicates and set operations
  (union, intersection, containment).
- `Matrix` 2×3 transform: `Translate`, `Scale`, `Rotate`, `Shear`, composed
  with `Mul`, inverted with `Invert`, and applied with `TransformPoint` /
  `TransformRect`.

Used pervasively: a resize is a scale transform, a codec crops with a `Rect`,
and `vector.ParseData` maps a path's user space to device space through a
`Matrix`.

## color {#color}

Colour science done correctly, in linear light where it matters.

- Colour models: RGB, XYZ, Lab, and conversions between them.
- **sRGB ↔ linear** transfer functions — so blends and resampling happen in
  linear light, not gamma-encoded space.
- **Premultiplied alpha** helpers, and Porter-Duff / separable **blend modes**.

This is the package that keeps compositing and resampling from producing the
dark-fringe artifacts that naive gamma-space math causes.

## raster {#raster}

The shared pixel-buffer and pixel-format substrate — the common image surface
every consumer reads and writes.

- A concrete in-memory pixel buffer with an explicit pixel format.
- Format conversions between buffers (e.g. straight ↔ premultiplied, RGBA ↔
  grayscale) built on `color`.

`raster` is the hand-off type: go-images processes it, go-widgets/painter draws
into it, go-webengine paints layout onto it.

## resample {#resample}

High-quality image resizing, separable and weight-precomputed.

- Kernels: **box / area** averaging (downscale by coverage), **bicubic**
  (Catmull-Rom), and **Lanczos**.
- Separable implementation (horizontal pass into a scratch buffer, then vertical)
  with per-axis weights computed once.
- Validated against reference implementations and the naive 2-D definition.

This is the first package landing in the module; go-images' `Resize` and the
desktop thumbnailer are the first consumers.

## codec {#codec}

One shared set of image decoders and encoders, so no consumer hand-rolls its
own.

- Decoders/encoders for **PNG, JPEG, WebP, ICO, ICNS, GIF, TIFF**.
- Subsumes the ad-hoc ICNS/ICO decoding the desktop stack used to carry, and the
  image loading go-webengine needs.
- **Netpbm and QOI headers are checked against the file** (v0.36.2): the
  decoders refuse a header that describes more raster than the file holds,
  before allocating. A 14-byte QOI header used to ask the allocator for 4.4 TB.
  See [What a Netpbm or QOI header may ask for](https://github.com/go-gfx/gfx#what-a-netpbm-or-qoi-header-may-ask-for)
  ([PR #82](https://github.com/go-gfx/gfx/pull/82)).
- **ICO is decoded by `codec` itself** (v0.36.3), no longer through a
  third-party package: only the chosen directory entry is decoded, nothing is
  allocated from a size field, and alpha follows Chromium's alpha/AND-mask
  rule. See [What an ICO directory may ask for](https://github.com/go-gfx/gfx#what-an-ico-directory-may-ask-for)
  ([PR #83](https://github.com/go-gfx/gfx/pull/83)).

## vector {#vector}

The 2D vector engine renderers build on instead of rolling their own.

- **Paths**: move/line/quad/cubic, with the usual builders, readable back with
  `Path.Each`.
- A **path-data parser**, `ParseData`, for the syntax SVG writes in `d`.
- An **anti-aliased scanline rasterizer** (coverage-based).
- **Strokes** (width, joins, caps) and **gradients** (linear / radial).

go-widgets/painter is the primary consumer; go-webengine paints CSS borders,
backgrounds and shapes through it.

### One path-data parser, three formats

`ParseData` lives here rather than in `svg` because three formats write the
same syntax. SVG 1.1 §8.3 defines it; ODF's `draw:path` carries it verbatim in
`svg:d`; and ECMA-388 Annex C gives OpenXPS's *abbreviated geometry syntax* as
an algorithm which, read against it, is a strict subset with a prefix — the
same command letters minus `T`, the same arc operands in the same order, the
same sweep sense, plus a leading `F0`/`F1` selecting the fill rule.

That prefix is deliberately not handled here: a fill rule belongs to the shape,
not to the outline, so an XPS reader strips it and chooses `EvenOdd` or
`NonZero` itself.

{{< callout type="warning" >}}
**The six numbers are not a row-major matrix.** SVG's `matrix()`, XPS's
`RenderTransform`, ODF's `draw:transform` and PDF's `cm` all write an affine as
`[a b c d e f]`, mapping `(x, y)` to `(a·x + c·y + e, b·x + d·y + f)`. Read
row-major, `b` and `c` swap — which leaves every *unrotated* drawing correct
and every rotated one mirrored. `ParseData` takes a `geometry.Matrix`, whose
named entries (`Xx, Yx, X0 / Xy, Yy, Y0`) make that transposition unwriteable.

This is measured, not supposed: planting that swap left the whole of the `svg`
package's test suite **green**, because every drawing in it is axis-aligned.
A differential run against the parser as it stood before the move reported
**57 518 points differing**.
{{< /callout >}}

# Packages

go-gfx is a single Go module, `github.com/go-gfx/gfx`, split into focused
packages. Import only what you need; each package is usable on its own and
depends on nothing third-party.

## geometry

Points, rectangles and affine transforms — the coordinate math every other layer
shares.

- `Point`, `Rect` value types with the usual predicates and set operations
  (union, intersection, containment).
- `Affine` 2×3 transform: translate, scale, rotate, shear, plus `Concat` /
  `Invert` and application to points and rectangles.

Used pervasively: a resize is a scale transform, a codec crops with a `Rect`, a
vector fill maps device space to user space through an `Affine`.

## color

Colour science done correctly, in linear light where it matters.

- Colour models: RGB, XYZ, Lab, and conversions between them.
- **sRGB ↔ linear** transfer functions — so blends and resampling happen in
  linear light, not gamma-encoded space.
- **Premultiplied alpha** helpers, and Porter-Duff / separable **blend modes**.

This is the package that keeps compositing and resampling from producing the
dark-fringe artifacts that naive gamma-space math causes.

## raster

The shared pixel-buffer and pixel-format substrate — the common image surface
every consumer reads and writes.

- A concrete in-memory pixel buffer with an explicit pixel format.
- Format conversions between buffers (e.g. straight ↔ premultiplied, RGBA ↔
  grayscale) built on `color`.

`raster` is the hand-off type: go-images processes it, go-widgets/painter draws
into it, go-webengine paints layout onto it.

## resample

High-quality image resizing, separable and weight-precomputed.

- Kernels: **box / area** averaging (downscale by coverage), **bicubic**
  (Catmull-Rom), and **Lanczos**.
- Separable implementation (horizontal pass into a scratch buffer, then vertical)
  with per-axis weights computed once.
- Validated against reference implementations and the naive 2-D definition.

This is the first package landing in the module; go-images' `Resize` and the
desktop thumbnailer are the first consumers.

## codec

One shared set of image decoders and encoders, so no consumer hand-rolls its
own.

- Decoders/encoders for **PNG, JPEG, WebP, ICO, ICNS, GIF, TIFF**.
- Subsumes the ad-hoc ICNS/ICO decoding the desktop stack used to carry, and the
  image loading go-webengine needs.

## vector

The 2D vector engine renderers build on instead of rolling their own.

- **Paths**: move/line/quad/cubic, with the usual builders.
- An **anti-aliased scanline rasterizer** (coverage-based).
- **Strokes** (width, joins, caps) and **gradients** (linear / radial).

go-widgets/painter is the primary consumer; go-webengine paints CSS borders,
backgrounds and shapes through it.

# Roadmap

go-gfx is a young foundation extracted from primitives that already existed,
scattered, across the fleet. The plan is to land the substrate package by
package, wiring each consumer over as its package stabilizes — incrementally,
never big-bang (go-widgets/painter in particular is co-edited live and migrates
one primitive at a time).

## Now — bootstrapping

- **Module skeleton** — `github.com/go-gfx/gfx`, Go floor 1.26.4, BSD-3-Clause,
  the package boundary documented.
- **`resample`** — the first landing package: box/area, bicubic (Catmull-Rom)
  and Lanczos, separable and weight-precomputed, validated against the naive
  2-D definition and reference implementations.
- **`raster`** + **`color`** (premultiplied alpha) — the pixel-buffer substrate
  and the colour math `resample` composites through.

## Next — shared codecs

- **`codec`** — PNG, JPEG, WebP, ICO, ICNS, GIF, TIFF, subsuming the hand-rolled
  ICNS/ICO decoding the desktop stack carries today.
- Wire [go-images](https://github.com/go-images/images) resize + the desktop
  thumbnailer onto `resample`.

## Then — geometry & vector

- **`geometry`** — points, rectangles, affine transforms.
- **`vector`** — paths, anti-aliased rasterizer, strokes, gradients — the engine
  [go-widgets/painter](https://github.com/go-widgets) hand-rolls today, migrated
  incrementally.

## Throughout — the quality bar

- **`CGO_ENABLED=0`** across the org; no Skia, no Cairo, no native shims.
- **Zero third-party dependencies** (go-asmgen only at generation time).
- **100% statement coverage**, including error branches, gated in CI.
- **Six architectures** — build + test on amd64 / arm64 natively and loong64 /
  riscv64 / ppc64le / s390x under qemu.
- **SIMD** hot kernels via [go-asmgen](https://github.com/go-asmgen) where they
  earn their keep, each validated bit-for-bit against a scalar oracle.

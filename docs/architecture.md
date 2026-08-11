# Architecture

## Why a foundation

2D graphics in most languages means binding a C engine — Skia, Cairo, Pixman.
Across the Go fleet, several libraries had each grown their *own* copy of the
same primitives:

- an image processor resampling pixels,
- a widget toolkit rasterizing vector paths,
- a web engine decoding image codecs,
- a desktop stack rendering icons and thumbnails.

That is duplicated, separately-tested, separately-buggy code. **go-gfx** extracts
the shared substrate into one pure-Go, cgo-free module so colour blending,
resampling, codecs and path rasterization are written and tested **once**.

It follows the classic layering — Skia / Cairo / `image/draw`: a foundation
below, many renderers on top.

```
                 ┌───────────────┐ ┌────────────────────┐ ┌──────────────┐
   consumers →   │   go-images   │ │ go-widgets/painter │ │ go-webengine │  …
                 └───────┬───────┘ └─────────┬──────────┘ └──────┬───────┘
                         │                   │                   │
                 ┌───────┴───────────────────┴───────────────────┴───────┐
   foundation →  │                      go-gfx/gfx                        │
                 │  geometry · color · raster · resample · codec · vector │
                 └───────────────────────────────────────────────────────┘
                                          │
                              ┌───────────┴───────────┐
   optional SIMD →            │  go-asmgen (gen-time)  │
                              └───────────────────────┘
```

## Boundaries

go-gfx deliberately draws its edges:

- **Not image processing.** Filters, edges, morphology, feature detection —
  that is [go-images](https://github.com/go-images/images), the scikit-image
  analogue, which is built *on* `raster` / `color` / `resample`.
- **Not a widget toolkit.** Widgets, layout and event handling are
  [go-widgets](https://github.com/go-widgets); its `painter` backend is built
  *on* `vector` / `raster` / `color`.
- **Not text shaping.** The text stack (fonts, shaping, glyph rasterization) is
  [go-opentype](https://github.com/go-opentype); it stays separate and is
  composed alongside go-gfx, not inside it.
- **Not 3D.** The bespoke rasterizers in the game/virtio ports are deliberate C
  ports or 3D pipelines and are out of scope.

## Dependencies

go-gfx depends on **nothing** fleet-side at runtime. Its only optional
dependency is [go-asmgen](https://github.com/go-asmgen), pulled in *at code
generation time* to emit SIMD assembly for hot kernels; the committed library
keeps a zero-dependency `go.mod`. The whole module cross-compiles to a static
binary on all six of Go's 64-bit targets.

## Consumers

| Consumer | Builds on | Uses it for |
| --- | --- | --- |
| [go-images](https://github.com/go-images/images) | `raster`, `color`, `resample` | filters, edges, morphology, geometry |
| [go-widgets](https://github.com/go-widgets)/painter | `vector`, `raster`, `color` | path fills, strokes, gradients |
| [go-webengine](https://github.com/go-webengine) | `resample`, `codec`, `vector` | painting HTML/CSS layout to an image |
| [go-widgets](https://github.com/go-widgets)/desktop | `codec`, `resample` | icon & thumbnail rendering |

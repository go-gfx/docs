# go-gfx documentation

**A pure-Go (no cgo) 2D graphics foundation.** Geometry, colour science,
pixel-buffer rasters, high-quality resampling, image codecs and a vector
rasterizer — the shared substrate that sits *below* the fleet's image-processing
and rendering libraries. The Go answer to what Skia, Cairo and `image/draw`
provide elsewhere.

go-gfx is **not** an image-processing library (that is
[go-images](https://github.com/go-images/images), the scikit-image analogue) and
**not** a widget toolkit (that is [go-widgets](https://github.com/go-widgets)).
It is the common layer both stand on, so colour blending, resampling, codecs and
path rasterization are written and tested **once** and shared across the whole
ecosystem.

```go
import (
    "github.com/go-gfx/gfx/codec"
    "github.com/go-gfx/gfx/resample"
)

// Decode, resize with a high-quality kernel, re-encode — all cgo-free.
src, err := codec.Decode(r)
if err != nil {
    log.Fatal(err)
}
thumb := resample.Resize(src, 256, 256, resample.Lanczos)
if err := codec.EncodePNG(w, thumb); err != nil {
    log.Fatal(err)
}
```

!!! note "Bootstrapping"
    go-gfx is a young foundation. The package set below is the target shape of
    the module; packages land incrementally, starting with `resample`, `raster`
    and `color`. This documentation tracks the module as it grows — see the
    [roadmap](roadmap.md) for what ships today.

## The packages

| Package | Purpose |
| --- | --- |
| [`geometry`](packages.md#geometry) | points, rectangles, affine transforms |
| [`color`](packages.md#color) | RGB/XYZ/Lab, sRGB↔linear, premultiplied alpha, blend modes |
| [`raster`](packages.md#raster) | shared pixel-buffer / pixel-format substrate |
| [`resample`](packages.md#resample) | box/area, bicubic (Catmull-Rom), Lanczos resizing |
| [`codec`](packages.md#codec) | PNG, JPEG, WebP, ICO, ICNS, GIF, TIFF decoders/encoders |
| [`vector`](packages.md#vector) | paths, anti-aliased rasterizer, strokes, gradients |

## Where to go next

- [Packages](packages.md) — what each package provides.
- [Architecture](architecture.md) — the foundation layering and why it exists.
- [Roadmap](roadmap.md) — the plan and what ships today.

Source: [github.com/go-gfx/gfx](https://github.com/go-gfx/gfx) · the shared 2D
graphics substrate under [go-images](https://github.com/go-images/images),
[go-widgets](https://github.com/go-widgets)/painter and
[go-webengine](https://github.com/go-webengine).

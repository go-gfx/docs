---
title: "2D on the GPU"
weight: 27
---

# 2D on the GPU {#gpu2d}

[`github.com/go-gfx/gpu2d`](https://github.com/go-gfx/gpu2d) (v0.3.0) renders
2D on the GPU, in pure Go. A frame is one backend-neutral list of **instances**,
one per primitive, 48 bytes each:

- `Rasterize` executes it on the CPU and is the reference.
- Package `webgpu` draws it in the browser through
  [go-gfx/webgpu]({{< relref "webgpu.md" >}}).
- Package `metal` draws it with a compute kernel through
  [go-macos/metal](https://github.com/go-macos/metal).

```go
l, err := gpu2d.NewList(1920, 1080)
l.FillRect(gpu2d.Rect{W: 1920, H: 1080}, gpu2d.RGBA{R: 236, G: 236, B: 236, A: 255})
l.PushClip(gpu2d.Rect{X: 20, Y: 20, W: 400, H: 300})
l.FillRoundRect(gpu2d.Rect{X: 40, Y: 40, W: 120, H: 32}, 8, gpu2d.RGBA{R: 0, G: 122, B: 255, A: 255})
l.PopClip()

err = l.Rasterize(buf)       // CPU reference
r, err := metal.New(1920, 1080)
err = r.Draw(l)              // the same frame on a Mac's GPU
```

Since v0.3.0, `Draw` is `Submit` plus `Wait`, and the pair overlaps the CPU
with the GPU (see [Asynchronous submit](#async)).

## The reference {#reference}

The list draws these primitives, all under nested translations and clips:

- fills and one-pixel outlines of rectangles, square or with anti-aliased
  rounded corners, with any number of rings;
- since v0.2.0, glyph masks (`DrawMask`), scaled images (`DrawImage`), and
  paths (`DrawAlpha`, fed by the go-gfx/gfx/vector rasterizer). Masks and
  images travel in a per-frame data arena that the GPU reads.

These are the primitives of go-widgets/painter's `PixelPainter`, and
`Rasterize` reproduces it **byte for byte**. The `internal/parity` test
compares the two over 3000 random scenes, with 51,947 instances including
2970 masks, 2999 images and 11,251 paths.

## Measured {#measured}

The figures are for a frame shaped like a widget toolkit's (`WidgetFrame`),
measured on an Apple M4 Max at a 1-minute load of 4 to 7, so they are
indicative:

| frame | PixelPainter | WebGPU, Chromium on Metal | Metal compute |
|---|---:|---:|---:|
| 1920 × 1080 | 13.1 ms | 0.99 ms | 1.08 ms |
| 3840 × 2160 | 52.5 ms | 1.62 ms | 2.79 ms |

The GPU columns are whole frames: the draw and the wait for the GPU to finish.

## Asynchronous submit {#async}

On a small frame the GPU is not the cost: an empty dispatch alone is 0.41 ms,
and `Draw` blocks the CPU for all of it. `Renderer.Submit` bins and packs a
copy of the list into one of two slots of input buffers, sends it and returns;
`Wait` collects the oldest frame:

```go
err = r.Submit(l) // returns at once; l may be Reset and re-recorded
record(l)         // the next frame, while the GPU draws this one
err = r.Wait()
frame := r.Pixels() // safe to read only when nothing is in flight
```

Two frames may be in flight, compositing in submission order over the same
`Pixels`. A third `Submit` or a `Draw` with frames in flight is `ErrBusy`, a
`Wait` with none is `ErrNothingInFlight`, and `Close` waits first. It rests on
`Device.Submit` and `Pending.Wait` in go-macos/metal v0.4.0. A 960 × 2834
widget frame on an M4 Max, with 1.5 ms of CPU work per frame (the size of the
gallery's recording), median of 10 interleaved rounds at a 1-minute load of
8.0 to 8.2, so indicative:

| CPU work per frame | `Draw` | `Submit` / `Wait` |
|---|---:|---:|
| 0 | 1.31 ms | 1.33 ms |
| 1.5 ms | 3.17 ms | 1.78 ms |

Recording and binning allocate nothing in steady state; go-macos/metal's
message sends cost 129 allocations (5.4 KB) per frame.

## Presenting in a browser canvas {#present}

Since v0.5.0, `webgpu.Renderer.Present(surface)` copies the renderer's target
into the surface's current texture on the GPU, so the frame goes to the canvas
without coming back to Go memory. Call it after `Draw`; the browser shows the
canvas at the end of the task. The surface must be the renderer's size and be
configured with `webgpu.CanvasDescriptor()`:

```go
desc := webgpu.CanvasDescriptor() // rgba8unorm, RENDER_ATTACHMENT | COPY_DST
// ... configure the canvas surface with desc ...
err = r.Draw(l, ground)
err = r.Present(surface) // copyTextureToTexture, byte-exact
```

`CanvasDescriptor` is rgba8unorm, which is the target's own format, so the copy
is byte-exact, with `COPY_DST` added to the default `RENDER_ATTACHMENT` because
`copyTextureToTexture` requires it of its destination. A surface configured
otherwise is refused with `ErrSurface`.

A caller that replaces a renderer, for instance when a canvas is resized, calls
`Renderer.Destroy()` on the old one (v0.5.1). It releases the buffers and the
target texture and refunds them to the device's quota; without it each
replacement keeps its memory charged. The renderer must not be used afterwards,
and a second `Destroy` is harmless. The pipeline is not refunded, because WebGPU
has no way to destroy one.
[go-widgets/webcanvas](https://github.com/go-widgets/webcanvas) v0.9.0 presents
this way.

## Pixels {#pixels}

| back-end | opaque surface | translucent surface |
|---|---|---|
| Metal compute | identical on every channel | identical on every channel |
| WebGPU | identical on every channel | colour identical; alpha up to 4 apart |

The Metal kernel blends with the reference's integer arithmetic. WebGPU uses
fixed-function blending, which rounds where `PixelPainter` floors
`a + dst·(255−a)/255`. A window surface is opaque, so the difference does not
show there.

## Bounds {#bounds}

- Every coordinate lies within ±2^24, the range in which float32 holds every
  integer exactly.
- Translations and clips saturate, and ring counts are capped.
- Binning for the Metal kernel is limited by `MaxBinned`, and a list past it
  is refused before anything is allocated.
- `FuzzList` checks that nothing is recorded past these bounds and that
  nothing is written past the surface.

## Next {#next}

1. A go-widgets `painter.Painter` adapter, presenting through the browser
   canvas and a `CAMetalLayer`.
2. A WGSL to MSL emitter in [wgsl]({{< relref "wgsl.md" >}}), so that the
   coverage expression is written once.

The design is in
[docs/DESIGN.md](https://github.com/go-gfx/gpu2d/blob/main/docs/DESIGN.md).

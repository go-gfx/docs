---
title: "2D on the GPU"
weight: 27
---

# 2D on the GPU {#gpu2d}

[`github.com/go-gfx/gpu2d`](https://github.com/go-gfx/gpu2d) (v0.12.0) renders
2D on the GPU, in pure Go. A frame is one backend-neutral list of **instances**,
one per primitive, 48 bytes each:

- `Rasterize` executes it on the CPU and is the reference.
- Package `webgpu` draws it in the browser through
  [go-gfx/webgpu]({{< relref "webgpu.md" >}}).
- Package `metal` draws it with a compute kernel through
  [go-macos/metal](https://github.com/go-macos/metal).
- Package `vulkan` (Linux) draws it through
  [go-gfx/vulkan](https://github.com/go-gfx/vulkan); see [Vulkan](#vulkan).
- Package `d3d11` (Windows, opt-in) draws it through
  [go-mswin/d3d11](https://github.com/go-mswin/d3d11); see
  [Direct3D 11](#d3d11).

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

Recording and binning allocate nothing in steady state. Since go-macos/objc
v0.12.0's allocation-light message sends, a whole `Submit` costs 21
allocations.

`Submit`'s cost is almost all CPU-side packing: binning the list into 16 × 16
tiles and copying it into the slot. Encoding and Metal's commit add about
14 µs.
- **v0.6.1:** binning runs in plain loops rather than a callback per
  (instance, tile) pair.
- **v0.6.2:** binning writes straight into the slot's Metal buffers, so the
  indices are never copied.

Measured on an M4 Max, 8 interleaved rounds at a 1-minute load of at most 8,
each column an A/B run against the previous tag:

| | v0.6.0 | v0.6.1 | v0.6.2 |
|---|---:|---:|---:|
| `Bin`, 960 × 2834 widget frame | 148.0 µs | 48.3 µs | 48.3 µs |
| `Submit`, `WidgetFrame` at 1920 × 1080 | 164.9 µs | 80.4 µs | 57.3 µs |
| `Submit`, `WidgetFrame` at 1920 × 5668 | 782.9 µs | 364.7 µs | 242.2 µs |

## Presenting in a browser canvas {#present}

Since v0.5.0, `webgpu.Renderer.Present(surface)` copies the renderer's target
into the surface's current texture on the GPU, so the frame goes to the canvas
without coming back to Go memory. Call it after `Draw`; the browser shows the
canvas at the end of the task. The surface must be the renderer's size and be
configured with `webgpu.CanvasDescriptor()`:

```go
desc := gpu2dwebgpu.CanvasDescriptor() // the browser's preferred canvas format
s, err := device.NewSurfaceFromCanvas(offscreen, desc)
err = r.Draw(list, nil)
err = r.Present(s) // byte-exact in either format
```

Since v0.6.0, `CanvasDescriptor` follows `getPreferredCanvasFormat`:

- **bgra8unorm** (macOS, Windows): `Present` draws the target into it with
  `PresentShader`, a `textureLoad` at each pixel. `unorm8 → float → unorm8`
  gives back the same byte.
- **rgba8unorm** elsewhere, with `COPY_DST`: `Present` copies with
  `copyTextureToTexture`, which is byte-exact because it is the target's own
  format.
- **Anything else** is refused with `ErrSurface`.

The preferred format matters, because a canvas in another one costs an extra
copy in the browser's compositor. `internal/presentbench` measured it with a
real page drawing `WidgetFrame` at 4K, timed until the GPU finished, over 8
interleaved rounds per platform, with the adapter logged in each:

| 4K frame | rgba8unorm | bgra8unorm |
|---|---:|---:|
| Apple M4 Max, macOS (prefers bgra8unorm) | 1.053 ms | **0.792 ms** (−24.8 %) |
| AMD Radeon PRO W7700, Linux on Vulkan (prefers rgba8unorm) | **0.704 ms** | 1.083 ms (+53.7 %) |

The preferred format wins on both, although they are opposite formats. That
is why `CanvasDescriptor` asks the browser instead of fixing one.

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
| WebGPU, `New` (fixed-function) | identical on every channel | colour identical; alpha up to 4 apart |
| WebGPU, `NewCompute` (v0.7.0) | identical on every channel | identical on every channel |

The Metal kernel blends with the reference's integer arithmetic. WebGPU's
default renderer uses fixed-function blending, which rounds where
`PixelPainter` floors `a + dst·(255−a)/255`. A window surface is opaque, so
the difference does not show there.

### Compute mode {#compute}

Since v0.7.0, `webgpu.NewCompute` draws with `kernel.wgsl`: the whole raster
kernel as one WGSL compute shader, with the same integer blend as Metal. It
is byte-identical to `Rasterize` on both grounds, measured in Chromium on an
M4 Max and on the CI's SwiftShader lane.

It is also slower, which is why it is not the default. Measured with Draw
plus Present, 8 interleaved rounds at a 1-minute load of at most 8:

| frame | `New` | `NewCompute` |
|---|---:|---:|
| 1080p widget frame | 244.9 µs | 385.0 µs |
| 4K widget frame | 745.3 µs | 1015.4 µs |
| 960 × 2834 gallery | 290.6 µs | 444.3 µs |

The compute frame includes CPU binning in wasm. Choose it where a translucent
surface must match the CPU byte for byte. The same kernel is the source for
the Vulkan and Direct3D 11 back-ends below
([NATIVE.md](https://github.com/go-gfx/gpu2d/blob/main/docs/NATIVE.md)).

## One kernel, three shading languages {#kernel}

`kernel.wgsl` is the only place the raster arithmetic is written. The native
back-ends do not carry a hand-written copy; each file is generated from it by
[go-gfx/wgsl]({{< relref "wgsl.md" >}}), and a test fails on every platform
when the WGSL changes without the generated file being regenerated:

| back-end | generated file | emitter |
|---|---|---|
| Metal | `metal/kernel_gen.metal` | `msl` |
| Vulkan | `vulkan/kernel_gen.spv` | `spirv` |
| Direct3D 11 | `d3d11/kernel_gen.hlsl`, compiled to `kernel_gen.dxbc` | `hlsl` |

## Vulkan (Linux) {#vulkan}

```go
r, err := vulkan.New(w, h) // errors.Is(err, vulkan.ErrUnsupported) without a Vulkan loader
copy(r.Pixels(), ground)
err = r.Draw(list)         // Submit, then Wait; r.Pixels() holds the frame
```

**How it is chosen.** There is no automatic choice inside gpu2d: a program
calls `vulkan.New(w, h)` (or `NewWith` for validation or a vendor filter) the
way it calls `metal.New`. It has the same shape as `metal`: `Draw`, `Submit`
and `Wait` over two slots, `Pixels`, `Resize`, `Close`, `Name`.

It runs the kernel through
[go-gfx/vulkan](https://github.com/go-gfx/vulkan), a pure-Go binding that
loads `libvulkan.so.1` through purego: no cgo, 64-bit Linux, compute only. The
buffers are host-visible and coherent, so `List.Bin` bins straight into the
slot's buffer, as on Metal. Two frames are really in flight, ordered by a
memory barrier at the start of each frame's command buffer. There is no
presentation yet.

**The proof.** CI runs the package on lavapipe (Mesa's software Vulkan) with
the Khronos validation layer, failing on any message. `TestMatchesReference`
finds 0 differing channels, on an opaque and a translucent ground.

**On a real GPU.** Measured on an AMD Radeon PRO W7700 (RADV, Mesa 25.2.8),
`GOMAXPROCS=1`, abbench with 8 rounds, 1-minute load below 2:

| measure | 1080p | 4K |
|---|---:|---:|
| `Draw` | 0.524 ms | 1.984 ms |
| CPU `Rasterize`, one core | 27.13 ms | 110.0 ms |
| `Draw` plus read-back into `Pixels` | 4.57 ms | 16.4 ms |
| `Draw` plus `Sync` | 2.12 ms | 7.13 ms |

`Draw` is 52 times (1080p) and 55 times (4K) faster than `Rasterize` on one
core. When the frame is read back, as a program that needs the bytes must, it is
5.9 and 6.7 times faster. The read-back goes through a stage in memory that
go-gfx/vulkan prefers to be host-cached (preferably, not guaranteed); `Sync`
waits for every frame in flight and downloads the device's frame into `Pixels`.
`Pixels` downloads the same way, without a way to return the error, and also
marks the frame for re-upload on the next `Draw`; the benchmark copies the
bytes into a sink. That upload is why `Draw` plus read-back (4.57 ms at 1080p)
costs about twice `Draw` plus `Sync` (2.12 ms).
These are a different machine from the M4 Max table above; do not compare the
two tables.

{{< callout type="warning" >}}
lavapipe proves the bindings and the kernel; the table above is RADV on one
card. Other GPUs and drivers have not been measured. After a fence timeout the
renderer is lost and `Pixels` may return torn bytes; only `Close` it.
{{< /callout >}}

**Fallbacks.** Where there is no loader (and on every platform other than
64-bit Linux), `New` returns an error satisfying
`errors.Is(err, vulkan.ErrUnsupported)`, and the caller keeps the CPU
reference. A frame past a device limit (`maxStorageBufferRange`, guaranteed
only at 2^27 bytes, or `maxComputeWorkGroupCount`) is refused with
`gpu2d.ErrTooLarge` before anything is dispatched.

## Direct3D 11 (Windows, opt-in) {#d3d11}

```go
r, err := d3d11.New(w, h) // errors.Is(err, d3d11.ErrUnsupported) off Windows
copy(r.Pixels(), ground)
err = r.Draw(list)        // Submit, then Wait; r.Pixels() holds the frame
```

**How it is chosen.** Opt-in: a program asks for `d3d11.New(w, h)`, or
`NewWith` with `Options.Software` to ask for WARP. Nothing selects it for a
program that did not.

It runs the kernel through
[go-mswin/d3d11](https://github.com/go-mswin/d3d11) (pure Go, COM through
syscall, no cgo). The kernel is `d3d11/kernel_gen.hlsl`, generated from
`kernel.wgsl`, and `kernel_gen.dxbc` is that HLSL compiled by `D3DCompile` on
the Windows CI lane and committed with a record of the compiler and hashes:
nothing is compiled at run time.

Direct3D 11 has no persistently mapped, GPU-writable buffer, so `Pixels` is a
host copy and `Draw` pays two copies that Metal and Vulkan do not: an upload of
the target when nothing is in flight, and a read-back into `Pixels` when the
last frame ends. Frames queued with `Submit` stay on the device between them.
`Renderer.Stats` splits the wall time into packing, target upload, device time
and read-back.

**The proof.** CI runs the package on WARP, the software device of a
`windows-latest` runner, and finds parity with `Rasterize` there.

{{< callout type="warning" >}}
Only WARP has run it. Parity on WARP measures WARP's arithmetic, not any GPU,
and no Windows GPU has run the package. Any timing taken on WARP is a software
figure and says nothing about a GPU, so none is quoted here. A device that
disagrees is to be named by `TestMatchesReference`.
{{< /callout >}}

**Fallbacks.** Off Windows the constructors return `ErrUnsupported`; with no
device, `ErrNoDevice`. A caller falls back to the CPU. A frame past the
device's buffer limit (128 MiB on WARP, so a square frame above 5792 pixels)
or past 65535 thread groups per dimension is refused with `gpu2d.ErrTooLarge`.

How the native back-ends are presented, and the measured split of the Direct3D
11 copies, is in
[docs/NATIVE.md](https://github.com/go-gfx/gpu2d/blob/main/docs/NATIVE.md).

## Bounds {#bounds}

- Every coordinate lies within ±2^24, the range in which float32 holds every
  integer exactly.
- Translations and clips saturate, and ring counts are capped.
- Binning for the Metal kernel is limited by `MaxBinned`, and a list past it
  is refused before anything is allocated.
- `FuzzList` checks that nothing is recorded past these bounds and that
  nothing is written past the surface.

## One coverage, written in WGSL {#generated}

Since v0.5.2 the Metal kernel's `Inst`, `cornerCenter` and `coverage` are
not written by hand. They are generated from the WebGPU shader by
[wgsl]({{< relref "wgsl.md#msl" >}})'s `msl` package (`go generate ./metal`
writes `metal/coverage_gen.metal`), so the coverage arithmetic exists once,
in WGSL, next to the Go reference.

- **Byte-identical:** the kernel matches the CPU reference, with max |Δ| 0
  over 3,686,400 channels.
- **Drift guard:** `TestCoverageMetalIsGenerated` fails on every platform
  when the WGSL changes without the Metal file being regenerated.

The go-widgets `painter.Painter` adapter ([go-widgets/gpupainter](https://github.com/go-widgets/gpupainter))
and presentation are done: a `CAMetalLayer` in go-widgets/window v0.90.0,
and the browser canvas in go-widgets/webcanvas v0.9.0.

### Rounded corners are exact since v0.12.0 {#corners}

Before v0.12.0 the kernels disagreed with `Rasterize` on some pixels of a
rounded corner at some radii (counts below). Since v0.12.0 the coverage of a corner is decided in 32-bit integers from
the exact squared distance, in the Metal and Vulkan kernel, in the CPU
reference and in the WebGPU shader. `MaxRadius` is 16383.

Measured at every radius from 1 to 64, the number of channels that differ from
`Rasterize` is now 0 on Metal, on RADV and on lavapipe, where it was 366,
6,580 and 2,742 respectively. Those three back-ends are the ones measured on
that corner-heavy corpus. Direct3D 11 on WARP and WebGPU in the browser ran the
random scenes only, not the corner corpus, so this page makes no radius-by-radius
claim for them.

The design is in
[docs/DESIGN.md](https://github.com/go-gfx/gpu2d/blob/main/docs/DESIGN.md).

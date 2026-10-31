---
title: "WebGPU (browser)"
weight: 25
---

# WebGPU for Go in the browser {#webgpu}

[`github.com/go-gfx/webgpu`](https://github.com/go-gfx/webgpu) is a typed Go
API over the browser's [WebGPU](https://www.w3.org/TR/webgpu/), for any Go
program built with `GOOS=js GOARCH=wasm` that runs in a window or a worker. It
is the shared layer under the go-wasmcontainers GPU host (`wasi:webgpu`,
`wasi-nn`) and the browser game renderers (go-quake1).

{{< callout type="info" >}}
**Released: v0.1.0.** Install it with `go get github.com/go-gfx/webgpu@v0.1.0`.
Before 1.0, a minor version may change the API. The go-wasmcontainers GPU
host runs on this release.
{{< /callout >}}

```go
import "github.com/go-gfx/webgpu"

g, err := webgpu.Navigator()                         // navigator.gpu
a, err := g.RequestAdapter(ctx, nil)                 // refuses a fallback adapter by default
d, err := a.RequestDevice(ctx, &webgpu.DeviceDescriptor{Quota: q})
b, err := d.CreateBuffer(webgpu.BufferDescriptor{Size: 64, Usage: webgpu.BufferUsageVertex | webgpu.BufferUsageCopyDst})
err = d.Queue().WriteBuffer(b, 0, vertices)          // range checked at the call
```

## What it covers {#what-it-covers}

- Adapters and devices (device loss as a channel), buffers and mapping,
  textures of every format of the specification, views, samplers,
  `WriteTexture` for dirty rectangles, texture and buffer copies.
- Bind group and pipeline layouts, bind groups, render and compute pipelines
  (sync and async), depth-stencil, blending, multisampling.
- Render passes with indexed, instanced and indirect draws, render bundles,
  compute passes, error scopes, uncaptured errors.
- Surfaces: an OffscreenCanvas with a WebGPU context, frames out as
  `ImageBitmap`s.

## Errors at the call {#errors-at-the-call}

WebGPU reports most mistakes asynchronously, on the device. The package checks
first and returns a `*webgpu.BoundsError` from the call: a write past a buffer,
a texture region outside its mip level, a linear layout that does not fit its
data (the specification's "validating linear texture data"), a missing usage.
The arithmetic is overflow checked, tested on 32-bit targets, and the copy
check is fuzzed against an unbounded-integer oracle.

## Quotas {#quotas}

A `Quota` bounds GPU memory (texture sizes are computed by the package),
pipelines, WGSL size, surface size, and the optional features and limits a
device may require (`timestamp-query` included). It is charged before the
browser is called and refunded on `Destroy`, `Release` or device loss; the
go-wasmcontainers playground gives one to each untrusted container.

## Batched encoding {#batched-encoding}

A `syscall/js` call costs about 200 ns, ten times the WebGPU call it carries.
A `Recorder` records a whole submit in Go and replays it with one call into
`replay.js`, a plain same-origin script (it runs under
`script-src 'self'`):

```go
webgpu.InstallReplay(ctx, "/webgpu-replay.js")
rec := device.NewRecorder()
p := rec.BeginRenderPass(passDesc)
p.SetPipeline(pipe)
for _, s := range surfaces {
	p.SetBindGroup(0, s.group)
	p.DrawIndexed(s.count, 1, s.first, 0, 0)
}
p.End()
seq, err := rec.Submit()
err = device.WaitSubmitted(ctx, seq)
```

Every command is checked as in the direct API. `replay.js` and its Go mirror
read the same format with the same checks; the Go reader is fuzzed, and a
browser test holds both to the same verdict on thousands of mutated streams.
Submissions are numbered on a timeline (package `timeline`): completing one
completes every earlier one, the fence model of virtio-gpu.

The stream format is exported as package `wire`, which has no GOOS
constraint. It contains the opcodes, `wire.Decode` (the Go reader) and
`wire.StreamVersion`. It also has `wire.Remap`, which rewrites every
object id through a mapping and refuses any id the mapping does not
grant. A stream recorded in one place can then be checked and replayed
against another object table without a second implementation of the
format. `copyTextureToTexture` is also in the batch, as
`Recorder.CopyTextureToTexture`. Adding it moved `StreamVersion` to 2.

Work recorded somewhere else can now reach a device in the browser:

- **Objects.** `EncodeDescriptor` turns a creation descriptor into bytes
  that name every referenced object by id in a header
  (`wire.RemapDescriptor` translates those ids). `DecodeDescriptor`
  rebuilds the descriptor with the receiver's objects. It refuses unknown
  fields, a wrong reference count and references of the wrong type.
- **Streams.** `Device.Replay(words, data, refs...)` submits a stream
  that names only the granted objects (`ReplayID` gives their ids). The
  stream is checked with `wire.Decode` first.
- **The remote device.** Outside the browser, `NewRemoteDevice(sink, …)`
  implements the creation calls, `Queue.WriteBuffer` and
  `Recorder.Submit` by sending descriptors and streams to a
  `RemoteSink`. It uses the browser path's checks and quotas. Readback,
  `WriteTexture` and encoders are not carried yet.

The first user is go-virtio's
[webgpu-batch](https://github.com/go-virtio/gpu/blob/main/docs/WEBGPU.md)
virtio-gpu context. A VM guest's go-gfx/webgpu program sends its work to
the [gpudev](https://github.com/go-virtio/gpudev) host, which replays it
on the browser's GPU. That path is validated in Chromium on Apple Metal.

Measured on an Apple M4 Max, Chromium 143, Metal, CPU time per frame. The
machine was loaded during these runs (1-minute load 10 to 16), so treat the
numbers as indicative:

| draws | direct calls | `Recorder` | the same frame in JavaScript |
|---:|---:|---:|---:|
| 1000 | 491 µs | 85 µs | 32 µs |
| 5000 | 2320 µs | 380 µs | 138 µs |

That is about six times faster than direct calls, against the design's
estimate of fifteen. The estimate came from a prototype that recorded with no
checks and no object table.

## Surfaces {#surfaces}

```go
if err := webgpu.CanvasSupport(); err != nil {
	// errors.Is(err, webgpu.ErrNoCanvasContext): fall back to another renderer
}
s, err := device.NewSurface(webgpu.SurfaceDescriptor{Width: 640, Height: 480})
tex, err := s.CurrentTexture()
bitmap, err := s.Present() // ImageBitmap, moved out without a copy
```

MDN's compatibility data lists the WebGPU context of `OffscreenCanvas` in
Chrome 113 (every platform from 144), Safari 26, and Firefox 141 on Windows
only. Elsewhere, `CanvasSupport` is the signal to fall back to another
renderer.

## Tests {#tests}

The platform-neutral code (descriptors, validation, quotas, the recorder) has
every block covered on Linux, macOS, Windows and linux/386. The calls run in a
real browser worker through `go test -exec` and Playwright. On macOS a GPU is
required and a fallback adapter fails the run; Linux runs on SwiftShader. Every
frame is checked pixel by pixel.

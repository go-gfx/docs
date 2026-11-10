---
title: "WGSL front end"
weight: 26
---

# WGSL front end {#wgsl}

[`github.com/go-gfx/wgsl`](https://github.com/go-gfx/wgsl) is a pure-Go front
end for the [WebGPU Shading Language](https://www.w3.org/TR/WGSL/): lexer,
parser, AST, printer, name resolution and type checking. It implements the
Candidate Recommendation Draft of 26 October 2026, with no cgo and only the
standard library.

```go
m, err := wgsl.Parse(src)    // *wgsl.Error with line and column on failure
text := wgsl.Print(m)        // canonical WGSL; parses back to an equal AST
info, errs := wgsl.Check(m)  // types, resolved names, struct layouts, entry points
err = wgsl.Validate(src)     // Parse + Check
```

```sh
wgsl check shader.wgsl       # exit 1 on the first invalid file
wgsl fmt -w shader.wgsl
```

## Why

* **Silent failures.** A guest shader that uses a reserved word (`meta`) fails
  inside the browser without a usable message. `Validate` rejects it at the
  host, with the position.
* **Rewrites.** Parse, change the AST, print valid WGSL again. This is the
  base for host-side limits such as bounding loop iterations.
* **Other back ends.** `Info` carries types, resolved declarations, struct
  layouts and entry points, so a WGSL to MSL emitter can be built on it.

## Loop budgets {#loop-budget}

`transform.LoopBudget` (and `wgsl transform`) rewrites every `loop`, `for` and
`while` so that it counts iterations against a budget taken from an `override`
or a uniform `u32`. Past the budget the loop sets an `atomic<u32>` flag in a
storage buffer and breaks, so the host can tell a truncated run from a complete
one. The rewrite works on the parse tree only, never on the checker's verdict.
It was run on an Apple GPU through Chrome WebGPU: a 10^8-iteration loop is cut
at the budget with the flag set, and a short loop is unchanged.

{{< callout type="warning" >}}
The browser remains the validation authority. `wgsl check` is not yet a
security boundary: against the CTS it still wrongly accepts 88 invalid cases
(uniformity analysis is missing, so its 91,497 cases are skipped) and wrongly
rejects none of the 130,057 valid ones.
{{< /callout >}}

## Metal Shading Language {#msl}

Since v0.5.0, package `msl` emits MSL for named functions of a checked
module, with every function they call and the structures they use:

```go
m, _ := wgsl.Parse(src)
info, _ := wgsl.Check(m)
out, err := msl.Functions(m, info, "coverage")
```

**Scope.** It covers plain functions over `bool`, `i32`, `u32` and `f32`,
their vectors, and structures of those. Anything else is refused with its
position, never emitted approximately: entry points, module-scope variables,
loops, arrays, textures.

**WGSL's semantics, not Metal's defaults.** Each rule is sourced in the
repository's [docs/MSL.md](https://github.com/go-gfx/wgsl/blob/main/docs/MSL.md)
from the two specifications:
- fast math is off for the file;
- library calls are the `precise::` ones;
- signed `+`, `-`, `*` and `<<` wrap;
- `/` and `%` give WGSL's results for a zero divisor and `MIN / -1`;
- shift counts are taken modulo 32, and `>>` on `i32` is arithmetic;
- `bitcast` keeps the bits.

**Checked on the GPU.** A test runs the emitted code on a Mac's GPU against
WGSL's definitions computed in Go:
- 13 integer operations over 361 edge pairs;
- `clamp`, `length`, `select`, `abs`, `min` and `max`, with the subnormal
  flush-to-zero WGSL allows.

With Metal's plain division, `1 / 0` comes back −1, where WGSL defines 1.
go-gfx/gpu2d's Metal kernel generates its coverage functions this way.

## SPIR-V {#spirv}

Since v0.9.0, `spirv.Module(module, info, spirv.Options{}, entries...)` emits
a SPIR-V 1.5 module (the Vulkan 1.2 environment) for compute entry points:
storage and uniform buffers of group 0, scalar and vector arithmetic, and
`if`, `for`, `while` and `loop` with `break` and `continue`. It is the subset a
fill kernel needs; the rest is refused as a `*spirv.Error` with its position.
[docs/SPIRV.md](https://github.com/go-gfx/wgsl/blob/main/docs/SPIRV.md) lists
both, with the specification sections used.

- The numbers come from Khronos's `spirv.core.grammar.json`, through a
  generator.
- A decoder checks every module on every platform (logical layout, structured
  control flow, the entry-point interface) and must refuse ten mutated ones.
- `spirv-val` validates the output in CI. It has not been run on a GPU by this
  package; go-gfx/gpu2d's `vulkan` back-end runs it on lavapipe.

## HLSL {#hlsl}

Since v0.9.0, `hlsl.Module(module, info, entries...)` emits High-Level Shading
Language for Shader Model 5.0, to be compiled for `cs_5_0` with `D3DCompile`
and `D3DCOMPILE_IEEE_STRICTNESS`: compute entry points, `read` storage arrays
as `StructuredBuffer` at `t<binding>`, `read_write` ones as
`RWStructuredBuffer` at `u<binding>`, a uniform as a `cbuffer`, WGSL's integer
and float rules, and every float operation kept `precise` (no fused `mad`). It
takes the whole of gpu2d's `kernel.wgsl`; the rest is refused as a
`*hlsl.Error` with its position
([docs/HLSL.md](https://github.com/go-gfx/wgsl/blob/main/docs/HLSL.md)).

- The emitted kernel and 24 refusals are checked on every platform.
- On Direct3D 11 WARP (a nested module on `windows-latest`, through
  go-mswin/d3d11) every emitted file is compiled by `D3DCompile` and the
  integer, float and conversion edge cases are held to WGSL's definition; two
  break checks show the tests can fail. No GPU has run it.

## Bounds

`Limits` caps the source size, the token count and the nesting depth. Deeply
nested input is refused with a positioned error; it does not overflow the
stack.

## Conformance

The checker runs the W3C gpuweb CTS validation cases (`webgpu:shader,validation,*`),
executed under Node with the GPU stubbed so that the parametrised tests expand
to 384,053 distinct (source, verdict) pairs. See
[CONFORMANCE.md](https://github.com/go-gfx/wgsl/blob/main/docs/CONFORMANCE.md)
for the per-directory table.

Not yet implemented: uniformity analysis (those cases are counted as skipped),
alias analysis of pointer arguments, the `buffer_view` and `swizzle_assignment`
extensions.

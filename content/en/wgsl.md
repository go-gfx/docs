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

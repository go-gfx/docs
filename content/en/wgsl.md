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

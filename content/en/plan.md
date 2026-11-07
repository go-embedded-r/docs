---
title: "Plan"
weight: 10
description: "The milestones: M0, M1 and M2 are done (v0.7.0), M3 (base R) is in progress, M4 to M8 are planned."
---

<!-- From the README and docs/m3-plan.md of go-embedded-r/r on main (status table). Change those first, then here. -->

**Status: in development.** M0, M1 and M2 are done, released as
[v0.7.0](https://github.com/go-embedded-r/r/releases/tag/v0.7.0) of
[go-embedded-r/r](https://github.com/go-embedded-r/r); M3 started in v0.8.0 and the current release is
[v0.8.1](https://github.com/go-embedded-r/r/releases/tag/v0.8.1) (slices
1 to 3 were v0.3.0 to v0.5.0, whose regex v0.6.0 replaced with a port of TRE; slice 2 was v0.3.0; v0.4.0 makes the `Eval`
limits bounded by default, see [Eval API]({{< relref "eval-api" >}}); the parser was
[v0.1.0](https://github.com/go-embedded-r/r/releases/tag/v0.1.0), deparse
[v0.2.0](https://github.com/go-embedded-r/r/releases/tag/v0.2.0)). The plan is
[`bib/r.md`](https://github.com/go-scikit/roadmap/blob/main/bib/r.md) of
[go-scikit/roadmap](https://github.com/go-scikit/roadmap). Every stage is
checked against the pinned GNU R 4.6.2, never against expectations written by
hand.

| Milestone | Contents | State |
| --- | --- | --- |
| M0 oracle harness | `internal/oracle`, `cmd/roracle`, `rds` (serialization reader and writer), golden files | **done** |
| M1 parser | `syntax`: lexer, parser, AST with source map, tokens, R's parse errors, `deparse`, `TopLevelNames` | **done** |
| M2 evaluator | `eval`: values, environments, promises, closures, control flow, replacement functions, subsetting, attributes, S3 dispatch, conditions, printing, the `Eval` API with limits; 175 of the 210 primitives (the other 35 refused with an explanatory error) and 110 base closures, environments as values, the apply family, character functions, regular expressions (a port of TRE, [go-embedded-r/tre](https://github.com/go-embedded-r/tre)), the math group, factors, matching, collation | **done, slices 1 to 4** (v0.7.0); see [Evaluator]({{< relref "evaluator" >}}), [Regular expressions]({{< relref "regex" >}}) and [Eval API]({{< relref "eval-api" >}}) |
| M3 base R | the rest of what `library(base)` exports, rewritten clean room; 22.2 % of base's 1 362 exports agree with R 4.6.2 at v0.8.0 (19.8 % at v0.7.0, 14.4 % at v0.6.0) | **in progress**; see [M3 plan / base coverage]({{< relref "m3-plan" >}}) |
| M4 to M8 | stats, graphics, notebook bridge, WebAssembly playground, CRAN ports | **planned** |

Nothing in the M4 to M8 row exists yet. The README's table groups M3 to M8 as
"base, stats, regex, graphics, notebook bridge, wasm playground, CRAN ports"; the
split of M4 onwards is not assigned subject by subject there, and this page does
not assign it either.

## Known gaps of M1

- the effect of `#line` directives on the line numbers of later errors;
- `parse(file=)` encoding options;
- an invalid UTF-8 input is parsed as R's lexer does, and for such text R
  cannot produce parse data, so its tokens are not compared;
- numeric constants that deparse differently on some platforms: see
  [Deparse]({{< relref "deparse" >}}).

See [The parser]({{< relref "parser" >}}) for the parity measurements.

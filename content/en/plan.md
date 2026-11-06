---
title: "Plan"
weight: 10
description: "The milestones: M0 and M1 are done, M2 (evaluator) has slices 1 to 3 done (v0.6.0 is current), M3 to M8 are planned."
---

<!-- From the README of go-embedded-r/r on main (status table). Change the repository's README first, then here. -->

**Status: in development.** M0 and M1 are done, and slices 1 to 3 of M2 are
done, released as
[v0.6.0](https://github.com/go-embedded-r/r/releases/tag/v0.6.0) of
[go-embedded-r/r](https://github.com/go-embedded-r/r) (slice 3 was v0.5.0, whose regex v0.6.0 replaces with a port of TRE; slice 2 was v0.3.0; v0.4.0 makes the `Eval`
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
| M2 evaluator | `eval`: values, environments, promises, closures, control flow, replacement functions, subsetting, attributes, S3 dispatch, conditions, printing, the `Eval` API with limits; 102 of the 210 primitives and 94 base closures, environments as values, the apply family, character functions and regular expressions (a port of TRE, [go-embedded-r/tre](https://github.com/go-embedded-r/tre)) | **slices 1 to 3 done** (v0.6.0); see [Evaluator]({{< relref "evaluator" >}}), [Regular expressions]({{< relref "regex" >}}) and [Eval API]({{< relref "eval-api" >}}) |
| M3 to M8 | base, stats, regex, graphics, notebook bridge, WebAssembly playground, CRAN ports | **planned** |

Nothing in the M3 to M8 row exists yet. The README names the subjects and does
not assign them to milestone numbers one by one, so neither does this page.

## Known gaps of M1

- the effect of `#line` directives on the line numbers of later errors;
- `parse(file=)` encoding options;
- an invalid UTF-8 input is parsed as R's lexer does, and for such text R
  cannot produce parse data, so its tokens are not compared;
- numeric constants that deparse differently on some platforms: see
  [Deparse]({{< relref "deparse" >}}).

See [The parser]({{< relref "parser" >}}) for the parity measurements.

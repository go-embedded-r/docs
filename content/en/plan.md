---
title: "Plan"
weight: 10
description: "The milestones: M0 and M1 (with deparse) are done in v0.2.0, M2 (evaluator) is in progress with slice 1 on main, M3 to M8 are planned."
---

<!-- From the README of go-embedded-r/r on main (status table). Change the repository's README first, then here. -->

**Status: in development.** M0 and M1 are done, released as
[v0.2.0](https://github.com/go-embedded-r/r/releases/tag/v0.2.0) of
[go-embedded-r/r](https://github.com/go-embedded-r/r) (the parser was
[v0.1.0](https://github.com/go-embedded-r/r/releases/tag/v0.1.0)). M2 is in
progress: its first slice is on `main`, not tagged. The plan is
[`bib/r.md`](https://github.com/go-scikit/roadmap/blob/main/bib/r.md) of
[go-scikit/roadmap](https://github.com/go-scikit/roadmap). Every stage is
checked against the pinned GNU R 4.6.2, never against expectations written by
hand.

| Milestone | Contents | State |
| --- | --- | --- |
| M0 oracle harness | `internal/oracle`, `cmd/roracle`, `rds` (serialization reader and writer), golden files | **done** |
| M1 parser | `syntax`: lexer, parser, AST with source map, tokens, R's parse errors, `deparse`, `TopLevelNames` | **done** (v0.2.0) |
| M2 evaluator core | `eval`: values, environments, promises, closures with R's argument matching, control flow, 71 of the 210 primitives (first slice) | **in progress**: slice 1 done, on `main`, not tagged; see [Evaluator]({{< relref "evaluator" >}}) |
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

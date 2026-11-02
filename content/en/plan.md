---
title: "Plan"
weight: 10
description: "The milestones: M0 (oracle harness) and M1 (parser) are done in v0.1.0; M2 to M8 are planned."
---

<!-- From the README of go-embedded-r/r at v0.1.0 (status table). Change the repository's README first, then here. -->

**Status: in development.** M0 and M1 are done, released as
[v0.1.0](https://github.com/go-embedded-r/r/releases/tag/v0.1.0) (2026-11-02)
of [go-embedded-r/r](https://github.com/go-embedded-r/r). The plan is
[`bib/r.md`](https://github.com/go-scikit/roadmap/blob/main/bib/r.md) of
[go-scikit/roadmap](https://github.com/go-scikit/roadmap). Every stage is
checked against the pinned GNU R 4.6.2, never against expectations written by
hand.

| Milestone | Contents | State |
| --- | --- | --- |
| M0 oracle harness | `internal/oracle`, `cmd/roracle`, `rds` (serialization reader and writer), golden files | **done** |
| M1 parser | `syntax`: lexer, parser, AST with source map, tokens, R's parse errors, `TopLevelNames` | **done** (no Go `deparse` printer yet) |
| M2 to M8 | evaluator, base, stats, regex, graphics, notebook bridge, WebAssembly playground, CRAN ports | **planned** |

Nothing in the M2 to M8 row exists yet. The README names the subjects and does
not assign them to milestone numbers one by one, so neither does this page.

## Known gaps of M1

- the effect of `#line` directives on the line numbers of later errors;
- a Go `deparse` printer: the tree is compared structurally;
- `parse(file=)` encoding options;
- an invalid UTF-8 input is parsed as R's lexer does, and for such text R
  cannot produce parse data, so its tokens are not compared.

See [The parser]({{< relref "parser" >}}) for the parity measurements.

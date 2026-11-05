---
title: "go-embedded-r documentation"
linkTitle: "Home"
type: docs
cascade:
  type: docs
description: "A pure-Go, embeddable R interpreter, in development: v0.5.0 completes M0 (oracle harness against GNU R 4.6.2), M1 (parser and deparse) and slices 1 to 3 of M2 (evaluator); its regex is an approximation, not TRE; M3 to M8 are planned."
---

**go-embedded-r is in development.
[v0.5.0](https://github.com/go-embedded-r/r/releases/tag/v0.5.0) (2026-11-05)
of [go-embedded-r/r](https://github.com/go-embedded-r/r) completes milestone M0
(the oracle harness, against the pinned GNU R 4.6.2), milestone M1 (the
parser and `deparse`, package `syntax`), which agrees with R's tree, tokens,
error text and deparse output on 842 of 842 files of R's base packages, and
slices 1 to 3 of milestone M2.** The evaluator (package `eval`) has 102 of R's
210 primitives and 94 base closures, with replacement, subsetting, attributes,
S3 dispatch, conditions, `print`, environments as values, the apply family,
string functions and regular expressions; 1 035 of 1 035 goldens match R 4.6.2.
**Its regular expressions are an approximation on Go's `regexp`, not TRE**: see
[Regular expressions]({{< relref "regex" >}}). M2 is not finished. Base, stats,
the rest of regex,
graphics, the notebook bridge, the WebAssembly playground and CRAN ports
(M3 to M8) are planned and not started. The interpreter is therefore not
finished.

The statements on these pages come from the README of go-embedded-r/r on
main and its `docs/m2-design.md`, and from the plan the README links,
[`bib/r.md`](https://github.com/go-scikit/roadmap/blob/main/bib/r.md) of the
go-scikit roadmap.

## What it will be

An R interpreter written in pure Go (`CGO_ENABLED=0`), meant to be embedded in
Go programs and to run in the browser through WebAssembly. Every stage is
checked against the reference implementation, the pinned **GNU R 4.6.2** (CRAN
tarball, SVN revision 90784), never against expectations written by hand.

## What it is for

The `r` cells of [go-notebook](https://go-notebook.github.io/), the way
[julia](https://github.com/go-embedded-julia/julia) backs its Julia cells and
[rbgo](https://github.com/go-embedded-ruby/ruby) its Ruby cells.

## Pages

| Page | What it holds |
| --- | --- |
| [Plan]({{< relref "plan" >}}) | the milestones: M0 and M1 done, M2 slices 1 to 3 done, M3 to M8 planned |
| [The parser]({{< relref "parser" >}}) | package `syntax`: the API, the tree, the errors, the bounds, and the parity table with R 4.6.2 |
| [Deparse]({{< relref "deparse" >}}) | `syntax.Deparse`: parity with R 4.6.2 and the platform divergences |
| [Evaluator]({{< relref "evaluator" >}}) | M2, package `eval`: the design choices, slices 1 to 3 (102 of 210 primitives and 94 base closures), judged by the oracle |
| [Regular expressions]({{< relref "regex" >}}) | v0.5.0: an approximation on Go's `regexp`, not TRE; what differs from R and what is refused |
| [Eval API]({{< relref "eval-api" >}}) | `NewSession`, `Options` and its default limits (bounded since v0.4.0), `Eval`, `Result` and `Error` |
| [The oracle harness]({{< relref "oracle" >}}) | `internal/oracle`, `rds`, `cmd/roracle`, the goldens, and how to get the oracle |
| [Licences]({{< relref "licences" >}}) | BSD-3-Clause, and what "clean room" means here |

## Repositories

| Repo | What it is |
| --- | --- |
| [r](https://github.com/go-embedded-r/r) | the interpreter (source) |
| [docs](https://github.com/go-embedded-r/docs) | this site |
| [go-embedded-r.github.io](https://github.com/go-embedded-r/go-embedded-r.github.io) | the landing page |

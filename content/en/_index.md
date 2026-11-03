---
title: "go-embedded-r documentation"
linkTitle: "Home"
type: docs
cascade:
  type: docs
description: "A pure-Go, embeddable R interpreter, in development: v0.2.0 completes M0 (oracle harness against GNU R 4.6.2) and M1 (parser and deparse); M2 (evaluator) is in progress, slice 1 on main; M3 to M8 are planned."
---

**go-embedded-r is in development.
[v0.2.0](https://github.com/go-embedded-r/r/releases/tag/v0.2.0) (2026-11-03)
of [go-embedded-r/r](https://github.com/go-embedded-r/r) completes milestone M0
(the oracle harness, against the pinned GNU R 4.6.2) and milestone M1 (the
parser and `deparse`, package `syntax`), which agrees with R's tree, tokens,
error text and deparse output on 842 of 842 files of R's base packages.**
Milestone M2, the evaluator (package `eval`), is in progress: its first slice,
71 of R's 210 primitives, is on `main` and not tagged yet. Base, stats, regex,
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
| [Plan]({{< relref "plan" >}}) | the milestones: M0 and M1 done, M2 in progress, M3 to M8 planned |
| [The parser]({{< relref "parser" >}}) | package `syntax`: the API, the tree, the errors, the bounds, and the parity table with R 4.6.2 |
| [Deparse]({{< relref "deparse" >}}) | `syntax.Deparse`: parity with R 4.6.2 and the platform divergences |
| [Evaluator (in progress)]({{< relref "evaluator" >}}) | M2, package `eval`: the design choices and slice 1, 71 of 210 primitives, judged by the oracle |
| [The oracle harness]({{< relref "oracle" >}}) | `internal/oracle`, `rds`, `cmd/roracle`, the goldens, and how to get the oracle |
| [Licences]({{< relref "licences" >}}) | BSD-3-Clause, and what "clean room" means here |

## Repositories

| Repo | What it is |
| --- | --- |
| [r](https://github.com/go-embedded-r/r) | the interpreter (source) |
| [docs](https://github.com/go-embedded-r/docs) | this site |
| [go-embedded-r.github.io](https://github.com/go-embedded-r/go-embedded-r.github.io) | the landing page |

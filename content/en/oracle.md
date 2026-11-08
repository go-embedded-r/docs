---
title: "The oracle harness"
weight: 30
description: "M0: internal/oracle, rds, cmd/roracle and the golden files, and how to install the pinned R 4.6.2."
---

<!-- From the README of go-embedded-r/r at v0.1.0, sections "M0: what exists", "Getting the oracle" and "Layout". Change the repository's README first, then here. -->

## What exists

- **`internal/oracle`** starts one `Rscript --vanilla` process (the pinned
  release; any other version is refused at the greeting) and talks to it over
  stdin/stdout. Requests are R's own serialization format, written by the Go
  `rds` package; replies are R serializations, one hex line each, so no JSON
  package is needed on the R side. Operations: `eval`, `deparse`, `parsedata`
  (`utils::getParseData`), `describe`, `reserialize`. Output, warnings and the
  condition class and message are captured and returned; each request gets a
  fresh environment; a per-call timeout kills R; `R_MAX_VSIZE` bounds its
  vector heap. The test suite starts a fake R from its own test binary, so the
  protocol and every failure mode are tested without R.
- **`rds`** reads and writes R's serialization format (XDR, versions 2 and 3,
  gzip or bzip2 compressed): vectors of every atomic type, lists, pairlists,
  calls, symbols, closures, environments (with sharing and cycles), namespace
  and package references, built-ins, external pointers, S4 objects and ALTREP
  objects (kept in serialized form; `Altrep.Materialize` expands the compact
  sequences, wrappers and deferred strings). A decoded value encodes back to
  the **same bytes**: attributes in order, the object and general-purpose
  bits, string encoding flags, NaN payloads (`NA_real_` is a NaN). Byte-code
  and promises are refused. The decoder is built for untrusted input: size
  limit after decompression (decompression bombs are refused), nesting limit,
  every claimed length checked against the bytes left before allocating. It is
  fuzzed (`FuzzDecode`, corpus in `rds/testdata/fuzz`). Since go-embedded-r/r
  [pull request 25](https://github.com/go-embedded-r/r/pull/25), which
  **shipped in v0.10.0** (the tag at c1e0bd8 came after pull requests 25 and 26
  were merged), an ALTREP object found in a **version-2**
  stream is expanded, as R's reader does (its `ReadItem` has no version test,
  while `WriteItem` emits ALTREP only for version 3 or later), so the decoded tree
  is always writable by the version-2 encoder; an unknown ALTREP class is a decode
  error, and the expansion is bounded by a cumulative budget of `MaxBytes/8`
  elements. Version 3 is unchanged.
- **`cmd/roracle`** regenerates the golden files (`roracle regen`), prints the
  identity of the R that answers (`roracle version`) and evaluates a snippet
  (`roracle eval 'expr'`).
- **Goldens** (`testdata/golden`): `deparse.jsonl` is `parse` + `deparse` of
  the snippets in `testdata/snippets.R` (the input of the M1 parser), and
  `rds.jsonl` is `serialize()` of the expressions of `testdata/values.R`. Both
  inputs are written for this repository. The tests replay the goldens without
  R: every value R serialized decodes, agrees with R's `typeof` and `length`,
  and re-encodes to R's exact bytes; values built in Go encode to the bytes R
  writes. With R, the live tests check that R reads Go's bytes back unchanged
  and that regenerating the goldens reproduces the committed files.

## Getting the oracle

The tests that need R skip when `Rscript` is not found. To run them, install
the pinned release, which conda-forge publishes for macOS arm64 and Linux
x86-64, and point `RGO_RSCRIPT` at it:

```sh
micromamba create -y -p ~/r-4.6.2 -c conda-forge --override-channels r-base=4.6.2
export RGO_RSCRIPT=~/r-4.6.2/bin/Rscript
RGO_REQUIRE_ORACLE=1 go test ./...      # a missing R is a failure, not a skip
go run ./cmd/roracle regen              # rewrite testdata/golden
```

`RGO_REQUIRE_ORACLE=1` turns the skips into failures (CI sets it).

## Layout

| Path | Role |
| --- | --- |
| `rds/` | R serialization reader and writer |
| `internal/oracle/` | the driver of the reference R, and the R side of the protocol (`oracle.R`) |
| `internal/gen/` | golden-file generators and readers |
| `cmd/roracle/` | the command that regenerates the goldens |
| `testdata/` | inputs written for this repository, and `golden/` |

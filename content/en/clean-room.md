---
title: "Clean room"
weight: 35
description: "v0.12.2: how base R's R code is rewritten and not copied, the similarity gate (cmd/cleanroom, winnowing), its thresholds and audit, the oracle guard and the provenance record. The gate is a floor, not proof."
---

<!-- From the README of go-embedded-r/r on main (v0.12.2), section "Clean-room process: the similarity gate", its NOTICE (rules 5 to 7) and the body of pull request 30. Change those first, then here. -->

go-embedded-r/r is BSD-3-Clause and the R code of base R is GPL-2-or-later, so
the rule is: **base R's R code is rewritten, never copied.** A rule that depends
on what a contributor (human or model) remembers is not enough. In v0.12.0,
`eval/baselib/datetime.R` was written from the help pages but partly from memory of
R's code, and came out close to R's text. It was rewritten in
[v0.12.2](https://github.com/go-embedded-r/r/releases/tag/v0.12.2)
([pull request 30](https://github.com/go-embedded-r/r/pull/30)), and
`cmd/cleanroom` now checks every file of `eval/baselib` against the R library code
of R 4.6.2.

**The gate is a floor, not proof.** It bounds accidents and careless
restructuring. A short function has one natural spelling, so a tiny file can be a
copy and pass, and the checker cannot see a rewrite from memory that happens to
differ. The provenance record below is the other half, and the gate does not
replace the rule.

## The similarity checker

- **Method.** Our own lexer (package `syntax`) cuts both files into tokens,
  comments dropped. K-grams of tokens are hashed and thinned by winnowing
  (Schleimer, Wilkerson and Aiken, *Winnowing: Local Algorithms for Document
  Fingerprinting*, SIGMOD 2003: the minimum hash of each window, the rightmost on a
  tie; any shared run of `k+w-1` tokens shares a fingerprint, and no run shorter
  than `k` can). Parameters: `k = 8`, `w = 4`.
- **Two views.** Identifiers kept (*exact*), and every identifier replaced by one
  placeholder (*normalised*, which sees a renamed copy). The score is containment:
  the share of our fingerprints found in one R file. The longest common token run
  is reported with `file:line` on both sides. Runs count code tokens only
  (literals and commas do not, so a table of month names is not a copy).
- **Formal argument lists are excluded.** `function(` up to the matching `)`,
  defaults included, is not fingerprinted in either view: a help page fixes the
  signature, so independent code must repeat it.
- **R's sources are read, never stored.** The checker reads `src/library/*/R/*.R`
  of an R 4.6.2 tree at check time (`RGO_RSOURCE`, or `--rsrc`) and prints only
  references (`file:line`), never R's text.
- **The gate runs in CI.** The oracle lane downloads the tarball by SHA-256 and
  runs the gate there (`RGO_REQUIRE_RSOURCE=1`: the gate must run, not skip). It
  also runs as a Go test, which skips without R's sources.

## Thresholds, and how they were calibrated

Measured on code known to be independent of R's library code, at `k = 8, w = 4`
(files of at least 200 tokens):

| Independent code | Exact containment | Normalised containment | Longest exact run | Longest normalised run |
| --- | --- | --- | --- | --- |
| our own test inputs (`testdata/*.R`, `testdata/eval/*.R`; 25 files), max | 0.04 | 0.07 | 10 | 29 |
| R's own tests and demos against its library code (144 files), p95 / p99 / max | 0.03 / 0.19 / 0.37 | 0.31 / 0.44 / 0.52 | 14 / 31 / 46 | 29 / 48 / 78 |

The gate sits above the p99 of the second row:

| Gate | Value |
| --- | --- |
| exact containment (files of at least 200 tokens) | **0.20** |
| normalised containment (files of at least 200 tokens) | **0.60** |
| longest exact run | **40** code tokens |
| longest normalised run | **60** code tokens |
| files of 30 to 199 tokens | **4** exact fingerprints shared with one R file (independent small files: at most 2 of 32) |

A file of 100 tokens has only a dozen fingerprints, which is why the ratios apply
to files of at least 200 tokens and smaller files are gated on an absolute count.

## The audit

At v0.12.2 every file of `eval/baselib` is under the gate. The file that
prompted it, `datetime.R`, scored before its rewrite exact **0.23**, normalised
**0.47**, longest exact run **88** tokens; after, **0.17**, **0.39**, **38**.

Three files were rewritten clean room in v0.12.2 (pull request 30): `datetime.R`
(written without the old file or R's R sources, from the Rd pages and oracle
experiments), `vectors.R` (`ifelse`, `append`, `isTRUE`/`isFALSE`, `NROW`/`NCOL`)
and `conditions.R`. Behaviour did not change: 2 028 of 2 028 goldens, and the
datetime fuzz against the old file gave the same results with the same four
counted exclusions.

## The oracle guard

Printing a base function in the reference R shows its GPL body, and this happened
once while checking a behaviour. `internal/oracle/oracle.R` therefore refuses, with
the error `oracle: printing base closures is disallowed by the clean-room rule`,
to print, deparse, `dput`, `body`, `as.list`, `format`, serialize or return any
closure of base, stats, utils, methods and the other packages of R, however it is
reached (auto-print, `:::`, `getAnywhere`, inside a list or an attribute, or a
copy made in the global environment, which is recognised by its body).
`.Internal` is refused in requests. Closures of our own print, and goldens that
compare function printing use closures defined in the snippet.

**Known gap**
([issue 31](https://github.com/go-embedded-r/r/issues/31), open): a base closure
whose body is under 100 serialized bytes can be copied out of the namespace and
then printed. Someone with an installed R can also still read its sources, which
is why the rule is procedural too.

## Provenance in every pull request

`.github/pull_request_template.md` asks, for any change under `eval/baselib/`:
the `.Rd` pages read, whether any R function body was seen and how (and who then
rewrote the file blind), and the gate score of the first draft before any
reshaping. Pull request 30 records its own provenance: `datetime.R` and
`vectors.R` were rewritten by the agent that built the gate, which saw only the
checker's references and scores; `conditions.R` was rewritten blind by a separate
agent, after the first rewrite was replaced because its author had read the old
10-line file; that agent saw one base closure's printed body when the oracle
printed it, which is why the guard exists.

## Run it

```sh
export RGO_RSOURCE=/path/to/R-4.6.2          # an unpacked, checked R source tree
go run ./cmd/cleanroom check                 # eval/baselib/*.R; exit 1 over the gate
go run ./cmd/cleanroom check --list 20 f.R   # every shared run of 20+ tokens, as file:line
go run ./cmd/cleanroom floor some/*.R        # the same scores for independent code
go test ./internal/cleanroom ./cmd/cleanroom # the gate as a test; skips without R's sources
```

See also [Licences]({{< relref "licences" >}}).

---
title: "Clean room"
weight: 35
description: "v0.12.3: how base R's R code is rewritten and not copied, the similarity gate (cmd/cleanroom, winnowing), its thresholds and audit, the oracle guard and the provenance record. The gate is a floor, not proof."
---

<!-- From the README of go-embedded-r/r on main (v0.12.3), section "Clean-room process: the similarity gate", its NOTICE (rules 5 to 7) and the bodies of pull requests 30 and 32. Change those first, then here. -->

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

At v0.12.3 every file of `eval/baselib` is under the gate. "Best R file" is the
one with the highest normalised containment; against R's largest files that is
generic structure, which is why the normalised column alone is not the gate.
"Before" is the first draft, scored with formal argument lists still
fingerprinted; "after" is the file in the repository, scored as the gate now does.

| File | Best R file | Exact | Normalised | Longest exact run (before) | Rewritten |
| --- | --- | --- | --- | --- | --- |
| `datetime.R` | `base/R/datetime.R` | 0.11 (before 0.23) | 0.34 (0.47) | 36 (88) | yes, from the Rd pages and the oracle, without the old file |
| `vectors.R` | `tools/R/QC.R` | 0.00 (0.00) | 0.40 (0.41) | 22 (74, `ifelse`) | `ifelse`, `append`, `isTRUE`, `isFALSE`, `NROW`, `NCOL` |
| `conditions.R` | `methods/R/RMethodUtils.R` | 0.00 (0.27) | 0.43 (0.43) | 6 (13) | yes, blind, by a second agent |
| `env.R` | `tools/R/QC.R` | 0.00 (0.01) | 0.52 (0.55) | 22 (26) | no |
| `support.R` | `tools/R/QC.R` | 0.01 (0.01) | 0.41 (0.47) | 30 (35) | no |
| `strings.R` | `base/R/namespace.R` | 0.00 (0.03) | 0.23 (0.28) | 26 (31) | no |

The longest runs left (22 to 36 tokens) are a documented one-liner or an error
message the oracle fixes.

Three files were rewritten clean room in v0.12.2 (pull request 30): `datetime.R`
(written without the old file or R's R sources, from the Rd pages and oracle
experiments), `vectors.R` (`ifelse`, `append`, `isTRUE`/`isFALSE`, `NROW`/`NCOL`)
and `conditions.R`. Behaviour did not change: 2 028 of 2 028 goldens, and the
datetime fuzz against the old file gave the same results with the same four
counted exclusions.

## A failed first draft becomes a Go native

Since v0.14.1: a baselib function whose first draft fails the gate is
implemented as a Go native from the Rd page and the oracle, never reshaped in R,
and the failed first-draft score is recorded in the pull request. Reshaping a
draft until the gate passes would make the gate the target. The cases that
fixed the rule (pull request 36):

| Function | First draft, blind, from `rle.Rd` / `scale.Rd` and the oracle | Gate | Now |
| --- | --- | --- | --- |
| `rle`, `inverse.rle` | longest exact run **51** tokens, normalised **64**, against `base/R/rle.R` (a blind author converged on R's own run) | 40 and 60 | `eval/rle.go` |
| `scale.default` | exact containment **0.26** against `base/R/scale.R` | 0.20 | `eval/scale.go` (R keeps the generic and a thin method) |

The v0.14.0 drafts of the same functions had been reshaped to pass (the
"redesigned" rows of the repository's gate table); that is what the rule now
forbids. The submitted `scale_rle.R` scores 0.00 exact, 0.45 normalised, longest
run 10 tokens.

## The oracle guard

Printing a base function in the reference R shows its GPL body, and this happened
once while checking a behaviour. `internal/oracle/oracle.R` therefore refuses, with
the error `oracle: printing base closures is disallowed by the clean-room rule`,
to print, deparse, `dput`, `body`, `as.list`, `format`, serialize or return any
closure of base, stats, utils, methods and the other packages of R, however it is
reached (auto-print, `:::`, `getAnywhere`, inside a list or an attribute, or a
copy made in the global environment, recognised with `identical()` against the
bodies and formals of every function of those packages, at any size, including
copies made with `body<-`, `attr<-` or `as.function`).
`.Internal` is refused in requests. Closures of our own print, and goldens that
compare function printing use closures defined in the snippet.

Until v0.12.3 a base closure whose body was under 100 serialized bytes could be
copied out of the namespace and printed
([issue 31](https://github.com/go-embedded-r/r/issues/31), closed by
[pull request 32](https://github.com/go-embedded-r/r/pull/32)); the size floor is
gone. Someone with an installed R can still read its sources, which is why the
rule is procedural too.

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

---
title: "Evaluator (in progress)"
weight: 27
description: "M2, package eval: the design choices, and slice 1 (71 of 210 primitives, judged by the oracle), on main and not yet tagged."
---

<!-- From the README of go-embedded-r/r on main and its docs/m2-design.md. Change the repository's README and design first, then here. -->

**Status: in progress.** Slice 1 of M2 is merged on `main`
([pull request 5](https://github.com/go-embedded-r/r/pull/5), CI green) and is
**not tagged yet**: the latest release,
[v0.2.0](https://github.com/go-embedded-r/r/releases/tag/v0.2.0), has the
parser and deparse (M1). The design is
[`docs/m2-design.md`](https://github.com/go-embedded-r/r/blob/main/docs/m2-design.md).

```go
s := eval.NewSession(eval.Options{})   // one global environment that persists
res, err := s.Eval("f <- function(x, y = 2) x * y; f(21)")
res.Value, res.Visible, res.Warnings   // the last value, whether R would print it, warnings
// err is *eval.Error (R's message and call), *syntax.Error or *syntax.LimitError
```

## Design choices

- **Clean room.** R's C code is a specification, never a source. The design is
  written from the R Language Definition, *R Internals*, the help pages and what
  the oracle (GNU R 4.6.2) does; each slice is checked by the oracle lane before
  it is merged.
- **Bounded.** Recursion depth (R's own `expressions` limit, 5000), vector
  allocation size and evaluation steps are session limits, refused with R's own
  error, never a Go crash.
- **Typed Go slices for vectors**, not `[]interface{}`: arithmetic loops run over
  `[]float64` and `[]int32`, and NA is a value inside the slice (R's INT_MIN and
  NaN-payload conventions), never a side bitmap.
- **Calls are `*Lang` with an argument slice**, not pairlists; a real pairlist type
  is only for `formals`, `attributes` and `pairlist()`.
- **Attributes are an ordered slice**, because their order is observable.
- **Symbols are interned**, so `quote(x) == quote(x)` is pointer equality.
- **`*Missing` is its own type**, for `f(x, )` and `missing()`.
- **No byte-code**: the AST walker is the semantics; a faster tier would be a
  later, measured decision.
- **Closures take promises**: lazy arguments, `missing`, `...`, with R's
  three-pass argument matching (exact name, unique partial name, position).
- **Lexical control flow is not panic-based** on the hot path (`break`, `next`,
  `return` travel as a flow code); errors and non-lexical exits use a typed Go
  panic recovered at the owning context.
- **The oracle judges.** Goldens are R's answers to inputs written in the
  repository, compared byte for byte; the tests replay them without R.

## Slice 1: 71 of R's 210 primitives

Constants and symbols, environments, `<-`/`=`/`<<-`, `function` and closures
with R's three-pass argument matching, promises (lazy arguments, `missing`,
`...`), `{`, `if`, `for`, `while`, `repeat`, `break`, `next`, `return`, `quote`,
and arithmetic, comparison and logic with R's recycling, NA rules, type
promotion and integer overflow; `c`, `list`, `length`, `is.*`, `as.*`,
`sum`/`prod`/`min`/`max`/`any`/`all`, `abs`/`sqrt`/`floor`/`ceiling`/`exp`,
`seq_len`, `seq_along`. Each primitive is exercised by the goldens; the rest of
the primitives raise `could not find function` until their slice. Complex
assignment targets (`x[i] <- v`), S3 dispatch and the condition system are the
next slices. Errors and warnings are R's own: `Error in f(1) : object 'y' not
found`.

## Judged by the oracle

| Check | Result |
| --- | --- |
| goldens: 314 snippets of `testdata/eval` (arithmetic, comparison, sequences, coercion, summaries, closures, matching, laziness, loops, errors), each compared with R 4.6.2's serialized value (byte for byte), visibility, warnings, and error message and call; replayed by `go test` without R | **314 / 314** |
| differential fuzz: random arithmetic, logic and summary expressions, R against Go (`go run ./cmd/rparity evalfuzz`; seeds 6, 7, 8) | **179 998 / 180 000** |
| `FuzzEval`: random source never panics, hangs or allocates without bound | 8.6 million executions, clean |

## Known divergences

All from the platform rather than the semantics.

- The bits of a NaN (its sign, whether it is quiet) are whatever the hardware
  made of it, so goldens and the fuzz compare NA against NaN, not their
  payloads.
- `exp` and the power with a fractional or negative exponent can differ from the
  C library's `pow`/`exp` in the last bit. The fuzz keeps the exponents integral
  and leaves `exp` out; they come with go-numerics in M4. The 2 of 180 000 above
  are one of each (a `pow` last bit, and `NA %% 3` added to a NaN).
- `min`/`max` of a character vector mixed with numbers is not collated like R's.
- The modulus family is written with explicit fused multiply-adds, as R is
  compiled on arm64, so it gives the same result everywhere; R on x86-64 Linux
  does not fuse, so its `%%` of huge doubles can differ there (the CI lane
  reports the evaluator fuzz on Linux, it does not gate it).

## Not started

Printing is a large, exact piece of R and comes with M3; the first slice
returns values, and the oracle lane compares values, not printed text. M3 to M8
(base, stats, regex, graphics, notebook bridge, wasm playground, CRAN ports) are
planned. Speed against R is not measured yet.

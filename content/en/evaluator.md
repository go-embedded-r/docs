---
title: "Evaluator"
weight: 27
description: "M2, package eval: the design choices, slices 1 and 2 (94 of 210 primitives and 32 base closures, 819 of 819 goldens), judged by the oracle."
---

<!-- From the README of go-embedded-r/r on main (v0.3.0) and its docs/m2-design.md. Change the repository's README and design first, then here. -->

**Status: slices 1 and 2 of M2 are done**, released in
[v0.3.0](https://github.com/go-embedded-r/r/releases/tag/v0.3.0) (2026-11-04,
[pull request 8](https://github.com/go-embedded-r/r/pull/8)); the current
release is [v0.4.0](https://github.com/go-embedded-r/r/releases/tag/v0.4.0)
([pull request 9](https://github.com/go-embedded-r/r/pull/9)), which bounds the
session limits by default. Slice 1 was on
`main` after v0.2.0. M3 onwards is planned. The design is
[`docs/m2-design.md`](https://github.com/go-embedded-r/r/blob/main/docs/m2-design.md);
the API is on [Eval API]({{< relref "eval-api" >}}).

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
- **Conditions**: handlers are a stack on the session. Exiting handlers unwind
  with a typed panic caught by their `tryCatch`, calling handlers run with the
  stack truncated below them, and restarts are found by name. Errors are
  signalled as conditions before they end the evaluation, so `tryCatch(error=)`
  and `withCallingHandlers(error=)` see them.
- **Replacement functions** use always-copy semantics; the ownership
  optimisation stays deferred.
- **The oracle judges.** Goldens are R's answers to inputs written in the
  repository, compared byte for byte; the tests replay them without R.

## Coverage: 94 of R's 210 primitives and 32 base closures

**Slice 1:** constants and symbols, environments, `<-`/`=`/`<<-`, `function` and
closures with R's three-pass argument matching, promises (lazy arguments,
`missing`, `...`), `{`, `if`, `for`, `while`, `repeat`, `break`, `next`,
`return`, `quote`, and arithmetic, comparison and logic with R's recycling, NA
rules, type promotion and integer overflow; `c`, `list`, `length`, `is.*`,
`as.*`, `sum`/`prod`/`min`/`max`/`any`/`all`,
`abs`/`sqrt`/`floor`/`ceiling`/`exp`, `seq_len`, `seq_along`.

**Slice 2:**

- replacement functions with copy-on-modify (`x[i] <- v`, `x[[i]] <- v`,
  `x$a <- v`, `names(x) <- v`, `attr(x, "a") <- v`, `f(g(x)) <- v`, user `"f<-"`
  functions);
- `[`, `[[` and `$` with R's drop, NA, out-of-bounds, negative,
  partial-matching and matrix/array rules;
- attributes (`attr`, `attributes`, `class`, `dim`, `dimnames`, `levels`,
  `structure`; factors keep levels);
- S3 (`UseMethod`, `NextMethod`, internal dispatch on `[`, `[[`, `$`, their
  assignment forms, `length`, `print`, and the group generics `Ops`, `Math`,
  `Summary`);
- the condition system (`stop`, `warning`, `message`, `signalCondition`,
  `tryCatch` with `finally`, `withCallingHandlers`, `try`, `on.exit`, the
  restarts `muffleWarning` and `muffleMessage`, `suppressWarnings`,
  `suppressMessages`, custom conditions);
- `print` and auto-print that reproduce R's text for atomic vectors, named
  vectors, lists, matrices and arrays, factors, conditions, closures and
  attributes (R's `formatReal` with 7 digits, `[k]` labels at width 80).

`matrix`, `array`, `letters` and `LETTERS` are there for the tests. Each
primitive and each of the 32 closures (`print`, `cat`, `paste`, `tryCatch`,
`structure`, `inherits`, ...) is exercised by the goldens; the rest of the
primitives raise `could not find function` until their slice. Errors and
warnings are R's own: `Error in f(1) : object 'y' not found`.

## Judged by the oracle

| Check | Result |
| --- | --- |
| goldens: 819 snippets of `testdata/eval` (arithmetic, comparison, sequences, coercion, summaries, closures, matching, laziness, loops, subsetting, assignment, attributes, S3, conditions, printing, errors), each compared with R 4.6.2's serialized value (byte for byte), visibility, printed output, standard error, warnings, and error message and call; replayed by `go test` without R | **819 / 819** |
| differential fuzz, arithmetic, logic and summary (`go run ./cmd/rparity evalfuzz`; seeds 3, 6, 7, 8) | **189 998 / 190 000** |
| differential fuzz, subsetting and assignment on vectors, lists, matrices and arrays (`evalfuzz --mode subset`; seeds 7, 11) | **25 969 / 26 000** |
| `FuzzEval`: random source never panics, hangs or allocates without bound (the corpus keeps the 9 crashers it found) | 140 million executions in the last two runs, clean |

## Known divergences

Of slice 2:

- `[[<-` with a vector subscript on a list that goes through an NA, an absent
  name or a logical (`x[[c(1, NA)]] <- v`, `x[[c("a", "c")]] <- v` on an
  unnamed list): R's message comes from its recursive indexing and ours differs
  (31 of 26 000 fuzz cases). A few R messages for `[[<-` on atomic vectors with
  0, negative or NA subscripts follow the oracle for the common cases only.
- **Function printing**: `print` of a function does not show R's
  `<environment: 0x...>` address (a placeholder), and `.Primitive` formals are
  not shown.
- **CJK width**: wide (CJK) characters are counted as one column in named
  vectors.
- Printing an array with more than 2^20 slices or 2^22 rows is refused instead
  of running for ever. `print` of list elements dispatches only to the methods R
  registers itself (factor, condition), as R does.

From the platform rather than the semantics:

- The bits of a NaN (its sign, whether it is quiet) are whatever the hardware
  made of it, so goldens and the fuzz compare NA against NaN, not their
  payloads.
- `exp` and the power with a fractional or negative exponent can differ from the
  C library's `pow`/`exp` in the last bit (the fuzz keeps the exponents integral
  and leaves `exp` out; they come with go-numerics in M4). The 2 of 190 000
  arithmetic fuzz cases are one of each (a `pow` last bit, and `NA %% 3` added to
  a NaN).
- `min`/`max` of a character vector mixed with numbers is not collated like R's.
- The modulus family is written with explicit fused multiply-adds, as R is
  compiled on arm64, so it gives the same result everywhere; R on x86-64 Linux
  does not fuse, so its `%%` of huge doubles can differ there (the CI lane
  reports the evaluator fuzz on Linux, it does not gate it).

## Not started

M3 to M8 (base, stats, regex, graphics, notebook bridge, wasm playground, CRAN
ports) are planned.

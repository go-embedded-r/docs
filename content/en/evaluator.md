---
title: "Evaluator"
weight: 27
description: "M2, package eval: the design choices, slices 1 to 4 (175 of 210 primitives and 110 base closures, 1 154 of 1 154 goldens), judged by the oracle; M2 is complete."
---

<!-- From the README of go-embedded-r/r on main (v0.7.0) and its docs/m2-design.md. Change the repository's README and design first, then here. -->

**Status: M2 is complete (slices 1 to 4)**, in
[v0.7.0](https://github.com/go-embedded-r/r/releases/tag/v0.7.0)
([pull request 12](https://github.com/go-embedded-r/r/pull/12), 2026-11-07):
175 of R's 210 primitives and 110 base closures, 1 154 of 1 154 goldens. The
other 35 primitives are **refused with an error that says why**
(`eval.Unsupported`, listed by `go run ./cmd/rparity missing`, not counted as
implemented): the complex and raw types, S4, formulas and expression vectors,
foreign calls (`.C`, `.Call`, `.External*`, `.Fortran`, `lazyLoadDBfetch`),
`.Internal`, and the tracing and debugging primitives. M3, base R, has started:
see [M3 plan / base coverage]({{< relref "m3-plan" >}}).

History: slice 2 was released in
[v0.3.0](https://github.com/go-embedded-r/r/releases/tag/v0.3.0)
([pull request 8](https://github.com/go-embedded-r/r/pull/8)); v0.4.0
([pull request 9](https://github.com/go-embedded-r/r/pull/9)) bounded the
session limits by default;
[v0.5.0](https://github.com/go-embedded-r/r/releases/tag/v0.5.0)
([pull request 10](https://github.com/go-embedded-r/r/pull/10), 2026-11-05)
added slice 3;
[v0.6.0](https://github.com/go-embedded-r/r/releases/tag/v0.6.0)
([pull request 11](https://github.com/go-embedded-r/r/pull/11), 2026-11-06)
made its regular expressions a port of TRE (v0.5.0's were an approximation on
Go's `regexp`); see [Regular expressions]({{< relref "regex" >}}).
The design is
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

## Coverage: 102 of R's 210 primitives and 94 base closures

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

**Slice 3:**

- environments as values (`environment`, `environment<-`, `new.env`,
  `globalenv`, `emptyenv`, `baseenv`, `parent.env`, `parent.frame`, `sys.call`,
  `sys.function`, `assign`, `get`, `get0`, `exists`, `mget`, `ls`, `rm`, `local`,
  `eval`, `evalq`, `list2env`, `as.environment`);
- the apply family (`lapply`, `sapply`, `vapply`, `Map`, `mapply`, `Reduce`,
  `Filter`, `Find`, `Position`, `do.call`, `match.fun`);
- character functions (`nchar`, `substr`, `substring`, `strsplit`, `sprintf`,
  `format`, `toupper`, `tolower`, `casefold`, `chartr`, `trimws`, `startsWith`,
  `endsWith`, `Encoding`, `enc2utf8`, `utf8ToInt`, `intToUtf8`, `validUTF8`);
- regular expressions (`grepl`, `grep`, `sub`, `gsub`, `regexpr`, `gregexpr`,
  `regmatches`): POSIX extended syntax with leftmost-longest matching, on a Go
  port of TRE since v0.6.0 (see [Regular expressions]({{< relref "regex" >}})).

**Slice 4** (v0.7.0) completes M2: the rest of the math group (`log` family,
trigonometric and hyperbolic functions, `gamma` family, `trunc`, `sign`, `round`
and `signif` with R 4's rounding algorithm), `cumsum`/`cumprod`/`cummax`/`cummin`,
`range`, `is.finite`/`is.infinite`/`is.nan`, `anyNA`, `nzchar`, `switch`,
`substitute`, `nargs`, `...length`/`...elt`/`...names`, `call`, `as.call`,
`forceAndCall`, `::`/`:::` (resolved in the base and global environments),
`%*%`/`crossprod`/`tcrossprod`, `seq.int`, `proc.time`, and `typeof`, `mode`,
`deparse`, `as.name`, `unique`, `duplicated`, `match`, `%in%`, `factor`, `levels`,
`format` of factors. **Collation** is decided and documented: see
[M3 plan / base coverage]({{< relref "m3-plan" >}}).

`matrix`, `array`, `letters` and `LETTERS` are there for the tests. Each
primitive and each of the 110 closures (`print`, `cat`, `paste`, `tryCatch`,
`structure`, `inherits`, ...) is exercised by the goldens; the rest of the
35 primitives that remain are refused with an error that says why. Errors and
warnings are R's own: `Error in f(1) : object 'y' not found`.

## Judged by the oracle

| Check | Result |
| --- | --- |
| goldens: 1 154 snippets of `testdata/eval` (math group, language primitives, factors and matching, environments, apply, strings, regular expressions, arithmetic, comparison, sequences, coercion, summaries, closures, matching, laziness, loops, subsetting, assignment, attributes, S3, conditions, printing, errors), each compared with R 4.6.2's serialized value (byte for byte), visibility, printed output, standard error, warnings, and error message and call; replayed by `go test` without R | **1 154 / 1 154** |
| differential fuzz, arithmetic, logic and summary (`go run ./cmd/rparity evalfuzz`; seeds 3, 6, 7, 8) | **189 998 / 190 000** |
| differential fuzz, subsetting and assignment on vectors, lists, matrices and arrays (`evalfuzz --mode subset`; seeds 7, 11, 21) | **35 963 / 36 000** |
| differential fuzz, regular expressions: random POSIX extended patterns, with lazy quantifiers, back-references, `\<` `\>` `\b` and non-ASCII text, through `grepl`, `sub`, `gsub`, `regexpr`, `gregexpr`, `strsplit`, `regmatches` (`--mode regex`; seeds 1, 7, 21; nothing excluded) | **60 000 / 60 000** |
| differential fuzz, character functions: `nchar`, `substr`, `substring`, `toupper`, `trimws`, `strsplit`, `sprintf`, `format`, `chartr` (`--mode strings`; seeds 1, 5, 21) | **20 000 / 20 000** |
| differential fuzz, `lapply`, `sapply`, `vapply`, `Map`, `mapply`, `Reduce`, `Filter`, `unlist`, `rep`, `seq`, `rev` (`--mode apply`; seeds 3, 21) | **18 000 / 18 000** |
| differential fuzz, the math group (`--mode math`; seeds 1, 4, 9). **Exclusions, counted:** 5 979 of the last run's 20 000 expressions are libm functions compared through `round(x, 6)`, because their last bit depends on the C library; `digamma`, `trigamma`, `gamma` and `lgamma` at 1e-10 are left out | **20 000 / 20 000** |
| `FuzzEval`: random source never panics, hangs or allocates without bound (the corpus keeps the 13 crashers it found) | about 70 million executions in the last runs, clean |

## Known divergences

Of slice 2:

- `[[<-` with a vector subscript on a list that goes through an NA, an absent
  name or a logical (`x[[c(1, NA)]] <- v`, `x[[c("a", "c")]] <- v` on an
  unnamed list): R's message comes from its recursive indexing and ours differs
  (31 of 26 000 fuzz cases in the earlier run). A few R messages for `[[<-` on
  atomic vectors with 0, negative or NA subscripts follow the oracle for the
  common cases only.
- **Function printing**: `print` of a function does not show R's
  `<environment: 0x...>` address (a placeholder), and `.Primitive` formals are
  not shown.
- Printing an array with more than 2^20 slices or 2^22 rows is refused instead
  of running for ever. `print` of list elements dispatches only to the methods R
  registers itself (factor, condition), as R does.

Of slice 3:

- **Regular expressions**: TRE's approximate-matching syntax is not ported,
  non-ASCII classes use Go's Unicode tables, `perl = TRUE` is refused, and
  `grepl` is 1.44 times slower than R (an open defect); see
  [Regular expressions]({{< relref "regex" >}}).
- `sort` of non-atomic values is not implemented.
- **Collation** (decided in slice 4): strings sort and compare as R 4.6.2 does
  under the oracle's `C.UTF-8`, which is ICU order, not code point (punctuation,
  digits, letters; accents after the plain letter; lowercase before uppercase).
  Strings outside Latin script compare by code point after the Latin letters, a
  documented divergence. See [M3 plan / base coverage]({{< relref "m3-plan" >}}).
- Widths of `sprintf` and `format` above 2^24 are refused. `sprintf("%05d", NA)`
  zero-pads the `NA` as macOS does, where glibc pads with spaces (the C library
  decides in R, so the golden leaves it out).

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

M3 (base R) is in progress. M4 to M8 (stats, graphics, notebook bridge, wasm
playground, CRAN ports) are planned.

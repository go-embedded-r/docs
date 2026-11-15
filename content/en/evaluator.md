---
title: "Evaluator"
weight: 27
description: "M2, package eval: the design choices, slices 1 to 4 (175 of 210 primitives and 110 base closures, 5 838 goldens at v0.21.0), judged by the oracle; M2 is complete, and M3 has added a base library, strings, conditions, environments, namespaces and dates and times."
---

<!-- From the README of go-embedded-r/r on main (v0.12.1) and its docs/m2-design.md. Change the repository's README and design first, then here. -->

**Status: M2 is complete (slices 1 to 4)**, in
[v0.7.0](https://github.com/go-embedded-r/r/releases/tag/v0.7.0)
([pull request 12](https://github.com/go-embedded-r/r/pull/12), 2026-11-07):
175 of R's 210 primitives and 110 base closures, 1 154 of 1 154 goldens at
that release. The
other 35 primitives are **refused with an error that says why**
(`eval.Unsupported`, listed by `go run ./cmd/rparity missing`, not counted as
implemented): the complex and raw types, S4, formulas and expression vectors,
foreign calls (`.C`, `.Call`, `.External*`, `.Fortran`, `lazyLoadDBfetch`),
`.Internal`, and the tracing and debugging primitives. M3, base R, has started:
see [M3 plan / base coverage]({{< relref "m3-plan" >}}).

Later releases, all 2026-11-07: v0.7.1
([pull request 13](https://github.com/go-embedded-r/r/pull/13)) takes tre v0.1.1
and streams the `sub`/`gsub` loop (`BenchmarkGsubGroup`, 20 000 lines: 247k to 87k
allocations per operation, 9.0 to 4.6 MB per operation; the timing against R is
deferred because of machine load, so no speed is claimed). v0.8.0 (slice 5)
starts the base library: our own R in `eval/baselib/*.R` (BSD, clean room) plus
Go natives, **1 202 of 1 202 goldens**, vector fuzz 18 000 of 18 000, base
coverage 22.2 %. v0.8.1
([pull request 16](https://github.com/go-embedded-r/r/pull/16)) takes tre v0.1.3.
v0.8.2 ([pull request 18](https://github.com/go-embedded-r/r/pull/18)) makes
arithmetic, comparison and `&`/`|` on two arrays with different dims raise R's
`non-conformable arrays`, and an array against a longer vector raise `dims
[product n] do not match the length of object [m]` (zero-length arrays are fine,
as in R); it also keeps dimnames padding in step (both were FuzzEval crashers).
v0.9.0 and v0.9.1 (slice 6) add the string and format functions. v0.10.0 (slice 7)
adds restarts and the call stack, **1 302 of 1 302 goldens**, base coverage 24.4 %
(23.3 % at v0.9.0). v0.11.0
([pull request 27](https://github.com/go-embedded-r/r/pull/27), 2026-11-08, slice
8) adds environments and a one-namespace model: **1 570 of 1 570 goldens**, env
fuzz 15 000 of 15 000, base coverage 26.4 % (360 of 1 362). Slices 6 to 8 are
described below. v0.12.0 (2026-11-08,
[pull request 28](https://github.com/go-embedded-r/r/pull/28), slice 9) adds dates
and times: **2 022 of 2 022 goldens**, base coverage 33.9 % (462 of 1 362).
v0.12.1 ([pull request 29](https://github.com/go-embedded-r/r/pull/29)) marks the
goldens that were recorded on macOS as platform goldens and reproduces how R
computes `%s`; the README then counts 2 028 goldens, 9 of them macOS-only.
v0.12.2 ([pull request 30](https://github.com/go-embedded-r/r/pull/30)) adds the
clean-room similarity gate and rewrites `datetime.R`, `vectors.R` and
`conditions.R` without changing behaviour; see [Clean room]({{< relref "clean-room" >}}).
v0.13.0 ([pull request 33](https://github.com/go-embedded-r/r/pull/33), slice 10)
adds connections and files: **2 356 goldens**, base coverage 37.7 % (513 of 1 362);
v0.13.1 ([pull request 34](https://github.com/go-embedded-r/r/pull/34)) is its
security audit. See [Files and connections]({{< relref "files-and-connections" >}}).
v0.14.0 (slice 11, [pull request 35](https://github.com/go-embedded-r/r/pull/35))
adds matrices, arrays and grouping (`apply`, `sweep`, `scale`, `aperm`, `split`,
`tapply`, `rle`, `rownames`/`colnames`, and the printing they need): 351 new
goldens, base coverage 40.8 % (556 of 1 362). v0.14.1
([pull request 36](https://github.com/go-embedded-r/r/pull/36), with the fix of
[pull request 37](https://github.com/go-embedded-r/r/pull/37)) makes `rle`,
`inverse.rle` and the arithmetic of `scale.default` Go natives after their blind
R drafts failed the clean-room gate: **2 737 goldens**. See
[Clean room]({{< relref "clean-room" >}}).
v0.15.0 (slice 12, [pull request 38](https://github.com/go-embedded-r/r/pull/38))
adds the complex and raw types, expression vectors (`expression`,
`parse(text =)`, `str2lang`, `eval`) and binary connections: 307 new goldens,
**3 048 in all**, base coverage 43.4 % (591 of 1 362), **183 of the 210 primitives**
with 27 still refused (S4, formulas, foreign calls, `.Internal`, tracing). Nothing
was written in R: the slice is Go natives. See
[Files and connections]({{< relref "files-and-connections" >}}) for the binary connections.

## Data frames and `table` (v0.16.0)

v0.16.0 (slice 14, [pull request 40](https://github.com/go-embedded-r/r/pull/40))
adds data frames: `data.frame`, `as.data.frame`, their extraction and replacement
methods, `merge`, `split`, `subset`, `transform`, `table`, `cut`, `addNA`,
`expand.grid`, `interaction`, `cbind`/`rbind`, compact `row.names`, `make.names`,
printing and formatting. Part is R in `eval/baselib` (`dataframe.R`, `dfmethods.R`,
`table.R`) and part Go natives; five first drafts failed the clean-room gate and
became natives, see [Clean room]({{< relref "clean-room" >}}). 241 new goldens,
**3 289 in all**, now sharded under `testdata/golden/eval/`; base coverage 48.4 %
(659 of 1 362). The data-frame fuzz mode (seeds 13, 21, 22, 23; about 2 970
programs each) found 0 differences, and `FuzzEval` ran 6.4 million executions
without a crasher.

Divergences: `summary` of a data frame or table, `Ops`/`Math`/`Summary` of data
frames, `aggregate`, `by`, `stack`, `within`, `ftable`, `xtabs`, and matrix and
list columns are not in this release; `df[i, "missing"]` errors where R answers
`NULL`; a long error call breaks its lines differently from R's.

## `summary`, `all.equal` and group methods (v0.17.0)

v0.17.0 (slice 15, [pull request 41](https://github.com/go-embedded-r/r/pull/41))
takes the deferred base functions that need no shared engine, chosen by
`rparity basecov` (its `--list-pure` list of closures that call no missing
`.Internal`): `summary` (default, factor, matrix, data frame) and its printing,
`format.pval`, `all.equal` (all methods), `isSymmetric`, `formals`, `body<-`,
`as.function`, `sys.parent`, the `Ops`, `Math` and `Summary` group methods of
factors and data frames, `within`, `hexmode` and `octmode`, `AsIs`, the date and
time `diff`, `range`, `split`, `length<-` and `xtfrm` methods, the condition
methods, `.Deprecated` and `.Defunct`. It adds 444 goldens, **3 733 in all**;
basecov goes from 659 to 782 of 1 362 (48.4 % to 57.4 %).

Behaviour changes: `attributes()` lists attributes in the order they were set, as
R 4.6 does; `zapsmall` rounds digits like R; `as.difftime` stores doubles.

Divergences: `formals` returns a list, not a pairlist; `mean(x, trim =)` of an
inexact sum can differ in the last bit; `Vectorize` is not tested (the oracle
cannot return it). Still deferred: `strwrap`, `pretty`, `kronecker`,
`dput`/`dget`/`dump`, `print.rle`, `seq.POSIXt`, `round`, `trunc` and `cut` of
dates, `by`, `file.info`, `scan`, `numeric_version`, `summary` of dates and
tables.

## Version numbers, dates and times, text and `by` (v0.18.0, v0.18.1)

v0.18.0 (slice 16, [pull request 45](https://github.com/go-embedded-r/r/pull/45))
takes what slice 15 deferred, in the order of `rparity basecov`: the version
numbers (`numeric_version`, `package_version`, `R_system_version`,
`getRversion`, `R.version`, `compareVersion`), the dates and times (`seq.POSIXt`,
`round`, `trunc` and `cut` of `Date` and `POSIXt`, `summary` of dates, times and
differences), `strwrap`, `pretty`, `kronecker` and `%x%`, `print.rle`, `dput`,
`dget`, `dump` and the options of `deparse`, `by`, `file.info` and its relatives,
the marks of `format`, `formatC` and `prettyNum`, and a few small closures. It
adds 765 goldens, **4 498 in all**; basecov goes from 782 to 869 of 1 362 (57.4 %
to 63.8 %). The slice 16 fuzz mode (seeds 41, 42, 43, 44; 3 000 programs each)
found 0 differences, and `FuzzEval` ran 3.4 million executions without a crasher.

Clean room: no R file failed at its first draft except `cut.Date` and
`cut.POSIXt`, converted to Go natives per the rule in v0.18.1
([pull request 46](https://github.com/go-embedded-r/r/pull/46)); v0.18.0 had
reworked the R draft until it passed, which the rule does not allow. See
[Clean room]({{< relref "clean-room" >}}).

Divergences: the oracle's arm64 build fuses `from + i * by` into one rounding and
its amd64 build does not, so ten snippets (the `pretty` and
`seq(length.out =)` ones) are compared within 8 units in the last place; `pretty`
follows the parameters the help page documents but is not a copy of R's
algorithm; `file.info` reports uid and gid 0 and the owner `root` for every file;
`print.rle` writes the `str` line for numbers, text, logicals and raw vectors
only; `sort_by` of a formula is not supported; `R.version` is the one of the
oracle's platform. Still deferred: `scan`, `read.table` and the readers of utils,
`summary` of tables (`pchisq`), formulas (`~`), `Vectorize`, `chkDots`,
`warnings`, `rowsum`, `jitter` and `sample`, `str`, `stack` and `unstack`
(utils), `aggregate` (stats), `array2DF`, the `[`, `[[` and `$<-` methods of
`POSIXlt`, the remaining `as.data.frame` methods.

## Formulas, `terms`, model frames, `aggregate` and `str` (v0.19.0)

v0.19.0 (slice 17, [pull request 47](https://github.com/go-embedded-r/r/pull/47))
opens formulas and what they allow: the `~` primitive, `terms`, the model frame,
`aggregate`, `stack`, `unstack`, `sort_by` with a formula; then, without
formulas, `str`, `rowsum`, `warnings`, `Vectorize`, `chkDots`, `array2DF`, the
`[`, `[[`, `[<-`, `[[<-` and `$<-` methods of `POSIXlt` and the remaining
`as.data.frame` methods. It adds 767 goldens, **5 265 in all**; basecov goes from
869 to 905 of 1 362 (63.8 % to 66.4 %), and 184 of the 210 primitives are
implemented (`~` is no longer refused). A search of 2 253 repositories of the
fleet found no formula parser, so the term algebra is written here, from
`?terms.object`, `?formula` and the oracle; `terms.formula` is a Go native.
The slice 17 fuzz mode (seeds 41, 42, 43, 44; 3 000 programs each) ends with 0
differences, and `FuzzEval` ran 4.1 million executions without a crasher after
a first run found and fixed `format.pval(0, c)`.

Clean room: the first drafts that failed the gate became Go natives, not
reworked: `model.frame.default`, `.MFclass`, `model.response` and
`get_all_vars` (longest exact run 80 of `formula17.R` against `models.R`),
`rowsum.default` and `rowsum.data.frame` (53 against `rowsum.R`), and `unname`
(48 against `unname.R`). See [Clean room]({{< relref "clean-room" >}}).

Bounds: a formula that expands to 2^20 terms or has 100 000 terms is refused
(`the formula expands to more than 65536 terms`), as is `aggregate` over 10^8
groups (`too many groups`) and the other 2^40-element cases of
`eval/bomb_slice17_test.go`.

Divergences: the oracle refuses to print what `str` shows of a list, a data
frame, a named vector, a matrix with dimnames, a formula or a function, and
refuses what `Vectorize` returns, so those are checked by Go tests, not by
goldens; `print` of a formula made in a function writes `<environment>` for the
address R prints; `terms` does not replace `poly` or `ns` in `predvars`, ignores
`neg.out` and warns on `abb`; `update` has no method but the formula;
`summary.warnings` and `format.warnings` are not written; the matrix and list
columns that `aggregate` builds (`FUN = range`) are not printed. Still
deferred: `scan`, `read.table`, `read.csv`, `read.delim`, `xtabs`, `ftable`,
`model.matrix` and the contrasts (slice 13 is blocked on linear algebra),
`summary` of tables, `median`, `var`, `quantile` and the other statistics of
the stats package, `serialize` and `unserialize`, the distributions and the
random number generators (shared engines).

## Table readers and writers, `xtabs` and statistics (v0.20.0)

v0.20.0 (slice 18, [pull request 49](https://github.com/go-embedded-r/r/pull/49))
adds the readers and writers of tables, `xtabs`, and the descriptive statistics.
Readers (Go natives, on the session's file system, `Options.FileRoot` and text
connections): `scan`, `read.table`, `read.csv`, `read.csv2`, `read.delim`,
`read.delim2`, `count.fields`, `type.convert`. Writers: `write.table`,
`write.csv`, `write.csv2`. `xtabs` (formula and data, a response or a `cbind` of
responses, `subset`, `na.action`, `addNA`, `exclude`, `drop.unused.levels`) and
`print.xtabs` are R; `summary.table` and `print.summary.table` are Go natives.
Statistics: `median`, `var`, `sd`, `cov`, `cor` (Pearson, Spearman, Kendall; the
five `use` modes), `quantile` (the nine types), `fivenum`, `IQR`, `mad`,
`weighted.mean`. They were first written here, because go-numerics has no
statistics kernel (slice 19 binds go-statistics/stats where it holds the bound);
each states an error bound asserted against math/big: `var` within 4u relative,
`cov` within 4u of the sum of the absolute products, `cor` within 8u absolute,
quantile types 4 to 9 within 4u of the larger neighbour, `median` and `fivenum`
within 2u, `weighted.mean` within 4u (u = 2^-53). R's own results differ from the
exact ones in the last bits (a quantile of values that cancel differs by up to 64
units in the last place); the fuzz compares them within 128 ulp and counts them.

It adds 537 goldens; basecov goes from 905 to 908 of 1 362 (66.7 %, most of the
new functions being utils or stats, not base); 184 of the 210 primitives. The
slice 18 fuzz mode (seeds 41, 42, 43, 44; 1 600 programs each) ends with 0
differences. Clean room: the first draft `xtabs18.R` failed the gate, and its
`summary.table` and `print.summary.table` became Go natives, see
[Clean room]({{< relref "clean-room" >}}). Bounds: a line of 240 000 fields, a
header that wide, a quote that never closes, 2^31 rows, more rows than
`MaxVector` and a table too large for `MaxFileBytes` are refused.

Divergences: `ftable` is deferred; `ecdf` is not
written; the numbers are parsed correctly rounded, R's own parser can differ in
the last bit beyond 15 digits (`numerals`); `type.convert` takes more than 15
significant digits as a loss; `quantile` of dates is supported for types 1 to 3
only, of unordered factors it refuses; `mad(x, low = TRUE)` of an empty vector
answers `NA` where R fails. The timing against R is not measured (the load was
50): open item go-embedded-r/r#48. Still deferred: `ftable`, `ecdf`, `read.fwf`,
`pchisq` and the test in `summary.table`, `model.matrix` (slice 13).

## Statistics on go-statistics, `pchisq` and `summary.table` (v0.21.0)

v0.21.0 (slice 19, [pull request 50](https://github.com/go-embedded-r/r/pull/50))
moves the statistics onto
[go-statistics/stats](https://github.com/go-statistics/stats) v0.1.1
(BSD-3-Clause) wherever it holds the bound slice 18 states, and finishes
`summary.table`. Bound to go-statistics: the ranks (`stats.RankData`), the tau-b
of `cor(method = "kendall")`, and `pchisq` and `qchisq` (central and non-central,
`lower.tail`, `log.p`, recycling, `NaNs produced`; `df = 0` is the point mass at
0). Compared against math/big: `Var` is 6e7 times over our 4u on `1e12 + U(0,1)`;
`Pearsonr` is 2e4 times over 8u on the same data; `Quantile` types 5 to 9, the
default 7 among them, are 18 to 24 times over; `Kendalltau` is within 0.25 of 8u;
`RankData` is exact.

Kept here, as gaps in go-statistics (the upstream issues are
go-statistics/stats#24 to #27): `var` and the covariance (a compensated kernel,
4u of the result; there is no covariance function at all), Pearson `cor` (8u
absolute), `quantile` types 5 to 9 and the R fuzz of types 2 and 3,
`weighted.mean` (4u of the sum of the absolute products), the signed pair count
of Kendall (`cov(method = "kendall")`), `fivenum`, `mad` (R's constant, `low`,
`high`, a centre and `na.rm`), `IQR`, the non-central chi-squared with `df = 0`,
and `qchisq` with `log.p`. Median and quantile are also slower than ours at 10^6
doubles (see go-embedded-r/r#48), so they were not bound.

`summary.table` now computes the statistic, `parameter`, `approx.ok` and
`p.value` of a table of several factors, and `print.summary.table` writes
`Test for independence of all factors`, the `Chisq`, `df` and `p-value` line and
the warning about the approximation as R does; `round` rounds a `digits` that is
not an integer to the nearest one, as R does.

Gates: 36 new goldens, **5 838 in all** (5 802 at v0.20.0), all agreeing with R
4.6.2; `pchisq` is closer to the exact value than R's (`pchisq(2, 3)`: exact
0.42759329552912018, ours 0.4275932955291202, R's 0.4275932955291213), so those
goldens compare 10 significant digits, 9 and 8 for the non-central ones; the
slice 18 fuzz mode (seeds 41 to 44) ends with 0 differences; `govulncheck` finds
nothing. The minimal program (`NewSession`, one `Eval`) grows from 10 334 742 to
11 127 917 bytes for `GOOS=js` (+7.7 %).

Divergences: `qchisq` of a `log.p` below -745; a non-central `df = 0`; `ftable`,
`ecdf` and `read.fwf` stay deferred. The timing against R is open item
go-embedded-r/r#48.

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

**Slice 10** (M3, v0.13.0 and v0.13.1) adds connections and files, documented on
[Files and connections]({{< relref "files-and-connections" >}}).

**Slice 9** (M3, v0.12.0 and v0.12.1) adds dates and times. **Classes**: `Date`,
`POSIXct`, `POSIXlt` (a list of the 11 components with `tzone` and `balanced`),
`difftime`. **Functions**: `Sys.time`, `Sys.Date`, `as.Date`, `as.POSIXct`,
`as.POSIXlt` (character with `format` and `tryFormats`, numeric with `origin`,
`Date`, between classes, factors), `strptime` (every conversion R's help page
lists, `%OS`, `%z`, 24:00, names matched without case, trailing text ignored),
`format`/`strftime`/`as.character` (`usetz`, `digits`, `%OSn`,
`options(digits.secs)`), `ISOdate`, `ISOdatetime`, `weekdays`, `months`,
`quarters`, `julian`, `difftime`, `as.difftime`, `units`, `units<-`, `format` and
`print` of differences, `seq.Date` (day, week, month, quarter, year, a number, a
difftime), `c`, `[`, `[[`, `[<-`, `rep`, `rev`, `sort`, `unique`, `mean`,
`range`, `diff` on all three classes, arithmetic, comparisons and
`Math`/`Summary` of `difftime` with R's rules. Non-finite times are kept apart as
R 4.2 does. **Time zones** come from Go's embedded IANA database (`time/tzdata`,
about 450 KB in the binary): `tz = ""` is the session's zone, the `TZ` variable
else the system's (the oracle runs with `TZ=UTC` and so do the goldens); a
repeated local hour reads as its first occurrence, as R's does. Support added on
the way: `options`, `getOption`, `deparse1`, `gettext`, `gettextf`, `rep.int`,
`is.factor`, `pmatch`, `unname`; `as.character` and `paste` of a factor give its
labels; `print` forwards its arguments to the method; `deparse` writes
attributes as `structure(...)`. `NewSession` now costs 56 µs and 1 091
allocations (626 at v0.11.0), measured at 1-minute load 5: `datetime.R` alone is
420 allocations, 110 closures defined eagerly at session start.

**Slice 8** (M3, v0.11.0) adds environments and a one-namespace model.
**Bindings**: `lockBinding`, `unlockBinding`, `bindingIsLocked`,
`lockEnvironment(bindings =)`, `environmentIsLocked`, `makeActiveBinding`,
`bindingIsActive`, `activeBindingFunction`, `delayedAssign`; every assignment
(`<-`, `<<-`, `assign`, `$<-`, `[[<-`, `list2env`, replacement forms) honours locks
and runs active bindings, with R's messages and calls; the base environment and
all its bindings are locked as in R (`assign("sum", 1, baseenv())` is refused).
`rm` takes `inherits`; `get`/`get0`/`exists`/`mget` take `mode`, `inherits` and
`ifnotfound` and give R's messages; `exists` does not force a promise. `as.list` is
generic (`as.list.environment` with `all.names` and `sorted`, `.default`,
`.function`, `.factor`); `eapply`, `with`, `within` (lists), `withVisible`,
`names(env)`. Environments carry attributes, so S3 dispatch works on them.
**Namespaces**: `asNamespace`, `getNamespace`, `isNamespace`, `isBaseNamespace`,
`getNamespaceName`, `getNamespaceExports`, `isNamespaceLoaded`, `requireNamespace`,
`.BaseNamespaceEnv`, `.GlobalEnv`, `topenv`, `::` and `:::`. The namespace of base
is an environment of its own (printed `<environment: namespace:base>`, enclosed by
the global environment) that shares the base frame. Errors raised directly under
`eval`, `evalq`, `local` and `with` report R's call. An active binding that read
itself used to overflow the Go stack; it is fixed, with a corpus entry and a bomb
test.

**Slice 7** (M3, v0.10.0) adds restarts (`withRestarts` with functions and
arguments, `computeRestarts`, `findRestart`, `invokeRestart(r, ...)`,
`tryInvokeRestart`), the call stack (`sys.nframe`, `sys.calls`, `sys.frames`,
`sys.frame`, `sys.on.exit`), `Recall`, `match.call` (positional, partial and exact
matching, `...` expanded from the caller, `expand.dots`), `errorCondition`,
`warningCondition`, `geterrmessage`, and `$` on a call. A calling handler or
`finally` no longer changes the visibility of the value.

**Slice 6** (M3, v0.9.0 and v0.9.1) adds string functions: `strrep`, `substr<-`,
`strtoi`, `make.unique`, `basename`, `dirname`, `formatC` (formats `d`, `f`, `e`,
`g`, `s`; flags `-`, `0`, `+`, space; `big.mark`; negative width), `prettyNum`,
`toString`, `sQuote`/`dQuote` (curly quotes, as the oracle's UTF-8 locale gives),
`shQuote`, `noquote` and `print(quote = FALSE)`, `file.path`; part 2 adds
`format(big.mark =)`, `formatC(mode =, format = "fg")`, `shQuote(type = "csh"/"cmd"/"cmd2")`
and `match.arg`, and `sprintf("%#x", 0)` prints no prefix, as C does. Text produced
by `sprintf`, `formatC` and `strrep` counts against `MaxAlloc`.

**Slice 5** (M3, v0.8.0) starts the base library. `eval/baselib/*.R` is base R's
glue written by us, BSD, under the clean-room rule of the M3 plan (`isTRUE`,
`isFALSE`, `xor`, `Negate`, `setNames`, `union`, `intersect`, `setdiff`,
`is.element`, `append`, `head`, `tail`, `ifelse`, `nrow`, `ncol`, `NROW`, `NCOL`,
`lengths`, `rep_len`, `diff`, `outer`), embedded, parsed once and evaluated into
the base environment of each session (43 µs per `NewSession`). Go natives:
`as.vector`, `order`, `rank` (`average`, `first`, `min`, `max`), `pmax`, `pmin`,
`which.max`, `which.min`, `tabulate`, `rowSums`/`colSums`/`rowMeans`/`colMeans`,
`t`, `diag`, `mean`, `stopifnot`.

`matrix`, `array`, `letters` and `LETTERS` are there for the tests. Each
primitive and each of the 110 closures (`print`, `cat`, `paste`, `tryCatch`,
`structure`, `inherits`, ...) is exercised by the goldens; the rest of the
27 primitives that remain (since v0.15.0; 35 before the complex and raw types) are refused with an error that says why. Errors and
warnings are R's own: `Error in f(1) : object 'y' not found`.

## Judged by the oracle

| Check | Result |
| --- | --- |
| goldens: 5 838 snippets (9 of them recorded for macOS only, `testdata/eval/platform_darwin.R`, and compared only there) of `testdata/eval` (matrices, arrays and grouping, vectors, math group, language primitives, factors and matching, environments, apply, strings, regular expressions, arithmetic, comparison, sequences, coercion, summaries, closures, matching, laziness, loops, subsetting, assignment, attributes, S3, conditions, printing, errors), each compared with R 4.6.2's serialized value (byte for byte), visibility, printed output, standard error, warnings, and error message and call; replayed by `go test` without R | **5 838 / 5 838** (v0.21.0; 5 802 at v0.20.0, 5 265 at v0.19.0, 4 498 at v0.18.1, 3 733 at v0.17.0, 3 289 at v0.16.0, 3 048 at v0.15.0, 2 737 at v0.14.1, 2 356 at v0.13.1, 2 028 at v0.12.1, 2 022 at v0.12.0, 1 570 at v0.11.0, 1 302 at v0.10.0, 1 202 at v0.8.0, 1 154 at v0.7.0) |
| differential fuzz, environments (`--mode env`; seeds 1, 2, 3, 5 000 programs each): `get`, `exists`, `get0`, `mget` with modes, `inherits` and `ifnotfound`; `assign`, `$<-`, `[[<-`, `local`, `evalq` into locked bindings and environments; `rm`; the lock functions; active bindings; `delayedAssign`; `as.list`, `eapply`, `with`, `within`; the namespace functions; `::`, `:::`; `topenv`, `environmentName`. **Exclusions, generated on purpose and counted (1 858 of the 15 000 programs, not in the figure)**: 473 print an environment's address (compared with the address replaced, 0 differ), 458 print a builtin (our `print` shows no formals; compared as `.Primitive("name")`, 0 differ), 485 use a package R installs besides base (compared exactly: 363 differ, the divergence of slice 8), 442 list a hashed environment unsorted (compared exactly: 210 differ, R's order is undefined) | **13 142 / 13 142** compared, plus the exclusions above |
| differential fuzz, connections and files (`--mode io`; seeds 1, 2, 3, 6 000 programs each, 11 families). **One exclusion, generated on purpose and counted (1 706 of the 18 000 programs, not in the figure)**: non-ASCII text through a connection (compared exactly except for the mark, 0 differ) | **16 294 / 16 294** compared |
| differential fuzz, dates and times (`--mode datetime`; seeds 1, 2, 3, 6 000 programs each): formatting with 40 conversions (`%s` included), parsing, `POSIXlt` fields, round trips through text, `difftime` in every unit, arithmetic and comparison on `Date`/`POSIXct`/`difftime`, `seq` by day, week, month, quarter, year, `c`, `rep`, `rev`, `sort`, `unique`, time zone conversion in 8 zones; years 1971 to 2037, never 01:00 to 03:59 local. **Exclusions, generated on purpose and counted (1 848 of the 18 000 programs, not in the figure)**: 462 in years before 1902 (R warns once per session, so the warning is not compared; the rest is, 0 differ), 463 in years after 2037 in zones with daylight saving (compared exactly, 0 differ), 445 `tzone` attributes of zones with unusual daylight abbreviations (compared exactly, 56 differ), 478 local times that a daylight-saving change skips or repeats (compared exactly, 64 differ) | **16 152 / 16 152** compared, plus the exclusions above |
| differential fuzz, `sprintf` and `format`, extended with 20 more formats (v0.9.1; seed 7; no exclusions) | **10 000 / 10 000** |
| differential fuzz, strings part 2: `strrep`, `substr<-`, `strtoi`, `make.unique`, `basename`, `dirname`, `formatC`, `prettyNum`, `toString`, `sQuote`/`dQuote`/`shQuote`, `noquote`, `file.path` (`--mode strings2`; seeds 1, 3; no exclusions) | **16 000 / 16 000** (after the first run's 255 differences in 6 000 were fixed) |
| differential fuzz, arithmetic, logic and summary (`go run ./cmd/rparity evalfuzz`; seeds 3, 6, 7, 8) | **189 998 / 190 000** |
| differential fuzz, subsetting and assignment on vectors, lists, matrices and arrays (`evalfuzz --mode subset`; seeds 7, 11, 21) | **35 963 / 36 000** |
| differential fuzz, regular expressions: random POSIX extended patterns, with lazy quantifiers, back-references, `\<` `\>` `\b` and non-ASCII text, through `grepl`, `sub`, `gsub`, `regexpr`, `gregexpr`, `strsplit`, `regmatches` (`--mode regex`; seeds 1, 7, 21; nothing excluded) | **60 000 / 60 000** |
| differential fuzz, character functions: `nchar`, `substr`, `substring`, `toupper`, `trimws`, `strsplit`, `sprintf`, `format`, `chartr` (`--mode strings`; seeds 1, 5, 21) | **20 000 / 20 000** |
| differential fuzz, `lapply`, `sapply`, `vapply`, `Map`, `mapply`, `Reduce`, `Filter`, `unlist`, `rep`, `seq`, `rev` (`--mode apply`; seeds 3, 21) | **18 000 / 18 000** |
| differential fuzz, vectors: `as.vector`, `order`, `rank`, `pmax`, `pmin`, `which.max`, `tabulate`, row/column sums and means, `t`, `diag`, `outer`, `mean`, set operations, `head`, `tail`, `ifelse`, `diff`, `append` (`--mode vec`; seeds 1, 5; no exclusions) | **18 000 / 18 000** |
| differential fuzz, the math group (`--mode math`; seed 9, 20 000 expressions). **Comparison classes, counted:** exact for everything except 5 644 libm cases (trigonometric, hyperbolic, logarithmic, `gamma`, `expm1`, `log1p`, `log(x, base)`) compared **within 4 ulp**, where the largest difference seen is 2 ulp (`log`, `log10`) and 1 ulp for the others; and 335 gamma-family cases (`lgamma`, `digamma`, `trigamma`) compared through `signif(x, 12)` because our routines are less accurate than R's (open [issue 14](https://github.com/go-embedded-r/r/issues/14)). `sinpi`, `cospi`, `tanpi` agree exactly | **20 000 / 20 000** |
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

Of slice 9 (v0.12.0 and v0.12.1):

- The C locale only (English names, `%x` is `%m/%d/%y`, `%c` the C form; `%E`
  and `%O` modifiers read as the plain conversion).
- R's warning `datetimes before 1902 may not be accurate` (once per session) is
  never given.
- The daylight abbreviation in the `tzone` attribute of a zone with unusual or no
  daylight time is found by scanning Go's transitions and differs from R's for
  some zones (56 of 445 fuzz cases: the attribute has three elements, the first is
  always right).
- Of the local times that a daylight-saving change skips or repeats, ours read as
  R's in most cases, and 64 of 478 fuzz cases differ.
- `%s` is not the epoch time on macOS: R gives the fields read as UTC unless the
  format also has `%z` or `%Z`, in which case it subtracts the zone's offset. That
  is reproduced exactly (the 9 goldens of `platform_darwin.R`, the fuzz). Whether
  glibc does the same is not known, so those goldens are not compared on Linux.
- The call shown in an error from a long `structure(...)` breaks its lines where
  R does not.
- `Date` and `POSIXct` have no `round`, `trunc`, `cut`, `summary`, `seq.POSIXt`,
  `Sys.setenv`, `Sys.timezone`, `OlsonNames`, and date columns in data
  frames were not checked in v0.17.0.

Of slice 8 (v0.11.0):

- **There is no package loading**: no `library`, `require`, `loadNamespace`,
  `attach`, `search`, `sys.source` (planned with the slices that need them).
- The installed packages other than base (`stats`, `utils`, ...) resolve through
  our base environment (`utils::head` works, `stats::sd` is "'sd' is not an
  exported object"), and any other name is `there is no package called ‘x’`.
- `parent.env(globalenv())` is base; R's is the last attached package.
- `as.list.environment` and `names(env)` list in creation order, where R's hashed
  order is unspecified.
- `sys.call()` and `parent.frame()` inside `eval`/`local` do not see R's eval
  frame.
- The text `requireNamespace("foo")` writes without `quiet` was read from the
  oracle, not goldened (its harness adds a connection warning).
- `print` of an environment does not list its attributes.
- `getNamespace(<environment>)` raises R's message with another call.
- `within` has no data frame method.

Of slice 7 (v0.10.0), all because R runs `tryCatch` and `try` through closures
we do not have:

- the `conditionCall` of a `stop()` made directly inside `try()` or `tryCatch()`
  at top level is empty here, and R's internal call there (`doTryCatch(...)`,
  `try(...)`);
- `match.call(expand.dots = FALSE)` keeps `...` as a list where R keeps a pairlist
  (`deparse` shows `list(` for `pairlist(`);
- absolute frame numbers differ from the oracle's, which runs each snippet inside
  its own harness frames, so the goldens compare numbers of frames relative to a
  starting point;
- `computeRestarts` has no top-level `abort` restart.

Of slice 5 (v0.8.0):

- **A matrix whose `dimnames` carry names** (`names(dimnames(x))`, as `table()`
  makes) prints without the variable-name header lines R adds.
- **`lgamma`, `digamma` and `trigamma` are less accurate than R's**: an open
  defect, [issue 14](https://github.com/go-embedded-r/r/issues/14); the fuzz
  compares them through `signif(x, 12)` and counts them.
- **`mean` sums in `float64`** (with R's second pass) where R uses `long double`:
  last-bit differences are possible on long sums.
- The math fuzz compares libm functions within 4 ulp (largest seen: 2), not
  exactly: the last bit of those functions depends on the C library.

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

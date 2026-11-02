---
title: "The parser"
weight: 20
description: "Package syntax (M1): the API, the tree, the errors, the bounds, TopLevelNames, number parsing, and the parity table with R 4.6.2."
---

<!-- From the README of go-embedded-r/r at v0.1.0, section "M1: the parser". Change the repository's README first, then here. -->

The package is `github.com/go-embedded-r/r/syntax`, released in
[v0.1.0](https://github.com/go-embedded-r/r/releases/tag/v0.1.0).

```go
f, err := syntax.ParseAll(src)                    // or ParseAllLimits(src, syntax.Limits{MaxBytes, MaxDepth, MaxNodes})
names, err := syntax.TopLevelNames(cell)          // names a cell assigns and reads
```

## The tree

R's own shape: operators, `if`/`for`/`{`/`[` and the rest are calls to a
symbol; `Function` carries the formals; `|>` is rewritten as R does; `->` is
`<-` with the sides swapped; a string callee is a symbol. Every node has a
`Span` (line, column and byte range). `File.Tokens` are the terminals of
`getParseData()` (`SYMBOL_FUNCTION_CALL`, `SYMBOL_SUB`, `EQ_FORMALS`,
`COMMENT`, `LINE_DIRECTIVE`...).

## Errors

`*syntax.Error` has `Line`, `Col` and `Message`, and its text is the message
`parse()` signals, byte for byte, source context and caret included. Limits
give a `*syntax.LimitError` (wraps `ErrLimit`). The statements before an error
are kept in the returned `File`.

## Bounds

Bytes, tree depth (also for chains such as `a+a+a+...`, built by a loop) and
node count; R's own 50-level context stack (`contextstack overflow`). Bomb
tests (200 000-deep chains of every shape) assert a refusal under a memory
limit.

## TopLevelNames

- `Assigned`: `<-`, `=`, `->`, `"x" <-`, `f(x) <-`, `x$a <-` (assign x),
  `assign("lit", ...)`, `for` variables.
- `Read`: free variables and called names not bound earlier in the cell, under
  R scoping (function parameters and locals, `local()`; a function body may use
  anything the cell assigns anywhere).
- `Dynamic` (with `Reasons`) when names cannot be resolved: `<<-`, assign/get
  with a computed name or another environment, eval, parse, source, attach,
  library, require, sys.function, `environment<-`, load, list2env...

## Numbers

A decimal literal is computed in double precision, digit by digit with an
explicit fused multiply-add (`math.FMA`), then scaled by a power of ten: what R
does on macOS arm64, where `long double` is `double` and the compiler fuses
`10*ans + digit`. 150 000 random literals are identical to that R's, bit for
bit, and the result is the same on every platform. R on x86-64 Linux
accumulates in 80-bit extended precision and can differ by one unit in the
last place on literals of more than 15 digits (the CI oracle lane saw it on
`1e300`), and by up to 6 ulp on literals near the subnormal range (measured on
R's own `tests/`: 290 literals differ, at most by 6 ulp). The parser is
deterministic and matches the 64-bit reference; the platform-dependent
literals are in `testdata/numbers.R`, compared within 16 ulp and kept out of
the goldens. One literal differs by more: `1.7976931348623157e308` (the
largest double) is `Inf` in 64-bit-double R and in the Go parser, and is kept
by 80-bit R. Hexadecimal literals follow R's rule (`0x1.8p1`).

## Parity with R 4.6.2

Judged by the oracle, never by expectations written by hand:
`go run ./cmd/rparity ...` parses the same text with R and with Go and compares

1. the terminal tokens and positions with `getParseData()`;
2. the tree with a dump of the language objects of `parse(keep.source = FALSE)`
   (names, constants as IEEE bits, empty arguments, formals);
3. the error text with `parse()`'s.

R's own sources are read where they are unpacked and never committed.

| Corpus | Inputs | Tree | Tokens | Error text, byte for byte |
| --- | --- | --- | --- | --- |
| R `src/library`: every `.R` file of the base packages (7.8 MB) | 844 (842 parse, 2 rejected) | **842 / 842** | **842 / 842** | **2 / 2** |
| R `tests/`: 95 files (1 is not UTF-8 and is skipped) | 94 (90 parse, 4 rejected) | **90 / 90** | **90 / 90** | **4 / 4** |
| our snippets (`testdata/snippets.R`) | 42 | **42 / 42** | **42 / 42** | n/a |
| our error cases (`testdata/errors.R`) | 52 (1 parses) | **1 / 1** | **1 / 1** | **51 / 51** |
| our edge cases: Unicode, raw strings, `\|>`, `\(x)`, `->>`, hex/`L`/`i` numbers (`testdata/edge.R`) | 93 (3 rejected) | **90 / 90** | **90 / 90** | **3 / 3** |
| odd input: CRLF, tabs, form feeds, Unicode spaces, bidi controls, invalid UTF-8, `=>` (`parity.Extras` + `values.R`) | 117 (36 rejected) | **81 / 81** | **81 / 81** | **36 / 36** |
| differential fuzz: 190 000 random edits of lines of R's sources (seeds 1, 2, 11) | 190 000 (about two thirds rejected) | **all** | **all** | **all** |
| decimal literals, random (up to 20 digits, exponents -330..330) | 100 000 | **100 000 identical doubles** | | |

## Speed

One core each, interleaved rounds, median of 11 after a warm-up round,
1-minute load 7.8 on an Apple M-series laptop: the 842 files of `src/library`
that both parse (7.8 MB) take **0.214 s** in Go (`syntax.ParseAll`, which also
builds the source map and the token list) and **0.269 s** in R's
`parse(text =, keep.source = FALSE)`: **Go/R = 0.80**
(`go run ./cmd/rparity bench DIR`).

## Known gaps

The effect of `#line` directives on the line numbers of later errors; a Go
`deparse` printer (the tree is compared structurally); `parse(file=)` encoding
options; an invalid UTF-8 input is parsed as R's lexer does, and for such text
R cannot produce parse data, so tokens are not compared.

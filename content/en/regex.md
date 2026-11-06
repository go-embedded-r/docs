---
title: "Regular expressions"
weight: 29
description: "v0.6.0: R's regular expressions run on a faithful Go port of TRE, R's own quirks reproduced behind RCompat. What is judged by R, what still differs, and the open speed defect."
---

<!-- From the READMEs of go-embedded-r/r and go-embedded-r/tre on main (v0.6.0 and v0.1.0). Change those first, then here. -->

**Since [v0.6.0](https://github.com/go-embedded-r/r/releases/tag/v0.6.0)
([pull request 11](https://github.com/go-embedded-r/r/pull/11), 2026-11-06),
R's regular expressions are TRE.** The matcher is
[github.com/go-embedded-r/tre](https://github.com/go-embedded-r/tre) v0.1.0
(see [The tre module]({{< relref "tre" >}})), a Go port of TRE, the engine R uses
for its default patterns (`perl = FALSE`, `fixed = FALSE`). It replaces the
approximation on Go's `regexp` of v0.5.0, whose lazy quantifiers, capture
priority in repeated groups and `\<`/`\>` differed from R.

`grepl`, `grep`, `sub`, `gsub`, `regexpr`, `gregexpr`, `regmatches` and
`strsplit` use it. Lazy quantifiers, capture priority in repeated groups,
back-references (`(a)\\1`, run by TRE's backtracking matcher) and `\<` `\>` behave
as in R. R's choice of byte mode for all-ASCII input and wide mode otherwise is
reproduced, with its quirks: an error text without the pattern for `grepl` and
`regexpr` in wide mode, and the empty-match tokens of `strsplit`.

## R's own TRE, behind `RCompat`

R 4.6.2 ships a modified copy of TRE. The port's `Options.RCompat` reproduces
the behaviours where that copy differs from upstream (`{,n}`, bound overflow, the
backtracking restart test, shared bracket positions, negated classes in copies of
a repetition), so that `go-embedded-r/r` answers like R. They were found by
diffing R's copy against upstream, used as a specification only: no R source is
copied, and the behaviours are judged by R itself.

## Judged by R

| Check | Result |
| --- | --- |
| R-level differential fuzz: random POSIX extended patterns, with lazy quantifiers, back-references, `\<` `\>` `\b` and non-ASCII text, through `grepl`, `sub`, `gsub`, `regexpr`, `gregexpr`, `strsplit`, `regmatches` (`--mode regex`; seeds 1, 7, 21; each function reported on its own line; nothing excluded) | **60 000 / 60 000** |
| goldens of `testdata/eval`, including regular expressions, against R 4.6.2 | **1 043 / 1 043** |

The port itself is also checked against upstream TRE; see
[The tre module]({{< relref "tre" >}}).

## What still differs

- **Approximate matching** (`{~1}`, `{+n}`): the syntax is not ported and is an
  error.
- **Non-ASCII classes** follow Go's Unicode tables, not a C library's
  `iswctype`, so classes of non-ASCII characters can differ from R's.
- **`perl = TRUE`** (PCRE) is refused: the call stops with an error that says it
  is not supported.
- **Exponential backtracking** is refused by a step limit, not run to the end.
  Some other pathological patterns are refused by limits too (pattern size, AST
  nodes, NFA states).
- **`useBytes`** and the multibyte-locale narrow API are not modelled.

## Speed: an open defect

`grepl` is **1.44 times slower than R 4.6.2** (Go time over R time), measured
when tre v0.1.0 shipped: load 5.3, 5 interleaved rounds, one core, 20 000 lines.
By the fleet rule, slower than the reference is an open defect:
[go-embedded-r/tre issue 3](https://github.com/go-embedded-r/tre/issues/3).
The other cases of the same table: `gsub` with a class 0.87, `gsub` with a group
1.13 (also above 1), `regexpr` 0.92, back-references 0.99.

## Related limits of slice 3

- `format` of a factor is not implemented, and `sort` of non-atomic values is
  not implemented.
- `sort` and `ls` order strings by code point, not by locale.

---
title: "The tre module"
weight: 30
description: "go-embedded-r/tre v0.1.3: a pure-Go port of TRE, BSD 2-Clause with the TRE notice kept, with RCompat for R's own copy."
---

<!-- From the README of go-embedded-r/tre on main (v0.1.0). Change the README first, then here. -->

[github.com/go-embedded-r/tre](https://github.com/go-embedded-r/tre)
([v0.1.0](https://github.com/go-embedded-r/tre/releases/tag/v0.1.0), 2026-11-06,
[pull request 1](https://github.com/go-embedded-r/tre/pull/1)) is a pure-Go port
of [TRE](https://github.com/laurikari/tre), the POSIX regular expression engine
(leftmost-longest, submatch addressing with tags, lazy quantifiers,
back-references, `\<` `\>` `\b` `\w` `\d` `\s`, `(?i)` and friends) that R uses
for its default patterns. It is what
[go-embedded-r/r]({{< relref "regex" >}}) runs since v0.6.0.

```go
re, err := tre.Compile(`(a|ab)(c|bcd)(d*)`, tre.Extended)
m, err := re.ExecString("abcd", 4, 0) // []tre.Match{{0,4},{0,1},{1,4},{4,4}}
```

Offsets are in characters (UTF-8 decoded), or bytes with `tre.UseBytes`.

## v0.1.1

[v0.1.1](https://github.com/go-embedded-r/tre/releases/tag/v0.1.1) (2026-11-07,
[pull request 7](https://github.com/go-embedded-r/tre/pull/7)) adds a lazily
built DFA for yes/no matches (used when no assertion depends on the position)
and a start-position skip in the backtracker. Engine-level `grepl` case: 5.95 ms
to 0.69 ms on 20 000 lines. The speed ratio against R is not yet measured (load
too high); issue 3 stays open.

## v0.1.2 and v0.1.3

Both 2026-11-07. [v0.1.2](https://github.com/go-embedded-r/tre/releases/tag/v0.1.2)
([pull request 8](https://github.com/go-embedded-r/tre/pull/8)) bounds the lazy
DFA and adds a bomb test.
[v0.1.3](https://github.com/go-embedded-r/tre/releases/tag/v0.1.3)
([pull request 9](https://github.com/go-embedded-r/tre/pull/9)) bounds the compile
memory for the wasm32 heap: tighter default limits, a parent-list budget, and a
peak-memory assertion in every bomb test, with no wasm skip. **Some extreme
patterns that R accepts are now refused** (`ErrESpace`; through
go-embedded-r/r v0.8.1 the message is `regular expression '...' is too large for
this evaluator`); see [Regular expressions]({{< relref "regex" >}}).

## Licence

**BSD 2-Clause, with the TRE notice kept** in the repository's `NOTICE` file. It
is ported from upstream TRE at commit `71bfcaf` (`lib/tre-parse.c`,
`tre-compile.c`, `tre-match-parallel.c`, `tre-match-backtrack.c`, `regexec.c`):
parse, AST, tagged NFA, the parallel matcher, and the backtracking matcher used
only when the pattern has back-references.

## RCompat

`Options.RCompat` reproduces the behaviours of R 4.6.2's modified copy of TRE
where they differ from upstream, so that `go-embedded-r/r` answers like R:

- `{,n}` keeps a minimum of -1, so it matches n+1 repetitions (`^a{,3}$` matches
  `aaaa`); a bound that overflows an int means "unbounded, or 1"; a bound above
  255 is "Invalid contents of {}"; `{m,n}` expands to `(|x(|x))` rather than
  `(x(x)?)?`.
- An empty `REG_LITERAL` pattern matches the empty string.
- The backtracking matcher decides whether to retry from a later start position
  by the current character instead of the character at the start of the attempt:
  `([abc]+)\w*\1` does not match `banana` in R.
- The items of a bracket expression (and the two cases of an `(?i)` literal)
  share one NFA position.
- `tre_copy_ast` forgets the negated classes of a copied literal: in wide mode
  `[^[:space:]]{1,3}` matches a space on its repetitions.
- Messages and limits: no `TRE_MAX_RE`, no `REG_BADMAX`, `assert` becomes an R
  error, one extra text for `tre_version`.

R's changes are GPL and were used as specification only; the behaviours are
re-implemented and judged by R itself.

## Resource limits

Pattern size, AST nodes, NFA states and transitions, the size of the
firstpos/lastpos sets, and match steps. A hostile pattern or subject gets an
error (`ErrESpace`, `ErrSteps`), never an out-of-memory or a hang.

## How it is checked

- **TRE's own tests**: upstream's `tests/retest.c` is run through a recording
  wrapper and replayed against the port: 286 compilations and 3514 executions
  (wide) and 3510 (narrow) with byte-for-byte equal return codes and match
  offsets. Excluded and counted: the 4 narrow-API patterns of retest's EUC-JP
  section (20 executions), which run in a multibyte locale the port does not have.
- **Differential fuzz** against upstream TRE built as C: 197,588 compared, 0
  mismatches. Excluded and counted: 2,410 approximate-matching patterns, 1
  backtracking case cut by the step limit.
- **R 4.6.2** for the R-level behaviour, through the regex fuzz of go-embedded-r/r
  (60 000 / 60 000, see [Regular expressions]({{< relref "regex" >}})).
- A fuzz target, bomb tests that assert refusals under `GOMEMLIMIT` and `ulimit
  -v`, `govulncheck`, 3 OSes with `-race`, 386, every GOOS, wasm.

## Not ported, and open issues

- **Approximate matching** (`{~n}`, `{+n}`, `{1i+1d<2}`): a pattern using it fails
  with `ErrApprox`.
- The C library's system-wctype, multibyte-string and user-string front ends: one
  engine over `[]rune` code units replaces them. Character classes use Go's
  `unicode` tables, so classes of non-ASCII characters can differ from a given C
  library's `iswctype`.
- Open: [issue 3](https://github.com/go-embedded-r/tre/issues/3), `grepl` is 1.44
  times slower than R 4.6.2; [issue 4](https://github.com/go-embedded-r/tre/issues/4),
  upstream TRE reads one byte past the pattern in `tre_regncompb`.

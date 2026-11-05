---
title: "Regular expressions"
weight: 29
description: "v0.5.0: regex on Go's regexp, an approximation of TRE and not a port. What differs from R, what is refused, and the planned TRE port."
---

<!-- From the README of go-embedded-r/r on main (v0.5.0), section "M2", slice 3. Change the repository's README first, then here. -->

**The regular expressions of v0.5.0 are an approximation of R's engine, not
TRE.** R's default engine, TRE, is not ported. The functions `grepl`, `grep`,
`sub`, `gsub`, `regexpr`, `gregexpr` and `regmatches` are implemented, but where
R uses TRE this implementation uses Go's standard `regexp`, and the answers can
differ from R's. A faithful TRE port is in progress and planned for v0.6.0; it
is not released.

## How it works

TRE's syntax (bounds, `\<`, `\>`, POSIX classes, ignored leading repetition
operators, `(?i)`, TRE's own error texts and the warning `TRE pattern
compilation error`) is translated onto Go's `regexp` (BSD-licensed, RE2, linear
time, no catastrophic backtracking) in POSIX leftmost-longest mode. No TRE
source and no R source is copied, so no notice beyond Go's is needed.

## Where it differs from R

What it cannot do is refused or documented, never answered wrongly on purpose:

- **Back-references** (`(a)\\1`): refused. The call stops with an error that says
  so.
- **`perl = TRUE`**: refused. The call stops with an error that says it is not
  supported.
- **Lazy (non-greedy) quantifiers** (`*?`): Go's leftmost-first semantics apply,
  where TRE picks different matches in some patterns. They are excluded from the
  fuzz.
- **Capture priority in repeated groups**: sub-match capture inside a repeated
  group can differ from TRE's POSIX rule (7 of 36 000 fuzz cases).
- **`\<` and `\>`**: both are treated as word boundaries, exact when followed or
  preceded by a word character.

## Judged by the oracle

Random POSIX extended patterns through `grepl`, `sub`, `gsub`, `regexpr`,
`gregexpr`, `strsplit` and `regmatches`, R 4.6.2 against Go (`go run
./cmd/rparity evalfuzz --mode regex`; seeds 1 and 21): **17 993 / 18 000**
identical. Lazy quantifiers are not in that corpus.

## Related limits of slice 3

- `format` of a factor is not implemented, and `sort` of non-atomic values is
  not implemented.
- `sort` and `ls` order strings by code point, not by locale.

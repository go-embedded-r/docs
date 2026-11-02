---
title: "Licences"
weight: 40
description: "BSD-3-Clause, and the clean-room rule: R is an oracle, never a source."
---

<!-- From the README of go-embedded-r/r at v0.1.0, section "Licence, and what clean room means here". Change the repository's README first, then here. -->

`go-embedded-r/r` is **BSD-3-Clause** (see its
[LICENSE](https://github.com/go-embedded-r/r/blob/main/LICENSE)). The sources
of R are GPL-2-or-later, so:

- **R is an oracle, never a source.** R is run as a black box and what it
  prints is recorded. No R source file, no R test file, no R help page, no
  `.Rout.save` transcript and no text of R's manuals is copied into the
  repository, and none is downloaded at run time. The golden files are the
  answers of R 4.6.2 to inputs written there.
- **Base R's R code is rewritten, not copied.** The R-language parts of the
  base packages (`base`, `stats`, `utils`, `methods`, ...) will be
  re-implemented under the BSD licence from the documented behaviour and from
  what the oracle does. Fetching the GPL code on first use, and a GPL port,
  were considered and rejected (decision of 2026-11-01, recorded in the
  roadmap).
- **Contributors work from specifications**: the *R Language Definition*, the
  *R Internals* manual, the help pages, and the oracle's behaviour. Do not
  open `src/main`, `src/nmath` or the R code of the base packages while writing
  the counterpart. See
  [CONTRIBUTING.md](https://github.com/go-embedded-r/r/blob/main/CONTRIBUTING.md)
  and [NOTICE](https://github.com/go-embedded-r/r/blob/main/NOTICE).

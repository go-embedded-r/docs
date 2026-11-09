---
title: "Files and connections"
weight: 31
description: "Slice 10 (v0.13.0, v0.13.1): connections, files, serialization, environment variables; the file system model and the security guarantees an embedder can rely on; the bounds; the divergences."
---

<!-- From the README of go-embedded-r/r on main (v0.13.1, slice 10) and the bodies of pull requests 33 and 34. Change those first, then here. -->

Slice 10 of M3 adds connections and files to package `eval`, in
[v0.13.0](https://github.com/go-embedded-r/r/releases/tag/v0.13.0)
([pull request 33](https://github.com/go-embedded-r/r/pull/33)); v0.13.1
([pull request 34](https://github.com/go-embedded-r/r/pull/34)) is the security
audit of it (bounds on path length and depth, and tests for Windows path forms, a
symbolic-link swap race, `Sys.sleep` limits and bounded `readRDS` and parser
input). Base coverage goes from 462 to 513 of 1 362 exports, **33.9 % to
37.7 %**; 2 356 goldens agree with R 4.6.2 (328 of them new).

## Security: what an embedder can rely on

These are properties of every `eval.Session`, tested and not just intended.

**The file system is private by default.** A session has an in-memory tree that
the host never sees and that no other session shares. R code running in it cannot
read or write a host file, and the cell's `getwd()`, `tempfile()` and `list.files()`
all stay inside the tree.

**`Options.FileRoot` is the only way to reach the host's disk**, and it is
confined. It gives the session a host directory as its root, opened with Go's
`os.OpenRoot` (`os.Root`). Tests cover:

- `..` (it is clamped at the root);
- absolute paths (they are read as relative to the root);
- symbolic links that point out of the root (they cannot leave it);
- a symbolic-link swap race (a path swapped for a link between the check and the
  use);
- Windows path forms.

`Session.Close` releases the root. Do not point `FileRoot` at a directory that
holds anything the cell must not read: the confinement is to that directory, not
inside it.

**The host's environment is never read.** `Sys.getenv` reads a table of the
session's own, `Options.Env`; nil means empty. `Sys.setenv` and `Sys.unsetenv`
change that table only.

**Everything is bounded.**

| Bound | Value |
| --- | --- |
| `Options.MaxFileBytes` | default `DefaultMaxFileBytes`, 256 MiB (`eval.Unbounded` removes it): the in-memory file system, any one file read or written under `FileRoot`, and what a text connection holds; past it the call fails with `the file size limit of this session was reached` |
| entries in the in-memory tree | 65 536 |
| path length | 4 096 bytes (v0.13.1) |
| path depth | 256 levels (v0.13.1) |
| open connections | 128 |
| sinks | 20 |
| `readRDS` | refuses a document that expands past the limit (gzip bombs) |
| `sink` output and `capture.output` | bounded; diverted output never reaches the `Result` |
| `Sys.sleep` | obeys `Timeout` and the context |

The bomb and boundary tests (`eval/io_test.go`) assert these refusals under a
memory limit.

## Connections

`file` (modes `r`, `w`, `a`, with `b`, `t` and `+`), `textConnection` (read;
write and append to a variable, `local`, anonymous with `textConnectionValue`; the
variable is updated line by line), `stdin`, `stdout`, `stderr`, `open`, `close`,
`flush`, `isOpen`, `isIncomplete`, `summary` (generic, with `summary.connection`
and `print.connection`), `readLines` (`n`, `ok`, `warn`; LF, CRLF and CR; the
incomplete-line warning), `writeLines` (`sep`), `readline` (non-interactive: it
writes the prompt and returns `""`), `nullfile`, `isatty`.

`cat` takes `file` (a path or a connection) and `append`, and places its
separators as R does. `sink` (a stack of 20, `append`, `split`,
`type = "message"`) and `capture.output` divert the evaluator's own output, so what
a cell prints reaches a connection and never the `Result`; a sink left open stays
across cells.

## Files

`file.exists`, `dir.exists`, `file.create`, `file.remove`, `file.rename`,
`file.copy`, `file.append`, `file.size`, `unlink` (wildcards, `recursive`),
`dir.create` (`recursive`), `list.files`/`dir` (`pattern`, `all.files`,
`full.names`, `recursive`, `ignore.case`, `include.dirs`, `no..`, `fixed`),
`list.dirs`, `tempfile`, `tempdir`, `getwd`, `setwd`, `normalizePath`,
`path.expand`, with R's warnings and messages.

## Serialization, code, environment variables and time

`readRDS` and `saveRDS` over package `rds` (gzip, `compress = FALSE`,
`version = 2`; vectors, lists, attributes, factors, dates, calls); `source`
(`local`, `print.eval`) and `sys.source`; `Sys.getenv`, `Sys.setenv`,
`Sys.unsetenv`; `Sys.sleep`.

## Judged by the oracle

| Check | Result |
| --- | --- |
| goldens (`testdata/eval/connections.R` is the new file) | **2 356 / 2 356** |
| differential fuzz, connections and files (`--mode io`; seeds 1, 2, 3, 6 000 programs each, 11 families: text connections read and written, `cat` separators, files by `cat`/`writeLines`/`readLines` and by connection, the file-system functions in a temporary directory, `sink` and `capture.output`, `readRDS`/`saveRDS` round trips, `Sys.setenv`/`Sys.getenv`, `source`; a path never reaches the output). **One exclusion, generated on purpose and counted (1 706 of the 18 000 programs, not in the figure)**: non-ASCII text that goes through a connection (R leaves it unmarked, ours is marked UTF-8; compared exactly except for the mark, 0 differ) | **16 294 / 16 294** compared. The first run found 215 differences in 1 000 programs (`cat`'s separators, `textConnection(open = "a")`'s warning, a tag on a nested object); they were fixed |

## Divergences

- A connection is an integer with its class and no `conn_id` attribute, numbered
  from 3 (R's are numbered from what is open); `stdout()` is always connection 1
  (R's is the current sink).
- Strings read from a connection are not marked unknown as R marks them, so
  `Encoding()` and the serialized bytes of non-ASCII text differ (the fuzz counts
  these, 1 706 programs).
- The variable of an output text connection is not locked.
- Not provided yet: `file("")` (the anonymous file), `gzfile`, `bzfile`, `url`,
  `pipe`, `fifo`, sockets, `rawConnection`, `readBin`, `writeBin`, `readChar`,
  `scan`, `seek`, `pushBack`, `showConnections`, `closeAllConnections`,
  `file.info`, `file.mtime`, `Sys.glob`, `parse`.
- `saveRDS` refuses closures, environments (other than the four standard ones),
  `ascii = TRUE`, and `bzip2`/`xz`, which R writes; `readRDS` reads a pairlist as
  a list and refuses environments, closures and S4 objects.
- `source` refuses `echo`, `verbose` and `exprs` (no expression vectors).
- `Sys.getenv()` with no argument returns a plain named vector (R's has class
  `Dlist`); `Sys.setenv(TZ = )` does not change the zone the date functions use
  (they still read the process's `TZ`).
- `path.expand` leaves `~` alone; `file.size` of a directory is 4 096.
- `summary` has no default method, so `summary(1:3)` says there is no applicable
  method where it said it could not find the function.
- A sink's output is moved to its connection when the evaluator next checks its
  limits or runs a connection function, so it is not copied while it piles up.

Found by the fuzz and fixed on the way: `cat` separated by position instead of by
argument; an element of a list that is an object printed under its `[[i]]` tag
where R's default method prints none.

## Clean-room record

Only `eval/baselib/connections.R` is new R code (`summary`, `print.connection`,
`capture.output`); everything else is Go, written from the oracle's behaviour. Its
similarity-gate scores (exact containment / normalised containment / longest exact
run, see [Clean room]({{< relref "clean-room" >}})): first draft **0.00 / 0.43 /
13**, submitted **0.00 / 0.42 / 13**, both under the gate (0.20 / 0.60 / 40). The
gate is a floor, not proof; the pull request records the provenance.

---
title: "Eval API"
weight: 28
description: "Package eval: NewSession, Options and its default limits, Eval, EvalContext, EvalIn, Get and Set, Result and Error."
---

<!-- From the README of go-embedded-r/r on main (v0.14.1) and the doc comments of eval/session.go. Change the repository first, then here. -->

The package is `github.com/go-embedded-r/r/eval`, in
[v0.14.1](https://github.com/go-embedded-r/r/releases/tag/v0.14.1) (bounded by
default since [v0.4.0](https://github.com/go-embedded-r/r/releases/tag/v0.4.0)).

```go
s := eval.NewSession(eval.Options{})   // bounded by default since v0.4.0
res, err := s.Eval("f <- function(x, y = 2) x * y; f(21)")            // one notebook cell
res, err = s.EvalContext(ctx, src)                                    // the same, cancellable
res.Value, res.Visible        // the last value, whether R would print it
res.Output, res.Stderr        // what print/cat and the auto-print wrote; message() output
res.Warnings                  // []eval.Warning{Message, Call}
// err is *eval.Error{Message, Call, Condition, Output, Stderr, Warnings} (R's own text),
// *syntax.Error or *syntax.LimitError; a timeout or a cancel unwraps to context.DeadlineExceeded / Canceled
s.Get("x"); s.Set("x", v)     // the global environment persists from one cell to the next
```

## Session

`NewSession(Options)` makes one interpreter: a global environment that persists
from one `Eval` to the next, so names assigned by one cell are seen by the next.

**A `Session` is not safe for concurrent use.** Cells run one at a time; an
embedder that serves several callers uses one session per caller, or serialises
the calls itself.

## Evaluating

- `Eval(src)` parses the source and evaluates its expressions in order in the
  global environment, as one notebook cell. The result is that of the last one.
- `EvalContext(ctx, src)` is the same, and ends with an `*eval.Error` wrapping
  the context's error when `ctx` is done.
- `EvalIn(ctx, src, env)` is `EvalContext` in the given environment, a child of
  the global one for a cell that must not leave names behind.
- `Get(name)` and `Set(name, v)` read and write the global environment.

## Limits: bounded by default

**Since v0.4.0 the zero `eval.Options{}` is bounded.** This is a breaking
change: before it, `MaxSteps`, `MaxAlloc` and `Timeout` were unbounded unless
set. A zero field now selects its default:

| Option | Meaning | Default |
| --- | --- | --- |
| `MaxSteps` | calls and loop iterations evaluated by one `Eval` | **30 000 000** |
| `MaxAlloc` | bytes of vector storage allocated in total by one `Eval`, copies included | **16 GiB** |
| `Timeout` | time one `Eval` may take; a context given to `EvalContext` bounds it too | **10 s** |
| `MaxDepth` | nesting of evaluations (R's `expressions`); beyond it, `evaluation nested too deeply` | **5000** |
| `MaxVector` | length of any vector the runtime allocates | **2^28** elements |
| `Parse` | a `syntax.Limits` bounding the source text of an `Eval` | see `syntax.Limits` |
| `MaxFileBytes` | (v0.13.0) the in-memory file system, any one file read or written under `FileRoot`, and what a text connection holds | **256 MiB** (`DefaultMaxFileBytes`); `eval.Unbounded` removes it |
| `FileRoot` | (v0.13.0) a host directory as the session's file-system root, confined with `os.OpenRoot` | **empty: a private in-memory tree** the host never sees; `Session.Close` releases a root |
| `Env` | (v0.13.0) the environment variables the session sees through `Sys.getenv` | **nil: empty**; the host's environment is never read |

The file-system and environment options are described, with the security
properties they give an embedder, on [Files and connections]({{< relref "files-and-connections" >}}).

The first three defaults were measured, not guessed (`RGO_MEASURE=1 go test -run
TestMeasureLimits -v ./eval`): the 1 035 goldens of v0.5.0 need at most 8 879 steps, 0.8 MB
and 3 ms, and the heaviest cells a notebook runs on purpose (a loop of 10^6
iterations: 3 million steps, 0.17 s; a list grown by 20 000 assignments: 1.6 GB
of copies, 0.63 s) set the scale. Each default is about ten times that.

A limit that is hit ends the evaluation with an R error that names it and leaves
the session usable.

**Opting out is explicit.** To remove a limit, say so: `MaxSteps:
eval.Unbounded` (also for `MaxAlloc` and `Timeout`; `eval.Unbounded` is `-1`).
A zero never means unbounded, and any other negative value panics in
`NewSession`. Use `Unbounded` only for trusted code. A context given to
`EvalContext` still stops an unbounded session.

The bomb tests (`while (TRUE) 0`, `numeric(1e12)`, `1:1e12`, unbounded
recursion, an allocation loop) assert a refusal under a Go memory limit, and in
CI under `ulimit -v`.

### `MaxAlloc` is a budget, not live memory

`MaxAlloc` counts every byte of vector storage one `Eval` allocates, copies
included, over the whole evaluation. It is a cumulative budget, not the memory
alive at any moment: a cell can stay far below 16 GiB resident and still spend
it. The 16 GiB default is sized for a native process. **An embedder with little
memory, such as a WebAssembly one, should set `MaxAlloc` much lower.**

## Result and errors

`Result` carries `Value`, `Visible`, `Output` (what `print()` and `cat()` wrote,
and the auto-printed value of every visible top-level expression, as an R
console shows them), `Stderr` (`message()` output) and `Warnings` (the warnings
raised and not muffled, each with its message and call).

An error is an `*eval.Error` with `Message` (R's `conditionMessage`), `Call` (the
deparsed `conditionCall`, empty for none), `Condition` (the condition object
that was signalled; nil for limits and interrupts), and the `Output`, `Stderr`
and `Warnings` the cell had produced before it failed. Its text is R's:
`Error in f(1) : object 'y' not found`. A parse failure is a `*syntax.Error` or
a `*syntax.LimitError`.

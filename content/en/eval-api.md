---
title: "Eval API"
weight: 28
description: "Package eval: NewSession, Options, Eval, EvalContext, EvalIn, Get and Set, Result and Error, and the limits an embedder must set."
---

<!-- From the README of go-embedded-r/r on main (v0.3.0) and the doc comments of eval/session.go. Change the repository first, then here. -->

The package is `github.com/go-embedded-r/r/eval`, in
[v0.3.0](https://github.com/go-embedded-r/r/releases/tag/v0.3.0).

```go
s := eval.NewSession(eval.Options{MaxDepth: 5000, MaxVector: 1 << 28, MaxAlloc: 1 << 30, MaxSteps: 0, Timeout: 5 * time.Second})
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

## Options, and what is off by default

A zero field selects its default. Only two limits have one:

| Option | Meaning | Zero value |
| --- | --- | --- |
| `MaxDepth` | nesting of evaluations (R's `expressions`); beyond it, `evaluation nested too deeply` | **5000** |
| `MaxVector` | length of any vector the runtime allocates | **1 << 28** |
| `MaxAlloc` | total bytes of vector storage one `Eval` allocates; beyond it the evaluation ends with an error | **no bound** |
| `MaxSteps` | number of calls one `Eval` evaluates | **no bound** |
| `Timeout` | time one `Eval` may take; a context given to `EvalContext` bounds it too | **none** |
| `Parse` | a `syntax.Limits` bounding the source text of an `Eval` | see `syntax.Limits` |

**`MaxAlloc`, `MaxSteps` and `Timeout` are off by default.** An embedder that
runs cells it did not write must set them: without them a cell can allocate or
loop for as long as the process lives.

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

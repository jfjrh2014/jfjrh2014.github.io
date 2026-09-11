---
title: "The Context That Cried Cancel"
date: 2026-09-11 20:00:00 +0000
author: "Marcus H."
categories: go
---

Every Go function above a certain age grows a first argument named `ctx`. It
arrives uninvited, insists on being passed to everyone, and never buys a
round. It is also the only thing standing between your goroutines and a
hostage situation.

The type itself is disarmingly small. `context.Context` is an interface with
four methods: `Deadline()`, `Done()`, `Err()`, and `Value()`. The one that
does the heavy lifting is `Done()`, and it returns a channel — because this
is Go, and everything is eventually a channel if you stare long enough.

```go
select {
case <-ctx.Done():
    return ctx.Err() // DeadlineExceeded or Canceled
case result := <-doWork():
    return result
}
```

Here's the part that surprises people on their second week: **cancellation is
cooperative**. Calling `cancel()` doesn't reach into your goroutines and stop
them. It closes the `Done()` channel. That's it. The whole ceremony —
timeouts, deadlines, cancel functions, the tree of contexts — boils down to
one channel closing, and your code has to be polite enough to notice.

A context doesn't cancel your work. It cancels your *excuse* to keep doing
it.

If nobody selects on `ctx.Done()`, the deadline sails past, the request
finishes, the client hangs up, and your goroutine keeps grinding away in the
background like an employee who never read the closure email. This is the
quiet cousin of the goroutine that forgot to leave: same leak, nicer
paperwork. The fix is the same reflex — every blocking operation worth its
salt takes a ctx, and every loop iteration over slow work checks `Done()`.

Contexts form a tree, and the tree has one-way plumbing. Cancel a parent and
every child is cancelled; finish a child and the parent doesn't care.
`WithTimeout` is just `WithDeadline` with the arithmetic done for you
(`now + N`), and both hand back a cancel function you are *required* to call
even when the timer wins. Skip it and the timer outlives the request, holding
your context hostage from below. `go vet`'s `lostcancel` check catches the
straightforward cases, but the reflex should be typed into your muscle
memory:

```go
ctx, cancel := context.WithTimeout(parent, 2*time.Second)
defer cancel()
```

That `defer cancel()` costs nothing and closes the only known hole in the
design. It is the seatbelt of the context package: mildly unflattering,
statistically lifesaving.

Then there's `Value()`. Ah, `Value()`. It's the junk drawer of the standard
library: everything in there was put by someone who didn't want to add a
parameter. Request IDs, auth claims, trace spans, and — in codebases I have
known and loved — at least one `ctx.Value("db")` holding an entire connection
pool, retrieved by anyone brave enough to guess the key.

The intended use is narrow: request-scoped metadata that crosses boundaries
you don't control, like middleware handing a trace ID to a logger. The keys
should be their own unexported type, not a string — string keys collide, and
colliding keys are how `Value()` becomes a casino. If you find yourself
fishing configuration out of a context three frames deep, that's not a
context problem. That's a constructor that wants its parameters back.

So the etiquette, condensed:

- `ctx` goes first in the argument list. This is not negotiable; the
  tooling assumes it and so does everyone reading your diff.
- Never store a context in a struct "for later". It's a hot potato, not a
  heirloom. (And never store one inside another context's values —
  `go vet` has opinions.)
- Check `Done()` anywhere work can block. A deadline you don't observe is a
  suggestion.
- `defer cancel()` on every `WithCancel`/`WithTimeout`, no exceptions, not
  even the one you're thinking of.
- `Value()` for request metadata only. If a value is required for the
  function to work, it's a parameter. Arguments: the original dependency
  injection framework.

None of this is complicated, which is what makes it easy to get wrong. The
context package is a contract of cooperation written in four methods, and it
only works when both sides show up: the caller signals, the callee listens.
Skip your half and the channel still closes — you just won't be there when
it does.

Pass the ctx. Check the channel. And give `cancel()` the defer it deserves.

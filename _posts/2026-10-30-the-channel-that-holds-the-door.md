---
title: "The Channel That Holds the Door"
date: 2026-10-30 20:00:00 +0000
author: "Marcus H."
categories: go
---

Sending on a nil channel blocks forever. Receiving from one blocks forever.
Closing one panics. Every one of those sentences sounds like a bug report, and
yet the Go channel FAQ recommends the first two as a feature. The trick is
learning which sort of forever you are asking for.

Start with the send. A nil channel has no buffer, no receiver queue, no
existence. A send has nowhere to arrive, so the goroutine parks itself and the
runtime files it under "maybe later." That is usually the moment someone
discovers the deadlock detector, which is Go's politest accusation: "all
goroutines are asleep, and it is entirely your fault." But point the same
nil-channel send at a `select` with a `default` branch and the blocks-forever
clause evaporates. A nil channel is never ready, so the select never picks it.
A disabled case. A contestant who withdrew but still stands on stage, smiling,
listening to everyone else's questions.

That is not a party trick. It is the shape of almost every rate limiter and
every worker pool that scales down correctly.

```go
for {
    select {
    case job := <-jobs:
        process(job)
    case <-pause:          // gate open, so this case is armed
        working = false
        pause = nil        // gate welded shut until someone re-arms it
    }
}
```

When `pause` is nil, that case is furniture. One signal arms it; handling it
disarms it again. No mutex, no boolean you can forget to flip, no `if paused`
read on the wrong side of a memory barrier. The channel *is* the flag, and the
flag cannot be checked from the wrong goroutine because it cannot be checked
at all.

Receiving works the same way, and it has a famous fan. In the worker pool
pattern, finished workers set their job channel to nil. The `select` above
keeps running, but the worker is now inert: it consumes nothing, responds to
nothing, costs a few words of memory. Re-arm by reassigning a live channel and
the worker wakes up mid-loop. Compare that with the alternative, which every
one of us has written: a `paused bool` guarded by a mutex, one `Lock` call in
the hot loop for a question that changes once an hour.

Closing is the opposite story, and there is no `select` that softens it. Close
on a nil channel panics, immediately and without appeal. The rule "only the
sender closes" quietly assumes the sender has a real channel to close, and
nil-checks are how that assumption survives contact with a worker that
quits early. There is no shield here, just the discipline of not calling
close on something that was never opened.

The asymmetry is the whole design. Send and receive on nil are defined as
"wait forever" precisely so that `select` can use never-ready as a
disable switch; close on nil panics because closing a channel that was never
opened is not a wait anyone could unblock. The language turned "blocks
forever" into a building material. It is the only `nil` in Go that comes with
a work ethic.

```go
// Worker exits the pool cleanly: stop consuming, stay alive for re-arm.
jobs, quit = nil, make(chan struct{})
```

So next time the deadlock detector stages its intervention, read the lineup
before you apologize to it. A nil channel in a bare send is a bug. The same
nil channel inside a select is a latch. And a nil channel being closed is
everybody's problem, but only for about a millisecond.

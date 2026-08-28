---
title: "The Gopher Who Would Be Freelance"
date: 2026-08-28 20:00:00 +0000
author: "Marcus H."
categories: go
---

Every job ad says it: *Go developer, senior, hybrid.* And every freelance ad
says the opposite: *contract-to-hire, remote, bring your own coffee.* Somewhere
in between stands the freelance Gopher, keyboard in one hand, invoice in the
other, wondering why `go mod tidy` can solve a dependency graph in four
seconds flat but nobody can solve a contract rate table in four weeks.

This is a field guide. Namespace: Go terms. You will be tested at compile
time.

## The `go.mod` Years

A freelance career begins the moment you stop being an employee and become a
module. Here's the thing about `go.mod`: it declares, up top, the minimum
language version you require. Not the maximum. The minimum. Recruiters
haven't read this. They ask for "7+ years of Go," a number reachable only by
people who were writing Go at Cambridge while the language was still on a
whiteboard.

Your own file, similarly, declares what you *need*:

```go
module freelance/gopher

go 1.21

require (
    github.com/clients/unicorn v1.2.3
    github.com/clients/nightmare v2.0.1 // indirect
)
```

Note `nightmare` lives in `// indirect`. Nobody ever directly required
nightmare work. It arrives as a dependency of dependencies, in your `go.sum`
at 2 a.m., and it must be pinned or your whole build breaks. Freelancing is
20% coding and 80% telling PMs why the build broke.

And the golden rule of versions applies to contracts too: **you cannot
downgrade a signed SOW**. MVS just means hourly rates only ever go up.

## `defer` Now or Never

Go's `defer` runs at function exit, not at scope exit. Junior developers
learn this the hard way. Freelancers learn it the hard way too, except the
function is your career and the exit is the handshake.

Every payment term must be a `defer`:

```go
func doWork(client Client) error {
    deliverables, err := ship(client.Spec)
    if err != nil {
        return err
    }
    defer client.Invoice(deliverables) // runs at exit, but log it first
    return client.Integrate()
}
```

Did the integration succeed? Invoiced. Did the client ghost you mid-project?
Invoiced. `defer` does not negotiate, which is more than can be said for
accounts payable, whose runtime keeps *panicking* in a goroutine nobody
waits on. Errors from deferred calls are silently swallowed unless you
check. So check.

## Goroutines and scope creep

We mentioned panicking in a goroutine nobody joins. Let's dwell there,
because "nobody joins it" is the single most common bug in independent
contracting. A goroutine has nothing to join. Nothing to join is what
freelancing *is*. But your work still needs to terminate.

Scope creep arrives as a friend: "while you're in there." And every
freelancer knows the code shape:

```go
for {
    select {
    case <-ctx.Done():
        return ctx.Err()
    case task := <-extra(askedFor):
        // one more "tiny" favor, forever
    }
}
```

Your original contract was the context. The moment "tiny favors" arrive on
the default case, you've leaked a goroutine: doing unbounded unpaid work
with no cancellation in sight. The fix is the same as in production; make
the extra requests a new channel, i.e. a **change request**, with its own
deadline and its own budget. Then `defer cancel()` the old arrangement and
wait for the client to acknowledge.

## What We Learned

Be a good module: declare your minimums, keep your `go.sum` honest, and let
MVS keep your rate on the floor where it belongs.

Invoice like `defer`: automatic, unconditional, checked for errors.

And terminate your goroutines: every "quick favor" without a context
deadline is a leak, and leaked goroutines never merge back to main.

Now if you'll excuse me, my `go vet --verbosity=client-relationship` just
fired. Apparently "I'll pay you soon" fails the format check.

---
title: "The Cleanup Crew That Never Forgets"
date: 2026-09-04 20:00:00 +0000
author: "Marcus H."
categories: go
---

Every language has an opinion about cleanup. C says "you had one job." Java
outsources it to a garbage collector with commitment issues. Go hands you
`defer` — the only feature in the language contractually obligated to show up
at the exit interview.

`defer` schedules a function call to run when the surrounding function
returns. That's the entire trick. `defer file.Close()` reads like what it is:
a promise that the door gets shut, no matter which exit everyone uses.

Three facts make it Go's most polite feature.

**Deferred calls stack up last-in, first-out.** Defer three things and they
run in reverse: the last one scheduled goes first, like a properly organized
fire drill where the people nearest the door got there last. This ordering is
not an accident — it's what makes cleanup compose. Open A, then open B, then
`defer` closes in B-then-A order. Resources unwind exactly backwards from how
they were acquired, like rewinding your steps after sneaking past a sleeping
cat.

**Arguments are evaluated when you defer, not when it runs.** This is the
classic interview question, and the classic bug:

```go
for i := 0; i < 3; i++ {
    defer fmt.Println(i)
}
```

prints `2 1 0`. The *call* is deferred; the *arguments* are photographed at
defer time, like a passport photo — whatever the variable does afterwards,
the picture already exists. If you want the value at run time, defer a
closure and let it capture the variable instead. (Go 1.22 changed loop
variables to be per-iteration, which quietly retired half the blog posts
about this, including the angrier draft of this one.)

**A deferred function can rewrite your return values.** With named results:

```go
func readConfig(path string) (err error) {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer func() {
        if cerr := f.Close(); cerr != nil {
            err = errors.Join(err, cerr)
        }
    }()
    // ... read ...
    return nil
}
```

The deferred closure runs *after* `return nil` computes but *before* the
function actually reports back, so it can edit `err` on the way out. Close
errors stop vanishing into the void. This is also the door `recover()` walks
through to convert panics into errors — the reason `defer func() { ... }()`
appears wherever panics are being bravely translated.

Now the warning label. `defer` is per-*function*, not per-block and not
per-iteration:

```go
for _, f := range files {
    fh, err := os.Open(f)
    if err != nil {
        return err
    }
    defer fh.Close() // every handle lives until the function returns
}
```

Process ten thousand files and you're holding ten thousand open handles at
once — the function returns and the doors all slam at the end, like a hotel
evacuation. The fix is to hoist the body into a helper function so each
`defer` fires each iteration. Small helper, big difference; it's the rare
refactor you can justify as garbage collection.

Last thing: price. For years, `defer` carried a reputation for being slow,
and people wrote cleanup code by hand — full of exactly the early returns
`defer` exists to fix. Go 1.14 made the common cases "open-coded": the
deferred call is compiled straight into the function's epilogue and costs a
few instructions. Today `defer` is cheaper than the argument about whether to
use it.

So yes — the semantics have corners, and you now own all of them. But the
headline stays: `defer` is Go promising that cleanup happens on the way out,
even when the way out is a panic. In a language with fourteen ways to leave a
function early, one keyword guarantees the last word.

And honestly, who among us couldn't use a feature that always shows up at
the end, does its job quietly, and fixes the mess before anyone notices?
`defer` isn't just idiomatic Go. It's a lifestyle.

*— Marcus H.*

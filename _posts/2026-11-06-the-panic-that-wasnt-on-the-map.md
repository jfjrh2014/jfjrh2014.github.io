---
title: "The Panic That Wasn't on the Map"
date: 2026-11-06 20:00:00 +0000
author: "Marcus H."
categories: go
---

Every Go programmer learns the panic story early: something goes wrong, the
runtime prints a stack trace, your program dies, and you reach for `recover`
if you must survive it. The story has a plot hole that most people never trip
over, because the hole sits in a place nobody walks: a goroutine that has
already returned.

Start with the part everyone knows. A panic unwinds the current goroutine.
Deferred functions run, in last-in-first-out order, and the first one to call
`recover` catches the value and the unwinding stops. The goroutine survives.
So does the program, if the panic happened on the main goroutine and you
caught it before it reached the top.

Now the plot hole. Suppose the panicking goroutine has no deferred call that
recovers. The unwinding finishes, and then the runtime finishes *the
goroutine*. Not the program. A panic in a spawned goroutine does not
propagate to its parent, because there is no such thing as a parent. Go
goroutines have no hierarchy; `go f()` starts a peer, not a child. So a panic
in a peer cannot unwind your stack, because it never touches your stack.

Instead it crashes the whole process. No recover on any other goroutine will
see it, because by the time the crash happens, the panicking goroutine is
already dead and the recover mechanism is per-goroutine. A `recover` in main
is not a net under the trapeze act across town. It is a net under one
trapeze, and each goroutine brought its own.

```go
func main() {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("recovered:", r) // never runs for goroutine panics
		}
	}()
	go func() {
		panic("boom") // kills the process, not this defer
	}()
	time.Sleep(time.Hour)
}
```

Run that and the recover line never prints. You get the stack trace and exit
status 2, and if you are unlucky you get it in production, because the test
suite that would have caught it passed. The panic happened in code the test
called directly, where the recover was on the same goroutine and everything
behaved.

The fix is boring and mandatory: every goroutine that runs code you do not
fully control gets its own recover wrapper. Not one shared helper deferred
from main, an actual defer *inside* the goroutine, first statement, before
anything can go wrong:

```go
go func() {
	defer func() {
		if r := recover(); r != nil {
			log.Printf("worker panicked: %v", r)
		}
	}()
	doRiskyWork()
}()
```

This is the reason every respectable worker-pool library and HTTP framework
does exactly that in its connection handlers. Go's own `net/http` recovers
panics in handler goroutines per connection; it does not trust you to do it,
and after the third Friday pager incident, neither should you.

There is a second wrinkle worth knowing. The crash takes the process down
even if thousands of other goroutines are mid-flight doing perfectly fine
work. There is no graceful degradation, no partial survival. It is the
opposite of the nil-channel trick from two weeks ago, where a nil channel in
a select is a polite no-op. A panic in a goroutine is the loudest no-op
possible: the goroutine you spawned has the power to end everything, and you
gave it that power in one five-character statement.

None of this means goroutines are dangerous and should be avoided. It means
the unit of panic containment is the goroutine, not the program, and the
language makes you draw the boundary yourself. Every time you write `go`,
ask what happens when that function panics. If the honest answer is "the
whole process dies and I find out on Monday," you have your homework: one
defer, first line, every time.

Your future self, reading a 3 a.m. stack trace with a coffee, will thank
you. Or at least will have fewer occasions to invent new vocabulary.

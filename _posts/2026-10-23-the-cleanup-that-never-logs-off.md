---
title: "The Cleanup That Never Logs On"
date: 2026-10-23 20:00:00 +0000
author: "Marcus H."
categories: go
---

Every Go programmer learns about `defer` early: it runs when the function
returns. What takes longer to learn is that `defer` is a *promise about the
function*, not a promise about the program. A program is allowed to end
without asking any function to return.

Consider the naughtiest line in the standard library: `os.Exit(1)`. It ends
the process immediately. Deferred functions? Still queued. They never run.
The goroutine holding your lock, the file waiting for its flush, the
connection pending a polite close, all abandoned mid-sentence. `os.Exit`
does not slam the door; the door was never part of the plan.

```go
func loadConfig(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close() // noble. dignified. skipped by os.Exit.

    cfg, err := parse(f)
    if err != nil {
        fmt.Fprintf(os.Stderr, "invalid config: %v\n", err)
        os.Exit(1) // f.Close() silently never runs
    }
    return apply(cfg)
}
```

Is the leaked file descriptor a disaster? One process is about to die, the
kernel reclaims everything anyway. The tragedy is not the leak. It is that
`defer` taught you cleanup is automatic, and `os.Exit` taught the test suite
otherwise.

Nobody notices until a test. Run `go test` and the runtime does the same
thing `os.Exit` does: when the last test finishes, it calls exit directly.
But there is a twist. A test that calls `os.Exit` itself does not merely skip
its deferred functions. It ends the *whole test binary*. The rest of the
test list, silently unexecuted. `go test` reports the package as failed, and
the only diagnostic is that the output stops mid-stream like a novel with
the last pages torn out.

The standard library's own testing docs admit this with a straight face:
calling `os.Exit` in a test or benchmark function terminates the test
entirely. No verdict, no list of survivors. The standard fix is to stop
calling `os.Exit` in library code at all: return errors upward and let
exactly one place, the bottom of `main`, decide to die.

```go
// library: always return, never exit
func Run(ctx context.Context) error { /* ... */ }

// main: the single executioner
func main() {
    if err := Run(context.Background()); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

One exit point, everything above it deferrable. The exit path is the one
function in the codebase that is allowed to be rude, because it is the last
thing that will ever run.

Panic deserves a footnote, because panics do respect `defer`. A panicking
goroutine unwinds the stack and runs every deferred call on the way up,
which is how `sync.Mutex` helpers and file handles survive panics. But only
on the panicking goroutine, and only until the top. If the panic reaches the
top of the stack, the runtime prints the trace and exits, and any deferred
function in *other* goroutines still never hears about it. So `defer` is
reliable within one goroutine's biography and hopeless across the building.

There is one more quiet exit: the default case of a goroutine finishing
while the main function returns. Deferred functions in those goroutines,
never invoked. The Go FAQ says it plainly: goroutines are not stopped
cleanly on program exit. If a worker holds a buffered channel of unsent
data, that data goes wherever abandoned data goes.

So the rule I write on whiteboards now: defer is for state that outlives
your interest but not your process. Config files, mutexes, transactions,
connections, all fine. Logs that must reach a disk before the process dies?
Flush them explicitly. A deferred flush is a wish. An explicit flush is a
line of code that runs.

If you take one thing from this: `defer` schedules work for the end of a
function's life. `os.Exit`, the test binary finishing, and a lone returning
main all end the program without waiting for that appointment. Cleanup is
not a phase of your program. It is a set of appointments your functions
keep, one function at a time, and the process is under no obligation to
drive anyone home.

Wrap your exit paths. Make main the only executioner. And when a test
vanishes without a verdict, before you blame the scheduler, check whether
somebody called `os.Exit` in the middle of it. That novel you were reading
was never missing pages. The author just stopped mid-sentence and the
printer shipped it anyway.

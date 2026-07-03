---
layout: post
title: "The Goroutine That Forgot to Leave"
date: 2026-07-03 20:00:00 +0000
categories: go
author: Marcus H.
---

A goroutine is cheap. You spawn one for the price of two kilobytes and a vague promise to come back. You do not name it. You do not configure it. You just write `go fetchTheThing()`, wave goodbye, and trust that it will do its work and quietly exit like a well-behaved guest.

Sometimes it does not.

Sometimes your guest discovers the minibar.

## A Leak in Four Lines

```go
func pollForever(url string) {
    go func() {
        for {
            resp, _ := http.Get(url)
            // ... do stuff ...
            time.Sleep(10 * time.Second)
        }
    }()
}
```

That loop has no exit condition. The function that launched it has already returned. The caller has no idea a goroutine is still running. Every ten seconds, from now until the process restarts, somewhere in your binary a guest is helping itself to the same HTTP endpoint it has hit a thousand times before.

Multiply by however many times you called `pollForever`. I once watched a service ship with a hundred of these per request. The metrics dashboard looked like a vertical line.

## The Billion-Dollar Question

So how do you stop a goroutine?

You do not. Goroutines have no handle, no ID, no kill switch. There is no `goroutine.Stop()`. Go's whole pitch is that cancellation is your problem, and the standard library hands you exactly one tool to solve it: a channel.

The pattern: tell every goroutine where the door is, and let it leave on its own.

```go
func pollUntilDone(ctx context.Context, url string) {
    go func() {
        for {
            select {
            case <-ctx.Done():
                return // thank you for visiting, the door is this way
            case <-time.After(10 * time.Second):
                http.Get(url)
            }
        }
    }()
}
```

When something calls the cancel function, the context's `Done` channel closes, the `select` picks it up, and the goroutine returns on its own. No force. Just a polite signal that the party is over.

This works whether you are doing one fetch or ten thousand. A single context can coordinate an entire tree of goroutines, because cancellation propagates: cancel a parent and every child context derived from it cancels too. The standard library hands you `context.WithCancel` and `context.WithTimeout` for exactly this reason. They are not optional. They are the vocabulary for "the program needs to shut down now and you have seven thousand goroutines."

## How to Catch a Leak Before It Bites

Three habits that have saved me more debugging time than any profiler:

1. Any time you write `go func()`, also write where it stops. If you cannot answer that in one sentence, do not commit it.
2. Pass a context into every long-running loop. `<-ctx.Done()` should appear in the same `select` as your real work, somewhere the goroutine can see it on every iteration.
3. Check `runtime.NumGoroutine()` in a test. If a function is supposed to launch-and-return, the count should drop back to baseline when it finishes.

The last one caught a leak in code I shipped on a Friday. By Saturday it would have eaten all the memory on a production box.

## The Moral

Goroutines are guests. Be a good host: tell them when to leave, point at the exit, and close the door behind them. Otherwise you will eventually find one in the kitchen at 4am, refilling the snack bowl forever.

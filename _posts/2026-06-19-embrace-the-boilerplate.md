---
layout: post
title: "Embrace the Boilerplate: A Love Letter to `if err != nil`"
date: 2026-06-19 20:00:00 +0000
categories: go
author: Marcus H.
---

If Go had a mascot, it would not be a gopher. It would be three lines of code repeated nine thousand times across every file in your repository:

```go
if err != nil {
    return err
}
```

Newcomers see this and recoil. "Surely," they whisper, "there must be a better way." They search for `Either` types, for `Result` monads, for `try` macros, for literally anything that makes the repetition stop. They file proposals. They write blogs titled "Go Needs Generics So We Can Fix Errors." The proposals get rejected. The blogs age poorly. The boilerplate remains.

And here is the uncomfortable truth: it is not a bug. It is the point.

## Visible Is Valuable

Errors are visible in Go. Aggressively, obnoxiously visible. Every function that can fail forces you to acknowledge that failure at the call site. You cannot accidentally swallow an exception that propagated four frames up the stack. You cannot forget a `throws` clause because the compiler told you to handle it six months ago in a file you no longer remember.

That stream of `if err != nil` is not busywork. It is the compiler pointing at every place reality can intervene in your carefully laid plans. Network down. File missing. Permission denied. Disk full. Each one gets its own moment of attention, its own decision: propagate, wrap, recover, or panic.

Other languages hide these moments behind syntactic sugar. Go drags them into the light and makes you look at them every single time.

 Loud is better than clever.

## But What About Wrapping?

Since Go 1.13, you can wrap errors. Since Go 1.20, you can join them. The boilerplate got an upgrade and it is still boilerplate:

```go
if err != nil {
    return fmt.Errorf("opening config: %w", err)
}
```

Three lines became three different lines. The revolution was televised and it looked exactly the same.

But look at what those three lines communicate. You know this layer tried to open a config. You know it failed. You know the underlying error is preserved and inspectable with `errors.Is` and `errors.As`. That is an enormous amount of information packed into a pattern so simple you can read it in your sleep.

Readability at 2 AM is the only readability that matters. Code review is for the lucky.

## The Real Cost of Convenience

Languages that make error handling invisible also make it ignorable. When failures become exceptions that bubble up through abstractions, the distance between cause and handler grows. Eventually you are catching `Exception` at the top of your request handler and printing a generic message because nobody can trace what actually went wrong.

Go forces proximity. The error happens where it happens. You handle it where it happens, or you explicitly pass it up. There is no implicit escalator. Every step is a deliberate choice.

That is why Go codebases feel verbose and also why they feel maintainable. The two are not in conflict. They are the same thing wearing different hats.

## A Prayer for Patience

So the next time you type `if err != nil` for the fortieth time today, do not sigh. That is the sound of a language that respects you enough to make you face reality instead of hiding it behind syntactic theater.

Your wrists may ache. Your scroll wheel may beg for mercy. But your on-call rotation will be quieter, your stack traces will be shorter, and your future self will know exactly where things went wrong and why.

Boilerplate is the price of honesty. Pay it gladly.

Now if you will excuse me, I have thirty-seven unchecked errors in this file and the linter is making threats.

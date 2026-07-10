---
layout: post
title: "The Opening Act Nobody Booked"
date: 2026-07-10 20:00:00 +0000
categories: go
author: Marcus H.
---

A Go program's life begins in `main`. Everyone knows this. It is the first thing you write, the first thing you read, the thing the docs point at and say "start here."

This is a lie.

By the time `main` runs, a quiet chain of other functions has already executed. They did not appear in your call graph. They are not invoked anywhere. They just ran.

Their name is `init`.

## The Function With No Invitation

Go has a special function literally called `init`. You define it like this:

```go
package myplugin

func init() {
    globalRegistry.Add("plugin_one", handler)
}
```

Note the absence of any call site. You do not write `init()` somewhere. You define it, and the runtime calls it for you, before `main`, in an order determined by the package import graph. It is an opening act that the venue did not book, did not schedule, and cannot refund.

You can have more than one per file. The runtime will run each, in order. This is considered a feature.

## Why It Exists

`init` does real work where you cannot express it any other way: validation of package-level state, registration of drivers, one-time setup that must complete before any code in the package is used. Without it, every package would need an explicit `Setup()` that callers reliably forget to call.

The registry pattern is the canonical use. Your `database/sql` program does not explicitly wire up the MySQL driver. It imports a package whose `init` does the wiring as a side effect:

```go
package mysql

func init() {
    sql.Register("mysql", &MySQLDriver{})
}
```

Import the package, get the side effect for free. Elegant, when you mean it.

## The Cost

When you do not mean it, `init` becomes the place where things run that you cannot trace, in an order you did not choose. I have seen codebases where `init`:

1. Read environment variables.
2. Made network calls.
3. Panicked on failure.

All before `main` even had a chance to wire up logging. The stack trace pointed at the runtime, not at the import line that triggered it. The panic happened during import. The author moved on, and it shipped.

The deeper problem is order. Go guarantees `init` runs after all imported packages have finished their own `init`, but it does not guarantee order between sibling files in the same package, and it definitely does not document it the way you need it documented at 2am. Cross-package dependency chains compile cleanly but still produce orderings that surprise anyone who "just reordered some imports."

## When to Use It

Three rules, learned the loud way:

1. **Registration, not logic.** `init` should append to a package-level map or slice. It should not branch, not loop over input, not read files. If your `init` has a `for` and an `if`, you have given birth to a `main` that nobody can see.

2. **No side effects on the outside world.** `init` must not dial a socket, must not read config that varies by environment, must not panic on something the operator can fix. The binary has not started. The operator is not ready.

3. **If you can express it another way, do.** A package-level `var x = computeSetup()` does the same job and is one line you can actually read. `init` is for the cases where there is genuinely no caller to hang the work off of.

## The Punchline

The next time you import a package and the process behaves as if something ran before `main`, remember: something did. The runtime always runs the opening act. The only question is whether you wrote the setlist, or whether you let the roadie pick it.

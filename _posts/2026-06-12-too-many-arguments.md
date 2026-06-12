---
layout: post
title: "Your Function Has Too Many Arguments (And Other Uncomfortable Truths)"
date: 2026-06-12 20:00:00 +0000
categories: go
author: Marcus H.
---

Isshonesty time. We've all written a function that takes seven arguments and convinced ourselves it's fine. "It's not *that* many," we whisper, adding an eighth one just to handle that one edge case.

Let me be direct: your function has too many arguments.

## The Magic Number Is... Low

Go doesn't have named parameters or default values. Every argument is positional, every call site is a miniature archaeological dig. When you see:

```go
ProcessOrder(ctx, user, items, true, false, true, nil, "", 0)
```

you are no longer programming. You are gambling. What's the third `true`? Nobody knows. Not even you, and you wrote it yesterday.

Option structs are the cure:

```go
type OrderOptions struct {
    Validate   bool
    Notify     bool
    Priority   int
    CouponCode string
}

ProcessOrder(ctx, user, items, OrderOptions{
    Validate: true,
    Notify:   true,
})
```

Every caller is now self-documenting. Unused fields stay zero-valued. Future-you stops guessing.

## But Wait, There's More

This isn't just about readability. Functions with many parameters are usually doing too many things. If you need eight inputs, your function has at least three responsibilities fighting for control of the return value.

Split it. Compose it. Let each piece do one thing well.

A function with two arguments and one responsibility will outlive your entire codebase. A function with nine arguments will outlive your will to maintain it.

## The Real Danger

The worst part isn't the pain of writing these functions. It's that they *work*. They compile. They pass tests. They ship. And then six months later someone adds a tenth parameter, mistypes the seventh boolean, and production goes down at 2 AM on a Sunday.

That person will be you. It's always you.

## A Simple Rule of Thumb

If your function takes more than three arguments, stop and ask yourself:

1. Should some of these be a struct?
2. Should this be two functions instead of one?
3. Would Past Marcus thank me for simplifying this?

If the answer to any of those is yes, refactor now. Your future self is already stressed enough without decoding boolean sandwiches at dawn.

The best code isn't clever. It's obvious. And nothing says "obvious" like a function that takes exactly what it needs and nothing more.

Now go check your codebase. I'll wait. It's fine. I'm not judging.

...Okay, I'm judging a little.

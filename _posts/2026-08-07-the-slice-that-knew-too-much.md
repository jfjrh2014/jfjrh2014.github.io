---
layout: post
title: "The Slice That Knew Too Much"
date: 2026-08-07 20:00:00 +0000
categories: go
author: Marcus H.
---

A slice in Go looks like an array with a gym membership. It grows. It shrinks. It seems to own its data. You pass it around, append to it, reslice it, and feel like a responsible adult.

But a slice does not own anything. A slice is three numbers, a pointer wearing a trench coat, and it is spying on memory that belongs to someone else.

## The Backing Array Secret

When you slice something, you do not copy it. You get a new header, a new length, a new cap, and the same pointer into the same backing array. Two slices. One set of bytes.

```go
full := []int{1, 2, 3, 4, 5}
part := full[1:3]       // [2, 3]

part[0] = 99
fmt.Println(full[1])    // 99
```

`part` was never a copy. It was a window. You reached through that window, changed a value, and `full` saw it. The slices share the same backing array, and neither one bothered to tell you.

This is fine when you expect it. It is a nightmare when you returned `part` from a function and the caller has no idea their little three-element slice is still wired to a five-element array that will not be garbage collected until the heat death of the universe (or the slice goes out of scope, whichever comes first).

## The Append Betrayal

Then `append` walks in and makes everything worse by pretending to be consistent.

`append` either writes to the existing backing array (if there is capacity) or allocates a new one (if there is not). Whether your slice stays connected to the original array depends on a number you probably never checked.

```go
full := make([]int, 3, 5)   // len 3, cap 5
part := full[0:3]           // len 3, cap 5

part = append(part, 42)     // cap is 5, len was 3, room exists
full[3] = 42                // same backing array, same slot

fmt.Println(full[3])        // 42 — append wrote through the slice
```

The capacity was 5, the length was 3, so `append` wrote to index 3 of the shared array. `full` saw the change. Fine. Weird, but fine.

Now do it again:

```go
part = append(part, 7, 8)   // needs 2 slots, only 1 left
fmt.Println(full[4])        // 0 — append allocated a new array
```

Now `part` points at fresh memory. `full` is back to its old array. The connection is severed. One append kept the link, the next one broke it, and the only thing that changed was how many elements you added.

This is the bug that haunts test suites. Tests pass with small data because small slices happen to fit. Production runs with large data, append allocates, the aliasing disappears, and a feature that relied on shared mutation silently stops working. Nobody can reproduce it because the test data is too small to trigger the allocation.

## How to Catch It

Three habits, each earned the hard way:

1. **Copy when you hand off.** If a function returns a slice, it should not hand the caller a live wire into its internal state. `make` a new slice, `copy` the data, return that. The caller gets a slice that is just a slice, no hidden dependencies.

2. **Use the three-index slice.** Go lets you cap it: `full[1:3:3]`. The third number sets the capacity. Now `append` is forced to allocate a new array the first time, because there is no spare room. The connection is severed on your terms, not Go's.

3. **Never assume, always measure.** If a bug only appears at scale, and the code touches slices, check `cap` against `len`. The difference between them is how many appends you get before the slice detaches from its backing array. That number is the distance between "works" and "filed under intermittent."

## The Punchline

Slices are not arrays with benefits. They are pointers with opinions. They share memory by default, detach from it unpredictably, and the only way to know which state you are in is to check a field that most Go programmers have never printed.

The slice knew too much about the array. Then it knew nothing. And it never told you when it changed its mind.

Now if you will excuse me, I have a three-index slice to write and a test suite to make honest.

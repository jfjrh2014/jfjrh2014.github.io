---
title: "The Map That Shuffles First"
date: 2026-10-02 20:00:00 +0000
author: "Marcus H."
categories: go
---

Every Go developer has a story that starts the same way: the test passed at
home and failed in CI, and the diff showed two lines printed in a different
order. The culprit was not a race condition. It was a map, doing what maps
do.

Range over a map in Go and you get the elements back in no particular order.
Not insertion order, not roughly sorted with occasional surprises. No order.
The spec does not promise one, and the runtime works hard to keep that
promise broken: every range statement starts from a random bucket, so the
same map can deal a different hand on two consecutive loops.

```go
counts := map[string]int{
	"ada": 3, "go": 7, "rust": 2,
}

for name, n := range counts {
	fmt.Println(name, n)
}
```

Run it twice. Two different orders. That is not entropy leaking in through
the floorboards. The house shuffles the deck on purpose.

Why? Because a promise not kept is worse than a promise never made. Early
map implementations happened to iterate in a stable-ish order, and programs
started leaning on it. If the runtime kept that accident alive, thousands of
programs would quietly depend on an implementation detail, and any rehash or
growth would break them at runtime. So the runtime randomizes the start
point and, in effect, dares you to depend on the order. The documentation
says the same thing, only politer.

The cure is one line. If your output, test, or error message needs a stable
order, own the order:

```go
keys := slices.Sorted(maps.Keys(counts))
for _, name := range keys {
	fmt.Println(name, counts[name])
}
```

`slices.Sorted` hands you ascending keys, and now the loop is boring in the
best possible way. Determinism is a subscription: you pay O(n log n) per
print, and your test output stops gaslighting you.

There is a second lesson hiding here. If you need ordered data, a map is the
wrong container. A slice keeps insertion order by contract, and small
"ordered map" needs (a handful of config entries, a staged list of steps)
are usually just a slice of key-value pairs wearing a trench coat. Reach for
the map when membership and lookup are the job; reach for the slice when
sequence is the job. That choice is the whole decision.

One warning for the sort-it-yourselfers: do not sprinkle sorting into every
loop out of fear. Sorting every iteration to dodge a shuffle you will never
observe is paying interest on a loan you never took. Sort where output
leaves the program: tests, logs, reports. Inside the loop, the order was
never real to begin with.

And if you genuinely need insertion order plus fast lookup, you will find
yourself writing a map plus a slice of keys, keeping the two in sync like a
duet. Go will not stop you. It will just shuffle one of them when you are
not looking, and the review comment will find it for you.

Maps are brilliant at what they promise: membership, lookup, deletion. They
never promised a parade in formation. The shuffle is a feature, the fix is a
one-liner, and your CI will thank you the first time the flake it caused
becomes a test you sorted.

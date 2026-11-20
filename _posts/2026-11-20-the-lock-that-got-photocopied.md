---
title: "The Lock That Got Photocopied"
date: 2026-11-20 20:00:00 +0000
author: "Marcus H."
categories: go
---

A `sync.Mutex` is a struct. Assign it to a new variable and Go copies it, the
way it copies any struct. You now hold two locks. The copy keeps whatever
state the original had at that moment, and after that the two never hear from
each other again.

That is the whole bug. The rest of this post is about where it hides.

## A lock born locked

Here is the smallest version. Lock a mutex, copy it, unlock the original:

```go
var mu sync.Mutex
mu.Lock()
clone := mu
mu.Unlock()

fmt.Println(mu.TryLock())    // true
fmt.Println(clone.TryLock()) // false
```

The original is free again. The clone was copied mid-lock, so it starts life
locked, and nobody holds the key. Call `clone.Lock()` and that goroutine waits
forever. It is a deadlock you got from an assignment, which is a lot of
trouble for one equals sign.

(`TryLock` arrived in Go 1.18. I use it here only to peek at the state. If you
reach for it in real code, ask yourself why first.)

## The receiver that works from a copy

Nobody writes `clone := mu` on purpose. They write this:

```go
type Counter struct {
	mu sync.Mutex
	n  int
}

func (c Counter) Inc() {
	c.mu.Lock()
	c.n++
	c.mu.Unlock()
}

var c Counter
for i := 0; i < 1000; i++ {
	c.Inc()
}
fmt.Println(c.n) // 0
```

A thousand calls, and the counter reads zero. The method has a value receiver,
so every call gets a fresh copy of the whole `Counter`: a private mutex, a
private `n`. It locks its own lock, bumps its own number, unlocks, and throws
the lot away. The lock is perfectly safe. It is just guarding a photocopy of
your data while the original sits unguarded in the next room.

The fix is one character:

```go
func (c *Counter) Inc() {
	c.mu.Lock()
	c.n++
	c.mu.Unlock()
}
```

With a pointer receiver, a thousand goroutines calling `Inc` end at exactly
`1000`. Every one of them reached for the same lock, which was the plan all
along.

## Places the copy sneaks in

The value receiver is the classic, but it has cousins. Any of these copy the
struct, lock included:

- passing the struct to a function by value
- returning it by value
- `for _, c := range counters`, where `c` is a copy of each element
- putting it in a map as a value and reading it back out

The range loop is the sneaky one, because it looks like a read. You only
wanted to print `c.n`. Go handed you a copy of each counter, mutex and all.

## Let vet do the proofreading

You do not have to spot these by eye. `go vet` ships a check called
`copylocks`, and it is on by default. Running it on the examples above
(Go 1.19 here) prints:

```
./main.go:13:9: Inc passes lock by value: vt.Counter contains sync.Mutex
./main.go:33:11: assignment copies lock value to clone: sync.Mutex
./main.go:55:9: range var c copies lock: vt.Counter contains sync.Mutex
```

All three caught, and it costs nothing. `go test` already runs a subset of vet
before your tests, but not the full suite, so put a plain `go vet ./...` in
CI. It is a cheaper reviewer than I am, and it never asks for a pastry.

## The house rules

A few habits keep the photocopier unplugged:

- If a struct holds a `sync.Mutex`, give it pointer receivers. All of them, so
  nobody mixes the two styles later.
- Pass and store it as `*T`. A slice of `*Counter` ranges without copying.
- Do not copy it after first use. The `sync` docs say exactly this, and they
  mean it for `WaitGroup`, `Once` and `Cond` too.

A lock is a promise that one goroutine at a time touches the data. Copy the
lock and you have two promises about two different things, and only one of
them is about the data you care about.

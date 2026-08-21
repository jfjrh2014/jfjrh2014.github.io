---
title: "The nil That Wasn't"
date: 2026-08-21 18:00:00 +0000
author: "Marcus H."
categories: go
---

There's a special kind of betrayal in Go that doesn't involve maps, channels, or
goroutines. It involves `nil`. Specifically, the `nil` you checked for and
*still* got bitten by.

Here's the crime scene:

```go
type Shop struct{ name string }

func findShop(name string) (*Shop, error) {
    // not found? hand back a nil pointer
    return nil, nil
}

func main() {
    var s *Shop = nil
    describe(s)
}

func describe(v interface{}) {
    if v == nil {
        fmt.Println("nothing here")
        return
    }
    fmt.Println("found:", v.(*Shop).name)
}
```

`describe` gets a `nil *Shop`. It checks `v == nil`. The check fails. The
program panics. You stare at the screen like someone who just checked their
pocket for keys that are already in their hand.

## The Two-Word Secret

An interface value in Go is not one thing. It's two words: a type and a value.
The comparison `v == nil` is only true when *both* words are empty.

When you assign a `nil *Shop` to an `interface{}`, the interface dutifully
records: type = `*Shop`, value = `nil`. One word filled, one word empty. That's
not `nil` as far as the interface is concerned. It's a `*Shop` that happens to
point at nothing. A beautifully wrapped empty box.

The 2014 Go FAQ calls this "a subtle bug trap that bites every Go programmer
sooner or later." I can confirm the "sooner or later" part empirically, having
been bitten on at least four separate occasions, each time with the deep
sincerity of someone who was *certain* it couldn't happen again.

## Why This Hurts

The trap hides exactly where error handling lives. Consider the most common
shape:

```go
func doThing() error {
    var p *MyErr = nil
    return p // compiles fine
}

func main() {
    err := doThing()
    if err != nil { // true! p != nil as an interface
        fmt.Println("failed:", err) // prints "<nil>"
    }
}
```

That's right: the program prints the word `failed` followed by `<nil>`. Your
error is simultaneously present (non-nil interface) and contains nothing (nil
pointer). Schrödinger would be proud, and your on-call rotation would not.

This is why every style guide on Earth says: *return literal `nil` for errors,
never a typed nil.* The function signature says `error`, so return the one true
`nil`, not a pointer that has merely achieved nilness.

## How to Catch It

Three defenses, in escalating order of paranoia:

**1. Never return typed nils.** If the return type is an interface, the value
you return on the happy path should be bare `nil`, full stop.

**2. Reflect, when you must accept interfaces.** If an API hands you an
`any` that might be a typed nil (JSON unmarshaling, database drivers, mock
frameworks are the usual suspects), check it honestly:

```go
func isReallyNil(v any) bool {
    if v == nil {
        return true
    }
    rv := reflect.ValueOf(v)
    switch rv.Kind() {
    case reflect.Ptr, reflect.Map, reflect.Slice,
        reflect.Chan, reflect.Func, reflect.Interface:
        return rv.IsNil()
    }
    return false
}
```

**3. Let vet find the obvious ones.** `go vet` (Standard since Go 1.12,
enhanced since 1.21) flags direct comparisons of interface-typed values against
typed nils in several shapes. It won't catch a typed nil *returned* through a
variable, so defense #1 remains a team rule, not a tool output.

## The Punchline

Go's designers chose this deliberately. A typed nil is sometimes exactly what
you want: a nil `*bytes.Buffer` is a perfectly functional empty reader, and
that only works because the interface remembers the type. The trap isn't a bug
in Go. It's the price of the feature. Two words, and the second one spends its
whole life being misunderstood.

So the next time `v != nil` is true but `v` prints as `<nil>`, don't blame the
compiler. It did exactly what you asked. You just asked in a language where
"nil" has layers — a lesson every gopher learns once, and then again
approximately every 18 months.

Check your pockets. The keys are in your hand.

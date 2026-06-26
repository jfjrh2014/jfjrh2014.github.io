---
layout: post
title: "Your Struct Implements Seventeen Interfaces It Never Met"
date: 2026-06-26 20:00:00 +0000
categories: go
author: Marcus H.
---

In Java, you announce your loyalties. `class Dog implements Animal, Comparable, Serializable`. Before the file even compiles, you have filed paperwork in triplicate naming every contract you intend to honor. The compiler checks your papers at the door.

In Go, nobody asks.

You write a struct. You give it a method. Somewhere across the codebase, an interface already exists that your struct now satisfies, and neither of you signed anything. The compiler noticed. You did not.

```go
type File struct{ /* ... */ }
func (f *File) Read(p []byte) (int, error) { /* ... */ }
```

That struct now implements `io.Reader`. Congratulations. It also implements every other interface ever written that has a single `Read` method with that signature. Some of those interfaces live in repositories you have never opened, written by people you will never meet, and your code works with them anyway.

## The Smallest Interface Wins

`io.Reader` is one method. `io.Writer` is one method. Together they are the entire foundation of streaming IO in the standard library. Files, network connections, buffers, compression wrappers, hash functions, all of them speak two words and understand each other perfectly.

This is not an accident. The Go philosophy is that interfaces should be small, ideally one method, and defined by the consumer rather than the producer. You do not define a `StorageClient` interface with eleven methods and then write a mock with eleven no-op implementations. You define what you need at the point where you need it:

```go
type ConfigLoader interface {
    Load(path string) ([]byte, error)
}
```

Three lines. Testable with a fake in under a minute. No mocking framework, no code generation, no reflection. A struct that has a `Load` method with that signature is your mock. You write the struct, you pass it in, you are done.

## Accept Interfaces, Return Structs

There is a piece of advice that gets repeated until it loses all meaning, then gets repeated more because it is still correct: accept interfaces, return concrete types.

Functions take interfaces so callers can pass anything that satisfies the contract. Functions return structs so callers get a known, concrete thing with documented behavior. Inverting this, returning an interface, is where the lying starts. The caller thinks they have a generic thing. They actually have one specific implementation wrapped in a lie that hides its real methods and prevents type assertions.

Return the struct. Let the caller decide what interface it satisfies. Trust them. They know their code better than you do.

## The Dark Side: Interface Pollution

Cheap as interfaces are, it is tempting to define one for everything. Every package gets its own `UserManager`, `UserReader`, `UserWriter`, `UserDeleter`, each wrapping a single struct that does all four. Five files of indirection so that the one real implementation does not feel lonely.

This is called interface pollution, and it is what happens when developers learn the word "decoupled" before they learn the word "premature." Interfaces exist to solve real decoupling problems: testability, multiple implementations, boundaries between packages. If you have one implementation and no tests, the interface is decoration. Delete it. The code will not miss it. The next reader will thank you.

## The Thing Nobody Mentions

Here is the part that does not fit in a slide deck. Implicit interfaces change how you read code. In a language with explicit declarations, you look at a class header to understand what it is. In Go, you look at what it does and you understand it by its behavior.

A `bytes.Buffer` is not interesting because of its type hierarchy. It is interesting because it reads, writes, grows, resets, and trims. Its identity is its method set. You learn it by using it, not by studying its ancestry.

This is harder at first. There is no class diagram holding your hand. But the code you write becomes more about verbs and less about nouns, more about what things do and less about which aristocratic lineage they claim.

And honestly, after a few months, the idea of declaring your intents to a compiler in writing starts to feel a little formal. Like wearing a name tag to your own breakfast.

Now if you will excuse me, I just discovered my struct implements an interface from 2018 and I need to go apologize to the test suite it breaks.

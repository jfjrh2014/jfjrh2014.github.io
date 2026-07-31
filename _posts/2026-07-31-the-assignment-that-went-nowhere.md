---
layout: post
title: "The Assignment That Went Nowhere"
date: 2026-07-31 20:00:00 +0000
categories: go
author: Marcus H.
---

Go has a feature that sounds like a superpower until it isn't. The compiler refuses to let you declare a variable and ignore it:

```go
func main() {
    src := resolveSource()
}
```

`src declared and not used.` Build fails. You feel protected. You feel seen. The language cares.

Then you write a package-level variable, assign to it, read it once with `_ = it`, and never realize that the one call site which actually needed it is getting an empty string.

The protection is narrower than it looks.

## The Hole in the Net

Go's unused-variable check is precise but local. It fires on a name you declare in a function and never reference in that same function. It does not extend to three places where dead values love to hide.

The first is package-level state. A global you assign and then read in the wrong spot will never trip the check. The name is "used." It just isn't used where it matters.

```go
var importSource string

func runImport(name, path string) {
    importSource = name          // assigned, never flagged
    _ = importSource             // read, compiler satisfied
    load(NewImporter(""))        // empty string, bug shipped
}
```

That `_ = importSource` is the tell. It exists to silence a warning that would have saved you. Remove the underscore and the build fails. Leave it in and the empty string walks out the front door dressed as correctness.

The second is shadowing. A variable declared in an inner scope hides one of the same name outside it, and the compiler is perfectly happy with both.

```go
src := "default"
if cfg.HasProfile() {
    src := cfg.Source()   // new variable, same name, dies here
    templates = load(src)
}
copyTo(src)               // still "default"
```

The inner `src` did its job and then ceased to exist. The outer one never heard about it. Everything type-checks. Nothing works.

The third is routing through `_`. Any expression assigned to `_` is "used" by definition. The compiler moves on. The value goes into the void and your program inherits the default.

## Why This Hurts

The common thread is that the bug hides behind code that reads as correct. There is an assignment. There is a reader. They have the same name. They point at different storage. The linter has nothing to say because, syntactically, everything is accounted for.

This is the class of bug that survives code review. The reviewer sees `importSource = name`, nods, and scrolls on. The name is on both sides of the equals sign. What could be wrong?

Everything, as it turns out. The assignment touches one variable and the reader consults another. They agree on a name and disagree on an address, and Go will never tell you.

## How to Catch It

Three habits, polished across a week of fixing the same bug in seven different files:

1. **Distrust the package-level variable.** If a value needs to travel from one function to another, pass it as an argument. The compiler checks arguments. It does not check whether your global reached its destination. Async state on package globals is a reentrancy bug wearing a trench coat.

2. **Read `_ = x` as a confession.** Any line that assigns to `_` is either intentional ignoring or a lie told to the compiler. If you cannot explain in a comment why the value is discarded, delete the line and see what breaks. Often what breaks is the bug you were hiding.

3. **Use the same name, the same address.** When a value is critical to a code path, name it once and let that name flow through unchanged. Two variables with identical names in overlapping scopes is a shadow waiting for a victim. `go vet -shadow` exists. Run it.

## The Punchline

The compiler will catch you when you declare a variable and never use it. It will not catch you when you use a variable that does not mean what you think it means.

That gap is where the production incidents live. Not in the spectacular nil dereference with a stack trace, but in the quiet assignment that arrived at the wrong address, shook hands with nobody, and let the empty string do the talking.

The value was declared. The value was assigned. The value was used. It was just never the value you wanted.

Now if you will excuse me, I have a global that thinks it matters and a constructor that has never heard of it.

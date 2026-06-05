---
layout: post
title: "The Five Stages of Grieving a nil Pointer"
date: 2026-06-05
categories: go
---

Every Go developer knows the feeling. Your service has been humming along in production for weeks. You deploy a tiny refactor — barely a diff, more of a gentle suggestion — and at 3:47 AM your pager coughs to life with the most feared three words in the language: **nil pointer dereference**.

What follows is grief. Pure, structured, Kübler-Ross-approved grief.

## 1. Denial

"This can't be nil. I just initialized it on the line above. The tests passed. The linter was happy. My rubber duck nodded solemnly."

You reread the function. You reread it again. You add a `fmt.Println` like a fingerprints-on-the-window detective. It is, of course, nil. The variable was initialized in a different scope, or returned early from a branch you forgot existed, or quietly shadowed by a loop variable three callbacks deep.

**Symptom:** Adding `if x == nil { return }` defensively and pretending this is "defensive programming."

## 2. Anger

"Why doesn't this language have optional types? Why is `nil` even a thing? Rust doesn't have this problem. My grandma doesn't have this problem. She doesn't write Go, but still."

This is the stage where you tweet things like *"Go's error handling is fine actually, it's `nil` that's the crime."* You consider filing a proposal. You read three old Go issue threads from 2017. You close them feeling worse.

**Symptom:** Strong opinions on `errors.Join` and a half-written blog post titled "We Deserve Better."

## 3. Bargaining

"Okay, what if I wrap everything in a struct? What if I add a `Valid() bool` method? What if I use `*T` only when I really, really mean it? What if I just sprinkle `if err != nil` a little harder?"

You discover `google/go-cmp` and convince yourself that better tooling will save you. It will not. But you write the comparison anyway, and for one holy moment the test passes.

**Symptom:** Your struct now has four helper methods named `IsZero`, `IsValid`, `NotNil`, and `Dave` (Dave is for luck).

## 4. Depression

You look at your call stack. The nil escaped from a goroutine that called a closure that read from a channel that was populated by a JSON unmarshaler that didn't know your field existed. The nil is thirteen frames above you. You are at its mercy.

You open the Go FAQ. It does not help. You open the spec. It does help, but in the way a doctor's diagnosis helps: you now have a name for the thing that's hurting you.

**Symptom:** The `// TODO: fix this` comment from 2024 suddenly makes sense.

## 5. Acceptance

You write the nil check. You write a test that constructs the nil on purpose. You add a comment explaining why the nil can happen, who is responsible for it, and what the caller should do. You begin to see nil not as a bug, but as a feature — a small, polite "no comment" from the runtime about a value that simply isn't there.

You ship. You sleep. The pager stays quiet.

**Symptom:** You start sentences with "So actually, `nil` is kind of elegant if you think about it..." at parties. People stop inviting you to parties.

## The Real Lesson

The five stages aren't really about `nil`. They're about every sharp edge in Go: channels that block forever, maps that race, goroutines that leak, `interface{}` that pretends to be a type. The language is small on purpose, and small languages hand you the loaded gun.

That's also why I love it. There's no macro to hide behind, no decorator to wave at the bad smell. When something is wrong in Go, it is *visibly, desperately wrong*, in your face, on line 47, with a stack trace that points right at the crime scene.

The fix is almost always the same: a small function, a clear comment, a test that doesn't apologize. Go doesn't reward cleverness. It rewards kindness to your future self.

So the next time the pager goes off at 3:47 AM, take a breath, open the file, look at the nil, and remember: you are not the first developer to grieve, and you will not be the last. Kübler-Ross just forgot to include `panic: runtime error: invalid memory address` as a footnote.

Stay typed, friends.

— Marcus H.

---
title: "The Wrap Sheet"
date: 2026-09-18 20:00:00 +0000
author: "Marcus H."
categories: go
---

An error in Go starts life as a decent witness. `errors.New("connection
refused")` saw the host and knows the exact second. Then it gets passed up
through three packages, each adding a note for the log file, and somewhere
along the way somebody writes this:

```go
return errors.New("failed to process order")
```

The witness is gone. The log now says an order failed, which has the
investigative value of a parking ticket that just says "car."

The fix is one character. `fmt.Errorf` has a wrapping verb, `%w`, and it
embeds the old error inside the new one:

```go
return fmt.Errorf("process order %d: %w", orderID, err)
```

The outer error keeps its own message plus a pointer to the one underneath.
Errors stack up like a tidy pile of parking tickets, each admitting fault and
naming the previous holder of the fault.

Which is why this line, beloved by tutorials and veteran of a thousand
codebases, eventually stops working:

```go
if err == ErrNotFound {
```

`==` compares the outermost error only. The day somebody wraps `ErrNotFound`
in context (they will, because the log line demanded it), you are comparing an
envelope to a letter. It compiles. Tests that exercise the happy path stay
green. And in production, every not-found quietly becomes a 500.

`errors.Is` is the replacement. It checks the error, calls `Unwrap()`, checks
what comes back, and keeps walking until it finds your sentinel or runs out of
pile. Same story for types. The assertion `err.(*fs.PathError)` inspects
exactly one layer, the outermost. `errors.As` inspects all of them:

```go
var pathErr *fs.PathError
if errors.As(err, &pathErr) {
    return pathErr.Path
}
```

The verb comes with a rule attached: use `%w` when callers need to identify
the underlying error, use `%v` when they don't. Sounds reasonable. In practice
nobody knows what callers will need in eighteen months, so `%v` spreads
through a codebase the way uninitialized maps spread through a junior's pull
request. Nothing crashes. No test fails. The messages in your logs look
perfectly normal. The chains are just gone, and with them every `errors.Is`
you were planning to write.

Worse is the deportation version:

```go
if err != nil {
    return errors.New("db error")
}
```

Nothing was wrapped here. The original error was questioned, escorted out of
the building, and given no forwarding address. The replacement has none of the
facts.

Since Go 1.20 the verb accepts more than one `%w`, and `errors.Join` stacks
several failures side by side:

```go
err := errors.Join(errProfile, errSettings)
```

`errors.Is` checks every branch, so your error chain is a family tree now.
Which is exactly what you want for "the page half-loaded and both halves
failed" reports, and exactly what you don't want to discover you don't have,
at 3am, from a log line that says `failed: failed`.

The whole habit costs about four characters per error site. Wrap where it
matters and match with `Is` and `As`, and every error that reaches your logs
can still name names. A good error tells you what failed. A wrapped one
testifies about where, when, and with whom. Every investigation needs a wrap
sheet. Yours is one verb wide.

---
title: "The Apology That Runs Twice"
date: 2026-10-16 20:00:00 +0000
author: "Marcus H."
categories: go
---

Some bugs are hard. Some bugs are just the same easy bug, wearing a trench
coat. A connection drops, the client retries, and the server executes the
same write twice. Nobody wrote a loop. Nobody called the function twice in
the code you can see. The second execution was born from the first one
failing loudly on the way home.

Go's standard library has a word for this: the RPC package calls it a
"repeatable request" in its own docs, and the rule it hands you is blunt. A
handler must either be idempotent, or the client must not retry it. That
sentence is the entire discipline of distributed systems, compressed into
eleven words you can fit on a sticky note.

An idempotent operation is one you can run any number of times and get the
same result as running it once. `SET key value` is idempotent. `INCR key`
is not. The difference sounds academic until your payment provider's webhook
fires twice and you learn which side of the line your invoice logic lives
on, with real money as the test fixture.

The classic fix is the idempotency key. The client mints a random token,
sends it with the request, and the server remembers every token it has
processed. Second attempt, same token? Return the first result instead of
doing the work again. Stripe didn't invent this pattern, but they
documented it so well that it's now the shape every payment API reaches
for. The code is almost disappointingly small:

```go
func handlePayment(key string, amount int) error {
	if done[key] {
		return nil // already handled, nothing to redo
	}
	done[key] = true
	return charge(amount)
}
```

Every part of that snippet is a lie, and the lies are the lesson. The
`done` map is in memory, so it evaporates on restart, which is exactly when
retries tend to arrive. The check-then-set is not atomic, so two requests
racing through the gate both see an empty set and both charge. Fixing it
for real means the memory has to be shared and the decision has to be
atomic, which usually means Redis or your database grows a constraint:

```go
INSERT INTO idempotency (key, result) VALUES ($1, $2)
ON CONFLICT (key) DO NOTHING
```

One insert, one winner, losers get the stored result. The database's
transaction machinery does the work your `if` statement was pretending to
do.

The subtle part is what "same result" means. If the first attempt failed
halfway, replaying the stored result replays the failure too, which is
correct but feels wrong to everyone. If the token is generated per retry
instead of per operation, the whole mechanism is decoration. The key must
identify the user's intent, not the attempt. Same checkout click, same key.

None of this is exotic. It's the same idea as `go mod tidy` being safe to
run repeatedly, or a `PUT` returning the same resource every time. The
systems that stay boring are the ones where every write answers the
question "what if this exact request happens again?" before the network
gets a vote. The retry storm is coming. The only choice you get is whether
it's a rerun or an encore.

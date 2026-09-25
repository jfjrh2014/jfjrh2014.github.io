---
title: "The Date That Cannot Change"
date: 2026-09-25 20:00:00 +0000
author: "Marcus H."
categories: go
---

Every language has a line of code its users memorize against their will.
Ruby has monkey patches gone wrong. JavaScript has "undefined is not a
function". Go has this:

```go
time.Now().Format("2006-01-02")
```

That is not a date. It is a sentence in a language that only ever speaks
one afternoon: Monday, January 2nd, 2006, at 15:04:05, Mountain Standard
Time. That is the reference time, and every layout in Go's time package is
a rearrangement of it. Write the numbers in any other order and you are
not formatting a timestamp, you are confiding in the parser.

Most languages use strftime, a set of cryptic but teachable tokens: %Y for
year, %m for month, %d for day. Learn six tokens and you can fake the
rest. Go's designers looked at that and decided the tokens were the
problem. Their fix: the layout is an example of itself. Want the date?
Write `2006-01-02`. Want a twelve-hour clock? Write `03:04:05PM`. The
format string doubles as its own documentation.

The catch is that the example has to be that exact example. Not the
current year. Not the date you are formatting today. The digits are
load-bearing: 1 is the month, 2 the day, 3 the hour, 4 the minute, 5 the
second, 6 the year, 7 the timezone offset. The long version spells it out:
01/02 03:04:05PM '06, plus -0700 for the zone, and it makes seven. It is a
phone number. Once you see it you cannot unsee it, which is precisely the
design working as intended.

The pattern pays rent in error messages. Feed the parser a mismatched
layout and it shows its work:

```text
parsing time "2026-09-25" as "01/02/2006":
    cannot parse "2026-09-25" as "01"
```

It tells you which chunk of the layout choked, quoting your input against
the reference tokens. Compare that to C's `strftime`, where a bad format
mostly gets you the wrong characters printed confidently, and the design
starts to look less like a joke and more like a choice with a bill
attached.

There are two classic landmines, and both come from treating the layout
like an actual date. The first is writing the current year into it, say
`Format("2026-01-02")`. The parser reads the digits it recognizes, drops
the rest in as literals, and hands you back a string no calendar on Earth
would accept. The second is timezone handling: a layout without `Z07:00`
will happily parse an offset-bearing timestamp and throw the offset away.
Every Go developer has lost at least one evening to that one, usually on a
December 31st, because timestamps love to betray you at year boundaries.

After a few years you stop noticing. You type `2006-01-02` from muscle
memory the way you type `if err != nil`, and the reference time becomes
background radiation. Which is the quiet punchline of the whole design: it
is the only date in computing with permanent job security. Every Go
timestamp ever formatted has been measured against that particular Monday
afternoon, and none of them have moved it. Go has been around long enough
that 2006 is retro, and the reference time does not care. It cannot
change. It is the standard.

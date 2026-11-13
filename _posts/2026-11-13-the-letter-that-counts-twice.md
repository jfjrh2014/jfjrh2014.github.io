---
title: "The Letter That Counts Twice"
date: 2026-11-13 20:00:00 +0000
author: "Marcus H."
categories: go
---

Ask Go how long the word "café" is and it will tell you 5. The word has four
letters. Go is not bad at counting. It is counting bytes, and it never promised
to count anything else.

A Go string is a read-only slice of bytes. The language does not check what
those bytes mean. Source files are UTF-8, so string literals usually hold UTF-8
text, and in UTF-8 the "é" takes two bytes. That is where the extra one comes
from:

```go
s := "café"

fmt.Println(len(s))                    // 5
fmt.Println(utf8.RuneCountInString(s)) // 4
fmt.Println(s[3])                      // 195
```

`len` returns the byte count, always, in constant time. If you want letters,
the `unicode/utf8` package will walk the string and count runes for you, which
costs a full pass over the data. And indexing with `s[3]` hands you a single
byte, 195, which is the first half of "é" and not a character of any kind.

## Range knows better than index

The `for range` loop over a string is the one place where Go decodes UTF-8 for
you. Each step gives you the byte offset and the rune that starts there:

```go
for i, r := range s {
	fmt.Println(i, string(r))
}
// 0 c
// 1 a
// 2 f
// 3 é
```

Look at the indexes. They are byte offsets, so after "é" at offset 3 the next
rune, if there were one, would start at 5. People who write `for i := 0; i <
len(s); i++` and then treat `s[i]` as a character get away with it right up to
the first customer named Zoë.

When range hits bytes that are not valid UTF-8, it does not panic. It returns
the replacement character U+FFFD and moves forward one byte. Your loop keeps
going, quietly printing little question-mark diamonds into the log.

## The slice that cuts a letter in half

Slicing works on bytes too, so a truncation that looks innocent can split a
character down the middle:

```go
short := s[:4]
fmt.Printf("%q\n", short)            // "caf\xc3"
fmt.Println(utf8.ValidString(short)) // false
```

That is how a "preview" field ends up with a broken glyph at the end, or how a
database with a strict encoding rejects a row your code thought was fine. The
byte at index 3 was the opening half of "é", and the closing half got left on
the cutting-room floor.

If you need to cut by characters, convert to runes first:

```go
r := []rune(s)
fmt.Println(string(r[:4])) // café
```

A `[]rune` is a slice of `int32` values, one per code point, so indexing and
slicing it behave the way your intuition wanted. It costs an allocation and
four bytes per rune, which is fine for a title and wasteful for a log file.

One more trap lives nearby. If `n` is an `int` holding 65, `string(n)` does
not give you "65". It gives you "A", the character with that code point. Go vet
flags the conversion, and `strconv.Itoa` is the function you wanted all along.

## Which one to reach for

The rule fits on a sticky note. `len` and indexing are about bytes: storage
sizes, buffer limits, wire formats. `range` and `[]rune` are about
characters: anything a human will read, count, or truncate. When text comes
from outside your program, check it with `utf8.ValidString` before you trust
it to be text at all.

Runes still are not the end of the story. Some things a person sees as one
letter are several code points glued together, like a flag emoji or an "é"
written as "e" plus a combining accent. Counting those properly needs a
grapheme library, which is a story for another Friday.

For now, the lesson is small and stubborn. Go counted "café" correctly. It just
answered a different question than the one you asked, and it told you so in
the docs years ago.

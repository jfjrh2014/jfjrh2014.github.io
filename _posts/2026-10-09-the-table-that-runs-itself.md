---
title: "The Table That Runs Itself"
date: 2026-10-09 20:00:00 +0000
author: "Marcus H."
categories: go
---

Somewhere out there, a test function is 400 lines long and named
`TestParseInput`. Inside it, the same three lines are repeated twenty-six
times, each block differing only in the input string and the expected
output. Every new case means scrolling, copying, pasting, and hoping the
paste landed in the right place. The function works. Nobody wants to touch
it, which is a different kind of problem.

The fix has been sitting in the testing package since before Go had
modules: the table-driven test. One table of cases, one loop, one body.

```go
func TestParseInput(t *testing.T) {
	tests := []struct {
		name  string
		input string
		want  string
	}{
		{"empty", "", ""},
		{"spaces", "  hi  ", "hi"},
		{"tabs", "\thi\t", "hi"},
		{"unicode spaces", "\u00a0hi", "hi"},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := ParseInput(tt.input); got != tt.want {
				t.Errorf("ParseInput(%q) = %q, want %q", tt.input, got, tt.want)
			}
		})
	}
}
```

The plain loop already beats the copy-paste convention. But `t.Run` is
where the payoff arrives. Each entry becomes a subtest with a real name,
and the test runner treats it like a first-class citizen. `go test -run
TestParseInput/unicode` runs exactly one case. When a case fails, the
output reads `--- FAIL: TestParseInput/unicode_spaces` instead of a line
number inside a wall of repeated assertions. The failing case announces
itself.

The name field is doing more work than it looks. A table entry with the
name `"case1"` has the same information content as a lottery ticket. The
name should describe what makes this case different: the empty string, the
tab-only input, the input with a unicode space that looks identical to a
regular one but isn't. Six months later, that name is the only thing
standing between you and re-deriving the case from the failure output.

Two habits make the pattern stronger. First, keep the assertion body
identical for every case: the table holds the differences, the body holds
the logic. The moment one case needs `if` branches inside the loop, the
table has stopped being a table, and a function per case is the more honest
structure. Second, since Go 1.22 the loop variable is a fresh binding each
iteration, so the old `tt := tt` capture ritual is finally dead code you
can delete from every example you ever copied.

For cases that don't share a body, the pattern has a bigger sibling:
data-driven tests where the table maps names to closures. Same idea, one
level up. And `testing` in recent Go versions grew helper functions
(`t.Parallel()`, `t.Cleanup()`) that work per subtest, so a table can
parallelize its cases with one line each.

The honest cost: setup code that fits awkwardly in a struct column. A case
needing its own server, temp dir, or timing window fights the table. Let
those be regular test functions. The table is a tool for the 90% of cases
that differ only in data, and it shines precisely because it makes the
differences boring.

The next time a test function grows past a screen, don't add a fourth
repeated block. Add a row. Future you, running one named subtest at 2am,
will send a thank-you note through time.

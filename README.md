# kotoba-lang/org-ieee-sed — POSIX `sed`, the substitute command

The `s` command from IEEE Std 1003.1, with **literal** patterns, written in
`.kotoba` and compiled to a standalone native executable.

```sh
./sed 's/PATTERN/REPLACEMENT/'  FILE...
./sed 's/PATTERN/REPLACEMENT/g' FILE...
```

Thirty-two cases agree with `/usr/bin/sed` on stdout, stderr and exit status.

## The pattern is literal, not a regular expression

The same boundary [`org-ieee-grep`](https://github.com/kotoba-lang/org-ieee-grep)
ships with, named for the same reason: `string-index-of` finds a literal
needle and there is no regular expression engine to call. So `s/^/>/` inserts
nothing here, where sed anchors and prefixes every line.

Any script whose pattern holds a metacharacter means something different to
the two implementations, so the suite compares only literal ones — comparing
the rest would be comparing two different questions.

There are also no escapes: a pattern containing the delimiter cannot be
written. The delimiter is whatever byte follows the `s`, so `s|X|-|` works.

## What is matched, measured 2026-09-10

```
s/X/-/    the FIRST occurrence on each line, not every one
s/X/-/g   every occurrence
s/zz/-/   no occurrence leaves the line exactly as it was
s/X//     an empty replacement deletes
s//-/     an EMPTY pattern is refused, not treated as matching everywhere
```

Under `g` the scan continues **after** each replacement, not from the start of
it, so `s/a/aa/g` terminates rather than looping.

### A control that passed, which meant the suite was weaker than it looked

Breaking that scan — advancing one byte instead of past the match — passed
**all 29** cases the suite had at the time. Every pattern in it was one byte
long, which makes those two rules the same thing.

The fixture that separates them is a two-character pattern over four repeats:
`s/aa/X/g` over `aaaa` is `XX`, and advancing by one byte gives `XXXa`. With
that case present the same control fails exactly the two `g` cases, as it
should have all along.

## The unterminated last line: three utilities, three answers

An unterminated last line is closed only when another **line** follows it
somewhere in the remaining input — not merely another operand:

```
sed s/X/-/ nonl plain   ->  a-b\na-bXc…    terminated
sed s/X/-/ nonl nope    ->  a-b            not: the follower is missing
sed s/X/-/ nonl empty   ->  a-b            not: the follower has no lines
```

So sed reads its operands as one stream of lines and only the very last line
of the whole input keeps a missing terminator.
[`org-ieee-cut`](https://github.com/kotoba-lang/org-ieee-cut) answers this
differently — it preserves each operand's own termination — and
[`org-ieee-sort`](https://github.com/kotoba-lang/org-ieee-sort) differently
again, terminating every operand before appending the next. Each was measured
on its own utility rather than carried across from a sibling.

The lookahead that decides this runs only when an operand actually ends
without a newline, so the extra reads are paid by the inputs that need the
answer.

## Capabilities

`:cli/args` (38), `:fs/app-data` (35), `:io/write` (37), `:io/write-error`
(39). A missing operand is reported byte-for-byte and the readable ones are
still written, exit 1.

## What this is not

Only `s`. No addresses (`1,3s/…`), no `-n`, `-e`, `-i`, `-E`, no `p`/`d`/`y`
commands, no `&` or `\1` in the replacement, no regular expressions. A bad
substitute flag (`s/X/-/q`, `s/X/-/gg`) is refused with the generic
`sed: unsupported script`, exit 1, where `/usr/bin/sed` names the script and
the flag — a named divergence in the suite (until 2026-09-16 the flag was not
checked at all).

## Standard input

With a script and no file operand `sed` reads standard input (wire 41
`:io/read`, 2026-09-16) — 52% of how it is invoked in agent tool use (6,355
of 12,267 over 1,268,018 measured Bash calls; `grep | sed` alone is 1,163).
A single input, so a missing final newline stays missing, as `/usr/bin/sed`
does. Whole-input form: input larger than the binary's string pool is
refused (exit 120), never edited short.

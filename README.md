# regex

Regular expressions for Fin — a small, correct regex engine built on a
**Pike VM** (Thompson NFA simulation). No exponential blowup on nested
quantifiers: `(a+)+$` on a long non-matching string runs in linear time,
where a backtracker would hang.

> **Verification status:** VERIFIED — `tests/test_regex.fin` passes against
> `finc 0.4.0` (contract 3), the example runs, and ~3,700 differential
> fuzz cases against Python's `re` agree on every supported construct
> (modulo the three documented semantic differences below).

## Install

Requires the `finn` package manager (https://github.com/M1778/finn):

```sh
finn add M1778/regex
```

## Quick Example

```fin
import { Match, regex_search, regex_group_text, regex_replace_all } from regex::fin;
import { free_str } from strings::std;

fun main() <noret> {
    let m <Match> = regex_search("(\\d+)-(\\d+)", "call 555-1234 now");
    if (m.found) {
        let area <string> = regex_group_text(&m, "call 555-1234 now", 1);
        printf("area code: %s\n", area);   // area code: 555
        free_str(area);
    }
    let redacted <string> = regex_replace_all("\\d", "call 555-1234", "X");
    printf("%s\n", redacted);              // call XXX-XXXX
    free_str(redacted);
}
```

> Import spelling: `regex::fin` means module `regex` (the file
> `src/regex.fin`), namespace `fin`. Import the names you need **by name** —
> qualified calls (`regex.regex_search(...)`) are refused by the backend; see
> `docs/guide/10-modules-and-imports.md` in the Fin repo.

## Supported syntax

| construct | meaning |
|---|---|
| `abc`, `\*`, `\.`, `\\`, … | literals; backslash escapes any punctuation |
| `.` | any char except `\n` |
| `*`, `+`, `?` | greedy repetition; lazy forms `*?`, `+?`, `??` |
| `{m}`, `{m,}`, `{m,n}` | bounded repetition (`m` ≤ 32, `n` ≤ 64) |
| `[abc]`, `[a-z]`, `[^0-9]` | classes, ranges, negation; `]`/`-` literal in edge positions |
| `\d \D \w \W \s \S` | digit / non-digit / word / non-word / space / non-space (also inside `[...]`, except `\D \W \S` which are rejected there) |
| `\n \r \t`, `\b`, `\B` | newline/CR/tab escapes, word boundary / non-boundary |
| `(...)`, `(?:...)` | capturing / non-capturing groups |
| `\|` | alternation (empty branches allowed: `a\|`, `\|b`) |
| `^`, `$` | start / end of text (not multiline) |

**Not in v0.1.0:** backreferences, lookaround (`(?=…)` is a compile error),
POSIX classes, `\x`/`\u` escapes, multiline mode, Unicode properties.
Invalid patterns make the matching functions `blame`; pre-check
user-supplied patterns with `regex_valid`.

## API Reference

```fin
// Match result. spans is flattened [s0,e0,s1,e1,...] (group 0 = whole
// match); -1/-1 for a group that did not participate. Empty when !found.
pub struct Match { found <bool>; start <int>; end <int>;
                   ngroups <int>; spans <&Collection<int>> }

pub fun regex_valid(pattern: string) <bool>
// True when pattern compiles. Use to pre-check user input.

pub fun regex_is_match(pattern: string, text: string) <bool>
// True when pattern matches the ENTIRE text.

pub fun regex_search(pattern: string, text: string) <Match>
pub fun regex_search_from(pattern: string, text: string, from: int) <Match>
// Leftmost-first match (from = byte index to start at, clamped).

pub fun regex_find_all(pattern: string, text: string) <&Collection<int>>
// All non-overlapping matches as flattened [start,end,...] pairs.
// Empty matches advance one char (no hangs, no duplicates).

pub fun regex_replace_first(pattern: string, text: string, repl: string) <string>
pub fun regex_replace_all(pattern: string, text: string, repl: string) <string>
// Replacement supports $0..$9 (group text; unmatched groups expand to
// empty) and $$ (literal $). Empty matches consume one char so
// replacements like "x*" terminate. Results are heap-allocated.

pub fun regex_split(pattern: string, text: string) <&Collection<string>>
// Split on matches; empty matches are skipped, not split on.

pub fun regex_group_count(m: &Match) <int>
pub fun regex_group_start(m: &Match, i: int) <int>   // -1 if absent
pub fun regex_group_end(m: &Match, i: int) <int>     // -1 if absent
pub fun regex_group_text(m: &Match, text: string, i: int) <string>
// Group 0 = whole match; "" when the group did not participate.
// Heap-allocated — free with strings::std.free_str.
pub fun regex_match_text(m: &Match, text: string) <string>
// Whole-match text; heap-allocated, free with free_str.
```

Groups are numbered by opening-paren order, like most engines.
`Match.spans` is owned by the `Match` value; there is no free API for it in
v0.1.0 (one small collection per search).

## Semantics

Leftmost-first (Perl-like), verified by differential fuzzing against
Python's `re`. Three **deliberate** differences from Python:

1. **`$` is end-of-text only.** Python's `$` also matches just before a
   trailing newline; `regex` follows Go/Rust/RE2 here. (So `a$` does not
   match `"a\n"`.)
2. **Empty-iteration captures.** If a group matches empty inside a
   repetition (`(|a)*`), Python records the group's empty span while the
   Pike VM drops it (reports -1/""). The overall match span still agrees.
3. **Empty-iteration exploration.** Python stops a repetition as soon as
   one iteration matches empty; the Pike VM keeps exploring the
   repetition's other alternatives in that iteration (RE2 behaves the same
   way). Overall spans can differ in pathological patterns like
   `(b*|\w)*`.

## finc 0.4.0 workarounds used (for the Fin testers)

Building this library surfaced four new backend quirks, all worked around
in the source and all reproducible:

- **#13 — `struct.field[i] = v` does not lower through an import**
  (`codegen: an assignment to this target is not lowered yet`,
  misattributed to the importing file). Workaround: `re_coll_set`
  helper taking the collection as a plain parameter.
- **#14 — forward reference to a function with a generic-typed parameter
  fails** (`Undefined function or type`). Affects mutually recursive
  functions (the parser's alt→concat→piece→atom→alt cycle). Workaround:
  non-generic wrapper struct `ReOuts` for the `&Collection<int>` outs
  parameter. Plain, `[char]`, and user-struct params forward-reference
  fine.
- **#15 — `from` is a keyword** (like the previously known `in`,
  `match`). Renamed to `frm`.
- **#16 — multi-line struct literal with trailing comma fails to parse**
  (confirms the day-2 finding: no trailing commas in struct literals).

Plus two engine bugs caught by real testing (not review): the compiled
program's **entry pc is not 0** when the pattern starts with a quantified
piece (`a*b` on `"b"` failed until `prog.entry` was threaded through),
and the alternation SPLIT chain **dangled its last split to after-alt**,
creating a spurious empty alternative (`a|b` matched `""`).

## Performance

Pike VM = linear time in `len(text) × len(program)` worst case.
`(a+)+$` and `(a|aa)+$` on 30 a's + `b` (classic catastrophic input)
return instantly; the test suite asserts this.

## License

MIT

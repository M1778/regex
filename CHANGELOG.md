# Changelog

All notable changes to this project will be documented here.

## [0.1.0] — 2026-10-21

### Added
- Pike VM regex engine (Thompson NFA simulation, linear-time, no
  catastrophic backtracking): literals, `.`, `* + ?` (greedy + lazy),
  `{m}`, `{m,}`, `{m,n}` (m ≤ 32, n ≤ 64), classes with ranges/negation,
  `\d \D \w \W \s \S \n \r \t \b \B`, capturing `(...)` and
  non-capturing `(?:...)` groups, alternation (incl. empty branches),
  `^`/`$` anchors.
- Public API: `regex_valid`, `regex_is_match`, `regex_search`,
  `regex_search_from`, `regex_find_all`, `regex_replace_first`,
  `regex_replace_all` ($0..$9, $$), `regex_split`,
  `regex_group_count/_start/_end/_text`, `regex_match_text`.
- Full lifecycle: README (API docs + semantics + finc workarounds),
  `tests/test_regex.fin` (all passing against finc 0.4.0),
  `examples/basic.fin` (verified output), GitHub Actions CI (builds
  finc from source), `finn.toml`, MIT license.
- Differential fuzzing vs Python `re` (~3,700 cases): agreement on all
  supported constructs modulo three documented semantic differences
  (`$` end-only; empty-iteration capture spans; empty-iteration
  alternative exploration).

### Known limitations
- No backreferences, lookaround, POSIX classes, `\x`/`\u` escapes,
  multiline mode, Unicode properties.
- `\D \W \S` rejected inside `[...]` (documented; use negation instead).
- `Match.spans` has no free API in v0.1.0.

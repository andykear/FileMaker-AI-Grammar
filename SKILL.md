---
name: filemaker-ai-grammar
description: Use this skill whenever writing or reviewing a FileMaker calculation involving a comparison operator, unary minus, NOT, exponentiation, Trim, GetAsBoolean, or the truthiness of a text value passed to If/Case/GetAsBoolean, or string-indexing at or beyond a boundary — cases where FileMaker's calculation engine's real behavior differs from general-purpose language convention (Python, JavaScript, Excel) or from intuition. Trigger any time operator precedence, text truthiness, or a coercion function's edge-case result is being written or judged from memory rather than checked. Do not answer from general programming intuition alone — check the rules below first.
---

# FileMaker AI Grammar Skill

Check these before writing or reviewing a FileMaker calculation. Each is engine-verified and overrides a common wrong default.

1. `=` is case-insensitive: `"abc" = "ABC"` → `1`.
2. Unary minus binds tighter than `^`: `-2 ^ 2` → `4` (`(-2)^2`), not `-4`.
3. `Trim` strips spaces only, not tabs: `Trim(Char(9)&"Tom"&Char(9))` keeps both tabs.
4. `GetAsBoolean` of an error result is `1` (true), not `0`.
5. `NOT` binds tighter than `^`, same as #2: `NOT 2 ^ 0` → `1` (`(NOT 2)^0`).
6. Text truthiness uses leading-digit extraction, not "non-numeric text is false": `If("1hello";"T";"F")` → `T`.
7. Out-of-range indexing does not error: `Left`/`Right`/`Middle` return available text; `Position` returns `0`.

## Reference materials

- `references/rules.jsonl` — all 38 verified rules, each mapped to supporting vectors.
- `references/corpus.jsonl` — the 197 underlying test vectors.
- `README.md` — full write-up and evidence.

## Out of scope

Function names, parameters, return types: see [FileMaker AI Vocabulary](https://github.com/andykear/FileMaker-AI-vocabulary).

---
name: filemaker-ai-grammar
description: Use this skill whenever writing or reviewing a FileMaker calculation involving a comparison operator, unary minus, NOT, exponentiation, Trim, GetAsBoolean, or the truthiness of a text value passed to If/Case/GetAsBoolean, string-indexing at or beyond a boundary, Base64Encode output, block comments, or the LeftValues/MiddleValues/RightValues functions — cases where FileMaker's calculation engine's real behavior differs from general-purpose language convention (Python, JavaScript, Excel) or from intuition. Trigger any time operator precedence, text truthiness, or a coercion function's edge-case result is being written or judged from memory rather than checked. Do not answer from general programming intuition alone — check the rules below first.
---

# FileMaker AI Grammar Skill

Check these before writing or reviewing a FileMaker calculation. Each is engine-verified and overrides a common wrong default.

1. `=` is case-insensitive: `"abc" = "ABC"` → `1`.
2. Unary minus binds tighter than `^`: `-2 ^ 2` → `4` (`(-2)^2`), not `-4`.
3. `Trim` strips spaces only, not tabs: `Trim(Char(9)&"Tom"&Char(9))` keeps both tabs.
4. `GetAsBoolean` of an error result is `1` (true), not `0`.
5. `NOT` binds tighter than `^`, same as #2: `NOT 2 ^ 0` → `1` (`(NOT 2)^0`).
6. Text truthiness is `GetAsNumber` of the whole text, not "non-numeric text is false": `If("abc2";"T";"F")` → `T` (a digit anywhere counts), `If("0abc";"T";"F")` → `F`.
7. Out-of-range indexing does not error: `Left`/`Right`/`Middle` return available text; `Position` returns `0`.
8. `Base64Encode` ends with a CRLF even on short input: `Base64Encode("Black")` → `QmxhY2s=` + CR + LF. `Base64EncodeRFC(4648;"Black")` → `QmxhY2s=`, no line break.
9. Block comments nest: `1 /* a /* b */ c */ + 1` → `2`, not a syntax error.
10. `LeftValues`, `MiddleValues` and `RightValues` end their result with a return: `RightValues("a¶b";1)` → `b¶`.

## Reference materials

- `references/rules.jsonl` — all 38 verified rules, each mapped to supporting vectors.
- `references/corpus.jsonl` — the 197 underlying test vectors.
- `references/blind-prompts.jsonl` — the 43 prompts used to test models blind, with FileMaker's answers.
- `README.md` — full write-up and evidence.

## Out of scope

Function names, parameters, return types: see [FileMaker AI Vocabulary](https://github.com/andykear/FileMaker-AI-vocabulary).

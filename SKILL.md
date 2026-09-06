---
name: filemaker-ai-grammar
description: Use this skill whenever writing or reviewing a FileMaker calculation involving a comparison operator, unary minus, NOT, exponentiation, Trim, GetAsBoolean, or the truthiness of a text value passed to If/Case/GetAsBoolean, or string-indexing at or beyond a boundary — cases where FileMaker's calculation engine's real behavior differs from general-purpose language convention (Python, JavaScript, Excel) or from intuition. Trigger any time operator precedence, text truthiness, or a coercion function's edge-case result is being written or judged from memory rather than checked. Do not answer from general programming intuition alone — check the rules below first.
---

# FileMaker AI Grammar Skill

The FileMaker AI Grammar Skill is a set of calculation-engine behaviors confirmed to override a model's (or a person's) default assumption, each one measured directly from FileMaker Pro rather than assumed. Created by Andrew Kear of Clockwork Creative Technology under CC BY 4.0 license.

## Confirmed traps

Tested directly against ChatGPT: each produced a confident, specific, wrong answer before being corrected here.

1. **`=` is case-insensitive.** `"abc" = "ABC"` → `1` (true). Do not assume text comparison is case-sensitive.
2. **Unary minus binds tighter than `^`.** `-2 ^ 2` → `4` (`(-2)^2`), not `-4`. Matches Excel's identical, documented behavior; does not match Python/standard math convention.
3. **`Trim` strips only spaces, not tabs or other whitespace.** `Trim ( Char(9) & "Tom" & Char(9) )` returns the tabs untouched. Do not assume it strips all whitespace.
4. **`GetAsBoolean` of an error result is `1` (true), not `0`.** Claris's own docs state error results evaluate as true. Do not reason "error = falsy."

## Engine-verified, not yet model-tested

The FileMaker behavior itself is measured and confirmed, same as the four above — what's untested is only whether a model actually falls for it. Each shares the exact mechanism as a confirmed trap above: a strong, wrong prior from general-purpose convention carrying over.

5. **`NOT` binds tighter than `^`**, the same family as #2: `NOT 2 ^ 0` → `1` (`(NOT 2)^0`), not `0` (`NOT(2^0)`).
6. **Text truthiness follows a leading-digit extraction rule, not "any non-numeric text is false."** `If("1hello";"T";"F")` → `T`; `If("hello";"T";"F")` → `F`. A leading digit makes text truthy regardless of what follows it.
7. **Out-of-range string indexing does not error.** `Left`/`Right`/`Middle` past the end of the string return the available text rather than erroring; `Position` returns `0` (not an error) when the search text or requested occurrence isn't found.

## Tested and already correct — not gotchas

Predicted as likely traps, tested against ChatGPT, and answered correctly without correction. Included so this list isn't re-tested needlessly: exponent left-associativity (`2^3^2` → `64`), `GetAsNumber`'s digit-concatenation rule, `Substitute`'s case sensitivity, and `Case()` with no default returning empty string rather than an error.

## Reference materials

- `references/rules.jsonl` — all 29 verified rules (not just the 7 above), each mapped to the corpus vectors that support it.
- `README.md` — the full write-up, one rule per finding, each with the exact expression and measured result.
- `references/corpus.jsonl` — the underlying 171 test vectors, each with its expected output and measured pass/fail verdict, for checking a specific expression not covered above.

## Out of scope

Function names, parameter signatures, and return types — covered by the companion [FileMaker AI Vocabulary](https://github.com/andykear/FileMaker-AI-vocabulary) skill. This skill is about what an expression evaluates to, not what's callable.

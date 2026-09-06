# FileMaker AI Grammar

[![License](https://img.shields.io/badge/license-CC%20BY%204.0-green)](https://creativecommons.org/licenses/by/4.0/)

**171 test vectors covering FileMaker's calculation engine, each one measured against real FileMaker Pro rather than assumed.**

FileMaker's calculation language has no published formal grammar. Claris does maintain a full operator order-of-evaluation table, at `operators-in-formulas.html`, just not cross-linked from the four separate operator-category pages.

This corpus takes a different approach from writing a grammar down: instead of describing it, it tests it. Every vector is a calculation expression, round-trip tested against a real FileMaker Pro file, with the actual returned value logged as the expected output. Re-running the harness against a new FileMaker version turns any behavior change into a diffable fact rather than a research question.

Developed by Andrew Kear of Clockwork Creative Technology and shared openly with the FileMaker/Claris community.

## The problem it solves

FileMaker's calculation engine has real, specific behaviors that don't match general-purpose language conventions or intuition — unary minus binding tighter than exponentiation, `Trim` stripping only spaces, `Substitute` being case-sensitive while `Position` isn't. Several of these are stated in Claris's own help pages; they're just easy to miss, or to contradict with intuition built from other languages. A model (or a person) reasoning from general programming conventions gets a predictable subset of these confidently wrong, with no signal that anything is off:

> *"-2 ^ 2 evaluates to -4. Exponentiation happens before the unary minus: -(2 ^ 2) = -4."* — ChatGPT, wrong on both the answer and the stated rule. FileMaker's unary minus binds tighter than `^`; the real answer is `4`.

> *"GetAsBoolean() returns False for an error result... GetAsBoolean ( ? ) evaluates to False."* — ChatGPT, confident and specific. Claris's own docs state error results evaluate as true; the measured result is `1`.

### Tested against ChatGPT

All 8 predicted traps below were run against ChatGPT, logged honestly rather than cherry-picked: 4 produced a confirmed wrong answer, 4 the model already had right.

| Prompt | Predicted wrong answer | Actual | Result |
|---|---|---|---|
| `"abc" = "ABC"` | False | `1` (true) | **confirmed trap** |
| `-2 ^ 2` | `-4` | `4` | **confirmed trap** |
| `2 ^ 3 ^ 2` | `512` | `64` | already correct |
| `Trim ( Char(9) & "Tom" & Char(9) )` | `"Tom"` | tabs survive | **confirmed trap** |
| `GetAsNumber ( "12abc34" )` | `12` | `1234` | already correct |
| `Substitute ( "ABC" ; "abc" ; "xyz" )` | `xyz` | `ABC` (no match) | already correct |
| `GetAsBoolean` of an error result | False | `1` (true) | **confirmed trap** |
| `Case ( 0 ; "a" )`, no default | error/null | `""`, no error | already correct |

No clean pattern in which ones land — `=` being case-insensitive is arguably the single most well-known FileMaker fact there is, and it still tripped the model up, while more obscure ones didn't. Worth treating each new candidate as untested until actually run, not reasoning from a guessed pattern.

## What's inside

- **`SKILL.md`** — a curated subset of 7 rules confirmed or strongly motivated to override a model's default assumption, formatted to load as a Claude skill.
- **`README.md`** — this file, which also carries the full write-up below.
- **`references/corpus.jsonl`** — 171 test vectors. Each is a calculation `Expression`, an `ExpectedOutput`, the comparison rule (`Exact()` for all 171), and a `Verdict` from the most recent run against FileMaker Pro. All pass.
- **`references/rules.jsonl`** — 29 named rules, each mapped to the corpus vectors that support it — the layer between raw pass/fail data and a stated fact.

```
Round ( 14.5 ; 0 )               → 15    (round-half-up, not round-half-to-even)
-2 ^ 2                           → 4     (unary minus binds tighter than ^)
Substitute ( "ABC" ; "abc" ; "xyz" )   → ABC    (case-sensitive, unlike Position)
```

## Deliberately just the behavior

This corpus handles what calling a function or operator actually does — the return value, the edge case, the coercion rule. It deliberately excludes what's callable at all: the full list of function names, parameter signatures and return types already exists in the companion **[FileMaker AI Vocabulary](https://github.com/andykear/FileMaker-AI-vocabulary)**, so this corpus doesn't duplicate it. Where a finding here depends on a function's documented signature, it names the function rather than re-specifying it.

## Target: FileMaker 26

Tested against FileMaker Pro 26.0.2 on macOS, day/month regional date format. One version tested; rerunning this harness on a new release turns any version difference into a diffable fact rather than a research project.

## Use it

Load the release zip into Claude's skills (keep the folder structure, so `references/` comes with it; enable code execution and file creation) to have the 7 highest-confidence traps checked automatically while a calculation is written or reviewed, with no special instruction needed. For anything beyond those 7, reference `references/rules.jsonl` or the findings below directly, in place of relying on memory or a general-purpose model's default assumptions about how an expression will evaluate.

```
"Does FileMaker's Trim() strip tabs as well as spaces?"    → references/rules.jsonl: trim_only_strips_spaces
"What does -2^2 evaluate to in FileMaker?"                  → references/rules.jsonl: unary_minus_binds_tighter_than_exponent
"Is Substitute case-sensitive?"                             → references/rules.jsonl: substitute_case_sensitive_position_not
```

`references/corpus.jsonl` is the ground truth behind every rule below — read it directly to check a finding's exact expression and measured result, or to extend the corpus with a new vector of the same shape.

## Verified findings

**Unary minus binds tighter than exponentiation**: `-2 ^ 2` → `4` (`(-2)^2`), not `-4` (`-(2^2)`) — opposite of Python/standard mathematical convention. Matches Excel's identical, documented behavior.

**FileMaker suppresses the leading zero on a fractional number with no integer part**: `2 ^ -2` → `.25`, not `0.25`.

**`GetAsBoolean` treats an error result as true**: `GetAsBoolean("?")` → `1`, and `GetAsBoolean(Evaluate("1+"))` (a real syntax error) → `1` — Claris's own docs state error results are evaluated as true. Boolean coercion of ordinary text follows the same leading-digit rule as `GetAsNumber`/`If`: `"1hello"` → `1`, `"Some text here."` → `0`.

**`GetAsNumber` concatenates digits rather than truncating at the first non-numeric character**: `GetAsNumber("12abc34")` → `1234`, matching Claris's own documented example `GetAsNumber("2 - 2")` → `22`.

**Numeric truthiness follows the same digit-extraction rule**: `If("hello";"T";"F")` → `F`; `If("1hello";"T";"F")` → `T`. `Case()`'s test parameter follows the identical rule: `Case("";"true-ish";"default")` → `default` (empty string is falsy), `Case(5;"true-ish";"default")` → `true-ish` (any non-zero number is truthy).

**Comparison is case-insensitive by default**: `"abc" = "ABC"` → `1`.

**`Let()` evaluates bindings left to right and allows shadowing**: `Let([x=1;x=x+1];x)` → `2`. A nested `Let` reverts its outer scope on exit: `Let(x=1;Let(x=2;x)&x)` → `"21"` — the inner binding is scoped only to the inner `Let`. A `$$global` variable set inside a `Let` persists after it, unlike a local variable.

**`Case()` with no default returns empty string**, not an error, and evaluates tests in order, stopping at the first true one even when a later test is also true.

**1-based string indexing, with several boundary behaviors**: `Middle("FileMaker";1;4)` → `File`; position `0` behaves the same as position `1`. Requesting more characters than exist (`Left`/`Right`) returns the whole string. `Position` returns `0` (not an error) when the search text isn't found, when the requested occurrence doesn't exist, or when occurrence `0` is requested; a negative occurrence count scans backward from the start position. A start position of `0` or negative clamps to the beginning of the string.

**`Substitute` is case-sensitive** (`Substitute("ABC";"abc";"xyz")` → `ABC`, no match), unlike `Position`/`PatternCount`, which are not — a real, easy-to-miss inconsistency between functions that otherwise seem like a family. Multiple substitution pairs apply in sequence to the already-substituted string, not simultaneously: `Substitute("ab";["a";"b"];["b";"c"])` → `"cc"`.

**`Trim` strips only spaces, not tabs**: `Trim(Char(9)&"Tom"&Char(9))` returns the tabs untouched. Most languages' equivalent trims all whitespace; FileMaker's does not.

**Block comments nest**, matching Claris's own documentation statement that they can: `1 /* outer /* inner */ still inside */ + 1` evaluates as if both comment markers were stripped as one unit, not closing at the first `*/`.

**List/separator functions recognize CR and LF as equivalent separators**, and `ValueCount` correctly counts values with or without a trailing separator.

**A timestamp's serial number decomposes exactly as `(date serial − 1) × 86400 + time serial`** (`Timestamp(Date(1;1;1);Time(13;53;20))` and `Timestamp(Date(10;21;2019);Time(9;10;30))` both match the formula to the exact integer). `Date(1;1;1)`'s own serial number is `1`; `Time()`'s serial number is total seconds since midnight (`Time(1;0;0)` → `3600`).

**`EvaluationError` codes, matched against `error-codes.md`**: unbalanced parenthesis → `1207`; too many parameters → `1202`; calling a function with *zero* arguments where at least one is required → `1204` ("Number, text constant, field name, or '(' expected"), not `1201` ("too few parameters"), which applies when some but not enough arguments are given; `1205` unterminated comment; `1206` unterminated string; two adjacent literals with no operator between them → `1212` ("An operator ... is expected here"), a more specific code than the more general `1208`. `EvaluationError` returns `0` for a valid expression, and returns `0` incorrectly when wrapping a variable that already holds an `Evaluate()` result instead of wrapping the `Evaluate()` call itself.

**Standard arithmetic precedence holds** (`*`/`^` bind tighter than `+`; `&` binds looser than `+`; comparison binds looser than `AND`; `AND` binds tighter than `OR`), and `//`/`/* */` comments are stripped correctly at the start, middle, or end of an expression, including inside a comment that itself contains quote characters. Comment markers appearing inside an actual string literal are correctly treated as literal text, not as real comment syntax.

**`NOT` binds tighter than `^`**: `NOT 2 ^ 0` → `1`, i.e. `(NOT 2) ^ 0`, not `NOT (2^0)` (`NOT 1` = `0`) — consistent with unary operators generally binding tightly in FileMaker (see the unary-minus finding above). Depends on `0 ^ 0` → `1`, tested directly as its own vector rather than left as an assumption inside this one.

**`^` is left-associative, not right-associative**: `2 ^ 3 ^ 2` → `64` (`(2^3)^2`), not `512` (`2^(3^2)`) — the opposite of the convention most general-purpose languages use for exponentiation (Python's `**` is right-associative).

**Comparison chains and `OR`/`XOR` are ordinary same-tier left-to-right evaluation, not special-cased**: `1 < 2 < 3` → `1`, evaluating as `(1<2) < 3` rather than Python-style chained comparison. `1 OR 0 XOR 1` → `0`, evaluating as `(1 OR 0) XOR 1`, since `OR` and `XOR` sit at the same precedence tier.

**Numeric literal tokenizing**: a leading `+` sign is accepted as a no-op sign; scientific notation (`1e3`) is accepted and computed correctly; a 30-digit integer literal is accepted with no precision loss. A literal with more than one decimal point, a trailing decimal point with no digits after it (asymmetric with a leading decimal point and no digits before it, which *is* accepted), and digit-group underscores (`1_000`) are all rejected outright, not silently reinterpreted.

**`GetAsDate`/`GetAsTime`/`GetAsTimestamp` outside their documented valid range**: `GetAsDate` returns an error for a serial number or text outside year 1–4000, rather than clamping or wrapping. `GetAsTime` passes an out-of-range hour (`"25:00:00"`) through unchanged as text rather than correcting or rejecting it, and discards the am/pm designator entirely (`"12:15 pm"` → `12:15`, indistinguishable from `12:15 am`) — measured on this file's 24-hour regional format, so flagged as possibly locale-dependent rather than confirmed engine-wide; not yet tested on a 12-hour-locale file.

**U+2028/U+2029 work as `ValueCount` separators**: both count as a real separator, same as CR/LF. A bare `Char(8232)`/`Code()` round-trip preserves the character correctly. Adjacent CR and LF count as two separate separators, not one combined one.

**`EvaluationError` on a missing field or divide-by-zero surfaces the same codes documented generally**: a nonexistent field reference → `102` (Field is missing); `1/0` → `15` (Can't divide by zero).

**A field name containing a space is referenced directly in a calc with no special quoting** (`Test::Line Total`, per Claris's own documented convention). **Bracket syntax accesses a specific repetition of a repeating field**, and **`List()` over a repeating field returns a CR-delimited list of its repetition values in order**.

**Adversarial pairs**: four of the findings above are paired against an explicit contrasting control, both in `references/corpus.jsonl` — `Trim` stripping surrounding spaces (`trim_strips_spaces_control`) against tabs surviving; explicit parentheses forcing the grouping unary-minus's default binding skips (`unary_minus_explicit_parens_control`, `-(2^2)` → `-4`) and forcing right-to-left exponent grouping (`exponent_explicit_right_assoc_control`, `2^(3^2)` → `512`); `Position` finding a case-different match (`position_case_insensitive_control`) against `Substitute`'s case-sensitive miss on the same kind of input.

**`Round` always rounds up at 0.5** (round-half-up, not round-half-to-even): `Round(14.5;0)` → `15`. On a negative half it rounds away from zero, not toward positive infinity: `Round(-14.5;0)` → `-15` — the positive case alone can't distinguish those two conventions, since they only diverge below zero. A negative precision argument rounds to tens/hundreds/thousands: `Round(29343.98;-3)` → `29000`.

**`Log(0)` and `Lg(0)` both error, despite their own docs saying 0 "returns nothing"** — that phrase means "produces no valid result" (an error, `?`), not literally an empty string. `Ln(0)` also errors, consistent with its own docs, which explicitly name `0` an error case rather than using the more ambiguous "returns nothing" wording `Log`/`Lg` use for the same input.

**`Tan(Radians(90))` does not error**, despite `Tan`'s own docs stating 90 degrees "cannot be used" — it returns a specific, very large floating-point number (`16331239353195370`), consistent with `Radians(90)` not landing exactly on the mathematical asymptote in double-precision floating point.

## Methodology notes

`Expression` is stored text that `Evaluate()` parses as a second, independent formula. A literal control character (`¶`, CR, LF, etc.) must be produced by a live function call inside that stored formula's own text — `"a" & Char(13) & "b"` — not embedded as a raw byte, which is treated as an insignificant line-break in the calculation editor's multi-line sense rather than as string content.

`Date()`, `Time()`, `Timestamp()` and the `GetAsDate`/`GetAsTime`/`GetAsTimestamp` family return values whose *displayed* text (day/month order, zero-padding, 12-hour-with-AM/PM vs. 24-hour) depends on the file's or system's regional date format. Asserting an exact formatted string is not a portable test, so every date/time vector in this corpus instead compares via `GetAsNumber()` (the locale-independent serial number) against an equivalently-constructed expression, never against a literal formatted string.

171 vectors across 24 subcategories. `List()`/repeating-field functions and field references containing spaces are covered via dedicated fields on the source `Test` table (a space-named field, and a 3-repetition number field) — no relationship or separate schema needed. Not yet covered: locale decimal-separator parsing (needs a system-level regional settings change, not a schema addition).

## Related FileMaker resources

**[FileMaker AI Vocabulary](https://github.com/andykear/FileMaker-AI-vocabulary)** — every FileMaker function and script step's name, signature and return type, compressed for use as resident context. This corpus is its behavioral companion: the vocabulary says what's callable, this says what calling it actually does.

**[Script XML Skill](https://github.com/andykear/FileMaker-XMLsnippet-Claude-Skill)**, **[Layout XML Skill](https://github.com/andykear/FileMaker-XMLsnippet-Layout-Claude-Skill)**, **[Field, Table & Value List Definitions](https://github.com/andykear/FileMaker-XML-field-definitions)**, **[XML Inspector](https://github.com/andykear/FileMaker-XML-inspector-open-source)**, **[XML Scrubber](https://github.com/andykear/FileMaker-XML-scrubber)** — companion resources on FileMaker's clipboard XML formats.

## Provenance

Every expected output was measured directly from FileMaker Pro, round-trip tested rather than transcribed from documentation or assumed from other languages' conventions. Where a result confirms or corrects something Claris's own help pages state, both are cited above.

## Licence & contributing

CC BY 4.0 — available for use, sharing, and adaptation with attribution. Corrections, missing edge cases, or additional vectors welcome via issues or pull requests.

## Version history

| Version | Notes |
|---|---|
| 1.0 | FileMaker 26.0.2 baseline. 171 test vectors, 29 rules, 8 AI-gotcha demos, 7-rule SKILL.md. |

---

*Clockwork Creative Technology — clockworkct.co.uk · github.com/andykear. Bespoke FileMaker development, automated artwork systems, and hosted solutions. Working on something and need a hand? Get in touch.*

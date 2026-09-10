[![Stars](https://img.shields.io/github/stars/andykear/FileMaker-AI-Grammar?style=social)](https://github.com/andykear/FileMaker-AI-Grammar)
[![License](https://img.shields.io/badge/license-CC%20BY%204.0-green)](https://creativecommons.org/licenses/by/4.0/)

# FileMaker AI Grammar

**197 test vectors · 38 rules · 7 high-confidence AI traps · tested against real FileMaker Pro 26.0.2.**

Every vector here is a calculation expression run against a real FileMaker file, with the value it actually returned logged as the expected output — not assumed, not guessed, not carried over from another language's convention.

Developed by Andrew Kear of Clockwork Creative Technology and shared openly with the FileMaker/Claris community.

## The problem

FileMaker's calculation language has no published formal grammar. Claris does maintain a full operator order-of-evaluation table, at `operators-in-formulas.html`. It just isn't cross-linked from the four separate operator category pages that describe the individual operators.

So this repo tests instead of describing. Every vector is round-trip tested against a real FileMaker Pro file, and the value it actually returned is what gets logged as the expected output.

That matters because FileMaker's calculation engine has real, specific behavior that doesn't match general-purpose language convention, or plain intuition. Unary minus binds tighter than exponentiation. `Trim` strips only spaces. `Substitute` is case sensitive while `Position` isn't. Some of this is stated in Claris's own help pages — it's just easy to miss, or easy to contradict with intuition carried over from another language. A model reasoning from general programming convention gets a predictable slice of this confidently wrong, with nothing to signal that anything is off.

### Tested against ChatGPT, Claude, and Fable

Every prompt below was run blind — no access to this repo, no skill loaded, no tools, a cold read from the model's own training — and logged honestly rather than cherry picked, all three models on the exact same 21 prompts. That first pass asked all 21 questions together, in one chat, as a batched quiz. To check the answer wasn't just an artefact of that framing, the 13 questions that came back wrong or shaky were asked again, varying the prompt — one question per call on its own, phrased plainly rather than as a quiz item — 3 independent samples per question per model. The Claude cells for those 13 rows show that second round as a fraction — samples correct out of 3 — instead of a single ✅/❌.

| Prompt | Actual | ChatGPT (free) | Claude Sonnet 5 | Claude Opus 5 | Claude Fable 5.1 |
|---|---|---|---|---|---|
| `"abc" = "ABC"` | `1` (true) | ❌ False | ✅ | ✅ | ✅ |
| `-2 ^ 2` | `4` | ❌ `-4` | ✅ | ✅ | ⚠️ 1/3 |
| `2 ^ 3 ^ 2` | `64` | ✅ | ✅ | ✅ | ✅ |
| `Trim ( Char(9) & "Tom" & Char(9) )` | tabs survive | ❌ stripped | ✅ | ✅ | ✅ |
| `GetAsNumber ( "12abc34" )` | `1234` | ✅ | ⚠️ 1/3 | ✅ | ⚠️ 2/3 |
| `Substitute ( "ABC" ; "abc" ; "xyz" )` | `"ABC"` | ✅ | ✅ | ✅ | ✅ |
| `GetAsBoolean` of an error result | `1` (true) | ❌ False | ⚠️ 2/3 | ❌ 0/3 | ⚠️ 2/3 |
| `Case ( 0 ; "a" )`, no default | `""` | ✅ | ✅ | ✅ | ✅ |
| `NOT 2 ^ 0` | `1` | ❌ `0` | ❌ 0/3 | ❌ 0/3 | ❌ 0/3 |
| `0 ^ 0` | `1` | ✅ | ✅ | ✅ | ✅ |
| `If ( "abc2" ; "yes" ; "no" )` | `"yes"` | ❌ `"no"` | ✅ | ✅ | ✅ |
| `If ( "0abc" ; "yes" ; "no" )` | `"no"` | ❌ `"yes"` | ✅ | ✅ | ✅ |
| `Middle ( "abc" ; 10 ; 5 )` | `""` | ✅ | ✅ | ✅ | ✅ |
| `GetValue ( List ( "a" ; "b" ) ; 5 )` | `""` | ✅ | ✅ | ✅ | ✅ |
| `Round ( -14.5 ; 0 )` | `-15` | ✅ | ✅ | ✅ | ✅ |
| `1 OR 0 XOR 1` | `0` | ⚠️ self-contradictory (stated `1`, own working showed `0`) | ✅ | ✅ | ✅ |
| `WordCount ( "x=y=1.5" )` | `3` | ❌ `1` | ✅ | ✅ | ✅ |
| `Tan ( Radians ( 90 ) )` | large finite number (`16331239353195370`) | ❌ `≈0.894` (wrong even as ordinary trig, not just the FileMaker edge case) | ✅ | ✅ | ✅ |
| `Base64Encode ( "Black" )` | `QmxhY2s=` + trailing CRLF | ❌ `QmxhY2s=`, no trailing character at all | ❌ 0/3 | ✅ | ❌ 0/3 |
| `Char(233) = ( Char(101) & Char(769) )` | `1` | ✅ | ✅ | ✅ | ✅ |
| `Exact ( Char(233) ; Char(101) & Char(769) )` | `0` | ❌ `1` (assumed `Exact` normalizes too — the inverse of the expected mistake) | ⚠️ 2/3 | ✅ | ✅ |

**Full/partial/fail across all 21 prompts, blind:** ChatGPT (free) 9/21 correct, 1 ambiguous. Claude Sonnet 5 16 full + 3 partial + 2 fail. Claude Opus 5 19 full + 0 partial + 2 fail. Claude Fable 5.1 16 full + 3 partial + 2 fail.

The fraction in each Claude cell for the 13 re-tested prompts comes from asking that exact question again, on its own: an isolated, single-question call with no shared context between questions and no quiz framing, natural phrasing, 3 independent samples per model, with every response explicitly confirming no outside tool or documentation was used. A `✅ 3/3` means solved cleanly every time; `⚠️` means right on some samples and wrong on others of the *identical* question; `❌ 0/3` means wrong every single time, not an unlucky roll. That distinction changes the story the single-shot table alone would tell:

- **`NOT 2 ^ 0` is the one genuinely universal blind spot** — `0/9` across all three models and every sample, always the same wrong answer, from assuming `^` binds tighter than `NOT` when FileMaker does the reverse.
- **`GetAsBoolean` of an error result looked like a shared weak spot in the first blind pass, but it's worse than that for Opus specifically** — `0/3`, while Sonnet and Fable are each an inconsistent `2/3` on the identical question.
- **`Base64Encode`'s trailing character is the opposite pattern** — not universal at all: Sonnet and Fable go a combined `0/6`, but Opus is a clean `3/3`.
- **Three prompts that looked like real traps in the first, batched blind read turn out to be fully solved once asked on their own** — `"abc2"` truthiness, `OR`/`XOR` precedence, and `WordCount`'s word-boundary rule all score `9/9` across all three models. These were failures of the framing or of an older single-shot answer, not persistent gaps.

The reasoning each model gave away its own failure mode. All three assumed `^` binds tighter than `NOT`, the same precedence-from-general-math-notation assumption behind `-2^2` (which fooled ChatGPT and Claude Sonnet 5's first blind answer, though Sonnet solves it consistently once re-tested in isolation). ChatGPT and Claude Sonnet 5 both assumed text truthiness requires a *leading* digit rather than a digit found *anywhere* in the string (the real rule is `GetAsNumber` of the whole string being non-zero) — Opus and Fable got it right in the first blind pass, and the isolated re-test confirms it's fully solved once asked alone. ChatGPT's `Tan` answer wasn't even correct ordinary trigonometry, independent of the FileMaker-specific edge case. Its `Exact()` mistake was the most interesting failure of the set: it assumed the *stricter-sounding* function would be the lenient one, the opposite of the usual "assumed FileMaker behaves like a normal language" trap.

Treat each new candidate as untested until it's actually run, for any model — not as a guess extended from a pattern.

## What it found

| Area | Headline finding |
|---|---|
| Precedence | `-2 ^ 2` is `4`, not `-4` — unary minus binds tighter than `^` |
| Coercion | `GetAsBoolean` of an error result is `1` (true) |
| Case sensitivity | `=` is case-insensitive; `Substitute` is not |
| Strings | `Trim` strips spaces only, not tabs |
| Numbers | `Round` breaks ties away from zero, not toward `+∞` |
| Dates & errors | `EvaluationError` codes checked one by one against Claris's own reference |
| Base64 | `Base64Encode` silently line-wraps by default (RFC 2045) |
| Unicode | `=` normalizes NFC/NFD; `Exact`, `Position`, `Substitute` don't |

Full detail, with the Claris documentation each finding confirms or corrects, below.

### Precedence and associativity

Unary minus binds tighter than exponentiation, so `-2 ^ 2` is `4`, not `-4` — matching Excel rather than Python's convention. `NOT` behaves the same way: `NOT 2 ^ 0` is `1`, evaluated as `(NOT 2) ^ 0`. That depends on `0 ^ 0` equaling `1`, now its own tested vector rather than an assumption buried inside another one. `^` is left-associative: `2 ^ 3 ^ 2` is `64`, the opposite of Python's right-associative `**`. Comparison chains and `OR`/`XOR` are ordinary same-tier left-to-right evaluation: `1 < 2 < 3` is `1`, evaluated as `(1<2) < 3`.

Standard precedence holds everywhere else: `*` and `^` tighter than `+`, `&` looser than `+`, comparison looser than `AND`, `AND` tighter than `OR`. `//` and block comments strip correctly anywhere in an expression, including one that contains quote characters, and comment markers inside a real string literal stay literal text. Block comments nest, matching Claris's own docs.

### Coercion and truthiness

`GetAsBoolean` treats an error result as true. `GetAsBoolean("?")` returns `1`, matching Claris's documentation and contradicting most people's instinct that an error should read false. `GetAsNumber` concatenates digits rather than stopping at the first non-numeric character: `GetAsNumber("12abc34")` is `1234`. `If` and `Case` follow the identical digit-extraction rule for truthiness — a leading digit makes text truthy no matter what follows it.

### Case sensitivity

Plain `=` comparison is case-insensitive by default, and so are `Position` and `PatternCount`. `Substitute` is not: `Substitute("ABC";"abc";"xyz")` returns `ABC` unchanged, a real miss between functions that otherwise look related. Four findings in this corpus are paired against an explicit contrasting control for exactly this reason — a case-insensitive match sitting next to a case-sensitive miss on the same input, and a default precedence result next to the same expression forced the other way with explicit parentheses.

### String functions

`Trim` strips spaces only, not tabs: `Trim(Char(9)&"Tom"&Char(9))` comes back with both tabs still attached. Indexing is one-based throughout — position `0` behaves the same as position `1` — and `Left`, `Right` and `Middle` return the available text rather than erroring when asked for more than exists. `Position` returns `0`, not an error, when nothing matches. `Substitute` applies multiple pairs in sequence against the already-substituted string, not simultaneously.

### Let and Case

`Let` evaluates its bindings left to right and allows a later one to shadow an earlier one in the same call. A nested `Let` reverts its outer scope on exit, though a `$$global` set inside one persists after it. `Case` with no default returns an empty string rather than an error, and stops at the first true test even when a later one is also true.

### Numbers

FileMaker suppresses the leading zero on a fraction with no integer part: `2 ^ -2` displays as `.25`. `Round` rounds halves away from zero — `Round(14.5;0)` is `15` and `Round(-14.5;0)` is `-15`. It's the negative case that actually proves it; the positive one alone can't tell "away from zero" apart from "up." A negative precision argument rounds to tens, hundreds or thousands: `Round(29343.98;-3)` is `29000`.

`Log(0)` and `Lg(0)` both error despite their own docs describing zero as "returns nothing" — which turns out to mean no valid result, not an empty string. `Ln(0)` errors too, and its docs are the one of the three that state this plainly rather than ambiguously. `Tan(Radians(90))` does not error, despite the function's own docs saying 90 degrees can't be used, because `Radians(90)` doesn't land exactly on the asymptote in floating point.

A leading `+` is accepted as a no-op sign, scientific notation computes correctly, and a thirty-digit integer loses no precision. Rejected outright rather than silently reinterpreted:

- more than one decimal point
- a trailing decimal point with nothing after it
- an underscore used as a digit separator

### Dates, times and errors

A timestamp's serial number decomposes exactly as the date serial minus one, times 86400, plus the time serial. `GetAsDate` errors outside year 1 to 4000 rather than clamping. `GetAsTime` passes an out-of-range hour through unchanged as text, and correctly applies the am or pm designator when present — `"12:15 pm"` and `"12:15 am"` parse to internal values exactly 12 hours apart. (Correction: an earlier version of this finding claimed the designator was discarded, based on the displayed text alone rather than checking the am case directly. It wasn't, and it's fixed here rather than left standing.)

`EvaluationError` codes were checked one by one against Claris's own reference:

- unbalanced parenthesis → `1207`
- too many parameters → `1202`
- zero arguments where at least one is required → `1204`, not the more general `1201`
- two adjacent literals with no operator between them → `1212`, not `1208`

`EvaluationError` itself returns `0` incorrectly when it wraps a variable that already holds an `Evaluate()` result instead of the call itself — an easy trap to build a test harness around by accident. A missing field returns `102`; dividing by zero returns `15`.

### Lists, separators and fields

List and separator functions treat CR and LF as equivalent, and U+2028 and U+2029 both work the same way, with a bare `Char`/`Code` round trip preserving either character correctly. A field name containing a space is referenced directly with no special quoting, bracket syntax reaches a specific repetition of a repeating field, and `List()` over one returns a CR-delimited list of its values in order.

### Base64

`Base64Encode` defaults to RFC 2045: a 200-character input wraps at 76 characters per line with CRLF endings, and even `Base64Encode("Black")` gets a trailing CRLF that Claris's own doc example doesn't show. `Base64EncodeRFC` falls back to RFC 4648 (no line breaks) for any `RFCNumber` it doesn't recognize, confirmed for both `9999` and a negative number (`-1`) — the doc only says "unrecognized," not whether sign matters.

The other two documented formats hold up too: `1421` wraps at 64 characters a line, `4880` wraps at 76 and appends a base64-encoded 24-bit CRC as a trailing line (`=I5aK`-shaped), and `2045` matches plain `Base64Encode` byte-for-byte.

`Base64Decode` is the more interesting half. It round-trips cleanly through `Base64Encode`'s own CRLF line breaks and through RFC 4880's CRC suffix, and silently ignores a stray space spliced into an encoded string — none of it documented. But a non-whitespace character outside the Base64 alphabet, including the URL-safe `-`/`_` variants, fails the whole calculation with a hard engine error (FileMaker error 17), not the graceful `?` most FileMaker functions return on failure. Confirmed as real runtime behaviour, not a literal-constant fluke, using a value that can't be known until the calculation runs.

Padding has a narrower rule: a data-length remainder of 1 (mod 4) is unrecoverable no matter how much `=` you add. A remainder of 2 or 3 just needs one `=` present, even short of the textbook count — `"QQ="` works, `"QQ"` alone doesn't. Excess padding is harmless either way.

### Unicode

`=` treats a precomposed character (NFC — é as one codepoint) and its decomposed form (NFD — e plus a separate combining accent) as equal. `Exact()`, `Position()`, `PatternCount()` and `Substitute()` all disagree: none of them normalize, so all four treat the identical-looking NFC and NFD strings as different text. A search or replace built on the assumption that "if `=` says two strings match, every other text function will find one inside the other" breaks the moment the source data and the search term use different normalization forms.

`Length()` and `Position()` don't share one counting unit either. A base letter plus a combining accent counts as one character, but a single emoji outside the Basic Multilingual Plane counts as two, and `Position()` indexes using that same two-unit count. Turkish casing isn't locale-aware here either: `Upper()`/`Lower()` map Turkish dotless ı and dotted İ to plain ASCII I/i rather than the Turkish-specific pairing — see Provenance for the regional-setting caveat this depends on.

Every one of these lives in `references/corpus.jsonl` with its exact expression and measured result, and in `references/rules.jsonl` grouped into 38 named rules, each pointing back at the vectors that support it.

## What it is not

**Not a formal grammar.** No EBNF, no parser, nothing that specifies syntax ahead of testing it. That's a deliberate choice, not a gap waiting to be filled.

**Not a function reference.** This corpus covers what calling a function or operator actually does — the return value, the edge case, the coercion rule — not function names, parameter order or return types. That already exists in the companion [FileMaker AI Vocabulary](https://github.com/andykear/FileMaker-AI-vocabulary), and duplicating it here would just be two places that can drift apart. Where a finding depends on a function's documented signature, it names the function and leaves the signature to that repo.

## Provenance

Tested against FileMaker Pro 26.0.2 on macOS, day/month regional date format, one version so far. Every expected output was measured directly from that file — not transcribed from documentation, not assumed from another language's convention. Where a result confirms or corrects something Claris's own help pages state, both are cited above.

Two build details worth knowing if you extend the corpus:

- `Expression` is stored text that `Evaluate()` parses as a second, independent formula. A literal control character has to come from a live function call inside that stored text, not a raw byte — the calculation editor treats a raw byte as an insignificant line break rather than string content.
- Date and time results display according to the file's regional format, so every date or time vector compares through `GetAsNumber()` — the locale-independent serial number — rather than against a literal formatted string.

197 vectors across 26 subcategories. Not yet covered:

- Locale-dependent decimal separator parsing — needs an actual regional settings change, not a schema addition.
- The Turkish-i casing findings — measured on this file's non-Turkish regional setting only.
- Base64 container-field round-trips, `Base64Decode`'s `fileNameWithExtension` parameter, and `CryptEncryptBase64`/`CryptDecryptBase64` — all need the record-based harness rather than a bare calculation-evaluation probe.

## Using it

Load the release zip into Claude's skills, keeping the folder structure so `references/` comes with it, and enable code execution and file creation. With the skill loaded, the seven highest-confidence traps get checked automatically while a calculation is written or reviewed — no special prompt needed.

```
"Does FileMaker's Trim() strip tabs as well as spaces?"    → references/rules.jsonl: trim_only_strips_spaces
"What does -2^2 evaluate to in FileMaker?"                  → references/rules.jsonl: unary_minus_binds_tighter_than_exponent
"Is Substitute case-sensitive?"                              → references/rules.jsonl: substitute_case_sensitive_position_not
```

For anything beyond those seven, `references/rules.jsonl` and the findings above cover the rest. `references/corpus.jsonl` is the ground truth behind all of it, if you want the exact expression and result for a specific claim, or want to extend the corpus with a new vector of the same shape.

## The rest of the collection

**[Menu](https://github.com/andykear)**

**Reference skills**

**[FileMaker Second Opinion](https://github.com/andykear/FileMaker-second-opinion)**\
**[FileMaker AI Vocabulary](https://github.com/andykear/FileMaker-AI-vocabulary)**\

**Research / Specialist**

**[FileMaker AI Grammar](https://github.com/andykear/FileMaker-AI-grammar)**

**Generation, paste-ready FileMaker XML**

**[Script XML Skill](https://github.com/andykear/FileMaker-XMLsnippet-Claude-Skill)** (XMSS, XMSC, XMFN)\
**[Layout XML Skill](https://github.com/andykear/FileMaker-XMLsnippet-Layout-Claude-Skill)** (XML2)\
**[Field, Table & Value List Definitions](https://github.com/andykear/FileMaker-XML-field-definitions)** (XMFD, XMTB, XMVL)

**Analyse a FileMaker solution in your browser**

**[Clockwork Inspector](https://github.com/andykear/FileMaker-XML-inspector-open-source)** (SaXML)\
**[XML Scrubber](https://github.com/andykear/FileMaker-XML-scrubber)** (SaXML + others)

## Licence

CC BY 4.0. Use it, adapt it, build on it, keep the attribution. Missing edge case or a wrong result? Open an issue or a pull request.

**Andrew Kear** · Claris Partner · Claris MVP · [clockworkct.co.uk](https://clockworkct.co.uk)

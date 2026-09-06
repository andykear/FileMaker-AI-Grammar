[![Stars](https://img.shields.io/github/stars/andykear/FileMaker-AI-Grammar?style=social)](https://github.com/andykear/FileMaker-AI-Grammar)
[![License](https://img.shields.io/badge/license-CC%20BY%204.0-green)](https://creativecommons.org/licenses/by/4.0/)

# FileMaker AI Grammar

171 test vectors covering FileMaker's calculation engine, each one measured against real FileMaker Pro rather than assumed. 29 rules pulled out of that corpus and a Claude skill built from the seven that matter most.

Developed by Andrew Kear of Clockwork Creative Technology and shared openly with the FileMaker/Claris community.

## The problem

FileMaker's calculation language has no published formal grammar. Claris maintains a full operator order of evaluation table, at `operators-in-formulas.html`, it just isn't cross linked from the four separate operator category pages that describe the individual operators.

So this repo tests instead of describing. Every vector here is a calculation expression, round trip tested against a real FileMaker Pro file, with the value it actually returned logged as the expected output.

The reason that matters: FileMaker's calculation engine has real, specific behavior that doesn't match general purpose language convention, or plain intuition. Unary minus binds tighter than exponentiation. `Trim` strips only spaces. `Substitute` is case sensitive while `Position` isn't. Some of this is stated in Claris's own help pages, it's just easy to miss, or to contradict with intuition carried over from another language. A model reasoning from general programming convention gets a predictable slice of this confidently wrong, with nothing to signal that anything is off.

### Tested against ChatGPT

All eight predicted traps below were run against ChatGPT, logged honestly rather than cherry picked. Four produced a confirmed wrong answer. Four the model already had right.

| Prompt | Predicted wrong answer | Actual | Result |
|---|---|---|---|
| `"abc" = "ABC"` | False | `1` (true) | confirmed trap |
| `-2 ^ 2` | `-4` | `4` | confirmed trap |
| `2 ^ 3 ^ 2` | `512` | `64` | already correct |
| `Trim ( Char(9) & "Tom" & Char(9) )` | `"Tom"` | tabs survive | confirmed trap |
| `GetAsNumber ( "12abc34" )` | `12` | `1234` | already correct |
| `Substitute ( "ABC" ; "abc" ; "xyz" )` | `xyz` | `ABC` (no match) | already correct |
| `GetAsBoolean` of an error result | False | `1` (true) | confirmed trap |
| `Case ( 0 ; "a" )`, no default | error or null | `""`, no error | already correct |

No clean pattern in which ones land. Case insensitive `=` is arguably the single most well known FileMaker fact there is, and it still tripped the model up, while more obscure ones didn't. Treat each new candidate as untested until it's actually run, not as a guess extended from a pattern.

## What it found

Precedence and associativity are where the general purpose instincts fail hardest. Unary minus binds tighter than exponentiation, so `-2 ^ 2` is `4`, not `-4`, matching Excel's identical documented behavior rather than Python's convention. `NOT` behaves the same way: `NOT 2 ^ 0` is `1`, evaluated as `(NOT 2) ^ 0`, and that depends on `0 ^ 0` equaling `1`, which is now its own tested vector rather than an assumption buried in another one. `^` itself is left associative, so `2 ^ 3 ^ 2` is `64`, the opposite of Python's right associative `**`. Comparison chains and `OR`/`XOR` are ordinary same tier left to right evaluation rather than anything special cased: `1 < 2 < 3` is `1`, evaluated as `(1<2) < 3`. Standard precedence holds everywhere else, `*` and `^` tighter than `+`, `&` looser than `+`, comparison looser than `AND`, `AND` tighter than `OR`, and `//` and block comments strip correctly at the start, middle or end of an expression, including one that contains quote characters, while comment markers sitting inside a real string literal stay literal text. Block comments nest, matching what Claris's own docs claim they do.

Coercion and truthiness follow one rule more consistently than most FileMaker developers expect. `GetAsBoolean` treats an error result as true, `GetAsBoolean("?")` returns `1`, which matches Claris's own documentation and contradicts most people's instinct that an error should read as false. `GetAsNumber` concatenates digits rather than stopping at the first non numeric character, so `GetAsNumber("12abc34")` is `1234`. `If` and `Case` follow the identical digit extraction rule for truthiness: text with a leading digit is truthy no matter what follows it, text without one is false.

Case sensitivity is inconsistent within FileMaker's own function family, and that inconsistency is real, not a testing artifact. Plain `=` comparison is case insensitive by default. `Position` and `PatternCount` are case insensitive too. `Substitute` is not: `Substitute("ABC";"abc";"xyz")` returns `ABC` unchanged, a genuine miss between functions that otherwise look related. Four of the findings in this corpus are paired against an explicit contrasting control for exactly this reason, so a case insensitive match sits right next to a case sensitive miss on the same kind of input, and a default precedence result sits next to the same expression forced the other way with explicit parentheses.

String handling has its own quiet traps. `Trim` strips spaces only, not tabs, so `Trim(Char(9)&"Tom"&Char(9))` comes back with both tabs still attached. Indexing is one based throughout, with position `0` behaving the same as position `1`, and `Left`, `Right` and `Middle` returning the available text rather than erroring when asked for more characters than exist. `Position` returns `0`, not an error, when the search text or the requested occurrence isn't found. `Substitute` applies multiple pairs in sequence against the already substituted string rather than simultaneously.

`Let` evaluates its bindings left to right and allows a later one to shadow an earlier one in the same call. A nested `Let` reverts its outer scope on exit, though a `$$global` variable set inside one persists after it. `Case` with no default result returns an empty string rather than an error, and stops at the first true test even when a later one is also true.

Numbers carry their own small surprises. FileMaker suppresses the leading zero on a fraction with no integer part, so `2 ^ -2` displays as `.25`. `Round` always rounds a half up, `Round(14.5;0)` is `15`, and on the negative side it rounds away from zero rather than toward positive infinity, `Round(-14.5;0)` is `-15`. The positive case alone can't tell those two conventions apart, since they only diverge below zero. A negative precision argument rounds to tens, hundreds or thousands, `Round(29343.98;-3)` is `29000`. `Log(0)` and `Lg(0)` both return an error despite their own docs describing zero as returning nothing, which turns out to mean no valid result rather than an empty string. `Ln(0)` also errors, and its docs are the one of the three that state this plainly rather than ambiguously. `Tan(Radians(90))` does not error, despite the function's own docs saying 90 degrees can't be used, because `Radians(90)` doesn't land exactly on the mathematical asymptote in floating point.

Numeric literals accept a leading `+` as a no op sign and scientific notation computed correctly, and hold a thirty digit integer with no precision loss. A literal with more than one decimal point, a trailing decimal point with nothing after it, or an underscore used as a digit separator are all rejected outright rather than silently reinterpreted.

Dates, times and errors were measured with the same discipline. A timestamp's serial number decomposes exactly as the date serial minus one, times 86400, plus the time serial. `GetAsDate` returns an error outside year 1 to 4000 rather than clamping. `GetAsTime` passes an out of range hour through unchanged as text and discards the am or pm designator entirely, though that was measured on this file's 24 hour regional format, so it's flagged as possibly locale dependent rather than confirmed across the engine. `EvaluationError` codes were checked one by one against Claris's own error code reference: unbalanced parenthesis is `1207`, too many parameters is `1202`, zero arguments where at least one is required is `1204` rather than the more general `1201`, and two adjacent literals with no operator between them is the more specific `1212` rather than `1208`. `EvaluationError` itself returns `0` incorrectly when it wraps a variable that already holds an `Evaluate()` result instead of wrapping the call itself, worth knowing since it's an easy trap to build a test harness around by accident. A missing field returns `102`, dividing by zero returns `15`. List and separator functions treat CR and LF as equivalent, and U+2028 and U+2029 both work as real separators the same way, with a bare `Char` and `Code` round trip preserving either character correctly. A field name containing a space is referenced directly with no special quoting, bracket syntax reaches a specific repetition of a repeating field, and `List()` over one returns a CR delimited list of its values in order.

Every one of these lives in `references/corpus.jsonl` with its exact expression and measured result, and in `references/rules.jsonl` grouped into 29 named rules, each pointing back at the vectors that support it.

## What it is not

Not a formal grammar. No EBNF, no parser, nothing that specifies syntax ahead of testing it. That's a deliberate choice, described above, not a gap waiting to be filled.

Not a function reference either. This corpus handles what calling a function or operator actually does, the return value, the edge case, the coercion rule. It doesn't carry function names, parameter order or return types, that already exists in the companion [FileMaker AI Vocabulary](https://github.com/andykear/FileMaker-AI-vocabulary), and duplicating it here would just be two places that can drift apart. Where a finding depends on a function's documented signature, it names the function and leaves the signature to that repo.

## Provenance

Tested against FileMaker Pro 26.0.2 on macOS, day and month regional date format, one version so far. Every expected output was measured directly from that file, not transcribed from documentation and not assumed from another language's convention. Where a result confirms or corrects something Claris's own help pages state, both are cited above.

Two things about how the vectors themselves were built are worth knowing if you extend the corpus. `Expression` is stored text that `Evaluate()` parses as a second, independent formula, so a literal control character has to come from a live function call inside that stored text rather than a raw byte, which the calculation editor treats as an insignificant line break instead of string content. And date and time results display according to the file's regional format, so every date or time vector compares through `GetAsNumber()`, the locale independent serial number, rather than against a literal formatted string.

171 vectors across 24 subcategories. Not yet covered: locale dependent decimal separator parsing, which needs an actual regional settings change to test rather than a schema addition, so it stays an open gap rather than a guess.

## Using it

Load the release zip into Claude's skills, keeping the folder structure so `references/` comes with it, and enable code execution and file creation. With the skill loaded, the seven highest confidence traps get checked automatically while a calculation is written or reviewed, no special prompt needed.

```
"Does FileMaker's Trim() strip tabs as well as spaces?"    → references/rules.jsonl: trim_only_strips_spaces
"What does -2^2 evaluate to in FileMaker?"                  → references/rules.jsonl: unary_minus_binds_tighter_than_exponent
"Is Substitute case-sensitive?"                              → references/rules.jsonl: substitute_case_sensitive_position_not
```

For anything beyond those seven, `references/rules.jsonl` and the findings above cover the rest, and `references/corpus.jsonl` is the ground truth behind all of it if you want the exact expression and result for a specific claim, or want to extend the corpus with a new vector of the same shape.

## The rest of the collection

**[AI Vocabulary](https://github.com/andykear/FileMaker-AI-vocabulary)**
Verified names, signatures and return types for every FileMaker function and script step, so an AI stops inventing them. This corpus is its behavioral companion: the vocabulary says what's callable, this says what calling it actually does.

**[Second Opinion](https://github.com/andykear/FileMaker-second-opinion)**
A reasoning skill that corrects an AI's tendency to mistake FileMaker's common solution for the one that holds up in production.

**[Script XML Skill](https://github.com/andykear/FileMaker-XMLsnippet-Claude-Skill)**
Makes AI generated scripts paste correctly. Full step ID dictionary and the hidden paste handler rules.

**[Layout XML Skill](https://github.com/andykear/FileMaker-XMLsnippet-Layout-Claude-Skill)**
Paste ready layout objects. All object types, flags decoded, element order confirmed.

**[Field Definitions](https://github.com/andykear/FileMaker-XML-field-definitions)**
Field and table definition XML, verified down to auto enter, validation and calculation options.

**[XML Inspector](https://github.com/andykear/FileMaker-XML-inspector-open-source)**
Reads a Save as XML export in the browser. Finds unreferenced fields, broken references, diffs two versions.

**[XML Scrubber](https://github.com/andykear/FileMaker-XML-scrubber)**
Strips API keys, passwords and internal hostnames from FileMaker XML before you share it with an AI tool.

## Licence

CC BY 4.0. Use it, adapt it, build on it, keep the attribution. Missing edge case or a wrong result? Open an issue or a pull request.

**Andrew Kear** · Claris Partner · Claris MVP · [clockworkct.co.uk](https://clockworkct.co.uk)

# Regular Expressions and Unicode

Regular expressions are where Unicode assumptions become visible. A regex of `[a-z]` looks innocent until a user with an accented name shows up. This chapter walks through what your regex engines actually do with Unicode text, and how to tell them what you mean.

## Why `[a-z]` doesn't match `á`

A *character class* like `[a-z]` is a range over code points (or code units, depending on the engine). The range `[a-z]` is the code points U+0061 through U+007A. That's it. Those are the lowercase ASCII letters; it says nothing about `á` (U+00E1), `č` (U+010D), or `ω` (U+03C9).

If you write `^[a-zA-Z]+$` as your "name must be alphabetic" check, you are saying "only ASCII English letters are acceptable." That is a specific business decision, not a neutral default. If you didn't mean to make it, you have a Unicode bug.

The fix depends on what you actually meant. If you meant "any letter," the fix is Unicode character properties.

## Unicode character properties

Every code point has a set of *properties* assigned by the Unicode Standard. The most important for regex work are:

- *General Category* — a two-letter classification. `Ll` (lowercase letter), `Lu` (uppercase letter), `Lo` (other letter — used for scripts without case, like Chinese), `Nd` (decimal digit), `Pc` (connector punctuation), etc. The one-letter prefixes group them: `L` is all letters, `N` is all numbers, `P` is all punctuation, `Z` is all separators, `C` is all control/unassigned/private-use.
- *Script* — which writing system the code point belongs to. `Latin`, `Cyrillic`, `Greek`, `Han`, `Arabic`, `Devanagari`, etc.
- *Block* — where in the Unicode code point space the character sits. Rarely what you want (see below).
- *Derived properties* like `Alphabetic`, `White_Space`, `Emoji`.

In Unicode-aware regex, you access properties with `\p{…}`:

- `\p{L}` matches any letter (any `L*` category).
- `\p{Ll}` matches any lowercase letter.
- `\p{N}` matches any numeric character.
- `\p{Script=Latin}` or `\p{Latin}` matches any code point in the Latin script.
- `\p{Emoji}` matches any emoji (derived property).
- `\P{…}` is the negation.

So the "any letter" regex is `\p{L}`, not `[a-zA-Z]`.

## Block vs. Script

A common mistake is to write `\p{InGreek}` (the *block*) when you mean `\p{Script=Greek}`. Blocks are ranges of code points and have no semantic meaning beyond their position: the "Greek" block contains Greek letters *and* some unassigned slots *and* some specifically-coptic letters, and does not contain the Greek Extended block. The *Script* property is what you almost always want.

Worse, in some engines `[\u0370-\u03FF]` is the Greek block's byte range and looks correct, but it misses the Greek Extended block (U+1F00–U+1FFF, where polytonic Greek lives). `\p{Script=Greek}` includes both.

## Unicode-aware vs. code-unit-aware regex

Most regex engines have a flag or mode that controls how they interpret the pattern against the input.

### JavaScript

Two relevant flags: `u` (Unicode) and `v` (Unicode sets, ES2024).

```javascript
"😀".match(/./);       // matches U+D83D (the high surrogate); .length === 1
"😀".match(/./u);      // matches U+1F600 (the whole code point)
"😀".match(/\p{Emoji}/u);  // works with /u
```

Without the `u` flag, JavaScript regexes operate on UTF-16 code units. `.` matches one code unit. `[\u{1F600}]` is a syntax error. `\p{…}` is not recognized.

With the `u` flag: `.` matches one code point. `[\u{1F600}]` is allowed. `\p{…}` is enabled.

With the `v` flag (superset of `u`): string character classes, set operations (intersection, subtraction), and more powerful property escapes. `[\p{Script=Greek}--\p{Letter}]` (code points in the Greek script that aren't letters).

### Python

- The `re` module is Unicode-aware by default in Python 3. `\w`, `\d`, `\s` match all Unicode word characters, digits, and whitespace. Use the `re.ASCII` flag to revert to ASCII-only.
- `re` does *not* support `\p{…}` or grapheme clusters. For those, use the third-party `regex` module.
- `regex` (not `re`) supports `\p{L}`, `\X` for grapheme clusters, `\N{name}` for code points by name, possessive quantifiers, and more.

```python
import regex
regex.findall(r"\X", "Hi 👋🏽")     # ['H', 'i', ' ', '👋🏽']
regex.findall(r"\p{L}", "café 1")  # ['c', 'a', 'f', 'é']
```

### Java

`java.util.regex.Pattern` supports `\p{L}`, scripts as `\p{IsGreek}`, and category escapes like `\p{Ll}`. To enable Unicode-aware `\w`, `\d`, `\s`, use the `Pattern.UNICODE_CHARACTER_CLASS` flag or the `(?U)` embedded flag.

Java does *not* have a grapheme cluster regex construct. Use `java.text.BreakIterator.getCharacterInstance()`.

### Go

`regexp` has limited Unicode property support: `\p{L}`, `\p{Greek}`, etc. work. The engine (RE2) is designed for guaranteed linear-time matching and intentionally excludes some features like backreferences. No grapheme cluster escape; reach for `rivo/uniseg`.

### Rust

The `regex` crate supports Unicode character classes, `\p{L}`, `\p{Script=…}`, case-insensitive Unicode matching. No grapheme cluster escape; `unicode-segmentation` crate.

### PCRE / PCRE2 / Perl-style

Under the `u` modifier (or `UTF8` / `UCP` options), PCRE treats the input as UTF-8, `.` matches one code point, `\p{L}` is recognized, `\X` matches a grapheme cluster.

## Case-insensitive matching under Unicode

A naive `CaseInsensitive` regex flag should match `k` to `K`. Under Unicode, it should also match:

- `k` to `K` to `K` (U+212A, KELVIN SIGN, which is a Latin K in compatibility form).
- `ß` to `SS` — the German eszett's uppercase is two letters, so `ß/i` should match `ss`.
- `i` to `I` to `İ` (Turkish dotted capital I), depending on locale.

Whether your engine does any of this varies. `java.util.regex.Pattern.CASE_INSENSITIVE` on its own is ASCII-only; pass `UNICODE_CASE` also. JavaScript's `i` flag under `u` handles simple folds (`ſ` → `s`) but not `ß` → `ss`. Python's `re` with `re.IGNORECASE` folds under Unicode, but `ß` to `ss` is *not* handled (the engine uses simple case folding).

If you need robust case-insensitive matching, case-fold both the pattern and the input up front, then match with a case-sensitive regex. This works around most of the engine-specific gaps.

## Grapheme-aware regex

Some engines have a `\X` escape meaning "one grapheme cluster."

- `regex` (Python third-party): yes.
- PCRE: yes.
- Perl: yes.
- Go: no.
- Java's `java.util.regex`: no.
- Rust's `regex` crate: no.
- JavaScript: no.

If you need grapheme-aware matching in a language without `\X`, segment the string into grapheme clusters first (Chapter 4) and then apply per-cluster logic.

## Practical recipes

### Validating usernames

"Letters, digits, underscores, 3–20 characters."

```javascript
// Unicode-letter aware:
const valid = /^[\p{L}\p{N}_]{3,20}$/u;
valid.test("user_01");      // true
valid.test("用户_1");         // true
valid.test("user name");    // false (space)
```

Remember to also apply a *confusables* check (Chapter 10) if this username is user-visible. Allowing any letter means allowing mixed-script usernames, which is a vector for homograph attacks.

### Stripping accents

```python
import unicodedata as ud
s = "Café résumé naïve"
decomposed = ud.normalize("NFD", s)
stripped = "".join(ch for ch in decomposed if not ud.combining(ch))
# 'Cafe resume naive'
```

Regex isn't the right tool here; normalization is.

### Counting words

```python
import regex
text = "Rouge et noir — 三国演义"
regex.findall(r"\p{L}+", text)
# ['Rouge', 'et', 'noir', '三国演义']
```

`\p{L}+` is the closest you can get to a language-neutral word boundary. For real word segmentation — which matters in Chinese, Japanese, and Thai, where there are no spaces between words — you need `java.text.BreakIterator.getWordInstance()` or ICU's equivalent.

## The regex tax

Regular expressions are a terrific pattern-matching tool. They are also one of the places where "it worked on my English test input" most reliably fails in production. Before you ship a regex, always ask:

- Is `.` matching a byte, a code unit, or a code point?
- Are `[a-z]`, `[A-Z]`, `\w`, `\d`, `\s` doing what I want for non-English input?
- Does case-insensitive matching handle the fold pairs I care about?
- Am I matching user-visible characters (grapheme clusters) or code points?

If any of those answers is "I don't know," find out before the regex goes near live data.

Next we look at input and output: what actually happens when your bytes move between programs.

# The Working Programmer's Cheat Sheet, Per Language

Every language picked a model for how its string type works, and those models are not the same. When you move between languages, you are moving between models, and the bug that bites you is usually at that seam. This chapter goes through the main languages you are likely to use, and for each one tells you: what a string *is*, what `length` counts, and where to reach when the built-ins aren't enough.

We will use a single reference string throughout: `"Hi 👋🏽"` — `H`, `i`, space, and a *waving hand with medium skin tone*. That last grapheme cluster is two code points: U+1F44B (waving hand) + U+1F3FD (Fitzpatrick modifier type 4).

- Grapheme clusters: 4.
- Code points: 5.
- UTF-16 code units: 7 (the two supplementary code points are surrogate pairs).
- UTF-8 bytes: 11 (H=1, i=1, space=1, waving hand=4, modifier=4).

## Python 3

*String type:* `str`. A sequence of Unicode code points. Since Python 3.3 (PEP 393), CPython uses a flexible internal representation — 1, 2, or 4 bytes per code unit depending on the largest code point in the string — but the semantics are uniform: indexing and `len()` count code points.

*Bytes type:* `bytes`. A sequence of bytes, unrelated to text semantically. `str.encode(...)` goes from `str` to `bytes`; `bytes.decode(...)` goes the other way. You must always specify an encoding at the boundary.

```python
>>> s = "Hi 👋🏽"
>>> len(s)                 # code points
5
>>> s[3]                    # one code point — the waving hand, without skin tone
'👋'
>>> len(s.encode("utf-8"))
11
```

*What `len()` counts:* code points. Not bytes, not grapheme clusters.

*Where the built-ins are enough:*

- `str.encode` / `bytes.decode` for encoding conversion.
- `unicodedata` stdlib module: normalization, category lookup, combining class.
- `casefold()` for case-insensitive comparison.

*Where you need a third-party library:*

- Grapheme clustering: the `regex` module (not `re`) supports `\X`, or use the `grapheme` package.
- Full Unicode collation: PyICU (`icu.Collator`) or `pyuca`.
- Locale-aware operations: PyICU.

*Gotchas:*

- `sys.getfilesystemencoding()` governs filenames on disk. Linux is `"utf-8"` on all modern systems; macOS is `"utf-8"`; Windows is `"utf-8"` as of Python 3.6+ on Windows 10+, was `"mbcs"` historically.
- Python 3 introduced the `surrogateescape` error handler specifically so that filesystem paths containing bytes that aren't valid UTF-8 can round-trip.

## JavaScript

*String type:* a sequence of 16-bit UTF-16 code units. This is not exactly the same as a sequence of code points: supplementary code points appear as pairs of code units. Strings can contain *lone surrogates*, which are code unit values that don't pair properly — these are not valid Unicode, but they are valid JavaScript strings.

*Bytes type:* `Uint8Array` or `ArrayBuffer`. You go between a string and bytes via `TextEncoder` / `TextDecoder`.

```javascript
const s = "Hi 👋🏽";
s.length;                   // 7 — UTF-16 code units
[...s].length;              // 4 — iterator splits by code point, but here counts 5? 
// actually:
[...s].length;              // 5 — iterator yields code points; our string has 5 of them
```

Wait — the string iterator yields code points, not grapheme clusters. So `[...s].length` gives 5 for our example, not 4. To get 4, use `Intl.Segmenter`:

```javascript
const seg = new Intl.Segmenter();
[...seg.segment(s)].length;   // 4 — grapheme clusters
```

*What `.length` counts:* UTF-16 code units. This is the infamous gotcha: `"😀".length` is 2, not 1.

*Where the built-ins are enough:*

- `String.prototype[Symbol.iterator]` for code-point iteration (`[...s]`, `for..of`).
- `String.prototype.codePointAt(index)` for reading a full code point (returns the whole supplementary code point even when `index` lands on a high surrogate).
- `String.fromCodePoint(cp)` for constructing.
- `TextEncoder` / `TextDecoder` for encoding to/from UTF-8 (WHATWG standard).
- `Intl.Collator` for locale-aware comparison and sorting.
- `Intl.Segmenter` for grapheme, word, and sentence segmentation. (ECMA-402 2022; shipping in all modern browsers and Node ≥ 16.)
- `String.prototype.normalize("NFC" | "NFD" | "NFKC" | "NFKD")` for normalization.

*Gotchas:*

- `"😀".length === 2`. If a user sees one character, your tweet-length counter shouldn't say two.
- `"😀"[0]` is the unpaired high surrogate, not the emoji. Indexing by UTF-16 is almost never what you want for display.
- Regular expressions with the `u` flag iterate code points; without it, they iterate code units and `[\uD83D\uDE00]` is a character class of two code units, not the emoji.
- JSON: `JSON.stringify("\uD83D")` returns `'"\\ud83d"'`, a lone surrogate embedded as an escape. Strict JSON parsers may reject this.

## Java

*String type:* `java.lang.String`, backed by a `char[]` where each `char` is a 16-bit UTF-16 code unit. Since Java 9 (JEP 254), the JVM may store ASCII-only strings in a `byte[]` internally (compact strings), but the API semantics are unchanged.

*Bytes type:* `byte[]`. You go via `String::getBytes(Charset)` and `new String(byte[], Charset)`.

```java
String s = "Hi 👋🏽";
s.length();                  // 7 — UTF-16 code units
s.codePointCount(0, s.length()); // 5 — code points
s.getBytes("UTF-8").length;  // 11 — bytes in UTF-8
```

*What `length()` counts:* UTF-16 code units.

*Where the built-ins are enough:*

- `String::codePoints()` (since Java 8) gives an `IntStream` of code points.
- `String::getBytes(Charset)` for encoding to bytes. Always pass a `Charset`; the no-arg version uses the platform default, which is a latent bug.
- `java.text.Normalizer` for NFC/NFD/NFKC/NFKD.
- `java.text.Collator` for locale-aware comparison.
- `java.text.BreakIterator.getCharacterInstance()` for grapheme clustering.

*Gotchas:*

- `s.charAt(i)` returns a `char`, which is a UTF-16 code unit — not a code point, not even a full Unicode character in the surrogate case. Use `s.codePointAt(i)` if you need a code point.
- `String::equalsIgnoreCase` uses the Unicode "simple" folding rules, which is better than `toLowerCase().equals(...)` but still not the same as a full Unicode case-insensitive collator.
- `Charset.defaultCharset()` is UTF-8 from Java 18 onward (JEP 400); before that it depended on the platform.

## Go

*String type:* `string`, which is an immutable sequence of bytes. Go strings are *conventionally* UTF-8 — the compiler encodes string literals as UTF-8, and the standard library assumes UTF-8 almost everywhere — but a `string` can legally contain any bytes.

*Rune type:* `rune` is an alias for `int32` and is used to represent a single code point. `for _, r := range s` iterates over code points (runes); invalid UTF-8 bytes yield `utf8.RuneError` (U+FFFD).

*Bytes:* `[]byte`. Convertible to/from `string` directly.

```go
s := "Hi 👋🏽"
len(s)                           // 11 — bytes
utf8.RuneCountInString(s)        // 5 — code points

for i, r := range s {
    fmt.Printf("%d %U %q\n", i, r, r)
}
// 0 U+0048 'H'
// 1 U+0069 'i'
// 2 U+0020 ' '
// 3 U+1F44B '👋'
// 7 U+1F3FD '🏽'
```

*What `len()` counts:* bytes.

*Where the built-ins are enough:*

- `range s` for code-point iteration.
- `unicode/utf8` for counting and validation.
- `strings.ToValidUTF8` for sanitizing.

*Where you need `golang.org/x/text`:*

- Normalization: `golang.org/x/text/unicode/norm`.
- Collation: `golang.org/x/text/collate`.
- Grapheme clustering: `rivo/uniseg` or `golang.org/x/text/unicode/norm` (for some use cases).
- Case folding: `golang.org/x/text/cases`.

*Gotchas:*

- `s[i]` gives a byte, not a rune. Indexing into a UTF-8 string by byte is almost never what you want.
- `len(s)` gives bytes.
- The gap in the iteration above (index 3 to index 7) reflects the 4-byte UTF-8 encoding of U+1F44B.

## Rust

*String type:* `String` (owned, growable) and `&str` (borrowed slice). Both are guaranteed valid UTF-8. Invalid UTF-8 cannot exist in a `&str` — if you need possibly-invalid bytes, use `[u8]` or `Vec<u8>`.

*Char type:* `char` is a 32-bit Unicode scalar value (a code point, excluding surrogates). `s.chars()` iterates over them.

*Bytes:* `&[u8]` / `Vec<u8>`. `as_bytes()` on `&str` is free (it is literally a `&[u8]`).

```rust
let s = "Hi 👋🏽";
s.len();                    // 11 — bytes
s.chars().count();          // 5 — code points

for c in s.chars() {
    println!("{:?}", c);
}
```

*What `len()` counts:* bytes.

*Where the built-ins are enough:*

- `chars()` for code-point iteration.
- `char::from_u32`, `u32::from(char)` for conversions.
- `to_lowercase()`, `to_uppercase()` — return iterators because one code point can map to many (e.g., `ß` → `ss`).

*Where you need crates:*

- Normalization: `unicode-normalization`.
- Grapheme clustering: `unicode-segmentation`.
- Full ICU behavior: the `icu` crate (ICU4X).
- Case folding: `caseless` or `unicase`.

*Gotchas:*

- `&s[0..1]` slices by byte index and *panics* if the boundary falls in the middle of a code point. This is safer than silent corruption, but it surprises newcomers.
- Rust cannot build a `char` from a surrogate code point. This is by design.

## Swift

*String type:* `String`, whose primary view is over *grapheme clusters* (Swift calls them *Characters* — note the capital letter and the distinction from C's `char`).

*Bytes / code units:* `s.utf8`, `s.utf16`, `s.unicodeScalars` give views over bytes, UTF-16 code units, and code points respectively.

```swift
let s = "Hi 👋🏽"
s.count                      // 4 — grapheme clusters
s.unicodeScalars.count       // 5 — code points
s.utf16.count                // 7 — UTF-16 code units
s.utf8.count                 // 11 — UTF-8 bytes
```

*What `.count` counts:* grapheme clusters. Swift is the one mainstream language whose default "character count" matches what users expect.

The tradeoff: `s.count` is *O(n)* — it has to walk the string and apply grapheme-break rules. Byte-length operations in Swift require you to explicitly ask for the `utf8` or `utf16` view, which is *O(1)*.

*Gotchas:*

- Random access by integer index is not directly supported. Use `String.Index`, which `String` provides methods for computing.
- Different views over the same string are not interchangeable.

## C and C++

This is the hardest section to write because "C" and "C++" span decades of assumptions and platforms.

### C: `char *` and `wchar_t *`

A C `char *` is a pointer to bytes. Whether those bytes represent ASCII, Latin-1, UTF-8, or something else depends on your program's conventions and the current locale. The C standard library's string functions (`strlen`, `strcmp`, `strchr`) work byte-by-byte, which is correct for UTF-8 *as byte operations* but doesn't give you any Unicode-aware semantics.

`wchar_t` is an implementation-defined wide-character type. Its width is:

- 16 bits on Windows (because Windows committed to UCS-2/UTF-16 early).
- 32 bits on Linux, macOS, and most other Unixes.

This means *C code using `wchar_t` is not portable* in the way you might assume: a `wchar_t *` means UTF-16 code units on Windows and UTF-32 code units on Linux.

### C11: `char16_t` and `char32_t`

C11 added `char16_t` and `char32_t` types, intended for UTF-16 and UTF-32 respectively, to get past the `wchar_t` portability mess. They are rarely used in practice; most C code that cares about Unicode has by now migrated to UTF-8 in `char *` and uses ICU for the hard operations.

### The pragmatic recipe

For new C/C++ code that needs to deal with text:

1. *Internal encoding:* UTF-8 in `char *` (or `std::string`, `std::u8string` in C++20).
2. *Windows interop:* convert to UTF-16 at the system call boundary. There is no shortcut; Windows's Unicode APIs take UTF-16.
3. *Heavy lifting:* ICU. Not C++'s `<locale>`, which is a quiet disaster.
4. *String length:* either `strlen` (bytes) or an explicit helper that counts code points / grapheme clusters depending on your need. Never trust that "character" is well-defined in a C context without looking up how it was counted.

### C++20 and beyond

C++20 has `std::u8string` (a `std::basic_string<char8_t>`) which is a UTF-8 string at the type level, and `std::format` has some Unicode awareness. These are improvements, but for anything real, still reach for ICU or `{fmt}`.

## A compact matrix

|Language | `length` counts | String type | Grapheme support |
|---|---|---|---|
|Python 3 | code points | `str` (code points) | third-party (`regex`, `grapheme`) |
|JavaScript | UTF-16 code units | UTF-16 | built-in (`Intl.Segmenter`) |
|Java | UTF-16 code units | UTF-16 | built-in (`BreakIterator`) |
|Go | bytes | UTF-8 bytes | third-party (`rivo/uniseg`) |
|Rust | bytes | UTF-8 bytes | crate (`unicode-segmentation`) |
|Swift | **grapheme clusters** | grapheme clusters | built-in, default |
|C `wchar_t` | code units (platform-dependent width) | pointer to wide chars | ICU |

The pattern: modern languages have converged on either UTF-8 (Go, Rust, C++20 `u8string`) or UTF-16 (Java, JavaScript, C# — mostly UCS-2 inheritance) as the internal representation. Python 3 went its own way (code-point-indexed, variable internal width). Swift went its own way on the other axis (grapheme-cluster-indexed). Both approaches are legitimate; each has its cost model.

Whatever language you are in, start by asking: *what unit am I currently looking at?* If you can answer that accurately, the rest follows.

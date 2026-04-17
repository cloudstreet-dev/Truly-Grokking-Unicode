# The Unicode Database

Everything we've discussed — normalization, case folding, collation, scripts, emoji properties, grapheme break rules — is ultimately table-driven. The tables are *the Unicode Database* (UCD), a bundle of text files published by the Unicode Consortium with every release of the standard. This chapter shows you what's in there, how to read it, and how to query it from code.

Knowing the UCD matters for two reasons. First, when your language's standard library lets you down, you can fall back to the raw data. Second, when someone asks a weird question — "what is the name of U+1F9E6?" — the UCD gives the definitive answer.

## What ships in the UCD

The Unicode Database is a directory of files, downloadable from [https://www.unicode.org/Public/](https://www.unicode.org/Public/) under the current version directory (e.g., `16.0.0/ucd/`). The core files:

- *UnicodeData.txt* — the most important file. One line per assigned code point with canonical properties.
- *PropList.txt* — additional simple properties (Alphabetic, White_Space, Bidi_Control, etc.).
- *DerivedCoreProperties.txt* — derived properties, including ID_Start, ID_Continue, and many more.
- *Scripts.txt* — the Script property for every code point.
- *Blocks.txt* — which block each code point belongs to.
- *CaseFolding.txt* — case-folding mappings.
- *SpecialCasing.txt* — language-specific casing rules (Turkish I, etc.).
- *CompositionExclusions.txt* — code points that cannot be composed even if they look decomposable.
- *NormalizationTest.txt* — test cases for normalization implementations.
- *GraphemeBreakProperty.txt* — the Grapheme_Cluster_Break property.
- *emoji/emoji-data.txt* — the emoji properties (Emoji, Emoji_Presentation, etc.).
- *confusables.txt* — the confusable characters table (part of UTS #39).

All files are ASCII text, semicolon-separated, with `#` introducing comments. The format is designed to be parseable by a tiny script; you don't need a library.

## Reading UnicodeData.txt

UnicodeData.txt is the canonical per-code-point file. Each line has 15 fields separated by semicolons:

```
0041;LATIN CAPITAL LETTER A;Lu;0;L;;;;;N;;;;0061;
00E9;LATIN SMALL LETTER E WITH ACUTE;Ll;0;L;0065 0301;;;;N;LATIN SMALL LETTER E ACUTE;;00C9;;00C9
```

The fields, in order:

1. *Code Point* (hex).
2. *Name*.
3. *General Category* (Lu, Ll, Lt, Mn, Nd, etc.).
4. *Canonical Combining Class* (integer; non-zero for combining marks).
5. *Bidi Class* (L, R, AL, EN, ES, etc.).
6. *Decomposition Mapping* (e.g., `0065 0301` for `é`).
7. *Numeric Type/Value 1* (for decimal digits).
8. *Numeric Type/Value 2* (for digits more broadly).
9. *Numeric Type/Value 3* (for any character with a numeric value, e.g., Roman numerals).
10. *Bidi Mirrored* (Y/N).
11. *Unicode 1 Name* (historical).
12. *ISO Comment* (obsolete).
13. *Simple Uppercase Mapping*.
14. *Simple Lowercase Mapping*.
15. *Simple Titlecase Mapping*.

Most of the time you care about fields 1, 2, 3, 6, 13, and 14. The others are important for specific tasks — bidi algorithm implementations, numeric parsing — but not for everyday use.

### Range compression

UnicodeData.txt doesn't list every assigned code point. Large contiguous ranges (like the CJK ideographs at U+4E00–U+9FFF) are represented as paired `First` / `Last` lines:

```
4E00;<CJK Ideograph, First>;Lo;0;L;;;;;N;;;;;
9FFF;<CJK Ideograph, Last>;Lo;0;L;;;;;N;;;;;
```

Every code point in the closed range has the properties shown. A parser must expand these.

## Looking up properties in code

Every modern language has some way to query the UCD. Here are the most useful.

### Python: the `unicodedata` module

```python
import unicodedata as ud

ud.name("é")              # 'LATIN SMALL LETTER E WITH ACUTE'
ud.category("é")          # 'Ll'
ud.combining("̈")          # 230  (the combining diaeresis)
ud.decomposition("é")     # '0065 0301'
ud.normalize("NFD", "é")  # 'e\u0301'
ud.numeric("½")           # 0.5
ud.bidirectional("א")     # 'R' (right-to-left)
```

Python's `unicodedata` is bundled with the interpreter and refreshed with each Python release to match a specific Unicode version. It handles the common properties; for less common ones (Script, Grapheme_Cluster_Break), use PyICU or the `icu` crate's Python bindings.

### JavaScript: `Intl` and limited String methods

JavaScript's built-in Unicode queries are narrower. `String.prototype.normalize` is the main one. For properties, use `\p{…}` in Unicode-aware regex:

```javascript
/\p{Script=Greek}/u.test("α");     // true
/\p{General_Category=Lowercase_Letter}/u.test("a");  // true
```

For programmatic lookup (by code point → property), there is no built-in. Use the `unicode-properties` package or similar.

### C/C++: ICU

ICU's `u_charType(cp)`, `u_charName(cp, ...)`, `u_getIntPropertyValue(cp, ...)` are the low-level queries. They are fast (ICU ships compiled property tables) and comprehensive.

### The command line: `uni` and `unicode`

Two very useful CLI tools:

- `uni` ([github.com/arp242/uni](https://github.com/arp242/uni)): a standalone Go tool for looking up code points by name, identifier, or literal character.
- The Perl `unicode` one-liner: `perl -CS -E 'for (0..0x10FFFF) { printf "%04X %s\n", $_, charnames::viacode($_) if charnames::viacode($_) =~ /GRINNING/ }'`.

And of course, `python3 -c 'import unicodedata; print(unicodedata.name(chr(0x1F600)))'`.

## Unicode Utilities (unicode.org)

The Unicode Consortium publishes a web-based set of *Unicode Utilities* at [util.unicode.org](https://util.unicode.org/UnicodeJsps/). The useful ones:

- *Character Properties*: type any character or code point, see all its properties.
- *List Unicode Characters*: regex-style queries over the character set. `\p{Sc}` to list currency symbols.
- *Unicode Converter*: convert between UTF-8, UTF-16, UTF-32, code points, HTML escapes, etc.
- *Transform*: apply normalization forms, case folding, transliteration.

Bookmark these. They are faster than writing a script for a one-off question.

## Staying current

Each Unicode release ships new assigned code points, new property values, and occasionally new properties. The cadence is roughly one release per year. The UCD files have the version number in their directory path.

When you're using a language's built-in `unicodedata` (or equivalent), you are using whatever Unicode version that language was built against. If you need the latest version — for a new emoji, a new script addition, a recently added property — you may need to install a more current ICU or a third-party library that tracks upstream.

For most production code, being one Unicode version behind is fine. The standard is designed so that older tables never become wrong; they only become incomplete.

## The file that will teach you the most

If you want to understand Unicode at a deep level, spend an hour reading UnicodeData.txt. Start from U+0000 and scroll. Notice:

- The gaps where unassigned code points sit.
- The long runs of CJK ideographs (represented only as First/Last pairs).
- The combining marks cluster in the U+0300s and U+0800s.
- The mathematical operators in U+2200s.
- The emoji starting at U+1F300.
- The Private Use Area at U+E000–U+F8FF (15,000 code points reserved for non-standard use).
- The supplementary planes starting at U+10000.

It is a map of human writing, as of the current Unicode version. It is also a record of every committee decision the Unicode Consortium has made. You will come to appreciate that Unicode is not a mess — it is a negotiated peace.

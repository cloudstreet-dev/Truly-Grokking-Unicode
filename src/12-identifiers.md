# Identifier Characters and Programming Languages

Programming languages have identifier rules: what characters can appear in a variable name, a function name, a class name. ASCII-only is the old default; most modern languages allow at least some Unicode. This chapter covers what is allowed where, what the Unicode Standard recommends, and why your codebase probably shouldn't use non-ASCII identifiers even when it can.

## Unicode Annex #31

*UAX #31* is the Unicode annex that defines a recommended grammar for identifiers. It has two central properties:

- *ID_Start*: the set of code points that can begin an identifier. Roughly: letters (general category `L*`) plus letter-numbers (`Nl`).
- *ID_Continue*: the set of code points that can appear after the start. `ID_Start` plus digits (`Nd`), connector punctuation (`Pc`, which includes `_`), and a selection of combining marks (`Mn`, `Mc`).

A language that conforms to UAX #31 allows identifiers of the form `ID_Start ID_Continue*`.

UAX #31 also defines stricter variants:

- *XID_Start* / *XID_Continue*: slightly adjusted sets that guarantee stability under NFKC normalization (`NFKC(ident)` has the same ID structure as `ident`).
- *Pattern_Syntax* and *Pattern_White_Space*: code points reserved for use as syntax in patterns (not allowed in identifiers).

The main guidance of UAX #31: use `XID_Start` / `XID_Continue`, normalize identifiers to NFC (or NFKC), and apply a profile from UTS #39 if security matters.

## Language-by-language

### Python

Python 3 follows UAX #31 closely. Specifically:

- Identifier = `XID_Start XID_Continue*`.
- Identifiers are compared after NFKC normalization. This means `café` and `café` refer to the same variable regardless of precomposed vs. decomposed form; it also means `ﬁnalize` (with the U+FB01 ligature) and `finalize` are the same name.

```python
>>> π = 3.14159
>>> print(π)
3.14159
>>> café = "latte"
>>> café == café     # same name, different spelling
True  # they're literally the same variable after NFKC
```

Python rejects emoji as identifiers (they are not in `XID_Start`).

### JavaScript

ES2015 onward: identifiers use `ID_Start` and `ID_Continue` (not `XID_*`). JavaScript does *not* normalize identifiers — `café` (NFC) and `cafe\u0301` (NFD) are *different* variables, both valid.

```javascript
const café = 1;         // precomposed
const cafe\u0301 = 2;   // decomposed — different variable
```

This is a footgun. Some style guides recommend against non-ASCII identifiers in JavaScript for precisely this reason.

JavaScript also allows Unicode escape sequences in identifiers: `\u00e9` is equivalent to a literal `é` in an identifier.

Emoji are not in `ID_Start`, so they cannot begin an identifier, but a few emoji *are* in `ID_Continue` because of their category. In practice, no modern engine lets you name a variable `x🙂`; the spec allows only specific code points, not all emoji.

### Java

Java identifiers use `Character.isJavaIdentifierStart` / `isJavaIdentifierPart`, which are based on but not identical to UAX #31. They allow all letters (including all scripts), digits, underscore, and currency symbols.

```java
int π = 3;
String $ = "dollar";    // valid in Java
```

Java does not normalize identifiers; `café` and `cafe\u0301` are different variables.

### Go

Go allows identifiers of letter (`Unicode general category Lu, Ll, Lt, Lm, Lo`) + letter/digit (`Lu, Ll, Lt, Lm, Lo, Nd`). Underscore is treated as a letter.

```go
var π = 3.14
func σ(x float64) float64 { return x * x }
```

Go does *not* normalize identifiers either. And Go has a visibility rule tied to the identifier: exported names must start with an uppercase letter. This is computed by Unicode's case property: `π` (lowercase pi) is unexported; `Π` (uppercase pi) is exported. The rule applies across every script that has case.

### Rust

Rust follows UAX #31 strictly. Identifiers are `XID_Start XID_Continue*`. Rust normalizes to NFC for identifier comparison.

Since Rust 1.53 (the "non-ASCII identifier" RFC), you can write:

```rust
let π = 3.14;
let café = "double espresso";
```

Rust specifically forbids identifiers whose NFKC normalization changes them (preventing a category of confusable bugs), and emits warnings for mixed-script identifiers via the `non_ascii_idents` lint family.

### Swift

Swift identifiers are extremely permissive. The start set includes letters, most symbols (including emoji), and some others; the continue set adds digits and combining marks.

```swift
let 🎉 = "party"         // valid
let 🐕 = "dog"
```

Swift's permissiveness has produced the most photogenic "look how quirky our language is" code samples on social media. It has not produced a lot of real production code using emoji as identifiers.

### C and C++

C11 and C++11 allowed limited Unicode in identifiers via `\u` / `\U` escapes. C++23 and recent C standards adopted UAX #31. Compiler support varies; GCC and Clang largely conform.

## Normalization and identifiers

The identifier equality question has two answers:

1. *Byte-equal*: two identifiers are the same iff their code points are identical. (JavaScript, Java, Go.)
2. *Normalization-equal*: two identifiers are the same iff their NFC (or NFKC) normalizations are identical. (Python, Rust.)

Normalization-equal is safer, because it prevents a category of confusable identifiers from coexisting. It also costs a little: the compiler must normalize every identifier before comparison. Most modern languages pick normalization-equal; the older ones stuck with byte-equal because that was how their string tables already worked.

## Mixed-script identifiers as a security concern

Consider a Python codebase with a variable named `admin`. An attacker contributing a PR introduces a function using `аdmin` — where the first letter is the Cyrillic `а` (U+0430). In a code review, the two look identical. Python's NFKC normalization does not fold Cyrillic to Latin, so `admin` and `аdmin` are distinct variables.

The attacker can now define `аdmin = True` in a module, and the reviewer who reads it as `admin = True` has no way to tell from the visible source that this is a different variable. Later code that references `admin` will use the real Latin one; the attacker's definition has no effect, but a clever variant of this attack can introduce bugs, dead code, or subtle vulnerabilities.

The mitigations are the same as for usernames (Chapter 10):

- Restrict identifiers to a limited set of scripts (UTS #39 profiles).
- Warn on mixed-script identifiers.
- Lint for confusables with existing names.

Rust's `non_ascii_idents` lint family includes `confusable_idents`, `mixed_script_confusables`, and `uncommon_codepoints`. Python has PEP 672 discussing the risks but does not yet enforce a restrictive profile. Most other languages leave this to external tools.

## Why your codebase shouldn't use non-ASCII identifiers

Even when your language allows it, the pragmatic recommendation is: *don't*.

- *Tooling*: grep, diff, and many legacy tools assume ASCII. They will often still work on UTF-8 identifiers, but weirdly.
- *Keyboards*: not every developer has every character on their keyboard. Typing `π` requires a Compose sequence or a Unicode input method, which slows code contribution.
- *Merge conflicts*: NFC vs. NFD differences that aren't distinguishable on screen can produce git conflicts that look like phantoms.
- *Editors*: older editors or corporate-mandated IDEs may not handle complex scripts (right-to-left, combining marks) correctly.
- *Search*: code search tools may not find `café` when you search for `cafe`, because identifiers with accents are less likely to show up in "standard" search queries.
- *Contributor inclusion*: if your project welcomes non-English-speaking contributors, having ASCII identifiers is the lowest-friction common denominator.
- *Security*: mixed-script attacks are possible, as above.

There are counter-arguments for mathematical or scientific code that specifically benefits from symbolic names (`π`, `σ`, `∇`). Those use cases are narrow and usually restricted to one clearly-scoped module.

The default for a new codebase: ASCII identifiers, with UTF-8-aware tooling for everything else.

## String *literals* are different

None of the above applies to string literals and comments. Your string literals should absolutely be able to contain any Unicode your users will produce: `"Hello, 世界"`, `"¿Qué tal?"`, `"🎉"`. The rules in this chapter are only about *identifiers* — the names of variables, functions, types, and other things your compiler tracks.

A good default:

- Identifiers: ASCII.
- String literals: UTF-8, including whatever Unicode the application needs.
- Source file encoding: UTF-8, with no BOM.

That set of choices avoids every Unicode-identifier hazard while losing nothing about your ability to handle Unicode *data*.

Next we look at the Unicode database itself — the data file that backs every property we have discussed.

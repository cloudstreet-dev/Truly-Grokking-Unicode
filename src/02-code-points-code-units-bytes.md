# Code Points, Code Units, Bytes

This is the most important chapter in the book. It is also the shortest, because the ideas are small; they are only confusing when they are left implicit. Once you name them, Unicode becomes tractable.

## Three levels of abstraction

Think of text as sitting on three layers:

```
┌────────────────────────────────────────────────────┐
│  The abstract character the Unicode Standard       │  <- code point
│  has assigned a number to.                         │
│      U+0041 = A   U+00E9 = é   U+1F600 = 😀        │
└────────────────────────────────────────────────────┘
                        │  (encoded by UTF-8, UTF-16, ...)
                        ▼
┌────────────────────────────────────────────────────┐
│  A sequence of atomic units of some encoding.      │  <- code units
│  UTF-8: 8-bit units.  UTF-16: 16-bit units.        │
└────────────────────────────────────────────────────┘
                        │  (serialized to memory/disk)
                        ▼
┌────────────────────────────────────────────────────┐
│  The bytes on the wire or on disk. Endianness may  │  <- bytes
│  matter for multi-byte code units.                 │
└────────────────────────────────────────────────────┘
```

You need names for all three of these layers, and you need to not confuse them.

### Code point

A *code point* is an integer assigned by the Unicode Standard to a specific abstract character. There are 1,114,112 possible code points, in the range U+0000 through U+10FFFF. About 155,000 are currently assigned; the rest are reserved.

A code point is written in hexadecimal with a `U+` prefix and at least four digits: `U+0041`, `U+00E9`, `U+1F600`. There are no leading zeros beyond the fourth digit, but the four-digit minimum is conventional.

A code point is *not* a number of bytes. `U+1F600` is the integer 128,512. Whether that integer takes 1, 2, 3, or 4 bytes to store depends on the encoding, which we are about to discuss. The code point itself is indifferent to storage.

Code points are conceptually grouped into 17 *planes* of 65,536 code points each:

- *Plane 0*, the *Basic Multilingual Plane* (BMP): U+0000–U+FFFF. Contains nearly every character used in modern living languages.
- *Plane 1*, the *Supplementary Multilingual Plane* (SMP): U+10000–U+1FFFF. Emoji, historical scripts, musical notation.
- *Plane 2*, the *Supplementary Ideographic Plane* (SIP): U+20000–U+2FFFF. Additional CJK ideographs.
- *Planes 3–13*: mostly unassigned.
- *Plane 14*: special-purpose characters, including tags and variation selectors.
- *Planes 15–16*: private use areas.

Why does the total max out at U+10FFFF and not something neater, like U+FFFFFF? Because of UTF-16. U+10FFFF is the largest code point that UTF-16 can represent with a surrogate pair. Unicode's upper bound is literally set by the encoding capacity of one of its encodings — a retrofit.

### Code unit

A *code unit* is the atomic piece of whatever encoding you are using. It is a fixed-size integer, and the size depends on the encoding:

- UTF-8 has 8-bit code units (so each code unit is also a byte).
- UTF-16 has 16-bit code units.
- UTF-32 has 32-bit code units.

A single code point may be represented by *one or more* code units, depending on the encoding and the code point. In UTF-8, code points above U+007F take multiple code units. In UTF-16, code points above U+FFFF take two code units. In UTF-32, every code point is exactly one code unit.

When a string API tells you its *length*, you need to know what unit it is counting. JavaScript counts UTF-16 code units. Python counts code points. Go's `len()` counts bytes. Rust's `str::len()` also counts bytes. Swift's `String.count` counts grapheme clusters (we will get to those in Chapter 4).

All of these are legitimate answers to "how long is this string?" depending on what you mean by "long." None of them is "the number of characters," because *character* has at least four meanings.

### Byte

A *byte* is eight bits. What ends up in a file or on a network socket. Code units that are one byte wide (UTF-8) are directly bytes. Code units that are wider than a byte (UTF-16, UTF-32) have to be serialized to bytes, which means choosing an *endianness* — most significant byte first, or least significant byte first. More on that when we cover UTF-16 in Chapter 3.

## A worked example: `"é😀"` in three encodings

Take a two-character string, `é😀`. In code points:

```
U+00E9 LATIN SMALL LETTER E WITH ACUTE
U+1F600 GRINNING FACE
```

Two code points. That's the abstract level; every encoding has to represent these two code points somehow.

### UTF-8

UTF-8 is a variable-width encoding with 8-bit code units. Here are the rules, which we will justify in Chapter 3:

- Code points U+0000–U+007F: 1 code unit. `0xxxxxxx`.
- Code points U+0080–U+07FF: 2 code units. `110xxxxx 10xxxxxx`.
- Code points U+0800–U+FFFF: 3 code units. `1110xxxx 10xxxxxx 10xxxxxx`.
- Code points U+10000–U+10FFFF: 4 code units. `11110xxx 10xxxxxx 10xxxxxx 10xxxxxx`.

U+00E9 = 0xE9 = decimal 233 falls in the second range. The binary is `11101001`. Padded to the 2-code-unit pattern:

```
110xxxxx 10xxxxxx
   00011    101001
```

which becomes `11000011 10101001` = `C3 A9`.

U+1F600 = decimal 128512. Binary `0001 1111 0110 0000 0000` (20 bits). Fits the 4-code-unit pattern:

```
11110xxx 10xxxxxx 10xxxxxx 10xxxxxx
     000   011111   011000   000000
```

which becomes `11110000 10011111 10011000 10000000` = `F0 9F 98 80`.

So the string `é😀` in UTF-8 is 6 bytes:

```
C3 A9 F0 9F 98 80
```

### UTF-16

UTF-16 uses 16-bit code units. For code points in the BMP (U+0000–U+FFFF), a single code unit holds the code point directly. For supplementary code points (U+10000–U+10FFFF), two code units called a *surrogate pair* are used.

U+00E9 is in the BMP, so it becomes one code unit: `0x00E9`.

U+1F600 is supplementary. The surrogate pair rule:

1. Subtract 0x10000: 0x1F600 − 0x10000 = 0x0F600.
2. Split the resulting 20-bit number into two 10-bit halves: high = 0x03D, low = 0x200.
3. High surrogate: 0xD800 + 0x03D = 0xD83D.
4. Low surrogate: 0xDC00 + 0x200 = 0xDE00.

So U+1F600 is the UTF-16 code unit pair `D83D DE00`.

The string `é😀` in UTF-16 is three code units:

```
00E9 D83D DE00
```

In bytes, we also have to choose endianness. Big-endian (UTF-16BE):

```
00 E9 D8 3D DE 00    (6 bytes)
```

Little-endian (UTF-16LE):

```
E9 00 3D D8 00 DE    (6 bytes)
```

(It is a coincidence that UTF-8 and UTF-16 land on the same byte count here. In general they do not. ASCII text is much smaller in UTF-8; CJK text is smaller in UTF-16.)

### UTF-32

UTF-32 uses one 32-bit code unit per code point. No surrogates, no variable width.

```
000000E9 0001F600     (2 code units, 8 bytes)
```

In big-endian bytes:

```
00 00 00 E9 00 01 F6 00
```

Most of those bytes are zeroes. That is why UTF-32 is rarely used in transit or on disk: it's fixed-width but space-inefficient.

## The same string, four "lengths"

Given `é😀`, here are four numbers, all of which could be called *the length*:

| What you count | Value |
|---|---|
| Code points | 2 |
| UTF-8 code units (= bytes) | 6 |
| UTF-16 code units | 3 |
| UTF-32 code units (= bytes / 4) | 2 |
| Grapheme clusters | 2 |

Here is what various languages say:

```python
# Python 3: len() counts code points
>>> len("é😀")
2
>>> len("é😀".encode("utf-8"))
6
```

```javascript
// JavaScript: .length counts UTF-16 code units
> "é😀".length
3
> [..."é😀"].length   // iterator splits by code point
2
```

```go
// Go: len() on a string counts bytes (Go strings are UTF-8)
len("é😀")                             // 6
utf8.RuneCountInString("é😀")          // 2  (code points)
```

```rust
// Rust: str::len() counts bytes
"é😀".len()              // 6
"é😀".chars().count()    // 2  (code points)
```

```swift
// Swift: String.count counts grapheme clusters
"é😀".count              // 2  (grapheme clusters, same as code points here)
"é😀".utf16.count        // 3
"é😀".utf8.count         // 6
```

None of these languages is wrong. They are each counting a specific, well-defined thing. The confusion comes from calling them all "length."

In Chapter 4 we will look at strings where the code-point count and the grapheme-cluster count disagree. You will like those examples. They are where the bugs live.

## What to remember

- A *code point* is a number that Unicode assigned to a character. It is encoding-independent.
- A *code unit* is the atomic piece of some encoding. Its size depends on the encoding.
- A *byte* is what you store. It equals a code unit only in UTF-8.
- When a function returns *the length* of a string, find out which of these it is counting.

Next, we'll look at the encodings themselves — how they work, why UTF-8 is the shape it is, and how to read its bytes with your bare eyes.

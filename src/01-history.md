# A Brief, Honest History

To understand Unicode, you have to understand what came before it, because Unicode's shape is a direct response to the problems that preceded it. This chapter is not nostalgic. It is diagnostic: each old encoding is a wound, and Unicode is the stitches.

## ASCII: 128 characters, 7 bits, 1963

*ASCII* (the American Standard Code for Information Interchange) is a table that maps the integers 0 through 127 to a set of 128 characters: the uppercase and lowercase Latin letters, the digits, a handful of punctuation marks, and a number of control characters like *newline* (10), *tab* (9), and *carriage return* (13). Each integer fits in seven bits. The eighth bit of a byte, on the hardware of the early 1960s, was typically used for parity.

ASCII is small, regular, and entirely adequate for *written English*, which is what its designers cared about. Here are the printable ASCII letters:

```
0x41  A        0x61  a
0x42  B        0x62  b
0x43  C        0x63  c
...            ...
0x5A  Z        0x7A  z
```

The uppercase and lowercase letters differ by exactly one bit (0x20). This is a design choice, not a coincidence; it made case folding cheap in hardware. You will see it pay off later in this book.

You already knew ASCII existed. The important thing to notice is what it does *not* contain. No `é`. No `ñ`. No `ß`. No Cyrillic, no Greek, no Arabic, no Hebrew, no Devanagari, no CJK. ASCII describes English and only English.

## The 8-bit wild west

Throughout the 1980s and 1990s, the eighth bit of the byte — no longer needed for parity — became an opportunity. You could assign meanings to the 128 values from 128 to 255 and double the size of your character set. *Everyone did.*

- *ISO 8859-1*, also called *Latin-1*, filled 128–255 with accented Latin letters for Western European languages: `é`, `ñ`, `ü`, `ø`, and so on.
- *ISO 8859-2* (Latin-2) did the same for Central and Eastern European languages: `č`, `ł`, `ő`, `ř`.
- *ISO 8859-5* was Cyrillic.
- *ISO 8859-6* was Arabic.
- *KOI8-R* was a different arrangement of Cyrillic, popular in Russia, designed so that if you stripped the high bit you got approximate Latin transliterations of the letters. `Д` (position 228) stripped to `d` (position 100).
- *Windows-1252* was Microsoft's extension of Latin-1 that stuffed additional characters (`€`, `™`, `"`, `"`) into the 0x80–0x9F range, which ISO 8859-1 had left for control codes. This is the encoding Windows calls *ANSI*, a name that has nothing to do with ANSI.
- *Shift-JIS* (Japanese), *Big5* (Traditional Chinese), and *GB2312* (Simplified Chinese) were *multi-byte* encodings — they used either one or two bytes per character, with a lead-byte convention to distinguish. These had to exist because Japanese and Chinese have tens of thousands of characters and cannot possibly fit in 256 slots.

The result, by 1995, was a thousand-character problem: the same bytes meant different things depending on which encoding the software had in mind, and there was no reliable way to find out which one that was.

Example: the byte `E9` decoded as Latin-1 is `é`. Decoded as Windows-1252 it is also `é` (these two encodings agree on this byte). Decoded as KOI8-R it is `щ`. Decoded as Shift-JIS it is either an error or, depending on context, part of a two-byte sequence. One byte, four different meanings, no in-band way to tell which.

When your data was born inside one encoding and consumed as another, the result had a name: *mojibake* — text turned to garbled characters. 文字化け. You have seen it.

## Unicode's goal

The founders of Unicode looked at this and asked a radical question. What if there were *one character set*, large enough to contain every writing system ever used by humans, and every symbol anyone ever wanted to standardize — and what if *every* encoding from then on was merely a way to serialize that single set to bytes?

Then the mojibake problem becomes a *transport* problem, not a *meaning* problem. The text "the letter é" would have one globally agreed-upon identity. If different programs serialized it into different bytes, they could still agree on what the underlying text was.

That single character set is what *Unicode* is. Specifically, Unicode assigns an integer (a *code point*) to every character it standardizes. Currently, there are 1,114,112 possible code points (the numbers 0 through 1,114,111), of which about 155,000 are assigned as of Unicode 16.0. The rest are reserved for future use.

Code points are written in hexadecimal with a `U+` prefix, zero-padded to at least four digits: `U+0041` is `A`, `U+00E9` is `é`, `U+1F600` is `😀`, `U+2603` is `☃`.

That is the whole big idea. One integer per character. The "character set" problem is solved by fiat: we all use the same set.

## The UCS-2 mistake

Unicode was originally designed around a different assumption. In its first version (1991), Unicode believed it could fit every character it would ever need into 65,536 slots — the range of a 16-bit integer. Two bytes per character, always, forever. The encoding that served this assumption was called *UCS-2*: pack each 16-bit code point into two bytes and you're done. Strings became arrays of 16-bit units and each unit was a character.

This was a mistake, but an honest one: nobody had yet counted how many distinct CJK characters actually existed in real historical use, and the first estimates were wildly low.

By 1996, Unicode had to concede that 65,536 code points were not enough. The character space was extended to 1,114,112 slots. But by then, several large systems had already committed to *two bytes per character, always*:

- *Windows NT* used UCS-2 internally, and its file APIs (`CreateFileW`, the `W` family) took 16-bit units.
- *Java*, launched in 1995, defined `char` as a 16-bit unit and `String` as an array of them.
- *JavaScript*, specified in 1997, defined string indexing in terms of 16-bit units.
- *Objective-C*'s `NSString` used 16-bit units.

All of these languages and runtimes had to retrofit a way to represent the *supplementary* code points (the ones above U+FFFF) using pairs of 16-bit units. That retrofit is called *surrogate pairs*, and it is the reason `"😀".length` is 2 in JavaScript. We will cover surrogate pairs in detail in Chapter 3.

The lesson is this: UCS-2 is *not the same as UTF-16*. UCS-2 was the original fixed-width two-bytes-per-code-point encoding, and it cannot represent anything above U+FFFF. UTF-16 is its successor — a variable-width encoding that uses one 16-bit unit for code points in the *Basic Multilingual Plane* (U+0000–U+FFFF) and a pair of 16-bit units for everything else. Languages that still call their string unit a `char` (Java, JavaScript, C#) are living in the space where this retrofit happened.

## Where we are now

Today:

- *UTF-8* is dominant on the web (around 98% of pages, by most counts), on Linux filesystems, in most network protocols, and in most modern programming languages' default I/O.
- *UTF-16* persists inside Windows, Java, JavaScript, and anywhere else that made the UCS-2 bet in the 1990s. It is still the *internal* string representation of those systems even when their I/O is UTF-8.
- *UTF-32* exists, is occasionally useful for internal work, and is almost never used for interchange.
- The single-byte encodings (Latin-1, Windows-1252, Shift-JIS, KOI8-R, the whole ISO 8859 series) are a dwindling but persistent minority. They live in legacy databases, old email archives, filesystems with historical data, and a truly surprising number of CSV exports from enterprise software.

Unicode is, at this point, not one of several character sets. It *is* the character set, and the other things people once called character sets are now best understood as alternative ways of not quite encoding Unicode.

With that in mind, we are ready to draw the distinctions that matter.

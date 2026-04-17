# The Encodings

Three encodings of Unicode matter in practice: UTF-8, UTF-16, and UTF-32. This chapter explains what each is doing mechanically, why it is shaped the way it is, and what pain each one can cause when you meet it in the wild.

## UTF-8

*UTF-8* is a variable-width encoding with 8-bit code units. It has four critical properties, and if you understand the properties, you understand the encoding.

### Property 1: ASCII-compatible

Any ASCII file (every byte < 0x80) is a valid UTF-8 file, and means exactly the same thing in UTF-8 as it did in ASCII. This is enormously important for backward compatibility. Every program that handled ASCII continues to handle UTF-8 *in part* without modification — it will just not understand the non-ASCII parts.

### Property 2: Self-synchronizing

Given an arbitrary byte in a UTF-8 stream, you can tell immediately whether it is a *start byte* (the first byte of a code point's encoding) or a *continuation byte* (a non-first byte):

| Byte pattern | Meaning |
|---|---|
| `0xxxxxxx` | 1-byte sequence (ASCII, U+0000–U+007F) |
| `110xxxxx` | start of 2-byte sequence |
| `1110xxxx` | start of 3-byte sequence |
| `11110xxx` | start of 4-byte sequence |
| `10xxxxxx` | continuation byte |

If you land in the middle of a stream and see `10xxxxxx`, you know you are mid-character; back up until you see something that isn't a continuation byte. This is what *self-synchronizing* means. It is why dropping a byte from a UTF-8 stream corrupts exactly one character, not the entire rest of the stream.

### Property 3: Variable-width, but bounded

Code points use 1, 2, 3, or 4 bytes in UTF-8. The number of bytes is determined by the code point's value:

| Code point range | Bytes | Pattern |
|---|---|---|
| U+0000 – U+007F | 1 | `0xxxxxxx` |
| U+0080 – U+07FF | 2 | `110xxxxx 10xxxxxx` |
| U+0800 – U+FFFF | 3 | `1110xxxx 10xxxxxx 10xxxxxx` |
| U+10000 – U+10FFFF | 4 | `11110xxx 10xxxxxx 10xxxxxx 10xxxxxx` |

The `xxx` bits are taken, in order, from the code point's binary representation (most-significant bit first) and packed into the `x` slots.

### Property 4: Lexicographic byte order matches code point order

If you compare two UTF-8-encoded strings byte by byte (as raw bytes, unsigned), the result is the same as comparing them code point by code point. This is a very convenient property: databases, filesystems, and network protocols that sort by bytes end up sorting Unicode strings by code point "for free." (This is not the same as sorting them *correctly for humans*, which we will cover in Chapter 6. But it is stable and predictable.)

### Reading a UTF-8 byte sequence with your bare eyes

Here are the bytes of our example string `é😀`:

```
C3 A9 F0 9F 98 80
```

- `C3` = `11000011`. Top bits `110` → start of 2-byte sequence. The 5 payload bits are `00011`.
- `A9` = `10101001`. Top bits `10` → continuation byte. The 6 payload bits are `101001`.
- Combine the payload bits: `00011 101001` = `00011101001` = 0xE9 = U+00E9 = `é`. ✓
- `F0` = `11110000`. Top bits `11110` → start of 4-byte sequence. Payload bits: `000`.
- `9F` = `10011111`. Continuation. Payload: `011111`.
- `98` = `10011000`. Continuation. Payload: `011000`.
- `80` = `10000000`. Continuation. Payload: `000000`.
- Combine: `000 011111 011000 000000` = `11111011000000000` = 0x1F600 = U+1F600 = `😀`. ✓

You can do this in your head with practice. It will pay off the first time you debug a mojibake problem at the byte level.

### Overlong encodings and why they're forbidden

The UTF-8 rules technically allow the same code point to be encoded in more than one way: you could encode U+0041 (`A`) in 1 byte (`0x41`) or "pad" it into a 2-byte sequence (`0xC1 0x81`) or a 3-byte sequence. The multi-byte forms are called *overlong* encodings, and they are forbidden by the UTF-8 spec.

The reason matters for security: if overlong encodings were legal, a filter that blocked `/` (0x2F) by looking only for the byte 0x2F could be bypassed by sending the overlong 2-byte encoding `0xC0 0xAF` or the 3-byte encoding `0xE0 0x80 0xAF`, both of which "mean" `/` but contain no 0x2F byte. Several real path-traversal vulnerabilities worked this way in the early 2000s, most famously in IIS. Modern UTF-8 decoders reject overlong forms.

### Valid code point range limits

A well-formed UTF-8 decoder rejects:

- Overlong encodings (as above).
- Bytes `0xC0`, `0xC1` (can only appear as overlong starts).
- Bytes `0xF5`–`0xFF` (would encode code points above U+10FFFF, which don't exist).
- UTF-16 surrogates (U+D800–U+DFFF) encoded in UTF-8. These code points are reserved for the UTF-16 surrogate mechanism and are not legal stand-alone characters. UTF-8 that contains them is *CESU-8* or *WTF-8*, not proper UTF-8.

## UTF-16

*UTF-16* is a variable-width encoding with 16-bit code units. It is the encoding that most UCS-2 systems retrofitted to when it became clear that 16 bits were not enough.

### BMP code points

For a code point in the range U+0000 – U+FFFF (the Basic Multilingual Plane), UTF-16 uses a single 16-bit code unit whose value equals the code point. This is the part that UCS-2 already did, and it is why UCS-2 and UTF-16 produce the same bytes for BMP text.

There is one wrinkle: the range U+D800 – U+DFFF is *reserved* in the Unicode standard — no characters are ever assigned there. That range is reserved specifically for UTF-16's surrogate mechanism.

### Supplementary code points: surrogate pairs

For a code point in U+10000 – U+10FFFF (supplementary), UTF-16 uses two 16-bit code units called a *surrogate pair*. The encoding:

1. Let `cp` be the code point. Subtract `0x10000` to get a 20-bit value `v`.
2. The *high surrogate* is `0xD800 | (v >> 10)`. This is in the range `0xD800 – 0xDBFF`.
3. The *low surrogate* is `0xDC00 | (v & 0x3FF)`. This is in the range `0xDC00 – 0xDFFF`.

Both halves are always 16-bit code units, and the high/low ranges don't overlap, so decoders can always tell which half of a pair they are looking at. A high surrogate without a following low surrogate (or vice versa) is a *lone surrogate* — malformed UTF-16.

Example: U+1F600 (😀).

```
cp  = 0x1F600
v   = cp - 0x10000 = 0x0F600
high = 0xD800 | (0x0F600 >> 10) = 0xD800 | 0x03D = 0xD83D
low  = 0xDC00 | (0x0F600 & 0x3FF) = 0xDC00 | 0x200 = 0xDE00
```

So 😀 in UTF-16 is the pair `D83D DE00`. You saw this in the previous chapter; now you know where it came from.

### Endianness and UTF-16

Because UTF-16 code units are 16 bits wide, serializing them to bytes requires picking an endianness. There are three related encodings:

- *UTF-16BE* — big-endian, most significant byte first. `D83D` → `D8 3D`.
- *UTF-16LE* — little-endian, least significant byte first. `D83D` → `3D D8`.
- *UTF-16* — the abstract encoding, which in practice is either BE or LE with an optional *Byte Order Mark* (BOM) at the start to tell you which.

The BOM is the code point U+FEFF (*ZERO WIDTH NO-BREAK SPACE*), encoded per the chosen endianness. In UTF-16BE the BOM is the bytes `FE FF`; in UTF-16LE it is `FF FE`. Some files have a BOM; some don't. Some specs require one; others forbid it. This is an endless source of pain.

### Lone surrogates, WTF-8, and JavaScript's leaky abstraction

JavaScript strings are sequences of 16-bit code units, not sequences of code points. You can create a JavaScript string containing a *lone surrogate* — a high surrogate not followed by a low surrogate, or vice versa. That string is not valid UTF-16 in the strict sense, but it is a valid JavaScript string.

```javascript
const s = "\uD83D";    // a lone high surrogate
s.length;              // 1
s.charCodeAt(0);       // 55357
```

If you try to JSON-encode this string, you get the escape `"\ud83d"`, which some JSON parsers tolerate and some reject. If you try to encode it as UTF-8, well-behaved encoders will refuse or replace it with U+FFFD (REPLACEMENT CHARACTER). WHATWG specs have a concept called *WTF-8*, which is UTF-8 that also encodes surrogates — a pragmatic compromise for dealing with JavaScript strings that contain them.

This is the tax you pay for UCS-2's ghost. In Python 3, you cannot construct a string containing a lone surrogate (well, not easily; `surrogateescape` is a specific escape hatch for filesystem paths). In JavaScript, you can, and so the language's `String` type is *not* quite "a sequence of Unicode code points."

## UTF-32

*UTF-32* uses one 32-bit code unit per code point, directly. No surrogates, no variable width, no rules to remember. The code point 0x1F600 is the code unit 0x0001F600.

This is conceptually the simplest encoding and is occasionally useful in-memory for algorithms that want random access to code points. But:

- It quadruples the size of ASCII text.
- It doubles the size of BMP text vs. UTF-16.
- It is almost never used on the wire or on disk.

In practice, UTF-32 shows up as an *internal* representation in some C libraries (`wchar_t` is 32 bits on Linux, 16 on Windows — we'll revisit in Chapter 7), and it is the implicit representation when a language like Python 3 gives you a string whose indexing counts code points. Python 3 CPython actually uses a flexible internal representation (1, 2, or 4 bytes per code unit depending on the string's largest code point), but semantically it behaves like UTF-32.

## Byte Order Marks, in detail

The *Byte Order Mark* (BOM) is the Unicode code point U+FEFF at the start of a file or stream. Its purpose depends on the encoding:

- In UTF-16 and UTF-32, the BOM genuinely carries information: it tells the reader whether the stream is big- or little-endian.
- In UTF-8, the BOM carries no byte-order information (UTF-8 has no byte-order choice). It exists only as a *signature* — a marker that the file is UTF-8. Its bytes are `EF BB BF`.

The UTF-8 BOM is controversial. Some tools expect it (Windows Notepad used to write it by default); some tools treat it as a literal character at the start of the file, breaking things. Unix tools tend not to expect it, so a UTF-8 BOM at the start of a shell script will cause the shebang line to not be recognized; a UTF-8 BOM at the start of a CSV will show up as a weird `` `` before your first column header.

*When to use a BOM:*

- *UTF-16* and *UTF-32* files: yes, if you have any reason to think the consumer might not know the endianness.
- *UTF-8* files: prefer not to. If the file's encoding is documented out-of-band (HTTP `Content-Type`, filename convention, project convention), don't add a BOM. Add one only if consumers explicitly require it.
- *Never* include a BOM in protocols that forbid it — JSON (RFC 8259), for instance, is required to be UTF-8 without a BOM.

*When to strip a BOM:*

- When reading input of unknown provenance, be liberal: if you see `EF BB BF` at the start of a UTF-8 stream, eat it. Most modern language stdlibs have an encoding called `utf-8-sig` that does exactly this.

## Choosing an encoding today

For almost every new use case, *the answer is UTF-8*, for the following reasons:

1. It is ASCII-compatible. Any ASCII tooling works on the ASCII portion.
2. It is the default encoding of the web, Unix filesystems, most databases, and most network protocols.
3. It is endian-free. No BOM needed, no byte-swapping at ingress.
4. It degrades gracefully: a stray byte corrupts exactly one character, not the rest of the stream.
5. For text with a lot of ASCII (source code, HTML, JSON), it is the most compact option.

Reasons to use something else:

- You are working inside a system whose internal string type is UTF-16 (JavaScript, JVM, Windows). Then the trade-off is at the boundary: your storage and I/O should still typically be UTF-8.
- You are working with almost entirely CJK text, where UTF-16 is slightly more compact than UTF-8.
- You need fixed-width random access to code points for an algorithm, and you cannot afford to iterate. Then UTF-32 in memory, UTF-8 everywhere else.

With the encodings in hand, we are ready to confront the most commonly misunderstood concept in all of Unicode: what a "character" actually is.

# Input, Output, and Interchange

Text does not live inside your program. It arrives from somewhere — a file, a socket, a form submission — and it leaves for somewhere. Every boundary between "in-memory strings" and "bytes on the wire" is a chance to decode or encode incorrectly, and most real-world Unicode bugs happen at these boundaries. This chapter maps the boundaries and the conventions that govern them.

The single most important principle: *encoding is a transport concern, not an identity concern.* At any boundary, you need to know which encoding is in use. The encoding is metadata, not data — it is information *about* the bytes, and it must be carried alongside the bytes (or established by convention between sender and receiver).

## HTTP

HTTP carries encoding information in the `Content-Type` header for text bodies:

```
Content-Type: text/html; charset=utf-8
Content-Type: application/json
Content-Type: text/plain; charset=iso-8859-1
```

- `text/*` types may specify a `charset` parameter. If absent, the historical default (RFC 2616) was `ISO-8859-1`, but modern practice and most tools default to `UTF-8` or infer from the body.
- `application/json` has a special rule: RFC 8259 requires JSON to be UTF-8 on the wire, and a `charset` parameter is not standard. Don't add one; some clients will reject it.
- `application/xml` and `text/xml` are ambiguous; the XML declaration inside the body is more authoritative than the `Content-Type`.

For *requests* (forms, uploads):

- `application/x-www-form-urlencoded`: form fields are percent-encoded. The encoding of the underlying bytes before percent-encoding is implicit; browsers use UTF-8 for forms on UTF-8 pages.
- `multipart/form-data`: each part can have its own `Content-Type` with a `charset` parameter, but most senders don't set it.

The lesson: for APIs you design, document the encoding, use UTF-8, and never rely on inference.

## HTML

HTML's encoding rules are layered, and browsers consult them in a specific order:

1. The byte-order-mark, if present (UTF-8 BOM `EF BB BF`, UTF-16 BOM `FE FF` or `FF FE`).
2. The HTTP `Content-Type` header's `charset` parameter.
3. The `<meta charset="utf-8">` tag (or old-style `<meta http-equiv="Content-Type" content="text/html; charset=utf-8">`), if it appears in the first 1024 bytes.
4. Browser's character-set-sniffing heuristics.
5. The user's locale default.

A well-formed HTML file in 2026 has:

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  ...
</head>
```

The `<meta charset>` tag must appear early enough that the encoding is known before any non-ASCII text is parsed. Browsers spec-mandate that it be in the first 1024 bytes.

## XML

XML's encoding is declared in the XML declaration at the top of the file:

```xml
<?xml version="1.0" encoding="utf-8"?>
```

The declaration is ASCII-only (required to be, so that any ASCII-compatible encoding will allow the declaration itself to be parsed). If no declaration is present, the spec says:

- If there is a BOM, use the corresponding encoding.
- Otherwise assume UTF-8.

XML processors are required to support at least UTF-8 and UTF-16.

## JSON

JSON is required to be Unicode. RFC 8259 (the current JSON spec) says JSON *must* be UTF-8 when exchanged between systems. The older RFC 7159 allowed UTF-8, UTF-16, or UTF-32; RFC 8259 tightened to UTF-8 only.

Inside a JSON string, non-ASCII characters can be escaped as `\uXXXX` (four hex digits = one UTF-16 code unit), or written literally. Supplementary code points must be written as surrogate pairs in escape form:

```json
{"face": "\uD83D\uDE00"}      // valid, 😀 encoded as surrogate pair
{"face": "😀"}                 // also valid, literal emoji in UTF-8
```

This surrogate-pair escape is a leaky abstraction from JavaScript — the only reason JSON expresses supplementary code points this way is that its lineage runs through JavaScript, which expresses strings in UTF-16. A strict JSON producer can choose which style to use; most modern producers write the literal UTF-8 bytes and only escape what must be escaped (quote, backslash, control characters).

A subtle pitfall: lone surrogates. JSON does not forbid them at the grammar level, so a JavaScript string containing a lone surrogate can be round-tripped through `JSON.stringify` and `JSON.parse`. But that JSON is not valid UTF-8 when the lone surrogate is written literally, and many parsers will reject escape-form lone surrogates too. This is one of the real sources of interoperability failure.

## File I/O

### The cardinal sin of `open()` without an encoding

```python
# WRONG in the general case:
with open("data.txt") as f:
    text = f.read()
```

In Python 3, `open()` in text mode uses a *default encoding*, which depends on the platform and environment:

- Since Python 3.15 (and configurable since 3.10 via `PYTHONUTF8=1` or `-X utf8`), the default is UTF-8 on all platforms.
- Before that, the default was `locale.getpreferredencoding()`, which could be `cp1252` on Windows, `UTF-8` on Linux, and so on. Same code, different platforms, different results.

Always specify:

```python
with open("data.txt", encoding="utf-8") as f:
    text = f.read()
```

And if you're reading bytes of uncertain provenance (a UTF-8 file that may have a BOM), use `encoding="utf-8-sig"`, which strips a leading BOM if present.

### Binary mode

Bytes are unambiguous. `open("data.bin", "rb")` returns `bytes`, no encoding involved. If you are dealing with structured binary or with data of unknown encoding, open in binary mode and decode explicitly.

### Line endings

Every platform has made its own peace with line endings: `\n` on Unix, `\r\n` on Windows, `\r` on old Macs. Text-mode I/O in Python 3 translates by default (reads: `\r\n` → `\n`; writes: `\n` → platform's line separator). Pass `newline=""` to disable translation.

This matters for *encoding* because CSV files and other line-based formats are sensitive to line terminators: a Windows-authored CSV read in text mode on Unix will have a trailing `\r` on every field if you don't use the `csv` module or specify `newline=""`.

## Environment and locale

`LANG`, `LC_ALL`, `LC_CTYPE`, and friends are Unix environment variables that together determine the *locale* — a triple of (language, territory, codeset). `LC_CTYPE=en_US.UTF-8` says "American English, UTF-8 codeset."

Programs that do any locale-sensitive operation consult these. The catch: *which* locale variable wins depends on the program and the operation. `LC_ALL` overrides everything. `LC_CTYPE` governs character classification and conversion. `LANG` is the default for anything unset.

If your terminal is showing `?` or mojibake:

1. Check `LANG` and `LC_ALL`. If they are empty or `C` or `POSIX`, you are in an 8-bit-no-Unicode-locale.
2. Set `LANG=en_US.UTF-8` (or your preferred locale). Run `locale -a` to see what's installed.
3. Make sure your terminal emulator is configured for UTF-8.

Python before 3.15 honored these via `locale.getpreferredencoding()`. Go and Rust essentially ignore them for program logic (they use UTF-8 internally regardless) but rely on the terminal's interpretation for output.

## Databases

### MySQL and the `utf8mb4` footgun

The most famous database Unicode gotcha: MySQL's `utf8` collation is not UTF-8. It is a three-byte-maximum subset of UTF-8 that does not support code points above U+FFFF. So it handles the BMP but *fails on emoji* (all of which are supplementary code points) and on supplementary CJK.

The correct encoding is called `utf8mb4` ("UTF-8, maximum 4 bytes per character") and it is what you want. MySQL 8.0 finally changed the default character set to `utf8mb4`; earlier versions defaulted to `utf8`.

If you have a pre-8.0 MySQL database, migrate to `utf8mb4` at the database level, the table level, the column level, and the connection level. All four. Missing one causes silent truncation on emoji.

### PostgreSQL

PostgreSQL supports UTF-8 as a first-class encoding via the `UTF8` (or `SQL_ASCII`, which is really "no encoding enforcement") server encoding. Set this at database creation time; it cannot be changed afterward.

PostgreSQL's `text`, `varchar`, `char` types all store the same bytes; the difference is only length enforcement. No `nvarchar` needed.

Collation is per-column (or per-database). Choose `C` collation for ASCII-byte-order, a locale like `en_US.UTF-8` for locale-aware ordering, or `"und-x-icu"` (or any ICU locale) for true UCA behavior if your build has `--with-icu`.

### SQL Server

SQL Server's native `varchar` is single-byte (a codepage determined by the database's collation); `nvarchar` is UCS-2 (not UTF-16 — meaning supplementary code points are stored as surrogate pairs but not handled as single characters by all functions).

Since SQL Server 2019, `varchar` columns can have a UTF-8 collation (`Latin1_General_100_CI_AS_SC_UTF8`), which stores data as UTF-8 bytes. `nvarchar` with a collation ending in `_SC` ("supplementary character") handles supplementary code points correctly.

The *TL;DR* for SQL Server in 2026: use `varchar` with a UTF-8 collation for new databases.

### SQLite

SQLite is UTF-8 by default and has been for a long time. `text` is UTF-8. There are also UTF-16 variants of some APIs, which nobody uses.

## Email (MIME)

Email is the oldest Unicode-bearing protocol, and it shows. MIME allows any encoding in message bodies via `Content-Transfer-Encoding: base64` or `quoted-printable` with a `Content-Type: text/plain; charset=utf-8` header. But *email headers* (From, To, Subject) are ASCII-by-default and use RFC 2047 "encoded-word" syntax for non-ASCII content:

```
Subject: =?UTF-8?B?SGVsbG8g8J+RiyE=?=
```

That is "Hello 👋!" in UTF-8, base64-encoded, in an email header. Parsers are expected to decode this transparently.

Internationalized email addresses (IDN in the local part and in the domain) are supported via the *SMTPUTF8* extension (RFC 6531), but support is uneven, so most email infrastructure still converts domains to *Punycode* (ASCII-safe mojibake for the transport).

## URLs and IDNs

URLs are defined by RFC 3986 to contain only ASCII characters. Non-ASCII text in a URL must be percent-encoded after first being encoded as UTF-8:

```
https://example.com/résumé
→ https://example.com/r%C3%A9sum%C3%A9
```

Domain names (the host portion) are different: they use *IDN* (Internationalized Domain Names) with *Punycode* encoding:

```
http://café.com
→ http://xn--caf-dma.com   (IDN, ASCII-Compatible Encoding)
```

We will return to IDN in Chapter 10, because it is a rich vein of security problems.

## The consistent rule

At every boundary where bytes cross into or out of your program, know the encoding. Write it down. Put it in the `Content-Type`, the `<meta charset>`, the `open(..., encoding="utf-8")`, the database connection string, the MIME header.

The bug is almost always somebody — a framework, a legacy tool, a junior developer — who thought the encoding "would be obvious." It isn't, ever. It is metadata, and metadata has to be written down.

Next: the security side of Unicode. Characters that look like other characters, and what to do about them.

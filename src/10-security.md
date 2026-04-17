# Unicode Security

Unicode is a universal character set, which means it contains every character that looks like every other character — and, in the hands of an attacker, that is a security issue.

This chapter is organized around the techniques: homograph attacks, Trojan Source, normalization mismatches, and invisible characters. For each, we cover the mechanism and the mitigations. The authoritative reference is *UTS #39: Unicode Security Mechanisms*.

## Homograph attacks

A *homograph* is a character that looks identical (or indistinguishable) to another character but is a distinct code point. The Cyrillic lowercase `а` (U+0430) and the Latin lowercase `a` (U+0061) render identically in nearly every font. A reader cannot tell them apart. But as bytes, as code points, as URL characters, they are distinct.

An attacker who registers `раypal.com` — spelled with a Cyrillic `р` and a Cyrillic `а` — and sets up a fake login page has an asset that looks to a user like `paypal.com` but points to an attacker-controlled host. This is the classic *IDN homograph attack*. (IDN stands for Internationalized Domain Name, the mechanism that allows non-ASCII characters in URLs.)

### Mitigations in browsers

Modern browsers deploy *IDN display policies* to reduce this risk:

- Chrome's policy combines whole-script confusables detection, script-mixing detection, and a large blocklist of known confusable patterns. A domain that triggers the policy is displayed in Punycode (the ASCII-safe encoding) in the address bar: `xn--ypal-uye.com` instead of `раypal.com`.
- Firefox has a similar policy, configurable via `network.IDN_show_punycode` and related preferences.
- Safari has its own heuristics, generally aggressive.

The policies are not identical across browsers, and an attacker who finds a gap in one can sometimes exploit it. Mixing-script detection (a URL whose host contains both Latin and Cyrillic characters) catches the most common attacks but not single-script attacks where the attacker uses a wholly non-Latin script that contains visual look-alikes of Latin letters.

### UTS #39 "confusables"

Unicode publishes a *confusables.txt* file that lists known visual confusables. You can use it to build your own checks: "does this username, when reduced to its confusable skeleton, collide with an existing user?" The *skeleton* is a deterministic reduction: map every character to its "representative" form, so that `а` (Cyrillic) and `a` (Latin) both map to the same skeleton character.

```python
# Using the pyicu binding or the third-party `confusables` package.
import confusables

confusables.is_confusable("раypal", "paypal")       # True
confusables.skeleton("раypal") == confusables.skeleton("paypal")   # True
```

For any identifier your system treats as unique to a user (username, project name, organization name), *always* check for confusable collisions with existing values.

### Restricted identifier profiles

UTS #39 defines *restriction levels* for identifiers:

- *ASCII-Only*: only ASCII characters.
- *Single Script*: only characters from one script.
- *Highly Restrictive*: single script, or one of a small set of common script combinations (Latin+Han+Hiragana+Katakana for Japanese, Latin+Han for traditional Chinese, etc.).
- *Moderately Restrictive*: same as above, plus Latin added to any single non-Latin script.
- *Minimally Restrictive*: allows broader mixing with some exclusions.
- *Unrestricted*: anything.

The right level for usernames is *Moderately Restrictive* at most. Anything below that permits mixing arbitrary scripts, and that is where homograph attacks come from.

## Trojan Source

In 2021, security researchers Nicholas Boucher and Ross Anderson published *Trojan Source*: a family of attacks that use Unicode's bidirectional override characters to make source code look different to a human reader than it does to a compiler.

### The mechanism

Unicode has *bidirectional formatting* control characters that affect how text is rendered, without changing what the text actually is:

- U+202A LEFT-TO-RIGHT EMBEDDING (LRE)
- U+202B RIGHT-TO-LEFT EMBEDDING (RLE)
- U+202D LEFT-TO-RIGHT OVERRIDE (LRO)
- U+202E RIGHT-TO-LEFT OVERRIDE (RLO)
- U+2066 LEFT-TO-RIGHT ISOLATE (LRI)
- U+2067 RIGHT-TO-LEFT ISOLATE (RLI)
- U+2068 FIRST STRONG ISOLATE (FSI)
- U+202C POP DIRECTIONAL FORMATTING
- U+2069 POP DIRECTIONAL ISOLATE

These exist for a legitimate reason: Arabic and Hebrew are right-to-left scripts, and mixed-direction text requires a grammar for how directionality flows. Unicode's *Bidirectional Algorithm* (UBA, UAX #9) uses these overrides to specify exceptions to the default.

When an attacker places these overrides inside source code — in a comment, in a string literal — they can cause the code to *render* in a deceptive order while being *parsed* in its actual order. Consider this C snippet:

```c
access_level = "user";
if (access_level != "user‮ ⁦// Check if admin⁩ ⁦") {
    // grant admin
}
```

What you see on screen is roughly "if (access_level != "user" // Check if admin) { grant admin }" — looks like a comment, and the string comparison reads as "user". But the bytes the compiler sees contain the full string `"user <RLO> <LRI>// Check if admin<PDI> <LRI>"`, and the actual comparison is against a much weirder string. The visible and the parsed interpretations diverge.

This is Trojan Source. The attack works against any language that allows these characters in comments or string literals — which is almost all of them.

### Mitigations

- *Lint rules*: modern linters flag bidirectional controls in source code. Run `rg --pcre2 '[\u202A-\u202E\u2066-\u2069]'` over a codebase to find them. Rust's compiler emits a warning since 1.56. GCC has `-Wbidi-chars`. Git since 2.35 warns when bidi controls appear in diffs.
- *Editor rendering*: VS Code and most modern editors display a warning marker for files containing bidi controls in source text.
- *Code review*: be suspicious of large innocuous-looking diffs that come from unfamiliar contributors. Trojan Source is not common in normal attacker traffic, but the technique is well-documented, and a curious reviewer can save their team.

## Invisible and zero-width characters

Unicode has a number of characters that are, or render as, nothing:

- U+200B ZERO WIDTH SPACE
- U+200C ZERO WIDTH NON-JOINER
- U+200D ZERO WIDTH JOINER
- U+2060 WORD JOINER
- U+FEFF ZERO WIDTH NO-BREAK SPACE (also the BOM; there is no way to tell which role it plays from the bytes alone)
- U+00AD SOFT HYPHEN
- U+2028 LINE SEPARATOR
- U+2029 PARAGRAPH SEPARATOR
- U+E0000 – U+E007F (tag characters, covered in Chapter 14)

If you allow these in identifiers, you allow usernames that look identical to existing ones. `admin` and `admin\u200b` render the same and collide in many UIs but differ byte-for-byte. An account with the latter name can log in; to a moderator scrolling a table, it looks like the former.

*Mitigation*: normalize inputs (NFKC or similar), strip Default_Ignorable_Code_Point characters, apply a restriction level, and case-fold.

## Normalization mismatches

If part of your system normalizes a string and another part doesn't, attackers can exploit the difference.

Classic example: a URL normalizer maps `example.com/café` to `example.com/café` (NFC). An authentication filter looks for the literal path `/café`, sees the user's non-normalized version (`cafe` + combining acute), says "this isn't the protected path," and waves them through. The application then receives the normalized path and serves the protected resource. *Normalization mismatch*.

### Where to watch for this

- URL path handling vs. ACL matching.
- Filename comparison in security-sensitive code.
- Username comparison vs. username storage.
- Header name matching (HTTP is usually ASCII, but email headers can contain non-ASCII).

### The rule

Pick one normalization form. Apply it at the boundary. Store, index, and compare in that form. Never compare an unnormalized input against a normalized stored value.

## Punycode, IDNs, and the `xn--` story

A *Punycode* string is the ASCII-compatible encoding of an IDN label. Domain names in DNS are restricted to ASCII plus hyphens and digits, so a non-ASCII label like `café` is encoded as `xn--caf-dma`, where `xn--` is the IDN prefix and `caf-dma` is a Punycode-encoded representation.

Important for security:

- *IDNA 2008* (the current IDN standard, RFC 5890-5894) specifies which code points are permitted in IDN labels. The set is restrictive and designed to reduce homograph risk.
- Browsers convert IDNs back to Unicode for display *only* if the domain meets the display policy. Otherwise they show the Punycode form.
- Applications that generate or parse URLs should use a proper IDN library (Python's `idna` package, for instance — *not* the built-in `encodings.idna`, which implements the older, less safe IDNA 2003).

## Defensive code patterns

### A safe username validator

```python
import unicodedata as ud
import re

# Allow letters and digits from a restricted set of scripts plus underscore.
ALLOWED = re.compile(r"^[\p{L}\p{N}_]{3,20}$")

def safe_username(name: str) -> str | None:
    n = ud.normalize("NFKC", name)
    if any(0x200B <= ord(c) <= 0x200F for c in n):  # zero-width
        return None
    if any(ud.category(c) == "Cf" for c in n):      # format controls
        return None
    if any(0x202A <= ord(c) <= 0x202E for c in n):  # bidi overrides
        return None
    if not ALLOWED.match(n):
        return None
    # Enforce moderately-restrictive script profile (example; production code
    # should use UTS #39 data via ICU or the confusables package).
    return n
```

This rejects bidi overrides, zero-width characters, format controls, and anything outside a basic letter/digit/underscore set after NFKC normalization. A production implementation would also compute the confusable skeleton and check it against existing usernames.

### Displaying untrusted text

- Strip or replace bidirectional control characters before displaying untrusted text in a terminal or web UI.
- Render zero-width characters visibly (many editors do this; `​` → `[ZWSP]`).
- When displaying a URL, display its Punycode form if the IDN display policy fails.

## What this chapter is not

We have not covered:

- Buffer overflows from miscalculated character counts (Chapter 7 is the prevention).
- Injection attacks (SQL, HTML, shell) where Unicode can sometimes bypass filters — the mitigation is always "filter after decoding, decode to a canonical form, and use parameterized queries / escape libraries," not regex-on-bytes.
- SSL certificate impersonation using IDNs, which is largely a certificate authority policy problem.

With the security tour done, the next chapter goes back to something much more fun: emoji.

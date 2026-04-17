# Normalization

At the end of the previous chapter, we met a problem. The letter `é` can be encoded as one code point (`U+00E9`) or two (`U+0065 U+0301`). Visually identical; byte-sequentially distinct. If you naively compare two strings for equality, you will sometimes say they are different when a human would say they are the same.

*Normalization* is Unicode's answer: a deterministic procedure that rewrites a string into one of four canonical forms, so that any two strings representing the same text end up with the same bytes.

## The four forms

Unicode defines four normalization forms, from *UAX #15*. Their names are almost self-explanatory once you know the two axes they vary on:

|           | Composed | Decomposed |
|-----------|----------|------------|
| Canonical | **NFC**  | **NFD**    |
| Compatibility | **NFKC** | **NFKD** |

The two axes:

- *Canonical* vs. *Compatibility*: how aggressively to treat two code points as "the same."
- *Composed* vs. *Decomposed*: whether to prefer single precomposed code points or base+combining-marks sequences.

You will meet all four in your career. Here is when each matters.

## Canonical equivalence: NFC and NFD

Two code-point sequences are *canonically equivalent* if they represent the same abstract character. `U+00E9` (precomposed `é`) and `U+0065 U+0301` (`e` + combining acute) are canonically equivalent by definition.

*NFC* (Normalization Form C) converts a string to its canonically composed form: wherever a base + combining sequence has a precomposed equivalent, use the precomposed one.

*NFD* (Normalization Form D) does the opposite: decomposes every precomposed character into its base + combining sequence.

NFC(NFC(s)) = NFC(s); NFD(NFD(s)) = NFD(s); NFC(NFD(s)) = NFC(s). Both forms are stable.

```python
>>> import unicodedata as ud
>>> a = "café"             # 4 code points, precomposed
>>> b = "cafe\u0301"       # 5 code points, decomposed
>>> a == b
False
>>> ud.normalize("NFC", a) == ud.normalize("NFC", b)
True
>>> ud.normalize("NFD", a) == ud.normalize("NFD", b)
True
```

You will often see NFC described as "the default" because it is what most text on the Web uses, what most users type, and what most fonts render most efficiently. When in doubt, normalize to NFC.

## Compatibility equivalence: NFKC and NFKD

Canonical equivalence preserves meaning exactly. *Compatibility equivalence* is looser: it will also fold together characters that have the same "underlying identity" but differ in formatting or presentation. For example:

- `ﬁ` (U+FB01, LATIN SMALL LIGATURE FI) is compatibility-equivalent to `fi` (U+0066 U+0069).
- `²` (U+00B2, SUPERSCRIPT TWO) is compatibility-equivalent to `2` (U+0032).
- `①` (U+2460, CIRCLED DIGIT ONE) is compatibility-equivalent to `1`.
- Half-width and full-width Latin letters fold together: `ＡＢＣ` → `ABC`.
- Some Arabic presentation forms fold to their base letters.

NFKC and NFKD apply these folds *in addition to* the canonical folds of NFC and NFD respectively. The "K" stands for *Kompatibility* — spelled with a K to distinguish it from C (which already meant Composed).

```python
>>> ud.normalize("NFC", "ﬁ")
'ﬁ'
>>> ud.normalize("NFKC", "ﬁ")
'fi'
```

NFKC and NFKD are *lossy* transformations: the formatting distinctions they erase are not recoverable. You should not normalize your data to NFKC and then store it unless you are sure you never wanted to preserve those distinctions. That said, NFKC is enormously useful for *search* and for *identifier comparison*, which we will cover in Chapter 12.

## When to normalize

The practical rule: normalize at *input boundaries*, not at every comparison.

- When text enters your system — form submission, API payload, database import, file read — decide on a normalization form and apply it once.
- Store normalized data.
- Then comparison, searching, and indexing can use fast byte-level operations.

The alternative — storing text unnormalized and normalizing on every comparison — is correct but slow, and invites subtle bugs where some comparison paths normalize and others don't.

Choose NFC for most user text. Choose NFD if you are doing linguistic analysis that cares about individual combining marks. Choose NFKC for case-insensitive searching, for username normalization, and for user-visible identifier comparison. Never choose NFKC if you need to preserve the distinction between, say, `ﬁ` and `fi` in stored user content.

## Canonical ordering

Here is a detail that bites in practice. Suppose a single base character has multiple combining marks:

```
a + acute + cedilla       U+0061 U+0301 U+0327
a + cedilla + acute       U+0061 U+0327 U+0301
```

Both render identically (both say "a with acute and cedilla"). Are they canonically equivalent?

Yes — but only because normalization *reorders* them. Unicode assigns each combining mark a *Combining Class* (CCC): a positive integer for non-spacing marks, zero for base characters and spacing marks. Normalization to NFC or NFD reorders combining marks so that marks with a lower combining class come first.

This means that after normalization, there is a single canonical ordering for any stack of combining marks, and byte equality corresponds to meaning equality.

## Filesystems and normalization: a real-world mess

Filesystems have had to answer the question "are these two filenames the same?" and they have given different answers.

### macOS / HFS+ / APFS

Historical HFS+ normalized filenames to a variant of *NFD* on write. APFS, which replaced HFS+ in 2017, is *normalization-insensitive* by default: it accepts filenames in any form, stores them as-given, but compares them case- and normalization-insensitively. On macOS, you can create a file named `café` (NFC) and open it as `café` (NFD), and the OS will treat them as the same file. The actual bytes stored in the directory depend on the filesystem version and the creator.

### Linux / ext4 / btrfs / xfs

Standard Linux filesystems are *byte-literal*: a filename is a sequence of bytes, end of story. Two filenames that differ only in normalization form are *different filenames*. You can have `café` and `café` in the same directory and the OS is happy about it.

This causes real problems when a macOS user sends a git repo to a Linux user and the two end up with filename variants that conflict.

### Windows / NTFS

NTFS preserves the filename's original case and normalization. Comparison is case-insensitive by default (as on Windows), but the comparison is otherwise byte-literal at the normalization level. Two files differing only by normalization form can coexist, though most Windows tools treat this as a surprise.

The portable rule: *pick one normalization form for your project (NFC is the best default) and stick to it*. If your build system cares about cross-platform filename compatibility, normalize on commit.

## The `\r\n` footnote

One last canonical-equivalence footnote. Normalization does *not* touch line endings. `\r\n`, `\n`, and `\r` remain distinct after normalization. If you need line endings normalized, do that separately. If you need BOMs normalized (stripped), do that separately too.

## Worked example: a search bar that finds `café` when the user types `cafe`

This is where the pieces come together.

```python
import unicodedata as ud

def normalize_for_search(s):
    # Decompose, drop combining marks, then lowercase.
    s = ud.normalize("NFD", s)
    s = "".join(ch for ch in s if not ud.combining(ch))
    s = s.casefold()
    return s

normalize_for_search("café")   # 'cafe'
normalize_for_search("CAFÉ")   # 'cafe'
normalize_for_search("cafe")   # 'cafe'
```

What we did:

- Decomposed: `é` → `e` + acute.
- Removed all combining marks (`ud.combining` returns non-zero for combining code points).
- Case-folded (more on that in Chapter 6; it is *not* the same as lowercasing).

This is a classic "fuzzy match" for user-facing search. It throws away accents, which is desirable in Chrome's address bar and disastrous for a dictionary that distinguishes `resume` from `résumé`. Know which one you want.

In the next chapter, we tackle the sibling topic: once strings are canonically comparable, how do we *sort* them? The answer is more than you might expect.

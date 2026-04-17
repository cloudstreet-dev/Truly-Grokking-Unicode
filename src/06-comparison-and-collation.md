# Comparison and Collation

There are two kinds of string comparison and they are almost never interchangeable.

- *Byte comparison* asks: are these two sequences of bytes identical? It is fast, deterministic, transitive, and useless for most user-facing work.
- *Collation* asks: in a human-readable sort order — specifically, the order a fluent reader of this language expects — which string comes first? It is slow, locale-dependent, full of exceptions, and absolutely necessary for user-facing lists.

This chapter covers both, plus case folding, which is subtler than it looks.

## Byte comparison and its pitfalls

If you want two strings to be "the same," normalize them (Chapter 5), encode them to the same encoding, and compare the bytes. This answers *are these bytes bit-for-bit the same?*. Good for primary keys, cache keys, anything mechanical.

It is a bad answer to *does the user think these are the same?* A user typing "CAFÉ" and searching for "café" expects a hit. A user visiting `/User/23` expects it to match `/user/23`. A user sorting `["Äpfel", "Apfel"]` in German will be unhappy with the byte order.

Byte comparison is a useful primitive, but it is not what "compare two strings" usually means in an interface.

## Case folding vs. lowercase

You might be about to say, "I will just `s.lower()` everything and compare." That will mostly work. Here is why it is not quite right.

*Lowercase* is a transformation that maps a string to its lowercase form, intending to produce text that looks correct when displayed. It is a *display* operation. It is locale-sensitive. In Turkish, lowercasing `I` (U+0049) yields `ı` (U+0131, dotless i), not `i`. `İ` (U+0130, capital I with dot above) lowercases to `i` (U+0069). Turkish distinguishes the dotted and dotless i, and its casing rules differ from English.

*Case folding* is a transformation that maps a string to a comparison-safe form. Its output may not be "correct lowercase"; it is merely a form where two strings that should compare equal under case-insensitive comparison produce the same bytes. It is locale-*insensitive* by default (you can ask for a Turkic-specific variant explicitly).

```python
>>> "ß".lower()
'ß'
>>> "ß".casefold()
'ss'
```

German `ß` (the *eszett*) uppercases to `SS` but historically had no lowercase for `SS`. Case folding turns `ß` into `ss` so that `"Straße"` and `"STRASSE"` compare equal when you case-fold both. Plain lowercasing doesn't do this.

The rule of thumb: for case-insensitive *comparison*, use `casefold()` (Python), `toLocaleLowerCase` + caution (JavaScript), or a collator with `sensitivity: "base"` (better). For case-changed *display*, use `lower()` / `toLowerCase()`, and pass a locale if you have one.

## Collation

*Collation* is the process of producing an order over strings that matches human expectations. It depends on a *locale*: the same sort call, run against the same list, can produce different results for a Swedish user and a German user — and that is correct.

### Why sort order differs by locale

In *Swedish*, the letters `å`, `ä`, and `ö` come at the end of the alphabet, after `z`. In *German*, `ä`, `ö`, `ü` are treated as variants of `a`, `o`, `u` and intercalate among them. In *Spanish* (older conventions), `ll` and `ñ` used to be their own letters; modern Spanish treats `ll` as two characters but `ñ` is still a separate letter after `n`.

So the list `["Apfel", "Äpfel", "Zebra"]` sorts as:

- German (phonebook variant): `Apfel`, `Äpfel`, `Zebra` — `ä` right next to `a`.
- Swedish: `Apfel`, `Zebra`, `Äpfel` — `ä` after `z`.
- Byte order (UTF-8 or code point): `Apfel`, `Zebra`, `Äpfel` — coincidentally matches Swedish here, but only because `Ä` (U+00C4) has a higher code point than `Z`. It is not a general truth.

You cannot sort a mixed list of names "correctly" in a single global order. You can only sort correctly *for a given locale*, and the locale has to come from somewhere — user preference, document language, URL parameter.

### The Unicode Collation Algorithm

*UCA* (Unicode Collation Algorithm, UAX #10) is the framework. It assigns each code point a sequence of *collation weights*, typically at three levels:

- *Primary* (letter identity: A vs. B).
- *Secondary* (diacritics: A vs. Á).
- *Tertiary* (case: A vs. a).

To compare two strings, the algorithm compares their level-1 weights first; if equal, their level-2 weights; if equal, their level-3. This naturally expresses "A and Á are the same letter but the accented one sorts after the unaccented one, and lowercase sorts after uppercase at the last tiebreak."

The default weights come from the *Default Unicode Collation Element Table* (DUCET). To get a locale-specific order, you overlay that locale's *tailoring*: a set of changes to the weights that encodes the conventions of that language. The *Common Locale Data Repository* (CLDR) publishes tailorings for hundreds of locales; every serious Unicode library ships a copy.

You almost never implement UCA yourself. You call:

- Python: `pyuca` package or `icu.Collator` from PyICU.
- JavaScript: `Intl.Collator`. Built in.
- Java: `java.text.Collator`.
- Go: `golang.org/x/text/collate`.
- Rust: the `icu` crate.
- Swift: `String.localizedStandardCompare(_:)`.
- C/C++: ICU.

### Intl.Collator, a practical tour

`Intl.Collator` is the single most ergonomic UCA implementation on any platform. If you have a browser or Node, you have it.

```javascript
const en = new Intl.Collator("en");
const de = new Intl.Collator("de");
const sv = new Intl.Collator("sv");

const names = ["Zebra", "Äpfel", "Apfel"];

names.slice().sort(en.compare);   // ["Apfel", "Äpfel", "Zebra"]
names.slice().sort(de.compare);   // ["Apfel", "Äpfel", "Zebra"]  — same in German
names.slice().sort(sv.compare);   // ["Apfel", "Zebra", "Äpfel"]  — Swedish puts ä after z

// Case-insensitive, accent-insensitive:
const base = new Intl.Collator("en", { sensitivity: "base" });
base.compare("Café", "cafe");     // 0 — equal at the primary level

// Natural numeric sort:
const nat = new Intl.Collator("en", { numeric: true });
["file10", "file2", "file1"].sort(nat.compare);  // ["file1", "file2", "file10"]
```

Memorize this one: for user-facing sorts in JavaScript, `Intl.Collator` is almost always the right answer. Do not write your own comparator.

### Case folding vs. collation level

Earlier we said case folding is for comparison. That is true, and collation gives you a different, finer-grained way to do the same thing: set the *sensitivity* to `base` (primary-level only) or `accent` (primary and secondary). For equality checks — "does the user's input match this list entry, ignoring case and accents?" — a `base`-sensitivity `Intl.Collator.compare(a, b) === 0` is generally the best answer, because it uses the locale's rules rather than a locale-free case-fold.

## Sorting a list of user names correctly

Here is the pragmatic recipe.

1. Determine the locale. Per-user preference if you have it; document language otherwise; fall back to `und` (undetermined) if nothing.
2. Use a proper collator (`Intl.Collator`, `java.text.Collator`, etc.).
3. Pass options you care about: numeric sort, case sensitivity, accent sensitivity.
4. Let the collator sort.

```javascript
function sortUsers(users, locale = "en") {
  const coll = new Intl.Collator(locale, {
    sensitivity: "base",
    numeric: true,
    usage: "sort",
  });
  return users.slice().sort((a, b) => coll.compare(a.name, b.name));
}
```

What *not* to do: compare normalized-lowercased strings with `<` / `>`. That will mostly work for ASCII names and fail in interesting ways on the 200th user whose name starts with `Ñ`.

## Stable and deterministic sorts

One last note. UCA's default behavior is that strings with the same primary weight are "equal" at that level; tie-breaking may descend to secondary, tertiary, and sometimes quaternary weights, and after that may fall back to byte order. This means UCA-sorted lists are not *quite* a strict total order in all cases; different implementations may disagree about exactly how ties break.

Most real sort algorithms today are *stable* (they preserve the order of equal elements), which is what you want: if two items are equal at the collation level you care about, their original order is preserved. JavaScript's `Array.prototype.sort` has been required to be stable since ES2019. Python's `sorted` and `list.sort` have always been stable. Java's `Collections.sort` is stable. Go's `sort.SliceStable` is stable, but `sort.Slice` is not.

With comparison and sorting in hand, we are ready to look at how real languages — each with its own internal string model — actually expose all of this to you.

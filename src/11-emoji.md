# Emoji, Properly

Emoji are the most visible success story of Unicode, and the most concentrated source of Unicode-related confusion in practice. A single emoji can involve multiple code points, a variation selector, a modifier, a font lookup, and a fallback cascade — all to render one tiny picture.

This chapter takes them seriously. If you understand how emoji work in Unicode, you understand most of Unicode's advanced text machinery.

## How emoji got into Unicode

*Emoji* (絵文字, "picture characters") originated in Japan in the 1990s. Japanese mobile carriers — DoCoMo, KDDI, SoftBank — each had their own proprietary encoding for a few hundred pictographs used in messages. When iPhone launched in Japan in 2008, Apple had to interoperate with these encodings, and Google followed for Android.

In 2010, with Unicode 6.0, emoji entered the standard. A core set of about 700 emoji was assigned code points, most of them in the new *Supplementary Multilingual Plane* (U+1F000 and up) precisely because the Basic Multilingual Plane was running out of room.

This was controversial. Some Unicode Consortium members argued that pictographs were not *text* in the sense that letters and numbers are text, and didn't belong in a character set. The counterargument was that people were already using these characters in text on their phones, and the choice was whether to standardize or watch proprietary encodings proliferate. Standardization won, and Unicode gained — and continues to gain — hundreds of emoji per year.

## Variation selectors

Some symbols can be rendered as either a *text* glyph (monochrome, aligned to the baseline) or an *emoji* glyph (colored, possibly more stylized). The default depends on the platform and the symbol.

Unicode defines two *variation selectors* that force one or the other:

- U+FE0E VARIATION SELECTOR-15 (VS15): forces *text* presentation.
- U+FE0F VARIATION SELECTOR-16 (VS16): forces *emoji* presentation.

These are invisible code points; they attach to the preceding character and change how that character is rendered.

```
U+2764 HEAVY BLACK HEART: ❤
U+2764 U+FE0E: ❤︎   (text presentation)
U+2764 U+FE0F: ❤️   (emoji presentation)
```

In practice, you mostly see VS16 in the wild, appended to symbols that Unicode considers borderline (text-by-default but often wanted as emoji). You will sometimes copy a "heart emoji" from a web page and get U+2764 U+FE0F — two code points — and wonder why your length count is off. Now you know.

## Fitzpatrick modifiers (skin tone)

In 2015, Unicode 8.0 added five *emoji modifiers* corresponding to skin tones from the *Fitzpatrick scale*, a dermatological classification:

- U+1F3FB EMOJI MODIFIER FITZPATRICK TYPE-1-2 (light skin tone)
- U+1F3FC EMOJI MODIFIER FITZPATRICK TYPE-3 (medium-light skin tone)
- U+1F3FD EMOJI MODIFIER FITZPATRICK TYPE-4 (medium skin tone)
- U+1F3FE EMOJI MODIFIER FITZPATRICK TYPE-5 (medium-dark skin tone)
- U+1F3FF EMOJI MODIFIER FITZPATRICK TYPE-6 (dark skin tone)

An emoji that supports skin tone (not all do) can be followed by a modifier to produce a tinted version. The base emoji *without* a modifier is conceptually the "default" (typically yellow in most font designs, specifically to avoid implying a particular skin tone).

```
👋         U+1F44B
👋🏻         U+1F44B U+1F3FB   (light skin)
👋🏼         U+1F44B U+1F3FC
👋🏽         U+1F44B U+1F3FD
👋🏾         U+1F44B U+1F3FE
👋🏿         U+1F44B U+1F3FF   (dark skin)
```

A modifier applied to an emoji that doesn't support it will render as the base emoji followed by a colored square, or as just the base emoji with the modifier ignored, depending on the font.

## Zero-Width Joiner sequences

The real combinatorial explosion lives in *ZWJ sequences*. The *zero-width joiner* (U+200D, ZWJ) joins adjacent emoji into a ligature if the font defines one.

Consider the family emoji `👨‍👩‍👧‍👦`. Its code points:

```
U+1F468 MAN
U+200D  ZWJ
U+1F469 WOMAN
U+200D  ZWJ
U+1F467 GIRL
U+200D  ZWJ
U+1F466 BOY
```

A font that "knows" this sequence renders it as a single family glyph. A font that doesn't renders it as `👨 👩 👧 👦` — the individual emoji, possibly with visible gaps where the ZWJ is.

Other common ZWJ sequences:

- Profession: `👩‍⚕️` = WOMAN + ZWJ + U+2695 (MEDICAL SYMBOL) + VS16 = "woman health worker."
- Relationships: `👩‍❤️‍👩` = WOMAN + ZWJ + HEAVY BLACK HEART + VS16 + ZWJ + WOMAN.
- Hair color: `👨‍🦰` = MAN + ZWJ + U+1F9B0 (EMOJI COMPONENT RED HAIR).
- Flag of Scotland: `🏴󠁧󠁢󠁳󠁣󠁴󠁿` (uses tag characters, Chapter 14).

Each of these is *one grapheme cluster*, rendered as one glyph (if supported), spanning many code points.

## Regional indicator flags

Flag emoji for sovereign countries work differently from everything above, and have a uniquely elegant design.

Unicode did not assign one code point per country flag. That would have been inflexible and politically fraught. Instead, Unicode defined 26 *regional indicator symbols* — U+1F1E6 (🇦) through U+1F1FF (🇿) — corresponding to the 26 Latin letters. A pair of consecutive regional indicators that spells an ISO 3166-1 alpha-2 country code renders as that country's flag.

```
🇺 + 🇸  (U+1F1FA + U+1F1F8)  → 🇺🇸
🇯 + 🇵  (U+1F1EF + U+1F1F5)  → 🇯🇵
🇪 + 🇺  (U+1F1EA + U+1F1FA)  → 🇪🇺
```

Each flag is one grapheme cluster, two code points.

This design means Unicode doesn't have to take a position on "is this a country?" for every disputed territory — it just provides the mechanism, and fonts decide what to render. For codes that don't correspond to a recognized country, the fonts typically render the letters as individual regional indicators without flag treatment.

Subnational flags (Scotland, Wales, England) use a different mechanism: *tag characters* (Chapter 14).

## Rendering is the font's job

A critical clarification: Unicode defines *what the code points are*. It does not define *what they look like*. Every emoji on your screen is a font glyph, chosen by your operating system or browser.

This is why the "same" emoji looks so different across platforms:

- Apple's 🦒 is drawn one way; Google's is drawn another; Microsoft's is different again.
- The pistol emoji (U+1F52B) was redesigned by most vendors around 2017 from a realistic handgun to a water pistol. The code point did not change; the glyphs did.
- Some vendors have experimented with not including certain emoji in their fonts (Facebook's Messenger initially omitted some characters); the code point exists and is interchanged, the rendered image is platform-dependent.

If your user pastes an emoji that your font doesn't know, it will render as a "tofu" — a box — or as a fallback symbol. The bytes are still correct; the font is incomplete. This is why emoji interop problems are usually font problems, not encoding problems.

### The consequence for layout

Because emoji are font glyphs, they have a *width* only at render time. In terminals — which assume monospace with a fixed column width — emoji can be 1 column wide, 2 columns wide, or "does your terminal handle this?" wide. The Unicode *East Asian Width* property (UAX #11) gives a hint, but it is a hint. Robust CLI tools that display emoji must query the terminal.

## Counting emoji correctly

You know this by now. If you need the "tweet length" of text that contains emoji, count *grapheme clusters*, not code points. A skin-tone family emoji (MAN-LIGHT + ZWJ + WOMAN-MEDIUM + ZWJ + BOY-DARK) might be 18 code points and 1 grapheme cluster, or 1 user-perceived "character."

```javascript
const seg = new Intl.Segmenter();
const msg = "Celebrating 🎉👨🏻‍👩🏾‍👧🏼";
[...seg.segment(msg)].length;    // the answer a user expects
```

## Detecting emoji

Unicode derives an *Emoji* property (and several related properties: `Emoji_Presentation`, `Emoji_Modifier`, `Emoji_Modifier_Base`, `Emoji_Component`, `Extended_Pictographic`). You can use these in regex:

```javascript
/\p{Emoji}/u.test("hello");    // false
/\p{Emoji}/u.test("🎉");        // true
/\p{Emoji}/u.test("1");         // true — digits are Emoji with VS15 variation
```

Note that surprising result: the digit `1` has the `Emoji` property set, because `1️⃣` (keycap one) is a valid emoji sequence. If you just want "pictograph-like emoji" — not digits, not `#`, not `*` — use `\p{Extended_Pictographic}`.

For production emoji detection, use a well-maintained library (`emoji-regex` in JavaScript, the `emoji` package in Python). Unicode adds emoji every year, and a hard-coded regex goes stale.

## A worked example: counting a tweet

Here is what a real "tweet length" counter should do.

```javascript
function tweetLength(text) {
  const seg = new Intl.Segmenter();
  return [...seg.segment(text)].length;
}

tweetLength("Hello");                    // 5
tweetLength("Hello 👋");                  // 7
tweetLength("Hello 👋🏽");                  // 7 (same — skin tone doesn't add a cluster)
tweetLength("Hello 👨‍👩‍👧‍👦");                  // 7 (family is one cluster)
tweetLength("Hello café");                // 10 (é is one cluster, precomposed or not)
```

Twitter's actual length counter is more generous than this: it weights certain ranges of code points differently. But the baseline — grapheme clusters as the unit of "characters" — is correct.

## Emoji are text now

The reason to take emoji seriously is that they *are* text. They are searched, they are typed, they are pasted, they are stored in databases, they are included in usernames, they are part of identifiers in some contexts. Every abstraction that works for letters must work for emoji too.

This is, in some sense, the final test of your Unicode code. If it works on `Hello world`, it works on ASCII. If it works on `Café résumé`, it works on Latin-script languages with diacritics. If it works on `👨‍👩‍👧‍👦`, it works on Unicode.

Next, we look at Unicode in programming language identifiers — and why, even when your language lets you name variables with emoji, you probably shouldn't.

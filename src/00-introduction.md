# Introduction: The Book That Would Have Saved You Three Bugs

You have written code that worked fine in testing and broke the moment a user pasted in their name.

You have seen the little box with the question mark — □ — where a letter should have been.

You have written `if (s.length > 10)` on a string and felt a quiet unease, because you half-remembered that some other language did this differently, and you were not sure which one was right.

You have stared at the bytes `C3 A9` in a hex dump and wondered what the `é` they were supposed to be had to do with `C3` or `A9`.

You have read *three separate tutorials* on UTF-8 and come away feeling like you understood it *right now*, in *this sentence*, but could not have explained it out loud five minutes later.

This book is for you.

## What this book is

A patient, honest tour of Unicode — what it is, what it isn't, how it is encoded into bytes, how those bytes travel between programs, how they are stored in files, how they are compared, how they are normalized, how they are sorted, how they can be used against you, and how your programming language treats them.

We will show you actual bytes. We will use real code. We will define each term precisely the first time we use it, and we will stick to those definitions. We will not tell you that Unicode is a mess, because it isn't. It is a careful, thoughtful standard that solved a genuinely hard problem. The confusion that surrounds it is mostly not its fault.

## What this book is not

It is not a replacement for the Unicode Standard itself. The standard is roughly 1,100 pages and encyclopedic; this book is a field guide. When you need to know the exact combining class of U+0308, you will still open *UnicodeData.txt*. But after reading this book, you will know what "combining class of U+0308" means, why you would care, and where that file is.

It is not a rant about JavaScript. (JavaScript does have some distinctive Unicode choices. We will explain them, with sympathy for the historical constraints that produced them.)

It is not a victory lap for languages that "got it right." No language has gotten all of Unicode right, because Unicode is still evolving and some of its answers are not universal.

## A small promise about vocabulary

The biggest single reason programmers get Unicode wrong is that four different things are all casually called *the character*. They are not the same thing. In order of abstraction, from most abstract to most concrete:

1. A *grapheme cluster* — what a human reader would call a "character." `é`, `😀`, `👨‍👩‍👧‍👦`. A user-perceived unit.
2. A *code point* — an integer assigned by the Unicode Standard. `U+0041` (`A`). `U+1F600` (`😀`). An abstract identity.
3. A *code unit* — the atomic piece of an encoding. For UTF-8 it is a byte. For UTF-16 it is a 16-bit value.
4. A *byte* — eight bits. What your disk actually stores.

We will draw those lines early and keep them drawn. When a language's standard library is counting something and calling it *length*, the first question is always: *length in what unit?*

## How to read this book

Front-to-back works, and the chapters are ordered so that each one uses vocabulary defined earlier. But the book is designed so you can also open it to a chapter in the middle, get a specific question answered, and close it again. Every chapter cross-references the ones it depends on.

If you remember only one thing from the whole book, let it be this:

> There are four different things called *character*. Every Unicode bug begins with someone mixing two of them up.

Everything else follows from that.

Let's begin with the history, because the most confusing parts of Unicode are the parts it inherited.

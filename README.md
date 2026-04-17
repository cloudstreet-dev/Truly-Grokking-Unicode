# Truly Grokking Unicode

Code points, code units, bytes. A patient tour of the text encoding system every programmer has been bitten by and few actually understand.

**Read the book:** <https://cloudstreet-dev.github.io/Truly-Grokking-Unicode/>

## Why this book

Every working programmer has an embarrassing "I thought characters were bytes" story. This book turns that confusion into clarity — without hand-waving, without pretending Unicode is simpler than it is, and without lying to you about what your language's standard library is actually doing.

Written for programmers who have been burned by Unicode and are ready to actually get it.

## Building locally

```sh
cargo install mdbook   # or: brew install mdbook
mdbook serve --open
```

Source lives in `src/`; the generated site is written to `book/` (gitignored).

## License

[CC0 1.0 Universal](./LICENSE) — public domain dedication. Use it however you like.

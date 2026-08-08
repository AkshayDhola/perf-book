# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Policy (read first)

From `README.md`: **"This book contains no material produced by generative AI,
and none will be accepted."** Do not write or reword book prose in `src/`.
Appropriate assistance here is limited to things like locating content,
checking links, verifying builds/tests, and reporting problems — leave the
wording to the author. The author also prefers suggestions be filed as issues
rather than pull requests.

## Commands

```bash
cargo install mdbook   # one-time toolchain setup
mdbook build           # render to book/ (gitignored)
mdbook serve           # local server at localhost:3000, live-reloads on change
mdbook test            # compile+run the Rust code blocks in the book
```

There is no per-test invocation: `mdbook test` is all-or-nothing over every
` ```rust ` block in the book. To isolate one chapter, copy its snippet into a
scratch file and run `rustc`/`cargo` on it directly.

## Structure

This is an mdbook, not a Rust crate — there is no `Cargo.toml`, and all content
is Markdown under `src/`.

- `src/SUMMARY.md` is the table of contents and the sole source of the chapter
  list and ordering. `book.toml` sets `create-missing = false`, so an entry in
  `SUMMARY.md` pointing at a nonexistent file **fails the build** rather than
  creating a stub. A new chapter needs both the file and the `SUMMARY.md` entry.
- Each remaining `src/*.md` is one self-contained chapter; there is no shared
  include or template layer.
- `.github/workflows/ci.yml` runs `mdbook build` then `mdbook test` on every PR
  and push, and deploys `book/` to GitHub Pages only on push to `master` of the
  `nnethercote/perf-book` repo.
- EPUB output (`mdbook-epub`) is commented out in `book.toml`, `README.md`, and
  CI; leave it that way unless the linked upstream issue is resolved.

## Style Conventions

Full guide is in `CONTRIBUTING.md`. The rules that matter most in practice:

- Text lines wrap at 79 characters (enforced only by `.editorconfig`, not CI).
  Lines holding links or other non-text elements may exceed it.
- External links use reference style, with the definition placed near its use,
  not collected at the end of the file.
- Section titles use title case ("Using an Alternative Allocator").
- Example links are the exception to reference style: inline, bolded, one per
  line — `[**Example**](url).` or `[**Example 1**](url),` / `[**Example 2**](url).`

Code fences: ` ```rust ` blocks are compiled by `mdbook test`, so a snippet
that cannot stand alone needs ` ```rust,ignore `. Shell commands and compiler
output use ` ```text ` or ` ```bash `; `Cargo.toml` fragments use ` ```toml `.

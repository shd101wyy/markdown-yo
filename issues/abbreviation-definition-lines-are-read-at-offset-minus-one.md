# Abbreviation definition lines are read at offset -1: a heap over-read on line 0, stray text in a paragraph

**Status:** Fixed
**Date:** 2026-10-22
**Severity:** Critical — a heap over-read (one byte before the source buffer) on ordinary input with abbreviations enabled; elsewhere the definition line leaks into the rendered paragraph

## Symptom

With `enable_abbr` (`--abbr`, the WASM API's full feature set):

```markdown
*[CSS]: Cascading Style Sheets

This is some CSS here.
```

renders correctly under the ordinary allocator, but an ASan build aborts with
`heap-buffer-overflow ... READ of size 1 ... 1 bytes before` the source
buffer. Running `scripts/run_fixture_tests.js` against an ASan binary failed
19 of the 23 `abbr` fixtures and all 4 `abbr_typographer` ones.

A definition that interrupts a paragraph or follows a list item rendered as
part of it:

| Input | markdown-it + markdown-it-abbr | markdown_yo |
| --- | --- | --- |
| `para\n*[X]: y\nmore X` | `<p>para</p>` `<p>more <abbr title="y">X</abbr></p>` | `<p>para\n\n*[<abbr title="y">X</abbr>]: y\nmore <abbr title="y">X</abbr></p>` |
| `- item\n*[X]: y\n\nX` | `<li>item</li>` | `<li>item\n\n*[<abbr title="y">X</abbr>]: y</li>` |

v0.0.19 (Yo 0.2.64) shows the same, so the Yo 0.2.67 migration did not cause it.

## Root cause

`prescan_abbr` (`src/block/abbr.yo`) marked every definition line it consumed
by setting the line's first-non-space offset to -1 (`set_ts(line, -1)`),
and `block_abbr`, which skips those lines, ran seventh in the block dispatch.
Every rule before it (table, fence, …) reads the byte at `bm(line) + ts(line)`,
which is `bm - 1`. For line 0 that is one byte before the buffer. For any
other line it is the previous line's newline, and the paragraph/list
continuation (which does not consult the abbreviation rule) took the line
in, starting from that newline.

## Fix

- The consumed mark is a per-line flag bit (bit 1 of the per-line flags byte,
  beside the `|` bit; `is_abbr_def` / `mark_abbr_def` in
  `src/block/state.yo`), so the line markers keep the values every rule
  expects.
- `block_abbr` runs first in the dispatch, so no other rule sees a consumed
  line.
- `_try_block_terminate` consults it, so a definition ends a paragraph, a
  setext-heading candidate or a reference title (markdown-it-abbr's
  `abbr_def` has `alt: ['paragraph', 'reference']`).

Regression test: `tests/main.test.yo`, "an abbreviation definition is consumed
without reading outside its line" (red before the fix: the first case under
ASan, the other two by their output).

Still open, same prescan: `issues/abbreviation-prescan-ignores-block-context.md`.

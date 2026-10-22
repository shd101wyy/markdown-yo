# The renderer reads the caller's bytes at normalized offsets: wrong output for CRLF input, a heap over-read for NUL input

**Status:** Fixed
**Date:** 2026-10-22
**Severity:** Critical — a heap over-read past the end of the source buffer on input holding `\0`, and wrong output for every CRLF document with an indented code block, a footnote or an abbreviation

## Symptom

A source that holds `\r` or `\0` is normalized before the block pass
(`\r\n` and `\r` become `\n`, `\0` becomes U+FFFD). Such input rendered
wrongly, and NUL input read past the source buffer:

| Input | markdown-it (+ footnote, abbr) | markdown_yo |
| --- | --- | --- |
| `a\r\n\r\n    code\r\n` | `<pre><code>code\n</code></pre>` | `<pre><code>  co\n</code></pre>` |
| `Text[^1].\r\n\r\n[^1]: Note.\r\n` (`--footnote`) | a footnote reference and the footnote section | `<p>Text<a href="Note.">^1</a>.</p>` |
| `*[X]: y\r\n\r\nX\r\n` (`--abbr`) | `<p><abbr title="y">X</abbr></p>` | `<p>\r</p>` |
| `x` + 20 × `\0` + `\n\n*[X]: yyyy…\n\nX\n` (`--abbr`) | the abbreviation applied | ASan: `heap-buffer-overflow`, `READ of size 1` |
| `    a` + 20 × `\0` + `b\n` (an indented code block) | `<pre><code>a����…b` | ASan: `heap-buffer-overflow`, `READ of size 8` |

The same input with `--footnote` and a `[^1]:` definition after the NULs
overflows too. `yo doc` enables both footnotes and abbreviations
(`src/doc/render_html.yo` in the Yo repository), and its `build_site.yo`
enables footnotes, so a doc comment or page saved with CRLF line endings
reaches this. v0.0.19 (Yo 0.2.64) shows the same.

## Root cause

`markdown_to_html` (`src/parser.yo`) creates the `HtmlRenderer` with the
caller's bytes (`src_ptr`, `src_len`, and `_orig_src`, the base the
abbreviation and footnote definition lookups read), then creates the
`BlockState`. When the source needs normalization, `BlockState.new` builds
the state over a normalized copy (`_build_block_state_from_normalized`), and
every offset the parse hands the renderer indexes that copy. The renderer
was never pointed at it: it kept reading the caller's bytes at the copy's
offsets. Each removed `\r` shifts the two buffers apart by one byte, and each
`\0` (one byte that becomes three) moves the copy's offsets toward and past
the end of the caller's buffer.

Text inside inlines rendered correctly only because the inline renderer
swaps `src_ptr` to the inline state's pointer around each inline run
(`src/inline/render.yo`); the block-level `escaped` call of an indented code
block and the `_orig_src` lookups do not go through that swap.

## Fix

`HtmlRenderer.set_source` points `src_ptr`, `src_len` and `_orig_src` at the
given bytes, and `_build_block_state_from_normalized` (`src/block/state.yo`)
calls it with the normalized buffer before the block pass. The state owns
that buffer (it moves into the state; a move keeps the heap buffer, so the
pointer stays valid), and the state lives until `markdown_to_html` returns.

Regression test: `tests/main.test.yo`, "a source that needs normalization
renders like its normalized form" (red before the fix under Yo 0.2.67,
0.2.68 and the tree compiler; the test runner builds with ASan by default).

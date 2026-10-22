# Footnote definitions nested in a blockquote or a list item are not registered

**Status:** Open
**Date:** 2026-10-22
**Severity:** Minor — output-fidelity divergence from markdown-it-footnote; the output is valid HTML

## Symptom

With `enable_footnote` (`--footnote`):

| Input | markdown-it + markdown-it-footnote | markdown_yo |
| --- | --- | --- |
| `A[^1].\n\n> [^1]: b` | `<p>A<sup …>[1]</sup>.</p>` `<blockquote></blockquote>`, then the footnote section with `b` | `<p>A[^1].</p>` `<blockquote></blockquote>` |
| `A[^1].\n\n- [^1]: b` | `<ul><li></li></ul>`, the reference and the footnote section with `b` | the reference stays text |

The definition's line is consumed (the container renders empty), but the
definition is never registered, so the reference finds nothing. v0.0.19
shows the same.

## Root cause

markdown_yo renders while it parses, so it registers footnote definitions
in `_prescan_footnotes` (`src/block/parser.yo`) before the block pass. The
prescan matches `[^label]:` only at a line's first non-space byte, so a
definition behind a blockquote marker or a list marker is never seen.
markdown-it-footnote's `footnote_def` is an ordinary block rule and
registers a definition in any container.

## Direction

Give the prescan the container context it needs (strip blockquote markers,
follow list-item content indentation), the same gap the abbreviation
prescan has (`issues/abbreviation-prescan-ignores-block-context.md`). The
block pass then skips a nested definition through the line it is registered
at (`HtmlRenderer.fn_def_at_line`), as it does a top-level one.

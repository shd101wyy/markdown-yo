# The abbreviation prescan ignores block context: code and HTML blocks define abbreviations, blockquotes do not

**Status:** Open
**Date:** 2026-10-22
**Severity:** Minor — output-fidelity divergence from markdown-it-abbr; the output is valid HTML

## Symptom

With `enable_abbr` (`--abbr`):

| Input | markdown-it + markdown-it-abbr | markdown_yo |
| --- | --- | --- |
| ` ```\n*[X]: y\n```\n\nX here` | `<p>X here</p>` | `<p><abbr title="y">X</abbr> here</p>` |
| `<div>\n*[X]: y\n</div>\n\nX here` (`html: true`) | `<p>X here</p>` | `<p><abbr title="y">X</abbr> here</p>` |
| `> *[X]: y\n\nX here` | `<blockquote></blockquote>` `<p><abbr title="y">X</abbr> here</p>` | `<blockquote><p>*[X]: y</p></blockquote>` `<p>X here</p>` |
| `> para\n*[X]: y\nX` (a lazy continuation line) | `<blockquote><p>para\n*[X]: y\nX</p></blockquote>` | `<blockquote><p>para\n*[<abbr title="y">X</abbr>]: y\n<abbr title="y">X</abbr></p></blockquote>` |
| `\| a \| b \|\n\|---\|---\|\n*[X]: y\n\nX` (a table row) | the row `<td>*[X]: y</td>`, then `<p>X</p>` | the row `<td>*[<abbr title="y">X</abbr>]: y</td>`, then `<p><abbr title="y">X</abbr></p>` |

The fenced and HTML-block cases still render the line as block content; only
the registration is wrong. In the lazy-continuation and table-row cases the
line is both rendered as text and registered, because the line is not at a
block start, so the block pass never reaches the abbreviation rule there.
v0.0.19 shows the same.

## Root cause

markdown-it parses `abbr_def` as an ordinary block rule, so a definition
counts only where a block may start, and applies all of them to the inline
text afterwards. markdown_yo renders while it parses (SAX), so it registers
definitions in `prescan_abbr` (`src/block/abbr.yo`) before the block pass.
That prescan looks at each non-empty line at column 0 on its own: it does not
skip fenced code or HTML blocks, and it does not look inside a blockquote
(`> *[X]: y`) or an indented list item.

## Direction

Give the prescan the block context it needs: skip fenced regions the way
`_prescan_references` already does, skip HTML blocks when `html` is on, and
strip blockquote markers before matching. A definition also only counts
where a block may start: not on a paragraph's lazy continuation line (inside
a blockquote or a list item) and not on a line that continues a table's body.
The prescan has to track that much block state
itself: registering in the block pass instead (where `block_abbr` already
runs only at a block start) would be too late for text the renderer has
already emitted. The consumed-line flag
(`mark_abbr_def`) then has to cover only lines the block pass reaches at a
block start.

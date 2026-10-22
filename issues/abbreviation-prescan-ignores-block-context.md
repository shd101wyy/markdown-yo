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

The fenced and HTML-block cases still render the line as block content; only
the registration is wrong. v0.0.19 shows the same.

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
strip blockquote markers before matching. The consumed-line flag
(`mark_abbr_def`) then has to cover only lines the block pass reaches at a
block start.

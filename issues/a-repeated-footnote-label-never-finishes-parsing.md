# A repeated footnote label never finishes parsing: the block pass loops and the output grows without bound

**Status:** Fixed
**Date:** 2026-10-22
**Severity:** Critical — with footnotes enabled, a document that defines one label twice never finishes rendering, and its memory grows until the process is killed

## Symptom

With `enable_footnote` (`--footnote`, the WASM API's full feature set, and
`yo doc`, whose `src/doc/render_html.yo` and `scripts/build_site.yo` in the
Yo repository both enable footnotes):

```markdown
A[^1].

[^1]: a

[^1]: b
```

never returns. A reference is not needed (`[^1]: a\n\n[^1]: b\n` loops too),
and neither is a second top-level definition: one inside a blockquote or a
list item (`> [^1]: b`, `- [^1]: b`) after a top-level one loops the same
way. An ASan build rendering `tests/fixtures/markdown_it/footnote.txt` (a
fixture file, which repeats `[^1]:` across its cases) with `--footnote`
reached 10 GB resident before it was killed. v0.0.19 (Yo 0.2.64) shows the
same. The fixture suite never hit it, because each fixture case renders on
its own.

markdown-it-footnote renders the example with the reference pointing at the
last definition (`<p>b ↩︎</p>` in the footnote section).

## Root cause

`_prescan_footnotes` registers every top-level definition before the block
pass, and the block pass skips a definition's lines (rule 7b in
`src/block/parser.yo`) by looking the definition up with `fn_find_def`,
which matches BY LABEL and returned the first match, then setting the line
to that definition's `line_end`. At the second definition of a label, the
lookup found the first one, whose end lies before the current line, so the
parse jumped back, reached the second definition again, and repeated
forever, emitting the text between the two each time.

## Fix

- Rule 7b looks the definition up by the line it starts on
  (`HtmlRenderer.fn_def_at_line`), so the skip always moves forward; a
  definition the prescan did not register at that line (one nested in a
  container) is not skipped through another definition's range.
- `fn_find_def` returns the LAST definition of a label, so a reference uses
  the content markdown-it-footnote uses.

Regression test: `tests/main.test.yo`, "a repeated footnote label finishes
and the last definition wins". Before the fix its inputs do not terminate:
the test binary was killed by a 120 s watchdog under Yo 0.2.67, 0.2.68 and
the tree compiler.

Still open, same prescan:
`issues/footnote-definitions-nested-in-a-container-are-not-registered.md`.

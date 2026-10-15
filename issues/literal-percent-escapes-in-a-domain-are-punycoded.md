# Literal percent-escapes in a link's domain are punycoded; markdown-it keeps them

**Status:** Open
**Date:** 2026-10-16
**Severity:** Minor — output-fidelity divergence from markdown-it on unusual input; the output is valid

## Symptom

| Input | markdown-it 14 | markdown_yo |
| --- | --- | --- |
| `[l](http://a%C3%A3/)` | `href="http://a%C3%A3/"` | `href="http://xn--a-yfa/"` |
| `[l](http://a%F0/)` | `href="http://a%F0/"` | `href="http://xn--a-q10i/"` |

## Root cause

`normalize_link` percent-encodes raw UTF-8 in the URL first, then scans the
encoded domain for `%C0`+ escapes and punycodes the percent-decoded label.
After the first pass a literal `%C3%A3` in the source is indistinguishable from
an encoded raw `ã`, so it is punycoded too. markdown-it punycodes only the raw
non-ASCII hostname (`mdurl.parse` + `punycode.toASCII`) and leaves escapes in
the hostname untouched.

## Fix direction

Decide which domain bytes to punycode from the source bytes (raw `>= 0x80`),
not from the escapes in the already-encoded output. The regression test for the
punycode over-read (`tests/main.test.yo`) pins today's labels and changes with
this fix.

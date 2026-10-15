# `punycode_encode` reads up to 3 bytes past its buffer

**Status:** Fixed
**Date:** 2026-10-16
**Severity:** Critical — a heap over-read on ordinary ASCII markdown; the bytes read land in the rendered `href`

Upstream record: Yo's `issues/fixed/vendor-markdown-punycode-encode-reads-past-the-buffer.md`.

## Symptom

```markdown
[l](http://a%F0/)
```

rendered `<a href="http://xn--a<0x80>-/">` under the ordinary allocator (the
`0x80` is whatever followed the allocation) and, under ASan
(`yo test ./tests --allocator system`), aborted with
`heap-buffer-overflow ... READ of size 1`.

## Root cause

`normalize_link` percent-decodes a domain holding a `%C0`+ escape into a
fresh, exactly-sized buffer and passes it to `encode_domain` →
`punycode_encode`. That decoder, unlike the other UTF-8 decoders in `src/`,
read the continuation bytes at `p+1..p+3` without checking them against the
label length, and took every lead byte that was not ASCII, `0xC0..0xDF` or
`0xE0..0xEF` as a four-byte lead — a bare continuation byte included. So a
trailing `%F0`, or a trailing `%80` after a `%C3%A3`, read 3 bytes past the
buffer.

## Fix

Each multi-byte arm of the decoder in `src/common/punycode.yo` is gated on its
continuation bytes lying inside the label, the four-byte lead is a range
(`0xF0..0xF7`), and everything else (a truncated sequence, a bare continuation
byte, `0xF8..0xFF`) decodes to one U+FFFD consuming one byte.

Regression test: `tests/main.test.yo`, "punycode of a truncated
percent-decoded UTF-8 domain stays in bounds" (red under ASan before the fix).

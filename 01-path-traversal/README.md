# Path traversal

> **OWASP:** A01:2021 Broken Access Control · **CWE:** CWE-22

<2-3 sentences on how you think about this class of vulnerability — the embryo of a blog-style write-up. Optional, fill as you go.>

## Labs

| # | Lab | Level | Mechanism | Done |
|---|-----|-------|-----------|------|
| 01 | [File path traversal, simple case](01-file-path-traversal-simple-case.md) | Apprentice | Basic traversal (../) | ⬜ |
| 02 | [Traversal sequences blocked with absolute path bypass](02-traversal-sequences-blocked-with-absolute-path-byp.md) | Practitioner | Absolute path bypass | ⬜ |
| 03 | [Traversal sequences stripped non-recursively](03-traversal-sequences-stripped-non-recursively.md) | Practitioner | Non-recursive stripping (....//) | ⬜ |
| 04 | [Traversal sequences stripped with superfluous URL-decode](04-traversal-sequences-stripped-with-superfluous-url-.md) | Practitioner | Double URL-encode (%252e) | ⬜ |
| 05 | [Validation of start of path](05-validation-of-start-of-path.md) | Practitioner | Start-of-path validation bypass | ⬜ |
| 06 | [Validation of file extension with null byte bypass](06-validation-of-file-extension-with-null-byte-bypass.md) | Practitioner | Null byte (%00) | ⬜ |

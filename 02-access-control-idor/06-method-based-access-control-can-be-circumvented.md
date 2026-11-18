# Method-based access control can be circumvented

**Category:** Access control / IDOR
**Level:** Practitioner
**Mechanism:** Authorization enforced only on one HTTP method — switch the verb to bypass

**Lab URL:** https://portswigger.net/web-security/access-control/lab-method-based-access-control-can-be-circumvented

---

## Vulnerability

The privileged action (promote a user to admin) is protected by a check that is bound to
the **HTTP method**, not to the user's role. The server rejects an unauthorized `POST` to
the admin endpoint, but the same action over a different method (`GET`) is not covered by
the check — so a low-privileged user performs the admin action by changing the verb.

## Reconnaissance & Exploitation

Logging in as the admin (`administrator:admin`), the panel promotes a user with:

```http
POST /admin-roles HTTP/2
Content-Type: application/x-www-form-urlencoded

username=carlos&action=upgrade
```

Replaying this with **wiener's** session cookie returned `401 Unauthorized` — the check
works for `POST`:

```http
POST /admin-roles HTTP/2
Cookie: session=<wiener session>

username=administrator&action=upgrade
```
```http
HTTP/2 401 Unauthorized
"Unauthorized"
```

Probing how the method is handled, changing `POST` → `POSTX` returned "missing parameter"
instead of "Unauthorized" — the routing/authorization is tied to the specific verb, not to
the action. So I converted the request to **GET** (params move to the query string) and set
the username to my own account:

```http
GET /admin-roles?username=wiener&action=upgrade HTTP/2
Host: 0a55001e0334f085829d42a10069004b.web-security-academy.net
Cookie: session=<wiener session>
```
```http
HTTP/2 302 Found
Location: /admin
```

The `302` (not `401`) means the action succeeded: wiener was promoted to administrator —
lab solved.

## Impact

Vertical privilege escalation: any authenticated low-privileged user can grant themselves
admin by issuing the privileged request over a method the access-control check doesn't
cover. The admin panel's protection is illusory — it guards the `POST` verb, not the
capability — so the entire role model collapses to "know the endpoint and switch the
method."

## Remediation

- Enforce authorization on the **action and the user's role**, independent of HTTP method.
  The server must verify "is this session an admin?" before performing the upgrade,
  regardless of `GET`/`POST`/anything else.
- Apply a single, centralized access-control check that every route to the function passes
  through, rather than per-method handlers that can diverge.
- Reject unexpected methods on sensitive endpoints (405 Method Not Allowed) instead of
  silently processing them.

## References

- **OWASP:** A01:2021 Broken Access Control
- **CWE:** CWE-285 (Improper Authorization); CWE-650 (Trusting HTTP Permission Methods on the Server Side)

## Notes

- The tell is the *difference* in responses by method: `POST` → `401 Unauthorized`, but
  `POSTX` → "missing parameter" (not 401). That means the auth decision is keyed to the
  literal verb, so a verb the check doesn't recognise sidesteps it.
- `GET` carries the parameters in the query string, `POST` in the body — same action, same
  params, just relocated. "Change request method" in Repeater does the conversion.
- General test for method-based flaws: take a privileged request, replay it with a
  low-priv session, then try `GET`, `HEAD`, `PUT`, arbitrary verbs. If any variant drops
  the authorization error, the control is method-bound and broken.
- Access control must be decided by *who* and *what*, never by *how* the request arrives.
# User ID controlled by request parameter, with unpredictable user IDs

**Category:** Access control / IDOR
**Level:** Apprentice
**Mechanism:** IDOR with GUIDs — unpredictable ID leaked elsewhere
**Lab URL:** https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-unpredictable-user-ids

---

## Vulnerability

The account page selects whose data to return from a user-controlled `id`, with no
server-side check that the `id` belongs to the session — the same IDOR as the previous
lab. Here the `id` is a GUID instead of a username, so it cannot simply be guessed. But
an unpredictable identifier is not an access control: the GUID is leaked elsewhere in
the application, so an attacker who finds it still reads the victim's account.

## Reconnaissance & Exploitation

Logged in as `wiener:peter`, my account used a GUID in `id`:

```http
GET /my-account?id=5494163d-17aa-446e-86fc-7cda85c8025e HTTP/2
```

Guessing another user's GUID is infeasible, so the task is to *find* carlos's GUID.
The blog surfaces it: a post authored by carlos links to his user ID.

```http
GET /blogs?userId=521c43c6-7c1b-4c61-8b27-4da9e201f87d HTTP/2
(carlos's author link — his GUID is exposed in the URL)
```

With carlos's GUID in hand, I replaced my own `id` with his on the account page (still
using my `wiener` session):

```http
GET /my-account?id=521c43c6-7c1b-4c61-8b27-4da9e201f87d HTTP/2
Host: 0a67004304010ad580e3c1e500bf0005.web-security-academy.net
Cookie: session=<redacted>
```

```http
HTTP/2 200 OK
...
Your username is: carlos
Your API Key is: pbkp17qhA6yvjeHDYBwl7PlCKcNzNEC7
```

Submitting carlos's API key solved the lab. The server still authorised purely on the
`id`, never checking it matched the authenticated session — the GUID only raised the
bar on *finding* the value, not on *using* it.

## Impact

Any user's account data is readable by anyone who obtains their GUID — and the
application hands those GUIDs out (author links, profile pages, API responses, referer
headers). So the "unpredictable" ID provides no real protection: an attacker harvests
GUIDs from public content and pivots each one into the victim's private data (here, API
keys). Same breach as a numeric IDOR, one recon step later.

## Remediation

- Enforce object-level authorisation: verify the session owns the requested `id` before
  returning data. This is the only real fix.
- Derive the account from the session for "my account" pages, removing the `id`
  parameter entirely.
- Treat unpredictable IDs (GUIDs) as defence in depth at most, never as access control —
  and avoid leaking them in URLs, author links and other reachable places.

## References

- **OWASP:** A01:2021 Broken Access Control
- **CWE:** CWE-639 (Authorization Bypass Through User-Controlled Key); CWE-284 (Improper Access Control)

## Notes

- Key lesson: *security through obscurity is not access control.* A random GUID only
  makes the reference harder to guess; it does nothing once the value is known.
- The exploit is two steps now: (1) find the victim's identifier leaked somewhere in the
  app, (2) replay the IDOR with it. Author/profile links are the usual leak.
- When an app uses GUIDs for objects, map where each GUID is exposed — that exposure is
  the real attack surface.
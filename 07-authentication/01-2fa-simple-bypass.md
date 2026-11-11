# 2FA simple bypass

**Category:** Authentication
**Level:** Apprentice
**Mechanism:** 2FA step not enforced on protected endpoints — skip the verification page

**Lab URL:** https://portswigger.net/web-security/authentication/multi-factor/lab-2fa-simple-bypass

---

## Vulnerability

The login is two-step: username/password, then a 2FA code sent by email. But the
session becomes fully authenticated after the **password** step, and the protected
pages do not check that the 2FA step was completed. The verification page is only a
navigational gate, not an access-control gate — so an attacker who knows the password
reaches the account by skipping straight to a protected URL.

## Reconnaissance & Exploitation

Logging into my own account, the password step (`POST /login`) returned a `302` to the
2FA page (`/login2`), where a code is requested:

```http
POST /login HTTP/2
→ 302 Found
Location: /login2
Set-Cookie: session=...
```

The key observation: the session cookie is already set after the password step, and
`/login2` is just the next page in the flow. I noted the post-login landing URL
(`/my-account`).

I then logged in with the victim's credentials (`carlos:montoya`). At the 2FA prompt —
without entering any code — I navigated directly to the protected endpoint:

```http
GET /my-account HTTP/2
Host: 0a25008a04c7f04b813117b20056008a.web-security-academy.net
Cookie: session=<carlos session, password step only>
```

```http
HTTP/2 200 OK
...
Your username is: carlos
```

carlos's account loaded without the 2FA code — lab solved. The server never verified
that this session had passed the second factor before serving the protected page.

## Impact

The second factor provides no protection: anyone with a valid username and password
(phished, reused, brute-forced) reaches the account by requesting a protected URL and
ignoring the 2FA page. The entire point of 2FA — stopping access when the password alone
is compromised — is defeated, because the factor is enforced on the *page*, not on the
*session's access to resources*.

## Remediation

- Enforce the 2FA state server-side on every protected request: the session must be
  marked "fully authenticated" only **after** the code is verified, and protected
  endpoints must reject sessions still at the password-only stage.
- Treat the flow as a state machine: password step → "pending 2FA" session (no access to
  anything but the verification endpoint) → code verified → authenticated session.
- Do not rely on redirects/UI order for security; a user can request any URL directly.

## References

- **OWASP:** A07:2021 Identification and Authentication Failures
- **CWE:** CWE-287 (Improper Authentication); CWE-306 (Missing Authentication for Critical Function)

## Notes

- The flaw is a missing check, not a broken code: the 2FA logic works, but the protected
  resource doesn't ask "did this session pass 2FA?" — only "is it logged in?".
- Test pattern for any multi-step auth: complete step 1, then request a post-login URL
  directly instead of doing step 2. If it loads, the later step isn't enforced.
- Forced browsing (requesting a known later-stage URL out of order) is the general
  technique — same idea as skipping a step in a multi-step checkout/workflow.
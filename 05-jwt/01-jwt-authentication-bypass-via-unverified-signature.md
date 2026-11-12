# JWT authentication bypass via unverified signature

**Category:** JWT
**Level:** Apprentice
**Mechanism:** Server does not verify the JWT signature — tamper the payload

**Lab URL:** https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-unverified-signature

---

## Vulnerability

The session token is a JWT, and the user's identity is carried in the `sub` claim of
the payload. The server **does not verify the token's signature**, so an attacker can
change the payload (e.g. `sub: wiener` → `sub: administrator`) and the server trusts the
tampered token — the signature exists but is never checked, defeating the entire point
of signing.

## Reconnaissance & Exploitation

After logging in as `wiener:peter`, the session cookie is a JWT. Decoding it showed a
standard structure — header (`alg: RS256`, `kid`), payload, signature:

```
header:  { "kid": "8cc4f2ea-...", "alg": "RS256" }
payload: { "iss": "portswigger", "exp": ..., "sub": "wiener" }
```

Requesting `/admin` with the original token was refused:

```http
GET /admin → 401 Unauthorized
"Admin interface only available if logged in as an administrator"
```

The admin check keys off the `sub` claim. In Burp's **JSON Web Tokens** tab I changed
`sub` from `wiener` to `administrator` and, crucially, left the signature untouched
("Do not automatically modify signature") — the test is whether the server validates it:

```
payload: { "iss": "portswigger", "exp": ..., "sub": "administrator" }
(signature unchanged — no longer matches the payload)
```

```http
GET /admin HTTP/2
Cookie: session=<JWT with sub=administrator, original signature>
```

```http
HTTP/2 200 OK
<h1>Users</h1> ... /admin/delete?username=carlos
```

The admin panel loaded despite the broken signature — proof the server never verifies
it. Requesting `/admin/delete?username=carlos` deleted carlos and solved the lab.

## Impact

Full authentication/authorisation bypass: anyone with a valid token for any account
forges a token for any user — including `administrator` — by editing one claim. Because
the signature is ignored, the JWT provides zero integrity; identity and privilege are
entirely attacker-controlled, leading to complete account takeover and admin compromise.

## Remediation

- Always verify the JWT signature on every request, with the correct algorithm and key,
  before trusting any claim. A library misconfiguration (e.g. `decode()` without
  `verify()`) is the usual cause.
- Reject tokens whose signature does not validate; never fall back to accepting an
  unverified or `alg: none` token.
- Treat all claims (`sub`, roles) as untrusted until the signature is verified — and
  derive authorisation server-side from the verified identity.

## References

- **OWASP:** A07:2021 Identification and Authentication Failures
- **CWE:** CWE-347 (Improper Verification of Cryptographic Signature)

## Notes

- A JWT is three base64url parts: `header.payload.signature`. The signature is the only
  thing stopping the client from editing the payload — if it isn't checked, the token is
  just editable plaintext.
- Test reflex: change a claim, leave the signature as-is, and send. If it's accepted,
  the signature isn't verified (this lab). If rejected, move to signature-focused attacks
  (weak key, `alg: none`, jwk/jku/kid — the later labs).
- "Do not automatically modify signature" in Burp is the correct setting here — the goal
  is precisely to send a payload the signature no longer matches.
# JWT authentication bypass via jwk header injection

**Category:** JWT  
**Level:** Practitioner  
**Mechanism:** Attacker-supplied RSA public key in the `jwk` header  
**Lab URL:** https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-jwk-header-injection

---

## Vulnerability

The application accepts a public key embedded in the JWT header through the `jwk` parameter and uses it to verify the token's signature. Because the server does not check that this key came from a trusted source, an attacker can create a key pair, sign a token with its private key, and include the corresponding public key in the token.

After changing the `sub` claim to `administrator`, the server accepts the token as an administrator session. The signature is valid for the supplied key, but the server has not established that the key belongs to a trusted issuer.

## Reconnaissance & Exploitation

**Step 1 — Capture the authenticated request.** Logged in as `wiener:peter` and sent the post-login `GET /my-account?id=wiener` request to Repeater. The original request returned `200 OK`. The JWT Editor tab showed the token's header, payload and signature.

```http
GET /my-account?id=wiener HTTP/2
Cookie: session=<original JWT>
```

```http
HTTP/2 200 OK
```

**Step 2 — Generate an RSA key pair.** In the JWT Editor **Keys** tab, created a new RSA key in JWK format using **Generate**, then saved it with **OK**.

**Step 3 — Embed the public key in the JWT.** Returned to the request in Repeater and used **Attack → Embedded JWK** in the JWT Editor. Selected the generated RSA key. The extension added the public key to the JWT header and signed the token with the corresponding private key.

**Step 4 — Change the subject to the administrator.** In the JWT Editor payload, changed the `sub` claim to `administrator`. Sent the request with the updated token.

```http
GET /my-account?id=administrator HTTP/2
Cookie: session=<JWT with sub=administrator and embedded JWK>
```

```http
HTTP/2 200 OK

Your username is: administrator
```

The account page displayed `administrator`, confirming that the application accepted the modified token as an administrator session.

**Step 5 — Delete the target user.** Used the administrator session to send the deletion request for `carlos`.

```http
GET /admin/delete?username=carlos HTTP/2
Cookie: session=<administrator JWT with embedded JWK>
```

```http
HTTP/2 302 Found
Location: /admin
```

The lab displayed **“Congratulations, you solved the lab!”** after the request.

## Impact

An attacker who can obtain a valid JWT can create a key pair, embed its public key in the token, and sign a modified token with the matching private key. If the application trusts the token-supplied key, the attacker can impersonate an administrator and perform privileged actions, including deleting users.

## Remediation

- Verify JWT signatures using keys obtained from a trusted, server-configured source associated with the expected issuer.
- Do not treat a public key supplied inside the JWT header as trusted solely because the token's signature matches it.
- Allowlist the expected signing algorithms and validate the issuer, audience, expiration and other required claims.
- Reject tokens with unexpected key-selection parameters or keys that do not match the trusted key set.

## References

- **OWASP:** A07:2021 Identification and Authentication Failures
- **CWE:** CWE-347 (Improper Verification of Cryptographic Signature)

## Notes

- The generated RSA key pair is controlled by the party that created it. The private key signs the modified token, while the public key is embedded in the JWT for verification.
- A valid signature proves that the token matches the supplied key. The vulnerability is that the application trusts that supplied key without confirming its origin.
- The lab was confirmed solved after the administrator session was used to delete `carlos`.
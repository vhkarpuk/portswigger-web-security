# JWT authentication bypass via jwk header injection

**Category:** JWT
**Level:** Practitioner
**Mechanism:** Attacker-supplied RSA public key in the JWT `jwk` header

**Lab URL:** https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-jwk-header-injection

---

## Vulnerability

The application accepts a public key embedded in the JWT header through the `jwk` parameter and uses it to verify the token's signature. It does not check whether this key came from a trusted source. An attacker can therefore generate a key pair, sign a modified token with the private key, and provide the matching public key in the token.

Because the server trusts the supplied key, changing the `sub` claim to `administrator` results in an authenticated administrator session.

## Reconnaissance & Exploitation

**Step 1 — Baseline as a normal user.** Logged in as `wiener:peter` and sent the post-login `GET /my-account?id=wiener` request to Repeater. The response returned `200 OK` and showed the account for `wiener`. In the JWT Editor tab, inspected the token header and payload, which showed `alg: RS256` and `sub: wiener`.

```http
GET /my-account?id=wiener HTTP/2
Cookie: session=<JWT, alg=RS256, sub=wiener>
```

```http
HTTP/2 200 OK

Your username is: wiener
```

**Step 2 — Generate an RSA key pair.** In Burp's **JWT Editor Keys** tab, selected **New RSA Key**, clicked **Generate**, and saved the generated key with **OK**.

**Step 3 — Embed the generated key.** Returned to the authenticated request in Repeater and selected **Attack → Embedded JWK** in the JWT Editor tab. Selected the generated RSA signing key. The extension embedded the corresponding public key in the JWT header and signed the token. With the original `sub` claim still set to `wiener`, the response continued to show the `wiener` account.

**Step 4 — Change the subject to the administrator.** Changed the payload's `sub` claim to `administrator` and sent the request to `GET /my-account?id=administrator` with the updated JWT. The response returned `200 OK` and showed `Your username is: administrator`.

```http
GET /my-account?id=administrator HTTP/2
Cookie: session=<JWT, sub=administrator, with embedded JWK>
```

```http
HTTP/2 200 OK

Your username is: administrator
```

This confirmed that the application accepted the token as an administrator session.

**Step 5 — Delete the target user.** Used the administrator session to send the deletion request for `carlos` found in the admin response.

```http
GET /admin/delete?username=carlos HTTP/2
Cookie: session=<administrator JWT with embedded JWK>
```

```http
HTTP/2 302 Found
Location: /admin
```

The browser displayed **“Congratulations, you solved the lab!”** after the deletion request.

## Impact

An attacker can forge a JWT for another user, including an administrator, and perform privileged actions. In this lab, the forged administrator session allowed the attacker to delete the `carlos` account.

## Remediation

- Verify JWT signatures using keys from a trusted, server-configured key set associated with the expected issuer.
- Do not trust a public key supplied in the JWT header as the source of truth for signature verification.
- Allowlist the expected signing algorithm and validate claims such as issuer, audience and expiration.

## References

- **OWASP:** A07:2021 Identification and Authentication Failures
- **CWE:** CWE-347 (Improper Verification of Cryptographic Signature)

## Notes

- The JWT Editor's **Embedded JWK** attack adds the generated public key to the JWT header and signs the token with the matching private key.
- Embedding the key does not change the identity by itself. The `sub` claim must identify the target account.
- The lab was confirmed solved after sending the administrator deletion request for `carlos`.
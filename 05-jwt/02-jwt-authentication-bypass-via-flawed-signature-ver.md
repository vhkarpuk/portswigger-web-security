# JWT authentication bypass via flawed signature verification

**Category:** JWT
**Level:** Apprentice
**Mechanism:** `alg: none` — server accepts unsigned JWTs

**Lab URL:** https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-flawed-signature-verification

---

## Vulnerability

The server accepts JWTs whose header declares `alg: none`, meaning "unsigned". The JWT
spec allows `none` for tokens whose integrity is guaranteed by other means, but a server
that honours it on session tokens lets an attacker strip the signature entirely and
present a self-made token. With no signature required, any claim — including the user's
identity — can be forged.

## Reconnaissance & Exploitation

The session cookie is a JWT (`alg: RS256`, `sub: wiener`). Requesting `/admin` was
refused because the admin check keys off `sub`:

```http
GET /admin → 401 Unauthorized
```

Unlike the previous lab (where the original signature was kept), here I switch the
algorithm off entirely. In Burp's **JSON Web Tokens** tab I made three changes:

1. Payload: `sub: wiener` → `sub: administrator`
2. Header: `alg: RS256` → `alg: none`
3. Removed the signature from the token — **keeping the trailing dot** after the payload

The token becomes `header.payload.` (empty third part):

```
header:  { "kid": "...", "alg": "none" }
payload: { "iss": "portswigger", "exp": ..., "sub": "administrator" }
signature: (removed — trailing dot kept)
```

```http
GET /admin HTTP/2
Cookie: session=<alg:none JWT, sub=administrator, no signature>
```

```http
HTTP/2 200 OK
<h1>Users</h1> ... /admin/delete?username=carlos
```

The admin panel loaded with an unsigned, self-crafted token. Requesting
`/admin/delete?username=carlos` returned `302 Location: /admin` — carlos deleted, lab
solved.

## Impact

Complete authentication bypass with no secret or key needed: an attacker forges a token
for any user, including `administrator`, simply by setting `alg: none` and writing the
desired claims. Worse than the unverified-signature case because it requires no starting
token tampering trick — the server itself invites unsigned tokens, so identity and
privilege are fully attacker-defined.

## Remediation

- Explicitly reject `alg: none` (and any unexpected algorithm) on session tokens.
  Pin the accepted algorithm server-side; never derive it from the attacker-controlled
  header.
- Require and verify a signature on every token with the correct key. A verification
  routine that trusts the header's `alg` is the root cause.
- Keep the JWT library updated and configured to allow only the intended algorithms
  (allowlist), not whatever the token claims.

## References

- **OWASP:** A07:2021 Identification and Authentication Failures
- **CWE:** CWE-347 (Improper Verification of Cryptographic Signature); CWE-345 (Insufficient Verification of Data Authenticity)

## Notes

- The `alg` header is attacker-controlled — never let it decide how (or whether) the
  token is verified. Trusting it is the whole bug class.
- Structural detail: the token must keep its format `header.payload.signature`, so after
  removing the signature the **trailing dot stays** (`header.payload.`) or the token is
  malformed.
- Escalation of the signature-attack ladder: lab 01 = signature not checked at all;
  this lab = `alg: none` accepted; next = weak key brute-force, then jwk/jku/kid header
  injection.
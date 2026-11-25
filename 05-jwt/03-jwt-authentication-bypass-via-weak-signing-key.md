# JWT authentication bypass via weak signing key

**Category:** JWT
**Level:** Practitioner
**Mechanism:** HS256 token signed with a guessable secret — brute-force the key, then forge tokens

**Lab URL:** https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-weak-signing-key

---

## Vulnerability

The application signs session JWTs with `HS256` (HMAC-SHA256), which uses one shared secret
to both sign and verify. The secret is extremely weak and appears in public wordlists of
common secrets. Anyone who obtains a valid token can test candidate secrets offline: if the
signature matches, the secret is found. With the secret in hand, the attacker can sign any
payload, so the server accepts a token claiming to be `administrator`.

## Reconnaissance & Exploitation

**Step 1 — Baseline as a normal user.** Logged in as `wiener:peter` and sent `GET /my-account`
to Repeater. Changing the path to `/admin` returned `401 Unauthorized` for `wiener`, so the
admin panel is gated on the identity inside the JWT.

```http
GET /admin HTTP/2
Cookie: session=<JWT, sub=wiener, alg=HS256>
```

```http
HTTP/2 401 Unauthorized
```

**Step 2 — Recover the signing key.** A HS256 token has three parts (`header.payload.signature`).
The first two are only base64url-encoded, so the whole token is enough to test any secret
offline. Hashcat mode `16500` is JWT (HMAC-SHA256/384/512):

```bash
hashcat -a 0 -m 16500 <YOUR-JWT> jwt.secrets.list
```

Output format is `<JWT>:<secret>`. If the run is repeated, add `--show` to print the
result again without re-cracking. Recovered secret: **`secret1`**.

> Use the full wordlist (`jwt.secrets.list` from the wallarm/jwt-secrets repository). A
> wordlist that is too small only reduces the search space; it does not confirm the result.

**Step 3 — Build the signing key in Burp.** Burp's Decoder was used to Base64-encode the
secret. The JWT Editor extension's **Keys** tab was then used to create a symmetric key
(`New Symmetric Key` → `Generate`), and the `k` value was replaced with the encoded secret.

```text
secret1          → Base64: c2VjcmV0MQ==
JWK "k" value    → c2VjcmV0MQ   (Base64url, no padding — see Notes)
```

**Step 4 — Forge the token.** In Repeater, the `GET /admin` request was switched to the
JWT Editor **JSON Web Tokens** tab:

- Payload: `"sub": "wiener"` → `"sub": "administrator"`
- Signed with the generated key via **Sign**, with `Don't modify header` selected so the
  `kid` and `alg` remain unchanged.

```http
GET /admin HTTP/2
Cookie: session=<JWT, sub=administrator, HS256 signed with secret1>
```

```http
HTTP/2 200 OK
<h1>Users</h1>
  wiener  - Delete  (/admin/delete?username=wiener)
  carlos  - Delete  (/admin/delete?username=carlos)
```

The admin panel loaded, which confirms the server accepted the forged signature.

**Step 5 — Delete the target user.** The forged session was reused to send the delete action:

```http
GET /admin/delete?username=carlos HTTP/2
Cookie: session=<JWT, sub=administrator, HS256 signed with secret1>
```

The lab reported **Congratulations, you solved the lab!**

## Impact

Full authentication bypass for any account. Once the HMAC secret is known, the attacker can
mint a valid session for any `sub` (here `administrator`) without knowing that user's
password. The admin functions, including deleting other users, then run under the forged
identity. Because the token verifies cryptographically, server-side logs show a normal
authenticated session.

## Remediation

- Use a **long, random secret** for HS256 — at least 256 bits from a CSPRNG (for example
  `openssl rand -base64 32`). Never use a dictionary word, a default, or a value copied from
  a tutorial.
- Store the secret in a secrets manager or environment variable, never in source code.
  Rotate it if there is any chance it was exposed; rotation invalidates all existing sessions.
- Where possible, use **asymmetric algorithms** (RS256/ES256) so verifiers hold only the
  public key and cannot forge tokens.
- Pin the expected algorithm on the server (allowlist) and set short `exp` lifetimes.
- Add monitoring for admin actions that happen without a matching login event.

## References

- **OWASP:** A07:2021 Identification and Authentication Failures
- **CWE:** CWE-347 (Improper Verification of Cryptographic Signature); CWE-326 (Inadequate Encryption Strength — weak key)
- **Hashcat:** mode 16500 (JWT)

## Notes

- **Why it works offline:** HS256 is symmetric. The attacker only needs one valid token and
  a guess to check; no request to the server is needed for each guess. Only a successful
  guess leaves any trace, and that trace is outside the target.
- **Recognise the candidate:** `"alg": "HS256"` in the header is the signal to try
  brute-forcing. An `RS256` token is not vulnerable to this attack, because the key is a
  private key and cannot be guessed.
- **JWK `k` encoding:** the JWK format expects Base64**url** without padding. Standard
  Base64 (`c2VjcmV0MQ==`) works here only because `secret1` produces no `+`, `/` or padding
  characters that differ between the two. In general, strip the `=` padding and replace
  `+` with `-` and `/` with `_`.
- **Keep the header unchanged:** choose `Don't modify header` when signing, so the `kid` and
  `alg` values stay as the server issued them. Only the payload changes.
- **Lab sequence:** 01 (signature not checked) → 02 (`alg: none` accepted) → 03 (weak key,
  this lab) → next: jwk, jku and kid header injection, where the attacker supplies the key
  material in the header rather than cracking it.
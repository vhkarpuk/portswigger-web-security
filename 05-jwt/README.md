# JWT

> **OWASP:** A07:2021 Identification and Authentication Failures · **CWE:** CWE-347

<2-3 sentences on how you think about this class of vulnerability — the embryo of a blog-style write-up. Optional, fill as you go.>

## Labs

| # | Lab | Level | Mechanism | Done |
|---|-----|-------|-----------|------|
| 01 | [JWT authentication bypass via unverified signature](01-jwt-authentication-bypass-via-unverified-signature.md) | Apprentice | Unverified signature | ⬜ |
| 02 | [JWT authentication bypass via flawed signature verification](02-jwt-authentication-bypass-via-flawed-signature-ver.md) | Apprentice | alg: "none" | ⬜ |
| 03 | [JWT authentication bypass via weak signing key](03-jwt-authentication-bypass-via-weak-signing-key.md) | Practitioner | Key brute-force (hashcat) | ⬜ |
| 04 | [JWT authentication bypass via jwk header injection](04-jwt-authentication-bypass-via-jwk-header-injection.md) | Practitioner | Key injection via jwk header | ⬜ |
| 05 | [JWT authentication bypass via jku header injection](05-jwt-authentication-bypass-via-jku-header-injection.md) | Practitioner | Injection via jku (key URL) | ⬜ |
| 06 | [JWT authentication bypass via kid header path traversal](06-jwt-authentication-bypass-via-kid-header-path-trav.md) | Practitioner | kid + path traversal | ⬜ |

# JWT

> **OWASP:** A07:2021 Identification and Authentication Failures Â· **CWE:** CWE-347

JSON Web Tokens carry authentication state the server must verify on every request, and the vulnerabilities come from verification that is weak or skipped. If the signature is not checked, relies on a guessable key, or trusts attacker-supplied header parameters, the token can be forged to impersonate any user. These labs cover unverified signatures, weak keys and the jwk, jku and kid header attacks.

## Labs

| # | Lab | Level | Mechanism | Done |
|---|-----|-------|-----------|------|
| 01 | [JWT authentication bypass via unverified signature](01-jwt-authentication-bypass-via-unverified-signature.md) | Apprentice | Unverified signature | â¬œ |
| 02 | [JWT authentication bypass via flawed signature verification](02-jwt-authentication-bypass-via-flawed-signature-ver.md) | Apprentice | alg: "none" | â¬œ |
| 03 | [JWT authentication bypass via weak signing key](03-jwt-authentication-bypass-via-weak-signing-key.md) | Practitioner | Key brute-force (hashcat) | â¬œ |
| 04 | [JWT authentication bypass via jwk header injection](04-jwt-authentication-bypass-via-jwk-header-injection.md) | Practitioner | Key injection via jwk header | â¬œ |
| 05 | [JWT authentication bypass via jku header injection](05-jwt-authentication-bypass-via-jku-header-injection.md) | Practitioner | Injection via jku (key URL) | â¬œ |
| 06 | [JWT authentication bypass via kid header path traversal](06-jwt-authentication-bypass-via-kid-header-path-trav.md) | Practitioner | kid + path traversal | â¬œ |

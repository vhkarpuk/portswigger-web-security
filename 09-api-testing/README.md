# API testing

> **OWASP:** OWASP API Security Top 10 2023 · **CWE:** CWE-285 / CWE-915 / CWE-650

API testing targets the gap between what an API exposes and what the client UI
actually uses. The server often trusts parameters, methods and endpoints that
the front end never calls, so the work is reconnaissance first: read the
documentation, diff the requests, probe for hidden endpoints and object
properties, then abuse what the application assumes no one will touch. These
labs cover documentation-driven abuse, hidden endpoints, server-side parameter
pollution and mass assignment.

## Labs

| # | Lab | Level | Mechanism | Done |
|---|-----|-------|-----------|------|
| 01 | [Exploiting an API endpoint using documentation](01-exploiting-api-endpoint-using-documentation.md) | Apprentice | API docs discovery -> unsafe method (DELETE/PATCH) | ✅ |
| 02 | [Exploiting server-side parameter pollution in a query string](02-exploiting-server-side-parameter-pollution-query-string.md) | Practitioner | SSPP (inject `&`/`#` into back-end query) | ✅ |
| 03 | [Finding and exploiting an unused API endpoint](03-finding-exploiting-unused-api-endpoint.md) | Apprentice | Hidden endpoint + method/verb probing | ✅ |
| 04 | [Exploiting a mass assignment vulnerability](04-exploiting-mass-assignment-vulnerability.md) | Practitioner | Mass assignment (hidden object fields) | ✅ |
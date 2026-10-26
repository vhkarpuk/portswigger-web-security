# Access control / IDOR

> **OWASP:** A01:2021 Broken Access Control · **CWE:** CWE-639 / CWE-284

<2-3 sentences on how you think about this class of vulnerability — the embryo of a blog-style write-up. Optional, fill as you go.>

## Labs

| # | Lab | Level | Mechanism | Done |
|---|-----|-------|-----------|------|
| 01 | [User ID controlled by request parameter](01-user-id-controlled-by-request-parameter.md) | Apprentice | Basic IDOR | ⬜ |
| 02 | [User ID controlled by request parameter, with unpredictable user IDs](02-user-id-controlled-by-request-parameter-with-unpre.md) | Apprentice | IDOR with unpredictable ID | ⬜ |
| 03 | [User ID controlled by request parameter with data leakage in redirect](03-user-id-controlled-by-request-parameter-with-data-.md) | Apprentice | Data leakage in redirect | ⬜ |
| 04 | [Insecure direct object references](04-insecure-direct-object-references.md) | Apprentice | IDOR on static file/resource | ⬜ |
| 05 | [URL-based access control can be circumvented](05-url-based-access-control-can-be-circumvented.md) | Practitioner | Bypass via X-Original-URL / X-Rewrite-URL | ⬜ |
| 06 | [Method-based access control can be circumvented](06-method-based-access-control-can-be-circumvented.md) | Practitioner | HTTP method swap | ⬜ |
| 07 | [Multi-step process with no access control on one step](07-multi-step-process-with-no-access-control-on-one-s.md) | Practitioner | Missing control on intermediate step | ⬜ |
| 08 | [Referer-based access control](08-referer-based-access-control.md) | Practitioner | Referer-based control | ⬜ |

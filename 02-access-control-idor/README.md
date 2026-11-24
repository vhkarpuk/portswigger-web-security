# Access control / IDOR

> **OWASP:** A01:2021 Broken Access Control · **CWE:** CWE-639 / CWE-284

Access control flaws occur when the application fails to enforce what a user is allowed to see or do, letting one user reach data or functions that belong to someone else. IDOR is the common shape: an object reference in the request (an id, a filename) is trusted without a server-side check that the current user owns that object. These labs cover horizontal access (reaching peer users) and vertical access (reaching higher privilege), plus the bypasses that defeat weak, request-controlled checks.

## Labs

| # | Lab | Level | Mechanism | Done |
|---|-----|-------|-----------|------|
| 01 | [User ID controlled by request parameter](01-user-id-controlled-by-request-parameter.md) | Apprentice | Basic IDOR | ✅ |
| 02 | [User ID controlled by request parameter, with unpredictable user IDs](02-user-id-controlled-by-request-parameter-with-unpre.md) | Apprentice | IDOR with unpredictable ID | ✅ |
| 03 | [User ID controlled by request parameter with data leakage in redirect](03-user-id-controlled-by-request-parameter-with-data-.md) | Apprentice | Data leakage in redirect | ✅ |
| 04 | [Insecure direct object references](04-insecure-direct-object-references.md) | Apprentice | IDOR on static file/resource | ✅ |
| 05 | [URL-based access control can be circumvented](05-url-based-access-control-can-be-circumvented.md) | Practitioner | Bypass via X-Original-URL / X-Rewrite-URL | ✅ |
| 06 | [Method-based access control can be circumvented](06-method-based-access-control-can-be-circumvented.md) | Practitioner | Switch POST to GET | ✅ |
| 07 | [Multi-step process with no access control on one step](07-multi-step-process-with-no-access-control-on-one-s.md) | Practitioner | Missing control on intermediate step | ✅ |
| 08 | [Referer-based access control](08-referer-based-access-control.md) | Practitioner | Spoofed Referer header | ✅ |
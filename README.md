# PortSwigger Web Security Academy — Lab Write-ups

Hands-on write-ups of PortSwigger Web Security Academy labs, focused on the
vulnerability classes that matter most for web application penetration testing.
Each write-up documents reconnaissance, exploitation (with raw HTTP request/response
as evidence), impact, and remediation, mapped to OWASP Top 10 and CWE.

**Scope:** 59 labs across 9 categories — the core web pentest surface.

## Progress

| Category | Done | OWASP |
|----------|------|-------|
| [Path traversal](01-path-traversal/) | 6 / 6 | A01:2021 Broken Access Control |
| [Access control / IDOR](02-access-control-idor/) | 5 / 8 | A01:2021 Broken Access Control |
| [SSRF](03-ssrf/) | 1 / 5 | A10:2021 SSRF |
| [File upload](04-file-upload/) | 0 / 6 | A05:2021 Security Misconfiguration |
| [JWT](05-jwt/) | 0 / 6 | A07:2021 Identification and Authentication Failures |
| [SQL injection](06-sql-injection/) | 2 / 9 | A03:2021 Injection |
| [Authentication](07-authentication/) | 0 / 7 | A07:2021 Identification and Authentication Failures |
| [Cross-site scripting](08-xss/) | 0 / 8 | A03:2021 Injection |
| [API testing](09-api-testing/) | 4 / 4 | OWASP API Security Top 10 2023 |
| **Total** | **18 / 59** | |

## Categories (attack order)

1. [Path traversal](01-path-traversal/)
2. [Access control / IDOR](02-access-control-idor/)
3. [SSRF](03-ssrf/)
4. [File upload](04-file-upload/)
5. [JWT](05-jwt/)
6. [SQL injection](06-sql-injection/)
7. [Authentication](07-authentication/)
8. [Cross-site scripting](08-xss/)
9. [API testing](09-api-testing/)

## How this repo is organised

- One folder per vulnerability class, numbered in the order I worked through them.
- One Markdown file per lab, following [`TEMPLATE.md`](TEMPLATE.md).
- Evidence is pasted as raw HTTP request/response in code blocks — searchable and
  faithful, the way a real client report reads.

## Author

Victor Karpuk — [github.com/vhkarpuk](https://github.com/vhkarpuk)

---

*Write-ups are my own. Labs belong to [PortSwigger Web Security Academy](https://portswigger.net/web-security).*
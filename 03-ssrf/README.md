# SSRF

> **OWASP:** A10:2021 SSRF Â· **CWE:** CWE-918

Server-side request forgery abuses functionality that makes the server fetch a URL, pointing it at internal systems the attacker cannot reach directly. The trust the back end places in its own network is the real vulnerability: localhost admin panels, cloud metadata endpoints and internal services all become reachable. These labs move from naive SSRF to blacklist bypasses and chaining with an open redirect.

## Labs

| # | Lab | Level | Mechanism | Done |
|---|-----|-------|-----------|------|
| 01 | [Basic SSRF against the local server](01-basic-ssrf-against-the-local-server.md) | Apprentice | Base SSRF (localhost/admin) | â¬œ |
| 02 | [Basic SSRF against another back-end system](02-basic-ssrf-against-another-back-end-system.md) | Apprentice | Pivot to internal back-end | â¬œ |
| 03 | [Blind SSRF with out-of-band detection](03-blind-ssrf-with-out-of-band-detection.md) | Practitioner | Blind + Burp Collaborator | â¬œ |
| 04 | [SSRF with blacklist-based input filter](04-ssrf-with-blacklist-based-input-filter.md) | Practitioner | Blacklist bypass (127.1, encoding) | â¬œ |
| 05 | [SSRF with filter bypass via open redirection vulnerability](05-ssrf-with-filter-bypass-via-open-redirection-vulne.md) | Practitioner | Chain with open redirect | â¬œ |

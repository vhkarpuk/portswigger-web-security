# Cross-site scripting

> **OWASP:** A03:2021 Injection · **CWE:** CWE-79

<2-3 sentences on how you think about this class of vulnerability — the embryo of a blog-style write-up. Optional, fill as you go.>

## Labs

| # | Lab | Level | Mechanism | Done |
|---|-----|-------|-----------|------|
| 01 | [Reflected XSS into attribute with angle brackets HTML-encoded](01-reflected-xss-into-attribute-with-angle-brackets-h.md) | Apprentice | Attribute context | ⬜ |
| 02 | [Reflected XSS into a JavaScript string with angle brackets HTML encoded](02-reflected-xss-into-a-javascript-string-with-angle-.md) | Apprentice | JS string context | ⬜ |
| 03 | [Reflected DOM XSS](03-reflected-dom-xss.md) | Practitioner | DOM reflected (source -> sink) | ⬜ |
| 04 | [Reflected XSS into HTML context with all tags blocked except custom ones](04-reflected-xss-into-html-context-with-all-tags-bloc.md) | Practitioner | Custom tags | ⬜ |
| 05 | [Reflected XSS with some SVG markup allowed](05-reflected-xss-with-some-svg-markup-allowed.md) | Practitioner | Bypass via SVG | ⬜ |
| 06 | [Exploiting cross-site scripting to steal cookies](06-exploiting-cross-site-scripting-to-steal-cookies.md) | Practitioner | Exploitation: cookie theft | ⬜ |
| 07 | [Reflected XSS into HTML context with most tags and attributes blocked](07-reflected-xss-into-html-context-with-most-tags-and.md) | Practitioner | Aggressive filter bypass | ⬜ |
| 08 | [Reflected XSS protected by very strict CSP, with dangling markup attack](08-reflected-xss-protected-by-very-strict-csp-with-da.md) | Practitioner | CSP + dangling markup | ⬜ |

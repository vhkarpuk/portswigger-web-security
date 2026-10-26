# Cross-site scripting

> **OWASP:** A03:2021 Injection Â· **CWE:** CWE-79

Cross-site scripting happens when user input is reflected into a page without proper output encoding, letting an attacker run JavaScript in the browser of another user. The real skill is not the payload but reading the injection context (HTML, attribute, JavaScript string, DOM sink) and defeating the filter in front of it. These labs cover the main contexts, filter and CSP bypasses, and turning XSS into impact such as cookie theft.

## Labs

| # | Lab | Level | Mechanism | Done |
|---|-----|-------|-----------|------|
| 01 | [Reflected XSS into attribute with angle brackets HTML-encoded](01-reflected-xss-into-attribute-with-angle-brackets-h.md) | Apprentice | Attribute context | â¬œ |
| 02 | [Reflected XSS into a JavaScript string with angle brackets HTML encoded](02-reflected-xss-into-a-javascript-string-with-angle-.md) | Apprentice | JS string context | â¬œ |
| 03 | [Reflected DOM XSS](03-reflected-dom-xss.md) | Practitioner | DOM reflected (source -> sink) | â¬œ |
| 04 | [Reflected XSS into HTML context with all tags blocked except custom ones](04-reflected-xss-into-html-context-with-all-tags-bloc.md) | Practitioner | Custom tags | â¬œ |
| 05 | [Reflected XSS with some SVG markup allowed](05-reflected-xss-with-some-svg-markup-allowed.md) | Practitioner | Bypass via SVG | â¬œ |
| 06 | [Exploiting cross-site scripting to steal cookies](06-exploiting-cross-site-scripting-to-steal-cookies.md) | Practitioner | Exploitation: cookie theft | â¬œ |
| 07 | [Reflected XSS into HTML context with most tags and attributes blocked](07-reflected-xss-into-html-context-with-most-tags-and.md) | Practitioner | Aggressive filter bypass | â¬œ |
| 08 | [Reflected XSS protected by very strict CSP, with dangling markup attack](08-reflected-xss-protected-by-very-strict-csp-with-da.md) | Practitioner | CSP + dangling markup | â¬œ |

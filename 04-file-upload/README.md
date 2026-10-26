# File upload

> **OWASP:** A05:2021 Security Misconfiguration Â· **CWE:** CWE-434

File upload vulnerabilities arise when the application accepts a file and later serves or executes it without verifying what it really is. The high-impact case is uploading a web shell and reaching it, turning a form field into remote code execution. These labs walk through the defences that try to stop this (Content-Type checks, extension blacklists, filename handling) and why each one can be bypassed.

## Labs

| # | Lab | Level | Mechanism | Done |
|---|-----|-------|-----------|------|
| 01 | [Remote code execution via web shell upload](01-remote-code-execution-via-web-shell-upload.md) | Apprentice | Web shell -> base RCE | â¬œ |
| 02 | [Web shell upload via Content-Type restriction bypass](02-web-shell-upload-via-content-type-restriction-bypa.md) | Apprentice | Content-Type bypass | â¬œ |
| 03 | [Web shell upload via path traversal](03-web-shell-upload-via-path-traversal.md) | Practitioner | Upload + path traversal in filename | â¬œ |
| 04 | [Web shell upload via extension blacklist bypass](04-web-shell-upload-via-extension-blacklist-bypass.md) | Practitioner | Blacklist bypass (.phtml / .htaccess) | â¬œ |
| 05 | [Web shell upload via obfuscated file extension](05-web-shell-upload-via-obfuscated-file-extension.md) | Practitioner | Null byte / double extension | â¬œ |
| 06 | [Remote code execution via polyglot web shell upload](06-remote-code-execution-via-polyglot-web-shell-uploa.md) | Practitioner | Polyglot (image + PHP) | â¬œ |

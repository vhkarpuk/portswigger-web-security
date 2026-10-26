# Authentication

> **OWASP:** A07:2021 Identification and Authentication Failures Â· **CWE:** CWE-287

Authentication vulnerabilities are flaws in how the application verifies identity: weak brute-force protection, broken multi-factor logic, guessable persistent tokens and flawed password-reset flows. The impact is direct account takeover, which is why these are among the first things to test. These labs cover 2FA bypasses, brute-force protection bypasses, offline password cracking and reset-logic flaws.

## Labs

| # | Lab | Level | Mechanism | Done |
|---|-----|-------|-----------|------|
| 01 | [2FA simple bypass](01-2fa-simple-bypass.md) | Apprentice | 2FA bypass (direct access post-2FA) | â¬œ |
| 02 | [Password reset broken logic](02-password-reset-broken-logic.md) | Apprentice | Broken reset logic | â¬œ |
| 03 | [Broken brute-force protection, IP block](03-broken-brute-force-protection-ip-block.md) | Practitioner | IP block bypass (X-Forwarded-For) | â¬œ |
| 04 | [2FA broken logic](04-2fa-broken-logic.md) | Practitioner | 2FA poorly bound to session | â¬œ |
| 05 | [Brute-forcing a stay-logged-in cookie](05-brute-forcing-a-stay-logged-in-cookie.md) | Practitioner | Brute-force "remember me" cookie | â¬œ |
| 06 | [Offline password cracking](06-offline-password-cracking.md) | Practitioner | Offline cracking (hashcat) | â¬œ |
| 07 | [Password reset poisoning via middleware](07-password-reset-poisoning-via-middleware.md) | Practitioner | Reset poisoning via header (optional) | â¬œ |

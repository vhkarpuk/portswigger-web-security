# Reflected XSS into a JavaScript string with angle brackets HTML encoded

**Category:** Cross-site scripting
**Level:** Apprentice
**Mechanism:** JavaScript-string context — break out of the string literal

**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/contexts/lab-javascript-string-angle-brackets-html-encoded

---

## Vulnerability

The search term is reflected **inside a JavaScript string literal** in an inline
`<script>` block. Angle brackets are HTML-encoded, so a `<script>` tag can't be
injected — but the single quote that delimits the JS string is **not** encoded, so the
input can close the string and inject JavaScript that the browser then executes as code.

## Reconnaissance & Exploitation

Searching for a marker showed it reflected inside a JS string, something like:

```html
<script>
  var searchTerms = 'MARKER';
  ...
</script>
```

`<` and `>` are encoded, so breaking out into HTML fails. The single quote is reflected
literally, so I close the string, run my code, and reopen the string to keep the
statement syntactically valid:

```
'-alert(1)-'
```

Reflected, the line becomes:

```javascript
var searchTerms = ''-alert(1)-'';
```

This parses as string `''` minus `alert(1)` minus string `''` — a valid JS expression,
so `alert(1)` runs. The request carries it URL-encoded in `search`:

```http
GET /?search='-alert(1)-' HTTP/2
Host: 0a3b00a703c550c5800a034200d7008b.web-security-academy.net
```

Loading the URL fired `alert(1)` — lab solved. No angle brackets were needed; breaking
out of the JS string was enough.

## Impact

Reflected XSS: an attacker crafts a URL with the payload and lures a victim to open it,
running arbitrary JavaScript in the victim's session on this origin — session-cookie
theft, actions as the victim, credential capture, keylogging. Encoding the angle
brackets gave false confidence while the string delimiter (`'`) was left unencoded,
leaving the sink exploitable.

## Remediation

- Context-aware encoding: for a value placed inside a JS string, JavaScript-escape it
  (`\'`, `\"`, `\\`, and ideally hex-escape non-alphanumerics), not just HTML-encode
  angle brackets.
- Better: don't inject user data into inline script at all. Pass it via a data attribute
  or `JSON.stringify` into a safe sink, or read it client-side from a non-executable
  location.
- Defence in depth: a CSP that forbids inline script execution would block this even if
  the reflection occurs.

## References

- **OWASP:** A03:2021 Injection (Cross-site Scripting)
- **CWE:** CWE-79 (Improper Neutralization of Input During Web Page Generation)

## Notes

- Context decides the break-out char again: HTML body → `<`; quoted attribute (lab 01)
  → `"`; **JS string → `'`**. Identify where the reflection lands first.
- `-alert(1)-` instead of `;alert(1)//`: the `-` makes the whole line a valid arithmetic
  expression (`'' - alert(1) - ''`), so it executes without a syntax error and without
  needing to comment out the rest. A clean, reliable JS-string breakout pattern.
- Same incomplete-filter theme: encoding one metacharacter (`<`/`>`) while leaving the
  context's real delimiter (`'`) unescaped is the bug.
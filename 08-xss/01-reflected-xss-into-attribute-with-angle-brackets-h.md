# Reflected XSS into attribute with angle brackets HTML-encoded

**Category:** Cross-site scripting
**Level:** Apprentice
**Mechanism:** Attribute-context XSS — break out of a quoted attribute with an event handler

**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/contexts/lab-attribute-angle-brackets-html-encoded

---

## Vulnerability

The blog search reflects the query into the page. Angle brackets (`<`, `>`) are
HTML-encoded, so a `<script>` tag can't be injected. But the input is reflected **inside
a quoted HTML attribute** and the double quote is **not** encoded — so the quote can
close the attribute early and a new event-handler attribute can be injected, running
JavaScript without any `<` or `>`.

## Reconnaissance & Exploitation

Searching for a marker string showed it reflected inside a quoted attribute, something
like:

```html
<input ... value="MARKER">
```

`<` and `>` come back HTML-encoded, so breaking out with a tag fails. The double quote,
however, is reflected literally — so I close the `value` attribute and add an
`onmouseover` handler:

```
"onmouseover="alert(1)
```

Reflected, the attribute becomes:

```html
<input ... value="" onmouseover="alert(1)">
```

The request carries it URL-encoded in the `search` parameter:

```http
GET /?search="onmouseover%3D"alert%281%29 HTTP/2
Host: 0ab4008e03cd6681827c29e50053008b.web-security-academy.net
```

Loading the URL and moving the mouse over the field fired `alert(1)` — lab solved. No
angle brackets were needed; breaking the quoted attribute and adding an event handler
was enough.

## Impact

Reflected XSS: an attacker crafts a URL with the payload and gets a victim to open it
(phishing link, etc.), running arbitrary JavaScript in the victim's session on this
origin. That means session-cookie theft, actions performed as the victim, credential
capture via fake prompts, or keylogging — full control of the victim's interaction with
the site. The encoding of `<`/`>` gave false confidence; leaving the quote unencoded
left the hole wide open.

## Remediation

- Context-aware output encoding: inside a double-quoted attribute, HTML-encode the
  double quote (`&quot;`) as well as `<`/`>`. Encoding only angle brackets is
  insufficient for attribute context.
- Prefer a framework/templating engine that auto-encodes by the output context rather
  than hand-rolled escaping.
- Defence in depth: a Content-Security-Policy that blocks inline event handlers would
  stop this even if the injection reflects.

## References

- **OWASP:** A03:2021 Injection (Cross-site Scripting)
- **CWE:** CWE-79 (Improper Neutralization of Input During Web Page Generation)

## Notes

- Context decides the payload. Here the sink is a **quoted attribute**, so the break-out
  char is `"`, not `<`. Always identify *where* the reflection lands (HTML body,
  attribute, JS string, URL) before choosing a payload.
- Encoding one dangerous char (`<`/`>`) while leaving another (`"`) is the classic
  incomplete-filter bug — test every metacharacter for the specific context.
- An event handler (`onmouseover`, `onfocus` + `autofocus`, `onload`) runs script with
  no tags at all — the go-to when angle brackets are filtered but attribute injection
  is possible.
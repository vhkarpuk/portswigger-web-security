# Reflected XSS into HTML context with most tags and attributes blocked

**Category:** Cross-site scripting
**Level:** Practitioner
**Mechanism:** WAF bypass — enumerate the one allowed tag/event (`body`/`onresize`), auto-trigger via a self-resizing iframe

**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/contexts/lab-html-context-with-most-tags-and-attributes-blocked

---

## Vulnerability

The search term is reflected into HTML, fronted by a WAF that blocks almost every tag and
event handler. A tiny allowed set survives — the `<body>` tag and the `onresize` event — and
`onresize` can be fired with **no user interaction** by loading the payload in an iframe whose
size is changed on load. Combining the surviving tag, the surviving event, and the iframe
resize trick gives script execution past the WAF.

## Reconnaissance & Exploitation

A standard vector is blocked by the WAF:

```
<img src=1 onerror=print()>   →   400
```

Enumerated the WAF with Intruder instead of guessing. **Which tags pass** — search value
`<§§>`, payloads = the tag list from the XSS cheat sheet:

```
GET /?search=<§§> HTTP/2
```

Nearly all returned `400`; only **`body`** returned `200`.

**Which events pass** on `<body>` — search value `<body%20§§=1>`, payloads = the event list:

```
GET /?search=<body%20§§=1> HTTP/2
```

Only **`onresize`** returned `200`.

So the usable primitive is `<body onresize=print()>`. But `onresize` needs a resize event, and
the lab forbids user interaction. The trick: host the payload in an **iframe** and resize the
iframe on load — the resize propagates to the framed page and fires `onresize`. Delivered via
the exploit server:

```html
<iframe src="https://0a6e0043035a5e3e803112df003b001c.web-security-academy.net/?search=%22%3E%3Cbody%20onresize=print()%3E" onload=this.style.width='100px'>
```

- `src` points at the **lab** with the payload (`">` breaks out, then `<body onresize=print()>`).
- `onload=this.style.width='100px'` changes the iframe's width once it loads, firing a resize.

`Store` → `Deliver exploit to victim`: the victim's browser loads the iframe, the iframe
resizes itself on load, the framed page's `onresize` fires, and `print()` runs with no
interaction — lab solved.

## Impact

Reflected XSS delivered to a victim with zero interaction, past a WAF. Arbitrary JavaScript
runs in the victim's session on the lab origin (here `print()`, but equally cookie theft,
actions as the victim, credential capture). The WAF's tag/attribute blacklist created false
assurance while leaving an exploitable tag and an auto-firing event reachable.

## Remediation

- Don't rely on a WAF blacklist as the control. Fix the sink: **context-aware output encoding**
  so the reflected value renders as text, never markup.
- If HTML is needed, sanitise with a vetted allowlist (DOMPurify) that strips event-handler
  attributes regardless of tag.
- Add a CSP forbidding inline event handlers, and set `X-Frame-Options`/`frame-ancestors` so the
  page can't be framed and resize-triggered from an attacker origin.

## References

- **OWASP:** A03:2021 Injection
- **CWE:** CWE-79 (Improper Neutralization of Input During Web Page Generation); CWE-184 (Incomplete List of Disallowed Inputs)

## Notes

- Same two-pass Intruder methodology as the SVG lab (05): enumerate allowed tags (`<§§>`),
  then allowed events on a surviving tag (`<body %§§=1>`). Here the WAF left exactly one of
  each: `body` and `onresize`.
- The no-interaction requirement is the real lesson: `onresize` normally needs a human to
  resize the window, but an **iframe that resizes itself on load** fires it automatically. Go-to
  trick whenever the only allowed event needs a size/scroll/focus change.
- Two different hosts, easy to mix up (this cost me a failed attempt): the iframe `src` must be
  the **lab** host; the iframe itself is hosted on the **exploit server** host. Pointing `src` at
  a stale/wrong lab ID = nothing fires. Also: must **Deliver to victim** — self-triggering
  `print()` in your own browser doesn't count.
- WAF/blacklist bypass is the recurring theme: the allowed set is always bigger than the blocked
  list, and one reachable tag + one auto-firing event is enough.
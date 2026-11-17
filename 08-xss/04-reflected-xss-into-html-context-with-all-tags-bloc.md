# Reflected XSS into HTML context with all tags blocked except custom ones

**Category:** Cross-site scripting
**Level:** Practitioner
**Mechanism:** Custom HTML tag + `onfocus` auto-triggered by a URL fragment (`#id`)

**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/contexts/lab-html-context-with-all-standard-tags-blocked

---

## Vulnerability

The search term is reflected into the HTML body, and the filter blocks every **standard**
HTML tag (`<script>`, `<img>`, `<svg>`, etc.) but not **custom** tags — any made-up element
name passes. A custom element still accepts event-handler attributes, so an injected custom
tag with an `onfocus` handler runs JavaScript, and the handler is fired automatically by
pointing the URL fragment at the element's `id`.

## Reconnaissance & Exploitation

A marker in `search` is reflected straight into the page's HTML. Standard tags are
stripped, but a custom tag name is allowed through, so the injection is a custom element
carrying an event handler:

```html
<custom-tag onfocus='alert(document.cookie)' id='x' tabindex='1'>
```

- `custom-tag` — not a standard tag, so the filter lets it through.
- `tabindex='1'` — makes the element focusable (normally a custom tag can't receive focus).
- `id='x'` — gives it an anchor target.
- `onfocus='alert(document.cookie)'` — the sink, fired when the element gains focus.

Delivered as a URL whose fragment (`#x`) points the browser at the element: on load, the
browser scrolls to and **focuses** `#x`, firing `onfocus` with no user interaction. The
reflected request:

```
GET /?search=<custom-tag onfocus='alert(document.cookie)' id='x' tabindex='1'>
```

URL-encoded in the exploit page, which redirects the victim and appends `#x`:

```html
<script>
location = 'https://0aec00220375c9fe80080dab007f00df.web-security-academy.net/?search=%3Ccustom-tag+onfocus%3D%27alert%28document.cookie%29%27+id%3D%27x%27+tabindex%3D%271%27%3E#x';
</script>
```

Stored on the exploit server and delivered to the victim, the page loaded, the fragment
focused the custom tag, and `alert(document.cookie)` fired — lab solved.

## Impact

Reflected XSS against a victim who opens an attacker-supplied link: arbitrary JavaScript
runs in the victim's session on this origin. Here it reads `document.cookie`, enabling
session hijacking; the same primitive allows actions as the victim, credential capture and
keylogging. A tag-name blacklist created false confidence — blocking known-dangerous tags
leaves the entire space of custom elements, which still execute event handlers.

## Remediation

- Don't blacklist tag names. Use context-aware **output encoding** so reflected input is
  rendered as text, not parsed as markup — the single reliable fix.
- If HTML must be allowed, use a vetted sanitiser (e.g. DOMPurify) with an **allowlist** of
  safe elements and attributes, which strips event-handler attributes regardless of tag name.
- Add a Content-Security-Policy that forbids inline event handlers (`script-src` without
  `unsafe-inline`), so an injected `onfocus` won't run even if markup slips through.

## References

- **OWASP:** A03:2021 Injection
- **CWE:** CWE-79 (Improper Neutralization of Input During Web Page Generation); CWE-184 (Incomplete List of Disallowed Inputs)

## Notes

- The auto-trigger is the trick: an event handler needs an event. `#id` in the URL makes the
  browser focus that element on load, firing `onfocus` with zero interaction. `tabindex` is
  what makes a non-interactive/custom element focusable in the first place.
- Blacklisting tags is the same failed model as elsewhere in this repo (path traversal, SSRF,
  file upload): enumerating the bad instead of defining the good. Custom elements are an
  unbounded set, so the list can never be complete.
- Other auto-fire handlers worth remembering when a tag is allowed but interaction isn't:
  `onfocus`+autofocus/`#id`, `onanimationstart` with a CSS animation, `onload` on media/SVG.
- Delivered via the exploit server because the payload must land in the *victim's* browser;
  the `<script>location=...</script>` just navigates them to the crafted search URL.
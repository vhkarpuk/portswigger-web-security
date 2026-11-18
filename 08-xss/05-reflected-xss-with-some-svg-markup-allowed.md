# Reflected XSS with some SVG markup allowed

**Category:** Cross-site scripting
**Level:** Practitioner
**Mechanism:** Blacklist misses SVG tags/events — enumerate the gaps and fire `onbegin`

**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/contexts/lab-some-svg-markup-allowed

---

## Vulnerability

The search term is reflected into HTML, and a blacklist blocks the common XSS tags and
event handlers. The blacklist is incomplete: a handful of **SVG** tags and one SVG
animation event slip through. Finding the gap (which tag and which event survive) and
combining them produces script execution.

## Reconnaissance & Exploitation

A standard payload is rejected:

```
<img src=1 onerror=alert(1)>   →   400 "Tag is not allowed"
```

Rather than guess, I enumerated the blacklist with Intruder. First, which **tags** pass —
search value set to `<§§>` (payload position between the brackets), payloads = the full tag
list from the XSS cheat sheet (sniper, 143 tags):

```
GET /?search=<§§> HTTP/1.1
```

Almost everything returned `400`; only a few returned `200`: `svg`, `animatetransform`,
`title`, `image`. So SVG tags survive the filter.

Next, which **event attributes** pass on a surviving tag — search value set to
`<svg><animatetransform%20§§=1>` (payload position at the attribute name), payloads = the
full event list from the cheat sheet (145 events):

```
GET /?search=<svg><animatetransform%20§§=1> HTTP/1.1
```

Only one returned `200`: **`onbegin`** (an SVG animation event that fires when the
animation starts). Combining the allowed tag and the allowed event, breaking out of the
reflected attribute context first:

```
"><svg><animatetransform onbegin=alert(1)>
```

Loaded as the search parameter:

```
https://0a81004f040ae5d083f6b9f80021008d.web-security-academy.net/?search=%22%3E%3Csvg%3E%3Canimatetransform%20onbegin=alert(1)%3E
```

`onbegin` fired on load and `alert(1)` ran — lab solved. No user interaction needed; the
SVG animation begins automatically.

## Impact

Reflected XSS via a crafted URL: arbitrary JavaScript runs in the victim's session on this
origin (cookie theft, actions as the victim, credential capture). The blacklist blocked the
obvious vectors but left SVG tags and the `onbegin` animation event reachable — enough for
full execution.

## Remediation

- Don't blacklist tags/attributes. Use context-aware **output encoding** so the reflected
  value renders as text, never markup — the only reliable fix.
- If HTML is required, sanitise with a vetted allowlist library (DOMPurify) that understands
  SVG and strips animation/event attributes regardless of tag.
- Add a CSP that forbids inline event handlers, so an `onbegin` won't execute even if markup
  slips through.

## References

- **OWASP:** A03:2021 Injection
- **CWE:** CWE-79 (Improper Neutralization of Input During Web Page Generation); CWE-184 (Incomplete List of Disallowed Inputs)

## Notes

- Methodology over payload: when a blacklist is in play, **enumerate it** instead of
  guessing. Two Intruder passes — tags first (`<§§>`), then events on a surviving tag
  (`<svg><animatetransform %§§=1>`) — map exactly what's allowed. 143 + 145 requests, done in
  Community (slow but fine; no Collaborator needed).
- `onbegin` is the key: it's an SVG SMIL animation event that fires automatically when the
  animation starts, so no click/focus is required — unlike `onclick`/`onmouseover`.
- Allowed tags here were `svg`, `animatetransform`, `title`, `image`; the winning event was
  `onbegin`. Worth remembering as a go-to when common tags are filtered.
- Same lesson as the custom-tag lab (04): an incomplete blacklist is not a control. The
  allowed set is always larger than the list of blocked items.
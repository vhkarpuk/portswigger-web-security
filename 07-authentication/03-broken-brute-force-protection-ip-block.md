# Broken brute-force protection, IP block

**Category:** Authentication
**Level:** Practitioner
**Mechanism:** Failed-login counter reset by a successful login — interleave valid logins to brute-force

**Lab URL:** https://portswigger.net/web-security/authentication/password-based/lab-broken-brute-force-protection-ip-block

---

## Vulnerability

The login blocks an IP after 3 consecutive failed attempts, but the failure counter is
**reset by any successful login** — not just by a cooldown. Because the attacker owns a
valid account, they can insert their own successful logins between the guesses against the
victim, keeping the consecutive-failure count below the threshold indefinitely and
brute-forcing without ever being blocked.

## Reconnaissance & Exploitation

Submitting 3 wrong passwords in a row returned:

```
You have made too many incorrect login attempts. Please try again in 1 minute(s).
```

The protection is **consecutive failures per IP**, and logging into my own account
(`wiener:peter`) between failures reset the counter back to zero. That turns the block
into a speed bump: alternate a valid login with each victim guess and the count never
reaches 3.

Captured the login request and sent `POST /login` to Intruder as a **pitchfork** attack,
with payload markers on both fields:

```http
POST /login HTTP/2
Host: 0a7600b4037df298824724a400dc003e.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

username=§carlos§&password=§peter§
```

Pitchfork advances both payload lists in lockstep (request *i* = username[i] + password[i]),
so I built two **aligned** lists that interleave a valid reset before every victim guess:

```
username list      password list
wiener             peter        <- valid login, resets counter to 0
carlos             123456       <- guess (fail count = 1)
wiener             peter        <- reset
carlos             password     <- guess
...                ...
```

Two settings made it work:
- **Resource pool → Maximum concurrent requests = 1.** Single-threaded sending preserves
  the order; concurrent requests race and 3 `carlos` can land before a `wiener`, tripping
  the block. (A first run with 2 guesses between resets *did* block mid-attack — the
  response length changed from 3487 to 3424, the "too many attempts" page, masking the
  real password. The 1-to-1 interleave fixed it.)
- **wiener first**, to clear any pre-existing count from manual testing.

Failed logins return `200` (the login page re-rendered with "Incorrect password").
A successful login returns `302` (redirect to `/my-account`). Filtering for the single
`carlos` row with a `302`:

```
Request  Payload 1   Payload 2    Status   Length
24       carlos      baseball     302      63
```

carlos's password is **`baseball`**. Logging in as `carlos:baseball` and opening
`/my-account` solved the lab.

## Impact

The rate-limit provides no real protection against a motivated attacker: anyone with a
single valid account (even a self-registered one) can brute-force any other account at
will by interleaving their own logins. Credential-stuffing and password-guessing attacks
proceed unthrottled, leading to account takeover — including of high-value accounts —
despite the appearance of brute-force defence.

## Remediation

- Do not reset the failed-attempt counter on a successful login to a **different**
  account. Track failures **per targeted username**, not only per IP, so guesses against
  `carlos` are counted regardless of successful logins to `wiener`.
- Apply protection keyed to the account being attacked: progressive delays or temporary
  lockout on the victim account after N failures, independent of other sessions.
- Combine IP-based and account-based throttling, and add CAPTCHA / step-up after repeated
  failures. Rate-limiting on IP alone is trivially bypassed (proxies, `X-Forwarded-For`,
  or this reset trick).

## References

- **OWASP:** A07:2021 Identification and Authentication Failures
- **CWE:** CWE-307 (Improper Restriction of Excessive Authentication Attempts); CWE-799 (Improper Control of Interaction Frequency)

## Notes

- The flaw is *what* resets the counter: a success on **any** account clears it, so the
  attacker's own valid login neutralises the defence for the victim.
- Pitchfork (not cluster bomb) because the two lists must stay aligned 1:1 — index *i* of
  each list is paired, giving `wiener/peter` resets interleaved with `carlos/<guess>`.
- Concurrency = 1 is not optional here: order *is* the exploit. Out-of-order sending let 3
  `carlos` failures stack and triggered the block, after which every guess returned the
  "too many attempts" page (status 200, different length) and the real password was never
  actually checked.
- Reading the results: `200` = failed login (page re-rendered), `302` = success (redirect).
  Sort by Status/Length and look for the lone `302` on a `carlos` row.
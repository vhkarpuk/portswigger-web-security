# SQL injection

> **OWASP:** A03:2021 Injection · **CWE:** CWE-89

SQL injection happens when user input is concatenated into a query instead of being
parameterised, letting an attacker change the query the database runs. Impact ranges
from authentication bypass to full extraction of the database, and often to the
underlying host. These labs cover UNION-based extraction, blind techniques (boolean and
time-based) and a WAF filter bypass.

> Note: the "Blind SQL injection with out-of-band interaction" lab requires Burp Suite
> Professional (Burp Collaborator) and is not solvable in Community Edition, so it is
> out of scope here.

## Labs

| # | Lab | Level | Mechanism | Done |
|---|-----|-------|-----------|------|
| 01 | [UNION attack, determining the number of columns](01-sql-injection-union-attack-determining-the-number-.md) | Practitioner | No. of columns (ORDER BY / nulls) | ✅ |
| 02 | [UNION attack, finding a column containing text](02-sql-injection-union-attack-finding-a-column-contai.md) | Practitioner | Column that accepts a string | ✅ |
| 03 | [UNION attack, retrieving data from other tables](03-sql-injection-union-attack-retrieving-data-from-ot.md) | Practitioner | In-band extraction via UNION | ✅ |
| 04 | [Listing the database contents on non-Oracle databases](04-sql-injection-attack-listing-the-database-contents.md) | Practitioner | Schema enumeration (information_schema) | ✅ |
| 05 | [Blind SQL injection with conditional responses](05-blind-sql-injection-with-conditional-responses.md) | Practitioner | Boolean blind | ✅ |
| 06 | [Visible error-based SQL injection](06-visible-error-based-sql-injection.md) | Practitioner | Error-based | ⬜ |
| 07 | [Blind SQL injection with time delays and information retrieval](07-blind-sql-injection-with-time-delays-and-informati.md) | Practitioner | Time-based blind | ⬜ |
| 08 | [SQL injection with filter bypass via XML encoding](08-sql-injection-with-filter-bypass-via-xml-encoding.md) | Practitioner | WAF bypass via XML encoding | ⬜ |
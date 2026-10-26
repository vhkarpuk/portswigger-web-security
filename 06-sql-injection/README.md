# SQL injection

> **OWASP:** A03:2021 Injection · **CWE:** CWE-89

<2-3 sentences on how you think about this class of vulnerability — the embryo of a blog-style write-up. Optional, fill as you go.>

## Labs

| # | Lab | Level | Mechanism | Done |
|---|-----|-------|-----------|------|
| 01 | [SQL injection UNION attack, determining the number of columns returned by the query](01-sql-injection-union-attack-determining-the-number-.md) | Practitioner | No. of columns (ORDER BY / nulls) | ⬜ |
| 02 | [SQL injection UNION attack, finding a column containing text](02-sql-injection-union-attack-finding-a-column-contai.md) | Practitioner | Column that accepts a string | ⬜ |
| 03 | [SQL injection UNION attack, retrieving data from other tables](03-sql-injection-union-attack-retrieving-data-from-ot.md) | Practitioner | In-band extraction via UNION | ⬜ |
| 04 | [SQL injection attack, listing the database contents on non-Oracle databases](04-sql-injection-attack-listing-the-database-contents.md) | Practitioner | Schema enumeration | ⬜ |
| 05 | [Blind SQL injection with conditional responses](05-blind-sql-injection-with-conditional-responses.md) | Practitioner | Boolean blind | ⬜ |
| 06 | [Visible error-based SQL injection](06-visible-error-based-sql-injection.md) | Practitioner | Error-based | ⬜ |
| 07 | [Blind SQL injection with time delays and information retrieval](07-blind-sql-injection-with-time-delays-and-informati.md) | Practitioner | Time-based blind | ⬜ |
| 08 | [Blind SQL injection with out-of-band interaction](08-blind-sql-injection-with-out-of-band-interaction.md) | Practitioner | OOB (Collaborator / DNS) | ⬜ |
| 09 | [SQL injection with filter bypass via XML encoding](09-sql-injection-with-filter-bypass-via-xml-encoding.md) | Practitioner | WAF bypass via XML encoding | ⬜ |

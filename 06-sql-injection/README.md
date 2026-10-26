# SQL injection

> **OWASP:** A03:2021 Injection Â· **CWE:** CWE-89

SQL injection happens when user input is concatenated into a query instead of being parameterised, letting an attacker change the query the database runs. Impact ranges from authentication bypass to full extraction of the database, and often to the underlying host. These labs cover UNION-based extraction, blind techniques (boolean, time-based and out-of-band) and a WAF filter bypass.

## Labs

| # | Lab | Level | Mechanism | Done |
|---|-----|-------|-----------|------|
| 01 | [SQL injection UNION attack, determining the number of columns returned by the query](01-sql-injection-union-attack-determining-the-number-.md) | Practitioner | No. of columns (ORDER BY / nulls) | â¬œ |
| 02 | [SQL injection UNION attack, finding a column containing text](02-sql-injection-union-attack-finding-a-column-contai.md) | Practitioner | Column that accepts a string | â¬œ |
| 03 | [SQL injection UNION attack, retrieving data from other tables](03-sql-injection-union-attack-retrieving-data-from-ot.md) | Practitioner | In-band extraction via UNION | â¬œ |
| 04 | [SQL injection attack, listing the database contents on non-Oracle databases](04-sql-injection-attack-listing-the-database-contents.md) | Practitioner | Schema enumeration | â¬œ |
| 05 | [Blind SQL injection with conditional responses](05-blind-sql-injection-with-conditional-responses.md) | Practitioner | Boolean blind | â¬œ |
| 06 | [Visible error-based SQL injection](06-visible-error-based-sql-injection.md) | Practitioner | Error-based | â¬œ |
| 07 | [Blind SQL injection with time delays and information retrieval](07-blind-sql-injection-with-time-delays-and-informati.md) | Practitioner | Time-based blind | â¬œ |
| 08 | [Blind SQL injection with out-of-band interaction](08-blind-sql-injection-with-out-of-band-interaction.md) | Practitioner | OOB (Collaborator / DNS) | â¬œ |
| 09 | [SQL injection with filter bypass via XML encoding](09-sql-injection-with-filter-bypass-via-xml-encoding.md) | Practitioner | WAF bypass via XML encoding | â¬œ |

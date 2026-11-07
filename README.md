# PortSwigger Web Security Academy — Lab Writeups

Bu depo, PortSwigger Web Security Academy lab'lerini çözerken aldığım notların ve writeup'ların derlemesidir. Her lab için Türkçe (`.tr.md`) ve İngilizce (`.en.md`) ayrı dosya bulunur.

> This repository collects my writeups for PortSwigger Web Security Academy labs. Each lab has a Turkish (`.tr.md`) and an English (`.en.md`) version.

## SQL Injection

| # | Lab | Seviye / Level | TR | EN |
|---|-----|----------------|----|----|
| 01 | SQL injection vulnerability in WHERE clause allowing retrieval of hidden data | Apprentice | [TR](sql-injection/01-where-clause-hidden-data.tr.md) | [EN](sql-injection/01-where-clause-hidden-data.en.md) |
| 02 | SQL injection vulnerability allowing login bypass | Apprentice | [TR](sql-injection/02-login-bypass.tr.md) | [EN](sql-injection/02-login-bypass.en.md) |
| 03 | SQL injection UNION attack, determining the number of columns returned by the query | Practitioner | [TR](sql-injection/03-union-sutun-sayisi.tr.md) | [EN](sql-injection/03-union-column-count.en.md) |
| 04 | SQL injection UNION attack, finding a column containing text | Practitioner | [TR](sql-injection/04-union-text-sutunu.tr.md) | [EN](sql-injection/04-union-text-column.en.md) |
| 05 | SQL injection UNION attack, retrieving data from other tables | Practitioner | [TR](sql-injection/05-union-diger-tablolar.tr.md) | [EN](sql-injection/05-union-other-tables.en.md) |
| 06 | SQL injection UNION attack, retrieving multiple values in a single column | Practitioner | [TR](sql-injection/06-union-tek-sutun-coklu-deger.tr.md) | [EN](sql-injection/06-union-multiple-values-single-column.en.md) |
| 07 | SQL injection attack, querying the database type and version on Oracle | Practitioner | [TR](sql-injection/07-oracle-versiyon.tr.md) | [EN](sql-injection/07-oracle-version.en.md) |
| 08 | SQL injection attack, querying the database type and version on MySQL and Microsoft | Practitioner | [TR](sql-injection/08-mysql-mssql-versiyon.tr.md) | [EN](sql-injection/08-mysql-mssql-version.en.md) |
| 09 | SQL injection attack, listing the database contents on non-Oracle databases | Practitioner | [TR](sql-injection/09-veritabani-icerigi-listeleme.tr.md) | [EN](sql-injection/09-listing-database-contents.en.md) |
| 10 | SQL injection attack, listing the database contents on Oracle | Practitioner | [TR](sql-injection/10-oracle-veritabani-icerigi.tr.md) | [EN](sql-injection/10-oracle-listing-database-contents.en.md) |


## Path Traversal

| # | Lab | Seviye / Level | TR | EN |
|---|-----|----------------|----|----|
| 01 | File path traversal, simple case | Apprentice | [TR](path-traversal/01-basit-path-traversal.tr.md) | [EN](path-traversal/01-simple-case.en.md) |
| 02 | File path traversal, traversal sequences blocked with absolute path bypass | Practitioner | [TR](path-traversal/02-absolute-path-bypass.tr.md) | [EN](path-traversal/02-absolute-path-bypass.en.md) |
| 03 | File path traversal, traversal sequences stripped non-recursively | Practitioner | [TR](path-traversal/03-non-recursive-stripping.tr.md) | [EN](path-traversal/03-non-recursive-stripping.en.md) |
| 04 | File path traversal, traversal sequences stripped with superfluous URL-decode | Practitioner | [TR](path-traversal/04-superfluous-url-decode.tr.md) | [EN](path-traversal/04-superfluous-url-decode.en.md) |
| 05 | File path traversal, validation of start of path | Practitioner | [TR](path-traversal/05-start-of-path-validation.tr.md) | [EN](path-traversal/05-start-of-path-validation.en.md) |
| 06 | File path traversal, validation of file extension with null byte bypass | Practitioner | [TR](path-traversal/06-null-byte-bypass.tr.md) | [EN](path-traversal/06-null-byte-bypass.en.md) |

# SQL Injection — Querying the Database Type and Version (MySQL and Microsoft)

**Lab:** SQL injection attack, querying the database type and version on MySQL and Microsoft
**Level:** Practitioner
**Category:** SQL Injection (UNION / information gathering)
**Tools:** Burp Suite (Proxy + Repeater), browser
**Lab link:** https://portswigger.net/web-security/sql-injection/examining-the-database/lab-querying-database-version-mysql-microsoft

---

## Vulnerability Summary

The product category filter is vulnerable to SQL injection and the result of the injected query is shown on the page, so we can use a `UNION` attack to display the database **version string**. The goal: display the version information.

This lab is the MySQL/Microsoft (MSSQL) counterpart of the previous Oracle lab. The syntax changes in two places: the comment character and the version function.

## Lab Description

![Lab description](images/08-lab-aciklama.png)

## Syntax Differences on MySQL/MSSQL

- **Comment character `#`:** In MySQL, a line comment is made with `#`. (Oracle/PostgreSQL/MSSQL use `--`; if `--` is used in MySQL it must be followed by a space: `-- `. So `#` is more practical in MySQL.) It is used directly in the raw request in Repeater so the browser does not interpret `#` as a URL fragment.
- **No `FROM` needed:** The `FROM dual` requirement from Oracle does not apply here; `SELECT` can be written without a `FROM`.
- **Version function `@@version`:** For both MySQL and Microsoft (MSSQL), version information is retrieved with `@@version`.

## Solution Steps

### 1. Find the column count

We send the request to Burp Repeater and test the column count (using `#` as the comment character):

```
Gifts' UNION SELECT NULL,NULL#
```

The response is `200 OK`. The query returns **2 columns**.

![Repeater — NULL,NULL#, 200 OK](images/08-repeater-sutun-sayisi.png)

- The leading `'` closes the original `category='...'` quote.
- `#` comments out the rest of the SQL.

### 2. Read the version (`@@version`)

We put the version information in one of the two columns and a filler text in the other:

```
Gifts' UNION SELECT @@version,'a'#
```

- `@@version` is a system variable that returns version information in MySQL and MSSQL.
- `'a'` is placed in column 2 because its value is not needed but the column count must still be 2. (Text was used because the column holding the version string accepts text.)

The response is `200 OK`; the version string is added to the page and the lab is marked as solved.

![Repeater — @@version,'a'#](images/08-repeater-versiyon.png)

> Note: I applied the version-retrieval method (per DBMS: `@@version` / `version()` / `v$version`) using a SQLi cheat sheet as reference.

### 3. Result

Once the payload is sent, the lab switches to "Solved".

![Lab solved](images/08-lab-cozuldu.png)

## Version Queries by DBMS

| DBMS | Version query | Comment char | Note |
|------|---------------|--------------|------|
| MySQL | `SELECT @@version` | `#` or `-- ` (with space) | no `FROM` needed |
| Microsoft (MSSQL) | `SELECT @@version` | `--` | no `FROM` needed |
| Oracle | `SELECT banner FROM v$version` | `--` | `FROM dual` required |
| PostgreSQL | `SELECT version()` | `--` | — |

In practice, which DBMS is in use is discovered through such experiments: the `#` comment and `@@version` working points to MySQL (or MSSQL); needing `FROM dual` points to Oracle.

## Remediation

- **Parameterized queries (prepared statements):** User input must not be concatenated into the query, so `UNION SELECT` cannot be injected.
- **Input validation (allow-list):** Validate `category` only against a known set of values.
- **Hide verbose errors and output:** Reflecting query results to the user makes leaking information such as the version easier. (Not sufficient on its own — the real fix is parameterized queries.)

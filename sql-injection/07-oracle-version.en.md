# SQL Injection — Querying the Database Type and Version (Oracle)

**Lab:** SQL injection attack, querying the database type and version on Oracle
**Level:** Practitioner
**Category:** SQL Injection (UNION / information gathering)
**Tools:** Burp Suite (Proxy + Repeater), browser
**Lab link:** https://portswigger.net/web-security/sql-injection/examining-the-database/lab-querying-database-version-oracle

---

## Vulnerability Summary

The product category filter is vulnerable to SQL injection and the query results are shown on the page, so we can use a `UNION` attack to display the result of an injected query. The backend probably runs a query like:

```sql
SELECT name, description FROM products WHERE category = 'Gifts'
```

Because the `category` parameter is embedded directly into the query, we can close the quote and append our own SQL. The goal: display the database **version string**.

This lab is specifically on **Oracle**. Two features that distinguish Oracle from other databases stand out in this solution: the `FROM dual` requirement and the `v$version` view.

## Lab Description

![Lab description](images/07-lab-aciklama.png)

## Solution Steps

### 1. Target string and normal page

The lab asks us to make the database retrieve: `Oracle Database 11g Express Edition Release 11.2.0.2.0 - 64bit Production, ...`. When the normal "Pets" page is viewed, the lab is still unsolved.

![Normal page — target version string](images/07-normal-sayfa.png)

### 2. Find the column count (Oracle: `FROM dual`)

For a UNION attack to work, our injected `SELECT` must return the **same number of columns** as the original query. We send the request to Burp Repeater and test:

```
Pets' UNION SELECT NULL,NULL FROM dual--
```

The response is `200 OK`, so the query returns **2 columns**.

![Repeater — NULL,NULL FROM dual, 200 OK](images/07-repeater-sutun-sayisi.png)

The Oracle-specific point here is `FROM dual`: in Oracle you cannot write a `SELECT` without a `FROM`. `dual` is a one-row dummy system table that exists exactly for this purpose. MySQL/MSSQL do not need it; this requirement is one of the clearest clues that the target is Oracle.

- The leading `'` closes the original `category='...'` quote.
- `--` comments out the rest of the SQL, so the leftover of the original query causes no error.
- (The `+` signs in the URL stand for spaces, because the payload is sent in a query parameter.)

### 3. Confirm the text column

Since the version string is text, we confirm there is a column we can print text into:

```
Pets' UNION SELECT 'a',NULL FROM dual--
```

The response is `200 OK`; column 1 is compatible with text data.

![Repeater — 'a',NULL FROM dual, 200 OK](images/07-repeater-metin-sutun.png)

### 4. Read the version (`v$version`)

After confirming we can print text, we retrieve the version information:

```
Pets' UNION SELECT banner,NULL FROM v$version--
```

- `v$version` is an Oracle system view that holds version information as rows.
- The `banner` column contains the human-readable version text ("Oracle Database 11g ...").
- We put `banner` in column 1 and `NULL` in column 2; we do not need column 2's value, but the column count must still be 2. `NULL` is a safe filler because it is compatible with any type.

The response is `200 OK`; the version string is added to the page among the product list, and the lab is marked as solved.

![Repeater — banner,NULL FROM v$version](images/07-repeater-versiyon.png)

## Why Oracle-Specific?

Retrieving version information differs across databases. We know the target is Oracle from two clues: the `FROM dual` requirement and the `v$version` view. Version queries by DBMS:

| DBMS | Version query |
|------|---------------|
| Oracle | `SELECT banner FROM v$version` or `SELECT version FROM v$instance` |
| Microsoft (MSSQL) | `SELECT @@version` |
| PostgreSQL | `SELECT version()` |
| MySQL | `SELECT @@version` |

![Version queries by DBMS](images/07-dbms-versiyon-tablosu.png)

In practice, which DBMS is in use is usually discovered through such experiments: requiring `FROM dual` points to Oracle, requiring `CONCAT` instead of `||` points to MySQL, and so on.

## Why Does Information Gathering Matter?

Knowing the database type and version is needed to choose the correct syntax and known vulnerabilities in later steps. Because the syntax of UNION payloads (e.g. `FROM dual`, system view names, string concatenation operators) varies by DBMS, the rest of the attack depends on this information.

## Remediation

- **Parameterized queries (prepared statements):** User input must not be concatenated into the query, so `UNION SELECT` cannot be injected.
- **Input validation (allow-list):** Validate `category` only against a known set of values.
- **Hide verbose errors and output:** Reflecting query results and error messages to the user makes leaking information such as the version easier. (Not sufficient on its own — the real fix is parameterized queries.)
- **Principle of least privilege:** The application's database account should not have needless access to system views (e.g. `v$version`).

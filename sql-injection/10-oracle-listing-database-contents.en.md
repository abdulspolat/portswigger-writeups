# SQL Injection — Listing the Database Contents (Oracle)

**Lab:** SQL injection attack, listing the database contents on Oracle
**Level:** Practitioner
**Category:** SQL Injection (UNION / Oracle data dictionary)
**Tools:** Burp Suite (Proxy + Repeater), browser
**Lab link:** https://portswigger.net/web-security/sql-injection/examining-the-database/lab-listing-database-contents-oracle

---

## Vulnerability Summary

The product category filter is vulnerable to SQL injection and the query results are shown on the page. This lab is the **Oracle** counterpart of the previous "non-Oracle" lab: we do not know the table name or the column names; we must discover them first, then retrieve the contents. The goal: obtain all usernames/passwords and log in as `administrator`.

The difference: Oracle has **no** `information_schema`. The metadata (data dictionary) is held in different views:

- `all_tables` → names of all accessible tables (`table_name` column).
- `all_tab_columns` → names of all columns (`column_name`), with which table they belong to (`table_name`).

Also, in Oracle every `SELECT` requires a `FROM` (see `FROM dual` in the Oracle version lab).

## Lab Description

![Lab description](images/10-lab-aciklama.png)

## Solution Steps

### 1. Normal request

We send the request to Burp Repeater and inspect the normal `GET /filter?category=Gifts` response.

![Repeater — normal request](images/10-repeater-normal.png)

### 2. List the tables

We retrieve table names from Oracle's `all_tables` view:

```
Gifts' UNION SELECT table_name,NULL FROM all_tables--
```

All accessible tables are listed in the results (including Oracle's system tables like `WWV_FLOW_*`, `WRI$_*` — this naming confirms the target is Oracle).

![Listing the tables](images/10-tablolari-listeleme.png)

In the list we find the table holding user data: **`USERS_LIICIG`**. (The table name gets a random suffix in each session.)

![users table found](images/10-users-tablosu.png)

### 3. List the column names

We retrieve the column names of `USERS_LIICIG` from the `all_tab_columns` view, filtering to that table:

```
Gifts' UNION SELECT column_name,NULL FROM all_tab_columns WHERE table_name='USERS_LIICIG'--
```

![Listing the column names](images/10-sutun-adlarini-listeleme.png)

This gives us the username and password column names: **`USERNAME_AIXDML`** and **`PASSWORD_AOGOPM`**.

![Columns in the browser](images/10-sutunlar-tarayici.png)

### 4. Retrieve the data

Now that we know the table and column names, we retrieve the usernames and passwords:

```
Gifts' UNION SELECT USERNAME_AIXDML,PASSWORD_AOGOPM FROM USERS_LIICIG--
```

![Repeater — retrieving data](images/10-repeater-veri-cekme.png)

Among the retrieved credentials is the password for the `administrator` user:

![Credentials — administrator, carlos, wiener](images/10-kimlik-bilgileri.png)

### 5. Log in as administrator

Using the `administrator` username and password, we log in via the login form; the lab is marked as solved.

![Lab solved — logged in as administrator](images/10-lab-cozuldu.png)

## Oracle vs. non-Oracle — Metadata Comparison

| Purpose | non-Oracle (PostgreSQL/MySQL/MSSQL) | Oracle |
|---------|--------------------------------------|--------|
| List tables | `SELECT table_name FROM information_schema.tables` | `SELECT table_name FROM all_tables` |
| List columns | `SELECT column_name FROM information_schema.columns WHERE table_name='...'` | `SELECT column_name FROM all_tab_columns WHERE table_name='...'` |
| `FROM` requirement | Not needed | Required for every `SELECT` (including `dual`) |

The logic is the same in both cases: first learn the table name, then that table's column names from the data dictionary, then retrieve the actual data. Only the names of the views used change by DBMS.

## Remediation

- **Parameterized queries (prepared statements):** User input must not be concatenated into the query, so expressions like `UNION SELECT ... FROM all_tables` cannot be injected.
- **Input validation (allow-list):** Validate `category` only against a known set of values.
- **Secure password storage:** Store passwords with a strong hashing algorithm (e.g. bcrypt).
- **Principle of least privilege:** The application's database account should not have needless access to data dictionary views (`all_tables`, `all_tab_columns`) and sensitive tables.

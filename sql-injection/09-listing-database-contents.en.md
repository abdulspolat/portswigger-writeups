# SQL Injection — Listing the Database Contents (non-Oracle / PostgreSQL)

**Lab:** SQL injection attack, listing the database contents on non-Oracle databases
**Level:** Practitioner
**Category:** SQL Injection (UNION / information_schema)
**Tools:** Burp Suite (Proxy + Repeater), browser
**Lab link:** https://portswigger.net/web-security/sql-injection/examining-the-database/lab-listing-database-contents-non-oracle

---

## Vulnerability Summary

The product category filter is vulnerable to SQL injection and the query results are shown on the page. The difference from previous labs: this time we do **not** know the table name or the column names. The application has a login function and the database holds a table of usernames/passwords; we must first discover that table and its columns, then retrieve its contents.

For this we use **`information_schema`**: on non-Oracle databases (PostgreSQL, MySQL, MSSQL) it is a standard schema that holds the database metadata — all tables and columns. The `pg_`-prefixed table names in this lab (e.g. `pg_available_extension_versions`) indicate the target is **PostgreSQL**.

## Lab Description

![Lab description](images/09-lab-aciklama.png)

## Solution Steps

### 1. Normal request

We send the request to Burp Repeater and inspect the normal `GET /filter?category=Gifts` response.

![Repeater — normal request](images/09-repeater-normal.png)

### 2. Determine the column count

We test the column count via `information_schema`:

```
Gifts' UNION SELECT NULL,NULL FROM information_schema.tables--
```

The response is `200 OK`; the query returns **2 columns**.

![Repeater — NULL,NULL, 200 OK](images/09-repeater-sutun-sayisi.png)

### 3. List the tables

We retrieve the names of the tables in the database. Table names are in the `table_name` column within `information_schema`:

```
Gifts' UNION SELECT table_name,NULL FROM information_schema.columns--
```

The table names are listed in the results (including PostgreSQL's `pg_`-prefixed system tables).

![Listing the tables](images/09-tablolari-listeleme.png)

In the list we find the table holding user data: **`users_wunxkq`**. (The table name gets a random suffix in each lab session.)

![users table found](images/09-users-tablosu.png)

### 4. Confirm the column count/types

We confirm there are two text columns on the found table:

```
Gifts' UNION SELECT 'a','b' FROM users_wunxkq--
```

The response is `200 OK`; both columns are compatible with text data.

![Repeater — 'a','b' FROM users_wunxkq](images/09-repeater-sutun-dogrulama.png)

### 5. List the column names

We retrieve the column names of `users_wunxkq` by filtering `information_schema.columns` to that table:

```
Gifts' UNION SELECT column_name,NULL FROM information_schema.columns WHERE table_name='users_wunxkq'--
```

This gives us the username and password column names: **`username_xylgoh`** and **`password_woqsrt`**. (The column names also carry a session-specific random suffix.)

![Listing the column names](images/09-sutun-adlarini-listeleme.png)

### 6. Retrieve the data

Now that we know the table and column names, we retrieve the usernames and passwords:

```
Gifts' UNION SELECT username_xylgoh,password_woqsrt FROM users_wunxkq--
```

![Repeater — username_xylgoh,password_woqsrt FROM users_wunxkq](images/09-repeater-veri-cekme.png)

Running it in the browser lists the usernames and passwords:

![Results in the browser](images/09-tarayici-sonuclar.png)

### 7. Log in as administrator

Using the retrieved `administrator` username and password, we log in via the login form; the lab is marked as solved.

![Lab solved — logged in as administrator](images/09-lab-cozuldu.png)

## Why `information_schema`?

In previous labs the table and column names were given to us. In real attacks they are unknown; this information is learned from the database **metadata**. On non-Oracle databases this metadata lives under the `information_schema` schema:

- `information_schema.tables` → names of all tables (`table_name` column).
- `information_schema.columns` → names of all columns (`column_name`), with which table they belong to (`table_name`). `WHERE table_name='...'` filters to a specific table's columns.

(On Oracle the same is done with the `all_tables` and `all_tab_columns` views; hence the lab name says "non-Oracle".)

## Remediation

- **Parameterized queries (prepared statements):** User input must not be concatenated into the query, so `UNION SELECT ... FROM information_schema...` cannot be injected.
- **Input validation (allow-list):** Validate `category` only against a known set of values.
- **Secure password storage:** Store passwords with a strong hashing algorithm (e.g. bcrypt), so that even leaked table contents do not directly yield usable passwords.
- **Principle of least privilege:** The application's database account should not have needless access to `information_schema` and sensitive tables.

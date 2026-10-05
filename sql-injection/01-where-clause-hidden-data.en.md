# SQL Injection — Retrieval of Hidden Data via WHERE Clause

**Lab:** SQL injection vulnerability in WHERE clause allowing retrieval of hidden data
**Level:** Apprentice
**Category:** SQL Injection
**Tools:** Browser (URL only), Burp Suite (optional)
**Lab link:** https://portswigger.net/web-security/sql-injection/lab-retrieve-hidden-data

---

## Vulnerability Summary

The product category filter inserts the user-supplied `category` parameter directly into a SQL query. When a category is selected, the backend runs a query like:

```sql
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

Two things matter here:

- The `category` value is concatenated into the query inside single quotes, so we can inject a quote and change the query's logic.
- The `released = 1` condition only returns published products. Unreleased products are normally hidden.

Goal: use injection to neutralize the `released = 1` condition and make the application display unreleased products as well.

## Lab Description

![Lab description](images/01-lab-aciklama.png)

## Solution Steps

This lab was solved entirely from the browser, by modifying only the `category` parameter in the URL. No additional tooling was required.

### 1. Understand the normal request

Clicking a category filter produces a URL like:

```
/filter?category=Gifts
```

Which maps to the backend query:

```sql
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

### 2. Build the payload

We inject the following value into the `category` parameter:

```
Gifts' OR 1=1--
```

Since spaces are encoded as `+` in the URL, the form entered in the address bar is:

```
/filter?category=Gifts'+OR+1=1--
```

Breaking the payload down:

- `Gifts'` → Closes the open single quote and terminates the original string literal.
- `OR 1=1` → Adds an always-true condition. Because it uses `OR`, every row satisfies the condition regardless of the `released` value.
- `--` → A SQL comment marker. It disables everything after it (the `AND released = 1` part).

After injection, the query becomes:

```sql
SELECT * FROM products WHERE category = 'Gifts' OR 1=1--' AND released = 1
```

Thanks to the comment, the query that actually executes is:

```sql
SELECT * FROM products WHERE category = 'Gifts' OR 1=1
```

Since `1=1` is always true, the `WHERE` condition holds for every product, so all products — published and unreleased — are returned.

### 3. Send the payload via the URL

Enter the crafted payload in the address bar and submit the request:

![URL with payload](images/01-payload-url.png)

### 4. Result

Once the request is sent, the application returns all products (including unreleased ones) and the lab is marked as solved.

![Lab solved](images/01-lab-cozuldu.png)

## Why It Works

The root cause is that user input is inserted into the SQL query via direct string concatenation. The application interprets the input as part of the query rather than as data, so injecting quotes and SQL keywords lets us alter the query's logic.

## Remediation

- **Parameterized queries (prepared statements):** User input should be bound as a parameter instead of being concatenated into the query text. Input is then always handled as data and never executed as SQL code.
- **Input validation (allow-list):** For known parameters like `category`, validate against a fixed set of allowed values.
- **Principle of least privilege:** The application's database account should hold only the privileges it actually needs.

> Note: Trying to "sanitize" input (e.g. escaping quotes) is not a reliable fix on its own; the real solution is parameterized queries.

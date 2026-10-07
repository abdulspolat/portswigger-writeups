# Path Traversal — Simple Case

**Lab:** File path traversal, simple case
**Level:** Apprentice
**Category:** Path Traversal (Directory Traversal)
**Tools:** Burp Suite (Proxy / Repeater)
**Lab link:** https://portswigger.net/web-security/file-path-traversal/lab-simple-case

---

## What Is Path Traversal?

Path traversal (directory traversal) arises when an application reads files from the server and inserts user input directly into the file path. An attacker can use the `../` (go up one directory) sequence to climb out of the intended directory and read arbitrary files on the server (application code, configuration files, system files like `/etc/passwd`).

On Linux, `../` moves up one directory; repeated enough times it reaches the filesystem root (`/`), from where the full path of the target file is written.

## Vulnerability Source

This lab contains a path traversal flaw in the display of product images. Images are served via this endpoint:

```
GET /image?filename=11.jpg
```

The application inserts the `filename` parameter directly into a file path and reads it, probably with logic like:

```
/var/www/images/ + <filename>
```

Because the `filename` value is never validated, we can use `../` sequences to climb out of the `images` directory and reach other files on the server. The goal: read the contents of `/etc/passwd`.

## Lab Description

![Lab description](images/01-lab-aciklama.png)

## Solution Steps

### 1. Capture the normal request

Opening a product page, we capture the image request with Burp Proxy. The request looks like:

```
GET /image?filename=11.jpg
```

![Normal image request](images/01-normal-istek.png)

### 2. Inject the payload

We replace the `filename` parameter with a path traversal payload pointing at the target file:

```
GET /image?filename=../../../etc/passwd
```

![Request with injected payload](images/01-payload-intercept.png)

The payload logic:

- `../` → go up one directory. Repeated three times (`../../../`), it climbs from the `images` directory up to the filesystem root (`/`).
- `etc/passwd` → the path to the target file from the root.

So the requested path is `images/../../../etc/passwd`, which normalizes to `/etc/passwd`. In this "simple case" lab the application performs no filtering or encoding checks, so plain `../` sequences work directly.

### 3. Result

The request returns `200 OK` and the response body shows the contents of `/etc/passwd` (the user list starting with `root:x:0:0:...`). This proves the path traversal succeeded and the lab is solved.

![/etc/passwd contents](images/01-passwd-icerigi.png)

## Why It Works

The application treats the user-supplied `filename` value as safe and uses it directly to build a file path. Because the operating system interprets `../` sequences as "go up one directory", the attacker can escape the bounds of the intended image directory and roam the filesystem freely. With no validation, canonicalization, or allow-list check in place, the attack succeeds in its simplest form.

## Remediation

- **Avoid putting user input into file paths at all when possible:** Serving files via an identifier/index (e.g. a database record ID) is safer than taking a raw filename.
- **Input validation (allow-list):** The filename should consist only of permitted characters; path separators (`/`, `\`) and `..` sequences must be rejected.
- **Canonicalization and boundary check:** After resolving the input to its canonical path, verify that the resulting path stays under the expected base directory (e.g. `/var/www/images/`). For example: take the `realpath`/canonical path first, then check whether it starts with the base directory.
- **Principle of least privilege:** The account running the application should not needlessly have read access to system files (e.g. `/etc/passwd`).

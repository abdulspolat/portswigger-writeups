# Path Traversal — Validation of Start of Path

**Lab:** File path traversal, validation of start of path
**Level:** Practitioner
**Category:** Path Traversal (Directory Traversal)
**Tools:** Burp Suite (Proxy / Repeater)
**Lab link:** https://portswigger.net/web-security/file-path-traversal/lab-validate-start-of-path

---

## Vulnerability Summary

This lab again contains a path traversal flaw in product images, but it works differently from the previous ones. The application receives the **full file path** via a request parameter (not a relative filename) and validates that the supplied path **starts with the expected folder**.

So in the normal request, `filename` already carries the full path:

```
GET /image?filename=/var/www/images/6.jpg
```

The application's only security check is: "does the supplied path start with `/var/www/images/`?" The goal is again to read `/etc/passwd`.

## Lab Description

![Lab description](images/05-lab-aciklama.png)

## Solution Steps

### 1. Normal request

We send the image request to Burp Repeater. Note that `filename` already contains the full path:

```
GET /image?filename=/var/www/images/6.jpg
```

Response: `200 OK`, the image is returned.

![Normal image request — full path](images/05-normal-istek.png)

### 2. Start with the expected folder, then climb up

Since the validation only looks at the **start** of the path, we begin the payload with the expected folder and then climb up with `../`:

```
GET /image?filename=/var/www/images/../../../etc/passwd
```

Response: `200 OK` and the body shows the contents of `/etc/passwd` (`root:x:0:0:...`). The lab is solved.

![/etc/passwd via /var/www/images/../../../etc/passwd](images/05-payload-passwd.png)

## Why It Works (The Logic)

The defense logic rests on a simple assumption: *"If the path starts with `/var/www/images/`, the file is inside that folder and is safe."* This assumption is false.

Let's examine our payload:

```
/var/www/images/../../../etc/passwd
└──── start ────┘└──── traversal ────┘
```

- **The start part** (`/var/www/images/`) exists to pass validation. When the application checks "does the path start with the expected folder?", the answer is **yes**, so the check passes.
- **The traversal part** (`../../../etc/passwd`) then kicks in after validation. When the operating system resolves the path, it applies the `../` sequences and climbs three directories up from `/var/www/images/` to the filesystem root, then goes to `/etc/passwd`:

```
/var/www/images/../../../etc/passwd
         ↓ (../ applied three times)
/etc/passwd
```

The root cause is: the application only checks **how the path starts**, but not **where it ends up** (the final, resolved target). A path starting with the right prefix does not guarantee the path will *stay inside* that directory — because `../` can escape out of it from within the same path. "The start is correct" and "the target is safe" are not the same thing.

## Remediation

- **Validate the final target, not the start (canonicalization + boundary check):** The correct approach is to resolve the path to its canonical form first (after applying `../` sequences), then check whether the result stays under the expected directory:
  ```
  canonical = realpath(input)         # applies ../, finds the real target
  if not canonical.startsWith(realpath("/var/www/images/")): reject
  ```
  With this method, `/var/www/images/../../../etc/passwd` first resolves to `/etc/passwd`, then is checked, and is rejected because it does not start with `/var/www/images/`.
- **Don't put input into file paths at all:** Serving files via an identifier/index is the most robust solution.
- **Principle of least privilege:** The application account should not have needless read access to system files.

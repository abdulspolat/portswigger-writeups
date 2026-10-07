# Path Traversal — Traversal Sequences Stripped Non-Recursively

**Lab:** File path traversal, traversal sequences stripped non-recursively
**Level:** Practitioner
**Category:** Path Traversal (Directory Traversal)
**Tools:** Burp Suite (Proxy / Repeater)
**Lab link:** https://portswigger.net/web-security/file-path-traversal/lab-sequences-stripped-non-recursively

---

## Vulnerability Summary

This lab again contains a path traversal flaw in product images. The defense this time: before using the user-supplied filename, the application **strips traversal sequences (`../`)** — it removes `../` patterns from the input and uses what remains. The goal is again to read `/etc/passwd`.

The critical point: this stripping is **non-recursive**, meaning it runs over the input **only once**. The solution exploits exactly this gap.

## Lab Description

![Lab description](images/03-lab-aciklama.png)

## Solution Steps

### 1. Normal request

We send the image request to Burp Repeater; the normal request returns the image (`200 OK`, `image/jpeg`):

```
GET /image?filename=64.jpg
```

![Normal image request](images/03-normal-istek.png)

### 2. Nested payload

Since a plain `../` would be stripped, we send a nested payload so that a `../` remains **after** the stripping runs:

```
GET /image?filename=....//....//....//etc/passwd
```

The response is `200 OK` and the body shows the contents of `/etc/passwd` (`root:x:0:0:...`). The lab is solved.

![/etc/passwd via ....// payload](images/03-payload-passwd.png)

## Why It Works (The Logic)

The root cause of this vulnerability is that **sanitization is performed non-recursively.** Let's go step by step.

The application finds and removes `../` (and probably `..\`) patterns in the input, but it performs this removal in **a single pass** and does not re-check the result. It says "remove, done" without checking whether a new `../` appeared after removal.

Our payload is made of `....//` blocks. Consider a single `....//` block:

```
....//
```

The application looks for a `../` pattern **inside** it. The `....//` sequence hides a `../` in its middle. Writing out the six characters of `....//` — `.` `.` `.` `.` `/` `/` — the application finds the inner `../` (`..` + `/`) and removes it. When one `../` is removed from `....//`, what remains is:

```
....//   →  (inner ../ removed)  →  ../
```

So each `....//` block becomes a clean `../` after the single-pass stripping. If the stripping were **recursive**, it would catch this newly-formed `../` and remove it too, and the attack would fail. But because the stripping is a single pass, the resulting `../` stays as is.

So the final path the server sees becomes:

```
....//....//....//etc/passwd
   │        │        │
   ▼        ▼        ▼
  ../      ../      ../        →   ../../../etc/passwd
```

After stripping we are left with exactly the `../../../etc/passwd` we wanted, climbing to the filesystem root and reaching `/etc/passwd`.

**In short:** The filter looks for `../`, but we give it input that still leaves a `../` behind after removal. The root cause is that the sanitization finishes in one pass without checking "is the output actually clean now?".

## Remediation

- **Don't use blacklist-based sanitization:** The "strip `../` sequences" approach is inherently fragile (bypassable with `....//`, encoding tricks, etc.). Even recursive stripping often fails to catch every variation.
- **Canonicalization + boundary check (recommended):** After resolving the input to its canonical path, verify that the resulting path stays under the expected base directory:
  ```
  canonical = realpath(base + input)
  if not canonical.startsWith(realpath(base)): reject
  ```
  This is safe regardless of how the input is crafted, because it relies on the OS's own path resolution.
- **Don't put input into file paths at all:** Serving files via an identifier/index is the most robust solution.
- **Principle of least privilege:** The application account should not have needless read access to system files.

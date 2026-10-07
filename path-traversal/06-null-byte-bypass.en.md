# Path Traversal — File Extension Validation Bypass with Null Byte

**Lab:** File path traversal, validation of file extension with null byte bypass
**Level:** Practitioner
**Category:** Path Traversal (Directory Traversal)
**Tools:** Burp Suite (Proxy / Repeater / Decoder)
**Lab link:** https://portswigger.net/web-security/file-path-traversal/lab-validate-file-extension-null-byte-bypass

---

## Vulnerability Summary

This lab again contains a path traversal flaw in product images. The defense this time: the application validates that the supplied filename **ends with the expected extension** (e.g. `.jpg`). The goal is again to read `/etc/passwd`.

The solution is to inject a **null byte (`%00`)** to both pass this extension check and make the operating system cut the path off before the extension when opening the file.

## Lab Description

![Lab description](images/06-lab-aciklama.png)

## Solution Steps

### 1. Normal request

We send the image request to Burp Repeater; the normal request returns the image (`200 OK`):

```
GET /image?filename=75.jpg
```

![Normal image request](images/06-normal-istek.png)

### 2. Classic traversal — blocked

```
GET /image?filename=../../../etc/passwd
```

Response: `400 Bad Request`, `"No such file"`. Because `/etc/passwd` does not end with the expected `.jpg` extension, the extension validation rejects the request.

![Extension validation blocked](images/06-traversal-engellendi.png)

### 3. Prepare the null byte

A null byte is written as `%00` in URL encoding and corresponds to a single `0x00` (zero) byte. In Burp Decoder we confirm that `%00` really decodes to a single null byte (`00` in the hex view):

![Burp Decoder — %00 → 0x00](images/06-decoder-nullbyte.png)

### 4. Bypass with the null byte

We craft the payload so that a null byte, followed by the expected extension, comes after the target file:

```
GET /image?filename=../../../etc/passwd%00.jpg
```

Response: `200 OK` and the body shows the contents of `/etc/passwd` (`root:x:0:0:...`). The lab is solved.

![/etc/passwd via ../../../etc/passwd%00.jpg](images/06-payload-passwd.png)

## Why It Works (The Logic)

This bypass arises because two different layers **interpret the same string differently.** Our payload:

```
../../../etc/passwd%00.jpg
```

**Layer 1 — The application's extension validation (high-level code).**
The application looks at the whole string: `../../../etc/passwd` + null + `.jpg`. Checking the end, it sees the string ends with `.jpg` and **passes** the extension check. This layer (e.g. the string types of high-level languages like Java or PHP) carries the null byte as an ordinary character and does not cut the string there.

**Layer 2 — The OS file-open call (low-level / C).**
When the application passes this path to the operating system to open the file, the underlying C-based file APIs treat strings as **null-terminated**. That is, they interpret the `0x00` byte as "the string ends here" and ignore everything after it (`.jpg`). The real path the OS sees is:

```
../../../etc/passwd%00.jpg
                   ▲
                   └── null byte: string ends here
→ the OS opens the file:  ../../../etc/passwd
```

Result: the `.jpg` extension exists only to fool the application's validation; the file actually opened is `/etc/passwd`.

**Root cause in short:** The layer doing the extension check treats the null byte as a normal character (so it sees `.jpg`), while the layer opening the file treats the null byte as end-of-string (so it drops `.jpg`). This interpretation difference lets us simultaneously pass the "extension is correct" check and reach the real target.

> Historical note: Null byte injection was especially effective in older systems that did string handling in a high-level language but delegated file access to lower layers using null-terminated strings. Most modern languages and frameworks now reject null bytes within a path; still, it may appear in legacy/mixed systems.

## Remediation

- **Reject dangerous characters including the null byte:** Unexpected characters in the filename, such as the null byte (`\0` / `%00`) and path separators, must be strictly rejected.
- **Canonicalization + boundary check (recommended):** After resolving the input to its canonical path, verify that the result stays under the expected base directory. The extension check should also be done on this resolved, safe path:
  ```
  canonical = realpath(base + input)
  if not canonical.startsWith(realpath(base)): reject
  if not canonical.endsWith(".jpg"): reject
  ```
- **Don't put input into file paths at all:** Serving files via an identifier/index is the most robust solution.
- **Principle of least privilege:** The application account should not have needless read access to system files.

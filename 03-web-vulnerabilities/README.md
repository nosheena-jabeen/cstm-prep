# GingerByte — Deliberately Vulnerable PHP Web Application

A purpose-built vulnerable e-commerce application for CSTM (Cyber Scheme Team Member) assessment preparation. GingerByte simulates a realistic small business web shop with intentional security weaknesses covering the most common web application vulnerability classes.

> **Warning:** This application is intentionally insecure. Run it only in an isolated lab environment. Never expose it to the internet or a production network.

---

## Architecture

| Component | Technology | Port |
|---|---|---|
| Web application | PHP 7 / Apache (Docker) | 8080 |
| Database | MySQL 5.7 (Docker) | 3306 (internal) |
| Database admin | phpMyAdmin (manual install) | 8080/phpmyadmin |

The application runs entirely in Docker Compose. The web and database containers communicate over an internal Docker network using the service name `db` as the hostname.

```
browser → localhost:8080 → [web container: Apache/PHP] → [db container: MySQL]
                                      ↓
                           /var/www/html/ (volume mount)
```

---

## Running the Lab

```bash
cd ~/gingerbyte-php
sudo docker compose up -d
```

Application: `http://localhost:8080`  
phpMyAdmin: `http://localhost:8080/phpmyadmin`  
Default DB credentials: `root` / `root`

---

## Vulnerability Index

### 1. SQL Injection — `login.php`, `search.php`

**Vulnerability:** User input is concatenated directly into SQL queries without prepared statements or parameterisation.

**Design decision:** Classic string concatenation (`"SELECT * FROM users WHERE username = '" . $_POST['username'] . "'"`) rather than `PDO::prepare()`. This allows authentication bypass via `' OR 1=1 --` and data extraction via UNION-based injection.

**OWASP:** A03 — Injection

---

### 2. Cross-Site Scripting (XSS) — `search.php`, `reviews`

**Vulnerability:** User-supplied input is reflected back into the page without HTML encoding.

**Design decision:** `echo $_GET['q']` outputs the search term directly into the HTML response. No call to `htmlspecialchars()`. Allows stored and reflected XSS payloads.

**OWASP:** A03 — Injection

---

### 3. Admin Interface Exposure — `/phpmyadmin`

**Vulnerability:** phpMyAdmin is accessible on the application port with default root credentials (`root` / `root`).

**Design decision:** phpMyAdmin installed into the web root with no access controls. MySQL root password left as `root`. Allows brute-force via Burp Intruder, direct data manipulation, and password hash extraction.

**OWASP:** A05 — Security Misconfiguration · A07 — Identification and Authentication Failures

---

### 4. Weak Password Storage — `users` table

**Vulnerability:** User passwords are stored as unsalted MD5 hashes.

**Design decision:** `password = MD5($password)` with no salt. MD5 is fast and widely pre-computed. All hashes in the `users` table are crackable with rockyou.txt using John the Ripper in seconds.

**OWASP:** A02 — Cryptographic Failures

---

### 5. Local File Inclusion / Path Traversal — `backup.php`

**Vulnerability:** The `file` GET parameter is passed directly to `file_get_contents()` with no path restriction.

**Design decision:** `file_get_contents($_GET['file'])` with no `realpath()` check or directory allowlist. Allows reading `/etc/passwd`, application source files, and database credentials via `../../../../etc/passwd`.

**OWASP:** A01 — Broken Access Control · A05 — Security Misconfiguration

---

### 6. Unrestricted File Upload — `admin/upload.php`

**Vulnerability:** The extension filter checks `explode('.', $filename)[1]` — the second dot segment — rather than the final extension.

**Design decision:** `$ext = $parts[1]` where `$parts = explode('.', $filename)`. A filename of `shell.jpg.php` produces `["shell", "jpg", "php"]`; the check reads `jpg` and passes. Apache executes `.php` as the final extension. Uploaded files land in the web-accessible `/uploads/` directory.

**OWASP:** A03 — Injection · A04 — Insecure Design · A05 — Security Misconfiguration

---

## Database Schema Overview

| Table | Purpose | Notable columns |
|---|---|---|
| `users` | Application accounts | `username`, `password` (MD5 hash) |
| `products` | Store inventory | `name`, `price`, `description` |
| `orders` | Purchase records | `user_id`, `total` |
| `order_items` | Line items per order | `order_id`, `product_id`, `quantity` |
| `reviews` | Product reviews | `product_id`, `user_id`, `body` |
| `messages` | Contact form submissions | `name`, `email`, `body` |
| `discount_codes` | Promo codes | `code`, `percent_off` |
| `admin_notes` | Internal notes | `note` |

---

## Files of Interest

| File | Vulnerability |
|---|---|
| `/var/www/html/backup.php` | Path traversal via `file` parameter |
| `/var/www/html/admin/upload.php` | Double extension bypass → RCE |
| `/var/www/html/login.php` | SQL injection (auth bypass) |
| `/var/www/html/search.php` | Reflected XSS |
| `/var/www/html/includes/db.php` | Database credentials (exposed via LFI) |
| `/var/www/html/phpmyadmin/` | phpMyAdmin (default credentials) |

---

## Intended Learning Outcomes

| Topic | Exercise |
|---|---|
| Admin interface discovery | Find phpMyAdmin via port/path enumeration |
| Credential brute force | Burp Intruder on phpMyAdmin login |
| Direct data manipulation | Edit product prices via phpMyAdmin |
| Password hash cracking | John the Ripper on MD5 hashes from users table |
| Local file inclusion | Path traversal via `backup.php?file=` |
| File upload bypass | Double extension `shell.jpg.php` → webshell RCE |

---

## Keeping GingerByte Vulnerable

Do not apply security patches to this application. The vulnerabilities are intentional. If something stops working, refer to the individual exercise write-ups in this repository to restore the vulnerable behaviour.

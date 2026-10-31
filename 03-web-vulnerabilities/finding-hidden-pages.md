# 07 — Finding Hidden Pages

GingerByte lab · CSTM preparation  
Covers: robots.txt disclosure · directory brute-force · hidden form fields · Base64-encoded credentials

---

## Exercise 1 — robots.txt Information Disclosure

### What it is

`robots.txt` is a plain-text file at the web root that tells search-engine crawlers which paths not to index. It is publicly readable by anyone — including attackers. Listing sensitive paths in `Disallow` entries does not protect them; it advertises them.

### Vulnerable file

**`/robots.txt`**

```
User-agent: *
Disallow: /admin
Disallow: /backup.php
Disallow: /config
Disallow: /install.php
```

### What an attacker sees

Every `Disallow` entry is a hint. The list above reveals:

| Path | Implication |
|---|---|
| `/admin` | Admin panel exists |
| `/backup.php` | Backup script — likely exposes credentials or data dumps |
| `/config` | Configuration directory — may contain DB credentials |
| `/install.php` | Installation script — may allow re-initialisation of the app |

### Exploitation steps

1. Browse to `http://target/robots.txt` — no tools required, no authentication
2. Note every `Disallow` entry
3. Visit each path manually:
   - `http://localhost:8080/backup.php` → returns unauthenticated DB credential dump and full user table including password hashes

### Security impact

- **Unauthenticated credential exposure** via `backup.php`
- **Admin panel location confirmed** — attacker knows exactly where to target brute-force or session attacks
- **No auth required** — all of the above is readable by anyone with a browser

### OWASP category

A05 — Security Misconfiguration

### Remediation

- Remove sensitive paths from `robots.txt` entirely — obscurity is not security, but there is no reason to advertise them
- Protect every sensitive path with server-side authentication regardless of whether it is listed
- Delete or disable `backup.php`, `install.php` and `/config` in production deployments

---

## Exercise 2 — Directory Brute-Force with dirb

### What it is

Even without `robots.txt`, hidden pages can be discovered by brute-forcing common path names against the server. `dirb` automates this using a wordlist, sending HTTP requests for each entry and recording which return a 200 response.

### Tool used

```bash
dirb http://localhost:8080
```

### Results (key findings)

```
+ http://localhost:8080/admin.php      (CODE:200|SIZE:22978)
+ http://localhost:8080/admin/         (CODE:200)
+ http://localhost:8080/admin/index.php (CODE:200|SIZE:27100)
+ http://localhost:8080/backup.php     (CODE:200|SIZE:...)
+ http://localhost:8080/robots.txt     (CODE:200|SIZE:94)
+ http://localhost:8080/uploads/       (CODE:200)
```

### What this confirms

- `/admin.php` and `/admin/index.php` both return **200** — the admin panel is live and accessible with no authentication prompt
- `/uploads/` directory is browsable — potential for file enumeration or upload abuse
- Results match the `robots.txt` disclosures from Exercise 1, confirming those paths are real and reachable

### Exploitation steps

1. Run `dirb http://localhost:8080` (default wordlist covers common paths)
2. Filter results for `CODE:200`
3. Visit confirmed paths — `/admin.php` loads the admin panel without requiring a login

### Security impact

- **Unauthenticated admin access** — any path returning 200 with no auth check is a critical finding
- **Confirms attack surface** — attacker now has a verified map of live endpoints

### OWASP category

A05 — Security Misconfiguration  
A01 — Broken Access Control (admin panel accessible without authentication)

### Remediation

- Require authentication on all admin paths at the server level (not just via nav links)
- Return 403 or redirect to login for unauthenticated requests to `/admin*`
- Disable directory listing on `/uploads/` and all other directories

---

## Exercise 3 — Hidden Form Fields & Privilege Escalation

### What it is

HTML forms can contain `<input type="hidden">` fields — values not shown on the rendered page but present in the page source and submitted with the form. If the server trusts these client-supplied values without server-side validation, an attacker can modify them before submission to alter application behaviour.

In GingerByte, the registration form contains a hidden `userrole` field that determines the role assigned to a new account. The server accepts whatever value the client sends.

### Vulnerable code

**`/var/www/html/register.php`**

```html
<form method="POST" action="/register.php">
  <!-- No CSRF token (intentional) -->
  <input type="hidden" name="userrole" value="c81e728d9d4c2f636f067f89cc14862c">
  <div class="form-group">
    <label>Username</label>
    <input type="text" name="username" autocomplete="off">
  </div>
  <div class="form-group">
    <label>Email</label>
    <input type="email" name="email">
  </div>
  <div class="form-group">
    <label>Password</label>
    <input type="password" name="password">
  </div>
  <button type="submit" class="auth-submit">Create account →</button>
</form>
```

### How the hash is cracked

The hidden value `c81e728d9d4c2f636f067f89cc14862c` is a 32-character hex string — the MD5 hash format. Pasting it into any online MD5 lookup (crackstation.net, hashes.com) reveals it is the MD5 hash of `2` (role ID 2 = customer). The attacker then tries MD5 of `1`:

```
MD5("2") = c81e728d9d4c2f636f067f89cc14862c  ← default (customer)
MD5("1") = c4ca4238a0b923820dcc509a6f75849b  ← admin
```

### Exploitation steps

1. Browse to `http://localhost:8080/register.php`
2. Press **Ctrl+U** (View Source) — the hidden `userrole` field is visible in the HTML
3. Recognise the 32-char hex value as MD5 format
4. Crack it: MD5("2") → role ID 2 = customer
5. Derive admin hash: MD5("1") = `c4ca4238a0b923820dcc509a6f75849b`
6. Open **Burp Suite**, enable intercept, and submit the registration form
7. In the intercepted POST request, change:
   ```
   userrole=c81e728d9d4c2f636f067f89cc14862c
   ```
   to:
   ```
   userrole=c4ca4238a0b923820dcc509a6f75849b
   ```
8. Forward the request — account is created with admin role

### Why encoding doesn't help

MD5 is a public, deterministic algorithm. MD5("1") is always the same value for every attacker on every system. Encoding or hashing a small integer provides no security — the attacker cracks it in seconds. The vulnerability is not the choice of encoding; it is that **the server accepts a role assignment from the client at all**.

### Security impact

- **Full privilege escalation** — any visitor can self-register as admin
- **Complete access control bypass** — admin functions, user management, and data are exposed
- **No user interaction or existing account required**

### OWASP category

A01 — Broken Access Control  
A04 — Insecure Design (role assigned client-side by design)

### Remediation

- **Never trust client-supplied role values** — remove the `userrole` field from the form entirely
- Assign roles server-side at registration time (e.g. all new registrations default to `customer`)
- Store role as an integer or enum in the database, not a client-submitted hash
- Add a CSRF token to the registration form to prevent cross-site form submission

---

---

## Exercise 4 — Base64-Encoded Credentials in Page Source

### What it is

Developers sometimes leave credentials, tokens, or configuration values in HTML comments — often as temporary debug notes that never get removed before deployment. Base64 encoding looks like scrambled text but is not encryption; it is trivially reversible by anyone who recognises the format.

### Vulnerable code

**`/var/www/html/includes/header.php`** (line 805)

```html
<!-- TODO: remove before deployment -- YWRtaW46R2luZ2VyQnl0ZTIwMjQh -->
</head>
```

### How the encoding is recognised

Base64 strings have a distinctive appearance: they use only letters, digits, `+`, `/`, and end with `=` or `==` padding. A 32-character string ending in `h` with no special characters is a strong indicator. Any attacker who sees this in source immediately tries to decode it.

### Exploitation steps

1. Browse to `http://localhost:8080/admin/index.php`
2. Press **Ctrl+U** to view page source
3. Search for `TODO` — find the HTML comment containing the Base64 string
4. Copy the string: `YWRtaW46R2luZ2VyQnl0ZTIwMjQh`
5. Open Burp Suite → **Decoder** tab
6. Paste the string → click **Decode as → Base64**
7. Output: `admin:GingerByte2024!`
8. Credentials: username `admin`, password `GingerByte2024!`

### Why Base64 is not security

Base64 is an encoding scheme, not encryption. It has no key and no secret — any tool (Burp Decoder, CyberChef, `base64 -d` on the command line) decodes it instantly. Storing credentials in Base64 provides no more protection than storing them in plaintext.

### Security impact

- **Hardcoded admin credentials exposed** to any unauthenticated user who views the page source
- **No authentication required** to retrieve them — the comment is served in every page that includes `header.php`
- Credentials can be used to log in directly or to attempt credential stuffing on other services

### OWASP category

A02 — Cryptographic Failures (sensitive data not protected)  
A05 — Security Misconfiguration (debug/dev artefact left in production)

### Remediation

- Never store credentials in HTML, JavaScript, or any client-facing file
- Remove all TODO comments and debug notes before deployment
- Use a secrets manager or environment variables for credentials
- Implement a pre-deployment checklist or automated secret scanning (e.g. `git-secrets`, `truffleHog`) to catch leaked credentials before they reach production

---

## Summary

| Exercise | Finding | Severity | OWASP |
|---|---|---|---|
| 1 — robots.txt | Sensitive paths advertised; backup.php exposes credentials unauthenticated | High | A05 |
| 2 — dirb | Admin panel confirmed live and accessible without authentication | Critical | A01, A05 |
| 3 — Hidden field | Privilege escalation to admin via `userrole` parameter tampering at registration | Critical | A01, A04 |
| 4 — Base64 comment | Admin credentials encoded in HTML comment in page source | Critical | A02, A05 |

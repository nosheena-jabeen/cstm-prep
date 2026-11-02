# Admin Interfaces

GingerByte lab · CSTM preparation  
Covers: admin interface discovery · Burp Suite brute force · direct database manipulation · password hash extraction and cracking

---

## Exercise 1 — Find the phpMyAdmin Interface

### What to check

Admin interfaces are often exposed on alternate ports or well-known paths. Common targets:
- `/phpmyadmin` on the main application port
- Dedicated management ports (8081, 8000)
- Default paths not linked from the application

### Steps

1. Run an nmap scan to identify HTTP services on the target:
   ```bash
   nmap -sV -p 8080,8081,8000 localhost
   ```
2. Browse to common phpMyAdmin paths:
   ```
   http://localhost:8080/phpmyadmin
   http://localhost:8081/phpmyadmin
   http://localhost:8000/phpmyadmin
   ```
3. Result: phpMyAdmin login page loads at `http://localhost:8080/phpmyadmin`
4. Note the interface — it provides direct database access and accepts login attempts

### Why this matters

phpMyAdmin is a common target because it exposes the entire database over HTTP. If accessible externally with weak or default credentials, an attacker has full read/write access to all data without needing to exploit the application itself.

---

## Exercise 2 — Crack the phpMyAdmin Root Password Using Burp Suite

### Steps

1. Open Burp Suite. Set browser proxy to Burp's listener (default `127.0.0.1:8080` — adjust if this clashes with the app port)
2. Browse to `http://localhost:8080/phpmyadmin` and submit a login attempt:
   - Username: `root`
   - Password: `anything` (placeholder)
3. Intercept the POST request in Burp → **Send to Intruder**
4. In Intruder:
   - Attack type: **Sniper**
   - Clear all payload positions; set `§` on the `password` parameter value only
5. Payloads tab → load `/usr/share/wordlists/rockyou.txt`
6. Start the attack and monitor responses:
   - Failed logins return a consistent response length
   - A **different status code or response length** indicates a successful login
7. Confirmed password: `root`
8. Log in to phpMyAdmin with `root` / `root`

### Security impact

- **Full database access** — read, modify or delete all application data
- **Credential theft** — dump all user accounts
- **Persistence** — create new database users or modify application logic via stored procedures

### OWASP category

A07 — Identification and Authentication Failures (weak/default credentials accepted)

---

## Exercise 3 — Modify Product Prices via phpMyAdmin

### Steps

1. Log into phpMyAdmin at `http://localhost:8080/phpmyadmin` (`root` / `root`)
2. In the left sidebar, click **gingerbyte** → **products**
3. Click the **Browse** tab to view all product records
4. Click the **pencil (Edit)** icon on any product row
5. Change the `price` field to a noticeably different value (e.g., `1.00` or `9999.99`)
6. Click **Go** to save
7. Browse to `http://localhost:8080` and refresh the store page
8. Confirm the modified price is displayed — direct database manipulation is reflected immediately in the application

### Security impact

- **Data integrity loss** — prices, stock levels, order data can be silently altered
- **Financial fraud** — an attacker could set prices to near-zero before placing orders
- **No application audit trail** — changes made directly in the database bypass application-level logging

### OWASP category

A01 — Broken Access Control (database admin interface reachable without application authentication)  
A05 — Security Misconfiguration (phpMyAdmin exposed with default credentials)

---

## Exercise 4 — Steal and Crack Password Hashes from the Users Table

### Steps

1. In phpMyAdmin → **gingerbyte** → **users** → **Browse**
2. Observe the `password` column contains MD5 hashes, not plaintext
3. Copy the hashes to a file on Kali:
   ```bash
   cat > ~/hashes.txt << 'EOF'
   0192023a7bbd73250516f069df18b500
   83e828b97ae868c09a235b5ec8cbfffc
   6c4a180b36896a0a8c02787eeafb0e4c
   EOF
   ```
4. Crack with John the Ripper:
   ```bash
   john --wordlist=/usr/share/wordlists/rockyou.txt --format=Raw-MD5 ~/hashes.txt
   john --show --format=Raw-MD5 ~/hashes.txt
   ```
5. John returns the plaintext passwords:
   - `0192023a...` → `admin123`
   - `83e828b9...` → `pawsword`
   - `6c4a180b...` → `password1`
6. Use recovered credentials to log in to the application as those users

### Security impact

- **Account takeover** — all user accounts compromised from a single database read
- **Password reuse** — plaintext passwords may be reused on email, banking, or other services
- **Privilege escalation** — admin account credentials recovered, full application access gained
- **MD5 is broken for password storage** — no salt, fast to crack; rockyou.txt cracks most in seconds

### OWASP category

A02 — Cryptographic Failures (MD5 without salt is not acceptable for password storage)  
A04 — Insecure Design (passwords recoverable via single database query)

### Remediation

- **Use bcrypt, Argon2 or scrypt** — these are slow by design, making brute force impractical
- **Never store plaintext or unsalted hashes** — even if the database is stolen, passwords must be unrecoverable
- **Restrict phpMyAdmin access** — firewall to localhost only; require certificate or VPN; disable on production
- **Change default credentials immediately** — `root`/`root` should never reach a production database

---

## Summary

| Exercise | Finding | Impact | OWASP |
|---|---|---|---|
| 1 — Discovery | phpMyAdmin accessible at `/phpmyadmin` on the application port | High | A05 |
| 2 — Brute force | Root password `root` cracked via Burp Intruder in seconds | Critical | A07 |
| 3 — Price manipulation | Direct DB edit reflected immediately in the application | High | A01, A05 |
| 4 — Hash cracking | MD5 hashes cracked with rockyou; all user passwords recovered | Critical | A02, A04 |

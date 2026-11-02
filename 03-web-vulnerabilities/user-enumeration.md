# User Enumeration

GingerByte lab · CSTM preparation  
Covers: username enumeration via verbose error messages · Burp Intruder automation

---

## Exercise 1 — Enumerate Valid Users via Login Error Messages

### What it is

User enumeration occurs when a login page returns different responses depending on whether a username exists in the database. An attacker can use this difference to build a list of valid usernames before attempting a password attack — significantly reducing the search space.

### Vulnerable code

**`/var/www/html/login.php`**

```php
// First checks username alone
$user_check = $conn->query("SELECT id FROM users WHERE username='$username'");

if ($result && $result->num_rows > 0) {
    // Successful login
    $_SESSION['user_id']   = $user['id'];
    $_SESSION['username']  = $user['username'];
    $_SESSION['role']      = $user['role'];
    header('Location: /index.php');
    exit;
} elseif (!$user_check || $user_check->num_rows === 0) {
    // ---- VULNERABILITY: reveals username does not exist ----
    $error = "Invalid username.";
} else {
    // ---- VULNERABILITY: reveals username exists but password is wrong ----
    // Also leaks the raw SQL query
    $error = "Incorrect password. Query: <code>$sql</code>";
}
```

### What an attacker observes

By submitting a fixed wrong password against different usernames, the error message reveals whether each username exists:

| Username tried | Response | Conclusion |
|---|---|---|
| `admin` | "Incorrect password." | ✅ Valid username |
| `backup-admin` | "Invalid username." | ❌ Does not exist |
| `dan` | "Invalid username." | ❌ Does not exist |
| `developer` | "Invalid username." | ❌ Does not exist |
| `north` | "Invalid username." | ❌ Does not exist |

### Bonus: SQL query leakage

When a valid username is found, GingerByte also leaks the raw SQL query in the error message:

```
Incorrect password. Query: SELECT * FROM users WHERE username='admin' AND password='wrongpassword'
```

This tells the attacker the exact query structure, making SQLi bypass trivial:
- Username: `admin' --` comments out the password check entirely
- The query becomes: `SELECT * FROM users WHERE username='admin' --' AND password='...'`

### Exploitation steps — manual

1. Navigate to `http://localhost:8080/login.php`
2. Try each username with password `wrongpassword`:
   - "Invalid username." → does not exist, move on
   - "Incorrect password." → valid username confirmed
3. Note all valid usernames for use in password attacks

### Exploitation steps — automated with Burp Intruder

1. Open Burp Suite and enable intercept
2. Submit a login attempt — capture the POST request
3. Send to **Intruder**
4. Set the attack position on the `username` parameter only
5. Load a username wordlist (e.g. `/usr/share/wordlists/seclists/Usernames/top-usernames-shortlist.txt`)
6. Set the password to a fixed wrong value
7. Start the attack
8. Filter results by response length or look for responses containing "Incorrect password" — these are valid usernames

### Security impact

- **Valid username disclosure** — attacker can confirm which accounts exist without any authentication
- **Accelerates brute-force attacks** — attacker now targets only confirmed valid usernames
- **Combined with SQL query leak** — attacker gains both a valid username and the query structure needed for SQLi bypass in a single step
- **No rate limiting** — nothing prevents automated enumeration at scale

### OWASP category

A01 — Broken Access Control  
A05 — Security Misconfiguration (verbose error messages)

### Remediation

- Return a **single generic message** for all failed login attempts: "Invalid username or password."
- Never reveal whether the username or the password was incorrect
- Never expose SQL queries or internal logic in error messages
- Implement **rate limiting** and **account lockout** after repeated failed attempts
- Consider **CAPTCHA** after a threshold of failures
- Log and alert on repeated failed login attempts from the same IP

---

## Summary

| Finding | Impact | OWASP |
|---|---|---|
| Different error messages for invalid username vs wrong password | Attacker can enumerate valid usernames | A05 |
| SQL query exposed in error message on valid username | Attacker gains query structure for SQLi bypass | A05 |
| No rate limiting on login attempts | Enumeration and brute-force can run unthrottled | A01 |

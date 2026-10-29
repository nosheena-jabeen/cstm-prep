# SQL Injection — Authentication Bypass and Privilege Escalation

## Overview

As part of a structured assessment of the GingerByte e-commerce application in my home lab, I examined the login form for SQL injection vulnerabilities.

A second SQL injection vulnerability was identified in the login endpoint at `/login.php`. The application constructed an authentication query by concatenating the submitted username and password directly into a SQL WHERE clause. By injecting a condition that is always true, it was possible to bypass the credential check entirely and authenticate as the first user returned by the query — which was the `admin` account. This constitutes both an authentication bypass and a privilege escalation, as the attacker gains administrative access without knowing any valid credentials.

> **Scope:** All testing documented here was performed against systems in my isolated home lab. The application was deliberately configured to be vulnerable for training purposes.

---

## Lab Scope

| System | Role |
|---|---|
| Kali Linux | Attacker |
| GingerByte (Docker, port 8080) | Target |

---

## The Vulnerable Query

The login form submitted a username and password via POST to `/login.php`. The application used these values to construct an authentication query by string concatenation:

```sql
SELECT * FROM users WHERE username = '$username' AND password = '$password'
```

If the query returns a row, the application creates a session as that user. No prepared statements or parameterisation were used, leaving the query open to injection through either field.

---

## Exploitation

### Payload

Submitted on the login form:

| Field | Value |
|---|---|
| Username | `' OR 1=1 -- ` |
| Password | *(any value)* |

### What the Query Became

```sql
SELECT * FROM users WHERE username = '' OR 1=1 -- ' AND password = '...'
```

The single quote closes the username string early. `OR 1=1` appends a condition that is always true, so the WHERE clause evaluates as true for every row in the table. The `-- ` comments out the remainder of the query, including the password check.

MySQL returns all matching rows — in this case, every user in the `users` table. The application takes the first row returned, which was the `admin` account, and creates an authenticated session for that user.

### Result

- Logged in successfully with no valid credentials.
- Session created as **admin**.
- The navigation bar displayed the admin username and an admin panel link (⚙).
- Full administrative access to the application was obtained in a single request.

---

## Context: Why Admin

The `users` table was previously enumerated via UNION-based SQL injection on the search endpoint (see `02-sql-injection-union-based.md`). The table structure included a `role` column:

```
id, username, password, email, role, address, created_at
```

The `admin` account occupies the first row of the table. When `OR 1=1` causes the query to match all rows, MySQL returns them in natural table order, and the application authenticates as whichever user appears first — in this case, `admin`.

This means the auth bypass does not just grant access as an arbitrary low-privilege user: it directly escalates to the highest-privilege account in the application.

---

# Finding: SQL Injection Authentication Bypass Leading to Privilege Escalation

## Description

The GingerByte login form accepted username and password values that were concatenated directly into a SQL authentication query without sanitisation or use of prepared statements. An attacker could inject SQL into the username field to replace the intended credential check with a condition that always evaluates as true, bypassing authentication entirely. Because MySQL returns rows in table insertion order and the admin account was inserted first, the bypass consistently resulted in authentication as the admin user.

## Evidence

**Request:**

```http
POST /login.php HTTP/1.1
Host: localhost:8080
Content-Type: application/x-www-form-urlencoded

username=%27+OR+1%3D1+--+&password=anything
```

**Decoded payload in username field:**

```
' OR 1=1 -- 
```

**Effective query executed:**

```sql
SELECT * FROM users WHERE username = '' OR 1=1 -- ' AND password = 'anything'
```

**Outcome:** Authenticated session created for `admin`. Admin panel accessible at `/admin.php`.

## Security Impact

An attacker with no knowledge of any credentials can gain administrative access to the application in a single request. Combined with the UNION-based injection identified on the search endpoint, the application exposes two independent SQL injection paths — one for data extraction and one for unauthenticated privilege escalation. Either vulnerability alone would represent a critical finding; together they demonstrate that the application's database interaction is comprehensively unprotected.

## Remediation

**Primary fix — use prepared statements for authentication:**

```php
// Secure approach
$stmt = $conn->prepare(
    "SELECT id, username, role FROM users WHERE username = ?"
);
$stmt->bind_param('s', $_POST['username']);
$stmt->execute();
$r = $stmt->get_result();
$user = $r->fetch_assoc();

if ($user && password_verify($_POST['password'], $user['password'])) {
    // Valid credentials — create session
    $_SESSION['username'] = $user['username'];
    $_SESSION['role']     = $user['role'];
}
```

The username is passed as a bound parameter, so the query structure cannot be altered by its content. The password is then verified using `password_verify()` against a stored hash — a separate step in PHP rather than a SQL comparison, which removes the password field from the query entirely.

**Additional recommendation — hash passwords:**

The credential dump obtained via UNION injection (see `02-sql-injection-union-based.md`) showed passwords stored in plaintext. Passwords should be stored as bcrypt hashes using `password_hash($password, PASSWORD_BCRYPT)` and verified with `password_verify()`. This ensures that even if the database is exfiltrated, credentials are not immediately usable.

---

# Key Lessons

- SQL injection on a login form does not require knowledge of any valid username or password. Injecting a tautology (`OR 1=1`) bypasses the credential check entirely.
- The `-- ` comment sequence in MySQL causes everything after it to be ignored, which is necessary here to neutralise the password condition that follows the username in the query.
- When `OR 1=1` matches all rows, the application receives the entire user table and typically processes the first row. Database insertion order determines which account is returned — making admin bypass reliable when the admin account was created first.
- Password comparison should never be performed inside the SQL query. Moving the password check into application code with `password_verify()` means that even if injection occurs, the attacker cannot manipulate the outcome of the password check.
- Prepared statements are the correct control. Attempting to sanitise input with `addslashes()` or `mysqli_real_escape_string()` is insufficient and can be bypassed under certain character set configurations.

---

# Assessment Summary

A SQL injection vulnerability in the `/login.php` authentication query allowed complete bypass of the login mechanism and direct escalation to administrative access. By injecting `' OR 1=1 -- ` into the username field, the intended `WHERE username = ? AND password = ?` check was replaced with a condition that returned all users. The application authenticated the session as the first returned row, which was the `admin` account.

No valid credentials were required. The attack required one request and was achieved through a browser form. Combined with the UNION-based injection on the search endpoint, which allowed full schema enumeration and plaintext credential extraction, the application has no protected database interaction surface. Both vulnerabilities share the same root cause: user input concatenated directly into SQL queries with no parameterisation.

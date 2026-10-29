# MD5 Role Escalation (Broken Access Control)

## Overview

Following the setup of an intentionally vulnerable PHP e-commerce application in my home penetration-testing lab, I assessed the user registration flow from Kali Linux without relying on knowledge of the server-side implementation.

The objective was to practise identifying and exploiting a privilege escalation vulnerability in which the application accepts an account role value from the client during registration, encoding it as an unsalted MD5 hash.

The assessment demonstrated how intercepting a registration request and substituting the correct hash value is sufficient to create an administrator account with no prior access or credentials.

> **Scope:** All testing documented here was performed against systems in my isolated home lab. The application was deliberately configured to be vulnerable for training purposes.

---

## Lab Scope

| System | Role |
|---|---|
| Kali Linux | Attacker |
| GingerByte (Docker, port 8080) | Target |

The target application is a fictional pet supplies shop built in PHP with a MySQL backend, running locally in Docker.

---

# 1. Identifying the Attack Surface

I began by browsing the application as an unauthenticated user.

The application had a registration page at `/register.php` that allowed new accounts to be created.

Before submitting the form, I viewed the page source to understand what fields were being sent to the server.

Within the form, I identified a hidden input field:

```html
<input type="hidden" name="userrole" value="c81e728d9d4c2f636f067f89cc14862c">
```

Hidden form fields are not visible to the user in the browser but are included in the POST request when the form is submitted.

The field name `userrole` suggested it was being used to set the role of the newly created account.

The value `c81e728d9d4c2f636f067f89cc14862c` appeared to be a hash. Its length of 32 hexadecimal characters is consistent with an MD5 digest.

---

# 2. Identifying the Hash Values

MD5 is a widely broken hashing algorithm. For common or short input values, pre-computed lookup tables (rainbow tables) can reverse the hash to its original input almost instantly.

I submitted the value to a public MD5 lookup:

```text
c81e728d9d4c2f636f067f89cc14862c → "2"
```

The hash was the MD5 digest of the integer `2`.

This suggested the application was encoding role identifiers as MD5 hashes of integers. A customer account used the hash of `2`.

It was reasonable to infer that an administrator account might use the hash of `1`:

```text
md5("1") = c4ca4238a0b923820dcc509a6f75849b
```

The page source also contained a comment that confirmed this inference, though it would not be visible in a real application.

---

# 3. Intercepting the Registration Request

I configured Burp Suite to intercept traffic from my browser and enabled Intercept in the Proxy tab.

I completed the registration form with a test account:

- **Username:** attacker
- **Email:** attacker@test.com
- **Password:** password123

Before submitting, I ensured Intercept was active.

When I submitted the form, Burp captured the following POST request:

```http
POST /register.php HTTP/1.1
Host: localhost:8080
Content-Type: application/x-www-form-urlencoded

username=attacker&email=attacker%40test.com&password=password123&userrole=c81e728d9d4c2f636f067f89cc14862c
```

The `userrole` parameter was visible in the request body as expected.

---

# 4. Substituting the Admin Hash

I modified the `userrole` parameter in Burp Suite before forwarding the request:

```
userrole=c81e728d9d4c2f636f067f89cc14862c
```

changed to:

```
userrole=c4ca4238a0b923820dcc509a6f75849b
```

I then forwarded the modified request.

The application returned a success message confirming the account had been created.

---

# 5. Verifying the Escalated Role

I navigated to `/login.php` and logged in using the credentials I had just registered.

After logging in, the navigation bar displayed an administrator panel link (⚙) that was not visible on standard customer accounts.

I was also able to access `/admin.php` directly.

To confirm the role stored in the database, I queried the users table:

```sql
SELECT username, role FROM users WHERE username = 'attacker';
```

Result:

```text
attacker | c4ca4238a0b923820dcc509a6f75849b
```

The database confirmed that the account had been created with the admin role hash.

---

# 6. Why the Application Was Vulnerable

The vulnerability existed because `register.php` read the role directly from the POST body without any server-side validation:

```php
$userrole = $_POST['userrole'] ?? md5('2');

$sql = "INSERT INTO users (username, password, email, role)
        VALUES ('$username', '$password', '$email', '$userrole')";
```

The server accepted whatever value was submitted for `userrole` and inserted it directly into the database. There was no check that the submitted value corresponded to a permitted role for a self-registered user.

The use of MD5 as an encoding mechanism also provided no real protection. MD5 is not a secure function for this purpose, and the hash values for short integer inputs are trivially reversible via lookup table.

---

# Finding: Privilege Escalation via Client-Supplied Role Parameter

## Description

The GingerByte registration endpoint accepted a `userrole` POST parameter and used its value to set the role of the newly created account. The role was encoded as an unsalted MD5 hash of a numeric role ID. An attacker could substitute the MD5 hash corresponding to an administrator role to register an account with full administrative privileges.

## Evidence

The following request was captured and modified in Burp Suite:

```http
POST /register.php HTTP/1.1
Host: localhost:8080
Content-Type: application/x-www-form-urlencoded

username=attacker&email=attacker%40test.com&password=password123&userrole=c4ca4238a0b923820dcc509a6f75849b
```

After forwarding this request, login with the registered credentials granted access to the administrator panel.

Database query confirmed the role hash was stored as submitted:

```text
attacker | c4ca4238a0b923820dcc509a6f75849b
```

## Security Impact

An unrestricted role parameter during registration allows any unauthenticated user to create an administrator account with no prior access or knowledge of existing credentials. An attacker with admin access could view and modify all user accounts, orders, and product data. This represents a complete bypass of the application's access control model.

## Remediation

The server should never accept role from the client. Role should be assigned server-side at registration, always defaulting to the least privileged role:

```php
// Secure approach
$role = 'customer'; // Always assigned server-side

$stmt = $conn->prepare(
    "INSERT INTO users (username, password, email, role) VALUES (?, ?, ?, ?)"
);
$stmt->bind_param('ssss', $username, $hashed_password, $email, $role);
$stmt->execute();
```

The hidden `userrole` field should be removed from the registration form entirely. Role should not be encoded as an MD5 hash; a plain string or integer with server-side validation is appropriate.

---

# Key Lessons

This exercise helped me understand how privilege escalation can be achieved through client-supplied input rather than exploiting a technical vulnerability in a running service.

I learned that:

- Hidden form fields are included in POST requests and are fully controlled by the client.
- The field name `userrole` in a registration form is a clear indicator that role assignment may be happening client-side.
- MD5 of short integer values is trivially reversible via public lookup tables.
- Viewing the page source before intercepting can reveal parameters and their expected values before a request is sent.
- The vulnerability is in the application's design — accepting role from the client — rather than in a missing input filter or encoding issue.

Manual interception made it straightforward to confirm exactly which parameter controlled the role and to verify the outcome at both the application and database level.

---

# Assessment Summary

The assessment identified a privilege escalation vulnerability in the registration flow of the GingerByte application.

Viewing the page source revealed a hidden `userrole` parameter containing an MD5 hash. Identifying the hash as `md5("2")` via lookup table, and inferring that `md5("1")` represented the admin role, provided the value needed to escalate privilege.

Intercepting the `POST /register.php` request in Burp Suite and substituting the admin hash before forwarding created an administrator account. Login with that account confirmed full administrative access.

The exercise demonstrated how a single client-controlled parameter in a registration flow can allow an unauthenticated attacker to bypass an application's entire access control model.

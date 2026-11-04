# IDOR — Password Reset / Account Takeover

GingerByte lab · CSTM preparation  
Covers: IDOR via hidden form field · account takeover · no CSRF token

---

## Exercise 1 — IDOR Password Reset Leading to Account Takeover

### What it is

Insecure Direct Object Reference (IDOR) occurs when an application trusts user-controlled input to identify a resource or perform an action without verifying that the requesting user is authorised to do so. Here, the password-change form in `/profile.php` renders a hidden `target_username` field pre-filled with the logged-in user's own username. The form looks like a standard self-service password change — no indication of a vulnerability is visible in the UI.

However, the server never checks whether the `target_username` value in the POST body matches the authenticated session. An attacker who intercepts the request in Burp Suite can silently change `target_username` to any other account and reset that account's password, with no knowledge of the current password required.

### Vulnerable code

**`/var/www/html/profile.php`**

```php
if ($_SERVER['REQUEST_METHOD'] === 'POST' && $_POST['action'] === 'change_password') {
    $target  = trim($_POST['target_username'] ?? '');
    $new_pw  = $_POST['new_password'] ?? '';
    $confirm = $_POST['confirm_password'] ?? '';

    // VULNERABILITY: no ownership check — target_username accepted from POST body
    // VULNERABILITY: plaintext password storage
    $t = $conn->real_escape_string($target);
    $p = $conn->real_escape_string($new_pw);
    $conn->query("UPDATE users SET password='$p' WHERE username='$t'");
}
```

The form that generates the hidden field:

```html
<input type="hidden" name="target_username" value="user2">
```

No server-side check compares `target_username` to `$_SESSION['username']`. The field is `type="hidden"`, so it is invisible in the browser — the vulnerability is only discoverable by intercepting the POST in a proxy.

### What an attacker observes

Submitting the password-change form as `user2` and intercepting with Burp Repeater reveals the hidden field in the POST body:

| POST parameter | Visible in browser UI | Value sent |
|---|---|---|
| `action` | No | `change_password` |
| `target_username` | No (hidden input) | `user2` |
| `new_password` | Yes | attacker-chosen |
| `confirm_password` | Yes | attacker-chosen |

Changing `target_username` from `user2` to `admin` and forwarding the request returns:

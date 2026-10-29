# Parameter Manipulation — Price Tampering & Discount Code Enumeration

## Overview

As part of a structured assessment of the GingerByte e-commerce application in my home lab, I examined the checkout flow for business logic vulnerabilities involving client-controlled parameters.

Two weaknesses were identified. The first involved a `price` parameter accepted by the add-to-basket endpoint — the application trusted a price value submitted by the client rather than looking it up server-side, allowing the price of any item to be set arbitrarily. The second involved a `discount_code` parameter on the checkout page — the application applied no rate limiting or lockout on discount code submissions, allowing automated enumeration of valid codes using Burp Intruder.

Both vulnerabilities shared the same root cause: the application accepted and acted on client-supplied values without validation.

> **Scope:** All testing documented here was performed against systems in my isolated home lab. The application was deliberately configured to be vulnerable for training purposes.

---

## Lab Scope

| System | Role |
|---|---|
| Kali Linux | Attacker |
| GingerByte (Docker, port 8080) | Target |

The target application is a fictional pet supplies shop built in PHP with a MySQL backend, running locally in Docker.

---

# Part 1 — Price Tampering

## 1. Identifying the Attack Surface

I began by browsing the application as a normal user to understand its functionality.

The application had:

- a product listing page
- individual product pages with an **Add to Basket** button
- a basket/cart page
- a checkout page

Adding an item to the basket sent a POST request to `/cart.php`.

I was interested in what parameters that request contained, as this is a common location for business logic weaknesses in e-commerce applications.

---

## 2. Intercepting the Add-to-Basket Request

I configured Burp Suite to intercept traffic from my browser and enabled Intercept in the Proxy tab.

When I clicked **Add to Basket** on a product page, Burp captured the following POST request:

```http
POST /cart.php HTTP/1.1
Host: localhost:8080
Content-Type: application/x-www-form-urlencoded

product_id=1&qty=1&price=12.99
```

The request body contained three parameters:

- `product_id` — the item being added
- `qty` — the quantity
- `price` — the price of the item

The presence of a `price` parameter in a client-submitted request immediately raised a concern.

In a correctly implemented e-commerce application, the price of a product should be determined by the server based on the product ID. It should not be accepted from the client, because the client can modify it to any value before the request is forwarded.

---

## 3. Modifying the Price Parameter

I modified the `price` parameter in Burp Suite before forwarding the request:

```
price=12.99
```

changed to:

```
price=0.01
```

I then forwarded the modified request.

---

## 4. Verifying the Tampered Price

After forwarding the modified request, I navigated to the basket page at `/cart.php`.

The basket displayed the item at the tampered price of **£0.01** rather than the original **£12.99**.

I then proceeded to the checkout page at `/checkout.php`.

The checkout total also reflected the tampered price.

This confirmed that the application had accepted and stored the client-supplied price without validating it against the server-side product database.

---

## 5. Why the Application Was Vulnerable

The vulnerability existed because `cart.php` read the price directly from the POST body and stored it in the user's session:

```php
$price = $_POST['price'] ?? $product['price'];
$_SESSION['cart_prices'][$pid] = $price;
```

At checkout, the application then preferred the session price over the database value:

```php
$effective_price = $_SESSION['cart_prices'][$pid] ?? $p['price'];
```

There was no check at either point that the submitted price matched the actual product price in the database.

---

# Part 2 — Discount Code Enumeration

## 1. Identifying the Target Parameter

With a product added to the basket, I navigated to the checkout page.

The checkout page included a discount code input field.

I submitted a test value to generate a request I could inspect in Burp Suite.

In Burp's HTTP history I located the resulting POST request and confirmed that the discount code was submitted as a parameter in the request body:

```
discount_code=1
```

This was the parameter I intended to fuzz.

---

## 2. Sending the Request to Intruder

With the request identified, I right-clicked it in Burp and selected **Send to Intruder**.

In the **Positions** section, I cleared all automatically placed payload markers.

I then manually highlighted the value of the `discount_code` parameter and clicked **Add §** to mark it as the injection point:

```
discount_code=§1§
```

I confirmed the attack type was set to **Sniper**.

In Sniper mode, Burp cycles through a single payload list and substitutes each value into the marked position in turn. This was appropriate here because I was testing one parameter with a list of candidate values.

---

## 3. Configuring the Payload List

I navigated to the **Payloads** tab.

I selected **Numbers** as the payload type and configured the range to generate integers from **100 to 199**.

This gave me 100 candidate values to test sequentially.

---

## 4. Running the Attack

I clicked **Start attack**.

Burp sent 100 requests, one for each candidate code, substituting the payload value into the `discount_code` parameter on each request.

I monitored the **Results** table as the attack ran, sorting by:

- **Length** — to identify responses that differed in size from the baseline
- **Status code** — to identify any responses that differed from the typical rejection response

A response with a different length or a distinct status code indicated that the application had accepted the submitted code rather than rejecting it.

---

## 5. Identifying the Valid Codes

As the attack ran, I sorted the results by **Length** column.

Across all 100 responses, 98 returned a consistent length indicating rejection. Two responses stood out with a noticeably shorter length, indicating the application had accepted those codes and returned a different page state:

| Payload | Result |
|---|---|
| 142 | ✓ Valid — distinct response length |
| 157 | ✓ Valid — distinct response length |
| All others | ✗ Invalid — consistent rejection length |

---

## 6. Verifying the Valid Codes

I navigated to the checkout page and entered code **142** into the discount code field, then clicked **Apply**.

The page updated to show the discount had been applied: **10% off** was deducted from the order total, and the summary displayed the discount amount and the reduced grand total.

This confirmed that code 142 was a live, working discount code that the application accepted — discovered entirely through response length comparison in an automated Intruder attack.

---

# Finding 1: Price Tampering via Unvalidated Client Parameter

## Description

The GingerByte application accepted a `price` POST parameter when adding items to the basket and used this value in the basket and checkout without server-side validation. An attacker could modify this parameter to purchase any product at an arbitrary price.

## Evidence

The following request was captured and modified in Burp Suite:

```http
POST /cart.php HTTP/1.1
Host: localhost:8080
Content-Type: application/x-www-form-urlencoded

product_id=1&qty=1&price=0.01
```

After forwarding this request, the basket and checkout both displayed a total of **£0.01** for an item with a database price of **£12.99**.

## Security Impact

An unrestricted price parameter allows an attacker to purchase any item at any price, including negative values or zero. This has direct financial impact on an organisation running such an application, as an attacker could acquire goods for negligible cost. The issue also undermines the integrity of the order system.

## Remediation

The server should never accept price from the client. The authoritative price should always be retrieved from the product database using the submitted product ID:

```php
// Secure approach
$r = $conn->query("SELECT price FROM products WHERE id = " . (int)$_POST['product_id']);
$product = $r->fetch_assoc();
$price = $product['price']; // Always from the database
```

The `price` parameter should be removed from the client-side form entirely to avoid suggesting that it is a valid input.

---

# Finding 2: Discount Code Enumeration via Unrestricted Parameter Fuzzing

## Description

The GingerByte checkout page accepted a discount code parameter without implementing any rate limiting, account lockout, or request throttling. An attacker could use Burp Intruder to submit a large number of candidate values in rapid succession and identify valid codes by comparing application responses.

## Evidence

The following request was sent to Intruder and used as the attack template:

```http
POST /checkout.php HTTP/1.1
Host: localhost:8080
Content-Type: application/x-www-form-urlencoded

discount_code=§payload§
```

Payloads: integers 100–199 (Sniper, Numbers payload type).

Two valid discount codes — **142** (10% off) and **157** (15% off) — were identified by their distinct response lengths compared to the 98 rejection responses. Code 142 was verified on the checkout page; the discount was applied and the order total dropped accordingly.

## Security Impact

Without rate limiting, an attacker can enumerate discount codes, voucher codes, or promotional values automatically. Depending on the application, a valid code may result in financial loss through unauthorised discounts, or expose promotional campaigns to misuse. The absence of any lockout mechanism means enumeration can proceed indefinitely without triggering a block or alert.

## Remediation

- Implement server-side rate limiting on discount code submissions (e.g., a maximum of five attempts per session or IP within a time window).
- Use codes that are sufficiently long and randomly generated to prevent sequential enumeration.
- Consider single-use codes that are invalidated once redeemed.
- Log and alert on repeated failed discount code submissions from a single source.

---

# Key Lessons

These exercises helped me understand how business logic vulnerabilities differ from more technical vulnerability classes, and how automated fuzzing can compound client-trust issues.

I learned that:

- Parameters in POST bodies are fully controlled by the client and cannot be trusted.
- The presence of a `price` field in a client request is a clear indicator that the application may be trusting input it should be calculating itself.
- The same principle applies to any value the server should be authoritative over — discount codes should be validated server-side against a database, not simply accepted if submitted.
- Burp Suite's Intercept function allows individual parameters to be modified before a request reaches the server.
- Sniper mode is suited to fuzzing a single parameter with a list of values — comparing response lengths is a reliable way to identify a different application behaviour without reading every response manually.
- Rate limiting is a critical control on any endpoint that accepts user-submitted values that can be enumerated — its absence means an attacker can test candidates at speed without restriction.

---

# Assessment Summary

The assessment identified two related vulnerabilities in the GingerByte checkout flow, both rooted in the application's acceptance of client-supplied values without server-side validation.

The first vulnerability allowed price tampering: intercepting the `POST /cart.php` request revealed a client-supplied `price` parameter. Modifying this to `0.01` caused the basket and checkout to process the tampered price, allowing goods to be purchased at an arbitrary value.

The second vulnerability allowed discount code enumeration: the checkout page accepted a `discount_code` parameter with no rate limiting. Sending the request to Burp Intruder with a Sniper attack and a sequential number list (100–199) as the payload produced two valid codes — **142** and **157** — within 100 requests, identifiable by their distinct response lengths compared to all rejection responses. Code 142 was verified on the page, applying a 10% discount to the order total.

Both findings demonstrate how a single design decision — trusting the client to supply a value the server should control — can introduce exploitable weaknesses in business-critical functions.

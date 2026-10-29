# Price Tampering (Parameter Manipulation)

## Overview

Following the setup of an intentionally vulnerable PHP e-commerce application in my home penetration-testing lab, I assessed the checkout flow from Kali Linux without relying on knowledge of the server-side implementation.

The objective was to practise identifying and exploiting a business logic vulnerability in which the application trusts a price value submitted by the client, rather than looking it up server-side.

The assessment demonstrated how intercepting and modifying a single POST parameter can allow an attacker to purchase goods at an arbitrary price.

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

I began by browsing the application as a normal user to understand its functionality.

The application had:

- a product listing page
- individual product pages with an **Add to Basket** button
- a basket/cart page
- a checkout page

Adding an item to the basket sent a POST request to `/cart.php`.

I was interested in what parameters that request contained, as this is a common location for business logic weaknesses in e-commerce applications.

---

# 2. Intercepting the Add-to-Basket Request

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

# 3. Modifying the Price Parameter

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

# 4. Verifying the Tampered Price

After forwarding the modified request, I navigated to the basket page at `/cart.php`.

The basket displayed the item at the tampered price of **£0.01** rather than the original **£12.99**.

I then proceeded to the checkout page at `/checkout.php`.

The checkout total also reflected the tampered price.

This confirmed that the application had accepted and stored the client-supplied price without validating it against the server-side product database.

---

# 5. Why the Application Was Vulnerable

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

# Finding: Price Tampering via Unvalidated Client Parameter

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

# Key Lessons

This exercise helped me understand how business logic vulnerabilities differ from more technical vulnerability classes.

I learned that:

- Parameters in POST bodies are fully controlled by the client and cannot be trusted.
- The presence of a `price` field in a client request is a clear indicator that the application may be trusting input it should be calculating itself.
- Burp Suite's Intercept function allows individual parameters to be modified before a request reaches the server.
- Modifying the price in the request is sufficient — no server-side knowledge is required to exploit this.
- The vulnerability exists at the point where the application stores the client-supplied value rather than looking it up, which means the issue is in application design rather than a missing input filter.

Manual interception made it straightforward to isolate exactly which parameter was responsible and confirm the effect at each stage of the purchase flow.

---

# Assessment Summary

The assessment identified a price tampering vulnerability in the add-to-basket function of the GingerByte application.

Intercepting the `POST /cart.php` request in Burp Suite revealed a client-supplied `price` parameter. Modifying this parameter to `0.01` and forwarding the request caused the basket and checkout to display and process the tampered price without any server-side validation.

The exercise demonstrated how a single unvalidated parameter in a business-critical function can allow an attacker to undermine the financial integrity of an e-commerce application.

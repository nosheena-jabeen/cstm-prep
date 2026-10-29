# Account Compromise — Burp Intruder (Pitchfork & Clusterbomb)

## Overview

As part of a structured assessment of a login endpoint in my home lab, I used Burp Suite's Intruder tool to perform two credential attacks against the application's login form, using different attack types to demonstrate how each is suited to different scenarios.

The first attack used **Pitchfork** mode to test paired credential combinations — appropriate when a list of username and password pairs is already available or suspected.

The second attack used **Clusterbomb** mode to test every possible combination of a username list against a password list — appropriate when the correct username-to-password pairing is unknown.

Both attacks demonstrated that the application's login endpoint had no account lockout or rate limiting, allowing repeated authentication attempts without restriction.

> **Scope:** All testing documented here was performed against systems in my isolated home lab. The application was deliberately configured to be vulnerable for training purposes.

---

## Lab Scope

| System | Role |
|---|---|
| Kali Linux | Attacker |
| Target application (local lab) | Target |

---

# Exercise 1 — Pitchfork Attack

## 1. Capturing the Login Request

I opened Burp Suite and launched the Burp browser from the Proxy tab.

I navigated to the login page of the target application and submitted a deliberately incorrect set of credentials to generate a POST request I could inspect:

```
username=testuser
password=testpass
```

In Burp's HTTP history I confirmed the login request had been captured and identified the parameters being used to submit credentials.

---

## 2. Sending the Request to Intruder

I right-clicked the captured login request and selected **Send to Intruder**.

In the **Intruder** tab, I navigated to the **Positions** section and clicked **Clear** to remove all automatically placed payload markers.

I then manually highlighted the value of the `username` field and clicked **Add §**, then did the same for the `password` field:

```
username=§testuser§&password=§testpass§
```

I set the attack type to **Pitchfork**.

Pitchfork mode uses multiple payload lists — one per marked position — and iterates through them in parallel. Position 1 takes its value from payload list 1 at the same index as position 2 takes its value from payload list 2. The lists are stepped through together rather than combined exhaustively.

This was appropriate here because I had a set of usernames I wanted to pair with corresponding passwords rather than test every possible combination.

---

## 3. Configuring the Payload Lists

I navigated to the **Payloads** tab.

For **Payload set 1** (usernames), I entered the following values:

```
admin
manager
test
support
```

For **Payload set 2** (passwords), I entered the following values:

```
admin123
password
qwerty
letmein
```

With Pitchfork, each pair is tested together: `admin:admin123`, `manager:password`, `test:qwerty`, `support:letmein`. This produces four attempts, not sixteen.

---

## 4. Running the Attack

I clicked **Start attack**.

Burp sent four requests, each substituting the corresponding pair of values from the two payload lists.

I monitored the results, looking for:

- A **different status code** — for example, a `302 Found` redirect indicating a successful login, compared to a `200 OK` returned for failed attempts
- A **different response length** — a successful login response typically differs in size from an error page

---

## 5. Identifying the Valid Credentials

By comparing status codes and response lengths across the four results, I identified one request that produced a `302 Found` response with a noticeably different length from the three failed login responses:

| Request | Username | Password | Status | Result |
|---|---|---|---|---|
| 1 | admin | admin123 | 302 | ✓ Valid |
| 2 | manager | password | 200 | ✗ Failed |
| 3 | test | qwerty | 200 | ✗ Failed |
| 4 | support | letmein | 200 | ✗ Failed |

The `admin:admin123` pair was confirmed as valid. This was the only result because Pitchfork pairs entries positionally — if the correct credentials were not at the same index in both lists, they would not have been tested together.

---

# Exercise 2 — Clusterbomb Attack

## 1. Reconfiguring the Attack

Using the same captured login request already loaded in Intruder, I changed the attack type from **Pitchfork** to **Cluster bomb**.

The payload markers remained the same:

```
username=§testuser§&password=§testpass§
```

Clusterbomb mode uses multiple payload lists and tests every possible combination of values across them. For two lists of four entries each, this produces 4 × 4 = 16 total requests.

This differed from the earlier Pitchfork attack, which paired entries positionally and produced only four requests. Clusterbomb is appropriate when the correct username-to-password pairing is not known — every username is tested against every password.

---

## 2. Updating the Payload Lists

I navigated to the **Payloads** tab and replaced both lists.

For **Payload set 1** (usernames), I entered the following values:

```
admin
alice
bob
charlie
```

For **Payload set 2** (passwords), I entered the following values:

```
admin123
password1
letmein
qwerty123
```

With Clusterbomb, Burp generated all 16 combinations:

```
admin:admin123    admin:password1    admin:letmein    admin:qwerty123
alice:admin123    alice:password1    alice:letmein    alice:qwerty123
bob:admin123      bob:password1      bob:letmein      bob:qwerty123
charlie:admin123  charlie:password1  charlie:letmein  charlie:qwerty123
```

---

## 3. Running the Attack

I clicked **Start attack**.

Burp sent 16 requests, one for each combination of username and password from the two payload lists.

I monitored the results, looking for:

- A **different status code** — for example, a `302 Found` redirect or a `200 OK` response that differed from the failed login pattern
- A **different response length** — a successful login response typically differs in size from an error page

---

## 4. Identifying the Valid Credentials

By comparing status codes and response lengths across the 16 results, I identified three requests that returned `302 Found` with a response length of approximately 23,000 bytes, compared to ~29,500–29,700 bytes for all failed login attempts:

| Request | Username | Password | Status | Length |
|---|---|---|---|---|
| 1 | admin | admin123 | 302 | ~23,000 |
| 11 | bob | letmein | 302 | ~23,215 |
| 16 | charlie | qwerty123 | 302 | ~23,212 |

Three valid credential pairs were discovered. The remaining 13 requests all returned `200 OK`, indicating failed authentication.

This demonstrated a key advantage of Clusterbomb over Pitchfork: where Pitchfork only found `admin:admin123` because it paired entries positionally, Clusterbomb found all valid pairs by testing every combination regardless of index alignment.

---

# Finding: Account Compromise via Credential Brute Force

## Description

The application's login endpoint did not implement account lockout or rate limiting. An attacker with candidate username and password lists could use Burp Intruder to test combinations systematically and identify valid credentials from the application's differing response.

Pitchfork is effective when credential pairs are already suspected — for example, from a data breach or known naming conventions. Clusterbomb is effective when the correct pairing is unknown, as it tests every combination and will find all valid accounts in the lists.

## Evidence

The following request was used as the attack template in both exercises:

```http
POST /login.php HTTP/1.1
Host: localhost:8080
Content-Type: application/x-www-form-urlencoded

username=§testuser§&password=§testpass§
```

**Pitchfork** — Payload set 1 (usernames): `admin`, `manager`, `test`, `support` | Payload set 2 (passwords): `admin123`, `password`, `qwerty`, `letmein` | 4 paired attempts | Valid pair: `admin:admin123`

**Clusterbomb** — Payload set 1 (usernames): `admin`, `alice`, `bob`, `charlie` | Payload set 2 (passwords): `admin123`, `password1`, `letmein`, `qwerty123` | 16 total combinations | Valid pairs: `admin:admin123`, `bob:letmein`, `charlie:qwerty123`

## Security Impact

An application with no account lockout allows an attacker to test credential lists without restriction. Pitchfork can confirm suspected pairs quickly. Clusterbomb can enumerate all valid accounts from a list, exposing multiple accounts in a single attack run. A successful login provides the attacker with full access to the compromised account's data and functionality.

## Remediation

- Implement account lockout or progressive delays after a defined number of failed login attempts.
- Apply rate limiting per IP address and per username on authentication endpoints.
- Use multi-factor authentication to reduce the impact of credential compromise.
- Monitor and alert on repeated failed login attempts from a single source.

---

# Key Lessons

These exercises introduced both Pitchfork and Clusterbomb attack types and clarified when each is appropriate.

I learned that:

- **Pitchfork** iterates through multiple payload lists in parallel, pairing corresponding entries rather than combining them exhaustively. It produces N requests for lists of length N and is suited to testing known or suspected credential pairs.
- **Clusterbomb** tests every possible combination across multiple payload lists. It produces N × M requests for lists of length N and M respectively, and is the right choice when the correct pairing is unknown.
- A `302 Found` response on a login endpoint typically indicates a redirect following successful authentication and is the clearest indicator of a valid credential pair in Intruder results.
- Response length differences confirm success when the status code alone is ambiguous.
- Clusterbomb revealed three valid accounts compared to Pitchfork's one, demonstrating that exhaustive combination testing is more comprehensive when account-to-password relationships are not known in advance.
- Account lockout is the primary control that makes both techniques impractical — without it, an attacker is limited only by list size and network speed.

---

# Assessment Summary

The assessment targeted the login endpoint of the target application using two Burp Intruder attack types.

**Pitchfork** paired four candidate usernames with corresponding passwords and confirmed `admin:admin123` as a valid credential pair from its `302 Found` response, while three other pairs returned `200 OK`.

**Clusterbomb** tested all 16 combinations of four usernames and four passwords. Three valid credential pairs were identified — `admin:admin123`, `bob:letmein`, and `charlie:qwerty123` — each returning a `302 Found` response with a response length approximately 6,000 bytes smaller than failed login responses.

Both attacks confirmed that the login endpoint permitted unrestricted brute force attempts with no account lockout or rate limiting in place. Clusterbomb's exhaustive approach exposed three accounts where Pitchfork's positional pairing found only one, illustrating the trade-off between speed and coverage across the two attack types.

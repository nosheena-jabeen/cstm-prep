# DNS Fundamentals

## Protocol Overview

- Default Transport: `UDP`
- TCP Usage: `TCP` is used for large responses and zone transfers
- Default Port: `53`
- Security by Default: Minimal (plaintext and unauthenticated)

---

## 1. What DNS Does

The Domain Name System (DNS) translates human-readable domain names into IP addresses that computers use to communicate.

When a user enters a domain such as `example.com`, the system must determine the IP address associated with that name before a connection can be established.

Without DNS, users would need to memorise numerical IP addresses for every service they want to access.

---

## 2. Why DNS Matters in Security

DNS is core internet infrastructure. From a security perspective it can:

- Expand the attack surface during reconnaissance  
- Reveal internal naming conventions  
- Expose mail servers and subdomains  
- Allow full domain enumeration if misconfigured  
- Be abused for redirection, phishing, or data exfiltration  

Understanding DNS supports both offensive reconnaissance and defensive monitoring.

---

## 3. How DNS Resolution Works

When a domain is requested:

1. The local system checks its cache.
2. If not found, it queries a configured DNS resolver.
3. The resolver queries:
   - Root servers
   - Top-Level Domain (TLD) servers (e.g., `.com`)
   - Authoritative name servers
4. The authoritative server returns the official record.
5. The resolver sends the result back to the client.

The browser can then connect to the correct IP address.

---

## 4. Forward vs Reverse Lookups

### Forward Lookup

Translates a domain name into an IP address.

Example:

```bash
dig example.com A
```

### Reverse Lookup

Translates an IP address into a hostname (if a `PTR` record exists).

Example:

```bash
dig -x 8.8.8.8
```

---

## 5. DNS Record Types

DNS records define how a domain behaves.

| Record | Purpose |
|---------|----------|
| `A` | Maps domain to IPv4 address |
| `AAAA` | Maps domain to IPv6 address |
| `CNAME` | Alias record |
| `MX` | Mail server location |
| `NS` | Authoritative name servers |
| `PTR` | Reverse lookup record |
| `SOA` | Zone configuration and administrative data |
| `TXT` | Arbitrary text (often verification/SPF) |
| `SPF` | Email sender validation |
| `DKIM` | Email authenticity verification |

---

## 6. What is a DNS Zone?

A DNS zone is a collection of records for a domain.

It can be thought of as the administrative container for:

- Subdomains  
- Mail configuration  
- Name server records  
- Service records  

Zones are stored on authoritative name servers.

---

## 7. DNS Tools

Common command-line tools:

- `nslookup`
- `dig`
- `host`
- `dnsenum`

---

# Practical Lessons

## Lesson 1 – Identify DNS Services on a Network

### Objective
Determine whether a host is running DNS.

### Command

```bash
nmap -sTU -p 53 <target>
```

### Observe

- Is TCP `53` open?
- Is UDP `53` open?
- Is the service identified as DNS?

---

## Lesson 2 – Perform Forward Lookups

Using `nslookup`:

```bash
nslookup example.com
```

Using `dig`:

```bash
dig example.com A
```

Using `host`:

```bash
host example.com
```

Reflection:

- Are multiple `A` records returned?
- Is there an `AAAA` record?
- Does the response suggest CDN usage?

---

## Lesson 3 – Perform Reverse Lookups

```bash
nslookup 8.8.8.8
```

Or:

```bash
dig -x 8.8.8.8
```

Reflection:

- Does a `PTR` record exist?
- What does the hostname suggest about the service?

---

## Lesson 4 – Subdomain Enumeration

Create a simple wordlist:

```bash
echo -e "www\nmail\nftp\nadmin\nvpn" > subdomains.txt
```

Loop through the list:

```bash
for sub in $(cat subdomains.txt); do host $sub.example.com; done
```

Observe:

- Which subdomains resolve?
- Which return `NXDOMAIN`?

---

## Lesson 5 – Zone Transfer Testing

Identify authoritative name servers:

```bash
dig NS example.com
```

Attempt a zone transfer:

```bash
dig axfr example.com @nameserver
```

Security Insight:

If successful, a zone transfer may expose all DNS records for a domain.

---

## Lesson 6 – Automated Enumeration with dnsenum

```bash
dnsenum example.com
```

Compare:

- What did automation discover?
- What did manual enumeration reveal?
- Are the results identical?

---

## Key Security Takeaways

- DNS operates in plaintext by default.
- Misconfigurations can expose infrastructure.
- Zone transfers should be restricted.
- DNS logging is important for detection.
- Manual enumeration often reveals more than automation alone.

---

## Summary

DNS translates domain names into IP addresses, but it also provides valuable infrastructure insight. Understanding DNS supports reconnaissance, attack surface mapping, and defensive monitoring.

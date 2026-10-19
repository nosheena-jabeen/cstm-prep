# SNMP Fundamentals (Simple Network Management Protocol)

## Overview

Before building and assessing a vulnerable SNMP service in my home lab, I wanted a clear reference for what SNMP actually is, how it works at a protocol level, and why a default installation is frequently one of the first things a penetration tester checks on a target host.

This document is a general-purpose summary rather than a record of a specific lab session — the commands here use placeholder targets and are expanded on with real IPs and evidence in the lab write-ups that follow.

---

## 1. What is SNMP?

Simple Network Management Protocol (SNMP) is a protocol used to monitor and manage networked devices — servers, routers, switches, printers, and anything else that implements it. It allows a management system to query a device for status information and, depending on configuration, to write configuration values back to it.

An SNMP-enabled device runs an **agent** — a small service that listens for incoming queries and responds with data from its **Management Information Base (MIB)**, a structured tree of values describing the device's state.

---

## 2. Protocol Overview

| Property             | Detail                                                                 |
| --------------------- | ----------------------------------------------------------------------- |
| Protocol              | SNMP (Simple Network Management Protocol)                               |
| Default port          | UDP/161 (agent queries); UDP/162 (traps, agent-initiated alerts)        |
| Transport             | UDP — connectionless, no handshake, no session state                    |
| Current versions      | v1 (legacy), v2c (most common in practice), v3 (adds encryption and strong authentication) |
| Security by default   | Minimal in v1/v2c — relies on a shared plaintext "community string" rather than real authentication |

SNMP runs over UDP rather than TCP, which has two practical implications for security testing: it makes the service fast to query in bulk, and it makes UDP scanning with `nmap -sU` the right approach for discovery rather than a standard TCP scan.

---

## 3. SNMP Versions

| Version | Authentication          | Encryption | Common Use                                        |
| -------- | ------------------------ | ----------- | --------------------------------------------------- |
| v1       | Community string         | None        | Legacy — still seen on older network hardware       |
| v2c      | Community string         | None        | Most widely deployed version in practice            |
| v3       | Username + auth protocol | Yes (`priv`) | Recommended for any new deployment; rarely seen by default |

v1 and v2c treat the community string essentially as a password sent in cleartext with every query. Anyone on the network who can see UDP traffic can capture both the community string and the response data. v3 addressed these weaknesses with proper authentication and optional encryption, but v1/v2c remain the default for most SNMP-enabled devices shipped from the factory.

---

## 4. Community Strings

A community string is the only access control mechanism in SNMPv1 and v2c. It is included in plaintext with every request, and the agent compares it against its configured list before responding.

| String type        | Typical default value | Effect                                      |
| ------------------- | ---------------------- | --------------------------------------------- |
| Read-only (`ro`)    | `public`               | Allows queries (GET, WALK) — no writes        |
| Read-write (`rw`)   | `private`              | Allows queries AND configuration writes       |

Other commonly used defaults encountered in the wild include `cisco`, `manager`, `ilmi`, `read`, and `write`. These ship from vendors as defaults and are frequently left unchanged.

Because community strings are sent in plaintext over UDP, capturing one on the wire is straightforward — and because they are often defaults, brute-forcing them from a short wordlist is highly reliable in practice.

---

## 5. The MIB Tree and OIDs

The data SNMP exposes is organised as a tree, called the Management Information Base (MIB). Each node in the tree has a numeric identifier called an **Object Identifier (OID)** — a dot-separated sequence of numbers uniquely identifying that piece of data across all devices and vendors.

Common system OIDs:

| OID              | Description                              |
| ----------------- | ------------------------------------------ |
| `1.3.6.1.2.1.1.1.0` | `sysDescr` — OS and kernel version string  |
| `1.3.6.1.2.1.1.3.0` | `sysUpTime` — time since last reboot       |
| `1.3.6.1.2.1.1.4.0` | `sysContact` — configured contact details  |
| `1.3.6.1.2.1.1.5.0` | `sysName` — configured hostname            |
| `1.3.6.1.2.1.1.6.0` | `sysLocation` — configured physical location |

These are standardised across all SNMPv1/v2c devices. Beyond the system branch, OIDs exist for network interfaces, routing tables, running processes, installed software, open TCP/UDP ports, and much more — all queryable without authentication if the community string is known or guessed.

---

## 6. Why SNMP Matters in Security

SNMP is consistently valuable in penetration testing because a single default community string query can return an unusually broad picture of a target:

- **OS fingerprinting** — exact kernel version, build flags, architecture
- **Hostname** — confirms the target's identity
- **Running processes** — reveals every service running on the host, including ones not yet discovered via port scanning
- **Network interfaces and routing** — internal network topology
- **Open ports** — TCP and UDP listeners the host is aware of
- **Installed packages** — software inventory useful for identifying vulnerable versions
- **Active sessions** — usernames of currently logged-in users

This means SNMP with a default community string can effectively replace or supplement a detailed Nmap scan — and can reveal services the port scan might have missed, particularly internal or management interfaces not exposed to the scanning host directly.

The read-write community string (`private`, or `rwcommunity`) is a more severe finding still: it allows an attacker to write configuration values back to the device, which on network hardware can mean rerouting traffic, and on servers can mean altering system parameters.

---

## 7. Identifying SNMP Services

Scan for SNMP on UDP/161:

```bash
sudo nmap -sU -p 161 <target>
```

Add NSE script for immediate system information:

```bash
sudo nmap -sU -p 161 --script=snmp-info <target>
```

---

## 8. Brute-forcing Community Strings

Create a wordlist of common community strings:

```bash
echo -e "public\nprivate\ncisco\nilmi" > community.txt
```

Use `onesixtyone` to test them against the target:

```bash
onesixtyone -c community.txt <target>
```

`onesixtyone` is fast because it uses UDP and fires requests without waiting for a connection — it can test thousands of strings per second.

---

## 9. Enumerating SNMP Data

Bulk walk of the entire MIB tree using a confirmed community string:

```bash
snmpwalk -v2c -c <community> <target> | tee snmpwalk_output
```

Query a specific OID directly:

```bash
snmpget -v2c -c <community> <target> <OID>
```

Metasploit's enumeration module:

```bash
msfconsole
use auxiliary/scanner/snmp/snmp_enum
set RHOSTS <target>
set COMMUNITY <community>
run
```

---

## Practical Methodology

**Step 1 — Discover:** confirm SNMP is running on UDP/161 and check for immediate information disclosure via `snmp-info`.

**Step 2 — Brute-force:** confirm whether default community strings are in use with `onesixtyone`.

**Step 3 — Enumerate broadly:** `snmpwalk` to capture the full MIB tree — note the volume of data returned as evidence.

**Step 4 — Query specifically:** `snmpget` against known OIDs to pull targeted, clearly-evidenced values.

**Step 5 — Structured enumeration:** Metasploit's `snmp_enum` for readable, categorised output — particularly useful for identifying running processes and cross-service visibility.

---

## Key Security Takeaways

- SNMPv1 and v2c have no real authentication — a community string sent in plaintext is the only access control.
- Default community strings (`public`, `private`) are left unchanged on a significant proportion of real-world devices.
- A single `snmpwalk` with `public` can return thousands of data points with no prior access or exploit.
- SNMP's process and port tables make it a map of other services running on the same host — finding SNMP first can eliminate the need to separately discover NFS, DNS, SMB, and other services via port scanning.
- v3 with authentication and encryption (`authPriv`) is the only version that addresses these weaknesses; v1/v2c should not be used on untrusted networks.

---

## Related Labs

For a hands-on build of a deliberately vulnerable SNMP service and a full exploitation walkthrough using these concepts, see:

`02-vulnerable-snmp-server.md`
`03-default-community-string-exploitation.md`

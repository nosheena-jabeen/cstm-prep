# Networking Foundations – Technical Interview Notes

This document consolidates core networking concepts commonly assessed in technical interviews. Each section explains the protocol or concept, how it works, and why it matters from a security perspective.

---

# 1. Networking Models

## OSI Model (Open Systems Interconnection)

The OSI model is a seven-layer conceptual framework used to describe how data moves from one system to another over a network. While it is not directly implemented in modern systems, it is essential for troubleshooting and understanding protocol interaction.

The seven layers are:

1. Physical – Handles raw bit transmission (cables, voltages, wireless signals).
2. Data Link – Responsible for MAC addressing and switching.
3. Network – Logical addressing and routing (IP).
4. Transport – End-to-end communication (TCP/UDP, port numbers).
5. Session – Session establishment and teardown.
6. Presentation – Data formatting, encoding, encryption.
7. Application – User-facing protocols (HTTP, DNS, FTP, SMTP).

From a security perspective, many attacks map cleanly to OSI layers:
- ARP spoofing → Layer 2
- IP spoofing → Layer 3
- TCP SYN flood → Layer 4
- XSS → Layer 7

Understanding the OSI model allows structured troubleshooting and layered security analysis.

---

## DoD / TCP-IP Model

The Department of Defense (DoD) model, also known as the TCP/IP model, simplifies networking into four layers:

1. Network Access
2. Internet
3. Transport
4. Application

Unlike OSI, this model reflects how real-world protocols are implemented. It combines several OSI layers into broader categories.

The TCP/IP model is more practical, while OSI is more educational.

---

# 2. Core Networking Protocols

## Internet Protocol (IP)

IP provides logical addressing and routing between networks. It is connectionless and does not guarantee delivery.

IPv4 uses 32-bit addresses (e.g., 192.168.1.10).
IPv6 uses 128-bit addresses.

IP packets contain:
- Source address
- Destination address
- TTL (Time To Live)
- Protocol identifier

Security relevance:
- IP spoofing attacks
- TTL manipulation (used in techniques like firewalking)

---

## DHCP (Dynamic Host Configuration Protocol)

DHCP automatically assigns network configuration to devices.

It follows the DORA process:

1. Discover – Client broadcasts request.
2. Offer – Server offers an IP address.
3. Request – Client requests offered address.
4. Acknowledge – Server confirms lease.

DHCP assigns:
- IP address
- Subnet mask
- Default gateway
- DNS servers

Security risks:
- Rogue DHCP servers
- DHCP starvation attacks
- Network redirection attacks

---

## DNS (Domain Name System)

DNS translates domain names into IP addresses.

Resolution process:
1. Local cache lookup
2. Recursive resolver
3. Root server
4. TLD server
5. Authoritative server

DNS uses port 53 (UDP primarily).

### SOA Record (Start of Authority)

The SOA record defines administrative information about a DNS zone, including:
- Primary name server
- Contact email
- Serial number
- Refresh and retry timers

SOA is critical for zone transfers and DNS replication.

Security relevance:
- Zone transfer misconfiguration
- DNS enumeration
- Data leakage

---

## Hosts File

The hosts file maps domain names to IP addresses locally before DNS is queried.

Locations:

Windows:
```
C:\Windows\System32\drivers\etc\hosts
```

Linux:
```
/etc/hosts
```

Security relevance:
- Malware persistence
- DNS bypass
- Redirection attacks

---

# 3. Addressing and Routing

## CIDR (Classless Inter-Domain Routing)

CIDR replaces rigid class-based addressing.

Example:
```
192.168.1.0/24
```

The `/24` indicates the number of bits used for the network portion.

CIDR enables efficient IP allocation and subnetting.

---

## Distance Vector Routing Protocols

Distance vector protocols determine routes based on distance metrics.

Examples:
- RIP (Routing Information Protocol)
- EIGRP

They share routing tables periodically and calculate best paths based on hop count or composite metrics.

Security concern:
- Route poisoning
- Rogue route injection

---

## Port Address Translation (PAT)

PAT allows multiple internal hosts to share a single public IP by translating internal IP addresses using unique source ports.

Also called NAT overload.

Security relevance:
- Hides internal addressing
- Breaks some peer-to-peer protocols
- Can complicate logging and attribution

---

## MAC Addresses and OUI

A MAC address is a 48-bit hardware identifier assigned to network interfaces.

The first 24 bits represent the Organizationally Unique Identifier (OUI), which identifies the manufacturer.

Example:
```
00:1A:2B:XX:XX:XX
```

Security relevance:
- MAC spoofing
- Device fingerprinting

---

# 4. Network Segmentation and Infrastructure

## VLANs (Virtual Local Area Networks)

VLANs logically segment a network into separate broadcast domains.

Benefits:
- Reduced broadcast traffic
- Improved performance
- Improved security separation

VLAN misconfigurations can allow VLAN hopping attacks.

---

## TACACS+

Terminal Access Controller Access-Control System (TACACS+) provides centralised authentication for network devices.

Used for:
- Router login
- Switch administration
- Device access auditing

Runs on TCP port 49.

---

## LDAP (Lightweight Directory Access Protocol)

LDAP is used to query and manage directory services, such as Active Directory.

Ports:
- 389 (LDAP)
- 636 (LDAPS)

Security relevance:
- Credential exposure
- LDAP injection
- Directory enumeration

---

# 5. Network Security Mechanisms

## ARP-Based MITM

ARP maps IP addresses to MAC addresses on local networks.

In ARP poisoning:
- Attacker sends forged ARP replies.
- Victim associates attacker’s MAC with gateway IP.
- Traffic flows through attacker.

This enables:
- Traffic interception
- Credential capture
- Session hijacking

---

## Firewalking

Firewalking is a reconnaissance technique used to determine which ports are permitted through a firewall.

It manipulates TTL values to infer firewall filtering behaviour.

Used to map firewall rulesets indirectly.

---

## WPA (Wi-Fi Protected Access)

WPA secures wireless networks using encryption and authentication.

WPA2 uses AES.
WPA3 improves protection against brute-force attacks and offline dictionary attacks.

---

## IPSec

IPSec secures IP traffic using:

- Authentication (AH)
- Encryption (ESP)

Modes:
- Transport mode
- Tunnel mode (commonly used in VPNs)

IPSec ensures confidentiality, integrity, and authenticity.

---

## Closed Ports – Why They Matter

Closed ports indicate a host is reachable but not accepting connections on that port.

While less dangerous than open ports, they:
- Confirm host availability
- Reveal firewall behaviour
- Provide reconnaissance data

---

## Setting a Source Port in nmap

Example:

```bash
nmap --source-port 53 <target>
```

This sets the source port of outgoing packets.

Why use it?
- Attempt firewall evasion
- Mimic trusted traffic (e.g., DNS traffic)
- Test filtering rules

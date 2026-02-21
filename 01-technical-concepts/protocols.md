# Protocols – Technical Interview Questions

---

## 1. Explain the Address Resolution Protocol (ARP)

Address Resolution Protocol (ARP) is used to map IP addresses to MAC addresses within a local network.

When a device wants to communicate with another device on the same subnet, it:

1. Broadcasts an ARP request asking “Who has this IP?”
2. The device with that IP responds with its MAC address.
3. The sender stores the mapping in its ARP cache.

ARP operates at Layer 2 (Data Link Layer) of the OSI model.

Security weakness:
ARP has no authentication mechanism, making it vulnerable to ARP poisoning attacks, which can lead to Man-in-the-Middle (MITM) attacks.

---

## 2. Explain the Internet Control Message Protocol (ICMP)

ICMP is used for network diagnostics and error reporting.

It operates at the Network layer and supports functions such as:

- Echo Request / Echo Reply (used by `ping`)
- Destination unreachable messages
- Time Exceeded (used in `traceroute`)

ICMP does not carry application data; it reports network-level conditions.

Security concerns:
- ICMP can be abused for network reconnaissance.
- ICMP tunnelling can be used for covert communication.
- ICMP flooding can be used in denial-of-service attacks.

---

## 3. Explain the User Datagram Protocol (UDP)

UDP is a connectionless transport protocol.

Characteristics:
- No handshake
- No guaranteed delivery
- No sequencing
- No congestion control

It is faster and has lower overhead than TCP.

Common uses:
- DNS
- VoIP
- Streaming
- Online gaming

Security considerations:
Because it is connectionless, it can be used in amplification attacks (e.g., DNS amplification).

---

## 4. What are the HTTP response codes?

HTTP response codes indicate the result of a request.

Categories:

- 1xx – Informational
- 2xx – Success (e.g., 200 OK)
- 3xx – Redirection (e.g., 301 Moved Permanently)
- 4xx – Client errors (e.g., 403 Forbidden, 404 Not Found)
- 5xx – Server errors (e.g., 500 Internal Server Error)

From a security perspective:
- 403 may indicate protected resources.
- 500 errors may leak internal information.
- 401 responses reveal authentication mechanisms.

---

## 5. Explain IPv6 and why it is not widely adopted

IPv6 was developed to address IPv4 exhaustion.

Key features:
- 128-bit address space
- Simplified header structure
- Built-in IPSec support
- No need for NAT

Reasons for slower adoption:
- Infrastructure upgrade costs
- Compatibility concerns
- Continued IPv4 usability through NAT
- Operational inertia

Security note:
IPv6 introduces new attack surfaces (e.g., rogue router advertisements).

---

## 6. What are the two IPSec modes?

IPSec operates in:

1. Transport Mode
   - Encrypts only the payload
   - Original IP header remains intact
   - Used for end-to-end communication

2. Tunnel Mode
   - Encrypts the entire original packet
   - Adds a new IP header
   - Commonly used in VPNs

Tunnel mode provides better protection for network-to-network communication.

---

## 7. What is the IS-IS protocol?

Intermediate System to Intermediate System (IS-IS) is a link-state routing protocol.

It is used primarily in large enterprise and service provider networks.

Characteristics:
- Fast convergence
- Scalable
- Supports IPv4 and IPv6

IS-IS operates directly over Layer 2, unlike OSPF which runs over IP.

---

## 8. Explain the Network File System (NFS) protocol

NFS allows file sharing between systems over a network.

It enables clients to mount remote directories as if they were local.

Common ports:
- 2049 (NFS)
- RPC-related ports

Security issues:
- Misconfigured exports
- No encryption in older versions
- Root squashing misconfigurations

---

## 9. Explain the Internet Key Exchange (IKE) Protocol

IKE is used to negotiate security associations in IPSec.

Functions:
- Authenticates peers
- Negotiates encryption algorithms
- Establishes shared keys

Versions:
- IKEv1
- IKEv2 (more secure and efficient)

IKE enables secure VPN setup.

---

## 10. What protocols are associated with VPN?

Common VPN protocols include:

- IPSec
- IKEv2
- OpenVPN
- L2TP
- PPTP (legacy and insecure)
- WireGuard

Modern secure implementations use:
- IPSec with IKEv2
- OpenVPN
- WireGuard

---

## 11. What protocols are associated with VoIP?

VoIP commonly uses:

- SIP (Session Initiation Protocol)
- RTP (Real-time Transport Protocol)
- RTCP
- H.323

Security concerns:
- Eavesdropping
- SIP registration hijacking
- DoS attacks

Encryption can be provided using SRTP and TLS.

---

## 12. Name and explain a Kerberos exploit

One example is Kerberoasting.

In Kerberoasting:
- An attacker requests service tickets for service accounts.
- The ticket is encrypted with the service account’s hash.
- The attacker extracts the ticket and attempts offline cracking.

If successful, the attacker gains access to privileged accounts.

---

## 13. What is a Pre-Shared Key (PSK) used for and what are the issues?

A PSK is a shared secret used for authentication, commonly in:

- WPA/WPA2 Wi-Fi
- VPN tunnels

Issues:
- If compromised, all users are affected.
- Weak PSKs can be brute-forced.
- Difficult to rotate in large environments.

PSKs do not scale well in enterprise settings.

---

## 14. How many versions of NFS are there and what is compatibility?

NFS versions include:

- NFSv2
- NFSv3
- NFSv4 (including 4.1 and 4.2)

Compatibility:
- Later versions support backward compatibility in many implementations.
- NFSv4 introduced:
  - Stateful protocol
  - Stronger authentication
  - Improved locking
  - Better security features

NFSv4 is more secure and efficient than earlier versions.

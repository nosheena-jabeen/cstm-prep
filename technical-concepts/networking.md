# Networking – Technical Interview Questions

---

## 1. Explain the Open Systems Interconnection (OSI) Model

The OSI model is a seven-layer conceptual framework used to describe how data travels between systems across a network. It does not represent a specific implementation, but rather a way of understanding how different networking functions are logically separated.

The seven layers are:

1. **Physical** – Handles raw transmission of bits across a medium (cables, radio waves, voltages).
2. **Data Link** – Responsible for MAC addressing and switching. Ensures reliable transmission between directly connected devices.
3. **Network** – Provides logical addressing and routing using IP.
4. **Transport** – Ensures end-to-end communication using protocols like TCP and UDP.
5. **Session** – Manages sessions between applications.
6. **Presentation** – Handles encryption, encoding, and formatting.
7. **Application** – Interfaces directly with end-user services (HTTP, FTP, DNS).

The OSI model is particularly useful for troubleshooting. For example, if you cannot ping a host, the issue may exist at Layer 1 (cable unplugged), Layer 2 (VLAN misconfiguration), or Layer 3 (routing issue). From a security perspective, many attacks map directly to OSI layers, which helps in understanding where defensive controls should be applied.

---

## 2. Explain the Department of Defense (DoD) Networking Model

The DoD model, also known as the TCP/IP model, is a simplified four-layer model used to describe real-world networking implementation.

The layers are:

1. Network Access
2. Internet
3. Transport
4. Application

Unlike the OSI model, the TCP/IP model reflects how modern networks actually operate. For example, the Internet layer corresponds to IP routing, while the Transport layer includes TCP and UDP.

The OSI model is more theoretical and educational, whereas the TCP/IP model is more practical and directly tied to protocol stacks implemented in operating systems.

---

## 3. Explain how DHCP works

Dynamic Host Configuration Protocol (DHCP) automatically assigns IP configuration settings to devices joining a network.

It follows a four-step process known as DORA:

1. **Discover** – The client broadcasts a request for an IP address.
2. **Offer** – The DHCP server responds with an available IP address.
3. **Request** – The client requests the offered address.
4. **Acknowledge** – The server confirms and leases the address.

DHCP assigns more than just an IP address. It also provides:
- Subnet mask
- Default gateway
- DNS servers
- Lease duration

From a security perspective, DHCP can be abused through rogue DHCP servers, which may redirect traffic to malicious gateways, enabling Man-in-the-Middle attacks.

---

## 4. What is a Start of Authority (SOA) record in DNS?

The SOA record defines authoritative information about a DNS zone. It specifies which server is the primary authority for the domain and includes administrative details such as:

- Primary name server
- Responsible party email
- Serial number
- Refresh, retry, and expire timers

The serial number is especially important because secondary DNS servers use it to determine whether zone data needs to be updated.

The SOA record is critical for DNS replication and zone management.

---

## 5. Explain what DNS is and how DNS works

The Domain Name System (DNS) translates human-readable domain names into IP addresses.

When a user enters a domain name:

1. The local system checks its cache.
2. If not found, it queries a recursive resolver.
3. The resolver contacts a root server.
4. The root server directs it to the appropriate Top-Level Domain (TLD) server.
5. The TLD server points to the authoritative name server.
6. The authoritative server returns the IP address.

DNS primarily uses UDP on port 53, though TCP is used for large responses and zone transfers.

Security implications include DNS spoofing, cache poisoning, and zone transfer misconfigurations.

---

## 6. Explain the IP protocol and IP addresses

The Internet Protocol (IP) provides logical addressing and routing between networks.

IP is connectionless and does not guarantee delivery. Reliability is handled by higher-layer protocols such as TCP.

IPv4 uses 32-bit addresses (e.g., 192.168.1.10), while IPv6 uses 128-bit addresses to support a vastly larger address space.

Each IP packet contains:
- Source address
- Destination address
- TTL (Time To Live)
- Protocol identifier

IP is foundational to routing and inter-network communication.

---

## 7. Explain Classless Inter-Domain Routing (CIDR)

CIDR replaces the older class-based IP addressing system.

Instead of rigid Class A, B, and C ranges, CIDR uses prefix notation:

192.168.1.0/24

The `/24` indicates that 24 bits are used for the network portion of the address.

CIDR allows flexible subnetting and efficient IP allocation, reducing waste and improving routing aggregation.

---

## 8. Explain how MITM works with ARP

Address Resolution Protocol (ARP) maps IP addresses to MAC addresses within a local network.

In ARP poisoning:

- The attacker sends forged ARP replies.
- Victims update their ARP tables with the attacker’s MAC address.
- Traffic intended for the gateway is sent to the attacker instead.

This allows interception, modification, or redirection of traffic.

ARP has no authentication mechanism, which makes it vulnerable by design.

---

## 9. What is the function of Port Address Translation (PAT)?

PAT allows multiple internal devices to share one public IP address.

It works by translating private IP addresses into a single public IP, differentiating sessions using unique source port numbers.

This is commonly referred to as NAT overload.

PAT conserves public IP space and hides internal network structure.

---

## 10. What is LDAP?

Lightweight Directory Access Protocol (LDAP) is used to query and manage directory services.

It is commonly used in environments such as Active Directory for authentication and user management.

LDAP runs on port 389 (unencrypted) and 636 (LDAPS).

Security concerns include LDAP injection and credential exposure if encryption is not used.

---

## 11. Name and explain two distance vector routing protocols

Two examples are:

- RIP (Routing Information Protocol)
- EIGRP (Enhanced Interior Gateway Routing Protocol)

Distance vector protocols determine the best path based on distance metrics such as hop count. They periodically share routing tables with neighbouring routers.

They are simpler than link-state protocols but converge more slowly.

---

## 12. Explain MAC addresses and OUI

A MAC address is a 48-bit hardware identifier assigned to network interfaces.

The first 24 bits represent the Organizationally Unique Identifier (OUI), which identifies the manufacturer.

MAC addresses operate at Layer 2 and are used for communication within local networks.

MAC spoofing can be used to bypass certain network restrictions.

---

## 13. What is firewalking?

Firewalking is a reconnaissance technique used to determine which ports are permitted through a firewall.

It works by manipulating packet TTL values and analysing ICMP responses to infer filtering rules.

It allows attackers to map firewall policies without direct access.

---

## 14. Explain WPA

Wi-Fi Protected Access (WPA) is a wireless security protocol used to protect Wi-Fi networks.

WPA2 uses AES encryption.
WPA3 improves resistance to brute-force attacks and enhances authentication security.

WPA secures data confidentiality and integrity in wireless communications.

---

## 15. Explain IPSec

IPSec secures IP communication through encryption and authentication.

It operates in:

- Transport mode (encrypts payload only)
- Tunnel mode (encrypts entire packet)

IPSec is commonly used in VPNs to provide secure site-to-site or remote access connectivity.

---

## 16. Where is the hosts file and what is its purpose?

The hosts file maps domain names to IP addresses locally before DNS is queried.

Windows:
C:\Windows\System32\drivers\etc\hosts

Linux:
 /etc/hosts

It is used for local name resolution and testing but can also be abused for redirection attacks.

---

## 17. Explain how VLANs work

Virtual Local Area Networks (VLANs) logically segment networks into separate broadcast domains.

Devices in different VLANs cannot communicate without routing.

VLANs improve security, performance, and traffic control.

Improper configuration may allow VLAN hopping attacks.

---

## 18. Explain TACACS

TACACS+ provides centralised authentication, authorisation, and accounting (AAA) for network devices.

It is commonly used for router and switch administration.

It operates over TCP port 49.

---

## 19. Why are closed ports a cause for concern?

Closed ports indicate that a host is reachable but not accepting connections on that port.

While not as dangerous as open ports, they:

- Confirm host availability
- Provide reconnaissance information
- May become open if services are enabled later

They reveal network presence and behaviour.

---

## 20. How would you set a source port in nmap and why?

You can set a source port using:

```bash
nmap --source-port 53 <target>
```

This may help bypass poorly configured firewalls that trust specific source ports (such as DNS traffic on port 53).

It is commonly used in firewall evasion testing.

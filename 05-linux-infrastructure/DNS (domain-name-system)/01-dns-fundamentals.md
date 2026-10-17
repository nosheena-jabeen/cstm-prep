# DNS Fundamentals for Penetration Testing

## Purpose

This page contains my working notes on the Domain Name System (DNS) from a penetration-testing perspective.

The aim is to understand not only the commands used during DNS enumeration, but what information is being requested, where that information comes from, and how DNS enumeration can contribute to wider network reconnaissance.

The practical application of these concepts is documented separately:

- [Building a Vulnerable DNS Server](02-building-vulnerable-dns-server.md)
- [DNS Zone Transfer Assessment](03-dns-zone-transfer-assessment.md)

---

# Protocol Overview

| Property | Details |
|---|---|
| Protocol | Domain Name System (DNS) |
| Default Port | 53 |
| Transport | UDP and TCP |
| UDP | Commonly used for standard DNS queries and responses |
| TCP | Used where required, including traditional full zone transfers (AXFR) |
| Primary Purpose | Mapping names to information such as IP addresses |

Traditional DNS queries do not inherently provide confidentiality, and poorly configured DNS infrastructure can disclose useful information about systems and services within an environment.

From a penetration-testing perspective, DNS can therefore be an important source of reconnaissance information.

---

# 1. What DNS Does

The Domain Name System translates human-readable names into information computers can use to communicate.

For example:

```text
www.example.com
        |
        | DNS
        v
192.0.2.10
```

Without DNS, users would often need to remember IP addresses rather than names such as:

```text
www.example.com
```

DNS is therefore often described as a distributed naming system for networks and the Internet.

However, DNS contains considerably more information than simple hostname-to-IP mappings.

It can also identify:

- authoritative nameservers;
- mail servers;
- aliases;
- IPv6 addresses;
- reverse mappings;
- service-related information; and
- administrative information about DNS zones.

This makes DNS useful during reconnaissance.

---

# 2. Why DNS Matters in Penetration Testing

DNS can provide information about the structure of a target environment before any attempt is made to interact directly with the systems identified.

DNS enumeration may reveal:

- hostnames;
- IP addresses;
- subdomains;
- authoritative nameservers;
- mail infrastructure;
- development systems;
- VPN infrastructure;
- administrative systems;
- naming conventions; and
- additional targets for subsequent enumeration.

For example, discovering:

```text
192.168.50.50
```

provides an IP address.

Discovering:

```text
admin.example.lab -> 192.168.50.50
```

provides both the address and contextual information about the possible purpose of the host.

DNS enumeration can therefore contribute to attack-surface mapping.

---

# 3. DNS Servers, Nameservers and Zones

These concepts are related but are not interchangeable.

## DNS Server

A DNS server is a system running software capable of answering DNS requests.

One example of DNS server software is BIND9.

A host running BIND may listen for DNS traffic on:

```text
53/udp
53/tcp
```

Finding port 53 open indicates that a DNS service may be present.

It does **not** automatically establish which domains or zones that server is responsible for.

---

## DNS Zone

A DNS zone is an administrative portion of the DNS namespace containing DNS records.

For example:

```text
northgreen.lab
```

may have a zone containing records such as:

```text
www.northgreen.lab
mail.northgreen.lab
vpn.northgreen.lab
admin.northgreen.lab
```

The zone is hosted by an authoritative nameserver.

---

## Nameserver

An authoritative nameserver holds authoritative DNS information for a zone.

The relationship may look like:

```text
northgreen.lab
       |
       | NS record
       v
cyberscheme.northgreen.lab
```

The hostname of the nameserver can then itself resolve to an IP address:

```text
cyberscheme.northgreen.lab
       |
       | A record
       v
192.168.50.20
```

Therefore:

```text
Zone
  |
  | NS
  v
Nameserver hostname
  |
  | A
  v
IP address
```

This distinction is important during DNS enumeration.

---

# 4. How DNS Resolution Works

When a client needs to resolve a domain name, it will normally first use its configured DNS resolver.

At a simplified level, if the resolver does not already have the required information cached, DNS resolution can involve:

```text
Client
   |
   v
Recursive Resolver
   |
   v
Root DNS Servers
   |
   v
Top-Level Domain Servers
   |
   v
Authoritative Nameserver
   |
   v
DNS Record
```

For example, when resolving a public domain:

1. The client asks its configured resolver.
2. The resolver checks whether the answer is already cached.
3. If necessary, the resolver follows the DNS hierarchy.
4. Root servers direct it towards the appropriate top-level domain.
5. The top-level domain servers identify the authoritative nameserver.
6. The authoritative nameserver provides the requested DNS information.
7. The resolver returns the result to the client.

In a private lab environment, this process may be much simpler because queries can be sent directly to the DNS server hosting the lab zone.

For example:

```bash
dig A host.example.lab @192.168.50.20
```

The `@` syntax tells `dig` which DNS server should receive the query.

---

# 5. Forward and Reverse DNS Lookups

## Forward Lookup

A forward lookup starts with a hostname and attempts to obtain information associated with that name.

For example:

```bash
dig example.com A
```

An A record query requests an IPv4 address.

Conceptually:

```text
example.com
     |
     | A
     v
IPv4 address
```

Other tools can also perform forward lookups:

```bash
nslookup example.com
```

```bash
host example.com
```

---

## Reverse Lookup

A reverse lookup begins with an IP address and attempts to identify an associated hostname.

This normally uses a PTR record.

For example:

```bash
dig -x 8.8.8.8
```

or:

```bash
nslookup 8.8.8.8
```

Conceptually:

```text
IP address
    |
    | PTR
    v
Hostname
```

Not every IP address has a PTR record, and the absence of one does not mean that the host does not exist.

---

# 6. Common DNS Record Types

DNS records describe different types of information.

| Record | Purpose |
|---|---|
| A | Maps a hostname to an IPv4 address |
| AAAA | Maps a hostname to an IPv6 address |
| CNAME | Creates an alias pointing one name to another |
| MX | Identifies mail servers for a domain |
| NS | Identifies nameservers for a DNS zone |
| PTR | Maps an IP address back to a hostname for reverse DNS |
| SOA | Contains administrative information about a DNS zone |
| TXT | Stores arbitrary text used for a variety of purposes |

Two particularly important records during basic DNS enumeration are **NS** and **A**.

An NS record might reveal:

```text
example.lab.    IN    NS    ns1.example.lab.
```

This means that:

```text
ns1.example.lab
```

is a nameserver for the `example.lab` zone.

It does **not** mean that `ns1.example.lab` is an IP address.

The corresponding A record might then reveal:

```text
ns1.example.lab.    IN    A    192.168.50.20
```

The full relationship is therefore:

```text
example.lab
     |
     | NS
     v
ns1.example.lab
     |
     | A
     v
192.168.50.20
```

---

# 7. TXT Records, SPF and DKIM

TXT records can contain text used for several purposes, including domain verification and email-security mechanisms.

For example, SPF policies are commonly published using TXT records.

DKIM public-key information is also commonly published in DNS using TXT records beneath selector-specific names.

SPF and DKIM should therefore not be thought of simply as equivalent record types alongside A, AAAA or MX. They are technologies that make use of information published through DNS.

---

# 8. Identifying DNS Services

During network enumeration, a tester may first need to determine whether a target is providing DNS.

Because DNS can use both TCP and UDP port 53, both should be considered.

For example:

```bash
nmap -sTU -p53 <target>
```

The relevant Nmap options are:

```text
-sT    TCP connect scan
-sU    UDP scan
-p53   scan port 53
```

A result may resemble:

```text
53/tcp    open    domain
53/udp    open    domain
```

This establishes that a DNS service appears to be present.

It does **not** establish that the server is authoritative for a particular domain.

---

# 9. Querying Nameserver Records

If a domain or zone is known, its NS records can be queried.

Using `dig`:

```bash
dig NS example.com
```

To send the query to a specific DNS server:

```bash
dig NS example.lab @192.168.50.20
```

Other tools can also request NS records:

```bash
host -t NS example.com
```

The objective is to identify the nameserver or nameservers associated with the zone.

For example:

```text
example.lab.    IN    NS    ns1.example.lab.
```

This provides a hostname.

An A-record query can then be used to determine its IPv4 address:

```bash
dig A ns1.example.lab @192.168.50.20
```

---

# 10. Subdomain Enumeration

DNS can also be used to identify additional hostnames.

If likely names are known or suspected, they can be tested individually.

For example:

```text
www
mail
ftp
admin
vpn
```

A simple wordlist can be created:

```bash
printf "www\nmail\nftp\nadmin\nvpn\n" > subdomains.txt
```

The entries can then be queried:

```bash
while read -r sub; do
    host "$sub.example.com"
done < subdomains.txt
```

Possible results include:

```text
www.example.com -> resolves
mail.example.com -> resolves
admin.example.com -> NXDOMAIN
```

`NXDOMAIN` indicates that DNS reports that the queried name does not exist.

Subdomain enumeration can therefore identify additional hosts that may warrant further investigation.

---

# 11. DNS Zone Transfers

DNS servers sometimes need to replicate zone information between authorised servers.

A full DNS zone transfer uses:

```text
AXFR
```

For example, an organisation might have:

```text
Primary DNS Server
       |
       | AXFR
       v
Secondary DNS Server
```

The secondary server can receive zone information from the primary.

Zone transfers are therefore **legitimate DNS functionality**.

The security issue occurs when a DNS server allows clients that should not be authorised to retrieve the zone.

---

# 12. Testing for an AXFR Zone Transfer

After identifying a nameserver, a tester can determine whether it permits a full zone transfer.

Using `dig`:

```bash
dig AXFR example.lab @<nameserver-IP>
```

For example:

```bash
dig AXFR northgreen.lab @192.168.50.20
```

If the transfer is appropriately restricted, the request should not provide the zone to an unauthorised client.

If it succeeds, the response may expose records such as:

```text
www.example.lab
mail.example.lab
admin.example.lab
vpn.example.lab
dev.example.lab
files.example.lab
```

and their associated IP addresses.

---

# 13. Why a Successful Zone Transfer Matters

A successful AXFR request does not automatically compromise the hosts contained within the zone.

Instead, it can disclose information useful for further reconnaissance.

For example, before DNS enumeration a tester may know only:

```text
192.168.50.20
```

After a successful zone transfer, the tester may discover:

```text
www.example.lab      -> 192.168.50.30
mail.example.lab     -> 192.168.50.40
admin.example.lab    -> 192.168.50.50
vpn.example.lab      -> 192.168.50.60
```

Those addresses can then become inputs for further host and service enumeration.

The workflow becomes:

```text
Discover DNS
      |
      v
Identify nameserver
      |
      v
Attempt AXFR
      |
      v
Zone information disclosed
      |
      v
Extract hostnames/IPs
      |
      v
Verify hosts
      |
      v
Enumerate services
```

DNS enumeration therefore feeds into the wider penetration-testing methodology.

---

# 14. DNS Information vs Network Discovery

DNS enumeration and network host discovery may reveal some of the same systems, but they approach the problem differently.

A network scan might identify:

```text
192.168.50.50
```

DNS enumeration might reveal:

```text
admin.example.lab -> 192.168.50.50
```

The first result identifies an address.

The second provides additional context about how the system is named and possibly what role it performs.

However, a DNS record does not guarantee that the associated system is currently online.

Records can become stale, systems can be offline, and addresses can change.

DNS findings should therefore be verified through subsequent enumeration.

---

# 15. Automated DNS Enumeration with dnsenum

`dnsenum` automates several DNS-enumeration tasks.

A basic invocation is:

```bash
dnsenum example.com
```

For a private lab zone where the DNS server must be specified explicitly:

```bash
dnsenum --dnsserver 192.168.50.20 northgreen.lab
```

Automated enumeration can be useful for identifying information quickly, but it should not replace understanding the underlying DNS queries.

Manual commands such as:

```bash
dig
host
nslookup
```

make it easier to understand exactly what information is being requested and which server is being queried.

A useful approach is therefore:

```text
Manual enumeration
       |
       v
Understand the DNS relationships
       |
       v
Automated enumeration
       |
       v
Compare and verify results
```

---

# 16. Common DNS Tools

## dig

Useful for making specific DNS queries and viewing detailed responses.

Examples:

```bash
dig A example.com
```

```bash
dig NS example.com
```

```bash
dig -x 8.8.8.8
```

```bash
dig AXFR example.lab @192.168.50.20
```

---

## host

Provides a straightforward interface for DNS lookups.

Examples:

```bash
host example.com
```

```bash
host -t NS example.com
```

```bash
host -t A example.com
```

---

## nslookup

Can perform forward, reverse and server-specific DNS queries.

Examples:

```bash
nslookup example.com
```

```bash
nslookup 8.8.8.8
```

---

## dnsenum

Automates multiple DNS-enumeration techniques.

Example:

```bash
dnsenum example.com
```

---

## Nmap

Nmap is not specifically a DNS-enumeration tool, but it can identify systems exposing DNS services.

Example:

```bash
nmap -sTU -p53 <target>
```

---

# 17. Practical DNS Enumeration Methodology

Rather than memorising unrelated commands, I use the following sequence as a basic DNS-enumeration workflow.

## Step 1 — Identify DNS

Determine whether the target exposes DNS:

```bash
nmap -sTU -p53 <target>
```

Question:

> Is a DNS service available over TCP and/or UDP port 53?

---

## Step 2 — Identify the Nameserver

Query NS records:

```bash
dig NS <domain> @<DNS-server>
```

Question:

> Which nameserver is responsible for the zone?

---

## Step 3 — Resolve the Nameserver

Query its A record:

```bash
dig A <nameserver> @<DNS-server>
```

Question:

> Which IPv4 address belongs to that nameserver?

---

## Step 4 — Enumerate DNS Records

Perform targeted lookups and investigate possible hostnames.

Question:

> What additional infrastructure can DNS reveal?

---

## Step 5 — Test Zone Transfer

Attempt an AXFR request against the authorised assessment target:

```bash
dig AXFR <domain> @<nameserver-IP>
```

Question:

> Will the server disclose the complete zone to this client?

---

## Step 6 — Process the Results

If additional hosts are discovered, record their:

- hostname;
- IP address;
- apparent purpose; and
- source of discovery.

Question:

> What new targets has DNS enumeration identified?

---

## Step 7 — Continue Network Enumeration

Verify the newly discovered systems and determine what services they expose.

DNS is therefore not the end of the enumeration process.

It is one source of information that can lead into the next stage of the assessment.

---

# 18. Key Security Takeaways

The main lessons I take from DNS enumeration are:

- DNS commonly uses both TCP and UDP port 53.
- Discovering port 53 identifies a DNS service, not automatically an authoritative nameserver for a particular zone.
- NS records identify nameservers.
- A records map hostnames to IPv4 addresses.
- AAAA records map hostnames to IPv6 addresses.
- PTR records support reverse DNS lookups.
- DNS zones contain records for part of the DNS namespace.
- AXFR performs a full DNS zone transfer.
- Zone transfers are legitimate functionality and are not inherently vulnerabilities.
- Zone transfers should be restricted to authorised systems.
- A successful unauthorised AXFR can disclose useful infrastructure information.
- DNS records should not automatically be treated as proof that a host is currently active.
- DNS discoveries can feed directly into subsequent network and service enumeration.
- Automated tools are useful, but manual enumeration makes troubleshooting and interpretation easier.

---

# Summary

DNS is more than a mechanism for converting domain names into IP addresses.

From a penetration-testing perspective, it can provide information about nameservers, hosts, addresses, mail infrastructure and the structure of an environment.

The most useful mental model I developed from practising DNS enumeration is:

```text
DNS service
     |
     v
Zone / domain
     |
     v
Nameserver
     |
     v
DNS records
     |
     v
Additional hosts
     |
     v
Further enumeration
```

Understanding these relationships makes commands such as `dig`, `host`, `dnsenum` and AXFR testing part of a logical enumeration process rather than isolated commands to memorise.

The accompanying practical labs demonstrate these concepts by first building an intentionally vulnerable BIND9 DNS server and then assessing it from Kali Linux.

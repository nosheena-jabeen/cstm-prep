# DNS Zone Transfer Assessment

## Overview

Following the creation of an intentionally vulnerable DNS server in my home penetration-testing lab, I assessed the server from Kali Linux without relying on the server-side configuration.

The objective was to practise a structured DNS enumeration methodology and determine whether the target DNS service permitted an unauthorised zone transfer.

The assessment demonstrated how an unrestricted AXFR request can disclose DNS records containing hostnames and IP addresses that can then be used to expand the scope of further enumeration.

> **Scope:** All testing documented here was performed against systems in my isolated home lab. The DNS server was deliberately configured to be vulnerable for training purposes.

---

## Lab Scope

The assessment was performed from the following attacker machine:

| System | Role | IP Address |
|---|---|---|
| Kali Linux | Attacker | `192.168.50.10` |

The initial target was:

```text
192.168.50.20
```

At the beginning of the assessment, the objective was to treat the target from the attacker's perspective rather than relying on knowledge of its server-side configuration.

---

# 1. DNS Service Discovery

I first wanted to establish whether the target was exposing a DNS service.

DNS commonly uses port 53 over both TCP and UDP, so I checked both protocols:

```bash
sudo nmap -sTU -p53 192.168.50.20
```

The scan identified port 53 as accessible over TCP and UDP.

This established that a DNS service was present on the target.

At this stage, however, I had only identified the **service**.

Finding an open DNS service does not by itself establish:

- which DNS zones the server is responsible for;
- whether it is authoritative for the target domain;
- what hostnames exist within the zone; or
- whether zone transfers are permitted.

Further DNS enumeration was therefore required.

---

# 2. Identifying the Nameserver

The practice domain used within the lab was:

```text
northgreen.lab
```

I queried the target DNS server for the domain's NS record:

```bash
dig NS northgreen.lab @192.168.50.20
```

The response contained:

```text
northgreen.lab.    IN    NS    cyberscheme.northgreen.lab.
```

This identified:

```text
cyberscheme.northgreen.lab
```

as a nameserver for the `northgreen.lab` zone.

The relationship can be represented as:

```text
northgreen.lab
      |
      | NS
      v
cyberscheme.northgreen.lab
```

This was an important distinction during the assessment.

`northgreen.lab` is the DNS zone being queried, whereas `cyberscheme.northgreen.lab` is the hostname of the nameserver identified by its NS record.

---

# 3. Resolving the Nameserver

Once the nameserver hostname had been identified, the next question was which IP address belonged to it.

An A record maps a hostname to an IPv4 address.

This can be queried manually using:

```bash
dig A cyberscheme.northgreen.lab @192.168.50.20
```

The resulting relationship was:

```text
northgreen.lab
      |
      | NS
      v
cyberscheme.northgreen.lab
      |
      | A
      v
192.168.50.20
```

In my lab, BIND also returned this A record in the additional section of the earlier NS query.

This meant that the authoritative DNS information ultimately led back to the server already being assessed at:

```text
192.168.50.20
```

---

# 4. Testing for a Zone Transfer

With the DNS service and nameserver identified, I tested whether the server permitted a full DNS zone transfer.

A full zone transfer uses the DNS request type:

```text
AXFR
```

I sent the request directly to the DNS server:

```bash
dig AXFR northgreen.lab @192.168.50.20
```

The request succeeded.

This demonstrated that the server allowed my Kali machine to retrieve the contents of the `northgreen.lab` zone despite the attacker machine having no legitimate requirement to perform a zone transfer.

---

# 5. Information Disclosed

The successful AXFR request disclosed multiple DNS records, including hosts such as:

```text
www.northgreen.lab
mail.northgreen.lab
admin.northgreen.lab
vpn.northgreen.lab
dev.northgreen.lab
files.northgreen.lab
```

These records also disclosed their corresponding IPv4 addresses:

```text
www.northgreen.lab      -> 192.168.50.30
mail.northgreen.lab     -> 192.168.50.40
admin.northgreen.lab    -> 192.168.50.50
vpn.northgreen.lab      -> 192.168.50.60
dev.northgreen.lab      -> 192.168.50.70
files.northgreen.lab    -> 192.168.50.80
```

This demonstrated why unrestricted zone transfers can be valuable during reconnaissance.

Before the AXFR request, my known target was:

```text
192.168.50.20
```

After the transfer, I had obtained several additional hostnames and IP addresses that could potentially represent further systems within the environment.

The DNS server had effectively disclosed information that could be used to construct a map of additional infrastructure.

---

# 6. Why the Disclosure Matters

The successful zone transfer did not itself prove that the newly discovered systems were vulnerable.

It also did not prove that every DNS record represented a currently active host. DNS information may be stale, systems may be offline, or addresses may no longer be in use.

Instead, the AXFR results provided **new enumeration targets**.

For example:

```text
admin.northgreen.lab -> 192.168.50.50
```

provides considerably more context than discovering only:

```text
192.168.50.50
```

through network scanning.

The hostname may suggest the intended function of a system and can help prioritise and inform subsequent enumeration.

The next stage of an assessment would therefore be to verify the discovered hosts and identify the services they expose.

---

# 7. Feeding DNS Results into Network Enumeration

The IP addresses disclosed through AXFR can be organised into a target list:

```text
192.168.50.30
192.168.50.40
192.168.50.50
192.168.50.60
192.168.50.70
192.168.50.80
```

For example, these could be saved as:

```text
axfr-targets.txt
```

Nmap can then read targets from a file using the `-iL` option:

```bash
nmap -iL axfr-targets.txt
```

The purpose of this next stage would be to establish which of the DNS-discovered hosts are actually reachable and then enumerate the services exposed by those systems.

The methodology therefore becomes:

```text
Discover DNS service
        |
        v
Identify nameserver
        |
        v
Resolve nameserver
        |
        v
Test AXFR
        |
        v
Zone transfer succeeds
        |
        v
Extract hostnames and IP addresses
        |
        v
Verify discovered hosts
        |
        v
Enumerate exposed services
        |
        v
Perform service-specific enumeration
```

This helped me understand DNS enumeration as part of a wider penetration-testing process rather than as an isolated set of DNS commands.

---

# 8. Manual Enumeration vs dnsenum

After completing the DNS enumeration manually, I tested the `dnsenum` tool.

My first attempt was:

```bash
dnsenum northgreen.lab
```

This failed with an NS query/network error.

The reason was the design of the lab.

`northgreen.lab` is a private practice zone that exists only on the DNS server at:

```text
192.168.50.20
```

The Kali machine was also connected to an isolated host-only network without a normal route to an external DNS resolver.

My manual `dig` commands had worked because I explicitly specified which DNS server should receive the query:

```bash
dig NS northgreen.lab @192.168.50.20
```

I therefore supplied the lab DNS server explicitly to `dnsenum`:

```bash
dnsenum --dnsserver 192.168.50.20 northgreen.lab
```

The enumeration then succeeded.

This was a useful troubleshooting exercise because the initial automated-tool failure did not mean that DNS enumeration itself had failed.

I already knew that the DNS service and AXFR worked because I had tested them manually.

Instead, I could compare the two approaches and identify that the automated tool had not initially been directed to the DNS server hosting my private lab zone.

---

# Finding: Unrestricted DNS Zone Transfer

## Description

The authoritative DNS server for `northgreen.lab` permitted an unauthorised client to perform a full AXFR zone transfer.

The attacker machine at `192.168.50.10` was able to request and retrieve zone information without being an authorised secondary DNS server.

## Evidence

The following request successfully retrieved the zone:

```bash
dig AXFR northgreen.lab @192.168.50.20
```

The response disclosed multiple hostnames and corresponding IP addresses within the `northgreen.lab` environment.

## Security Impact

An unrestricted DNS zone transfer can expose information about an organisation's infrastructure to an unauthorised user.

The disclosed information may include:

- server hostnames;
- IP addresses;
- naming conventions;
- systems whose names suggest administrative or development functions;
- mail, VPN, web or file infrastructure; and
- additional targets for subsequent enumeration.

The issue therefore provides an attacker with reconnaissance information that may reduce the effort required to identify potentially interesting systems.

A successful zone transfer does **not** automatically mean those systems are vulnerable. The disclosed information instead assists subsequent discovery and enumeration.

## Remediation

Zone transfers should be restricted to systems that have a legitimate requirement to receive them, such as explicitly authorised secondary DNS servers.

The deliberately vulnerable lab configuration allowed transfers from arbitrary clients:

```text
allow-transfer { any; };
```

A production configuration should instead restrict transfers according to the organisation's DNS architecture and authorised secondary servers.

The DNS configuration should also be reviewed after changes to ensure that temporary troubleshooting or deployment settings have not unintentionally left zone transfers broadly accessible.

---

# Key Lessons

This exercise helped connect several DNS concepts into a single penetration-testing methodology.

I learned that:

- TCP and UDP port 53 can indicate the presence of a DNS service.
- Finding a DNS service does not automatically identify the zones for which it is authoritative.
- NS records identify nameservers for a DNS zone.
- A records map hostnames to IPv4 addresses.
- AXFR requests can be used to request a full DNS zone transfer.
- Zone transfers are legitimate DNS functionality; the vulnerability occurs when access to them is insufficiently restricted.
- A successful zone transfer can expose additional hostnames and IP addresses that become inputs for further enumeration.
- DNS records do not prove that a host is currently online and should be verified.
- Manual enumeration makes it easier to understand what information is being requested and why.
- Automated tools are useful, but their failures need to be interpreted rather than accepted blindly.

Most importantly, I found manual DNS enumeration more useful for understanding the process than immediately relying on automated tooling.

Instead of treating DNS enumeration as a collection of commands, I could follow the information obtained at each stage and use it to decide what to investigate next.

---

# Assessment Summary

The assessment began with a single known target:

```text
192.168.50.20
```

Service enumeration identified DNS on TCP and UDP port 53.

Manual DNS queries then identified the `northgreen.lab` nameserver and its associated IPv4 address. An AXFR request demonstrated that the server permitted an unauthorised full zone transfer.

The resulting zone information disclosed multiple additional hostnames and IP addresses that could be fed into subsequent network and service enumeration.

The exercise demonstrated how a DNS configuration weakness can turn a single known DNS server into a source of information about a wider environment.

---

## Related Lab

For the server-side configuration and explanation of how the vulnerable environment was created, see:

`01-vulnerable-dns-server.md`

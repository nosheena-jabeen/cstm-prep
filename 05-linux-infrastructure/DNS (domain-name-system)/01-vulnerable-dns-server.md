# Building a Vulnerable DNS Server with BIND9

## Overview

As part of my penetration testing home lab, I built an intentionally vulnerable DNS server to practise DNS enumeration and understand DNS zone transfers from both the administrator and attacker perspectives.

Rather than only practising DNS commands against a pre-built target, I wanted to understand what was happening on the server itself: what makes a machine a DNS server, how an authoritative zone is configured, how DNS records relate hostnames to IP addresses, and what configuration allows an unauthorised zone transfer to occur.

The final environment consists of a Kali Linux attacker machine and an Ubuntu Server running BIND9 on an isolated VMware host-only network.

> **Important:** The insecure DNS configuration documented here was created deliberately for training. The vulnerable server is isolated from my normal network and is not exposed to the Internet.

---

## Lab Architecture

The lab was built using VMware Workstation Pro.

I created a dedicated host-only VMware network:

| Component | Configuration |
|---|---|
| Virtual network | VMnet2 |
| Network type | Host-only |
| Subnet | `192.168.50.0/24` |
| DHCP | Disabled |
| Internet access | None |

Two virtual machines were connected to this network:

| Host | Purpose | IP Address |
|---|---|---|
| Kali Linux | Attacker | `192.168.50.10` |
| Ubuntu Server | DNS server | `192.168.50.20` |

The resulting architecture was:

```text
              VMnet2
         192.168.50.0/24
          (Host-only)
               |
       +-------+-------+
       |               |
   Kali Linux      Ubuntu Server
  192.168.50.10    192.168.50.20
   Attacker          BIND9
```

Using a host-only network allows the vulnerable systems to communicate with one another without exposing the deliberately insecure services to my normal network.

---

## Configuring the Kali Attacker

Because DHCP was disabled on VMnet2, Kali did not automatically receive an IPv4 address.

I initially assigned the attacker machine an address manually:

```bash
sudo ip addr add 192.168.50.10/24 dev eth0
```

I verified the configuration using:

```bash
ip addr show eth0
ip route
```

This placed Kali on the `192.168.50.0/24` lab network.

---

## Building the DNS Server

I created a minimal Ubuntu Server virtual machine with:

- 2 virtual processors
- 4 GB RAM
- 20 GB virtual disk
- OpenSSH Server installed

During installation, the VM was temporarily connected to VMware's NAT network so that packages could be downloaded.

Once Ubuntu was installed, I installed BIND9 and the DNS utilities:

```bash
sudo apt update
sudo apt install bind9 bind9-utils dnsutils -y
```

I then confirmed that the BIND service was running:

```bash
systemctl status bind9
```

BIND runs the DNS server process (`named`) and listens for DNS traffic on port 53.

After the required packages were installed, I shut down the VM and moved its network adapter from the NAT network to the isolated VMnet2 network.

---

## Configuring the DNS Server's Address

The Ubuntu network interface was identified as `ens33`.

Initially, I assigned it the following address manually:

```bash
sudo ip addr add 192.168.50.20/24 dev ens33
```

Connectivity between the attacker and server was then tested in both directions.

From Ubuntu:

```bash
ping -c 3 192.168.50.10
```

From Kali:

```bash
ping -c 3 192.168.50.20
```

Both hosts could communicate successfully.

### Making the Address Persistent

Ubuntu was using Netplan, with its configuration stored in:

```text
/etc/netplan/00-installer-config.yaml
```

I changed the interface from DHCP to a static configuration:

```yaml
network:
  ethernets:
    ens33:
      dhcp4: false
      dhcp6: false
      addresses:
        - 192.168.50.20/24
  version: 2
```

I validated and applied the configuration:

```bash
sudo netplan generate
sudo netplan apply
```

After rebooting the server, `192.168.50.20` remained assigned to `ens33`, confirming that the static configuration was persistent.

---

## Naming the DNS Server

I changed the Ubuntu hostname to:

```text
cyberscheme
```

using:

```bash
sudo hostnamectl set-hostname cyberscheme
```

The server could then be identified locally as:

```text
cyberscheme
```

It is important to distinguish the machine's hostname from its role within DNS.

At this stage:

- `cyberscheme` was the hostname of the Ubuntu machine.
- BIND9 was the DNS server software running on that machine.
- The machine had not yet been configured as the authoritative nameserver for my practice DNS zone.

---

## Confirming the DNS Service from Kali

Before creating an authoritative zone, I scanned the Ubuntu server from Kali to confirm that DNS was exposed.

I checked both TCP and UDP port 53:

```bash
sudo nmap -sTU -p53 192.168.50.20
```

The scan identified DNS on both protocols.

This demonstrated an important distinction:

> Discovering an open DNS service does not, by itself, identify which DNS zones that server is authoritative for.

The next step was therefore to configure an authoritative zone.

---

## Creating the `northgreen.lab` Zone

I used the fictional domain:

```text
northgreen.lab
```

I added the following zone declaration to:

```text
/etc/bind/named.conf.local
```

```text
zone "northgreen.lab" {
    type master;
    file "/etc/bind/db.northgreen.lab";
};
```

This tells BIND that it holds the primary copy of the `northgreen.lab` zone and that the zone's records are stored in:

```text
/etc/bind/db.northgreen.lab
```

I validated the BIND configuration with:

```bash
sudo named-checkconf
```

---

## Creating the Zone File

I created:

```text
/etc/bind/db.northgreen.lab
```

with the following initial configuration:

```text
$TTL    604800
@       IN      SOA     cyberscheme.northgreen.lab. admin.northgreen.lab. (
                        2
                        604800
                        86400
                        2419200
                        604800 )

@               IN      NS      cyberscheme.northgreen.lab.
cyberscheme     IN      A       192.168.50.20
```

Two records here were particularly important to understand.

The NS record:

```text
@ IN NS cyberscheme.northgreen.lab.
```

identifies `cyberscheme.northgreen.lab` as a nameserver for the zone.

The A record:

```text
cyberscheme IN A 192.168.50.20
```

maps that hostname to its IPv4 address.

Conceptually:

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

I validated the zone using:

```bash
sudo named-checkzone northgreen.lab /etc/bind/db.northgreen.lab
```

and restarted BIND:

```bash
sudo systemctl restart bind9
```

---

## Verifying the Authoritative Nameserver

From Kali, I queried the DNS server directly:

```bash
dig NS northgreen.lab @192.168.50.20
```

The server returned:

```text
northgreen.lab.    IN    NS    cyberscheme.northgreen.lab.
```

The response also provided the A record for the nameserver:

```text
cyberscheme.northgreen.lab.    IN    A    192.168.50.20
```

At this point the relationship was:

```text
northgreen.lab
        |
        | authoritative NS
        v
cyberscheme.northgreen.lab
        |
        | IPv4 address
        v
192.168.50.20
        |
        | DNS service
        v
TCP/UDP 53
```

---

## Testing the Secure Configuration

Before deliberately introducing the vulnerability, I attempted a full DNS zone transfer from Kali:

```bash
dig AXFR northgreen.lab @192.168.50.20
```

The transfer failed.

This was the expected secure behaviour: an arbitrary client on the network should not automatically be permitted to retrieve the complete contents of the DNS zone.

This also gave me a useful baseline against which to test the vulnerable configuration.

---

## Introducing the Zone Transfer Vulnerability

DNS zone transfers are legitimate functionality. They can be used, for example, to replicate zone information between authorised DNS servers.

The vulnerability occurs when transfers are insufficiently restricted.

For the purposes of this isolated lab, I deliberately modified the `northgreen.lab` configuration:

```text
zone "northgreen.lab" {
    type master;
    file "/etc/bind/db.northgreen.lab";
    allow-transfer { any; };
};
```

The following directive is intentionally insecure:

```text
allow-transfer { any; };
```

It permits arbitrary clients to request a transfer of the zone.

I validated the configuration:

```bash
sudo named-checkconf
```

and restarted BIND:

```bash
sudo systemctl restart bind9
```

---

## Populating the Zone

To demonstrate the information that could be exposed through an unrestricted zone transfer, I added several fictional hosts:

```text
www             IN      A       192.168.50.30
mail            IN      A       192.168.50.40
admin           IN      A       192.168.50.50
vpn             IN      A       192.168.50.60
dev             IN      A       192.168.50.70
files           IN      A       192.168.50.80
```

I also incremented the SOA serial before validating and reloading the updated zone.

```bash
sudo named-checkzone northgreen.lab /etc/bind/db.northgreen.lab
sudo systemctl reload bind9
```

These records simulate the type of infrastructure information that may exist within a DNS zone.

---

## Final Vulnerable Environment

The completed environment now contains an authoritative DNS server with deliberately unrestricted zone transfers:

```text
                     VMnet2
                192.168.50.0/24
                      |
           +----------+----------+
           |                     |
       Kali Linux          cyberscheme
     192.168.50.10        192.168.50.20
       Attacker                |
                               |
                             BIND9
                               |
                         northgreen.lab
                               |
             +-----------------+-----------------+
             |        |        |       |         |
            www      mail    admin    vpn       dev/files
```

The server was now ready to be assessed from Kali.

---

## What I Learned

Building the DNS server before attacking it helped clarify several concepts that I had previously treated as separate commands to memorise.

In particular:

- BIND9 is the software providing the DNS service.
- DNS normally listens on TCP and UDP port 53.
- A DNS server can exist without being authoritative for a particular zone.
- A zone defines DNS information for a namespace such as `northgreen.lab`.
- An NS record identifies a nameserver for that zone.
- An A record can map that nameserver's hostname to an IPv4 address.
- AXFR is a legitimate mechanism for transferring a complete DNS zone.
- The security issue is not the existence of AXFR itself, but allowing unauthorised clients to perform the transfer.
- Configuration validation before restarting services helps identify errors without unnecessarily breaking the running service.

Most importantly, building both the secure and deliberately vulnerable configurations allowed me to observe the difference between a refused and successful zone-transfer request rather than simply memorising that "AXFR is a vulnerability."

---

## Next: Assessing the Vulnerability

The second part of this lab approaches the environment entirely from the attacker's perspective.

Starting from Kali, I enumerate the DNS service, identify the authoritative nameserver, attempt an AXFR zone transfer, extract additional hosts from the disclosed zone, and consider how those results feed into subsequent network enumeration.

**Next:** `02-zone-transfer-assessment.md`

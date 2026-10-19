# Building a Vulnerable SNMP Server

## Overview

As part of my penetration testing home lab, I configured an SNMP service on Ubuntu using its default settings — specifically, leaving the `public` read-only community string in place and making the agent listen on all interfaces rather than only localhost.

Unlike the NFS or DNS labs, there is no "deliberately introduced" misconfiguration here in the sense of changing a security-relevant option from a hardened default. The vulnerability *is* the default configuration: `snmpd` ships with a `public` community string that grants unauthenticated read access to the entire MIB tree, and making the agent reachable on the network rather than only on localhost is the only additional step required to expose it.

The environment reused the same Kali Linux attacker machine and Ubuntu Server (`cyberscheme`) on the isolated VMware host-only network from the earlier DNS and NFS labs.

> **Important:** The insecure SNMP configuration documented here was created deliberately for training. The vulnerable server is isolated from my normal network and is not exposed to the Internet.

---

## Lab Architecture

The same VMnet2 host-only network from previous labs was reused:

| Component        | Configuration     |
| ----------------- | ----------------- |
| Virtual network   | VMnet2            |
| Network type      | Host-only         |
| Subnet            | `192.168.50.0/24` |
| DHCP              | Disabled          |
| Internet access   | None              |

| Host          | Purpose     | IP Address      | Services running        |
| ------------- | ----------- | --------------- | ------------------------ |
| Kali Linux    | Attacker    | `192.168.50.10` | —                        |
| Ubuntu Server | Lab target  | `192.168.50.20` | DNS (53), NFS (2049), SMB (445), SNMP (UDP/161) |

> **No-clash note:** SNMP runs on UDP/161, which does not overlap with any previously configured service. Its configuration file (`/etc/snmp/snmpd.conf`) is entirely separate from `/etc/exports`, `/etc/bind/`, and `/etc/samba/`. All services on `cyberscheme` can run simultaneously without interaction.

---

## Installing SNMP

I installed the SNMP daemon on `cyberscheme`:

```
sudo apt update
sudo apt install snmpd -y
```

Confirming the service started:

```
systemctl status snmpd
```

By default, `snmpd` starts but listens only on `127.0.0.1` — it is not reachable from the network until the agent address is changed.

---

## Configuring the Vulnerable State

I edited `/etc/snmp/snmpd.conf` and made two changes:

**Change 1 — make the agent listen on all interfaces:**

Default:
```
agentaddress  127.0.0.1,[::1]
```

Changed to:
```
agentaddress  0.0.0.0
```

Without this change, the agent is only reachable locally and the service is invisible to any external scanner or query from Kali. Changing this to `0.0.0.0` is the equivalent of a server administrator making SNMP "available on the network" — a common configuration step that is frequently made without considering what the default community string grants to anyone who can reach it.

**Change 2 — confirm the default `public` community string:**

```
rocommunity public default
```

- `rocommunity` — read-only community string; any host presenting this string can query (GET, WALK) but not write
- `public` — the industry-standard default, present on the vast majority of SNMP-enabled devices out of the box
- `default` — permits access from any source address, rather than restricting to a specific subnet

This line was already present in the default config — I verified it was uncommented and in place rather than adding it. The only actual change required to expose the service was the `agentaddress` line above.

I then restarted the service to apply the configuration:

```
sudo systemctl restart snmpd
```

---

## Why the Default Configuration Is the Vulnerability

SNMP's security problem is structural rather than the result of a specific misconfiguration choice made by an administrator. The default `snmpd` installation ships in a state where:

- A well-known, widely-documented community string (`public`) is already configured
- That string grants read access to thousands of data points about the host
- No audit logging of queries is enabled by default
- No rate limiting or source restriction is applied

The only step an administrator typically takes to "enable SNMP" is making it network-accessible — which is exactly what the `agentaddress` change does. The moment that happens, the `public` community string becomes reachable by anyone on the network.

This is distinct from the NFS `no_root_squash` scenario, where an administrator made an explicit, security-relevant choice to disable a protection. Here, the vulnerability exists before any deliberate misconfiguration — the default installation is the vulnerable state.

---

## What I Learned

- `snmpd` ships with a `public` read-only community string already configured; enabling network access is sufficient to expose it.
- The distinction between "service listening on localhost" and "service listening on all interfaces" is the difference between a local management tool and a network-accessible information disclosure source.
- Because SNMP uses UDP, there is no TCP handshake, no connection logging by default, and no session concept — queries arrive, are answered, and leave no trace in standard service logs.
- A single default installation, reachable on the network, can expose OS version, hostname, running processes, active user sessions, network interfaces, and routing tables — all without authentication.

---

## Next: Assessing the Misconfiguration

The next document approaches the service from the attacker's perspective: discovering SNMP, brute-forcing the community string, and enumerating the full scope of information disclosure — including cross-service visibility of DNS, NFS, and Samba services also running on the same host.

**Next:** `03-default-community-string-exploitation.md`

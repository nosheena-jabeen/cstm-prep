# Building a Vulnerable SMB Server

## Overview

As part of my penetration testing home lab, I installed and configured Samba on Ubuntu, first establishing a secure-by-default share requiring valid credentials, then deliberately reconfiguring it to allow anonymous read-write access — one of the most commonly encountered SMB misconfigurations in real-world assessments.

Unlike the SNMP lab where the default installation was itself the vulnerability, SMB required a deliberate configuration change to introduce the insecure state. This makes it a useful comparison: the secure default (`guest ok = no`) and the vulnerable state (`guest ok = yes`, writable) differ by only a few lines in a single config file.

The environment reused the same Kali Linux attacker machine and Ubuntu Server (`cyberscheme`) on the isolated VMware host-only network from earlier labs. A local account was also created to support credential-based exercises that follow.

> **Important:** The insecure SMB configuration documented here was created deliberately for training. The vulnerable server is isolated from my normal network and is not exposed to the Internet.

---

## Lab Architecture

| Component        | Configuration     |
| ----------------- | ----------------- |
| Virtual network   | VMnet2            |
| Network type      | Host-only         |
| Subnet            | `192.168.50.0/24` |
| DHCP              | Disabled          |
| Internet access   | None              |

| Host          | Purpose     | IP Address      | Services running                                    |
| ------------- | ----------- | --------------- | ---------------------------------------------------- |
| Kali Linux    | Attacker    | `192.168.50.10` | —                                                    |
| Ubuntu Server | Lab target  | `192.168.50.20` | DNS (53), NFS (2049), SNMP (UDP/161), SMB (TCP/445)  |

> **No-clash note:** Samba listens on TCP/139 and TCP/445, which do not overlap with any previously configured service. Its configuration (`/etc/samba/smb.conf`) is entirely separate from `/etc/exports`, `/etc/bind/`, and `/etc/snmp/snmpd.conf`. Its share path (`/srv/smb/share`) is distinct from the NFS export path (`/srv/nfs/share`). All services run simultaneously without interaction.

---

## Installing Samba

Samba requires internet access to install. `cyberscheme` was temporarily switched to VMware NAT (VMnet8) for installation, then returned to the isolated VMnet2 lab network before any lab work continued.

On `cyberscheme`:

```
sudo apt update
sudo apt install samba -y
```

Samba installs two key daemons:
- `smbd` — handles file and print sharing; the core SMB service
- `nmbd` — handles NetBIOS name resolution and browsing (legacy but commonly running alongside `smbd`)

Confirming the service started:

```
systemctl status smbd
```

---

## Creating the Share Directory

```
sudo mkdir -p /srv/smb/share
sudo chown nobody:nogroup /srv/smb/share
sudo chmod 755 /srv/smb/share
```

- `/srv/smb/share` — dedicated path for the SMB share, clearly separated from `/srv/nfs/` (no path overlap with the existing NFS export)
- `nobody:nogroup` — no real user account owns the directory, making ownership behaviour clearly visible when testing anonymous access
- `755` — readable and executable by everyone, writable only by the owner; a safe starting point before introducing the misconfiguration

---

## Creating a Lab User Account

A local user account was created to support credential-based exercises (password spraying and attempted command execution). The same credentials were added to both the OS and Samba — Samba maintains its own password database (`smbpasswd`) independently of `/etc/shadow`, so both must be set.

```
sudo useradd -m labuser
sudo passwd labuser
```

Password set to: `password123` (intentionally weak — this is the target credential in the password spray exercise).

Adding the same account to Samba's user database:

```
sudo smbpasswd -a labuser
```

This creates a Samba entry for `labuser` with the same password, allowing NetExec to validate credentials against the SMB service.

---

## Secure-by-Default Configuration

I edited `/etc/samba/smb.conf` and added a share definition at the bottom:

```
[secure_share]
   path = /srv/smb/share
   browseable = yes
   read only = yes
   guest ok = no
```

- `[secure_share]` — the share name visible to clients
- `browseable = yes` — the share appears in share listings
- `read only = yes` — no write access, even for authenticated users
- `guest ok = no` — **the key line**; requires valid credentials, anonymous access refused

I validated the config with `testparm` before restarting:

```
testparm
sudo systemctl restart smbd
```

`testparm` loads and parses `smb.conf` and reports any syntax errors before the service attempts to apply the configuration — the equivalent of `exportfs -ra` or `named-checkconf` in the NFS and DNS labs.

The `testparm` output also revealed a relevant global default already present in the Samba installation:

```
map to guest = Bad User
```

This setting means that if a client presents an invalid username, Samba maps that connection to the guest account rather than rejecting it outright. With `guest ok = no` on the share, the mapped guest still cannot access it — but this global setting becomes significant once `guest ok = yes` is introduced.

---

## Confirming the Secure Baseline from Kali

An anonymous share listing from Kali demonstrated an important nuance:

```
smbclient -L //192.168.50.20 -N
```

Even with `guest ok = no`, Samba returned the full share list:

```
Sharename     Type    Comment
---------     ----    -------
print$        Disk    Printer Drivers
secure_share  Disk
IPC$          IPC     IPC Service (cyberscheme)
```

This is a common Samba behaviour: listing shares and *accessing* share content are separate permission checks. An anonymous client can see that `secure_share` exists even though it cannot access its contents. The share name itself — depending on what it reveals about the server's purpose — can be informative to an attacker.

An anonymous connection attempt to the share itself was then refused, confirming the baseline:

```
smbclient //192.168.50.20/secure_share -N
```

Access denied — the secure state was confirmed.

---

## Introducing the Vulnerability

I edited `/etc/samba/smb.conf` and replaced the `[secure_share]` block with:

```
[open_share]
   path = /srv/smb/share
   browseable = yes
   read only = no
   guest ok = yes
   writable = yes
```

And updated the directory permissions to match:

```
sudo chmod 777 /srv/smb/share
```

The key changes:
- `guest ok = yes` — anonymous access is now permitted; combined with the global `map to guest = Bad User`, any connection attempt succeeds without credentials
- `read only = no` / `writable = yes` — write access granted, including to unauthenticated guests
- `chmod 777` — the underlying filesystem directory permits writes from any user, including `nobody` (which is what Samba maps anonymous connections to at the OS level)

```
testparm
sudo systemctl restart smbd
```

`testparm` confirmed `[open_share]` with `guest ok = Yes` and `read only = No`.

---

## Final State Summary

At the end of this build, `cyberscheme` has:

| Configuration item         | Value                                            |
| -------------------------- | ------------------------------------------------ |
| Share name                 | `open_share`                                     |
| Share path                 | `/srv/smb/share`                                 |
| Guest access               | `guest ok = yes`                                 |
| Write access               | `writable = yes`                                 |
| Filesystem permissions     | `chmod 777`                                      |
| Global guest mapping       | `map to guest = Bad User`                        |
| Lab user (OS + Samba)      | `labuser` / `password123`                        |
| Null session (IPC$)        | Permitted (default Samba behaviour)              |

This configuration supports all planned exercises: anonymous share access and file upload; Nmap NSE enumeration; null session user enumeration via `enum4linux`; password spraying with NetExec; and remote command execution attempts.

---

## What I Learned

- The difference between a secure and insecure Samba share is often just two lines in `smb.conf` (`guest ok` and `read only`/`writable`) — a configuration that could easily be changed temporarily for testing and never reverted.
- Samba's share listing behaviour means share names are typically visible anonymously even when share content is protected — the name itself is information.
- `map to guest = Bad User` in the global config interacts with per-share `guest ok` settings in a way that isn't always obvious: it effectively means that any connection from any client can succeed as guest once `guest ok = yes` is set on a share.
- Both the Samba config (`smb.conf`) and the underlying filesystem permissions (`chmod`) must be considered together — `guest ok = yes` on a directory owned `755` with `nobody:nogroup` means anonymous users can read but not write at the filesystem level, regardless of what `smb.conf` says about writability.
- Samba maintains its own separate password database — a user created with `useradd` is not automatically an SMB user; `smbpasswd -a` must be run separately to add them.

---

## Next: Assessing the Misconfiguration

The next document approaches the service from the attacker's perspective: discovering SMB with Nmap, enumerating shares and users, connecting anonymously, confirming read and write access, and testing credentials — demonstrating the full impact of the configuration built here.

**Next:** `03-anonymous-smb-access-exploitation.md`

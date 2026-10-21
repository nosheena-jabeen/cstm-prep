# Building a Vulnerable SSH Server

## Overview

As part of my penetration testing home lab, I configured the existing SSH service on `cyberscheme` to be exploitable by creating a user account with a weak, guessable password and confirming that password authentication was enabled with no brute-force protections in place.

Unlike most services in this lab, SSH was already running on `cyberscheme` from initial setup — openssh-server was installed during the bulk package install. The vulnerability introduced here is not in SSH itself but in its configuration and the credentials attached to it: password authentication enabled, no lockout policy tuned for a lab environment, and a password that appears in any common wordlist.

> **Important:** The weak credential configuration documented here was created deliberately for training. The vulnerable server is isolated from my normal network and is not exposed to the internet.

---

## Lab Architecture

| Component         | Configuration     |
| ----------------- | ----------------- |
| Virtual network   | VMnet2            |
| Network type      | Host-only         |
| Subnet            | `192.168.50.0/24` |
| DHCP              | Disabled          |
| Internet access   | None              |

| Host          | Purpose     | IP Address      | Services running                                                                        |
| ------------- | ----------- | --------------- | --------------------------------------------------------------------------------------- |
| Kali Linux    | Attacker    | `192.168.50.10` | —                                                                                       |
| Ubuntu Server | Lab target  | `192.168.50.20` | DNS (53), NFS (2049), SNMP (UDP/161), SMB (445), TFTP (UDP/69), FTP (TCP/21), SSH (22) |

> **No-clash note:** SSH runs on TCP/22, which does not overlap with any previously configured service.

---

## SSH Already Running

openssh-server was pre-installed during the initial bulk package install. Confirming the service is active:

```bash
sudo systemctl status ssh
```

SSH listens on TCP/22 by default. No changes to the port or listening address were needed.

---

## Creating the Target User

A dedicated user account was created with a weak password that would appear in a common wordlist:

```bash
sudo useradd -m -s /bin/bash ssh-brute
echo "ssh-brute:Password1" | sudo chpasswd
```

- `useradd -m` — creates the user with a home directory (`/home/ssh-brute`)
- `-s /bin/bash` — assigns bash as the login shell
- `chpasswd` — sets the password non-interactively

The username `ssh-brute` signals its purpose in the lab context. The password `Password1` is a predictable capitalisation-plus-number pattern that appears in most password wordlists.

---

## Confirming Password Authentication is Enabled

On Ubuntu, the default SSH configuration permits password authentication. Confirmed with:

```bash
sudo sshd -T | grep passwordauthentication
```

Output:

```
passwordauthentication yes
```

Password authentication was already enabled. No changes to `/etc/ssh/sshd_config` were required.

---

## Creating the Lab Wordlist

A small wordlist was created on Kali Linux representing the kind of targeted wordlist an attacker might use against a known username:

```bash
sudo bash -c 'cat > /usr/share/wordlists/wordlist-small.txt << EOF
admin
password
letmein
Password1
dragon
qwerty
monkey
EOF'
```

This wordlist is intentionally short — sufficient to demonstrate brute-force success without the noise of a full dictionary attack. In a real assessment, tools like Hydra would typically be run with `rockyou.txt` or a custom wordlist derived from OSINT about the target organisation.

---

## Why pam_faillock Matters Here

Ubuntu 22.04 includes `pam_faillock` by default, which locks user accounts after a configurable number of failed login attempts. Running brute-force tools in the following exercises will trigger this lockout — a realistic side effect of any brute-force attack.

Between exercises, if the account becomes locked, reset it on `cyberscheme`:

```bash
sudo faillock --user ssh-brute --reset
```

This is intentional lab behaviour rather than something to avoid — in a real assessment, account lockout is an observable side effect of brute force that can alert defenders.

---

## Final State Summary

| Configuration item        | Value                                                             |
| ------------------------- | ----------------------------------------------------------------- |
| SSH port                  | TCP/22                                                            |
| Password authentication   | `passwordauthentication yes` (default, confirmed)                 |
| Target user               | `ssh-brute`                                                       |
| Target password           | `Password1` (weak — present in common wordlists)                  |
| User home directory       | `/home/ssh-brute`                                                 |
| Login shell               | `/bin/bash`                                                       |
| Brute-force protection    | `pam_faillock` active (default Ubuntu config)                     |
| Lab wordlist              | `/usr/share/wordlists/wordlist-small.txt` (Kali)                  |

---

## What I Learned

- SSH's default Ubuntu configuration already permits password authentication — no changes were needed to introduce this weakness. The vulnerability is in the credential, not the SSH configuration itself.
- `pam_faillock` is active by default on Ubuntu 22.04 and will lock accounts after failed attempts. Brute-force tools will trigger it, and this is a realistic representation of real-world behaviour — attackers face the same problem.
- Creating a short, targeted wordlist is more realistic for a lab than running a full dictionary attack. A real attacker would tailor their wordlist based on knowledge of the target's password policy.
- The weak password `Password1` follows a pattern (capital first letter, common word, trailing digit) that is extremely common in real environments and appears in virtually every major wordlist.

---

## Next: Assessing the Configuration

The next document approaches the service from the attacker's perspective: discovering SSH via Nmap, brute-forcing the password with Metasploit and Hydra, logging in manually with the cracked credentials, and establishing key-based persistent access.

**Next:** `03-ssh-exploitation.md`

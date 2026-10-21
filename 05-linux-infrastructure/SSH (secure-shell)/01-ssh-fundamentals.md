# SSH Fundamentals (Secure Shell)

## Overview

Before building and assessing a vulnerable SSH configuration in my home lab, I wanted a clear reference for what SSH is, how its two authentication methods work, and why it remains one of the most important services to assess during a penetration test — both as an initial access vector and as a persistence mechanism once access has been gained.

This document is a general-purpose reference. Commands here use placeholder targets and are expanded with real IPs and evidence in the lab write-ups that follow.

---

## 1. What is SSH?

Secure Shell (SSH) is a cryptographic network protocol for operating network services securely over an unsecured network. It is the standard mechanism for remote command-line access to Linux and Unix systems, and is widely used for file transfer (SCP, SFTP) and tunnelling.

Unlike its predecessor Telnet, SSH encrypts all traffic including credentials, making it significantly harder to intercept. However, encryption does not protect against weak passwords, reused credentials, or poorly managed keys — which are the primary SSH attack surfaces in penetration testing.

---

## 2. Protocol Overview

| Property             | Detail                                                                    |
| -------------------- | ------------------------------------------------------------------------- |
| Protocol             | SSH (Secure Shell), version 2 is standard                                 |
| Port                 | TCP/22                                                                    |
| Transport            | TCP — connection-oriented                                                 |
| Encryption           | AES, ChaCha20 (negotiated at connection time)                             |
| Authentication       | Password or public key (both supported simultaneously by default)         |
| Banner disclosure    | Server presents version string before authentication                      |

---

## 3. Authentication Methods

### Password Authentication

The client sends a username and password. The server checks the password against the system's authentication database (PAM on Linux). All traffic is encrypted, but the underlying weakness is that passwords can be guessed or brute-forced if they are weak or common.

```
ssh user@<target>
# Prompted for password — sent encrypted over the wire
```

### Key-Based Authentication

The client presents a cryptographic key pair: a private key held locally and a public key stored on the server in `~/.ssh/authorized_keys`. The server challenges the client to prove it holds the private key without transmitting it. No password is sent or required.

```bash
# Generate a key pair
ssh-keygen -t rsa -b 4096

# Add public key to the server
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

# Connect using the private key
ssh -i ~/.ssh/id_rsa user@<target>
```

Key-based authentication is considered more secure than passwords because:
- No secret is transmitted over the network
- A 4096-bit RSA key cannot be brute-forced with current technology
- Private keys can be passphrase-protected as an additional layer

However, key-based auth introduces its own risks — a private key found on a compromised system can be used to access any server that trusts it.

---

## 4. SSH Key Pair Structure

| File                        | Location               | Description                                           |
| --------------------------- | ---------------------- | ----------------------------------------------------- |
| Private key (`id_rsa`)      | `~/.ssh/id_rsa`        | Secret — never shared, used to authenticate           |
| Public key (`id_rsa.pub`)   | `~/.ssh/id_rsa.pub`    | Shared — placed on servers in `authorized_keys`       |
| Authorised keys             | `~/.ssh/authorized_keys` | Server-side list of trusted public keys             |

Permission requirements (SSH enforces these strictly):

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_rsa
```

SSH will refuse to use keys with overly permissive permissions.

---

## 5. Why Pentesters Target SSH

SSH is a high-value target because successful access grants an interactive shell — the most complete form of access possible on a Linux system.

**Password cracking:** If password authentication is enabled and no rate limiting or lockout policy is in place, automated tools can attempt thousands of username/password combinations. A successful brute force results in direct shell access.

**Key-based attacks:** If an attacker gains write access to a user's `~/.ssh/authorized_keys` file — through any other vulnerability — they can add their own public key and return via SSH without needing a password, even if the user's password is later changed. This is a persistence technique.

**Credential reuse:** SSH passwords obtained through brute force, database dumps, or other means are frequently reused across services. A cracked SSH password may also unlock sudo access, FTP, or application accounts.

**Post-exploitation pivot:** Once inside a system via SSH, an attacker can enumerate the local environment, escalate privileges, access other services, and move laterally.

---

## 6. Tools

| Tool                              | Purpose                                                                 |
| --------------------------------- | ----------------------------------------------------------------------- |
| `nmap -p 22 -sV`                  | Confirm SSH is running and identify the server version                  |
| `hydra`                           | Online brute-force against SSH using a username/wordlist                |
| `auxiliary/scanner/ssh/ssh_login` | Metasploit module for SSH credential brute-forcing                      |
| `ssh`                             | Standard client for manual login (password or key)                      |
| `ssh-keygen`                      | Generate SSH key pairs                                                  |

---

## 7. Identifying SSH Services

Version scan to confirm SSH and identify the implementation:

```bash
nmap -n -sV -p 22 <target>
```

The SSH banner is returned before authentication:

```
22/tcp open  ssh  OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
```

Version disclosure is significant: the exact OpenSSH version and Ubuntu release can be matched against known CVEs without ever attempting a login.

---

## 8. Practical Methodology

**Step 1 — Discover:** TCP scan on port 22 with version detection to confirm SSH and identify the version.

**Step 2 — Brute force (if password auth is enabled):** Use Metasploit `ssh_login` or Hydra with a relevant wordlist. Both tools confirm a valid credential pair on success.

**Step 3 — Manual login:** Connect using the cracked credentials and confirm shell access. Note the hostname, current user, and accessible directories.

**Step 4 — Establish key-based persistence:** Generate a key pair on the attacker machine, add the public key to the target's `authorized_keys`, and confirm passwordless login. This simulates how an attacker would maintain access independently of the cracked password.

**Step 5 — Assess scope:** Identify what the user can access, whether sudo is available, and what other services or files are reachable from the shell.

---

## Key Security Takeaways

- Password authentication with no rate limiting or lockout policy is directly vulnerable to automated brute force — Hydra and Metasploit can cycle through thousands of attempts in seconds.
- The SSH banner discloses the server version before any login attempt — useful for CVE matching without interaction.
- Key-based access, once established, persists independently of the user's password. Removing a user's password or changing it does not revoke access if their `authorized_keys` file contains an attacker-controlled key.
- `pam_faillock` on modern Ubuntu will lock accounts after repeated failed login attempts — brute force leaves traces and has operational side effects even when it succeeds.
- SSH is frequently the means by which attackers maintain long-term access after initial compromise — auditing `authorized_keys` files is a critical step in incident response.

---

## Related Labs

`02-vulnerable-ssh-server.md`
`03-ssh-exploitation.md`

# FTP Fundamentals (File Transfer Protocol)

## Overview

Before building and assessing a vulnerable FTP service in my home lab, I wanted a clear reference for what FTP is, how it works at a protocol level, and why it remains a common source of findings in penetration testing assessments despite being a well-understood and decades-old protocol.

This document is a general-purpose summary rather than a record of a specific lab session — the commands here use placeholder targets and are expanded on with real IPs and evidence in the lab write-ups that follow.

---

## 1. What is FTP?

File Transfer Protocol (FTP) is a client-server protocol for transferring files over a network. It is one of the oldest application-layer protocols in common use, predating the modern internet. FTP was designed for simplicity and interoperability, not for security — it transmits both credentials and file data in plaintext, making it trivially interceptable on any shared or monitored network.

Despite its age and well-documented weaknesses, FTP remains in widespread use in enterprise environments, particularly for internal file distribution, legacy application support, and automated transfer processes that predate more secure alternatives.

---

## 2. Protocol Overview

| Property              | Detail                                                                      |
| ---------------------- | --------------------------------------------------------------------------- |
| Protocol               | FTP (File Transfer Protocol)                                                |
| Control port           | TCP/21 — command channel (login, commands)                                  |
| Data port              | TCP/20 (active mode) or ephemeral port (passive mode)                       |
| Transport              | TCP — connection-oriented                                                   |
| Authentication         | Username and password (transmitted in plaintext)                            |
| Encryption             | None (plain FTP) — FTPS adds TLS; SFTP is a separate SSH-based protocol    |
| Directory listing      | Supported (`ls`, `dir`)                                                     |
| Anonymous access       | Optional — server can be configured to allow login with username `anonymous`|

---

## 3. How FTP Works

FTP uses two separate TCP connections:

**Control connection (TCP/21):** Carries commands and responses for the duration of the session. Login credentials pass through this channel in plaintext.

**Data connection (TCP/20 or ephemeral):** A separate connection opened for each file transfer or directory listing. This is the source of FTP's two operating modes:

- **Active mode:** The client tells the server which port to connect back to; the server initiates the data connection from port 20. Commonly blocked by firewalls protecting the client.
- **Passive mode:** The server opens a random high port and tells the client to connect to it. The client initiates the data connection. The modern default — firewall-friendly from the client side.

```
Client                        Server (TCP/21)
  |                                 |
  |--- USER anonymous ------------->|
  |<-- 331 Password required -------|
  |--- PASS user@example.com ------>|
  |<-- 230 Login successful --------|
  |--- PASV --------------------->  |
  |<-- 227 Entering Passive Mode -- |   (server opens data port)
  |--- RETR backup.txt ------------>|
  |<-- (data connection opened) ----|
  |<-- (file contents) -------------|
  |--- QUIT ------------------------>|
```

The key security observation: the `USER` and `PASS` commands pass over the wire in plaintext. Any observer on the same network segment can read credentials.

---

## 4. Anonymous FTP

FTP servers can be configured to allow login with the username `anonymous` and no real password (convention is to enter an email address, which is not validated). This was originally designed for public file distribution — software archives, documentation, public datasets.

In misconfigured environments, anonymous access is enabled on servers that were never intended to be public, exposing internal files without requiring any credentials. When write access is also granted to anonymous users (a separate misconfiguration), an unauthenticated attacker can upload files to the server.

---

## 5. FTP vs TFTP vs SFTP

| Feature           | FTP                          | TFTP                         | SFTP                              |
| ------------------ | ----------------------------- | ----------------------------- | ---------------------------------- |
| Transport          | TCP                           | UDP                           | TCP (over SSH)                     |
| Authentication     | Username + password           | None                          | Key-based or password              |
| Encryption         | None                          | None                          | Full (SSH)                         |
| Directory listing  | Yes                           | No                            | Yes                                |
| Port               | TCP/21 (control)              | UDP/69                        | TCP/22                             |
| Use case           | General file transfer         | Network device boot/config    | Secure file transfer               |

---

## 6. vsftpd Configuration

On Linux, the most common FTP server is `vsftpd`. Its configuration lives at `/etc/vsftpd.conf`:

| Option                       | Effect                                                                             |
| ----------------------------- | ---------------------------------------------------------------------------------- |
| `anonymous_enable=YES`        | Enables anonymous login — the key misconfiguration for anonymous access findings   |
| `write_enable=YES`            | Permits write commands (required globally for any write access)                    |
| `anon_upload_enable=YES`      | Allows anonymous users to upload files — must be combined with `write_enable`      |
| `anon_mkdir_write_enable=YES` | Allows anonymous users to create directories                                       |
| `local_enable=YES`            | Allows local system accounts to log in                                             |
| `chroot_local_user=YES`       | Restricts authenticated local users to their home directory                        |

The default vsftpd installation on Ubuntu has `anonymous_enable=NO`. Anonymous access is introduced by explicitly setting it to `YES`.

**vsftpd chroot constraint:** vsftpd refuses to run when its anonymous root directory (`/srv/ftp` by default) is writable by the FTP user. This means a writable subdirectory must be created for uploads — the anonymous root itself must have restricted permissions (`chmod 555`), with a subdirectory (`uploads/`) set to `chmod 777`.

---

## 7. Identifying FTP Services

Version scan to confirm the service and identify the implementation and version:

```bash
nmap -sSV -p 21 <target>
```

NSE script to test for anonymous access automatically:

```bash
nmap -p 21 --script=ftp-anon <target>
```

The `ftp-anon` script attempts an anonymous login and, if successful, lists accessible files. A positive result in a scan is equivalent to finding `anonymous_enable=YES` in the config — confirmed without touching the service manually.

---

## 8. Interacting with FTP

Connect and log in anonymously:

```bash
ftp <target>
# Username: anonymous
# Password: [blank or any email address]
```

Common FTP client commands:

| Command            | Effect                                              |
| ------------------- | --------------------------------------------------- |
| `ls`                | List directory contents                             |
| `cd <dir>`          | Change directory                                    |
| `get <file>`        | Download a file from the server                     |
| `put <file>`        | Upload a file to the server (if write access exists)|
| `bye`               | Exit the FTP session                                |

---

## Practical Methodology

**Step 1 — Discover:** TCP scan on port 21 with version detection to confirm FTP and identify the implementation.

**Step 2 — Test for anonymous access:** Run `ftp-anon` NSE script; if it returns a file listing, anonymous login is confirmed without manual interaction.

**Step 3 — Enumerate manually:** Connect as `anonymous`, list contents, note any sensitive files. Attempt `get` on any files of interest.

**Step 4 — Test write access:** Attempt `put` with a test file. If successful, confirm the file landed in the appropriate directory on the server.

**Step 5 — Assess scope:** Understand what the FTP root and any subdirectories contain and what processes interact with them.

---

## Key Security Takeaways

- FTP transmits credentials and file contents in plaintext — interception requires only network access, no active attack.
- Anonymous access (`anonymous_enable=YES`) removes the credential requirement entirely — any host that can reach TCP/21 can retrieve any readable file.
- Write access combined with anonymous login allows unauthenticated file upload — depending on what other services process files from that directory, this can be a vector for further compromise.
- The `ftp-anon` NSE script provides immediate automated confirmation of anonymous access — a positive result is a finding, no manual interaction required.
- vsftpd version disclosure via banner (`220 (vsFTPd 3.0.5)`) reveals the server version to any connecting client — useful for matching against known CVEs.

---

## Related Labs

For a hands-on build and exploitation walkthrough:

`02-vulnerable-ftp-server.md`
`03-anonymous-ftp-access-exploitation.md`

For the vsftpd 2.3.4 backdoor (CVE-2011-2523) — a separate, distinct finding requiring a specific vulnerable version:

`metasploitable/vsftpd-234-backdoor.md` *(planned)*

# TFTP Fundamentals (Trivial File Transfer Protocol)

## Overview

Before building and assessing a vulnerable TFTP service in my home lab, I wanted a clear reference for what TFTP actually is, how it differs from FTP, and why a protocol with no authentication whatsoever continues to appear in penetration testing assessments.

This document is a general-purpose summary rather than a record of a specific lab session — the commands here use placeholder targets and are expanded on with real IPs and evidence in the lab write-ups that follow.

---

## 1. What is TFTP?

Trivial File Transfer Protocol (TFTP) is a simple, lightweight file transfer protocol designed for environments where simplicity matters more than security. It was originally developed for transferring small files — most commonly OS images and configuration files — during network device boot sequences and firmware updates.

TFTP is called "trivial" deliberately: it implements only the bare minimum needed to read and write files across a network. There is no authentication, no encryption, no directory listing, and no login mechanism of any kind. A client that knows a filename can request it, and if the server allows it, the file is transferred.

---

## 2. Protocol Overview

| Property              | Detail                                                                    |
| ---------------------- | --------------------------------------------------------------------------- |
| Protocol               | TFTP (Trivial File Transfer Protocol)                                       |
| Port                   | UDP/69                                                                      |
| Transport              | UDP — connectionless, no session establishment                              |
| Authentication         | None — by design                                                            |
| Encryption             | None — all transfers in plaintext                                           |
| Directory listing      | Not supported — client must know the filename in advance                    |
| Common use cases       | PXE boot (network OS loading), router/switch firmware and config transfer, embedded device provisioning |

---

## 3. How TFTP Works

TFTP uses a simple request-response model over UDP:

1. The client sends a **Read Request (RRQ)** or **Write Request (WRQ)** to the server on UDP/69, specifying the filename and transfer mode.
2. The server responds by opening a new ephemeral port and beginning the transfer.
3. Files are transferred in 512-byte blocks (by default), with each block acknowledged before the next is sent.
4. Transfer ends when a block smaller than 512 bytes is received, signalling the end of the file.

```
Client                              Server (UDP/69)
  |                                      |
  |--- RRQ "config.txt" --------------->|
  |<-- DATA block 1 (512 bytes) --------|
  |--- ACK block 1 ------------------->|
  |<-- DATA block 2 (remainder) --------|
  |--- ACK block 2 ------------------->|
  |        (transfer complete)          |
```

The key point is that there is no login, no challenge, no credential exchange — just a filename and a transfer.

---

## 4. TFTP vs FTP

| Feature           | TFTP                         | FTP                              |
| ------------------ | ----------------------------- | --------------------------------- |
| Transport          | UDP                           | TCP                               |
| Authentication     | None                          | Username and password             |
| Encryption         | None                          | None (plain FTP) / yes (FTPS)     |
| Directory listing  | Not supported                 | Supported (`ls`, `dir`)           |
| File download      | Yes (if file exists and readable) | Yes (with permissions)        |
| File upload        | Optional (server must permit) | Yes (with permissions)            |
| Port               | UDP/69                        | TCP/21 (control), TCP/20 (data)   |
| Common use         | Network boot, device config   | General file transfer             |

---

## 5. Why TFTP Matters in Security

TFTP's complete lack of authentication makes it one of the more straightforward services to exploit when it is found accessible on a network:

- **No credentials required** — any client can request any file by name without proving identity
- **Sensitive file exposure** — network devices commonly store configuration files on TFTP servers; these often contain credentials, network topology, SNMP community strings, and other sensitive data
- **Upload capability** — if the server is configured to allow writes (`--create`), an attacker can write arbitrary files to the server with no authentication
- **UDP means no connection** — TFTP leaves minimal trace compared to TCP-based protocols; there is no session establishment to log
- **Still commonly found** — TFTP remains in use in enterprise environments for PXE booting and network device provisioning, and is often left running and world-accessible long after its original purpose has been fulfilled

---

## 6. tftpd-hpa Configuration

On Linux, the most common TFTP server is `tftpd-hpa`. Its configuration lives at `/etc/default/tftpd-hpa`:

| Option              | Effect                                                                      |
| -------------------- | ----------------------------------------------------------------------------- |
| `TFTP_DIRECTORY`     | The root directory the server serves files from                               |
| `TFTP_ADDRESS`       | The address and port to listen on; `":69"` means all interfaces               |
| `--secure`           | Chroots the server to `TFTP_DIRECTORY` — clients cannot traverse outside it   |
| `--create`           | Allows clients to upload (write) files — **disabled by default; the vulnerable option** |

The default installation does **not** include `--create` — uploads are disabled out of the box. Adding `--create` is the deliberate misconfiguration that enables the write access finding.

---

## 7. Identifying TFTP Services

TFTP runs on UDP, so a standard TCP scan misses it entirely:

```bash
sudo nmap -sU -p 69 <target>
```

A version scan can help confirm the implementation:

```bash
sudo nmap -sU -p 69 -sV <target>
```

---

## 8. Interacting with TFTP

Download a file:

```bash
tftp <target>
tftp> get <filename>
tftp> quit
```

Upload a file:

```bash
tftp <target>
tftp> put <local_file> <remote_filename>
tftp> quit
```

TFTP has no `ls` or directory listing — you must know the filename in advance. In a real assessment, filenames are often guessed from common defaults: `config.txt`, `running-config`, `startup-config`, `backup.cfg`, firmware image names, and so on.

---

## Practical Methodology

**Step 1 — Discover:** UDP scan on port 69 to confirm TFTP is running.

**Step 2 — Test read access:** Attempt to retrieve common filenames — configuration files, backup files, firmware images. No credentials are needed if the server is accessible.

**Step 3 — Test write access:** Attempt to upload a file. If `--create` is enabled, the upload will succeed. Confirm the file landed on the server.

**Step 4 — Assess scope:** Understand what the TFTP root directory contains and how it is used — a configuration backup directory is a much more significant finding than a directory used only for PXE boot images.

---

## Key Security Takeaways

- TFTP has no authentication by design — exposure to untrusted networks is always a finding, regardless of what files are currently present.
- Sensitive files on TFTP servers (network device configs, credential files) represent immediate data exposure.
- Upload capability (`--create`) turns a read finding into a write finding — arbitrary file creation on the server without any authentication.
- TFTP on UDP/69 is routinely missed by TCP-only port scans, meaning it can persist unnoticed in network assessments that don't include UDP scanning.
- The protocol transmits in plaintext — file contents are visible to any observer on the same network segment.

---

## Related Labs

For a hands-on build and exploitation walkthrough:

`02-vulnerable-tftp-server.md`
`03-unauthenticated-tftp-file-access-exploitation.md`

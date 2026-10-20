# SMB Fundamentals (Server Message Block)

## Overview

Before building and assessing a vulnerable SMB service in my home lab, I wanted a clear reference for what SMB actually is, how it works at a protocol level, and why anonymous share access, null session enumeration, credential attacks, and remote execution are consistently valuable findings in penetration testing.

This document is a general-purpose summary rather than a record of a specific lab session — the commands here use placeholder targets and are expanded on with real IPs and evidence in the lab write-ups that follow.

---

## 1. What is SMB?

Server Message Block (SMB) is a network file-sharing protocol that allows systems to share files, printers, and other resources across a network. Originally developed by IBM and later extended by Microsoft, it is the primary file-sharing protocol on Windows networks and is also widely implemented on Linux and Unix systems through **Samba** — an open-source SMB implementation.

An SMB server exposes **shares** — named directories or resources that clients can connect to and interact with. Access to each share is governed by its own permission configuration, and clients authenticate (or fail to authenticate) before being granted access.

---

## 2. Protocol Overview

| Property              | Detail                                                                  |
| ---------------------- | ------------------------------------------------------------------------- |
| Protocol               | SMB (Server Message Block)                                                |
| Common ports           | TCP/445 (direct SMB); TCP/139 (SMB over NetBIOS)                          |
| Transport              | TCP — connection-oriented, reliable                                       |
| Linux implementation   | Samba (`smbd` for file/print sharing, `nmbd` for NetBIOS name resolution) |
| Windows implementation | Native, built into all modern Windows versions                            |
| Security by default    | Varies — modern SMB3 adds encryption and signing, but misconfigurations remain extremely common |

---

## 3. SMB Versions

| Version | Key Characteristics                                                                          |
| -------- | ---------------------------------------------------------------------------------------------- |
| SMB1     | Legacy; insecure; disabled by default on modern systems; responsible for WannaCry/EternalBlue (MS17-010) |
| SMB2     | Introduced with Windows Vista; improved performance and security                               |
| SMB3     | Introduced with Windows 8/Server 2012; adds encryption, secure negotiation, and signing; SMBGhost (CVE-2020-0796) showed even SMB3 can have critical flaws |

Even with SMB2/3 in use, the most common SMB findings in penetration testing are configuration-level rather than protocol-level. Anonymous access, default credentials, weak share permissions, and null sessions are far more frequently encountered than protocol vulnerabilities.

---

## 4. Key Samba Configuration Concepts

Samba's behaviour is controlled by `/etc/samba/smb.conf`, which contains a `[global]` section and per-share sections.

Common share-level options:

| Option                    | Effect                                                                  |
| -------------------------- | ------------------------------------------------------------------------- |
| `guest ok = yes`           | Allows anonymous (unauthenticated) access                                 |
| `guest ok = no`            | Requires valid credentials — the secure default                           |
| `read only = yes`          | Read-only access, even for authenticated users                            |
| `read only = no`           | Read-write access                                                         |
| `writable = yes`           | Equivalent to `read only = no`                                            |
| `browseable = yes`         | Share appears in listings; `no` hides it from listings                    |
| `map to guest = Bad User`  | Maps failed authentication attempts to the guest account — combined with `guest ok = yes`, allows any connection to succeed anonymously |
| `valid users = <user>`     | Restricts share access to specific users only                             |

---

## 5. Why SMB Matters in Security

SMB is one of the highest-value targets in network penetration testing because:

- It is ubiquitous — present on nearly every Windows machine and many Linux servers
- Share listings are often returned anonymously even when content is protected, leaking share names
- SMB null sessions (unauthenticated IPC$ connections) allow user and group enumeration on many configurations
- Write access to a share, even as an anonymous user, enables file planting
- Valid credentials allow remote command execution on Windows targets via tools like NetExec
- SMB message signing, if not required, leaves the environment open to relay attacks
- Legacy vulnerabilities (EternalBlue/MS17-010) and modern ones (SMBGhost/CVE-2020-0796) demonstrate that both configuration and protocol flaws exist

---

## 6. SMB Tools to Know

| Tool                  | Purpose                                                                       |
| ---------------------- | ----------------------------------------------------------------------------- |
| `smbclient`            | SMB client — list shares, connect, upload/download files                      |
| `enum4linux`           | Comprehensive SMB/RPC enumeration — shares, users, groups, password policy    |
| `netexec` (nxc)        | Modern pentest tool — credential testing, password spraying, command execution, NTDS.dit dumping |
| `nmap --script "smb*"` | NSE scripts for SMB discovery, OS detection, capabilities, vulnerability scanning |
| `net use` (Windows)    | Windows built-in for connecting to shares and testing null sessions            |

---

## 7. Identifying SMB Services

```bash
nmap -sT -p 445 <target>
nmap -sV -p 139,445 <target>
nmap -sT -p 445 --script "smb*" <target>
```

Key things to look for in NSE output:
- `smb-protocols` — which versions are enabled (SMB1 present is a finding)
- `smb2-security-mode: Message signing enabled but not required` — a consistently findable misconfiguration enabling relay attacks
- `smb2-capabilities` — advertised capabilities per dialect
- `smb-vuln-*` results — whether known CVEs are present

---

## 8. Enumerating SMB Shares

List available shares anonymously:

```bash
smbclient -L //<target> -N
```

Connect to a specific share anonymously:

```bash
smbclient //<target>/<share> -N
```

Non-interactive listing (clean for documentation):

```bash
smbclient //<target>/<share> -N -c "ls"
```

Download a file from within a share:

```bash
get FILENAME
```

---

## 9. Comprehensive Enumeration

```bash
enum4linux -a <target>
```

Runs all checks: shares, users (via SID/RID cycling), groups, OS information, password policy, workgroup/domain. All without credentials if null sessions are permitted.

---

## 10. Credential Attacks with NetExec

Password spray against SMB:

```bash
netexec smb <target> -u <username> -p <password>
```

Remote command execution (Windows targets only):

```bash
netexec smb <target> -u <username> -p <password> --exec-method smbexec -x whoami
```

Dump NTDS.dit — Active Directory password hashes (requires domain admin, Windows DC only):

```bash
netexec smb <target> -u <username> -p <password> --ntds
```

> **Note:** Remote command execution and NTDS.dit dumping via NetExec require a Windows target. These techniques do not apply to Linux/Samba, which does not implement the Windows service execution and VSS mechanisms NetExec relies on for these operations. See the Windows AD lab for practical demonstrations of these techniques.

---

## Practical Methodology

**Step 1 — Discover:** confirm SMB on TCP/445 with Nmap; run `smb*` NSE scripts for protocol versions, capabilities, and security mode.

**Step 2 — List shares:** attempt anonymous share listing — names are often returned even when content is protected.

**Step 3 — Test access:** attempt anonymous connection to each share; test read and write access independently.

**Step 4 — Enumerate users:** run `enum4linux -a` for null session user, group, and policy enumeration.

**Step 5 — Credential attack:** use discovered usernames with NetExec to password spray against SMB.

**Step 6 — Remote execution:** with valid credentials on a Windows target, attempt command execution via NetExec; escalate toward NTDS.dit if domain admin credentials are available.

---

## Key Security Takeaways

- Share listings are typically returned anonymously — share names themselves are information disclosure.
- `guest ok = yes` combined with `map to guest = Bad User` makes anonymous access effectively unavoidable once configured.
- `Message signing enabled but not required` is a real, reportable misconfiguration that enables SMB relay attacks.
- `enum4linux` can enumerate valid usernames without credentials, providing targets for subsequent password attacks.
- NetExec bridges "found a weak password" and "have a shell" on Windows targets — credential validation, command execution, and hash dumping all flow from the same tool.
- The same techniques that apply to Linux/Samba (share enumeration, null sessions, credential testing) also apply to Windows; the additional Windows-specific techniques (remote execution, NTDS.dit) require a Windows target and are covered in the Windows AD lab.

---

## Related Labs

For hands-on build and exploitation walkthroughs (Linux/Samba):

`02-vulnerable-smb-server.md`
`03-anonymous-smb-access-exploitation.md`

For Windows-specific SMB attacks (NetExec command execution, NTDS.dit dump):

`Windows-AD/` *(planned — requires Windows Server DC)*

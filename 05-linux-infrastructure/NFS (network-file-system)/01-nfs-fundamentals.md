# NFS Fundamentals (Network File System)

## Overview

Before building and attacking a vulnerable NFS server in my home lab, I wanted a clear reference for what NFS actually is, how it works at a protocol level, and which configuration choices turn it from a convenient file-sharing protocol into a privilege escalation path.

This document is a general-purpose summary rather than a record of a specific lab session — the commands here use placeholder targets (`<target>`, `/shared`) and are expanded on with real IPs and evidence in the lab write-ups that follow.

---

## 1. What is NFS?

Network File System (NFS) lets a server share directories with other machines over a network, so that a client can mount a remote directory and interact with it as though it were local.

A server **exports** a directory (for example, `/shared`), and an authorised client **mounts** it, after which reads and writes to that mount point are transparently sent to the server.

---

## 2. Protocol Overview

| Property          | Detail                                                        |
| ------------------ | --------------------------------------------------------------- |
| Protocol            | NFS (Network File System)                                       |
| Default port        | TCP/2049 (NFSv4); NFSv3 additionally depends on `rpcbind`/TCP-UDP 111 and dynamically assigned RPC ports |
| Transport            | Built on Sun RPC (Remote Procedure Call) for all versions; NFSv4 consolidated this onto a single well-known port, removing the need for `rpcbind` |
| Purpose              | Remote file sharing across networked systems                    |
| Security by default  | Minimal — historically relies on network-level and client-asserted identity rather than strong authentication |

NFSv3 requires `rpcbind` (port 111) to map RPC program numbers to the dynamic ports used by `mountd`, `nfsd`, and related services, which is why older NFS deployments often show several open ports beyond 2049. NFSv4 removed this dependency by running everything over a single TCP connection to port 2049, which simplifies both deployment and firewalling.

---

## 3. Why NFS Matters in Security

NFS is frequently misconfigured, and its risks tend to fall into a few recurring categories:

- Exports reachable by broader networks than intended, or with no host restriction at all
- Weak or absent authentication — NFS has historically trusted the UID/GID a client asserts, rather than verifying identity cryptographically
- `no_root_squash`, which allows a client's root user to act as genuine root on the server
- Sensitive files (credentials, SSH keys, configuration, backups) reachable through an export
- World-writable exported directories enabling file tampering or planting malicious content

In penetration testing, a misconfigured NFS export can provide direct file access without any exploit, and in the right conditions can lead to full privilege escalation on the server.

---

## 4. How NFS Works

1. The server declares which directories are shared, and with what permissions, in `/etc/exports`.
2. The server applies that configuration (`exportfs`) and starts or reloads the NFS service.
3. A client queries the server for available exports.
4. The client mounts the chosen export at a local mount point.
5. File operations against that mount point are transparently translated into NFS requests to the server.

```
Server                                    Client
  |                                          |
  | 1. Export declared in /etc/exports       |
  | 2. exportfs -ra applies the export        |
  |                                          |
  |<----------- 3. showmount -e -------------|
  |------------ export list ---------------->|
  |                                          |
  |<----------- 4. mount request -------------|
  |------------ mount established ----------->|
  |                                          |
  |<=========== 5. file I/O over mount =======>|
```

Example export entry:

```
/shared 192.168.1.0/24(rw,sync,no_subtree_check)
```

This means:

- `/shared` is the directory being exported.
- It is accessible to any host in `192.168.1.0/24`.
- `rw` grants read-write access.
- `sync` commits writes to stable storage before acknowledging them to the client (the safer default; `async` improves performance at the cost of acknowledging writes before they're guaranteed durable).
- `no_subtree_check` disables an additional consistency check NFS performs when exporting a subdirectory of a filesystem rather than the filesystem's root — commonly disabled, as the check can itself cause access problems without meaningfully improving security.

---

## 5. Key `/etc/exports` Options

| Option            | Effect                                                                 |
| ------------------- | ------------------------------------------------------------------------- |
| `rw`                 | Grants read-write access                                                  |
| `ro`                 | Grants read-only access                                                   |
| `sync` / `async`     | Controls whether writes are confirmed durable before being acknowledged   |
| `no_subtree_check`   | Disables subtree consistency checking (common when exporting a subdirectory) |
| `root_squash`        | **Default.** Maps a client's root UID (0) to an unprivileged account (typically `nobody:nogroup`) on the server |
| `no_root_squash`     | Disables that protection — a client's root user retains genuine root privileges over the export |
| `all_squash`         | Maps **every** client UID/GID to an unprivileged account, regardless of what the client asserts |

`root_squash` exists specifically because NFS, by default, trusts the UID a client presents. Without it, any client machine's root user would automatically be treated as root by the server for files within the export.

---

## 6. The Risk of `no_root_squash`

If `no_root_squash` is set on an export:

- A client's root user is **not** remapped, and retains genuine root privileges over files within that export.
- An attacker with root access to any host permitted to mount the export can create, modify, or delete files as root on the server.
- If the export includes sensitive paths — SSH configuration, cron jobs, setuid binaries, or (in the worst case) the filesystem root itself — this can lead directly to full privilege escalation or remote account compromise on the NFS server.

This is one of the most consistently exploitable NFS misconfigurations, precisely because it requires no vulnerability in NFS itself — only a permissive export definition.

---

## 7. Identifying NFS Services

Scan for the NFS port directly:

```bash
nmap -p 2049 <target>
```

For NFSv3 environments, also check `rpcbind`, since it's often present alongside NFS and can reveal additional RPC-based services:

```bash
nmap -p 111 <target>
```

A version/service scan can help distinguish NFS from other services sharing the port range and may reveal the NFS version in use:

```bash
nmap -sV -p 111,2049 <target>
```

---

## 8. Enumerating NFS Shares

Once NFS is confirmed, list the exports the server is offering:

```bash
showmount -e <target>
```

Example output:

```
Export list for 192.168.1.10:
/shared 192.168.1.0/24
```

This shows what is exported and which hosts or networks are permitted to mount it — important for understanding scope before attempting to mount anything.

---

## 9. Mounting an NFS Share

Create a local mount point:

```bash
mkdir /mnt/nfs
```

Mount the share:

```bash
mount -t nfs <target>:/shared /mnt/nfs
```

Browse the contents as if they were local:

```bash
ls -la /mnt/nfs
```

Unmount when finished:

```bash
umount /mnt/nfs
```

---

# Practical Methodology

## Step 1 — Identify NFS on a Target

```bash
nmap -p 2049 <target>
nmap -p 111 <target>
```

Questions to answer:

- Is TCP/2049 open?
- Is `rpcbind` (111) also present, suggesting NFSv3?

## Step 2 — Enumerate Exported Shares

```bash
showmount -e <target>
```

Questions to answer:

- Which directories are exported?
- Are exports restricted to specific hosts or networks, or open broadly?
- Does any export path suggest sensitive content (`/home`, `/etc`, `/`, backup directories)?

## Step 3 — Mount and Inspect

```bash
mount -t nfs <target>:/shared /mnt/nfs
ls -la /mnt/nfs
```

Look specifically for:

- SSH keys and `authorized_keys` files
- Configuration files containing credentials
- Backup files or archives
- Scripts that might run with elevated privileges elsewhere
- World-writable directories

## Step 4 — Test for `no_root_squash`

As root on the client:

```bash
touch /mnt/nfs/testfile
ls -la /mnt/nfs/testfile
```

- If the file is owned by `nobody:nogroup` on the server, `root_squash` is active (the default, secure behaviour).
- If the file is owned by `root:root` on the server, `no_root_squash` is in effect — client-side root is being trusted directly, and further privilege-escalation testing is warranted.

## Step 5 — Assess the Wider Implications

- Are exports limited to specific IP ranges, or reachable by any client?
- Is write access (`rw`) enabled where it isn't required?
- Does `no_root_squash` apply to an export containing sensitive paths — or worse, the filesystem root?
- Could write access through the export be used to influence another service (for example, planting an SSH key, modifying a cron job, or replacing a script executed with elevated privileges)?

---

## Key Security Takeaways

- NFS, especially in its older forms, relies heavily on network-level and client-asserted trust rather than strong authentication.
- Enumeration (`showmount -e`) and mounting are separate steps from understanding the *privileges* an export grants — the two must be tested together.
- `root_squash` is the specific control preventing client-side root from becoming server-side root; `no_root_squash` removes it entirely.
- The impact of `no_root_squash` scales with the scope of the export: a narrow, purpose-specific directory is a much smaller exposure than exporting `/`.
- Write access through an over-permissioned NFS export can be a stepping stone to compromising other services on the same host (SSH, cron, setuid binaries), not just a file-disclosure issue in isolation.

---

## Related Labs

For a hands-on build of a deliberately vulnerable NFS server and a full exploitation chain using these concepts, see:

`04-vulnerable-nfs-server.md`
`05-nfs-ssh-key-privilege-escalation.md`

# NFS Fundamentals (Network File System)

## Protocol Overview

- Protocol: `NFS`
- Default Port: `2049`
- Transport: Typically `TCP` (older versions may rely on `RPC` services)
- Purpose: Remote file sharing across networked systems
- Security by Default: Minimal (often relies on network-level trust)

---

## 1. What is NFS?

Network File System (NFS) allows a system to share directories and files with other machines over a network.

It enables users to mount remote file systems as if they were local directories.

For example, a Linux server may export `/shared`, and another machine can mount it and access its contents directly.

---

## 2. Why NFS Matters in Security

NFS is frequently misconfigured.

Security risks include:

- Anonymous mounting of shared directories
- Exposure of sensitive files
- Weak permission controls
- Root privilege abuse through misconfiguration
- Information disclosure

In penetration testing, improperly configured NFS shares can provide direct access to sensitive files or allow privilege escalation.

---

## 3. How NFS Works

1. A server exports a directory.
2. The exported directory is listed in `/etc/exports`.
3. A client connects to the server.
4. The client mounts the shared directory locally.

Example export entry:

```
/shared 192.168.1.0/24(rw,sync,no_subtree_check)
```

This means:

- `/shared` is being exported
- Accessible to the `192.168.1.0/24` network
- With read-write (`rw`) permissions

---

## 4. Key NFS Configuration Concepts

### /etc/exports

Defines which directories are shared and with what permissions.

Common options:

- `rw` – Read and write access
- `ro` – Read-only access
- `sync` – Changes written immediately
- `no_root_squash` – Dangerous option that preserves root privileges
- `root_squash` – Maps remote root to a non-privileged user

---

## 5. The Risk of no_root_squash

If `no_root_squash` is enabled:

- A remote root user maintains root privileges
- Attackers can create files as root
- Privilege escalation becomes possible

This is one of the most common NFS exploitation paths.

---

## 6. Identifying NFS Services

Scan for NFS:

```bash
nmap -p 2049 <target>
```

Check RPC services:

```bash
nmap -sV <target>
```

You may also see related services such as `rpcbind` (port `111`).

---

## 7. Enumerating NFS Shares

List available exports:

```bash
showmount -e <target>
```

This displays exported directories.

Example output:

```
Export list for 192.168.1.10:
/shared 192.168.1.0/24
```

---

## 8. Mounting an NFS Share

Create a local mount directory:

```bash
mkdir /mnt/nfs
```

Mount the share:

```bash
mount -t nfs <target>:/shared /mnt/nfs
```

Now you can browse the contents locally:

```bash
ls -la /mnt/nfs
```

---

# Practical Lessons

## Lesson 1 – Identify NFS on a Target

### Objective
Determine whether NFS is running.

```bash
nmap -p 2049 <target>
```

Check for `rpcbind`:

```bash
nmap -p 111 <target>
```

Observe:

- Is port `2049` open?
- Is `rpcbind` active?

---

## Lesson 2 – Enumerate Exported Shares

```bash
showmount -e <target>
```

Questions:

- Which directories are exported?
- Are they restricted to certain networks?
- Is the export publicly accessible?

---

## Lesson 3 – Mount and Inspect

Mount the share:

```bash
mount -t nfs <target>:/shared /mnt/nfs
```

Inspect:

```bash
ls -la /mnt/nfs
```

Look for:

- Backup files
- SSH keys
- Configuration files
- Scripts
- World-writable directories

---

## Lesson 4 – Check for Root Squash Misconfiguration

If the export includes `no_root_squash`, test file creation:

```bash
touch /mnt/nfs/testfile
```

Check ownership:

```bash
ls -la /mnt/nfs
```

If files are created as root, this may allow privilege escalation.

---

## Lesson 5 – Security Assessment Questions

- Are exports limited to specific IP ranges?
- Is write access enabled?
- Is `no_root_squash` configured?
- Are sensitive directories exposed?

---

## Key Security Takeaways

- NFS often relies on trust-based network controls.
- Misconfigured exports can expose sensitive data.
- `no_root_squash` can lead to privilege escalation.
- Export restrictions should be tightly controlled.
- Monitoring and proper permission management are essential.

---

## Summary

NFS enables remote file sharing but can introduce serious security risks if misconfigured. Proper enumeration and inspection of NFS services can reveal sensitive data exposure and potential privilege escalation paths.

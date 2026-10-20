# Building a Vulnerable FTP Server

## Overview

As part of my penetration testing home lab, I installed and configured `vsftpd` on Ubuntu, placed a simulated sensitive file in the FTP root directory, and deliberately enabled anonymous access with upload permissions — introducing the two key misconfigurations that make FTP a finding in real assessments: unauthenticated read access to sensitive files and unauthenticated write access to the server.

Unlike TFTP where the absence of authentication is a protocol-level design choice, vsftpd's default installation has `anonymous_enable=NO`. Anonymous access here is an explicit configuration change — a deliberate decision that has often been made in real environments for convenience, for legacy compatibility, or simply because it was enabled temporarily and never reverted.

The environment reused the same Kali Linux attacker machine and Ubuntu Server (`cyberscheme`) on the isolated VMware host-only network from earlier labs. vsftpd was pre-installed during an earlier bulk install session to avoid repeatedly switching the VM between isolated and NAT networks.

> **Important:** The insecure FTP configuration documented here was created deliberately for training. The vulnerable server is isolated from my normal network and is not exposed to the internet.

---

## Lab Architecture

| Component        | Configuration     |
| ----------------- | ----------------- |
| Virtual network   | VMnet2            |
| Network type      | Host-only         |
| Subnet            | `192.168.50.0/24` |
| DHCP              | Disabled          |
| Internet access   | None              |

| Host          | Purpose     | IP Address      | Services running                                                              |
| ------------- | ----------- | --------------- | ------------------------------------------------------------------------------ |
| Kali Linux    | Attacker    | `192.168.50.10` | —                                                                              |
| Ubuntu Server | Lab target  | `192.168.50.20` | DNS (53), NFS (2049), SNMP (UDP/161), SMB (445), TFTP (UDP/69), FTP (TCP/21) |

> **No-clash note:** FTP runs on TCP/21, which does not overlap with any previously configured service. Its root directory (`/srv/ftp`) is separate from all other service paths.

---

## Installing vsftpd

vsftpd was pre-installed on `cyberscheme` during a bulk package install session:

```
sudo apt update
sudo apt install vsftpd -y
```

Confirming the service started:

```
sudo systemctl status vsftpd
```

---

## Creating the FTP Root and Placing a Sensitive File

The FTP root directory must be created and assigned to the `ftp` system user. vsftpd enforces a strict chroot constraint: the anonymous root itself must **not** be writable by the FTP process — if it is, vsftpd refuses to start with a `500 OOPS: vsftpd: refusing to run with writable root inside chroot` error.

The directory structure therefore requires two components: a non-writable root (for files served to anonymous users) and a writable subdirectory (for uploads):

```
sudo mkdir -p /srv/ftp
sudo chown ftp:ftp /srv/ftp
sudo chmod 555 /srv/ftp
```

- `chown ftp:ftp` — the directory is owned by the `ftp` system account
- `chmod 555` — read and execute permissions for everyone; **not writable** (vsftpd chroot requirement)

A simulated sensitive backup file was placed in the root:

```
echo "ftp-backup sensitive data" | sudo tee /srv/ftp/backup.txt
```

A writable uploads subdirectory was created for the anonymous upload finding:

```
sudo mkdir /srv/ftp/uploads
sudo chown ftp:ftp /srv/ftp/uploads
sudo chmod 777 /srv/ftp/uploads
```

- `chmod 777` — the `uploads/` subdirectory is world-writable, allowing the FTP process to write files on behalf of anonymous users

---

## Default Configuration

The default configuration lives at `/etc/vsftpd.conf`. Key default values relevant to this lab:

```
anonymous_enable=NO
local_enable=YES
write_enable=YES
```

Out of the box, anonymous access is disabled. The service is effectively secure for anonymous access without any changes.

---

## Introducing the Vulnerability: Enabling Anonymous Access and Uploads

I edited `/etc/vsftpd.conf` and changed the following lines:

```
anonymous_enable=YES
write_enable=YES
anon_upload_enable=YES
anon_mkdir_write_enable=YES
```

- `anonymous_enable=YES` — **the key change**; enables login with username `anonymous` and any password
- `write_enable=YES` — required globally for any write operation; without this, `anon_upload_enable` has no effect
- `anon_upload_enable=YES` — permits anonymous users to upload files (into directories with write permission at the filesystem level)
- `anon_mkdir_write_enable=YES` — permits anonymous users to create subdirectories

Applied the change:

```
sudo systemctl restart vsftpd
sudo systemctl status vsftpd
```

The service restarted cleanly with anonymous access and uploads now enabled.

---

## Final State Summary

| Configuration item          | Value                                               |
| --------------------------- | --------------------------------------------------- |
| FTP root directory          | `/srv/ftp`                                          |
| Root directory permissions  | `chmod 555` (non-writable — vsftpd requirement)     |
| Uploads directory           | `/srv/ftp/uploads`                                  |
| Uploads dir permissions     | `chmod 777` (writable by FTP process)               |
| Anonymous access            | `anonymous_enable=YES`                              |
| Anonymous uploads           | `anon_upload_enable=YES`                            |
| Write enabled globally      | `write_enable=YES`                                  |
| Sensitive file present      | `backup.txt` (simulated backup file in FTP root)    |
| Authentication required     | None for anonymous access                           |

---

## What I Learned

- vsftpd's default state is secure for anonymous access — `anonymous_enable=NO` is the default. The finding here requires an explicit change, which means it was put there deliberately or left over from a configuration that was never cleaned up.
- The chroot constraint (`500 OOPS` error on writable root) is a common point of confusion when first setting up vsftpd with upload support. The fix — non-writable root with a writable subdirectory — is the correct pattern.
- `write_enable=YES` and `anon_upload_enable=YES` are independent settings that must both be present for anonymous uploads to work. Either alone is insufficient.
- The FTP banner discloses the vsftpd version to any connecting client — `220 (vsFTPd 3.0.5)`. This is useful information for an attacker matching the version against known CVEs.
- The combination of `anonymous_enable=YES` and `anon_upload_enable=YES` creates both a read finding and a write finding from a single service.

---

## Next: Assessing the Misconfiguration

The next document approaches the service from the attacker's perspective: discovering FTP via Nmap, confirming anonymous access using the `ftp-anon` NSE script, connecting anonymously, downloading the sensitive file, and uploading an arbitrary file — demonstrating the full impact of the configuration built here.

**Next:** `03-anonymous-ftp-access-exploitation.md`

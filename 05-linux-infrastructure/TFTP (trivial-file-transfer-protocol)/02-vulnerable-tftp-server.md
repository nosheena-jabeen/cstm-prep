# Building a Vulnerable TFTP Server

## Overview

As part of my penetration testing home lab, I installed and configured `tftpd-hpa` on Ubuntu, placed a simulated sensitive configuration file in the TFTP root directory, and deliberately enabled file uploads — introducing the two key misconfigurations that make TFTP a finding in real assessments: accessible sensitive files and unauthenticated write access.

Unlike services such as SMB or FTP, TFTP has no secure-by-default state when it comes to authentication — the protocol simply has none. The meaningful configuration choices are whether the service is network-accessible at all, what is placed in its root directory, and whether uploads are permitted.

The environment reused the same Kali Linux attacker machine and Ubuntu Server (`cyberscheme`) on the isolated VMware host-only network from earlier labs.

> **Important:** The TFTP configuration documented here was created deliberately for training. The service is isolated from my normal network and is not exposed to the Internet.

---

## Lab Architecture

| Component        | Configuration     |
| ----------------- | ----------------- |
| Virtual network   | VMnet2            |
| Network type      | Host-only         |
| Subnet            | `192.168.50.0/24` |
| DHCP              | Disabled          |
| Internet access   | None              |

| Host          | Purpose     | IP Address      | Services running                                              |
| ------------- | ----------- | --------------- | -------------------------------------------------------------- |
| Kali Linux    | Attacker    | `192.168.50.10` | —                                                              |
| Ubuntu Server | Lab target  | `192.168.50.20` | DNS (53), NFS (2049), SNMP (UDP/161), SMB (TCP/445), TFTP (UDP/69) |

> **No-clash note:** TFTP runs on UDP/69, which does not overlap with any previously configured service. Its root directory (`/srv/tftp`) is distinct from all other service paths. All services run simultaneously without interaction.

---

## Installing tftpd-hpa

`tftpd-hpa` required internet access to install. `cyberscheme` was temporarily switched to VMware NAT (VMnet8) for installation, then returned to the isolated VMnet2 lab network before any lab work continued.

On `cyberscheme`:

```
sudo apt update
sudo apt install tftpd-hpa tftp-hpa -y
```

- `tftpd-hpa` — the TFTP server daemon
- `tftp-hpa` — the TFTP client, useful for local testing from the server side

---

## Default Configuration

The default configuration lives at `/etc/default/tftpd-hpa`:

```
# /etc/default/tftpd-hpa

TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/srv/tftp"
TFTP_ADDRESS=":69"
TFTP_OPTIONS="--secure"
```

| Option                      | Meaning                                                                 |
| ---------------------------- | ------------------------------------------------------------------------- |
| `TFTP_USERNAME="tftp"`       | The server runs as the unprivileged `tftp` system account                 |
| `TFTP_DIRECTORY="/srv/tftp"` | Files are served from this directory                                      |
| `TFTP_ADDRESS=":69"`         | Listens on UDP/69 on all interfaces                                       |
| `TFTP_OPTIONS="--secure"`    | Chroots the server to `TFTP_DIRECTORY` — clients cannot traverse outside  |

Notably absent from the defaults: `--create`. Without it, the server is read-only — clients can download files that exist in the root directory, but cannot upload new ones.

---

## Creating the TFTP Root and Placing a Sensitive File

```
sudo mkdir -p /srv/tftp
sudo chmod 777 /srv/tftp
```

- `/srv/tftp` — dedicated TFTP root, consistent with the naming convention used for other services in this lab (`/srv/nfs/`, `/srv/smb/`)
- `chmod 777` — allows the `tftp` user to read and write files within the directory

A simulated sensitive configuration file was placed in the root to represent the type of content commonly found on real TFTP servers — network device configuration backups:

```
echo "router-backup confidential config" | sudo tee /srv/tftp/config.txt
```

In a real environment this might be a router or switch configuration file containing interface definitions, routing tables, SNMP community strings, or even local credentials. The file exists here as a clearly labelled simulation.

---

## Introducing the Vulnerability: Enabling Uploads

I edited `/etc/default/tftpd-hpa` and added `--create` to the options:

```
TFTP_OPTIONS="--secure --create"
```

`--create` allows TFTP clients to upload (write) new files to the server's root directory. This is disabled by default because there is no authentication to control who can write — enabling it means any client that can reach UDP/69 can write arbitrary files to the server.

Applied the change:

```
sudo systemctl restart tftpd-hpa
sudo systemctl status tftpd-hpa
```

The service restarted cleanly with uploads now permitted.

---

## Final State Summary

| Configuration item       | Value                                   |
| ------------------------ | --------------------------------------- |
| TFTP root directory      | `/srv/tftp`                             |
| Directory permissions    | `chmod 777`                             |
| Listening address        | `0.0.0.0:69` (all interfaces)           |
| Upload permitted         | Yes (`--create`)                        |
| Authentication           | None (by protocol design)               |
| Sensitive file present   | `config.txt` (simulated device config)  |

---

## What I Learned

- TFTP's vulnerability is the protocol itself — there is no authentication model to misconfigure. The relevant choices are network exposure, directory contents, and whether uploads are enabled.
- `--create` is the one configuration option that meaningfully changes the risk profile: without it, an attacker can only read files they already know exist; with it, they can also write.
- The `--secure` flag (chroot) limits traversal to the TFTP root — without it, a client could potentially reference paths outside the intended directory. `--secure` should always be present, but it does nothing to address the absence of authentication.
- `chmod 777` on the TFTP root is required for uploads to work at the filesystem level — the `--create` flag in tftpd-hpa and the filesystem permissions must both permit writes for an upload to succeed.
- TFTP persisting in a network environment long after its original purpose is a common real-world finding — a directory full of old router config backups with no authentication protecting them is exactly the kind of thing that appears in enterprise assessments.

---

## Next: Assessing the Misconfiguration

The next document approaches the service from the attacker's perspective: discovering TFTP via UDP scan, downloading the sensitive configuration file without credentials, and uploading an arbitrary file to the server — demonstrating the full impact of an exposed, upload-enabled TFTP service.

**Next:** `03-unauthenticated-tftp-file-access-exploitation.md`

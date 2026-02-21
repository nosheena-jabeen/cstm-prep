# Older Technology – Technical Interview Questions

---

## 1. Name some security models and what is the purpose of security models?

Security models are formal frameworks used to define how access control and information security policies should be enforced.

Examples include:

- **Bell-LaPadula Model** – Focuses on confidentiality. Enforces “no read up, no write down.”
- **Biba Model** – Focuses on integrity. Enforces “no read down, no write up.”
- **Clark-Wilson Model** – Focuses on data integrity through well-formed transactions.
- **Brewer-Nash (Chinese Wall) Model** – Prevents conflicts of interest.

The purpose of security models is to provide structured rules that guide how systems should enforce confidentiality, integrity, and access control.

---

## 2. What are the issues with the Wired Equivalent Privacy (WEP) protocol?

WEP was an early Wi-Fi security protocol designed to provide encryption equivalent to wired networks.

However, it has significant weaknesses:

- Uses RC4 with weak implementation
- Uses a short 24-bit Initialization Vector (IV)
- Reuses IVs frequently
- Vulnerable to statistical key recovery attacks

Attackers can capture enough traffic and recover the encryption key within minutes.

WEP is considered obsolete and insecure.

---

## 3. What are the security issues associated with Lanman (LM) passwords?

Lanman (LM) hashing is an outdated Windows password hashing method.

Weaknesses include:

- Passwords converted to uppercase
- Split into two 7-character chunks
- No salting
- Weak DES-based hashing

This makes LM hashes highly vulnerable to brute-force and rainbow table attacks.

Modern systems disable LM hashing by default.

---

## 4. Explain the purpose of the phpMyAdmin application

phpMyAdmin is a web-based interface for managing MySQL and MariaDB databases.

It allows administrators to:

- Create and modify databases
- Run SQL queries
- Manage users
- Import/export data

Security risks arise when:
- Exposed publicly
- Weak credentials are used
- Default installations remain accessible

If compromised, attackers may gain full database access.

---

## 5. How would you find and download files from an FTP or TFTP server?

For FTP:

1. Connect using an FTP client.
2. Attempt anonymous login.
3. Use commands such as:
   - `ls` (list files)
   - `get filename` (download file)

For TFTP:

TFTP is simpler and often does not require authentication.

Files can be downloaded using:

```
tftp <server-ip>
get filename
```

Security concerns:
- FTP transmits credentials in plaintext
- TFTP lacks authentication and encryption

---

## 6. What enumeration can SMTP be used for?

SMTP can be used for user enumeration.

Using commands like:

- `VRFY`
- `EXPN`
- `RCPT TO`

An attacker can determine whether email addresses exist on the server.

This can assist in phishing campaigns or password attacks.

Many modern servers disable these commands.

---

## 7. What is steganography?

Steganography is the practice of hiding data within other files, such as:

- Images
- Audio files
- Video files

Unlike encryption, which hides the content of data, steganography hides the existence of data.

It can be used for:
- Covert communication
- Malware payload concealment
- Data exfiltration

---

## 8. What is a Relative Identifier (RID) Master?

In Active Directory, the RID Master is one of the FSMO (Flexible Single Master Operations) roles.

It is responsible for:

- Allocating RID pools to domain controllers
- Ensuring unique Security Identifiers (SIDs)

Each account in a domain has:
- Domain SID
- Unique RID

The RID Master ensures no duplicate SIDs are created.

---

## 9. Explain hashing functions such as MD5 and SHA1

Hashing functions convert input data into a fixed-length output (hash).

Properties:
- One-way
- Deterministic
- Fixed size output

MD5:
- 128-bit output
- Vulnerable to collisions
- No longer secure

SHA1:
- 160-bit output
- Also vulnerable to collision attacks

Modern systems use SHA-256 or stronger algorithms.

---

## 10. Explain with examples some protocols used for Wide Area Networks (WAN)

WAN protocols enable communication across large geographical distances.

Examples:

- PPP (Point-to-Point Protocol)
- MPLS (Multiprotocol Label Switching)
- Frame Relay (legacy)
- HDLC (High-Level Data Link Control)

These protocols facilitate routing and communication between remote networks.

---

## 11. What is a One-Time Pad?

A One-Time Pad is an encryption method where:

- A random key is used
- The key is the same length as the message
- The key is used only once

If implemented correctly, it provides perfect secrecy.

However, it is impractical due to:
- Key distribution challenges
- Key storage requirements

---

## 12. What is a Schema Master?

The Schema Master is another FSMO role in Active Directory.

It controls modifications to the AD schema.

The schema defines:
- Object types
- Attributes
- Data structures

Only one Schema Master exists per forest.

Schema changes affect the entire Active Directory forest.

---

## 13. Explain null sessions and restrict anonymous settings

A null session allows unauthenticated connections to a Windows system via SMB.

Attackers could use null sessions to:
- Enumerate users
- Enumerate shares
- Gather domain information

“Restrict anonymous” settings were introduced to prevent this by limiting unauthenticated enumeration.

Modern Windows systems disable null sessions by default.

---

## 14. What vulnerability might be present if SMB version 1 was enabled?

SMBv1 is outdated and insecure.

Major vulnerabilities include:

- EternalBlue exploit
- Lack of encryption
- Weak authentication

SMBv1 was exploited in the WannaCry ransomware attack.

It should be disabled on modern systems.

# Current Technology – Technical Interview Questions

---

## 1. What is Root Squashing?

Root squashing is a security feature used in Network File System (NFS). It prevents remote root users from having root-level privileges on an NFS-mounted share.

When root squashing is enabled, if a remote user connects as root, the NFS server maps that user to a less privileged account (typically `nobody`).

Without root squashing, a remote root user could:
- Modify system files
- Change ownership of sensitive files
- Potentially escalate privileges on the NFS server

Root squashing helps prevent privilege abuse in shared file systems.

---

## 2. Explain how run0 and polkit can be used as a more secure alternative to sudo

`sudo` allows users to execute commands as root based on predefined permissions in `/etc/sudoers`.

`run0` is a newer utility designed to reduce the attack surface associated with `sudo`, particularly by avoiding complex environment inheritance issues.

`polkit` (PolicyKit) is a framework used for defining fine-grained authorisation policies. Instead of granting broad root privileges, polkit allows applications to request specific elevated actions, which are evaluated against policies.

Advantages over traditional sudo:
- More granular permissions
- Better integration with modern Linux systems
- Reduced risk of privilege escalation through misconfigured sudo rules
- Improved auditing and policy enforcement

---

## 3. What is DMARC?

DMARC (Domain-based Message Authentication, Reporting, and Conformance) is an email authentication protocol that builds on SPF and DKIM.

It allows domain owners to:
- Define how receiving mail servers should handle failed SPF/DKIM checks
- Receive reports on email authentication results

DMARC policies:
- `none` – Monitor only
- `quarantine` – Mark suspicious emails
- `reject` – Reject failed messages

DMARC helps reduce spoofing and phishing attacks.

---

## 4. What is the purpose of SPF?

Sender Policy Framework (SPF) allows domain owners to specify which mail servers are authorised to send email on their behalf.

It works by publishing a DNS TXT record listing permitted IP addresses.

When receiving mail, the recipient server checks:
- Whether the sender IP matches the SPF record

SPF helps prevent email spoofing but does not guarantee message integrity.

---

## 5. What is the purpose of DKIM?

DomainKeys Identified Mail (DKIM) provides cryptographic verification of email authenticity.

It works by:
- Signing outgoing emails with a private key
- Publishing the corresponding public key in DNS

The receiving server verifies the signature using the public key.

DKIM ensures:
- Message integrity
- Sender authenticity

Unlike SPF, DKIM protects message content from tampering.

---

## 6. What are the security issues with FTP?

FTP transmits data in plaintext, including:
- Usernames
- Passwords
- File contents

This makes it vulnerable to:
- Credential interception
- Man-in-the-Middle attacks
- Session hijacking

Additionally:
- Anonymous login can expose sensitive files
- Active/passive modes complicate firewall rules

Secure alternatives include:
- SFTP (SSH File Transfer Protocol)
- FTPS (FTP over TLS)

---

## 7. What is Kerberos?

Kerberos is a network authentication protocol that uses tickets and symmetric cryptography to securely authenticate users.

It relies on:
- Key Distribution Center (KDC)
- Ticket Granting Ticket (TGT)
- Service tickets

Instead of transmitting passwords over the network, Kerberos issues time-limited tickets.

Security concerns include:
- Pass-the-ticket attacks
- Kerberoasting
- Golden Ticket attacks

---

## 8. Explain directory traversal exploitation

Directory traversal occurs when an application allows attackers to access files outside the intended directory.

Example:
```
../../etc/passwd
```

This happens when user input is not properly validated.

Attackers can:
- Access sensitive configuration files
- Read application source code
- Extract credentials

Mitigation includes:
- Input validation
- Path canonicalisation
- Limiting file system access

---

## 9. What are the four types of API?

Common API types include:

1. Open APIs (Public APIs)
2. Partner APIs
3. Internal (Private) APIs
4. Composite APIs

From a security standpoint:
- Poor authentication
- Broken object-level authorisation
- Excessive data exposure
- Injection vulnerabilities

are common API risks.

---

## 10. How would you fingerprint a database server?

Database fingerprinting involves identifying:

- Service version
- Banner information
- Error messages
- Response behaviour

Tools:
- Nmap service detection
- Manual queries
- Error-based enumeration

Fingerprinting helps determine:
- Exploitable vulnerabilities
- Default credentials
- Configuration weaknesses

---

## 11. Explain symmetrical and asymmetrical cryptography

Symmetric cryptography:
- Uses the same key for encryption and decryption
- Faster
- Used for bulk encryption

Asymmetric cryptography:
- Uses a public/private key pair
- Slower
- Used for key exchange and authentication

Most modern systems combine both:
- Asymmetric for key exchange
- Symmetric for data encryption

---

## 12. Difference between SQL and NoSQL database vulnerabilities

SQL vulnerabilities often involve:
- Injection attacks
- Improper query sanitisation

NoSQL vulnerabilities may involve:
- JSON injection
- Insecure deserialisation
- Weak schema enforcement

SQL is structured and relational.
NoSQL is flexible and often document-based.

---

## 13. What is OVAL?

Open Vulnerability and Assessment Language (OVAL) is a standard used to describe system configuration information and vulnerabilities.

It allows:
- Automated vulnerability assessment
- Standardised security checks
- Integration into compliance tools

Used in enterprise vulnerability management systems.

---

## 14. Difference between Phishing, Smishing and Vishing

Phishing:
- Email-based deception

Smishing:
- SMS-based phishing

Vishing:
- Voice-based social engineering

All aim to:
- Steal credentials
- Install malware
- Obtain sensitive information

---

## 15. Tools to enumerate DNS for zone transfers

Common tools:
- `dig`
- `host`
- `nslookup`
- `dnsenum`
- `fierce`

Zone transfer attempts use:
```
dig axfr example.com @nameserver
```

Misconfigured DNS can expose all domain records.

---

## 16. What is a Security Identifier (SID)?

A SID uniquely identifies users and groups in Windows.

Format example:
```
S-1-5-21-...
```

Parts include:
- Revision
- Authority identifier
- Domain identifier
- Relative Identifier (RID)

RIDs identify specific accounts (e.g., 500 = Administrator).

---

## 17. Two IPsec encoding methods

IPsec uses:

1. Authentication Header (AH)
2. Encapsulating Security Payload (ESP)

AH:
- Provides integrity and authentication
- Does not encrypt data

ESP:
- Provides encryption, integrity, authentication
- More widely used

---

## 18. In key signing do you use the private or public key?

You sign with the private key.

Verification is done using the public key.

---

## 19. What ports must be open on a Domain Controller?

Common required ports:

- 53 (DNS)
- 88 (Kerberos)
- 389 (LDAP)
- 445 (SMB)
- 135 (RPC)
- 3268 (Global Catalog)

These support authentication and directory services.

---

## 20. Explain SNMP and weaknesses

SNMP manages network devices.

Versions:
- SNMPv1
- SNMPv2
- SNMPv3

Weaknesses:
- v1 and v2 use plaintext community strings
- Default "public" community strings
- Information leakage

SNMPv3 provides encryption and authentication.

---

## 21. Trade-offs between symmetrical and asymmetrical cryptography

Symmetric:
- Fast
- Requires secure key exchange

Asymmetric:
- Slower
- Enables secure key distribution

Hybrid encryption combines both.

---

## 22. Vulnerabilities in SSL and TLS

Examples:
- POODLE
- BEAST
- Heartbleed
- Downgrade attacks
- Weak cipher suites

Mitigation:
- Disable SSL
- Enforce TLS 1.2+
- Strong ciphers

---

## 23. Purpose of RADIUS

RADIUS provides centralised authentication, authorisation, and accounting (AAA).

Commonly used for:
- VPN authentication
- Wi-Fi authentication

Runs on UDP port 1812.

---

## 24. Where are passwords stored on servers?

Linux:
```
/etc/shadow
```

Windows:
- Security Account Manager (SAM)
- Active Directory database (NTDS.dit)

Passwords are stored as hashes, not plaintext.

---

## 25. Explain SQL Injection (SQLi)

SQL injection occurs when user input is unsafely incorporated into database queries.

Example:
```
' OR 1=1 --
```

This can allow:
- Authentication bypass
- Data extraction
- Data modification

Prevention:
- Parameterised queries
- Prepared statements
- Input validation
- Least privilege database accounts

---

## 26. If password hashes cannot be cracked can the hashes be used in any way and how would you do this?

Yes. Even if a password hash cannot be cracked into its plaintext form, it can sometimes still be used in attacks such as Pass-the-Hash (PtH).

In Windows environments, authentication mechanisms like NTLM allow a user to authenticate by presenting the hash itself instead of the plaintext password. If an attacker obtains an NTLM hash, they may be able to:

- Authenticate to other systems
- Move laterally within a network
- Access remote services (e.g., SMB, WMI)

This works because the authentication protocol validates the hash rather than requiring the actual password.

This makes hash protection critical. Defensive measures include:
- Restricting NTLM usage
- Enforcing Kerberos
- Using Credential Guard
- Limiting lateral movement privileges

---

## 27. Explain how a Cross-Site Request Forgery (CSRF) exploit works

Cross-Site Request Forgery (CSRF) occurs when a victim is tricked into performing an unwanted action on a web application where they are already authenticated.

A CSRF attack works as follows:

1. The victim logs into a legitimate site.
2. The attacker sends the victim a malicious link or embeds malicious code in a webpage.
3. When the victim visits the malicious page, their browser automatically sends an authenticated request to the legitimate site.
4. The action is executed because the server trusts the session cookie.

The key issue is that the application relies only on session cookies for authentication and does not verify the origin of the request.

Mitigations include:
- CSRF tokens
- SameSite cookie attributes
- Referer header validation
- Re-authentication for sensitive actions

---

## 28. Explain session hijacking (with respect to web application testing)

Session hijacking occurs when an attacker gains access to a valid session token and uses it to impersonate a legitimate user.

Web applications use session cookies to maintain state. If an attacker can obtain the session ID via:

- XSS
- Network sniffing (if HTTP is used)
- Malware
- Predictable session IDs

They can replay that session token in their own browser and gain access to the victim’s account.

Mitigations:
- HTTPS enforcement
- HttpOnly cookies
- Secure cookie flags
- Session rotation
- Short session lifetimes

---

## 29. What is Internet Information Services (IIS)?

Internet Information Services (IIS) is a web server developed by Microsoft for hosting web applications and websites on Windows Server.

It supports:
- ASP.NET
- Static websites
- FTP services
- Application pools

From a security perspective, IIS misconfigurations can lead to:
- Directory traversal
- Misconfigured authentication
- Exposure of sensitive files
- Web.config leaks

---

## 30. Explain Baseboard Management Controller (BMC) hacking and IPMI

A Baseboard Management Controller (BMC) is a microcontroller embedded in server hardware that allows remote management, even when the system is powered off.

IPMI (Intelligent Platform Management Interface) is the protocol used to communicate with BMCs.

Security risks include:
- Default credentials
- Weak authentication
- Remote console access
- Full system control

If compromised, attackers can:
- Reboot servers
- Access virtual consoles
- Modify firmware
- Extract system memory

BMC interfaces should never be exposed to the internet.

---

## 31. What are the pros and cons of using the cloud vs on-prem (in terms of cyber security)?

Cloud Pros:
- Built-in redundancy
- Managed security controls
- Scalability
- Rapid patching

Cloud Cons:
- Shared responsibility model confusion
- Misconfiguration risks
- Data sovereignty concerns

On-Prem Pros:
- Full control
- Direct physical security management

On-Prem Cons:
- Maintenance overhead
- Slower patch cycles
- Higher hardware costs

Security in both environments depends heavily on configuration and management practices.

---

## 32. What technical controls are available in the cloud to help with cyber security?

Common cloud security controls include:

- Identity and Access Management (IAM)
- Multi-factor authentication
- Network security groups
- Security monitoring (e.g., SIEM integration)
- Encryption at rest and in transit
- Web Application Firewalls (WAF)
- Logging and audit trails
- Key management services

These controls help enforce least privilege and improve visibility.

---

## 33. What are the most popular containerization solutions?

Popular containerization platforms include:

- Docker
- Kubernetes
- Podman
- OpenShift

Docker provides container packaging.
Kubernetes orchestrates and manages container deployments at scale.

---

## 34. What’s the difference between virtual machines and containerization?

Virtual Machines (VMs):
- Emulate full hardware
- Run separate operating systems
- Higher resource usage

Containers:
- Share host OS kernel
- Lightweight
- Faster startup
- Less isolation than VMs

Containers are more efficient, but VMs provide stronger isolation boundaries.

---

## 35. What are the signs a mobile device has been jailbroken?

Indicators include:

- Presence of unauthorised app stores
- Root access availability
- Disabled sandboxing
- Modified system files
- Suspicious background processes

Jailbreaking removes manufacturer restrictions and weakens security controls.

---

## 36. What file types are associated with Android apps?

Android applications use:

```
.apk
```

APK files contain:
- Compiled code
- Resources
- Manifest files
- Certificates

---

## 37. What file types are associated with Apple apps?

Apple iOS applications use:

```
.ipa
```

IPA files are packaged app archives containing:
- Executable binaries
- Assets
- Metadata

---

## 38. Explain the use case for Datagram Transport Layer Security (DTLS)

DTLS provides encryption for UDP-based communications.

Since TLS works over TCP, DTLS was developed for:

- VoIP
- Video streaming
- Real-time applications

DTLS provides:
- Confidentiality
- Integrity
- Authentication

While preserving UDP's low latency.

---

## 39. What are the built-in user accounts on a Windows server?

Common built-in accounts include:

- Administrator
- Guest
- DefaultAccount
- SYSTEM
- LocalService
- NetworkService

These accounts have predefined roles and varying privilege levels.

---

## 40. Name some protocols in the ICS / OT space

Industrial Control Systems (ICS) and Operational Technology (OT) commonly use:

- Modbus
- DNP3
- OPC
- BACnet
- PROFINET

These protocols often lack encryption and authentication, making them vulnerable to interception and manipulation.

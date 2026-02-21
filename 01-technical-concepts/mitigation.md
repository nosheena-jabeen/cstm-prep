# Mitigation – Technical Interview Questions

---

## 1. How do you mitigate Server Message Block (SMB) issues?

Mitigating SMB-related risks involves reducing exposure, hardening configurations, and enforcing modern security standards.

Key mitigations include:

- Disable SMBv1 (legacy and vulnerable to exploits like EternalBlue).
- Enforce SMB signing to prevent MITM attacks.
- Restrict SMB access to internal networks only.
- Use firewall rules to block TCP 445 from external networks.
- Implement least privilege on file shares.
- Enable auditing to monitor access attempts.
- Keep systems patched.

In enterprise environments, segmentation and limiting lateral movement are critical to preventing SMB-based attacks.

---

## 2. How do you mitigate an external-facing RDP server with an expired TLS certificate?

An expired TLS certificate weakens encrypted communication and may expose users to MITM attacks.

Mitigation steps include:

- Immediately renew and replace the TLS certificate with a valid one from a trusted CA.
- Configure RDP to enforce Network Level Authentication (NLA).
- Restrict RDP exposure using:
  - VPN access instead of direct exposure
  - IP allowlisting
  - Bastion hosts or jump servers
- Enable MFA for remote access.
- Monitor RDP logs for brute-force attempts.

Best practice: RDP should not be directly exposed to the internet.

---

## 3. How do you mitigate Machine-in-the-Middle (MITM) attacks?

Mitigation depends on the attack vector but generally includes:

- Enforce HTTPS with valid TLS certificates.
- Use certificate pinning where appropriate.
- Enable HSTS (HTTP Strict Transport Security).
- Use secure Wi-Fi protocols (WPA3).
- Enforce SMB signing.
- Implement mutual authentication protocols.
- Deploy network segmentation and intrusion detection systems.

On local networks, dynamic ARP inspection can help prevent ARP poisoning.

---

## 4. How do you mitigate SQL Injection (SQLi)?

Primary mitigations include:

- Use parameterised queries or prepared statements.
- Avoid dynamic SQL concatenation.
- Implement strict input validation.
- Apply least privilege to database accounts.
- Use stored procedures safely.
- Implement Web Application Firewalls (WAFs).
- Conduct regular code reviews and testing.

The most effective control is proper input handling and query parameterisation.

---

## 5. How do you mitigate socially engineered staff?

Mitigation requires both technical and human controls.

Technical controls:
- Email filtering and anti-phishing tools
- MFA for critical systems
- Access controls and least privilege

Human controls:
- Regular security awareness training
- Phishing simulations
- Clear reporting procedures
- Verification policies (e.g., call-back verification for financial requests)

Security culture is critical in reducing social engineering risk.

---

## 6. How do you mitigate a Cross-Site Scripting (XSS) vulnerability?

Mitigation includes:

- Proper output encoding (HTML, JavaScript, URL encoding).
- Input validation and sanitisation.
- Implement Content Security Policy (CSP).
- Use HttpOnly and Secure cookie flags.
- Avoid directly rendering user input in HTML.

Frameworks that automatically escape output reduce XSS risk significantly.

---

## 7. What is the mitigation for a ransomware attack where files have been locked?

If files are already encrypted:

- Isolate infected systems immediately.
- Do not power off systems unless necessary.
- Identify ransomware strain.
- Restore from secure, offline backups.
- Report incident to relevant authorities.
- Conduct forensic investigation to identify entry point.

Preventative measures include:
- Regular offline backups
- Patch management
- Endpoint detection and response (EDR)
- Network segmentation
- MFA enforcement

Paying ransom is generally discouraged and does not guarantee file recovery.

---

# Scope / Laws / Risk – Technical Interview Questions

---

## 1. What should be in a scope document for a security test?

A scope document defines what is authorised and what is prohibited during a security engagement.

It should include:

- Client details and authorised contacts
- Systems, IP ranges, and domains in scope
- Explicit exclusions (out-of-scope systems)
- Type of testing permitted (external, internal, web, social engineering)
- Testing window and time restrictions
- Allowed techniques (e.g., no DoS, no password spraying)
- Data handling and evidence procedures
- Emergency contact and escalation process
- Legal authorisation statement

A clear scope protects both the tester and the client legally and operationally.

---

## 2. What laws affect security testing?

In the UK, key laws include:

- Computer Misuse Act 1990
- Data Protection Act 2018
- GDPR
- Human Rights Act 1998
- Regulation of Investigatory Powers Act 2000 (RIPA)
- Police and Justice Act 2006

Security testing without explicit authorisation can be considered unauthorised access under the Computer Misuse Act.

---

## 3. What risk mitigation can you put in place before conducting a security test?

Risk mitigation includes:

- Clear scope and written authorisation
- Change management notification
- Backup verification
- Scheduling outside business hours
- Avoiding denial-of-service techniques unless approved
- Rate limiting scans
- Using test accounts instead of production accounts
- Ensuring logging is enabled for traceability

The aim is to reduce operational impact while achieving testing objectives.

---

## 4. What is CVSS and CVE?

CVE (Common Vulnerabilities and Exposures):
A unique identifier assigned to publicly known vulnerabilities.
Example:
CVE-2023-XXXXX

CVSS (Common Vulnerability Scoring System):
A standardised scoring system used to assess the severity of a vulnerability.

CVSS scores range from 0.0 to 10.0 and consider:
- Attack complexity
- Privileges required
- User interaction
- Impact (confidentiality, integrity, availability)

CVE identifies the vulnerability.
CVSS measures its severity.

---

## 5. What are the testing regulations regarding CHECK level testing?

CHECK is a UK government-backed scheme for accrediting penetration testers.

Key aspects:
- Requires security clearance
- Mandates adherence to government testing standards
- Specifies reporting formats
- Requires strict evidence handling procedures

CHECK testing is often required for government or critical infrastructure environments.

---

## 6. What part of the Human Rights Act 1998 affects pen testers?

Article 8 – Right to respect for private and family life.

Security testing may involve:
- Accessing personal data
- Reviewing communications
- Inspecting stored data

Pen testers must ensure:
- Proportionality
- Necessity
- Lawful authorisation

Testing should not unnecessarily invade privacy beyond agreed scope.

---

## 7. What is RIPA 2000 and does it affect security testing?

The Regulation of Investigatory Powers Act 2000 (RIPA) governs surveillance and interception of communications.

It affects:
- Monitoring communications
- Interception activities

If testing involves intercepting live communications, legal authorisation may be required.

In most standard penetration tests, RIPA does not directly apply, but caution is required in monitoring activities.

---

## 8. What is GDPR and how does it affect security testing?

GDPR regulates the processing of personal data.

During security testing:
- Personal data may be accessed
- Sensitive information may be discovered

Testers must:
- Minimise data exposure
- Securely store evidence
- Report data breaches appropriately
- Avoid unnecessary data extraction

GDPR requires lawful basis and proper handling of personal information.

---

## 9. Name and explain three security standards

1. ISO/IEC 27001
   - International standard for Information Security Management Systems (ISMS).
   - Focuses on risk management and control implementation.

2. NIST Cybersecurity Framework
   - US-based framework structured around:
     Identify, Protect, Detect, Respond, Recover.

3. PCI-DSS
   - Payment Card Industry Data Security Standard.
   - Applies to organisations handling credit card data.

Security standards provide structured control frameworks.

---

## 10. What are the risks of enumerating and assessing a multifunction printer?

Risks include:

- Service disruption
- Device crashes due to aggressive scanning
- Access to stored documents
- Exposure of LDAP credentials
- Disclosure of stored print jobs

Printers often run outdated firmware and may contain sensitive cached data.

---

## 11. What are the risks of enumerating and assessing an ICS device?

ICS/OT devices are often fragile.

Risks include:

- System outages
- Physical process disruption
- Safety hazards
- Legacy protocol instability
- Regulatory impact

Active scanning may cause unexpected device reboots.

Testing ICS environments requires extreme caution and often passive techniques.

---

## 12. What is the Police and Justice Act 2006 and how does it affect security testing?

The Police and Justice Act 2006 amended the Computer Misuse Act.

It criminalised:

- Making, supplying, or obtaining tools intended for computer misuse.

Security professionals must ensure:
- Tools are used lawfully
- Engagements are authorised
- Intent is clearly documented

Without authorisation, possession and use of certain tools could be questioned.

---

## 13. Explain how to keep to scope during a security test

Maintaining scope involves:

- Strict adherence to defined IP ranges and domains
- Avoiding pivoting into out-of-scope systems
- Logging all actions taken
- Confirming ambiguous assets before testing
- Escalating uncertainties to the client
- Regular communication with stakeholders

If unexpected access is gained to out-of-scope systems, testing should stop and the client must be notified.

Staying in scope protects legal standing and professional credibility.

---

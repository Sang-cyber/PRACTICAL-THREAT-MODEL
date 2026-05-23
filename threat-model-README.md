# Practical Threat Model Report
> STRIDE-based security assessment of a message board application across production and corporate network environments.

**Dataflow Diagram:** [threatmodelmessageapp.netlify.app]((https://practicalthreatmodel.netlify.app/))

---

## Executive Summary

A structured threat model for a message board web application operating across two distinct network environments — a production network exposed to the Internet, and a corporate network containing internal identity and administrative systems.

### Critical Findings

| Risk | Rating |
|---|---|
| Active Directory Compromise | CATASTROPHIC |
| Corp to Prod Network Pivot | CATASTROPHIC |
| SQL Injection | CRITICAL |
| Stored XSS to Session Hijack | CRITICAL |

### Immediate Priorities
- Parameterized queries across all database interactions
- Output encoding and Content Security Policy (CSP)
- MFA enforcement on all privileged accounts
- Strict firewall segmentation between corporate and production

---

## System Architecture

### Production Network
- **Frontend Application** — Internet-facing entry point for all user traffic
- **Backend Server 1** — Message board business logic processing
- **Backend Server 2** — Authentication and user session management
- **Message Board Database** — Stores user posts, content, and metadata
- **Corporate User Database** — Mirrors or references corporate identity

### Corporate Network
- **Active Directory (AD)** — Central identity authority
- **Employee Workstations** — Endpoints used by corporate staff and administrators
- **Internal Databases** — Supporting business data systems
- **Corp/Prod Firewall** — Security boundary between corporate and production zones

---

## Trust Boundaries

| Data Flow | Direction | Classification |
|---|---|---|
| Internet to Frontend | Inbound | Untrusted to Trusted (HIGH RISK) |
| Frontend to Backend Servers | Internal | Trusted to Privileged |
| Backend to Databases | Internal | Privileged to Critical Data |
| Corporate to Production | Cross-boundary | Corporate Trust to Prod (CRITICAL) |
| Employee Laptop to Active Directory | Internal | Corp User to Identity Authority |

---

## Protected Assets

| Asset | Sensitivity |
|---|---|
| User Credentials and Password Hashes | CRITICAL |
| Session Tokens and Auth Cookies | CRITICAL |
| Active Directory — Corporate Identity | CRITICAL |
| Database Records | CRITICAL |
| Administrative Access Privileges | CRITICAL |
| Message Content and Metadata | HIGH |
| Network Segmentation Integrity | HIGH |
| System Availability | MEDIUM |

---

## STRIDE Analysis Summary

### Frontend (Internet-Facing)
| STRIDE | Threat | Impact | Likelihood | Mitigation |
|---|---|---|---|---|
| S | Session cookie theft | Critical | High | MFA, HttpOnly cookies, session expiry |
| T | HTTP parameter tampering | High | High | Server-side validation, signed tokens |
| R | No audit trail for user actions | Medium | Medium | Immutable audit logs |
| I | IDOR exposes other users data | High | High | Authorization checks on all resources |
| D | HTTP flood / volumetric DDoS | High | High | WAF, CDN, rate limiting |
| E | Authentication bypass via logic flaw | Critical | Medium | Robust auth framework, pen testing |

### Backend Servers
| STRIDE | Threat | Impact | Likelihood | Mitigation |
|---|---|---|---|---|
| S | Fake backend service impersonation | High | Low | Mutual TLS between services |
| T | Internal API request tampering | High | Medium | Signed requests, schema validation |
| R | No backend service logs | High | Medium | Centralized logging with correlation IDs |
| I | Stack trace exposure in errors | Medium | High | Custom error pages, structured handling |
| D | Resource exhaustion via heavy queries | High | Medium | Query limits, circuit breakers |
| E | Privilege escalation to admin API | Critical | Medium | RBAC, least privilege |

### Databases
| STRIDE | Threat | Impact | Likelihood | Mitigation |
|---|---|---|---|---|
| S | Stolen DB credentials | Critical | Medium | Secret vault, credential rotation |
| T | SQL injection | Critical | High | Parameterized queries, ORM |
| R | No database audit trail | High | Medium | DB audit logging, SIEM |
| I | Full database dump / exfiltration | Critical | Medium | Encryption at rest, access controls |
| D | Query flood / DB overload | High | Medium | Connection pooling, rate limiting |
| E | App account gains DBA rights | Critical | Low | Separate read/write/admin accounts |

### Active Directory
| STRIDE | Threat | Impact | Likelihood | Mitigation |
|---|---|---|---|---|
| S | Kerberoasting / Pass-the-Hash | Critical | High | Kerberos hardening, Protected Users group |
| T | Rogue admin account creation | Critical | Medium | RBAC, AD change monitoring |
| R | Audit log deletion | High | Medium | SIEM with off-box log forwarding |
| I | NTDS.dit dump | Critical | Medium | AD tiering, LSASS protection, Credential Guard |
| D | Domain controller overload | High | Low | Redundant DCs, rate limiting |
| E | Standard account to Domain Admin | Critical | Medium | AD tiering, just-in-time access, PAM |

---

## Attack Scenarios (Kill Chains)

### Scenario 1 — Stored XSS to Session Hijack
```
Attacker posts malicious script
→ Victim loads page
→ Script exfiltrates session cookie
→ Attacker reuses cookie
→ Account takeover
Impact: CRITICAL
```

### Scenario 2 — SQL Injection to Database Dump
```
Attacker submits crafted input
→ Unsanitized query executed
→ Full database extracted including password hashes
→ Offline cracking
Impact: CRITICAL
```

### Scenario 3 — Phishing to Active Directory Compromise
```
Targeted phishing email
→ Employee clicks
→ Credentials harvested
→ AD authenticated
→ Domain Admin escalation
Impact: CATASTROPHIC
```

### Scenario 4 — Corp to Prod Lateral Movement
```
Compromised workstation
→ AD credentials stolen
→ Corp/Prod firewall traversal
→ Production systems accessed and controlled
Impact: CATASTROPHIC
```

---

## Risk Prioritization

| Threat | Impact | Likelihood | Priority | Rating |
|---|---|---|---|---|
| Active Directory Compromise | Critical | Medium | P1 — Immediate | CATASTROPHIC |
| Corp to Prod Network Pivot | Critical | Medium | P1 — Immediate | CATASTROPHIC |
| SQL Injection | Critical | High | P1 — Immediate | CRITICAL |
| Stored XSS | High | High | P1 — Immediate | CRITICAL |
| Credential Stuffing | High | High | P2 — Short-term | HIGH |
| Privilege Escalation (App) | Critical | Medium | P2 — Short-term | HIGH |
| IDOR / Broken AuthZ | High | High | P2 — Short-term | HIGH |
| DDoS / Service Disruption | Medium | High | P3 — Planned | MEDIUM |
| Information Disclosure (Errors) | Medium | High | P3 — Planned | MEDIUM |

---

## Security Controls

| Threat | Primary Controls | Control Type |
|---|---|---|
| SQL Injection | Parameterized queries, ORM | Preventive |
| Stored XSS | Output encoding, CSP | Preventive |
| Credential Stuffing | MFA, rate limiting | Preventive + Detective |
| Session Hijacking | HttpOnly cookies, short TTL | Preventive |
| AD Compromise | MFA, PAW, AD tiering | Preventive + Detective |
| Corp to Prod Pivot | Strict firewall, micro-segmentation | Preventive + Detective |
| Privilege Escalation | RBAC, least privilege, PAM | Preventive + Detective |
| DB Exfiltration | Encryption at rest, data masking | Preventive + Detective |
| DDoS | CDN, WAF, upstream scrubbing | Preventive + Corrective |

---

## Recommended Firewall Policy (Corp/Prod Boundary)

| Rule | Source | Destination | Protocol | Port | Action |
|---|---|---|---|---|---|
| 1 | Internet | DMZ/Frontend | TCP | 443 | PERMIT |
| 2 | Internet | DMZ/Frontend | TCP | 80 | REDIRECT to HTTPS |
| 3 | DMZ/Frontend | Prod Backend | TCP | 8080, 8443 | PERMIT |
| 4 | Prod Backend | Prod DB | TCP | 5432/3306 | PERMIT |
| 5 | Corp Network | Prod (Mgmt) | TCP | 22 (SSH) | PERMIT via PAM |
| 6 | Corp Network | Active Directory | TCP/UDP | 88, 389, 636 | PERMIT |
| 7 | Internet | Prod Backend | Any | Any | DENY |
| 8 | Any | Any | Any | Any | DENY + LOG |

---

## Author

**Raj Sanghvi**
Security Analyst | GRC | AI Security Engineering
[GitHub](https://github.com/Sang-cyber)

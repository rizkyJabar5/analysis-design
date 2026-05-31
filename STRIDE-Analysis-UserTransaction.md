# STRIDE Analysis - User Transaction Process

## Process Overview

**Elements:**
1. User Login
2. Validation with JWT
3. Transaction Database

---

## 1. VALID STRIDE (Secure Design - Mitigated Threats)

### S - Spoofing

| Threat | Mitigation |
|--------|------------|
| Attacker impersonates legitimate user | Multi-factor authentication (MFA) required |
| Stolen credentials reused | Password hashing with bcrypt/scrypt + salt |
| Session hijacking | JWT with short expiration, secure cookie flags |
| Token theft | Token binding to device fingerprint |

**Secure Flow:**
```
User → Login (HTTPS + MFA) → JWT Validation (issuer/audience check) → Transaction DB (encrypted)
```

### T - Tampering

| Threat | Mitigation |
|--------|------------|
| JWT token modification | HMAC signature verification |
| Request body manipulation | Input validation + schema validation |
| Database record alteration | Database audit triggers + immutable logs |
| Man-in-the-middle modification | TLS 1.3 with certificate pinning |

**Controls:**
- JWT signed with RS256/ES256 (asymmetric)
- Request signing with API key
- Database integrity constraints

### R - Repudiation

| Threat | Mitigation |
|--------|------------|
| User denies transaction | Immutable audit log with timestamps |
| Admin denies data access | Audit trail with user attribution |
| Non-repudiation failure | Digital signatures on critical operations |

**Controls:**
- WORM (Write Once Read Many) audit logs
- Signed audit entries
- Centralized logging with PII redaction

### I - Information Disclosure

| Threat | Mitigation |
|--------|------------|
| Credentials exposed in transit | TLS 1.3 encryption |
| Sensitive data in logs | PII redaction in centralized logging |
| Database breach | TDE (Transparent Data Encryption) |
| Token leakage | Short-lived JWT, secure storage |

**Controls:**
- No secrets in code/config repos
- Encrypted backups
- Secure key management (KMS)

### D - Denial of Service

| Threat | Mitigation |
|--------|------------|
| Brute force login attempts | Rate limiting + account lockout |
| API endpoint flooding | WAF + DDoS mitigation |
| Database overload | Query timeouts + connection pooling |

**Controls:**
- Rate limiting at API gateway
- Circuit breaker patterns
- Auto-scaling infrastructure

### E - Elevation of Privilege

| Threat | Mitigation |
|--------|------------|
| Unauthorized data access | RBAC/ABAC authorization checks |
| Admin role escalation | Scope validation in JWT |
| SQL injection | Parameterized queries |

**Controls:**
- OPA (Open Policy Agent) enforcement
- Least privilege database users
- Input sanitization

---

## 2. INVALID STRIDE (Insecure Design - Vulnerable Threats)

### S - Spoofing (VULNERABLE)

| Threat | Impact |
|--------|--------|
| No MFA - password only | Account takeover with stolen password |
| Weak password policy | Brute force successful |
| No CAPTCHA | Automated credential stuffing |
| Session fixation | Session hijacking |

**Insecure Flow:**
```
User → Login (HTTP, no MFA) → JWT Validation (none) → Transaction DB (plaintext)
```

### T - Tampering (VULNERABLE)

| Threat | Impact |
|--------|--------|
| No JWT signature verification | Token forgery |
| No input validation | SQL injection, XSS |
| Client-trusted amounts | Transaction value manipulation |
| No request signing | API parameter tampering |

### R - Repudiation (VULNERABLE)

| Threat | Impact |
|--------|--------|
| No audit logging | Cannot trace malicious actions |
| Writable logs by app user | Log tampering possible |
| No transaction signing | Denial of transactions |

### I - Information Disclosure (VULNERABLE)

| Threat | Impact |
|--------|--------|
| Plaintext HTTP | Credential sniffing |
| Verbose error messages | Stack trace exposure |
| PAN/CVV stored plaintext | Database breach exposes card data |
| Secrets in git repo | Credential extraction |

### D - Denial of Service (VULNERABLE)

| Threat | Impact |
|--------|--------|
| No rate limiting | Brute force floods server |
| No WAF | DDoS successful |
| No circuit breaker | Cascading failures |

### E - Elevation of Privilege (VULNERABLE)

| Threat | Impact |
|--------|--------|
| No authorization check | Any user accesses any data |
| Shared DB admin user | Full database access |
| No scope validation | JWT claim manipulation |

---

## 3. Network-Level STRIDE Verification

### Trust Boundaries

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Client    │────▶│   Server    │────▶│  Database   │
│  (User)     │     │  (API)      │     │  (Storage)  │
└─────────────┘     └─────────────┘     └─────────────┘
   TB1: Internet      TB2: Internal      TB3: Data
```

### Network Threats per Boundary

| Boundary | STRIDE Category | Valid (Mitigated) | Invalid (Vulnerable) |
|----------|-----------------|-------------------|----------------------|
| TB1: Internet | Spoofing | MFA + Cert Pinning | Password only |
| TB1: Internet | Tampering | TLS 1.3 | HTTP plaintext |
| TB1: Internet | DoS | Rate Limit + WAF | No protection |
| TB2: Internal | Tampering | Request signing | No validation |
| TB2: Internal | Elevation | mTLS + RBAC | No auth check |
| TB3: Data | Info Disclosure | TDE + KMS | Plaintext storage |
| TB3: Data | Tampering | Audit triggers | Writable logs |

---

## 4. Summary Matrix

| STRIDE | Valid Design | Invalid Design |
|--------|--------------|----------------|
| **S**poofing | MFA, cert pinning | Password only |
| **T**ampering | TLS, JWT signing | HTTP, no validation |
| **R**epudiation | Immutable audit | No logging |
| **I**nfo Disclosure | Encryption, PII redaction | Plaintext, verbose errors |
| **D**enial of Service | Rate limit, WAF | No protection |
| **E**levation | RBAC, OPA, least privilege | No authorization |

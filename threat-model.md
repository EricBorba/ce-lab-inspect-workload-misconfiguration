# Threat Model: Simple Web Application

## Architecture
- Frontend: React app (S3 + CloudFront)
- Backend: Node.js API (ALB + EC2)
- Database: PostgreSQL (RDS)

## Assets
- Customer PII in database
- User session tokens
- API keys for third-party services

## STRIDE Analysis

### S - Spoofing
**Threat:** Attacker steals user session cookie
- **Mitigation:** HttpOnly cookies, short-lived tokens, MFA
- **Priority:** High

### T - Tampering
**Threat:** SQL injection modifies data
- **Mitigation:** Parameterized queries, input validation, WAF
- **Priority:** Critical

### R - Repudiation
**Threat:** User denies placing order
- **Mitigation:** CloudTrail logs, application audit trail
- **Priority:** Medium

### I - Information Disclosure
**Threat:** Public S3 bucket exposes data
- **Mitigation:** S3 Block Public Access, encryption
- **Priority:** Critical

### D - Denial of Service
**Threat:** DDoS attack on ALB
- **Mitigation:** AWS Shield, CloudFront, rate limiting
- **Priority:** Medium

### E - Elevation of Privilege
**Threat:** EC2 role has admin access
- **Mitigation:** Least privilege IAM, regular reviews
- **Priority:** Critical

# Security Standards

## Core Security Requirements

- **All inputs validated and sanitized** before processing
- **No secrets in code, configs, or logs** — use vault/secret management
- **All developers use 2FA** for all systems
- **Zero-trust between all services** — authenticate every request

## Implementation Requirements

- **Environment variables pass through vault**
- **Credentials auto-rotate** on schedule and after incidents
- **Containers run as non-root users** with minimal permissions
- **All APIs require authentication and authorization**

## HIPAA Compliance

When handling PHI (Protected Health Information):

- **Data Encryption**: TLS 1.3+ in transit, AES-256 at rest
- **Access Control**: Role-based permissions with principle of least privilege
- **Audit Logging**: Record all PHI access with who/what/when/why
- **Data Segmentation**: Isolate PHI in dedicated storage
- **Backup Strategy**: Regular testing of backup/restore procedures
- **BAAs**: Business Associate Agreements with all vendors
- **Incident Response**: Documented procedures for potential breaches

## PCI-DSS Requirements

When handling payment data:

- **Network Segmentation**: Isolate payment processing from other systems
- **Sensitive Data Handling**: Tokenization over storage when possible
- **Vulnerability Scanning**: Regular automated scanning of all components
- **Penetration Testing**: Annual third-party security assessment
- **Patch Management**: Clear process for security updates
- **Minimal Surface Area**: Remove unnecessary services and dependencies

## Enforcement

- Static analysis checks for leaked secrets
- Security scans on all dependencies
- Automated penetration testing
- Failed security audits block releases

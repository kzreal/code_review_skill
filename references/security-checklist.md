# Security Review Checklist

Cross-language security review checklist. Apply to all code regardless of language.

## Injection

### SQL Injection
- [ ] All SQL queries use parameterized statements / prepared statements / ORM
- [ ] No string concatenation, f-strings, or template literals in SQL queries
- [ ] User input is never directly embedded in query strings
- [ ] ORM raw query methods are reviewed for parameterization

### Command Injection
- [ ] No `Runtime.exec()`, `subprocess.run(shell=True)`, `child_process.exec()` with unsanitized input
- [ ] Use array-form arguments: `subprocess.run(["ls", user_input])`, `spawn("ls", [user_input])`
- [ ] Environment variables in commands are validated

### XSS (Cross-Site Scripting)
- [ ] No `dangerouslySetInnerHTML` (React) or `v-html` (Vue) with user content
- [ ] HTML sanitization (DOMPurify) for any user-generated HTML rendering
- [ ] Content Security Policy headers configured
- [ ] URL construction from user input is validated (javascript: protocol, etc.)

### SSRF (Server-Side Request Forgery)
- [ ] Server-side HTTP requests validate target URLs against allowlist
- [ ] No direct user input in URL construction for outgoing requests
- [ ] Internal service URLs are not exposed to user control

### Path Traversal
- [ ] File paths from user input are sanitized and validated
- [ ] `path.resolve()` / `Path.resolve()` / `os.path.realpath()` used to canonicalize paths
- [ ] Verify resolved path stays within expected directory

---

## Authentication & Authorization

### Authentication
- [ ] Passwords are hashed with strong algorithm (bcrypt, scrypt, argon2), never MD5/SHA
- [ ] JWT tokens: validate signature, expiry, issuer, audience on every request
- [ ] Session tokens are cryptographically random, sufficiently long
- [ ] Login rate limiting is in place
- [ ] Multi-factor authentication for sensitive operations

### Authorization
- [ ] Every endpoint checks authorization, not just authentication
- [ ] IDOR protection: resource ownership verified before access
- [ ] Role-based access control is enforced at the appropriate layer
- [ ] Admin endpoints are separately protected
- [ ] API keys have appropriate scope restrictions

### Session Management
- [ ] Sessions expire after reasonable inactivity period
- [ ] Session invalidation on password change
- [ ] Concurrent session limits for sensitive accounts
- [ ] Secure cookie flags: `HttpOnly`, `Secure`, `SameSite`

---

## Data Protection

### Sensitive Data at Rest
- [ ] Passwords and credentials are hashed, not encrypted or stored in plain text
- [ ] PII (personally identifiable information) is encrypted in database if required
- [ ] Database backups are encrypted
- [ ] File uploads are stored outside web root

### Sensitive Data in Transit
- [ ] HTTPS enforced for all connections
- [ ] TLS version 1.2+ minimum
- [ ] Certificate pinning for mobile clients (if applicable)
- [ ] No sensitive data in URL parameters (use POST body or headers)

### Logging
- [ ] No passwords, tokens, or API keys in log output
- [ ] No PII in logs without masking/hashing
- [ ] Log injection is prevented (sanitize log input)
- [ ] Audit logs for sensitive operations (login, data export, permission changes)

---

## Input Validation

### Server-Side
- [ ] All external input is validated at the server boundary
- [ ] Input validation fails closed (reject unexpected, don't try to fix)
- [ ] Type, length, range, and format are all checked
- [ ] Upload file types and sizes are validated
- [ ] Content-Type headers match actual content

### Client-Side
- [ ] Client validation is treated as UX enhancement, not security control
- [ ] Server re-validates everything the client sends

---

## Cryptography

### Common Issues
- [ ] No custom cryptographic algorithms — use established libraries
- [ ] AES-256 for symmetric encryption, RSA-2048+ or ECDSA for asymmetric
- [ ] Random numbers for security purposes use `java.security.SecureRandom`, `secrets` module, `crypto.randomBytes()`
- [ ] IVs/nonces are unique per encryption operation and never reused
- [ ] Password hashing uses dedicated algorithms (bcrypt, argon2), not generic hashes

### Key Management
- [ ] Encryption keys are rotated periodically
- [ ] Keys are stored in key management systems (KMS, Vault), not in code or config files
- [ ] Different keys for different environments (dev, staging, production)

---

## Dependency & Supply Chain

- [ ] Dependencies are pinned to specific versions with lockfiles
- [ ] Regular vulnerability scanning: `npm audit`, `pip-audit`, `safety`, OWASP Dependency-Check
- [ ] No unnecessary dependencies — each one is attack surface
- [ ] Build scripts (`postinstall`, etc.) are reviewed
- [ ] Container images use minimal base images and are scanned

---

## Concurrency Security

- [ ] Race conditions in financial / state-changing operations (TOCTOU)
- [ ] Database-level locking for critical updates: `SELECT ... FOR UPDATE`
- [ ] Idempotency for payment and order operations
- [ ] Distributed locks for multi-instance deployments
- [ ] Rate limiting on expensive or sensitive operations

---

## Deserialization

- [ ] No `ObjectInputStream` (Java) with untrusted data
- [ ] No `pickle.loads()` (Python) with untrusted data
- [ ] No `yaml.load()` without `SafeLoader` (Python)
- [ ] No `eval()` / `Function()` (JavaScript) with user input
- [ ] JSON schema validation for all parsed external JSON

---

## Configuration & Infrastructure

- [ ] Debug mode disabled in production
- [ ] Error pages don't expose stack traces or internal details
- [ ] CORS configured with specific origins, not `*`
- [ ] HTTP security headers: `X-Content-Type-Options`, `X-Frame-Options`, `Strict-Transport-Security`
- [ ] Secrets not in source code, Docker images, or public config files
- [ ] Database connections use least-privilege credentials

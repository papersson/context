# Modern Web Application & Service Security Standards

**Synthesized from the complete OWASP Cheat Sheet Series (113 sheets)**

This document distills every OWASP Cheat Sheet into a single, actionable reference for building and operating secure web applications and services. Sections are ordered by **priority** -- the most dangerous and commonly exploited areas come first.

---

## Priority Legend

| Level | Meaning |
|-------|---------|
| **P0 -- CRITICAL** | Exploited constantly in the wild. A gap here will get you breached. |
| **P1 -- HIGH** | Frequently exploited or high blast radius. Must be addressed before production. |
| **P2 -- MEDIUM** | Important defense-in-depth. Should be addressed in a mature security program. |
| **P3 -- STANDARD** | Best practice. Reduces attack surface and improves resilience. |

---

# P0 -- CRITICAL

These are the most commonly exploited vulnerability classes. They appear in virtually every breach post-mortem. Get these right first.

---

## 1. Injection Prevention

Injection remains the most dangerous class of web vulnerabilities. It encompasses SQL injection, OS command injection, LDAP injection, XPath injection, and NoSQL injection.

### Universal Rules

1. **Use parameterized queries / prepared statements for all database access.** This is the single most effective defense against SQL injection. Never concatenate user input into query strings.
2. **Use safe APIs that avoid interpreters entirely.** Prefer library functions over shell commands (e.g., use `mkdir()` not `system("mkdir /dir")`).
3. **Apply allowlist-based input validation** as a secondary layer. Validate data type, length, range, and format. Deny-lists are trivially bypassed.
4. **Apply context-specific output encoding** when rendering user data. The encoding must match the output context (HTML body, HTML attribute, JavaScript, CSS, URL).

### SQL Injection

- **Primary defense:** Prepared statements with bind parameters. Every major language supports them:

  | Language | Mechanism |
  |----------|-----------|
  | Java | `PreparedStatement` with `setString()` |
  | .NET | `SqlCommand` with `Parameters.Add()` |
  | Python | DB-API `cursor.execute(sql, params)` |
  | PHP | PDO `prepare()` with named/positional placeholders |
  | Ruby | ActiveRecord or `?` placeholders |
  | Rust | SQLx compile-time checked macros or `.bind()` |
  | Node.js | Driver-native parameterized queries |

- **Stored procedures** are equally effective when they avoid internal dynamic SQL.
- **Allowlist mapping** for elements that cannot be parameterized (table names, column names, sort direction) -- map user input to hardcoded safe values.
- **Escaping is explicitly discouraged** by OWASP as "fragile" and unable to guarantee prevention.
- **Least privilege:** Application database accounts should never have DBA/admin rights, should be restricted to specific tables and operations (SELECT, UPDATE, DELETE), and should never own the database.

### OS Command Injection

- Avoid OS commands entirely when library alternatives exist.
- If unavoidable, use parameterized execution (Java `ProcessBuilder` with separate args; never concatenate).
- Allowlist commands and validate arguments, excluding metacharacters (`& | ; $ > < \ !`).
- Use `--` delimiter to mark end of arguments.
- Even properly escaped commands remain vulnerable to argument injection -- a distinct threat.

### NoSQL Injection

- Never use string concatenation to build queries. Use driver-native query objects.
- Reject `$`-prefixed keys in user input (blocks `$where`, `$regex`, `$expr` operator injection).
- Prefer high-level ODMs (Mongoose, Spring Data). Avoid `.eval()` and raw query execution.

### LDAP Injection

- **DN escaping:** Escape `\ # + < > , ; " =` and leading/trailing spaces.
- **Search filter escaping:** Escape `* ( ) \ NUL` per RFC 4515.
- Use framework-based protection: OWASP ESAPI, .NET `Encoder.LdapFilterEncode()`, or Java's parameterized `ctx.search()`.

---

## 2. Cross-Site Scripting (XSS)

XSS is the most prevalent web vulnerability. It enables session hijacking, credential theft, defacement, and malware distribution.

### Primary Defenses

1. **Use framework auto-escaping.** Modern frameworks (React, Angular, Vue, Django, Rails, Twig) escape output by default. Understand and avoid their security bypass mechanisms:
   - React: `dangerouslySetInnerHTML`
   - Angular: `bypassSecurityTrustAs*`
   - Django: `|safe`, `mark_safe()`
   - Rails: `raw()`, `html_safe()`, `<%==`
   - Twig/Blade: `{!! !!}`, `|raw`

2. **Context-specific output encoding** when framework auto-escaping is insufficient:
   - **HTML body:** Entity-encode `& < > " '`
   - **HTML attributes:** `&#xHH;` format; always quote attribute values
   - **JavaScript:** `\xHH` encoding; variables only in quoted strings
   - **CSS:** CSS hex encoding; variables only in property values
   - **URLs:** Percent-encode, then HTML-attribute-encode

3. **HTML sanitization** for user-authored rich content: Use **DOMPurify** (or equivalent). Do not modify sanitized content afterward.

4. **Safe DOM APIs:** Prefer `textContent`, `insertAdjacentText()`, `setAttribute(safeName, value)`, `createElement()`.

### DOM-Based XSS

- Replace `innerHTML` with `textContent`/`innerText` wherever possible.
- Build dynamic UI with `createElement()` / `setAttribute()` / `appendChild()`.
- Use `JSON.parse()` instead of `eval()` for JSON.
- Never use `eval()`, `setTimeout(string)`, `setInterval(string)`, or `new Function(string)` with untrusted input.

### XSS Filter Evasion

OWASP catalogs 100+ evasion techniques that defeat naive input filters, including encoding bypasses, event handler exploitation, SVG/XML injection, data URIs, and CSS injection. **Input filtering alone is an incomplete defense.** Context-aware output encoding is mandatory.

### Content Security Policy (CSP)

CSP is a critical defense-in-depth layer, not a primary defense:

- **Strict nonce-based policy:** `script-src 'nonce-{RANDOM}' 'strict-dynamic'; object-src 'none'; base-uri 'none';`
- Generate a unique nonce per response using a CSPRNG.
- `strict-dynamic` automatically trusts scripts loaded by already-trusted scripts.
- Move inline scripts to external files. Replace `onclick=` handlers with `addEventListener`.
- Use `frame-ancestors 'none'` or `'self'` to prevent framing (supersedes X-Frame-Options).

---

## 3. Broken Authentication

Authentication failures account for the majority of account compromises. MFA alone prevents 99.9% of them.

### Password Policy

- **Minimum 8 characters with MFA; minimum 15 characters without MFA.**
- Maximum length at least 64 characters to support passphrases.
- Allow all characters including Unicode and whitespace. Eliminate composition rules.
- Never silently truncate passwords.
- Check against breached password databases (Pwned Passwords API).
- Do **not** require mandatory periodic rotation. Prefer strong passwords + MFA.
- Include a strength meter (zxcvbn-ts recommended).

### Password Storage

Algorithms in priority order:

| Algorithm | Configuration | Notes |
|-----------|--------------|-------|
| **Argon2id** | 19 MiB memory, 2 iterations, 1 parallelism | Preferred. Winner of 2015 Password Hashing Competition. |
| **scrypt** | N=2^17, r=8, p=1 | Fallback when Argon2id unavailable. |
| **bcrypt** | Work factor 10+ | Legacy. 72-byte input limit. |
| **PBKDF2** | 600,000 iterations (HMAC-SHA-256) | Only for FIPS-140 compliance. |

- Unique random salt per password. Modern algorithms handle this automatically.
- **Peppering:** Shared secret stored separately (vault or HSM). HMAC the hash with pepper as key.
- Re-hash with stronger algorithms on next user login.

### Multi-Factor Authentication (MFA)

- **FIDO2/WebAuthn/Passkeys** are the strongest option: phishing-resistant, no manual code entry.
- **TOTP** (authenticator apps) is the recommended standard for all users.
- **Avoid SMS** for applications handling PII or financial data (SIM-swap vulnerable, prohibited by NIST).
- **Security questions are deprecated** per NIST SP 800-63 and do not constitute a valid factor.
- Multiple instances of the same factor type (password + PIN) is NOT MFA.

### Enumeration Prevention

- Return identical error messages for all authentication failure modes.
- Ensure identical processing logic, HTTP status codes, and response times for existing vs. non-existing accounts.
- Apply to login, registration, password reset, and email change flows.

### Credential Stuffing Defense

Layer these controls:
1. MFA (mandatory for admins and high-privilege accounts)
2. Risk-based triggers (new device, unusual location, anonymization services)
3. CAPTCHA on suspicious attempts
4. IP-based graduated responses
5. Device and connection fingerprinting (JA3, HTTP/2 fingerprinting)
6. Breached password checking (ASVS 2.1.7)
7. Multi-step login flows with CSRF tokens

### Forgot Password

- Consistent messages for existing and non-existing accounts. Uniform response times.
- Tokens: cryptographically random, single-use, time-limited, stored hashed.
- Hard-code base URLs or validate against trusted domain allowlist (prevents host header injection).
- Post-reset: require normal login; never auto-login; send notification email; offer to invalidate all sessions.

---

## 4. Broken Access Control & Authorization

### Core Principles

1. **Deny by default.** The application must explicitly permit access.
2. **Validate permissions on every request** using application-wide middleware/filters, not per-method checks.
3. **Enforce server-side.** Client-side checks are trivially bypassed.
4. **Prefer ABAC/ReBAC over RBAC.** Attribute-based and relationship-based models scale better and reflect real-world access patterns.
5. **Protect against IDOR.** Verify user ownership of every accessed object. Retrieve objects based on authenticated user identity, not client-supplied identifiers.
6. **Enforce authorization on static resources** including cloud storage (S3, GCS, Azure Blob).
7. **Exit safely on failure.** Centralize failure handling. Never expose sensitive information.

### Session Management

- **Cookie security attributes:** `Secure`, `HttpOnly`, `SameSite=Lax` or `Strict`. Omit `Domain` to restrict to origin server.
- Use generic cookie names (not `JSESSIONID`, `PHPSESSID`).
- Minimum 128 bits of session ID uniqueness via CSPRNG.
- **Regenerate session ID** after every privilege level change (login, password change, role switch).
- **Exchange via cookies only** -- never URL parameters.
- **Idle timeout:** 2-5 minutes for high-value, 15-30 minutes for low-risk. Server-side enforcement mandatory.
- **Absolute timeout:** 4-8 hours maximum regardless of activity.
- **Logout:** Server-side invalidation mandatory. Return `Clear-Site-Data` header.
- Never store session IDs or credentials in `localStorage` (a single XSS exposes everything).

### OAuth 2.0

- Use authorization code grant with PKCE exclusively. Never use Implicit Flow or Resource Owner Password Credentials.
- Implement sender-constraining via DPoP (RFC 9449) or mTLS (RFC 8705).
- Restrict token audience to specific resource servers. Verify audience on every request.
- Use asymmetric client authentication (`private_key_jwt` or mTLS).

### CSRF Prevention

- **Prerequisite:** XSS prevention must come first -- XSS defeats all CSRF mitigations.
- **Stateful apps:** Synchronizer Token Pattern with cryptographically random per-session tokens.
- **Stateless apps:** Signed Double-Submit Cookie (HMAC-signed, binding session ID + random value + server secret).
- **All apps:** Check `Sec-Fetch-Site` header (98%+ browser support).
- **SameSite cookies** supplement but do not replace tokens.

---

## 5. Secrets Management

Hardcoded secrets are found in virtually every codebase audit. They are trivially exploitable and frequently lead to full compromise.

### Core Rules

- **Never hardcode secrets** in source code, Docker images, config files, or environment variables.
- **Centralize** secrets in a dedicated management system (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, GCP Secret Manager).
- **Encrypt** secrets at rest, in transit, and ideally in use. Use AES-256-GCM or ChaCha20-Poly1305.
- **Least privilege:** No engineer should access all secrets. Scope permissions to individual secrets.

### Lifecycle

- **Creation:** Cryptographically robust generation. Transmit through separate secure channels.
- **Rotation:** Automated, proportional to sensitivity. User credentials: rotate only upon suspected compromise (per NIST).
- **Revocation:** Immediate deauthorization on compromise.
- **Expiration:** Defined lifespans; verify active status before trusting.

### Dynamic Secrets (Preferred)

Generate short-lived credentials at runtime rather than rotating static ones. On service restart, credentials automatically expire, reducing the compromise window.

### CI/CD

- Treat CI/CD systems as production environments.
- Prevent secret exfiltration through logging.
- Use designated service accounts with scoped access.
- **Hierarchy:** Dedicated secrets management (best) > runtime retrieval > native CI/CD tooling (worst).

### Detection

- Pre-commit hooks and IDE-level secret detection.
- Tools: truffleHog, git-secrets, GitGuardian, Yelp Detect Secrets.
- If secrets are committed: immediate revocation, automated new secret creation, git-history rewriting.

---

# P1 -- HIGH

These vulnerabilities are frequently exploited or have high blast radius when they occur.

---

## 6. Server-Side Request Forgery (SSRF)

SSRF allows attackers to make the server send requests to internal services, cloud metadata endpoints, and other unintended targets.

### Defenses

- **Allowlist approach (internal apps):** Validate input types with battle-tested libraries. Accept only validated IPs or domain names, never complete URLs. Verify domains against allowlist before DNS resolution. Disable HTTP redirect following.
- **Blocklist approach (external requests):** Validate IP/domain format. Verify all resolved IPs are public (not private/loopback/link-local). Allowlist protocols (HTTP/HTTPS only). Prevent DNS rebinding attacks.
- **Cloud metadata protection:** Migrate to IMDSv2 (AWS). Block `169.254.169.254` and equivalent endpoints.
- SSRF extends beyond HTTP to FTP, SMB, SMTP, `file://`, `gopher://` -- restrict accordingly.
- **Network layer:** Firewall rules restricting outbound connections from application servers.

---

## 7. XML External Entity (XXE) Prevention

### Primary Rule

**Disable DTDs (external entities) completely.** This prevents ~95% of XXE attacks and all Billion Laughs denial-of-service.

| Platform | Configuration |
|----------|--------------|
| **Java** | `setFeature("http://apache.org/xml/features/disallow-doctype-decl", true)` |
| **.NET** | Safe by default in 4.5.2+. Earlier: set `XmlResolver = null` |
| **PHP** | Safe by default in 8.0+. Earlier: `libxml_set_external_entity_loader(null)` |
| **Python** | Use `defusedxml` package. Standard library parsers are vulnerable. |
| **C/C++** | libxml2: avoid `XML_PARSE_NOENT` and `XML_PARSE_DTDLOAD` |

- SOAP messages containing DOCTYPE declarations violate the spec -- reject them.
- Entity expansion attacks (Billion Laughs, Quadratic Blowup) require entity expansion depth/size limits.

---

## 8. Insecure Deserialization

### Core Principle

Avoid native serialization formats. Use pure data formats (JSON, XML). Sign serialized data and reject unsigned input.

| Language | Key Guidance |
|----------|-------------|
| **Java** | Override `ObjectInputStream.resolveClass()` with allowlist. **Never use XMLDecoder.** Safe alternatives: fastjson2 (default), jackson-databind (without polymorphism). |
| **.NET** | **Never use `BinaryFormatter`** (Microsoft declares it unsafe and unsecurable). Use `DataContractSerializer`. Set `TypeNameHandling = None`. |
| **PHP** | Replace `unserialize()` with `json_decode()`. |
| **Python** | Avoid `pickle`/`cPickle`, `PyYAML.load()`, `jsonpickle`. Use `yaml.SafeLoader`. |

---

## 9. Cryptographic Storage & Transport

### Algorithms

- **Symmetric encryption:** AES-256 in GCM mode (authenticated encryption). Never use ECB mode.
- **Asymmetric encryption:** Elliptic Curve with Curve25519 preferred. RSA minimum 2048-bit keys.
- **Hashing (general):** SHA-256 or SHA-512.
- **Hashing (passwords):** Argon2id > scrypt > bcrypt > PBKDF2 (see section 3).
- **Random numbers:** Always use a CSPRNG. `SecureRandom` (Java), `secrets` (Python), `crypto.randomBytes` (Node.js), `crypto/rand` (Go).
- **Never implement custom cryptographic algorithms.**

### Key Management

- Use the DEK/KEK pattern: Data Encryption Key encrypts data, Key Encryption Key encrypts the DEK. Store separately.
- Store keys in HSMs, cloud key vaults, or external secrets services. Never in source code or environment variables.
- Each key should serve one function only (encryption OR authentication OR signing).
- Document rotation procedures before you need them. Rotation triggers: compromise, cryptoperiod expiration, algorithm vulnerability.

### TLS Configuration

- **Default to TLS 1.3**, fall back to TLS 1.2 only if necessary.
- **Disable:** SSL 2.0, SSL 3.0, TLS 1.0, TLS 1.1, null ciphers, anonymous ciphers, export ciphers.
- Enable only GCM cipher suites where possible. Use Mozilla's TLS configuration generator.
- Minimum 2048-bit certificate keys. Use SHA-256 for hashing. Include all FQDNs in SAN.
- **Disable TLS compression** (prevents CRIME vulnerability).
- **Use TLS for ALL pages**, not just sensitive ones.
- **HSTS:** `Strict-Transport-Security: max-age=63072000; includeSubDomains; preload`. Start with short max-age during rollout.

---

## 10. HTTP Security Headers

Every response should include these headers:

| Header | Recommended Value | Purpose |
|--------|-------------------|---------|
| `Strict-Transport-Security` | `max-age=63072000; includeSubDomains; preload` | Force HTTPS |
| `Content-Security-Policy` | Strict nonce-based or hash-based policy | XSS defense-in-depth |
| `X-Content-Type-Options` | `nosniff` | Block MIME-type sniffing |
| `X-Frame-Options` | `DENY` | Prevent clickjacking (legacy; use CSP `frame-ancestors`) |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Prevent URL leakage |
| `Permissions-Policy` | `geolocation=(), camera=(), microphone=()` | Disable unnecessary browser APIs |
| `Cross-Origin-Opener-Policy` | `same-origin` | Prevent Spectre-class attacks |
| `Cross-Origin-Resource-Policy` | `same-site` | Restrict resource loading |
| `Cache-Control` | `no-store` (for sensitive responses) | Prevent caching of sensitive data |

**Remove:** `Server`, `X-Powered-By`, `X-AspNet-Version`.

**Disable:** `X-XSS-Protection` (set to `0` -- legacy auditors can introduce vulnerabilities).

---

## 11. File Upload Security

### Validation

- **Allowlist extensions** (never denylist). Guard against double extensions (`.jpg.php`), null bytes (`.php%00.jpg`).
- Never trust the `Content-Type` header alone. Validate file signatures (magic bytes) in conjunction.
- Generate random filenames (UUID). If user-supplied names are needed: enforce max length, restrict to `[a-zA-Z0-9\-\. ]`, block leading dots and sequential dots.

### Sanitization

- Rewrite images to strip injected payloads. Use Content Disarm & Reconstruct (CDR) for PDFs and Office documents.
- Avoid accepting ZIP files (zip bombs, path traversal). Scan through antivirus/sandbox.

### Storage (ranked by security)

1. Separate hosting infrastructure entirely.
2. Outside webroot with admin-only access.
3. Inside webroot with write-only permissions.

- Require authentication for uploads. Verify authorization for both upload and download.
- Enforce file size limits. Protect upload forms from CSRF.

---

## 12. Software Supply Chain Security

### Dependencies

- Pin versions using lockfiles. Use `npm ci` / `yarn install --frozen-lockfile` in CI.
- Continuously scan with OWASP Dependency-Check, `npm audit`, Snyk, Retire.js.
- Generate SBOMs automatically in CI/CD (CycloneDX or SPDX format).
- Assess open-source components for: active maintenance, vulnerability reporting, test coverage, license alignment.

### Build Process

- Digitally sign all artifacts and validate signatures before deployment.
- Use ephemeral build environments destroyed after completion.
- Store CI/CD configs in version control. Generate verifiable provenance per SLSA standards.

### NPM-Specific

- Use scoped packages (`@yourorg/package`). Configure `.npmrc` to route scopes to private registry.
- Use `--ignore-scripts` flag and allowlist legitimate lifecycle scripts.
- Enable 2FA with `npm profile enable-2fa auth-and-writes`.
- Watch for typosquatting and AI-hallucinated package names ("slopsquatting").

---

# P2 -- MEDIUM

Important defense-in-depth measures for a mature security program.

---

## 13. API & Service Security

### REST APIs

- Serve exclusively over HTTPS. Enforce access control at every endpoint.
- Reject JWT tokens with `{"alg":"none"}`. Validate using server-configured algorithms. Validate `iss`, `aud`, `exp`, `nbf` claims.
- Allowlist permitted HTTP methods per resource. Reject others with `405`.
- Validate Content-Type headers. Return generic error messages only -- never stack traces.
- Credentials in request body or headers, never in URLs or query strings.
- Security headers on all responses: `Cache-Control: no-store`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Strict-Transport-Security`.

### GraphQL

- **Query depth limiting** to prevent deeply nested abuse.
- **Query cost analysis** with field-level computational costs and rejection thresholds.
- **Disable introspection in production.** Remove GraphiQL/exploration tools.
- Enforce authorization on every query/mutation, on both edges and nodes.
- Mitigate batching attacks: track object request counts per caller, limit concurrent operations.

### gRPC

- TLS 1.2+ mandatory. Implement mTLS for zero-trust. Short-lived certificates (90 days max) with automated rotation.
- JWT validation via interceptors before handler execution. 15-60 minute token expiration.
- Use `protoc-gen-validate` for input validation rules beyond protobuf type safety.
- Set max message sizes (e.g., 4 MB). Limit streaming sessions and message counts.
- **Disable gRPC reflection in production.**

### WebSocket

- Always use `wss://` (WebSocket Secure). Allowlist-based origin validation during handshake.
- Periodically re-validate sessions (30-minute intervals). Per-message authorization -- never assume connection grants unlimited access.
- Cap payload size (64 KB typical). Rate-limit messages (100 msg/min baseline).
- JSON schema + allowlist validation on all messages. Use `JSON.parse()` exclusively.

### Microservices

- Centralize coarse-grained access control at the API gateway.
- Use centralized policy with embedded PDP (policy evaluated locally via library or sidecar).
- For identity propagation: decouple external tokens from internal identity structures. Never expose internal tokens externally.
- Service-to-service auth: mTLS or signed tokens over TLS.

---

## 14. Infrastructure Security

### Docker

- Keep host kernel and Docker Engine updated. Containers share the host kernel.
- **Never expose `/var/run/docker.sock`** to other containers (socket access = root).
- Run as non-root (`USER myuser` in Dockerfile). Never use `--privileged`.
- Drop all capabilities (`--cap-drop all`), add back only needed ones.
- Use `--security-opt=no-new-privileges`. Apply Seccomp/AppArmor/SELinux profiles.
- Run with `--read-only` filesystem. Mount volumes read-only (`:ro`).
- Set resource limits (`--memory`, `--cpus`). Use rootless mode or Podman.
- Use `docker secret create` for secrets. Never bake secrets into images.

### Kubernetes

- Restrict dashboard access. Enable RBAC with `--authorization-mode=Node,RBAC`.
- Prefer OIDC with short-lived tokens for API authentication.
- Enforce **restricted** Pod Security Standards on namespaces.
- Default-deny NetworkPolicies for all pod-to-pod traffic. Explicit allowlists only.
- Mount secrets as read-only volumes, never environment variables. Use external secret managers.
- Enable encryption at rest for etcd. Enable audit logging.
- SecurityContext: `runAsNonRoot: true`, `readOnlyRootFilesystem: true`, drop all capabilities.
- Monitor for shell execution inside containers, unexpected file reads, outbound connections.

### Cloud Architecture

- Conduct risk assessments and threat modeling before architecture decisions.
- Use private subnets for databases and backends. Public subnets only for load balancers and bastion hosts.
- Deploy WAF at entry points. Enable DDoS protection (AWS Shield, Cloud Armor, Azure DDoS Protection).
- Log all L7 HTTP calls. Alert on 4xx/5xx spikes, resource anomalies, cost overruns.
- Understand your shared responsibility model (IaaS vs PaaS vs SaaS).

### Network Segmentation

Three-layer architecture:

| Zone | Contains |
|------|----------|
| **FRONTEND** | Load balancers, WAF, web servers |
| **MIDDLEWARE** | Application services, auth, message queues |
| **BACKEND** | Databases, LDAP/AD, crypto key storage |

- Allow: FRONTEND <--> MIDDLEWARE (same app); MIDDLEWARE <--> external networks.
- Prohibit: Cross-application traffic between zones; direct backend-to-backend access.

### CI/CD Security

- Disable auto-merge. Mandate human review. Protected branches. Signed commits.
- Execute builds in isolated nodes. Require manual approval before prod deploy.
- Never hardcode secrets in repos. Encrypt at rest. Prevent appearance in logs.
- Pin dependency versions with immutable references. Validate integrity via checksums.
- Vet plugins for reputation, maintenance, security impact.

### Zero Trust Architecture

Seven principles: protect all resources regardless of location; encrypt and authenticate all communication; per-session access with short-lived credentials; dynamic policy-based decisions; continuous asset monitoring; real-time enforcement; comprehensive telemetry.

Implementation roadmap: Foundation (months 1-6: inventory, MFA, ZTNA), Controls (months 6-18: micro-segmentation, identity-aware proxies), Advanced (months 18-36: behavioral monitoring, automated response).

---

## 15. Logging & Monitoring

### Events to Always Log

- Input/output validation failures
- Authentication successes and failures
- Authorization failures
- Session management failures (creation, expiry, invalidation)
- All higher-risk functionality: user admin, privilege changes, sensitive data access, crypto operations, data import/export

### Event Attributes

Record: timestamp (ISO 8601 with UTC offset), interaction ID, application ID/version, hostname, service name, source address, user identity, event type, severity, description.

### Data to Never Log

Source code, session IDs (use hashed values), access tokens, passwords, database connection strings, encryption keys, payment card data, sensitive PII.

### Standardized Vocabulary

Use consistent event names across all services. Critical events by category:

| Category | Key Events |
|----------|-----------|
| **Authentication** | `login_success`, `login_fail`, `login_fail_max`, `login_lock`, `impossible_travel` |
| **Authorization** | `authz_fail`, `authz_change`, `authz_admin` |
| **Session** | `session_created`, `session_expired`, `session_use_after_expire` |
| **Input Validation** | `input_validation_fail` |
| **Malicious Behavior** | `sqli`, `excess_404`, `attack_tool`, `cors_violation` |

### Storage

- Separate partitions with strict permissions. Exclude from web-accessible locations.
- Build tamper detection. Transmit over secure protocols. Implement retention policies.
- Integrate with SIEM and incident response. Synchronize clocks across all servers.

---

## 16. Error Handling

- **Return generic messages to users. Log detailed errors server-side.**
- Use 4xx for client errors, 5xx for server errors.
- Disable detailed error pages in production. Never expose stack traces, file paths, SQL queries, or technology versions.
- Monitor 5xx occurrences as early indicators of application issues.

---

## 17. Denial of Service Prevention

### Application Layer

- Deploy cheap validation first to minimize resource impact.
- Implement graceful degradation. Eliminate single points of failure.
- Implement session timeouts. Restrict file upload sizes. Limit total request size.
- Use authentication to gate access to expensive functions.
- Beware: account lockout mechanisms can be weaponized as DoS.

### Network Layer

- Rate limiting with minimum/maximum ingress data rates, connection timeouts, per-resource load limits.
- Caching and static resource separation. Commercial DDoS protection for volumetric attacks.

---

## 18. Database Security

### Relational

- Disable TCP access when possible; use local sockets. Bind to localhost only.
- Place databases in a separate network zone from application servers.
- Enforce TLS 1.2+ on all connections. Dedicate each account to a single application.
- Never use root/sa/SYS accounts. Grant only essential permissions at table/column/row level.
- Store credentials outside web root, encrypted, never in source control.
- Platform hardening: MSSQL (disable xp_cmdshell, CLR, SQL Browser); MySQL (run mysql_secure_installation, disable FILE privilege).

### NoSQL

- Enable mandatory authentication. Implement RBAC with minimum permissions.
- Bind to internal interfaces only. Disable server-side code execution (MongoDB `db.eval`).
- Log connection attempts, auth failures, admin actions, suspicious commands.

---

## 19. Mass Assignment Prevention

Attackers inject unintended parameters (e.g., `isAdmin=true`) through automatic HTTP-to-object binding.

### Three Defense Strategies

1. **Allowlisting (recommended):** Explicitly declare which fields accept user input.
2. **Blocklisting (secondary):** Explicitly prohibit sensitive fields. More error-prone.
3. **DTO Pattern:** Intermediate Data Transfer Objects containing only editable fields.

Framework implementations: Spring MVC `@InitBinder` with `setAllowedFields()`, Laravel `$fillable`, Rails strong parameters, Node.js manual field picking.

---

## 20. Clickjacking Defense

1. **CSP `frame-ancestors` directive (preferred):** `frame-ancestors 'none'` blocks all framing.
2. **`X-Frame-Options`:** `DENY` or `SAMEORIGIN`.
3. **SameSite cookies:** `Strict` or `Lax` as supplementary control.

Deploy multiple mechanisms simultaneously. Add headers to all HTML responses.

---

## 21. Subdomain Takeover Prevention

Attackers reclaim deprovisioned cloud resources that DNS records still point to. Impact: session cookie theft, CSP bypass, phishing, OAuth compromise, TLS certificate acquisition.

Prevention: correct DNS record lifecycle (update DNS before removing cloud resource), DNS inventory, automated weekly scans for dangling records, restrict wildcard DNS, scope cookies with `__Host-` prefix, use exact-match OAuth redirect URIs.

---

# P3 -- STANDARD

Best practices that reduce attack surface and improve resilience.

---

## 22. Threat Modeling & Secure Design

### Threat Modeling Process

1. **Model the system** with Data Flow Diagrams showing trust boundaries.
2. **Identify threats** using STRIDE or equivalent methodology. Rank by likelihood x impact.
3. **Respond:** Mitigate, eliminate, transfer, or accept each threat.
4. **Validate:** Verify comprehensive coverage and testability of mitigations.

Perform threat modeling early in design and iterate throughout the lifecycle.

### Secure Product Design Principles

- Least Privilege and Separation of Duties
- Defense-in-Depth (multiple layered controls)
- Zero Trust (assume all untrusted; verify everything; continuously monitor)
- Ten code fundamentals: input validation, secure error handling, auth/authz, cryptography, least privilege, secure memory management, no hardcoded secrets, security testing, code auditing, patching

### Attack Surface Analysis

- Map all entry/exit points: UI forms, HTTP headers, cookies, APIs, files, databases, runtime arguments.
- Focus on high-risk areas: internet-facing code, auth logic, crypto implementation, admin interfaces.
- Track changes over time using Relative Attack Surface Quotient.

---

## 23. Secure Code Review

### Techniques

- **Code pattern analysis:** Input processing, DB queries, file operations, auth logic, crypto, error handling.
- **Data flow analysis:** Identify sources, follow processing, check sinks, validate trust boundaries.
- **Threat-based review:** Align with OWASP Top 10, STRIDE, attack trees.
- **Business logic review:** State management, race conditions, transaction integrity, workflow bypass.

### Checklists

- Cryptography: AES-256, RSA-2048+, ECDSA P-256+. No custom crypto.
- Input validation: Allowlist-based, server-side, with context-specific encoding on output.
- Authentication: MFA, secure password storage, session regeneration.
- Authorization: Verify on every request, test deny-by-default.

---

## 24. Browser-Side Security

### Prototype Pollution Prevention

1. Use `Map` and `Set` instead of plain objects.
2. Create objects with `Object.create(null)`.
3. Use `Object.freeze()` / `Object.seal()` for protection.
4. Node.js: `--disable-proto=delete` flag.

### DOM Clobbering Prevention

- DOMPurify with `SANITIZE_NAMED_PROPS: true`.
- Always use `var`, `let`, or `const` (never implicit globals).
- Enable `"use strict"` mode.

### XS-Leaks (Cross-Site Leaks)

Browser side-channel attacks exploiting cross-site communication metadata. Key defenses:
- `Cross-Origin-Opener-Policy: same-origin`
- `Cross-Origin-Resource-Policy: same-origin`
- `SameSite` cookies
- Fetch Metadata validation (`Sec-Fetch-Site`, `Sec-Fetch-Dest`)
- `Cache-Control: no-store` for sensitive resources

### Third-Party JavaScript

- **Subresource Integrity (SRI):** Add `integrity` and `crossorigin` attributes to all third-party script tags.
- Confine tag managers to approved data layer access.
- Sandbox vendor scripts in iframes from a different domain.
- Require contractual evidence of secure coding from vendors.

### Cookie Theft Mitigation

- Server-side monitoring of session environmental changes (IP, User-Agent, Accept-Language).
- Graduated response: CAPTCHA for suspect activity, re-authentication for sensitive operations.

---

## 25. Multi-Tenant Security

### Top Threats

Cross-tenant data leakage, tenant impersonation, broken isolation, IDOR, noisy neighbor, privilege escalation, shared resource poisoning.

### Key Controls

- Establish tenant context early in middleware. Derive from verified JWT claims, not client-supplied IDs.
- Database isolation: separate databases (highest), separate schemas, shared tables with RLS, or hybrid.
- Always validate resource belongs to current tenant. Use composite keys (tenant_id + resource_id).
- Prefix all cache keys with tenant identifier. Validate on retrieval.
- Per-tenant rate limits. Separate API keys per tenant.
- Tenant-specific KMS keys for high-security tiers.
- Return 404 (not 403) for cross-tenant resource requests.

---

## 26. Serverless / FaaS Security

- **Role-per-function.** Scope IAM actions to specific resource ARNs. Never use `"Action": "*"`.
- Deploy in private subnets with controlled egress. Isolate sensitive functions separately.
- Treat all event payloads as untrusted. Validate against injection.
- Never store secrets in environment variables. Use vault services with ephemeral credentials.
- Never assume clean execution context between invocations.

---

## 27. Infrastructure as Code Security

- IDE plugins for early detection (TFLint, Checkov). Threat model early.
- Static analysis for misconfigurations. Dependency scanning. Container image scanning.
- **Immutable infrastructure:** Build to exact spec; provision new infra for changes; decommission old.
- Label and track all deployed resources. Securely erase on decommission.
- Continuous compliance monitoring with real-time alerting.

---

## 28. Vulnerability Disclosure

### For Organizations

- Publish dedicated security contact mechanisms, `security.txt` (RFC 9116), PGP keys.
- Provide clear reporting guidelines with safe harbor provisions.
- Acknowledge receipt promptly. Provide regular status updates.
- Only implement bug bounty after establishing mature internal vulnerability processes.

### For Researchers

- Ensure testing is legal and authorized.
- Submit reproducible reports over encrypted channels.
- Coordinate publication timing with vendors.

---

## 29. AI & LLM Security

### AI Agent Security

- **Tool security:** Grant minimum tools required. Per-tool permission scoping. Separate tools for different trust levels.
- **Human-in-the-loop:** Require approval for high-impact/irreversible actions. Risk classify all operations.
- **Memory security:** Validate data before storage. Isolate between users/sessions. Set expiration and size limits.
- **Output validation:** Validate before execution. Rate limits and scope limits. Detect exfiltration attempts.
- **Multi-agent:** Trust boundaries between agents. Message signing. Circuit breakers for cascading failures.

### LLM Prompt Injection Prevention

- Treat all external data as untrusted (user messages, retrieved documents, API responses, emails, code comments).
- Use delimiters and clear boundaries between instructions and data.
- Apply input sanitization, encoding detection, and length limits.
- Monitor outputs for system prompt leakage and sensitive data exposure.
- **No single mechanism is sufficient.** Best-of-N attacks demonstrate systematic defeat of individual defenses.

### MCP (Model Context Protocol) Security

- Inspect tool descriptions and schemas before approval -- the entire schema is an injection surface.
- Pin definitions using cryptographic hashes; alert on changes (rug pull defense).
- Run local servers in sandboxed environments. Restrict filesystem and network access.
- Require explicit user confirmation for destructive/financial/data-sharing operations.
- Treat tool responses as untrusted input; sanitize before returning to LLM context.
- Sign JSON-RPC messages with asymmetric keys. Include nonces and timestamps.

### Secure AI Model Ops

- Version-controlled, auditable training pipelines. Validate training data.
- Sign model binaries. Encrypt model weights at rest. Restrict access to training logs.
- Apply authentication, rate limiting, and input validation to inference APIs.
- Monitor input distribution, output entropy, and latency for drift detection.
- Include adversarial examples in testing. Use canary releases with rapid rollback.

---

## 30. Mobile Application Security

### Architecture

- All authentication and authorization server-side. Never trust the client.
- Request only necessary permissions. Ensure app signing.
- Store device-specific revocable tokens, not passwords.

### Data Protection

- Encrypt sensitive data at rest and in transit. Use platform APIs (Keychain/Keystore).
- Leverage hardware security (Secure Enclave on iOS, StrongBox on Android).
- Monitor caching, logging, and background snapshots for data leakage.
- Always use HTTPS. Do not override SSL certificate validation.

### Platform-Specific

- **Android:** Use Android Keystore with hardware backing. Implement Play Integrity API.
- **iOS:** Secure tokens in Keychain. Use Apple Secure Enclave. Implement App Attest API (iOS 14+).

### Runtime Protection

- Detect debugging, hooking, code injection, emulator/rooted/jailbroken execution.
- Verify app signatures at runtime. Obfuscate the binary.

---

## 31. Privacy Protection

- Strong cryptography for all data at rest and in transit.
- HSTS enforcement. Certificate pinning for mobile apps.
- **Panic modes:** Allow threatened users to delete data or access fake accounts.
- Remote session invalidation: let users see and terminate active sessions.
- Support anonymity network access (Tor, I2P) where appropriate.
- Transparent communication about data handling and legal protection limits.

---

## 32. Legacy Application Management

- Document all legacy apps with version numbers, configurations, network requirements.
- Treat as inherently high risk. Network-level isolation (subnet, IP allowlisting, VPN).
- Regular automated vulnerability scans. When patching is impossible, add access restrictions.
- Encrypt at rest and in transit. For apps limited to unencrypted protocols, maximize network restrictions.
- Cross-train multiple staff. Prevent single points of knowledge failure.
- Develop custom APIs to convert proprietary logs to SIEM-compatible formats.

---

## 33. Email Security

- Use tested libraries for email validation, not custom regex.
- Normalize domains to lowercase. Convert internationalized domains to punycode.
- Watch for homoglyph attacks (Latin vs. Cyrillic lookalikes).
- **Ownership verification:** Cryptographically secure, random, single-use, time-limited tokens.
- **Anti-enumeration:** Consistent responses for login and reset flows. Eliminate timing discrepancies.
- **Email change workflows:** Require re-authentication. Notify existing email. Confirm new email.
- Mask email addresses in logs. Never log verification tokens or URLs.

---

# Cross-Cutting Principles

These themes recur across all 113 cheat sheets:

### 1. Defense in Depth
No single control is sufficient. Layer application controls, network controls, and monitoring. Every sheet emphasizes this.

### 2. Allowlist Over Denylist
Every input validation sheet recommends positive validation. Deny-lists are trivially bypassed.

### 3. Server-Side Enforcement
Client-side validation is UX. Security validation happens server-side.

### 4. Least Privilege
Apply to database accounts, IAM roles, network access, API permissions, container capabilities, secrets access, and user permissions.

### 5. Parameterization / Safe APIs
Separate code from data at every boundary: SQL parameters, LDAP parameters, ProcessBuilder args, template engines.

### 6. Context-Aware Encoding
Output encoding must match the rendering context. HTML encoding in a JavaScript context provides zero protection.

### 7. Dangerous APIs to Avoid

| API | Risk |
|-----|------|
| `eval()`, `new Function()` | Code injection |
| `innerHTML`, `document.write()` | XSS |
| `BinaryFormatter` (.NET) | Insecure deserialization |
| `XMLDecoder` (Java) | Insecure deserialization |
| `pickle.load()` (Python) | Code execution |
| `unserialize()` (PHP) | Code execution |
| `child_process.exec()` (Node.js) | Command injection |
| `--privileged` (Docker) | Container escape |

### 8. Modern Algorithm Baselines

| Purpose | Algorithm | Minimum |
|---------|-----------|---------|
| Password hashing | Argon2id | 19 MiB memory, 2 iterations |
| Symmetric encryption | AES-GCM | 128-bit keys (256 preferred) |
| Asymmetric encryption | ECC Curve25519 | -- |
| RSA | RSA-OAEP | 2048-bit keys |
| TLS | TLS 1.3 (1.2 fallback) | GCM ciphers only |
| General hashing | SHA-256 | -- |
| Signatures | SHA-256+ | Never SHA-1 |
| Random numbers | CSPRNG | Platform-specific secure source |

---

# Language & Framework Quick Reference

## Node.js
- Enforce request body size limits. Use `helmet` middleware for security headers.
- Use `npm ci --omit=dev` in production. Run `npm audit` continuously.
- Never use `eval()`, `child_process.exec()`, or `vm` module with untrusted input.
- Node.js v20+ permission model: `node --permission --allow-fs-read=/uploads/`

## Python / Django
- `DEBUG = False` in production. Run `./manage.py check --deploy`.
- Use `django.contrib.auth` with `AUTH_PASSWORD_VALIDATORS`. Enable `SecurityMiddleware`.
- Enable CSRF middleware. Use `{% csrf_token %}`. Minimize `|safe` / `mark_safe()`.
- For DRF: Use explicit `Meta.fields` allowlists in serializers. Never `"__all__"`.

## Ruby on Rails
- `config.force_ssl = true`. Use database-backed sessions.
- Use Devise or AuthLogic. Enforce strong parameters for mass assignment protection.
- Use cancancan or pundit for authorization. Run Brakeman for static analysis.

## Java
- Use `PreparedStatement` with typed setters. Use OWASP Java Encoder for output encoding.
- Crypto: Use Google Tink when possible. AES-256-GCM with 12-byte unique nonces.
- Never use `XMLDecoder`. Safe JSON: fastjson2, jackson-databind without polymorphism.
- Log4j: Use JSON Template Layout. Parameterized logging. Max string length limits.

## C/C++
- Build with `-Wall -Wextra -Wconversion -Wformat=2 -Wformat-security`.
- `-fstack-protector-all` for stack canaries. `-z,relro -z,now` for full RELRO.
- `-fPIE -pie` for Position Independent Executables.
- Use assertions liberally. Verify with `checksec`.

---

# Virtual Patching

When you cannot fix the code immediately:

1. Deploy WAF rules in "Log Only" mode first.
2. **Allowlist approach (preferred):** Define valid input characteristics; reject non-conforming.
3. **Blocklist approach (faster but weaker):** Detect specific attack patterns.
4. Never create exploit-specific patches -- block the underlying attack vector class.
5. Document with rule IDs. Reassess for removal when permanent fixes land.

---

# Testing & Validation

- **Authorization testing automation:** Formalize the authorization matrix in machine-readable format. One test case per role testing all services.
- **Secure code review:** Both baseline (full codebase) and diff-based (pull requests).
- **Dependency scanning:** Continuous. Multiple tools. CVE-specific ignore rules only (never blanket).
- **SAST/DAST integration:** Embed in CI/CD pipeline. Fail builds on security findings in critical categories.

---

*Synthesized from the OWASP Cheat Sheet Series. Source: https://cheatsheetseries.owasp.org/*

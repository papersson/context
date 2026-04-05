# Cybersecurity

## Taxonomy

```
Corporate Governance
├── Governance (board-level oversight)
│   ├── Board Leadership & Purpose
│   ├── Stakeholder Rights & Engagement
│   ├── Disclosure & Transparency
│   ├── Audit, Risk & Internal Control
│   └── Remuneration & Succession
└── Management (executive functions)
    ├── Strategy
    ├── Finance
    ├── Operations
    ├── Legal & Regulatory
    ├── Human Resources
    └── Risk Management
        ├── Strategic Risk
        ├── Financial Risk
        ├── Operational Risk
        ├── Compliance & Legal Risk
        ├── Reputational Risk
        └── Information Security Risk
            ├── Physical Security
            ├── Personnel Security
            └── Cybersecurity Risk ← this file
                ├── Governance, Risk & Compliance (GRC)
                ├── Identity & Access Management (IAM)
                ├── Application Security
                ├── Network Security
                ├── Endpoint Security
                ├── Data Security
                ├── Infrastructure & Cloud Security
                ├── Security Operations
                ├── Resilience & Recovery
                ├── Security Awareness & Training
                └── Security Assurance & Testing
```

## Corporate Governance

Corporate governance is the system by which organizations are directed and controlled. It splits into two layers. **Governance** is board-level oversight: setting direction and ensuring accountability. **Management** is the executive layer: executing strategy and running operations. COBIT 2019 and the OECD Principles both make this separation explicit.

### Governance

The board's job. Not the day-to-day, but the oversight and accountability layer.

- **Board Leadership & Purpose**: Setting the organization's mission, values, and strategic direction. Ensuring management delivers on them.
- **Stakeholder Rights & Engagement**: Protecting shareholder and stakeholder interests, ensuring equitable treatment and participation.
- **Disclosure & Transparency**: Accurate, timely reporting on financial and non-financial matters.
- **Audit, Risk & Internal Control**: Overseeing audit functions, risk appetite, and the systems that catch problems before they escalate.
- **Remuneration & Succession**: Aligning executive compensation with long-term performance. Planning for leadership transitions.

### Management

The executive team's job. Running the organization day-to-day.

- **Strategy**: Defining objectives, competitive positioning, and resource allocation.
- **Finance**: Capital management, accounting, budgeting, financial controls, treasury, investor relations.
- **Operations**: Delivering the product or service. Supply chain, production, logistics, customer support.
- **Legal & Regulatory**: Contracts, intellectual property, litigation, navigating the regulatory environment.
- **Human Resources**: Hiring, development, culture, organizational design, labor relations.

## Risk Management

Identifying, assessing, and responding to threats across the organization. COSO ERM 2017 and ISO 31000 both stress that risk management is not a standalone silo but a lens applied to strategy and operations alike.

- **Strategic Risk**: Poor strategy choices, market shifts, competitive disruption, business model misalignment. Tends to be the most consequential category.
- **Financial Risk**: Market risk, credit risk, liquidity risk.
- **Operational Risk**: Process failures, human error, supply chain disruptions.
- **Compliance & Legal Risk**: Failing to meet regulatory requirements, exposure to litigation. Fines, sanctions, lawsuits.
- **Reputational Risk**: Damage to brand, trust, or public perception. Usually a second-order effect of other risks materializing.

## Information Security Risk

Protecting the confidentiality, integrity, and availability of information in all forms: digital, paper, verbal. That triad (CIA) is the foundation. Information security is broader than cybersecurity because it includes physical and personnel controls that have nothing to do with computers. ISO 27001 is the defining standard.

- **Physical Security**: Facility access controls, environmental protections (fire, flood, power), secure disposal of media, surveillance.
- **Personnel Security**: Background screening, acceptable use policies, insider threat management.
- **Cybersecurity Risk**: Protecting digital assets, electronic systems, and networks from attack. The technical core of information security, and the focus of the rest of this document.

---

## Cybersecurity Risk

Everything below here is the cybersecurity domain: protecting digital assets from cyber threats, organized by domain.

### Governance, Risk & Compliance (GRC)

Cybersecurity has its own governance layer. NIST CSF 2.0 added "Govern" as a top-level function specifically because organizations were neglecting it.

- **Policy & Standards**: Security policies, acceptable use rules, baseline configurations, framework adoption (NIST CSF, ISO 27001, CIS Controls).
- **Regulatory Compliance**: Meeting external requirements like SOC 2, PCI-DSS, HIPAA, GDPR. Audit preparation, evidence collection, gap analysis.
- **Risk Assessment**: Threat modeling, risk registers, likelihood and impact analysis, risk appetite definition.
- **Supply Chain Risk Management**: Vendor due diligence, third-party risk assessment, contractual security requirements, continuous monitoring of supplier posture.

### Identity & Access Management (IAM)

Controlling who and what can access which resources, under what conditions. NIST, ISO, Gartner, and CISA all treat IAM as a top-level security domain. It governs access to applications, infrastructure, networks, cloud platforms, and physical facilities.

#### Access Management (AM)

The runtime decisions: is this person who they claim to be, and are they allowed to do this?

**Authentication (AuthN)** verifies identity. It answers "who are you?"

- **Multi-Factor Authentication (MFA)**: Two or more independent factors: something you know (password), something you have (phone, hardware key), something you are (biometrics).
- **Single Sign-On (SSO)**: Authenticate once, access multiple systems without re-authenticating. The mechanism: a centralized authorization server maintains its own session (typically a cookie). When a user switches to a different client application, the redirect to the AS finds the existing session, so the AS issues a new token without prompting for credentials. SSO is a side effect of centralizing authentication in one server that remembers its sessions.
- **Federation**: Trusting authentication performed by another organization's identity provider. Enables SSO across organizational boundaries.
- **Identity Provider (IdP)**: A service that authenticates users and issues identity assertions. Okta, Microsoft Entra, and Google Workspace are examples.
- **Passwordless / Passkeys**: Authentication without passwords, using public-key cryptography (FIDO2/WebAuthn). The credential never leaves the device.

**Authorization (AuthZ)** determines what an authenticated identity is allowed to do. It answers "are you permitted?" See *Access Control Models* under "Access Control in Practice" for how these models work and compose.

- **Access Control List (ACL)**: A list on each resource specifying which principals have which permissions. Simple but doesn't scale.
- **Groups**: Collections of users managed centrally (often in LDAP/AD). Simplify ACL management but conflate organizational identity with permissions.
- **Role-Based Access Control (RBAC)**: Permissions assigned to roles, users assigned to roles. Separates organizational grouping from authorization.
- **Attribute-Based Access Control (ABAC)**: Decisions based on attributes of subject, resource, action, and environment. More flexible than RBAC, typically layered on top of it.
- **Principle of Least Privilege**: Granting only the minimum permissions necessary, and no more.

**Adaptive Access** adjusts authentication and authorization requirements based on risk signals: device trust, location, behavior patterns, time of day. When risk is elevated, the system can require step-up authentication.

#### Privileged Access Management (PAM)

Controls for accounts with elevated access: admin accounts, root, service accounts with broad permissions.

- **Credential Vaulting & Secrets Management**: Storing privileged credentials (passwords, API keys, certificates) in a secure vault with access controls, audit logging, and automatic rotation.
- **Session Management**: Recording, monitoring, and isolating privileged sessions. When an admin connects to a production database, the session gets logged.
- **Just-in-Time Access**: Granting elevated privileges only when needed, for a limited time. No standing admin access.

#### Identity Governance & Administration (IGA)

The "who has access to what, and should they?" problem. Where PAM protects the most powerful accounts, IGA manages the full lifecycle of all access.

- **Identity Lifecycle**: Provisioning access when someone joins, adjusting when they move roles, revoking when they leave (Joiner/Mover/Leaver). SCIM automates this across systems.
- **Access Certification**: Periodic reviews where managers confirm their team's access is still appropriate.
- **Role Management**: Defining roles, mining existing access patterns to discover roles, enforcing separation of duties (one person shouldn't both approve and process payments).

#### Identity Types

Different contexts call for different approaches to identity.

- **Workforce IAM**: Employees, contractors, partners. Managed by IT and HR, governed by corporate policy, authenticated against corporate directories.
- **Customer IAM (CIAM)**: Consumers of your product. Self-registration, social login, consent management, progressive profiling. Privacy and friction matter more here than in workforce contexts.
- **Non-Human Identities (NHI)**: Any identity that represents a workload, service, or automated process rather than a person. These vastly outnumber human identities and tend to be over-privileged. The category includes several distinct types with different security properties:
  - *Directory service accounts (LDAP/AD)*: entries in a directory that represent non-human actors. Often authenticated with a password that is shared across environments and rarely rotated. The traditional approach, and the least secure.
  - *Kubernetes service accounts*: not accounts in the directory sense, but a Kubernetes resource that provides an identity to pods. The service account token (a JWT signed by the cluster's API server) can be mounted into pods and used to authenticate to the Kubernetes API or, via workload identity federation, to external services.
  - *Cloud workload identities*: AWS IAM roles for EC2/Lambda/EKS, GCP service accounts, Azure Managed Identities. The cloud platform issues short-lived credentials to the workload based on its compute identity, eliminating the need for stored secrets.
  - *SPIFFE identities*: a platform-agnostic standard for workload identity. Each workload gets a cryptographic identity (an X.509 certificate or JWT) automatically issued and rotated by the SPIFFE runtime (SPIRE). Works across Kubernetes, VMs, and bare metal. CNCF Graduated project.
  - *API keys and static tokens*: shared secrets passed in headers. No built-in identity lifecycle, rotation, or expiry. Still common for third-party API access.

  The industry direction is toward eliminating long-lived credentials entirely. Directory service account passwords and static API keys are being replaced by short-lived, automatically rotated credentials issued by the platform (Kubernetes, cloud provider) or a workload identity system (SPIFFE). The principle: the workload proves what it is based on where it runs, rather than presenting a secret that could be copied elsewhere.

#### Identity Threat Detection & Response (ITDR)

Detecting attacks that target the identity layer: credential theft, privilege escalation, lateral movement, compromised service accounts.

#### Identity Infrastructure

The foundational systems that store and verify identity.

- **Directory Services**: LDAP (Lightweight Directory Access Protocol) is the standard protocol for querying and managing directory entries (users, groups, organizational units). Active Directory is Microsoft's implementation of an LDAP-compatible directory, adding Kerberos authentication, Group Policy, and DNS integration on top. Cloud successors like Entra ID (formerly Azure AD) and Okta Universal Directory move to REST/SCIM APIs but follow the same conceptual model. The directory is the authoritative source for who exists and what groups they belong to.
- **Identity Proofing**: Verifying that a person is who they claim to be *before* giving them an identity in the system. KYC, document verification, biometric matching. This is distinct from authentication.
- **Decentralized Identity**: Verifiable credentials, digital wallets, DIDs (Decentralized Identifiers). The user holds their own credentials rather than depending on a central IdP. The EU's eIDAS 2.0 regulation mandates digital identity wallets.

#### IAM Protocols & Standards

The wire protocols that implement the concepts above. To understand why they exist, it helps to see the progression from simple to complex.

**Simple token authentication** is the starting point. Your app has a login endpoint. The user sends their username and password, the server verifies the credentials, generates a random token, stores it, and returns it. The client sends this token on subsequent requests instead of the password. One server, one endpoint, no indirection. This is what most web frameworks provide out of the box and it works fine for a single application managing its own users.

The problems with simple token auth emerge as you scale. If you have multiple services (microservice architecture), each one would need to verify the password or share a user database. If a third-party application needs to access the user's resources, the user would have to give their password to that third party, which is unacceptable. And there's no standardization: every app invents its own login endpoint, token format, and expiry rules.

**OAuth 2.0** solves the third-party problem by separating the authorization server (where the user authenticates) from the resource server (where the data lives). The user authenticates with the authorization server, which they trust, and the third-party application never sees the password. It gets a scoped token instead. This is why OAuth has two endpoints (`/authorize` for the user-facing redirect, `/token` for the back-channel exchange) and a redirect flow: the indirection keeps the user's credentials away from the client application.

OAuth 2.0 is a delegation protocol. The resource owner grants a client limited access to their resources without sharing credentials.

- *Roles*: Resource Owner, Client, Authorization Server, Resource Server.
- *Client Types*: Confidential (can keep a secret, e.g. a server-side app) vs. Public (can't, e.g. an SPA or mobile app).
- *Grant Types*:
  - **Authorization Code**: Redirect-based. The client gets a short-lived code, then exchanges it for a token via back channel. Most secure for server-side apps.
  - **Client Credentials**: Machine-to-machine. The client authenticates directly with no user involved.
  - **Device Authorization**: For devices with limited input (TVs, CLIs). The user authorizes on a separate device.
  - **Implicit (deprecated)**: Token returned directly in the URL. Replaced by Authorization Code + PKCE.
  - **Resource Owner Password (deprecated)**: Client collects the password directly. Defeats the purpose of OAuth.

  The grant types split along a fundamental line: **is a user present?** Authorization Code (and Device Authorization) exists because a user is actively delegating access to a client application. The redirect flow, consent screen, and code exchange all exist to keep the user's credentials away from the client while letting the user choose what to share. Client Credentials exists because sometimes there is no user — a backend service needs to act on its own behalf, not on behalf of any person. The resource owner concept does not apply: the client *is* the principal, authenticating directly with the authorization server using its own credentials. This distinction determines which security mechanisms apply: user-present flows need PKCE, redirect URI validation, and consent; client credentials flows need strong client authentication (see *Client Authentication Methods* below) and careful scoping, since there is no human to approve or limit the grant.

- *Tokens*:
  - **Access Token**: Credential the client presents to the resource server. Short-lived and opaque to the client.
  - **Refresh Token**: Used to get a new access token without re-involving the user. Longer-lived.
  - **Bearer Token**: Possession alone is sufficient to use it. Must be protected in transit and storage.
- *Key Mechanisms*:
  - **Authorization Code**: A short-lived, single-use code exchanged for tokens via back channel.
  - **Redirect URI**: Pre-registered callback URL. Prevents code interception.
  - **State Parameter**: CSRF protection on the authorization request.
  - **PKCE (Proof Key for Code Exchange, RFC 7636)**: The client sends a hash of a random verifier in the auth request, then proves it knows the verifier at token exchange. Essential for public clients.
  - **Scope**: Limits what the access token allows (e.g. `read:email`).
  - **Back Channel**: Server-to-server communication, not visible to the browser. Token exchange happens here.
  - **Front Channel**: Communication via browser redirects. The auth request and code delivery happen here.
  - **Token Introspection (RFC 7662)**: The resource server asks the authorization server whether a token is valid.
  - **Token Revocation**: Explicitly invalidating a token before it expires.

**OpenID Connect (OIDC)** is an identity layer on top of OAuth 2.0. OAuth handles authorization ("can this app access your data?"), and OIDC adds authentication ("who is this user?").

- **ID Token**: A JWT containing claims about the user and the authentication event. Unlike access tokens, the client is meant to read and validate it.
- **UserInfo Endpoint**: Returns claims about the authenticated user.
- **Claims**: Name-value pairs about the user (`sub`, `email`, `name`).
- **Relying Party (RP)**: The OIDC term for the client application.

**SAML (Security Assertion Markup Language)**: An XML-based protocol for exchanging authentication and authorization data between an identity provider and a service provider. It predates OAuth and OIDC but remains common in enterprise SSO.

**FIDO2 / WebAuthn**: Passwordless authentication using public-key cryptography. The user authenticates with a hardware key or platform authenticator (fingerprint, face).

**SCIM (System for Cross-domain Identity Management)**: A standard for automating user provisioning and deprovisioning across systems.

**mTLS (Mutual TLS)**: Both client and server present certificates during the TLS handshake. Used in OAuth for certificate-bound tokens and client authentication.

#### Client Authentication Methods

When a confidential client calls the authorization server's token endpoint (to exchange an authorization code, refresh a token, or request client credentials), it must prove its identity. OAuth defines several methods, each with different security properties:

- **`client_secret_basic`**: the client sends its client ID and secret in an HTTP Basic Authorization header. The simplest method. The secret is a shared credential that both the client and AS know, so it must be stored securely on both sides and rotated periodically. If the secret leaks, anyone can impersonate the client.
- **`client_secret_post`**: the client sends the secret in the POST body instead of the header. Functionally equivalent to `client_secret_basic`.
- **`private_key_jwt`** (JWT client assertion): the client signs a short-lived JWT with its private key and sends it as a `client_assertion` parameter. The AS verifies the signature against the client's registered public key (fetched from the client's JWKS endpoint). The private key never leaves the client, so there is no shared secret to leak. This is JWT in a different role: not as an access token or ID token, but as a way for the client to authenticate itself to the AS. The assertion JWT contains `iss` and `sub` (both the client ID), `aud` (the AS token endpoint), `exp`, and `jti` claims.
- **`tls_client_auth`** (mTLS client authentication): the client presents a TLS client certificate during the connection to the token endpoint. The AS validates the certificate against a pre-registered expected value (subject DN or SAN). Combined with certificate-bound access tokens (RFC 8705), this creates an end-to-end proof-of-possession chain: the client proves its identity to the AS via mTLS, and the resulting token is bound to the same certificate.

The progression from `client_secret_basic` to `tls_client_auth` follows a consistent direction: eliminating shared secrets and moving authentication toward cryptographic proof of key possession. For high-security environments, the OAuth Security BCP (RFC 9700) recommends `private_key_jwt` or `tls_client_auth` over shared secrets.

#### How OAuth and OIDC Relate

OAuth solved the delegation problem but didn't standardize identity. The access token is associated with whatever user authorized it, and the resource server can often figure out who that user is through introspection (RFC 7662). But OAuth doesn't guarantee this. Before OIDC existed, apps would complete an OAuth flow, then call a provider-specific endpoint (Facebook's `/me`, Twitter's `/account/verify_credentials`) to discover the user's identity. Every provider did it differently, and the approach had security holes (notably token substitution attacks, where a token from a malicious app could be replayed against a different app with no way to detect it).

OIDC standardized the "who is this user" part. It shares OAuth's plumbing (same authorization server, same redirect flow, same token endpoint) and adds a few things: the UserInfo endpoint, a discovery endpoint (`.well-known/openid-configuration`), and a JWKS endpoint for signing keys. In practice you do both at once: one code exchange, but you get back an access token (OAuth, permission) and an ID token (OIDC, identity as a signed JWT with `aud` and `iss` claims that make substitution detectable).

OIDC uses OAuth scopes as its trigger mechanism. The `openid` scope tells the authorization server to issue an ID token. The `profile` and `email` scopes request specific identity claims. So the boundary between "OAuth scopes for permissions" and "OIDC for identity" is blurrier than it first appears: OIDC's identity features are activated *through* OAuth scopes.

The authorization server and the identity provider are the same server. The OIDC spec calls it an "OpenID Provider": an OAuth 2.0 authorization server that can also authenticate users and issue identity claims. Okta, Entra, Auth0 all work this way.

**The full progression**: simple token auth (one app, one login endpoint) → OAuth (separate authorization server, delegation without sharing passwords) → OIDC (standardized identity on top of OAuth). Each step solves a problem the previous step can't handle. For a single app managing its own users, simple token auth is fine. For third-party access or shared identity, you need OAuth. For knowing who the user is in a standardized way, you need OIDC.

#### Why Centralize into an Authorization Server

Even if your API has no third-party clients, centralizing authentication in a dedicated authorization server provides significant advantages. A central AS gives you: a single place to enforce MFA and credential policies, consistent login experience across all internal apps (SSO), centralized audit logging for all authentication events, and the ability to add third-party clients later without rearchitecting. Services don't implement their own authentication — they validate tokens. This is why OAuth has become popular as a standard for API security broadly, not just for the third-party delegation use case it was originally designed for.

The tradeoff is that the AS becomes a single point of failure: if it's down, nothing authenticates. One-size-fits-all session and MFA policies may not fit every service. And migrating away from a specific AS vendor (Okta, Auth0, Entra) is painful. But the alternative — each service implementing its own authentication, user database, and credential management — is worse at any meaningful scale.

The AS centralizes *authentication* and *token issuance*. Fine-grained *authorization* (can this user edit this specific document?) remains the responsibility of each service. The AS decides "this is patrik, here's a token." The service decides "patrik has the editor role, so he can edit." Use separate access tokens for each API — reusing one token across multiple APIs increases the blast radius if a token is stolen, and defeats the purpose of the `aud` claim.

#### Token Formats

An **opaque token** is a random string with no embedded meaning. The server stores the associated data and looks it up on each request. Must be validated via introspection (RFC 7662). Simple and easy to revoke (delete the record), but requires server-side state.

A **JWT (JSON Web Token)** is a self-contained token: the claims are encoded in the token itself, so the recipient can validate it without calling back to the issuer. JWTs have become the dominant token format in modern web development. The OIDC ID token is a JWT. OAuth access tokens are increasingly JWTs. Framework session tokens (Auth.js, Rails) are often JWTs stored in cookies. When a developer encounters a token in 2026, it's almost certainly a JWT.

This dominance traces to the standardization of the JWT specification in 2015, which triggered library support in every major language, which triggered framework adoption, which made JWTs the natural choice when OIDC needed a token format. The ability to decode and inspect a JWT (header tells you the algorithm, payload tells you the claims, signature ties it together) is a baseline skill.

**The JOSE standards family.** JWT is not a standalone spec. It builds on a collection of standards collectively known as JOSE (JSON Object Signing and Encryption), designed to work together:

- **JWS (JSON Web Signature, RFC 7515)**: Defines how to sign JSON objects with HMAC or digital signatures. A signed JWT is a JWS. Most JWTs in practice are signed JWTs.
- **JWE (JSON Web Encryption, RFC 7516)**: Defines how to encrypt JSON objects. An encrypted JWT (JWE) has five Base64URL-encoded parts: header, encrypted key, initialization vector, ciphertext, and authentication tag. Use JWE when claims contain sensitive data that intermediaries shouldn't read.
- **JWK (JSON Web Key, RFC 7517)**: A JSON format for cryptographic keys and key metadata. Authorization servers publish their public keys as a JWK Set (JWKS) so resource servers can verify token signatures. The JWK format also carries the algorithm associated with each key (the `alg` field), which matters for security (see below).
- **JWA (JSON Web Algorithms, RFC 7518)**: Specifies which signing and encryption algorithms can be used. Defines the algorithm identifiers (`HS256`, `RS256`, `A256GCM`, etc.) that appear in JWT headers and JWK metadata.

**Standard claims.** The JWT spec defines a set of registered claims, each a three-letter JSON property. A useful way to think about them is in two categories. **Positive claims** state facts: `iss` (issuer — who created this token), `iat` (issued-at — when), and `sub` (subject — about whom). **Constraints** limit how the token may be used: `aud` (audience — which APIs this token is intended for), `nbf` (not-before — reject if used too early), `exp` (expiry — reject if used too late), and `jti` (JWT ID — a unique identifier for replay detection).

Each constraint defends against a specific attack:
- `iss` + `aud` together prevent **reflection attacks**: a token issued for API A gets captured and replayed against API B. If API B checks that its own identifier is in the `aud` claim and that `iss` is a trusted issuer, the replayed token is rejected.
- `exp` + `nbf` bound the **replay window**. A stolen token is only useful for a limited time.
- `jti` enables **replay detection**: the recipient remembers seen token IDs and rejects duplicates. This reintroduces server-side state (you must store seen `jti` values), which partially undermines the "no server-side state" selling point of JWTs. TLS largely prevents replay attacks in transit, but `jti` matters when tokens might leak through logs, error messages, or browser history.

**The `alg` header problem.** The flexibility of JOSE is also its biggest weakness. The JWT header contains an `alg` field that tells the recipient which cryptographic algorithm was used. This means the recipient must trust an unauthenticated claim about how to authenticate the message — a circular dependency. In 2015, security researcher Tim McLean discovered that many JWT libraries allowed an attacker to change the `alg` header to `none`, instructing the library to skip signature validation entirely. Other algorithm confusion attacks exist: switching from an asymmetric algorithm (RS256) to a symmetric one (HS256) and using the public key as the HMAC secret.

The root cause is that JOSE was designed for **cryptographic agility** — the ability to change algorithms over time as weaknesses are discovered. This is a legitimate goal, but encoding the algorithm choice in the token itself is the wrong place for it. A better approach is **key-driven cryptographic agility**: store the algorithm as metadata on the key (in the JWK's `alg` field on the server), not in the token header. When you rotate keys, you can change the algorithm. The `kid` (key ID) header safely indicates which key was used; the server looks up the key and its associated algorithm. The attacker can't influence the algorithm choice because it's bound to the key on the server.

**PASETO** (Platform-Agnostic Security Tokens) is an alternative to JOSE that eliminates algorithm negotiation entirely. Each PASETO version uses a fixed set of algorithms: version 1 uses AES and RSA, version 2 uses Ed25519 and XChaCha20-Poly1305. You pick a version, you get a fixed algorithm. No `alg` header, no algorithm confusion attacks.

**Signing vs encryption.** Most JWTs are signed (JWS): the claims are base64-encoded and readable by anyone, but the signature ensures they haven't been tampered with. This provides integrity and authenticity but not confidentiality. When claims contain sensitive data (PII, internal attributes, role information), you need encrypted JWTs (JWE). Encryption alone is not enough, because many encryption modes don't protect integrity — an attacker who can't read the ciphertext may still be able to manipulate it (chosen ciphertext attacks). This is why JWE uses **authenticated encryption** algorithms (like AES-GCM or AES-CBC with HMAC) that provide both confidentiality and integrity in a single operation.

In practice, use a JWT library and pick a standard authenticated encryption algorithm (`A256GCM` or `A256CBC-HS512`). The library handles IVs, padding, authentication tags, and serialization. Don't implement the cryptographic primitives yourself. The value of understanding the underlying concepts is configuring the library correctly: choosing authenticated encryption (not bare encryption), using separate keys per purpose, never revealing decryption failure details to callers (padding oracle prevention), and validating claims (`aud`, `exp`, `iss`) after decryption.

### Application Security

Securing the software itself: how it's built, what it accepts, what it exposes.

- **Secure Development Lifecycle**: Threat modeling, static and dynamic analysis (SAST/DAST), secure code review, security testing in CI/CD.
- **Input Security**: Defending against injection (SQL, command), XSS, CSRF, SSRF. Validate at the API layer, not just the database: the database will catch some invalid input through schema constraints, but sending garbage to it wastes resources and some business rules can't be expressed in a schema. When validating, prefer **allowlists** over denylists: define what acceptable input looks like (e.g. "alphanumeric, 1-30 characters") and reject everything else, rather than trying to enumerate every possible malicious input. For SQL injection specifically, **prepared statements** (parameterized queries) are effectively a complete solution: the database receives the query structure and user-supplied values separately, so input can never be interpreted as SQL. The only cases where prepared statements don't help are dynamic identifiers (table names, column names, sort order) that can't be parameterized; those must be validated against an allowlist. As a defense-in-depth measure, the database user that the API connects with should have minimal privileges: if the API only reads and writes rows, the database user should only have SELECT, INSERT, UPDATE, DELETE on the specific tables it needs. No DROP, ALTER, or GRANT. Even if an injection slips through, the damage is bounded by what the database user is allowed to do. **Row-level security** policies take this further: the database itself enforces which rows a query can see, independent of the application logic. This means even a compromised application can only access data the database policy allows for that user context.
- **Output Security**: Securing what the API sends back. Don't echo user input in error responses: even valid JSON can become a reflected XSS vector if the browser interprets the response as HTML. Use a proper serialization library, never string concatenation, and return generic error messages rather than reflecting invalid input. Set **security headers** on all API responses: always specify `Content-Type` explicitly (never let the framework guess), add `X-Content-Type-Options: nosniff` (prevents the browser from guessing content type), set a `Content-Security-Policy` (restricts what scripts can run even if XSS succeeds), and use cache control headers to prevent sensitive responses from being cached by browsers or proxies.
- **API Security**: API gateways, authentication and token validation at the API layer, schema validation. **Application-level DoS** (layer 7) attacks send syntactically valid requests at high volume to overwhelm your API. Unlike network-level DDoS, this traffic looks like legitimate requests, so firewalls can't filter it. The defense is rate limiting as early as possible in the request path, ideally at the load balancer or reverse proxy before requests even reach your API servers. Excess requests get rejected with a 429 status rather than crashing the backend under load. As defense in depth, also enforce rate limits in each API server itself: if the proxy misbehaves or is misconfigured, individual servers should still be hard to bring down. Genuine clients can retry later. Rate limiting should be the very first check, before authentication, because authentication itself can be expensive (cryptographic verification, database lookups). Never let unauthenticated requests consume significant server resources.
- **Software Supply Chain**: Managing dependency risk. SBOMs, dependency scanning, artifact signing, lock files.

### Network Security

Securing communication paths and network infrastructure.

- **Transport Security**: **TLS (Transport Layer Security)** provides two things: server authentication and encryption. Every HTTPS connection is an authentication event — the client verifies the server's identity before any data is exchanged. TLS sits on top of TCP/IP and is largely transparent to the application layer. HTTPS is just HTTP over TLS. (SSL is the old name; TLS replaced it, but "SSL" still shows up in terminology like "SSL termination.") In a TLS handshake, the server presents a certificate signed by a trusted Certificate Authority (CA), proving it is who it claims to be. The client verifies the certificate matches the hostname it intended to connect to, then both sides negotiate encryption keys for the session. Without TLS, an attacker on the network can read passwords, tokens, and data in transit, and can impersonate the server. **mTLS (mutual TLS)** extends authentication to the client side: both sides present and verify certificates during the TLS handshake. Standard TLS authenticates the server; mTLS makes both directions authenticated. This is the standard for service-to-service communication in microservice architectures and service meshes, where both ends need to prove their identity. **SSL termination** is when a load balancer or reverse proxy handles TLS decryption so that backend servers receive plain HTTP. The proxy-to-backend connection may be unencrypted (within a trusted network) or re-encrypted (TLS re-encryption). Certificate management (issuance, renewal, revocation) and certificate transparency (public logging of all issued certificates) round out the operational side. In practice, **public key infrastructure (PKI)** — the processes for creating, distributing, and revoking certificates — is complex to operate manually; **service meshes** (Linkerd, Istio) automate PKI and mTLS for microservice deployments. See *Service-to-Service Authentication* under "In Practice" for details.
- **Perimeter & Segmentation**: The traditional model is perimeter security: put services on a corporate network, require employees to connect via VPN if remote, and make internal services unreachable from the public internet. This significantly reduces the attack surface. External attackers can't reach internal APIs, and network-level DoS against internal services is much harder. But the perimeter model assumes everything inside the network is safe, which breaks the moment someone gets in: a compromised employee laptop, a malicious insider, or a stolen VPN credential puts the attacker inside with access to everything. This is why zero trust says internal services should authenticate, authorize, and rate-limit as if they were public. The network perimeter is still useful as one layer, but it shouldn't be the only layer. In practice: firewalls at the edge, VPNs for remote access, microsegmentation and network policies between internal services, and zero trust pushing verification down to the individual workload level.
- **DDoS Protection**: Distributed Denial of Service attacks at the network level (layer 3/4) flood the target with raw traffic. A common technique is DNS amplification: the attacker sends small queries to many DNS servers with the victim's IP as the spoofed return address, and the victim gets flooded with the much larger replies. These volumetric attacks are handled by network infrastructure, not application code: firewalls, DDoS protection services (Cloudflare, AWS Shield), and sufficient network capacity to absorb the load.
- **DNS Security**: DNSSEC, DNS filtering, protection against DNS hijacking and poisoning.

### Endpoint Security

Securing the devices that people and workloads run on.

- **Endpoint Detection & Response (EDR/XDR)**: Monitoring endpoints for malicious activity, with investigation and response capabilities.
- **Device Management**: MDM (Mobile Device Management), UEM (Unified Endpoint Management), device hardening, patch enforcement.
- **Malware Defenses**: Antivirus, anti-malware, behavioral detection.

### Data Security

Protecting information at rest, in transit, and in use.

- **Encryption**: At rest (disk, database), in transit (TLS), key management (KMS, HSMs), key rotation.
- **Data Classification**: Labeling data by sensitivity: public, internal, confidential, restricted. Classification drives handling requirements.
- **Data Loss Prevention (DLP)**: Controls to prevent sensitive data from leaving the organization. Email scanning, endpoint monitoring, cloud DLP.
- **Privacy & PII Protection**: Controls for personal data specifically. Consent, minimization, retention limits, right to deletion. Driven by regulations like GDPR and CCPA.

### Infrastructure & Cloud Security

Securing the platforms applications run on.

- **Cloud Security Posture Management (CSPM)**: Continuously scanning cloud configurations for misconfigurations and policy violations.
- **Container & Runtime Security**: Image scanning, runtime policies, admission controllers, supply chain verification for container images. **Container hardening**: running as a non-root user, dropping unnecessary Linux capabilities, using minimal base images (distroless/Alpine), read-only root filesystems, pod security policies or admission controllers to enforce these constraints cluster-wide.
- **Configuration Management**: Hardening baselines, drift detection, Infrastructure as Code (IaC) scanning, CIS Benchmarks.

### Security Operations

The day-to-day work of defending the organization.

- **Asset Management**: Knowing what you have. Inventory of hardware, software, cloud resources, data stores. You can't protect what you don't know exists.
- **Vulnerability Management**: Scanning for vulnerabilities, prioritizing by risk (CVSS), patching, tracking remediation.
- **Monitoring & Logging**: Two kinds of logging serve different purposes. **Observability logging** helps you debug and monitor: request latencies, error rates, connection pool exhaustion. It's operational, the audience is developers and SREs, and entries can be sampled or rotated to save storage. Retention is weeks to months. **Audit logging** is a security and legal record: who did what, when, from where. It's for compliance, incident investigation, and accountability. Entries must never be dropped, sampled, or modified. Retention is months to years, often mandated by regulation (SOC 2, HIPAA, GDPR). Access should be restricted to auditors, not the administrators being monitored. Both types often flow through the same infrastructure (SIEM, centralized log aggregation) but have different guarantees around completeness, immutability, and access control.
- **Threat Detection & Intelligence**: IDS/IPS, anomaly detection, behavioral analytics, threat intelligence feeds.
- **Incident Response**: Playbooks, triage, containment, forensics, post-mortems, communication plans.

### Resilience & Recovery

Surviving incidents and recovering from them.

- **Business Continuity Planning**: How the organization keeps operating during a disruption. Alternate processes, communication plans, critical function prioritization.
- **Disaster Recovery**: Technical recovery of systems and data after a major incident. RTO (Recovery Time Objective) and RPO (Recovery Point Objective) define the targets.
- **Backup & Data Recovery**: Regular backups, tested restores, offsite and offline copies, immutable backups to protect against ransomware.

### Security Awareness & Training

Phishing simulations, role-specific training (developers get secure coding, executives get social engineering awareness), new-hire onboarding, ongoing reinforcement.

### Security Assurance & Testing

Validating that controls actually work. This is a cross-cutting practice that tests every other domain.

- **Penetration Testing**: Simulated attacks against specific targets (applications, networks, cloud) to find exploitable vulnerabilities.
- **Red Teaming**: Adversary simulation across the full kill chain. Tests detection and response, not just technical controls.
- **Bug Bounties**: Crowdsourced vulnerability discovery. External researchers find and report issues for rewards.

---

## Overloaded Terms

**Client**: In web development, "client" means the browser or frontend. In OAuth, "client" is the application (typically your backend) that requests access tokens and calls resource servers. In a modern web app, these are almost always different things.

**Server**: Ambiguous in OAuth because there are three: the authorization server, the resource server, and the client's own backend (which is the OAuth "client"). Always qualify which one.

**Token**: Could mean an OAuth access token, a refresh token, an OIDC ID token, a session token or cookie, a CSRF token, or a JWT. Each serves a different purpose.

**Authorization**: In OAuth, the resource owner granting the client permission to access their resources. In general IAM, determining whether an authenticated principal may perform an action. Related but at different granularities. To add to the confusion, HTTP gets its own terminology wrong: the `Authorization` header carries *authentication* credentials, the `WWW-Authenticate` challenge is returned with a 401 status code named "Unauthorized" which actually means *unauthenticated*. Meanwhile, 403 Forbidden is the actual *authorization* failure (you're authenticated, but not permitted). The HTTP spec treats the two concepts as interchangeable, which they are not.

**Service Account**: At least four different things depending on context. In LDAP/Active Directory, a directory entry representing a non-human actor, often with a password that rarely rotates. In Kubernetes, a resource that provides an identity to pods — not an account in the traditional sense but a mechanism for mounting tokens and binding RBAC policies. In cloud IAM (GCP, AWS), an identity for a compute workload that can assume roles and access cloud APIs. In OAuth, a client registered for the Client Credentials Grant. The common thread is "non-human identity," but the implementation, lifecycle, and security properties differ completely across these contexts.

**JWT**: Plays at least three distinct roles. As an **access token** — the credential a client presents to a resource server (the most common mental model). As an **ID token** — the OIDC identity assertion the client reads to identify the user (always a JWT by spec). As a **client assertion** — how a confidential client proves its identity to the authorization server during token exchange (`private_key_jwt`), replacing a client secret with a signed JWT. Same format, fundamentally different purposes.

## In Practice

How the concepts in this document play out in a modern deployment.

### Web Application Architecture

Modern web apps are typically built with hybrid frameworks (Next.js, SvelteKit, Nuxt, Remix) that render pages on the server and hydrate into interactive frontends in the browser. Server components handle data fetching and secrets, client components handle interactivity. Pure SPAs (a JavaScript bundle talking to a separate API, with no server of its own) have fallen out of favor because of SEO, performance, and the security problems of being a fully public OAuth client. Pure server-rendered apps without JavaScript interactivity are niche (HTMX, Hotwire).

Because these frameworks have a server backend, they act as **confidential OAuth clients** using the **Authorization Code Grant**. The server handles the code exchange and token storage. The browser never sees the access token.

This pattern is sometimes called **BFF (Backend for Frontend)**: your server is a thin proxy that holds tokens and forwards requests. The browser gets a session cookie from your app, and your server makes API calls with the access token on the user's behalf.

#### Role Mapping

| Component | Web dev term | OAuth role | Notes |
|---|---|---|---|
| Browser / frontend | "the client" | User-agent | The resource owner interacts through it. Not an OAuth role. |
| Your backend / BFF | "the server" | **Client** (confidential) | Holds client secrets, manages tokens, proxies API calls. |
| External APIs (Stripe, GitHub, etc.) | "APIs", "services" | **Resource server(s)** | Some use OAuth tokens, others use API keys. |
| Auth provider (Auth0, Okta, your own) | "auth", "login" | **Authorization server** | Issues tokens. May also be the OIDC identity provider. |
| The end user | "user" | **Resource owner** | The human who grants or denies access. |

The backend is the center of trust. It authenticates the browser via a **session cookie** (plain HTTP session management, unrelated to OAuth) and holds access tokens for each external API. The browser never talks to external APIs or the authorization server directly, except during the initial redirect-based authorization flow.

There are two separate trust relationships here, operating at two separate hops:

```
              Session management                     Hop 2 depends on destination
              (cookie, bearer token,
               or JWT)
                                          ┌─ mTLS ──→ Internal service
Browser ←─────────────────────────→ BFF ──┤            (mesh, automatic)
  "which session is this?"                │
                                          ├─ OAuth ──→ Auth0
                                          │            (authorization code grant)
                                          │
                                          └─ Bearer ─→ GitHub, Slack
                                                       (token from OAuth or Vault)
```

Session management (hop 1) and the backend's outbound authentication (hop 2) are not alternatives — they solve different problems at different boundaries. The session mechanism keeps the browser logged in to your backend. Hop 2 depends on where the BFF is calling: internal services use mTLS via the mesh (no tokens, no OAuth), external identity providers use OAuth, and external APIs use bearer tokens or API keys. The browser is unaware of what happens on hop 2 — it only ever sees its session cookie.

A simple app with its own user database uses session management but no OAuth at all — OAuth enters the picture only when a separate authorization server or third-party API is involved. For internal services behind a service mesh, the BFF just makes a plain HTTP call and the mesh handles authentication transparently.

#### Multiple External APIs

A real backend usually talks to several external services: a payment processor, a third-party data API, a cloud storage provider, and so on. Some of these use OAuth tokens (your backend does an OAuth flow to get a token for the GitHub API). Others use plain API keys (Stripe). The backend manages all of it: different credentials for different resource servers, all invisible to the browser.

### Zero Trust

The dominant security architecture model. The core idea is "never trust, always verify": every request is authenticated and authorized regardless of network location. Being inside the corporate network no longer grants implicit trust.

Traditional security drew a perimeter (firewall, VPN) and trusted everything inside it. Zero trust removes that assumption. Instead, identity becomes the primary control plane. Every access request is evaluated based on who is asking, what they're asking for, from what device, and under what conditions.

NIST SP 800-207 defines the architecture. CISA's Zero Trust Maturity Model breaks it into five pillars: Identity, Devices, Networks, Applications & Workloads, and Data. Identity is the first pillar because without strong identity, none of the other pillars work.

Zero trust is not a product you buy. It's a design philosophy that cuts across IAM, network security, endpoint security, and data security. Implementing it is a gradual process of removing implicit trust and replacing it with explicit verification.

### User-Facing Authentication

Authentication in a web app comes down to one problem: HTTP is stateless, so the server has no built-in way to know that request #2 comes from the same person as request #1. After the initial login (whether via a simple login endpoint, OAuth, or any other mechanism), every subsequent request needs to carry a token that proves the session. There are three approaches, each removing a dependency from the previous one.

#### Approach 1: Session Cookies

The server stores session state, the browser gets a cookie containing the session ID. Two halves of one mechanism: the session is the data, the cookie is the delivery.

**Cookies** are the transport. The server sends a `Set-Cookie` header, the browser stores the cookie and attaches it to every subsequent request to the same domain automatically. **Sessions** are the server-side state: a record like "session abc123 belongs to patrik, created at 10:42, expires in 8 hours." The browser never sees what's in the session. It only holds a random string that means nothing without the server's lookup table.

Session storage varies by deployment. Most frameworks default to **in-memory** storage (fast, zero configuration, but sessions are lost on restart and can't be shared across server instances). For production with multiple servers behind a load balancer, sessions move to a shared store: **Redis or Memcached** (fast, shared, the most common production choice), a **database** like Postgres (durable, queryable, slower but you already have one), or a distributed cache. Client-side sessions (encrypted JWTs in cookies, as used by Rails and NextAuth) eliminate the store entirely at the cost of harder revocation and a 4KB size limit.

Best for first-party browser apps where the API and frontend are on the same origin. Limitations: cookies are scoped to a domain, so they don't work well for cross-origin APIs, mobile clients, or third-party consumers. Cookies are also a browser mechanism; non-browser clients (mobile apps, CLIs, backend services) can technically handle cookies but rarely do.

#### Approach 2: Bearer Tokens (Opaque)

The server issues an opaque token (a random string), the client stores it and explicitly attaches it in an `Authorization: Bearer <token>` header (or a custom header like `X-Auth-Token`) on each request. The `Bearer` scheme was originally designed for OAuth 2.0 but is now widely used for all kinds of tokens. The server looks up the token in a token store (database, Redis) to validate it. This is the same data structure as a session store (key-value lookup indexed by a random string), just different terminology: "session store" implies cookies and rich server-side data, "token store" implies bearer tokens and API clients. In the BFF pattern, the session store acts as both: the cookie gets you to the session, and the session holds the OAuth tokens.

No cookies involved, so it works for any client: browsers, mobile apps, CLIs, other services. The tradeoff: the client is responsible for storing and sending the token (no automatic browser behavior), and the server still needs a store for lookups.

When used in a browser, the frontend JavaScript stores the token (in localStorage, sessionStorage, or a variable) and attaches it to each request. This makes the token accessible to any JavaScript running on the page, which means an XSS vulnerability can steal it.

Tokens stored server-side should be **hashed** before writing to the database, just like passwords. If the database is compromised, raw tokens can be used directly, but hashed tokens can't. Simple hashing prevents token theft but not forgery: an attacker with write access could insert a fake token record. **HMAC** (Hash-based Message Authentication Code) solves this by incorporating a server-side secret key into the hash. The server computes the HMAC when creating and verifying tokens, so an attacker who can write to the database but doesn't know the secret can't forge valid tokens. HMAC also avoids timing attacks on database lookups.

#### Framework Secrets

Most web frameworks require a master secret: Auth.js has `AUTH_SECRET`, Rails has `secret_key_base`, Django has `SECRET_KEY`. These serve the same purpose: a single root secret from which per-purpose cryptographic keys are derived via HKDF (HMAC-based Key Derivation Function). Different cookies, tokens, and CSRF protections get different derived keys even though they all come from the same root secret.

What the derived keys protect varies by framework. Auth.js **encrypts** its session JWTs by default (A256CBC-HS512, providing both confidentiality and integrity), so the payload is hidden from the browser. Rails and Django **sign** their session cookies by default (HMAC), so the payload is readable but tamper-proof. In all cases, without the secret, anyone could forge valid session tokens for any user. These secrets must never be committed to version control and should be injected via environment variables.

#### Approach 3: Self-Contained Tokens (JWTs)

Same as bearer tokens, but the token itself contains the claims, signed and optionally encrypted. The server validates the token by checking the signature (or decrypting) rather than looking it up in a database. No server-side state needed per token.

The tradeoff is revocation. With server-side state (approaches 1 and 2), you delete the record and the token is instantly invalid. With a self-contained token, there is no record to delete. The token is valid until it expires. There are several strategies, each reintroducing some server-side state:

- **No revocation**: rely on short expiry and accept that tokens can't be invalidated early. Almost never acceptable — few APIs handle no sensitive data and can tolerate users being unable to log out.
- **Allowlist**: store every valid token's `jti` in a database. To revoke, delete the record. To validate, check the record exists. This is the simplest and most secure default, though it means every token validation requires a database lookup (which is also true for opaque tokens).
- **Blocklist**: only store revoked token IDs. Smaller table than an allowlist, but entries must persist until the token would have expired anyway. Short token lifetimes keep the list small.
- **Attribute-based revocation**: rather than blocking individual tokens, block by attributes — "invalidate all tokens for user Mary issued before Friday lunchtime." This handles mass revocation without adding millions of rows. When Facebook discovered a vulnerability in September 2018 that exposed 50 million accounts, they had to revoke 90 million tokens. An allowlist would have required deleting 90 million records; an attribute-based approach records a single rule.
- **Short-lived + refresh tokens**: issue access tokens with very short expiry (5-15 minutes) and pair them with longer-lived refresh tokens stored server-side. The access token can't be revoked, but the damage window is small. Revocation happens at the refresh token: when the user logs out or their access is revoked, the refresh token is deleted and no new access tokens are issued. This is the standard OAuth pattern.

The allowlist and blocklist approaches reintroduce server-side state, which seems to defeat the purpose of self-contained tokens. But a hybrid is often the right answer: the JWT carries claims so the server can make some decisions without a database hit (show which groups a user belongs to, display unread counts), while the database provides instant revocation. Token attributes can be split between the JWT and the database depending on sensitivity and how often they change — basic identity claims in the JWT, revocation status and mutable permissions in the database.

Self-contained tokens also have size limits when stored in cookies (4KB per RFC 6265).

#### The Progression

Each approach removes a dependency. Session cookies depend on browser behavior *and* server-side state. Opaque bearer tokens remove the browser dependency but keep server-side state. Self-contained JWTs remove both, at the cost of harder revocation. The right choice depends on your clients (browser-only vs multi-platform), your infrastructure (can you run a session store?), and your revocation requirements.

In practice, the progression is not linear. The industry hasn't picked one approach; it's landed on hybrids. You start with server-side state, move to stateless JWTs for scalability, then add server-side state back for revocation. Traditional web frameworks (Django, Laravel, Express) default to session cookies. API-first and serverless architectures tend toward JWTs. But many production setups use short-lived JWTs for API access paired with server-side refresh token storage for revocation, getting scalability on the hot path (token validation) and revocability where it matters (session lifecycle). The question is not "which approach" but how much server-side state to add back and for which operations.

#### Cookies vs Tokens in the Browser: Security Tradeoffs

For browser-based apps, the choice between cookies and bearer tokens in JavaScript comes down to which attack vector you'd rather defend against:

- **Cookies** (`HttpOnly` + `Secure` + `SameSite=Lax`): immune to token theft via XSS because JavaScript can't read the cookie. But the browser sends cookies automatically on every request to that domain, which historically enabled CSRF (a malicious site tricks the browser into making an authenticated request). `SameSite=Lax` (now the browser default) largely solves this by blocking cookies on cross-site POST requests.
- **Bearer tokens in JavaScript**: immune to CSRF because the browser doesn't attach them automatically, so a malicious site can't trigger an authenticated request. But any JavaScript running on the page can read the token, so a single XSS vulnerability can steal it.

XSS is generally considered harder to fully prevent than CSRF (one missed input validation and you're exposed), and `SameSite` has made CSRF largely a solved problem. For browser apps, cookies with `HttpOnly` + `Secure` + `SameSite=Lax` are the more defensive choice. This is what the BFF pattern uses: the backend sets a session cookie, the browser sends it automatically, and the backend holds the OAuth tokens server-side.

Bearer tokens in headers are the right choice for non-browser clients (mobile apps, CLIs, service-to-service) where there's no browser to manage cookies and CSRF doesn't exist.

**In short: cookies for browsers, bearer tokens for everything else.**

#### The Full Login-to-Expiry Lifecycle

Regardless of which token approach you use, the login flow with OAuth/OIDC looks the same:

```
1. User clicks "Log in"
   Browser redirects to authorization server (front channel)

2. User authenticates with the authorization server
   (password, fingerprint, whatever)

3. Authorization server redirects back to your backend with a code
   (front channel, browser is just a courier)

4. Your backend exchanges the code for tokens (back channel, server-to-server)
   Gets back:
     - ID token (JWT): "this is patrik"              ← OIDC
     - Access token: "can access these resources"     ← OAuth
     - Refresh token: "use this to get new access tokens" ← OAuth

5. Your backend reads the ID token, establishes a session
   (cookie, opaque token, or JWT depending on approach)

6. Subsequent requests carry the session token
   Backend identifies the user, finds access tokens, calls external APIs

7. Access token expires (say after 1 hour)
   Backend uses refresh token to get a new one (back channel)
   User doesn't notice.

8. Session expires (say after 8 hours)
   Next request: backend says "session gone, log in again"
   Back to step 1.
```

The JWT ID token from OIDC populates the session once at login, then its job is done. The OAuth access tokens live inside the session (however the session is stored). Session lifetime and access token lifetime are independent: an access token might expire in 1 hour while the session lasts 8, with the refresh token silently renewing access tokens in between.

Common session expiry patterns: fixed expiry (30 minutes for banking, 30 days for social media), sliding expiry (resets on every request, dies from inactivity), or both combined (sliding window of 30 minutes plus an absolute maximum of 8 hours).

#### Cookie Security

When using session cookies (approach 1), the cookie contains just the session ID: a random string like `sess_a8f29c3b7d4e`. No user data, no tokens, no claims. But it is sensitive because it's a *key* to all of that. Whoever has that string can send it to your server and become that user's session. Stealing a session cookie is equivalent to stealing a login.

Key attributes that protect it:

- `HttpOnly`: JavaScript can't read the cookie. If an attacker injects a script via XSS, they can't extract the session ID and send it to their own server. Note that `HttpOnly` prevents *exfiltration*, not *exploitation*: an XSS attacker can still make authenticated requests from the victim's browser, since the cookie is attached automatically. But they can't steal the cookie for use elsewhere.
- `Secure`: The cookie is only sent over HTTPS, never plain HTTP. Prevents interception on the wire.
- `SameSite`: Controls whether the cookie is sent on cross-site requests. `Strict` means the cookie is never sent from other sites (strongest, but breaks OAuth redirects since the redirect back from the authorization server is cross-site). `Lax` sends the cookie on top-level GET navigations (clicking a link, following a redirect) but blocks it on cross-site subresource requests (images, scripts, iframes) and cross-site POST requests. `None` sends it everywhere (requires `Secure`, used when cross-site cookies are genuinely needed).
- `__Host-` prefix: A cookie named `__Host-session` must be set with `Secure`, must not have a `Domain` attribute, and must have `Path=/`. This prevents subdomain attacks and ensures the cookie is bound to the exact host. OWASP recommends this for session cookies.

#### What the Browser Sees vs What the Backend Sees

```
Browser ──cookie or token──> Your Backend ──API key──────> Stripe
                                          ──Bearer token──> GitHub API
                                          ──mTLS──────────> Internal service
                                          ──Bearer token──> Google Calendar API
```

The left side (browser/client to backend) uses session cookies or bearer tokens. The right side (backend to external APIs) uses OAuth access tokens or API keys. These are separate worlds. The session is the bridge: it maps the client's session token to the collection of credentials your backend needs to call external services on that user's behalf.

#### What the Service Mesh Sees

Within a cluster, the picture is different. Sidecar proxies (the service mesh data plane) intercept all traffic. The application code makes plain HTTP calls; the proxies upgrade them to mTLS. Identity comes from certificates issued by the mesh CA, not from tokens in headers.

```
                             ┌── Cluster ────────────────────────────────┐
                             │                                           │
Browser ══HTTPS══> Ingress ──┤──> [proxy] Service A [proxy] ══mTLS══     │
                   Controller│        │                          │       │
                             │        └══mTLS══> [proxy] Service B       │
                             │                                           │
                             │    [proxy] Service C ══mTLS══> [proxy]    │
                             │           Database                        │
                             └───────────────────────────────────────────┘
```

The ingress controller terminates external TLS and forwards traffic into the mesh. From there, every hop is mTLS — authenticated at the transport layer, encrypted, and identity-verified by the mesh before application code runs. Network policies further restrict which services can communicate at all. See *Service-to-Service Authentication* below for details.

#### CORS (Cross-Origin Resource Sharing)

Browsers enforce a **same-origin policy**: JavaScript on `app.example.com` cannot make requests to `api.otherdomain.com` by default. This is a security boundary that prevents a malicious page from making authenticated requests to your API using the user's cookies.

CORS is the mechanism for relaxing this boundary in a controlled way. Your API sends response headers telling the browser which origins are allowed to call it, which HTTP methods they can use, and whether cookies should be included. For non-simple requests (anything beyond basic GET/POST with standard headers), the browser sends a **preflight request** (an OPTIONS request) first to check whether the actual request is permitted.

CORS matters for API security because misconfigured CORS headers can expose your API to cross-origin attacks. A permissive `Access-Control-Allow-Origin: *` combined with `Access-Control-Allow-Credentials: true` would let any website make authenticated requests to your API using the victim's existing session cookie. The attacker doesn't need to authenticate: the victim's browser already has valid credentials and sends them automatically based on the destination domain, not the page the user is on. This is essentially CSRF enabled by permissive CORS. The most realistic scenario isn't an external attacker probing your APIs, but an XSS vulnerability in one of your own apps being used to make cross-origin requests to another of your own services.

`SameSite` cookies and CORS work together as defenses. `SameSite=Lax` prevents the cookie from being sent on cross-site subresource requests (so the request arrives unauthenticated). Strict CORS headers prevent the browser from allowing the cross-origin call in the first place. You want both.

In the BFF pattern, CORS is simpler because the browser only talks to its own backend (same origin), and the backend handles all cross-origin API calls server-side where CORS doesn't apply. This removes an entire class of cross-origin attacks from the picture.

### Access Control in Practice

Three layers handle access control in a modern web app, each answering a different question. This is a practical mental model, not a named industry standard, but it captures how the pieces divide up in practice.

**Layer 1: OAuth scopes** answer "what can this *app* do on behalf of a user?" Scopes are fundamentally about delegation: the user grants a third-party app a subset of their own access. This is discretionary access control (DAC) — the user decides how much to delegate. By contrast, the permissions in Layer 3 are mandatory access control (MAC) — a central authority (the API owner) decides who can do what, and users have no control over their own permission level. Scopes and permissions serve different purposes and are designed differently: permissions reflect organizational policies ("employees in this role can read all documents on the shared drive"), while scopes anticipate how users might delegate to third parties ("this app wants to read your repos and manage your gists, allow?").

Scopes set a delegation ceiling. Every user who grants `read:repos` to your app grants the same *scope*, but the effective access differs per user: a user with 2 repos and a user with 200 repos both grant `read:repos`, but the app sees very different data. The actual access is the intersection of the granted scope and the user's own privileges. A scope can't grant powers the user doesn't have.

**Layer 2: OIDC** answers "who is this *user*?" The ID token says "this is patrik@example.com" and may include claims like `email_verified: true`. Your app uses this to create a session. The ID token is for the client application to identify the user — it should not be used as an access token to call APIs. Access tokens are for API access; ID tokens are for establishing identity at the client. The OIDC standard doesn't define role or permission claims, though tokens can carry custom claims (and often do in practice, as we'll see below).

**Layer 3: Your app's authorization** answers "what can this *user* do in *this app*?" This is where your access control model lives. It's internal to your application, not part of OAuth or OIDC. When patrik makes a request, your app checks the session (knows it's patrik via OIDC), looks up his permissions (from your database), and decides whether to allow the action. OAuth and OIDC are not involved in this decision.

The rest of this section covers how Layer 3 works internally: the access control models, how they compose, and the architectural patterns around them.

#### Access Control Models

Access control models form a progression, each solving problems the previous one can't handle. In practice you layer them rather than choosing one.

**ACLs (Access Control Lists)** are the simplest model: each resource has a list of entries specifying which principals have which permissions. A user's effective access is determined by finding their entry on the resource's list. ACLs are straightforward for small systems but don't scale: if you have a million users and a million resources, you could end up with a billion ACL entries. And if permissions aren't cleaned up when they're no longer needed, users accumulate privileges over time, violating least privilege.

**Groups** simplify ACL management by collecting users into named sets. Instead of granting permissions to individual users, you grant them to groups: all members of the `engineering` group get the same access. A user can belong to many groups, and groups can contain other groups, creating hierarchical structures (the `project-managers` group is a member of the `employees` group).

Groups are typically managed centrally in a directory service like LDAP/AD, where they reflect organizational structure: departments, teams, projects. LDAP defines two forms of groups: *static groups* (`groupOfNames` or `groupOfUniqueNames`) that explicitly list their members, and *dynamic groups* (`groupOfURLs`) whose membership is defined by search queries against the directory — any entry matching the query is automatically a member.

The fundamental problem with groups is that they conflate two concerns: *who someone is* (organizational membership) and *what they can do* (permission grants). When multiple groups independently grant the same permission, revoking access for a specific user becomes an archaeology problem. Say Alice has access to the billing dashboard through three groups: `finance-team` (her department), `quarterly-reviewers` (a cross-functional group), and `billing-admins` (added during an incident, never removed). If Alice changes teams and you remove her from `finance-team`, she still has billing access via the other two. To fully revoke access, you need to discover every group that grants it, and removing her from those groups may affect other permissions she legitimately needs. There is no clean place to express "this specific user should not have this specific permission" — you would need per-user deny rules, which introduce precedence complexity and make the system non-monotonic (adding a group membership could *reduce* access, violating expectations).

This is the **effective permissions problem**: a user's actual access is the union of all permissions from all their groups (including nested groups), and this union is expensive to compute, hard to audit, and difficult to safely modify.

**RBAC (Role-Based Access Control)** addresses the group problem by introducing roles as an explicit intermediary between users and permissions. Instead of assigning permissions directly to users or groups, permissions are assigned to roles, and users are assigned to roles. The key separation:

- **Groups** answer "who is this person?" — they reflect organizational reality (department, team, project). They're managed centrally, often in LDAP/AD, and tend to be shared across the organization.
- **Roles** answer "what can this person do in this application?" — they map to permissions within a specific application. They're managed by the application itself and are scoped to that application's domain.

A `moderator` role collects the permissions "can read posts" and "can delete offensive posts." Assigning someone the moderator role grants exactly those permissions, and revoking the role revokes exactly those permissions — a single point of control rather than the group archaeology problem. If the permissions associated with moderation change over time, you update the role definition, and all moderators get the updated permissions without touching individual assignments.

RBAC systems typically don't allow permissions to be assigned to individual users, only to roles. This restriction simplifies auditing: reviewing who has access means reviewing role assignments and role definitions, not scanning every individual permission grant. It also enables separation of duties — defining and assigning permissions to roles is a different responsibility from assigning roles to users.

Groups and roles work together: group membership can inform role assignment ("everyone in the `engineering` LDAP group gets the `developer` role in this app"), but the group itself doesn't directly carry application permissions. The bridge between them is a mapping layer, not identity.

Role assignments can be **static** (an admin assigns users to roles, stored in a database) or **dynamic** (rules determine role activation based on context — a call center worker gets the `customer-support` role only during their contracted shift hours). The NIST RBAC model includes a concept of **sessions** where a user can activate only a subset of their roles at a time, similar to scoped tokens in OAuth. This limits the blast radius if a session is compromised: the user acts with only the authority they actually need.

RBAC's weakness is **role explosion**: as the number of distinct access patterns grows, you end up with hundreds of fine-grained roles (`billing-reader-us-east`, `billing-reader-eu-west`) that are nearly as hard to manage as per-user permissions.

**ABAC (Attribute-Based Access Control)** handles policies that RBAC can't express through simple role assignments. In ABAC, access control decisions are made dynamically for each request based on attributes in four categories:

- **Subject attributes**: who is making the request — username, groups, department, clearance level, how they authenticated.
- **Resource attributes**: what is being accessed — the resource's URI, classification label, owner.
- **Action attributes**: what they're trying to do — the HTTP method, the operation type.
- **Environment attributes**: the context — time of day, IP address, device trust level.

The output is a permit or deny decision. For example: "deny moderators from deleting posts outside office hours" checks the action (DELETE), the subject (role = moderator), and the environment (time of day), and denies if the time falls outside 9-to-5.

When multiple ABAC rules match a request and produce conflicting decisions, you need a **combining strategy**. The safest default is **deny overrides**: start with a default decision (permit or deny), and any explicit deny from any rule overrides all permits. When layering ABAC on top of existing RBAC, the practical approach is **default-permit with deny overrides**: if RBAC already approved the request, ABAC rules can only add restrictions, never grant access that RBAC denied. This means the two systems compose cleanly — RBAC grants, ABAC restricts — and a broken ABAC rule can't accidentally open access.

The industry reference architecture for ABAC is **XACML**, which separates four components:

- **PEP (Policy Enforcement Point)**: intercepts requests and rejects anything denied by policy. This is the middleware or filter in your API.
- **PDP (Policy Decision Point)**: evaluates the access control rules and returns permit/deny. This is the rules engine.
- **PIP (Policy Information Point)**: retrieves and caches attribute values from external sources (directories, databases, user info endpoints).
- **PAP (Policy Administration Point)**: the interface for defining and managing policies.

These components can be collocated or distributed. Modern implementations like **Open Policy Agent (OPA)** and **AWS Cedar** fill the PDP role with developer-friendly policy languages, replacing XACML's verbose XML. OPA uses Rego, Cedar uses a purpose-built language — both externalize authorization logic from application code entirely, separating the enforcement point (your app) from the decision point (the engine).

**ABAC's flexibility is also its risk.** It's easy to build overly complex rule sets that are hard to audit and hard to predict. Small changes to rules can have dramatic impacts, and it can be difficult to tell when rules silently stop matching due to changes in the data they evaluate. The best practice is to **layer ABAC over RBAC** as a defense-in-depth strategy: RBAC provides the coarse foundation that holds even if ABAC rules break, and ABAC adds contextual constraints that RBAC can't express. Other best practices: version-control your policies, automate testing of policy changes against expected outcomes, and measure the performance overhead of policy evaluation early.

#### Beyond Identity-Based Models

The models above are all **identity-based**: they determine access based on who you are. Other models take fundamentally different approaches.

**ReBAC (Relationship-Based Access Control)** models permissions as relationships in a graph: "patrik is an editor of document X", "the engineering team owns project Y." Google's Zanzibar paper introduced this approach, and it's how Google Drive and similar systems handle sharing. Implementations include SpiceDB, Ory Keto, and Permify.

**Capability-based security** takes an entirely different approach. Identity-based systems rely on **ambient authority**: credentials like session cookies are attached to every request automatically, regardless of what the request does. This creates **confused deputy** vulnerabilities — a trusted component (the "deputy") is tricked into misusing its authority on behalf of an attacker. CSRF is the classic example: the browser (the deputy) automatically attaches the victim's cookie to a request forged by a malicious site. XSS, SQL injection, and many other attacks follow the same pattern — a process with broad authority is tricked into acting on an attacker's behalf.

Capability-based security eliminates ambient authority. A capability is an unforgeable reference to a resource combined with a set of permissions on that resource. If you have the capability, you can access the resource. If you don't, you can't even address the resource to attempt access. No identity check, no permission lookup. The contrast with identity-based systems: an authentication token conveys wide-ranging access to many resources and must be short-lived to limit damage. A capability conveys narrow access to one resource and can be long-lived precisely because its scope is limited. A capability doesn't identify a user — it directly conveys permissions.

Capabilities naturally support the **principle of least authority (POLA)** and make delegation straightforward: you can hand someone a capability to a specific resource without giving them access to your entire account. This is why capabilities are widespread on the web in the form of capability URIs — password reset links, GitHub Gists, Dropbox shared folders, Google Docs "anyone with the link" sharing. A **capability URI** embeds an unguessable token into a URI, combining resource identification with authorization in a single reference. The token can be encoded in the path (`/resource/abCd9..`), query parameter (`/resource?tok=abCd9..`), fragment (`/resource#tok=abCd9..`), or userinfo component (`abCd9..@api.example.com/resource`). Since REST already uses URIs to identify resources, capability URIs are a natural fit for REST APIs.

The tradeoffs: capabilities are harder to audit (who has access to what?) because possession of the token *is* the authorization, and there's no central registry of who holds which capabilities. Some capability systems don't support revocation at all. When revocation is supported, revoking a widely shared capability may deny access to more people than intended, because you can't selectively revoke for one holder.

**Macaroons** are a token format designed for capability-based systems. A macaroon is a capability token that carries **caveats** — restrictions that can be added by anyone but never removed. The holder can attenuate (narrow) the token's power before passing it on, but never broaden it. **First-party caveats** encode conditions the API checks locally: "only valid before 5pm", "only for GET requests." **Third-party caveats** require the client to obtain a **discharge macaroon** from an external service proving a condition is met: "prove you're an employee of Acme Corp", "prove you're over 18." The API verifies the discharge cryptographically without needing to contact the third-party service itself. Contextual caveats — appended just before the token is used — can bind the token to a specific request or session, further limiting misuse.

#### Access Control Architecture

Some identity providers (Okta, Entra) let you define roles and groups on their side and include them as custom claims in the token (`roles: ["editor"]`). This blurs the line between OIDC and your app's authorization. Centralizing roles in the IdP works well when multiple apps share the same role model. Managing roles in your app works better when they're app-specific.

**Separate access control from business logic.** Access control checks should run in middleware or filters *before* the application logic, not inside it. This ensures permissions are enforced consistently regardless of how the functionality is accessed, and changing the access control model doesn't require touching every endpoint. There's a spectrum of how far you take this separation: inline checks in business logic (simple but scattered and easy to forget), middleware or filters (consistent, centralized, applied per route), and external policy engines (fully decoupled, independently auditable and testable). Moving along this spectrum increases consistency but adds architectural complexity.

**Least privilege has a practical balance.** Too many permissions and the blast radius when something goes wrong is large. Too few permissions and people work around the system with shared credentials, service accounts, or "just give me admin." Overly restrictive access control that gets bypassed is worse than a slightly more permissive model that people actually follow. The goal is the minimum permissions people need to do their jobs without friction that drives them to circumvent the system.

#### Service-to-Service Authorization

Where user-facing authorization layers OAuth scopes, OIDC identity, and app-level permissions, service-to-service authorization layers transport identity, network policy, mesh policy, and application logic:

1. **mTLS identity** (transport layer): the service mesh establishes who the calling service is via its client certificate. This is the equivalent of authentication in the user-facing model.
2. **Network policies** (L3/L4): Kubernetes network policies restrict which pods can communicate at all, based on labels, namespaces, and ports. Coarse-grained, but stops entire classes of lateral movement.
3. **Service mesh authorization policies** (L7): Istio authorization policies or Linkerd server authorizations restrict by HTTP method, path, and headers, using the mTLS identity. For example: "the order service can GET /api/customers but cannot DELETE."
4. **Application-level authorization**: the receiving service checks the caller's identity (from the mTLS certificate or a token) against its own permission model. Business logic the mesh can't express: "the order service can read customer records, but only for customers associated with orders it is processing."

Each layer catches what the layer above misses. Network policies stop unauthorized connectivity. Mesh policies stop unauthorized request patterns. Application authorization enforces business rules. Within the cluster, you generally do not add OAuth client credentials on top of mTLS — the mesh already provides a stronger identity than a bearer token, and adding an AS call on every internal request adds latency and a single point of failure for no security gain. OAuth client credentials is for cross-boundary calls: between clusters, between organizations, or when calling third-party APIs where there is no shared mesh or PKI.

### OAuth 2.0 in Practice

The Authorization Code Grant with PKCE is the standard flow for almost everything now. Server-side apps use it with a client secret plus PKCE. Public clients (SPAs, mobile apps, CLIs) use it with PKCE alone. The Implicit Grant and Resource Owner Password Grant are both deprecated.

For machine-to-machine communication (no user involved), the Client Credentials Grant is standard. A backend service authenticates directly with the authorization server using its own credentials.

Pure SPAs that talk directly to an authorization server using Authorization Code + PKCE still exist, mostly for internal tools and dashboards, but for user-facing applications the preference has shifted toward keeping tokens server-side via the BFF pattern.

### Service-to-Service Authentication

In a microservice architecture, services communicate over the network using internal APIs. Securing this communication is a different problem from securing user-facing APIs: the clients are other services, not browsers, and the traffic flows within a cluster or data center rather than over the public internet.

#### Why TLS Inside the Network

The traditional assumption — that traffic inside the corporate network or cluster is safe — is wrong. If an attacker gains a foothold (a compromised container, a stolen credential, a vulnerable internal service), they can observe all unencrypted network traffic within the cluster. By sniffing communication between services, they can capture database passwords, API tokens, and user data, then use those credentials to access other systems directly, bypassing the compromised service entirely.

TLS inside the network protects against more than just eavesdropping. The certificate-based authentication built into TLS also prevents **DNS cache poisoning** (an attacker injects fake DNS responses to redirect traffic to a malicious server) and **ARP spoofing** (an attacker manipulates the mapping between IP addresses and hardware addresses to intercept traffic at a lower level). These attacks rely on the lack of authentication in low-level network protocols. Firewalls stop them from outside the network, but if the attacker is already inside — which is the scenario zero trust prepares for — TLS is what prevents them from expanding their access.

#### Public Key Infrastructure (PKI)

Enabling TLS for every service requires generating certificates and distributing private keys. The set of procedures and processes for creating, distributing, managing, and revoking certificates is called **public key infrastructure (PKI)**.

Running a PKI is operationally complex:

- Private keys and certificates must be distributed to every service and kept secure.
- Certificates must be issued by a private **certificate authority (CA)**. For additional security, use a hierarchy: a root CA and one or more intermediate CAs. Services that are publicly accessible must obtain their certificates from a public CA.
- Servers must present a correct certificate chain (service certificate + intermediate CA certificate) so the client can trace trust back to the root CA.
- Certificates must be **revoked** when a service is decommissioned or a private key is suspected compromised. Revocation is done by publishing **certificate revocation lists (CRLs)** or running an **OCSP (Online Certificate Status Protocol)** service.
- Certificates must be **renewed** automatically before they expire. Because revocation is imperfect (not all clients check CRLs promptly), short expiry times are preferred — they limit the window during which a compromised certificate remains valid. Tools like **cert-manager** automate certificate issuance and renewal in Kubernetes, obtaining certificates from either a public CA (Let's Encrypt) or a private organizational CA (Hashicorp Vault).

**Intermediate CAs** improve the security model. Instead of the root CA directly issuing certificates to every service (which requires the root CA to be online and exposed to compromise), you keep the root CA keys offline and use them only to periodically sign intermediate CA certificates. The intermediate CA then issues certificates to individual services. If the intermediate CA is compromised, you revoke its certificate and issue a new one — the root CA remains untouched. In multi-cluster deployments, each cluster can have its own intermediate CA, and name constraints in the intermediate certificate can restrict which hostnames it can issue certificates for.

#### How mTLS Gets Deployed

mTLS is a TLS feature, not a Kubernetes or cloud feature. Any two processes that can establish a TCP connection can do mTLS — VMs, bare-metal servers, containers, IoT devices. The question is who manages the certificates and who terminates TLS. There are three approaches, each automating what the previous one makes you do by hand:

- **Application code**: each service configures TLS directly (e.g., `ssl_context.verify_mode = CERT_REQUIRED` in Python, TLS config in Go's `http.Server`). You generate a certificate per service from your CA, deploy the cert and key to the server, and configure the application to present its cert and verify the peer's cert. This works anywhere — no Kubernetes, no mesh, no infrastructure dependencies. The cost is that every service must handle TLS, and certificate distribution and rotation is manual.
- **Per-service reverse proxy**: an NGINX, Envoy, or HAProxy instance sits in front of each service (or inside each pod as an extra container) and handles TLS. The application code stays plain HTTP. You still manage certificates yourself, but the application code is decoupled from TLS.
- **Service mesh**: a control plane automates everything — certificate issuance, distribution, rotation, and revocation. Application code is plain HTTP. This is the most automated option but adds infrastructure complexity.

Kubernetes by itself provides none of this. Pods communicate over plain HTTP on the cluster network by default. Kubernetes provides the networking (pod-to-pod connectivity, DNS, Services) but not TLS. Adding mTLS requires one of the approaches above.

#### Service Mesh

A **service mesh** automates TLS, certificate distribution, and mTLS for all service communication. It is the standard approach in large Kubernetes deployments where managing certificates manually across hundreds of pods is not practical.

A service mesh works by injecting a lightweight proxy as a **sidecar container** into every pod. The proxy intercepts all network traffic entering and leaving the pod. Because all communication flows through the proxies, they can transparently upgrade connections to TLS: a service makes a plain HTTP request to another service, and the sending proxy upgrades it to HTTPS. The receiving proxy terminates TLS and forwards the plain HTTP request to the application container. The application code doesn't need to handle TLS at all.

The service mesh has two layers:

- The **control plane** manages the CA, generates certificates for each service based on Kubernetes service metadata, distributes them to the proxies, and periodically reissues them. It also handles configuration, monitoring, and policy.
- The **data plane** is the collection of sidecar proxies that handle the actual traffic. They enforce TLS, collect metrics, and can apply traffic policies.

Linkerd and Istio are the two main implementations. Linkerd is simpler to deploy and configure, with a smaller attack surface. Istio has more features (including L7 authorization policies — see below) but is more complex.

#### mTLS in Depth

Standard TLS authenticates the server to the client: the server presents a certificate, and the client verifies it. The client doesn't need to authenticate because the server doesn't initiate connections. This is sufficient for the web, where the server needs to prove its identity to the browser.

**Mutual TLS (mTLS)** adds client-side certificates: both sides present and verify certificates during the handshake, authenticating each other. In a service mesh, every service has its own certificate (issued by the mesh's CA), so every connection is mutually authenticated. This is significantly more secure than application-level authentication mechanisms (API keys, bearer tokens) because the authentication happens at the transport layer before any application code runs, and the cryptographic guarantees are stronger.

mTLS is not by itself "more secure than TLS" — there are no attacks against TLS at the transport layer that mTLS prevents. The server certificate already prevents the client from connecting to a fake server. The value of mTLS comes from the **client certificate**: it gives the server a strongly authenticated client identity, which can be used to enforce authorization policies. Without mTLS, the server has to rely on application-level credentials (tokens, API keys) that can be stolen, replayed, or forged more easily.

#### Network Policies

Even with mTLS authenticating every connection, you may want to restrict which services can talk to each other at the network level. **Network policies** are Kubernetes resources that define allowed ingress and egress traffic for pods, acting as a firewall within the cluster.

A network policy selects pods by label and specifies which other pods (also by label), namespaces, or IP ranges can communicate with them, and on which ports. For example, a policy for the database pod might allow ingress only from pods labeled `app=natter-api-service` in the same namespace, and only on TCP port 9092. All other traffic to that pod is denied.

Network policies operate at layers 3/4 (IP addresses, ports, protocols). For finer-grained control — restricting by HTTP method, path, or headers — **service mesh authorization policies** (e.g. Istio authorization policies) operate at layer 7 using the service identities from mTLS client certificates. For example, you can allow one service to only make GET requests to another service. Service mesh authorization is not a replacement for application-level authorization (it can't prevent SSRF, for instance, since the malicious request comes from a legitimate service), but it limits lateral movement and reduces the blast radius of a compromised service.

#### Ingress Controllers

Services within a Kubernetes cluster communicate over the internal network, secured by mTLS via the service mesh. But external clients (browsers, mobile apps) need a way in. An **ingress controller** is a reverse proxy that sits at the cluster edge, accepting all incoming requests from outside the network.

The ingress controller handles:

- **TLS termination**: external clients connect over HTTPS. The ingress terminates the TLS connection, then forwards the request internally. If the ingress controller is part of the service mesh, the internal forwarding is automatically upgraded to mTLS.
- **Rate limiting**: applied before requests reach internal services, protecting the cluster from application-level DoS.
- **Audit logging**: recording incoming requests at the edge.
- **Routing**: directing requests to the appropriate internal service based on hostname or path.

The ingress controller functions as an API gateway for the cluster. Common implementations include NGINX, Envoy, and Traefik.

#### Service-to-Service Authentication Mechanisms

When services communicate (not on behalf of a user), several authentication patterns apply:

- **mTLS via service mesh**: The standard for intra-cluster communication. The mesh handles certificate distribution and rotation automatically. The service identity is derived from Kubernetes metadata (namespace, service account).
- **Client Credentials Grant**: The calling service authenticates with an OAuth authorization server using its own client ID and secret, and gets an access token. No user is involved. Used when services need scoped authorization beyond what mTLS identity provides.
- **Workload Identity**: Cloud platforms (AWS IAM roles, GCP service accounts, Azure Managed Identities) assign identities to compute workloads. The workload proves its identity to other services without managing secrets manually.
- **API Keys**: A shared secret passed in a header. Simple but offers no rotation, expiry, or scope mechanisms out of the box. Still widely used for third-party API access (Stripe, Twilio).

#### The Authentication Progression

These mechanisms form a progression, each eliminating a vulnerability of the previous step:

1. **API key**: a shared secret in a header. No identity, no expiry, no scope by default. If leaked, anyone can use it until manually rotated.
2. **JWT-as-credential**: the service mints a JWT and presents it directly to the API (no OAuth, no AS). Better than an API key (self-contained, expirable, signed), but there is no central authority governing issuance. Any holder of the signing key can create tokens.
3. **Client credentials + client secret**: the service authenticates to a central authorization server with a client ID and secret, and receives a scoped, short-lived access token. The AS governs issuance, enforces expiry, and can revoke tokens. But the client secret is a long-lived shared credential that must be stored securely on both sides.
4. **Client credentials + JWT assertion (`private_key_jwt`)**: instead of a shared secret, the service proves its identity to the AS by signing a short-lived JWT with its private key. The AS verifies the signature against the registered public key. The private key never leaves the service, and the assertion JWT expires in seconds. Eliminates the shared-secret problem.
5. **mTLS**: identity proven at the transport layer during the TLS handshake, before any application code runs. No application-level credential to steal or replay. The service mesh CA manages the full certificate lifecycle automatically.
6. **Certificate-bound access tokens (RFC 8705)**: combines mTLS with OAuth. The access token includes a `cnf` claim containing the SHA-256 thumbprint of the client's TLS certificate. The resource server verifies that the certificate on the mTLS connection matches the thumbprint in the token. Even if the token leaks, it is useless without the corresponding private key.

Each step moves trust from shared secrets toward cryptographic proof of key possession. The industry direction is toward steps 5-6: mTLS for transport identity, with certificate-bound tokens for cross-boundary calls where OAuth is needed.

#### Workload Identity

In a Kubernetes cluster, workload identity replaces manually managed service credentials with platform-issued, automatically rotated cryptographic identities.

**SPIFFE (Secure Production Identity Framework for Everyone)** is the CNCF standard (Graduated, 2022) for workload identity. Each workload gets a SPIFFE ID — a URI like `spiffe://trust-domain/ns/namespace/sa/service-account` — and a corresponding X.509 certificate (SVID — SPIFFE Verifiable Identity Document) that encodes this ID. SPIRE is the reference implementation. SPIFFE decouples workload identity from the infrastructure it runs on: the same identity model works across Kubernetes, VMs, and bare metal. Istio issues SPIFFE-compatible SVIDs as part of its mTLS certificate management. Google, Netflix, GitHub, Uber, and Amazon have adopted SPIFFE in production.

**Cloud workload identity federation** lets workloads authenticate to cloud APIs without storing long-lived credentials. The mechanism: the Kubernetes service account token (a JWT signed by the cluster's API server) is exchanged for short-lived cloud credentials via a trust relationship between the cluster and the cloud provider. AWS offers IAM Roles for Service Accounts (IRSA) and EKS Pod Identity, GCP offers Workload Identity Federation, and Azure offers Workload Identity Federation via Microsoft Entra. All three cloud providers now recommend eliminating long-lived service account keys.

The convergence: SPIFFE provides a platform-neutral identity layer, cloud workload identity federation provides cloud-specific integration, and service meshes provide the transport. These are complementary, not competing.

#### Secrets Management

Services need credentials: database passwords, API keys, signing keys, TLS certificates. The problem is distributing, rotating, and revoking them securely.

**Kubernetes Secrets** are the starting point. A Secret is a Kubernetes resource containing base64-encoded key-value pairs, accessible to pods via volume mounts or environment variables. File-mounted secrets are preferred over environment variables: files update when the secret changes (no pod restart needed), and environment variables leak more easily through process listings, crash dumps, and debug logs. Kubernetes Secrets are stored in etcd and can be encrypted at rest with etcd encryption, but by default base64 encoding is the only protection — it is not encryption. Don't commit Secret YAML files to version control.

**External secret stores** provide stronger guarantees: HashiCorp Vault, AWS Secrets Manager, GCP Secret Manager, Azure Key Vault. These offer encryption, fine-grained access control, audit logging, and automatic rotation. The External Secrets Operator syncs secrets from these stores into Kubernetes Secrets, bridging external stores with the Kubernetes API.

**Credential distribution patterns** for getting secrets into pods:
- **Sidecar injection**: a vault agent runs as a sidecar container and writes secrets to a shared volume that the application container reads.
- **Init container**: fetches secrets at startup before the main container starts.
- **CSI driver**: mounts secrets as a filesystem backed by the secret store (Secrets Store CSI Driver).
- **Direct API calls**: the application fetches secrets from the secret store at runtime using the workload's own identity for authentication.

The goal: no long-lived credentials in code, config, or environment variables. Credentials are fetched at runtime, scoped to the workload, and rotated automatically.

#### Token Binding and Proof-of-Possession

Bearer tokens have a fundamental weakness: possession is sufficient to use them. If a token leaks (via logs, error messages, a compromised proxy, or browser history), the attacker can use it from anywhere. Token binding addresses this by tying the token to a key the client holds, making stolen tokens useless without the corresponding private key.

**Certificate-bound access tokens (RFC 8705)**: the access token includes a `cnf` (confirmation) claim containing the SHA-256 thumbprint of the client's TLS certificate. When the client presents the token to the resource server over mTLS, the resource server checks that the certificate in the TLS connection matches the thumbprint in the token. The API only needs to compare hashes — it does not need to validate the certificate chain or check CA trust, because the authorization server already did that when it issued the token. This significantly simplifies the API's implementation of client certificate authentication.

**DPoP (Demonstrating Proof of Possession, RFC 9449)**: designed for contexts where mTLS is impractical — browser-based apps, mobile clients. The client generates a key pair, includes the public key when requesting a token, and on each API call sends a DPoP proof (a signed JWT proving possession of the private key). The resource server verifies the proof matches the key bound to the token.

The common principle: both mechanisms convert bearer tokens into sender-constrained tokens. The token is bound to a key held by the legitimate client. The OAuth Security BCP (RFC 9700) recommends sender-constrained tokens for high-security environments.

### Application-Level Attacks

Attack patterns that exploit application logic rather than network or infrastructure vulnerabilities.

#### SSRF (Server-Side Request Forgery)

An SSRF attack occurs when an API accepts a URL from a user and fetches it from inside the trusted network. The attacker submits a URL pointing to an internal service — one they can't reach directly from outside the firewall — and the API, running inside the network, makes the request on their behalf. The attacker can use this to probe internal services for vulnerabilities, steal credentials returned from internal endpoints, or trigger actions via internal APIs.

The Capital One breach (July 2019) is the canonical example. An attacker exploited an SSRF vulnerability in a Web Application Firewall to request the **AWS metadata service** (`169.254.169.254`), a simple HTTP server available on the local network of every EC2 instance that responds with instance credentials. The stolen credentials were then used to access S3 buckets containing 100 million customer records. The AWS metadata service has since been hardened (IMDSv2 requires a PUT request with a hop-limited token), but SSRF remains dangerous wherever an API fetches user-supplied URLs.

**Prevention** follows the same allowlist-over-denylist principle as input validation:

- **Allowlisting** (preferred): check URLs against a set of permitted hostnames, domain names, or exact URLs. Only URLs matching the allowlist are fetched. This is the most secure approach but isn't always feasible — a link-preview service, for example, needs to fetch arbitrary URLs.
- **Blocklisting** (when allowlisting isn't feasible): extract the hostname from the URL, resolve it to all IP addresses, and reject any address that is local or private:
  - Loopback addresses (`127.0.0.0/8`) — always refers to the local machine, allowing access to other containers in the same pod.
  - Link-local addresses (`169.254.0.0/16` in IPv4, `fe80::/10` in IPv6) — reserved for the local network segment. The AWS metadata service lives at `169.254.169.254`.
  - Private-use ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, site-local IPv6) — nodes and pods within a Kubernetes cluster typically use these.
  - Multicast addresses and the wildcard address (`0.0.0.0`).
  - IPv6 unique local addresses (`fd00::/8`).

Resolving the hostname to **all** IP addresses (not just the first one) is important because a hostname may resolve to both a public and a private address. If any resolved address is private, the request should be rejected.

**DNS rebinding** is a bypass technique: the attacker controls a DNS server that first returns a public IP (passing the blocklist check), then on subsequent resolution returns a private IP (which the application uses for the actual request). Defenses include pinning the resolved IP for the duration of the request and re-checking after following redirects (since redirects can also point to internal addresses).

---

## Roles

Who does what in the security and infrastructure landscape. These roles overlap significantly and organizations draw the boundaries differently. Smaller companies combine several of these into one person; larger companies specialize further.

### Security Roles

**CISO (Chief Information Security Officer)**: The executive responsible for the organization's security program. Reports to the CEO, CTO, or board. Sets security strategy, manages risk appetite, owns compliance, and is accountable when things go wrong. Not a technical role in most organizations, though many CISOs have technical backgrounds.

**Security Engineer**: Builds and maintains security infrastructure: SIEM configuration, detection rules, firewall policies, vulnerability scanning pipelines, identity provider setup. The line between security engineer and the more specialized roles below is blurry. In many companies this is the generalist security role.

**Security Architect**: Designs security into systems before they're built. Reviews architectures, defines security standards, chooses frameworks and tools. Works across teams rather than owning a specific system. More senior and strategic than a security engineer.

**Application Security (AppSec) Engineer**: Focused on securing the software itself. Runs SAST/DAST tooling, reviews code for vulnerabilities, builds secure coding guidelines, triages bug bounty reports, and works with development teams to fix issues. Often embedded in or closely partnered with engineering teams.

**IAM Engineer**: Specializes in identity and access management. Configures identity providers (Okta, Entra), manages SSO integrations, builds RBAC/ABAC models, handles provisioning/deprovisioning, runs access reviews. In larger organizations this is a dedicated team.

**SOC Analyst (Security Operations Center)**: Monitors alerts, investigates incidents, performs triage. Tier 1 analysts handle initial triage and escalation, tier 2 analysts do deeper investigation, tier 3 analysts handle complex incidents and threat hunting. Shift-based work in many organizations.

**Penetration Tester / Red Team**: Tests defenses by simulating attacks. Pen testers focus on specific targets (test this app, this network segment). Red teamers simulate real adversaries across the full kill chain, testing detection and response, not just technical controls. Some organizations have internal teams, others hire external firms.

**GRC Analyst**: Manages governance, risk, and compliance. Prepares for audits (SOC 2, ISO 27001, PCI-DSS), maintains policy documentation, tracks risk registers, manages vendor security questionnaires. More process-oriented than technical.

**Cloud Security Engineer**: Secures cloud infrastructure specifically. IAM policies in AWS/GCP/Azure, CSPM tooling, container security, infrastructure-as-code scanning. Overlaps with both security engineering and cloud/platform engineering.

### Infrastructure and Platform Roles

**Platform Engineer**: Builds and maintains the internal developer platform: CI/CD pipelines, deployment tooling, service templates, observability stack, developer self-service. The goal is to make it easy for application developers to ship reliably without dealing with infrastructure directly. This role has largely absorbed what "DevOps engineer" used to mean.

**Site Reliability Engineer (SRE)**: Focuses on reliability: uptime, latency, error budgets, incident response, capacity planning. Originated at Google. SREs write software to automate operational work, define SLOs (Service Level Objectives), and manage the tension between shipping features and maintaining reliability. Overlaps with platform engineering, but SRE is more focused on production behavior and incident response.

**Infrastructure Engineer**: Manages the underlying compute, storage, and networking. In cloud-native organizations this means Terraform/Pulumi, Kubernetes, networking policies, and cloud account structure. In organizations with on-premises infrastructure it also means physical servers, data centers, and hardware. The title is becoming less common as it splits into platform engineering (developer-facing) and cloud engineering (infrastructure-facing).

**DevOps Engineer**: A role title that was widespread around 2015-2022 but has become ambiguous. Originally meant someone who bridges development and operations, automating deployments and infrastructure. In practice the responsibilities have split: the developer-facing tooling side became platform engineering, and the production reliability side became SRE. Many job postings still use the title, but the trend is toward the more specific roles.

### How These Roles Interact

Security doesn't sit in a silo. The modern pattern is "shift left": integrate security into development and operations rather than bolting it on at the end.

**AppSec + Development**: AppSec engineers work with developers to catch vulnerabilities during development (code review, SAST in CI) rather than after deployment.

**Security + Platform**: Platform engineers build security into the developer platform itself: automated secret injection, default network policies, pre-hardened container base images. When the platform is secure by default, individual teams don't have to think about it.

**SRE + Security Operations**: Incident response draws on both. SREs handle availability incidents, SOC analysts handle security incidents, and the boundary blurs when a security incident causes an outage (ransomware, DDoS).

**IAM + Everyone**: Identity touches every team. Platform engineers configure workload identity. Developers integrate with identity providers. SREs manage service account lifecycles. IAM engineers set the policies that govern all of it.

## Secure Development Lifecycle

How security fits into the process of building and running software. The common pattern is an engineer designs and builds something with little security input, then security concerns get discovered late, when they're expensive to fix. The ideal process integrates security throughout, mostly by making the secure path the default path.

### Threat Modeling

The single most valuable step, and the one most often skipped. Before building anything, someone thinks through: what data does this handle? Who should access it? What happens if it's compromised? What's the blast radius? This takes an hour, not a week, and it changes what you build.

An internal tool that handles no PII and sits behind a VPN needs very different security from a customer-facing API that processes payments. Without a threat model, teams make architectural decisions that are hard to undo later: storing secrets in plaintext, running without auth because "it's internal", giving every service admin access to the database. An hour of threat modeling at the start prevents months of remediation later.

#### How to Threat Model

Start by drawing a **dataflow diagram**: boxes for processes, data stores, and external entities, with arrows showing how data moves between them. Then identify **trust boundaries**: lines where control changes hands. Data flowing within a single process is less interesting than data crossing from an external client into your API, or from your API into a database under a different user account. The flows that cross trust boundaries are where threats concentrate.

Apply **STRIDE** to each flow that crosses a trust boundary. STRIDE is an acronym for six threat categories:

- **Spoofing**: pretending to be someone else. Countered by authentication.
- **Tampering**: altering data you're not supposed to alter. Countered by integrity controls (signatures, checksums, input validation).
- **Repudiation**: denying that you did something you actually did. Countered by audit logging.
- **Information disclosure**: revealing information that should be private. Countered by encryption and access controls.
- **Denial of service**: preventing legitimate users from accessing the system. Countered by rate limiting, resource quotas, redundancy.
- **Elevation of privilege**: gaining access to functionality you're not supposed to have. Countered by authorization and least privilege.

The goal is to identify general categories of threats, not to enumerate every possible attack. For each threat you identify, you decide: mitigate it (add a control), accept it (the risk is low enough), or eliminate it (redesign so the threat doesn't apply).

Not every threat needs a bespoke mitigation. Many common threats (spoofing, tampering, information disclosure) are addressed by consistently applying basic security mechanisms: authentication, TLS, input validation, access control, audit logging, and rate limiting. The threat model tells you which of these mechanisms matter for your specific system and where to apply them.

For a deeper treatment, Adam Shostack's "Threat Modeling: Designing for Security" (Wiley, 2014) is the standard reference.

#### Foundational Security Controls

Five basic controls address all six STRIDE threats. Applied consistently, they handle the majority of common attacks against an API or service:

1. **Rate limiting** → counters Denial of Service. The outermost layer: reject overload before doing any expensive work.
2. **Encryption (HTTPS/TLS)** → counters Information Disclosure and Tampering. Protects data in transit and, with encryption at rest, on disk.
3. **Authentication** → counters Spoofing. Verifies the caller is who they claim to be.
4. **Audit logging** → counters Repudiation. Records who did what and when. Logs after authentication (so you know who is acting) but before authorization (so you capture denied attempts too, which may indicate an attack). Log both the request and the response: if the process crashes mid-request and you only log responses, you lose the trace of what caused the crash. Write audit logs to durable storage (filesystem, database) so they survive crashes, and store them independently (no foreign keys to other tables) so the record stays consistent even if other data changes or gets deleted. In production, send logs to a centralized SIEM for correlation across systems. Restrict audit log access to a separate set of users (auditors), not the system administrators being monitored: an admin who can delete their own audit trail defeats the purpose (this is the principle of **separation of duties**).
5. **Access control** → counters Elevation of Privilege, Tampering, and Information Disclosure. Decides whether a request is allowed or denied. If access is denied, return a 403 Forbidden immediately without running any application logic.

These form a pipeline that every request passes through in order: rate limiting first (cheapest check, protects availability), then authentication (identify the caller), then audit logging (record the attempt regardless of outcome), then access control (enforce permissions), and finally the application logic. The ordering matters: authentication failure doesn't immediately reject the request, because you want to log unauthenticated attempts, and a later access control decision may reject it. Only rate limiting and access control actually block requests.

### The Platform as Security Infrastructure

Most security should be infrastructure and platform work, not per-project work. The individual engineer shouldn't configure TLS, secrets injection, or workload identity from scratch. The platform team builds these into service templates that every team inherits by default.

What a mature platform provides out of the box:

- **Identity**: authentication handled by a shared identity provider, not reinvented per service.
- **Secrets**: injected at runtime by the platform (vault, KMS), never committed to code or manually configured.
- **Network**: mTLS between services, network policies, and segmentation by default.
- **Observability**: logging, monitoring, and audit trails configured automatically.
- **Container security**: pre-hardened base images, image scanning in CI.

When the platform is secure by default, individual teams don't have to think about it. The engineer focuses on their threat model (what data, who accesses it, what's the blast radius) and the platform handles the mechanics.

### Security in the Development Pipeline

Security feedback should show up like linter warnings, not as a gate at the end.

- **Design**: the security architect reviews designs that touch sensitive data or introduce new trust boundaries. Most services don't need a bespoke review because the platform handles the common cases.
- **Implementation**: CI runs SAST (static analysis), dependency scanning, and container image scanning automatically. Results appear in the PR. The AppSec engineer is available to consult, not standing in front of a gate.
- **Deployment**: access to production is just-in-time, audited, and scoped. The platform provisions infrastructure with appropriate security controls based on what the developer declares they need (a database, a queue, an external API).
- **Production**: SRE and security operations monitor for reliability and security. Vulnerabilities get patched through automated pipelines. Access reviews happen periodically.

### Scaling with Organizational Maturity

A startup with 5 engineers can't afford a platform team, an AppSec team, and an IAM team. That's all the same person, and they're also writing features. The process scales with the organization: what starts as "think about security for an hour before building" evolves into a platform that encodes those decisions for everyone. The practices above describe the target state, not the starting point.

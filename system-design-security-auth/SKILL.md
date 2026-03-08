---
name: system-design-security-auth
description: Security and authentication for system design — OAuth 2.0, JWT, SSO, session management, HTTPS/TLS, encryption, password storage, XSS prevention, and API security. Use when implementing authentication flows, choosing between session and token-based auth, securing APIs, handling passwords, or designing secure systems.
---

# Security & Authentication — System Design Reference

Based on *System Design Interview* (Vol. 1 & 2) by Alex Xu & Sahn Lam; *ByteByteGo* archive.

## 1. Authentication Methods Overview

| Method | How It Works | State | Scalability | Best Use Case |
|--------|-------------|-------|-------------|---------------|
| **Session/Cookie** | Server stores session → sends session ID cookie to browser | Stateful | Limited (sticky sessions or shared store) | Traditional web apps, server-rendered |
| **JWT** | Server issues signed token → client stores and sends in `Authorization` header | Stateless | High (no server state) | SPAs, mobile apps, microservices |
| **OAuth 2.0** | Delegated authorization via authorization server → issues access token | Stateless | High | Third-party API access, social login |
| **SSO** | Central IdP authenticates once → issues tokens for multiple service providers | Stateless | High | Enterprise, multi-app ecosystems |
| **API Key** | Static key sent in header/query → server validates against key store | Stateless | High | Server-to-server, public APIs, rate-limited access |
| **mTLS** | Both client and server present certificates → mutual verification | Stateless | High (but cert management complex) | Service mesh, zero-trust internal APIs |

**Decision heuristic:** Session for simple server-rendered apps → JWT for stateless microservices → OAuth for third-party delegation → SSO for enterprise multi-app → mTLS for service-to-service.

## 2. OAuth 2.0

### Grant Types

| Grant Type | Flow | Client Type | Use Case |
|-----------|------|-------------|----------|
| **Authorization Code** | User → AuthZ server → code → token exchange | Confidential (server-side) | Web apps with backend |
| **Authorization Code + PKCE** | Same as above + code verifier/challenge | Public (SPA, mobile) | SPAs, native mobile apps |
| **Client Credentials** | Client authenticates directly → token | Confidential (machine) | Service-to-service, background jobs |
| **Resource Owner Password** | User gives credentials to client → token | Trusted first-party only | Legacy migration only — **avoid** |
| ~~**Implicit**~~ | Token returned directly (no code exchange) | Public | **Deprecated** — use Auth Code + PKCE instead |

### Authorization Code Flow (Simplified)

```
1. User clicks "Login with Google"
2. App redirects to AuthZ server (/authorize?response_type=code&client_id=...&redirect_uri=...&scope=...)
3. User authenticates + consents
4. AuthZ server redirects back with ?code=AUTHORIZATION_CODE
5. App backend exchanges code for tokens (POST /token with code + client_secret)
6. AuthZ server returns { access_token, refresh_token, expires_in }
7. App uses access_token to call resource API
```

### PKCE (Proof Key for Code Exchange)

Prevents authorization code interception for public clients (no client_secret).

```
1. Client generates random code_verifier + derives code_challenge = SHA256(code_verifier)
2. /authorize includes code_challenge + code_challenge_method=S256
3. /token includes original code_verifier
4. AuthZ server verifies SHA256(code_verifier) == code_challenge
```

## 3. JWT (JSON Web Token)

### Structure

```
HEADER.PAYLOAD.SIGNATURE
  │        │         │
  │        │         └─ HMACSHA256(base64(header) + "." + base64(payload), secret)
  │        └─ Claims: { "sub": "user123", "exp": 1700000000, "iat": 1699999000 }
  └─ { "alg": "HS256", "typ": "JWT" }
```

### Standard Claims

| Claim | Name | Purpose |
|-------|------|---------|
| `iss` | Issuer | Who created the token |
| `sub` | Subject | User/entity the token represents |
| `aud` | Audience | Intended recipient (API URL) |
| `exp` | Expiration | Token expiry (Unix timestamp) |
| `iat` | Issued At | When token was created |
| `nbf` | Not Before | Token not valid before this time |
| `jti` | JWT ID | Unique token identifier (revocation) |

### Access Token vs Refresh Token

| | Access Token | Refresh Token |
|---|---|---|
| **Purpose** | Authorize API requests | Obtain new access tokens |
| **Lifetime** | Short (5–15 min) | Long (days–weeks) |
| **Storage** | Memory (SPA) or httpOnly cookie | httpOnly cookie or secure server-side |
| **Sent to** | Resource server | Authorization server only |
| **Revocable** | Hard (stateless) | Easy (server-side check) |

### JWT Pros & Cons

| Advantages | Disadvantages |
|-----------|--------------|
| Stateless — no server session store | Cannot be revoked before expiry without blocklist |
| Scales horizontally — any server can validate | Token size larger than session ID (~1 KB vs ~32 bytes) |
| Self-contained — carries user data in payload | Sensitive data in payload if not encrypted (JWE) |
| Cross-domain friendly | Must handle token refresh flow |
| Mobile/API-native | Vulnerable if signing key is compromised |

### JWT vs Session Comparison

| Aspect | Session-Based | JWT |
|--------|-------------|-----|
| **Server state** | Session store required (memory/Redis/DB) | No server state |
| **Scalability** | Sticky sessions or shared store | Any server can validate |
| **Revocation** | Delete session from store | Need blocklist or short TTL |
| **Storage** | Cookie (session ID) | `Authorization` header or cookie |
| **CSRF risk** | Yes (cookies auto-sent) | No (if using `Authorization` header) |
| **XSS risk** | Lower (httpOnly cookie) | Higher if stored in localStorage |
| **Mobile support** | Awkward (cookies on mobile) | Native fit |

## 4. SSO (Single Sign-On)

### How SSO Works

```
1. User visits app-a.com → not authenticated → redirect to IdP
2. User authenticates at IdP (identity.corp.com)
3. IdP issues token/assertion → redirects back to app-a.com
4. User visits app-b.com → redirect to IdP → already authenticated → instant redirect with token
5. User is logged into both apps with one authentication
```

### SAML vs OpenID Connect

| | SAML 2.0 | OpenID Connect (OIDC) |
|---|---|---|
| **Based on** | XML | JSON / JWT (built on OAuth 2.0) |
| **Token format** | XML assertion | ID Token (JWT) + Access Token |
| **Transport** | HTTP POST / Redirect binding | HTTP REST APIs |
| **Best for** | Enterprise SSO, legacy systems | Modern web/mobile apps, SPAs |
| **Complexity** | High (XML parsing, certificates) | Lower (JSON, standard OAuth flows) |
| **Discovery** | Metadata XML | `.well-known/openid-configuration` |
| **Adoption trend** | Legacy / declining | Growing — industry default for new apps |

**Rule of thumb:** Use OIDC for new applications. Use SAML only when integrating with enterprise IdPs that require it.

## 5. Session-Based Authentication

### How Sessions Work

```
1. Client sends credentials (POST /login)
2. Server validates → creates session object → stores in session store
3. Server sends Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Strict
4. Client includes cookie on every request automatically
5. Server looks up session_id in store → retrieves user data
6. On logout → server deletes session from store
```

### Session Storage Options

| Store | Pros | Cons | Best For |
|-------|------|------|----------|
| **In-memory** | Fastest; simplest | Lost on restart; not shared across instances | Single-server dev/prototype |
| **Redis** | Fast; shared across instances; TTL support; persistence | Extra infra; network hop | Production multi-instance apps |
| **Database** | Durable; queryable; auditable | Slowest; DB load increase | Compliance requirements; audit trails |
| **Signed cookies** | No server store needed; stateless | Size limit (~4 KB); data visible to client | Minimal session data; low-traffic apps |

## 6. HTTPS & TLS

### TLS 1.3 Handshake (Simplified)

```
Client                              Server
  │                                    │
  ├── ClientHello ──────────────────► │  (supported ciphers, key share, random)
  │                                    │
  │ ◄── ServerHello + Certificate ─── ┤  (chosen cipher, key share, cert chain)
  │ ◄── Finished ──────────────────── ┤
  │                                    │
  ├── Finished ──────────────────────► │
  │                                    │
  │ ◄═══ Encrypted Application Data ══► │  (1-RTT handshake complete)
```

**TLS 1.3 vs 1.2:** 1-RTT handshake (vs 2-RTT); removed insecure ciphers (RC4, 3DES, SHA-1); 0-RTT resumption available (with replay risk).

### Certificate Chain of Trust

```
Root CA (self-signed, pre-installed in OS/browser trust store)
  └── Intermediate CA (signed by Root CA)
        └── Server Certificate (signed by Intermediate CA)
```

Browsers validate: domain match → expiry → signature chain → not revoked (OCSP/CRL).

### Certificate Types

| Type | Validation | Issuance Time | Use Case |
|------|-----------|---------------|----------|
| **DV** (Domain Validation) | Domain ownership only | Minutes | Blogs, personal sites, APIs |
| **OV** (Organization Validation) | Domain + org identity verified | Days | Business websites |
| **EV** (Extended Validation) | Domain + org + legal verification | Weeks | Banking, e-commerce (green bar) |

### Why HTTPS Everywhere

- Prevents eavesdropping on credentials, tokens, personal data
- Prevents MITM (man-in-the-middle) injection of ads, malware, tracking
- Required for HTTP/2 and HTTP/3
- SEO ranking factor (Google)
- Required for Service Workers, Geolocation API, WebRTC

## 7. Encoding vs Encryption vs Hashing vs Tokenization

| | Encoding | Encryption | Hashing | Tokenization |
|---|---|---|---|---|
| **Purpose** | Data format conversion | Data confidentiality | Data integrity/verification | Data substitution |
| **Reversible** | Yes (no key needed) | Yes (with correct key) | No (one-way) | Yes (via token vault) |
| **Key required** | No | Yes | No | No (vault lookup) |
| **Examples** | Base64, URL encoding, Unicode | AES-256, RSA, ChaCha20 | SHA-256, bcrypt, Argon2 | Payment tokenization |
| **Use case** | Data transmission, serialization | Secrets at rest/in transit | Password storage, checksums | PCI DSS compliance, PII protection |
| **Security** | None — easily decoded | Strong with proper key management | Strong for verification | Strong — token has no mathematical relation to original |

**Key insight:** Encoding ≠ security. Never encode secrets (e.g., Base64 a password). Encrypt data you need to retrieve. Hash data you only need to verify. Tokenize data that must pass through untrusted systems.

## 8. Password Storage

### Rules

1. **Never store plaintext** passwords
2. **Never use** plain MD5 or SHA-256 — vulnerable to rainbow tables
3. **Always salt** — unique random string per password
4. **Use slow hashes** — bcrypt, Argon2, scrypt (intentionally expensive)

### Algorithm Comparison

| Algorithm | Type | Salt Built-in | Configurable Cost | Memory-Hard | Recommended |
|-----------|------|--------------|-------------------|-------------|-------------|
| **bcrypt** | Adaptive hash | Yes | Yes (rounds) | No | Yes — proven, widely supported |
| **Argon2id** | KDF (winner of PHC 2015) | Yes | Yes (time, memory, parallelism) | Yes | Yes — best for new systems |
| **scrypt** | KDF | Yes | Yes (N, r, p) | Yes | Yes — good alternative |
| MD5 / SHA-256 | Fast hash | No | No | No | **No** — too fast, rainbow tables |
| PBKDF2 | KDF | External | Yes (iterations) | No | Acceptable (FIPS compliance) |

### Validation Flow

```
1. User enters password
2. System fetches stored salt + hash from DB
3. System computes hash(password + salt)
4. Compare computed hash with stored hash
5. Match → authenticated; No match → rejected
```

## 9. Common Security Threats

| Threat | What It Is | Prevention |
|--------|-----------|------------|
| **XSS** (Cross-Site Scripting) | Attacker injects malicious script into web page viewed by others | Input sanitization; Content Security Policy (CSP); escape output; httpOnly cookies |
| **CSRF** (Cross-Site Request Forgery) | Attacker tricks user's browser into making unwanted requests to authenticated site | CSRF tokens; `SameSite` cookie attribute; verify `Origin`/`Referer` headers |
| **SQL Injection** | Attacker inserts SQL code via user input to manipulate database | Parameterized queries / prepared statements; ORM; input validation; least-privilege DB user |
| **DDoS** (Distributed Denial of Service) | Flood of traffic overwhelms server resources | Rate limiting; WAF (Web Application Firewall); CDN absorption; auto-scaling; IP blacklisting |
| **MITM** (Man-in-the-Middle) | Attacker intercepts communication between client and server | HTTPS/TLS everywhere; certificate pinning; HSTS |
| **Credential Stuffing** | Attacker uses leaked username/password pairs from other breaches | Rate limiting; MFA; breach-check APIs (HaveIBeenPwned); anomaly detection |
| **Broken Auth** | Weak login flows, no MFA, poor session management | MFA; account lockout; strong password policy; session timeout |
| **Insecure Deserialization** | Attacker modifies serialized objects to execute code or escalate privileges | Validate/sign serialized data; avoid deserializing untrusted input; use safe formats (JSON) |

## 10. API Security Best Practices

| Practice | Implementation |
|----------|---------------|
| **Authentication** | OAuth 2.0 / JWT for user APIs; API keys for service APIs; mTLS for internal |
| **Authorization** | RBAC or ABAC; check permissions per endpoint; principle of least privilege |
| **Input validation** | Validate type, length, range, format on server side; reject unexpected fields |
| **Rate limiting** | Token bucket or sliding window; per-user and per-IP limits; return `429 Too Many Requests` |
| **HTTPS** | TLS 1.2+ everywhere; HSTS header; no mixed content |
| **CORS** | Whitelist specific origins; avoid `Access-Control-Allow-Origin: *` for authenticated APIs |
| **Error handling** | Generic error messages to client; detailed logs server-side; never expose stack traces |
| **Audit logging** | Log who did what, when, from where; tamper-proof log storage; retain per compliance |
| **Request signing** | HMAC signature of request body + timestamp → prevents tampering and replay |
| **Secrets management** | Rotate API keys; use vaults (HashiCorp Vault, AWS Secrets Manager); never hardcode |

## 11. Firewall & Network Security

### Top 6 Firewall Use Cases

| Use Case | What It Does |
|----------|-------------|
| **Port-based rules** | Allow/block traffic on specific ports (80/HTTP, 443/HTTPS, 22/SSH) |
| **IP address filtering** | Whitelist trusted IPs; blacklist known malicious IPs |
| **Protocol-based rules** | Allow/block by protocol (TCP, UDP, ICMP) |
| **Time-based rules** | Enforce different access rules during business hours vs off-hours |
| **Stateful inspection** | Monitor state of active connections; only allow traffic matching established connections |
| **Application-based rules** | Control traffic by application (block BitTorrent, allow Slack) — Layer 7 firewalls |

### Network Security Layers

| Layer | Protection | Tools |
|-------|-----------|-------|
| **Perimeter** | External traffic filtering | Firewall, WAF, DDoS protection (Cloudflare, AWS Shield) |
| **Network** | Internal segmentation | VPC, subnets, security groups, NACLs |
| **Host** | Per-machine hardening | OS firewall (iptables), antivirus, patch management |
| **Application** | App-level controls | Input validation, auth, rate limiting, CSP |
| **Data** | Data protection | Encryption at rest/in transit, tokenization, DLP |

### VPN Basics

```
1. Establish encrypted tunnel between device and VPN server
2. All traffic routed through VPN server
3. IP address masked — appears as VPN server's IP
4. ISP/network operators see encrypted traffic only
```

**Use cases:** Remote access to corporate networks; bypass geo-restrictions; protect on public Wi-Fi.

## 12. DevSecOps

### Shift-Left Security

```
Traditional:  Plan → Code → Build → Test → Deploy → [Security Audit]
Shift-Left:   Plan → [Threat Model] → Code → [SAST] → Build → [SCA] → Test → [DAST] → Deploy → [Monitor]
```

### Security in CI/CD Pipeline

| Stage | Security Activity | Tools |
|-------|------------------|-------|
| **Code** | Static analysis (SAST) | Semgrep, SonarQube, CodeQL, Snyk Code |
| **Dependencies** | Software Composition Analysis (SCA) | Snyk, Dependabot, Renovate, npm audit |
| **Build** | Container image scanning | Trivy, Snyk Container, Docker Scout |
| **Test** | Dynamic analysis (DAST) | OWASP ZAP, Burp Suite, Nuclei |
| **Secrets** | Secret detection | GitLeaks, TruffleHog, detect-secrets |
| **Deploy** | Infrastructure scanning (IaC) | Checkov, tfsec, KICS |
| **Runtime** | Monitoring + anomaly detection | Falco, Datadog, Sentry |

### Key DevSecOps Practices

| Practice | Description |
|----------|------------|
| **Automated security checks** | Every PR triggers SAST/SCA scans; block merge on critical findings |
| **Secret management** | Vault-based secrets; no secrets in code/config; rotate regularly |
| **Container security** | Minimal base images; non-root containers; image signing |
| **Threat modeling** | STRIDE or PASTA framework at design phase |
| **Vulnerability management** | Track CVEs; SLA for remediation (critical: 24h, high: 7d, medium: 30d) |
| **Continuous monitoring** | Runtime protection; WAF rules; alerting on anomalies |

## 13. Payment Security Quick Reference

| Problem | Solution |
|---------|----------|
| Request/response eavesdropping | HTTPS everywhere |
| Data tampering | Encryption + integrity monitoring (HMAC) |
| Man-in-the-middle attack | TLS with certificate pinning |
| Password storage | Salt + adaptive hashing (bcrypt / Argon2) |
| Data loss | Multi-region DB replication + snapshots |
| DDoS | Rate limiting + WAF + firewall |
| Card theft | Tokenization — store tokens, not card numbers |
| PCI compliance | PCI DSS standard; use hosted payment pages from PSP |
| Fraud | Address verification (AVS), CVV, behavioral analysis |

## Quick Decision Guide

```
Need to authenticate users in a web app?
  ├── Server-rendered, single domain → Session + Cookie
  ├── SPA or mobile app → JWT (access + refresh tokens)
  └── Multiple apps, one identity → SSO (OIDC)

Need to authorize third-party access?
  └── OAuth 2.0
        ├── Server-side app → Authorization Code
        ├── SPA/mobile → Authorization Code + PKCE
        └── Service-to-service → Client Credentials

Need to store passwords?
  └── Argon2id (preferred) or bcrypt

Need to secure an API?
  ├── External → OAuth 2.0 + rate limiting + HTTPS
  ├── Internal → mTLS + service mesh
  └── Public data → API key + rate limiting
```

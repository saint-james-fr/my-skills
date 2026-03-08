# Security & Authentication — Detailed Reference

## OAuth 2.0 Detailed Flows

### Authorization Code Flow (Full Detail)

```
┌──────┐       ┌──────────┐       ┌──────────────┐       ┌──────────────┐
│ User │       │  Client  │       │  AuthZ Server │       │ Resource API │
└──┬───┘       └────┬─────┘       └──────┬───────┘       └──────┬───────┘
   │                │                     │                      │
   │  1. Click      │                     │                      │
   │  "Login"       │                     │                      │
   │───────────────►│                     │                      │
   │                │                     │                      │
   │                │  2. Redirect to     │                      │
   │                │  /authorize         │                      │
   │◄───────────────│  ?response_type=code│                      │
   │                │  &client_id=...     │                      │
   │                │  &redirect_uri=...  │                      │
   │                │  &scope=read        │                      │
   │                │  &state=xyz         │                      │
   │                │                     │                      │
   │  3. User authenticates              │                      │
   │  + grants consent                   │                      │
   │────────────────────────────────────►│                      │
   │                │                     │                      │
   │  4. Redirect to redirect_uri        │                      │
   │  ?code=AUTH_CODE&state=xyz          │                      │
   │◄────────────────────────────────────│                      │
   │                │                     │                      │
   │  5. Forward    │                     │                      │
   │  auth code     │                     │                      │
   │───────────────►│                     │                      │
   │                │                     │                      │
   │                │  6. POST /token     │                      │
   │                │  grant_type=        │                      │
   │                │  authorization_code │                      │
   │                │  &code=AUTH_CODE    │                      │
   │                │  &client_secret=... │                      │
   │                │  &redirect_uri=...  │                      │
   │                │────────────────────►│                      │
   │                │                     │                      │
   │                │  7. Return          │                      │
   │                │  {access_token,     │                      │
   │                │   refresh_token,    │                      │
   │                │   expires_in: 3600} │                      │
   │                │◄────────────────────│                      │
   │                │                     │                      │
   │                │  8. GET /api/data   │                      │
   │                │  Authorization:     │                      │
   │                │  Bearer <token>     │                      │
   │                │─────────────────────────────────────────►│
   │                │                     │                      │
   │                │  9. Return data     │                      │
   │                │◄─────────────────────────────────────────│
```

### Client Credentials Flow (Machine-to-Machine)

```
┌──────────┐                    ┌──────────────┐       ┌──────────────┐
│  Client  │                    │  AuthZ Server │       │ Resource API │
└────┬─────┘                    └──────┬───────┘       └──────┬───────┘
     │                                 │                      │
     │  1. POST /token                 │                      │
     │  grant_type=client_credentials  │                      │
     │  &client_id=...                 │                      │
     │  &client_secret=...             │                      │
     │  &scope=...                     │                      │
     │────────────────────────────────►│                      │
     │                                 │                      │
     │  2. {access_token, expires_in}  │                      │
     │◄────────────────────────────────│                      │
     │                                 │                      │
     │  3. GET /api/resource           │                      │
     │  Authorization: Bearer <token>  │                      │
     │─────────────────────────────────────────────────────►│
     │                                 │                      │
     │  4. Return data                 │                      │
     │◄─────────────────────────────────────────────────────│
```

No user interaction — the client authenticates itself directly. Used for service-to-service communication, batch jobs, and backend microservices.

### Authorization Code + PKCE Flow

```
┌──────┐       ┌──────────┐       ┌──────────────┐
│ User │       │ SPA/App  │       │  AuthZ Server │
└──┬───┘       └────┬─────┘       └──────┬───────┘
   │                │                     │
   │                │  1. Generate:       │
   │                │  code_verifier =    │
   │                │    random(128)      │
   │                │  code_challenge =   │
   │                │    SHA256(verifier)  │
   │                │                     │
   │  2. Redirect   │                     │
   │◄───────────────│  /authorize?        │
   │                │  code_challenge=... │
   │                │  &code_challenge_   │
   │                │   method=S256       │
   │                │  &client_id=...     │
   │                │                     │
   │  3. Auth + consent                  │
   │────────────────────────────────────►│
   │                │                     │
   │  4. ?code=AUTH_CODE                 │
   │◄────────────────────────────────────│
   │───────────────►│                     │
   │                │                     │
   │                │  5. POST /token     │
   │                │  code=AUTH_CODE     │
   │                │  &code_verifier=... │ ← proves possession
   │                │────────────────────►│
   │                │                     │
   │                │  6. Verify:         │
   │                │  SHA256(verifier)   │
   │                │  == challenge?      │
   │                │                     │
   │                │  7. {access_token,  │
   │                │   refresh_token}    │
   │                │◄────────────────────│
```

**Why PKCE matters:** Public clients (SPAs, mobile) cannot securely store a `client_secret`. PKCE prevents authorization code interception attacks by proving the same client that initiated the flow is the one exchanging the code.

### Token Refresh Flow

```
┌──────────┐                    ┌──────────────┐
│  Client  │                    │  AuthZ Server │
└────┬─────┘                    └──────┬───────┘
     │                                 │
     │  1. API call returns 401        │
     │  (access_token expired)         │
     │                                 │
     │  2. POST /token                 │
     │  grant_type=refresh_token       │
     │  &refresh_token=...             │
     │  &client_id=...                 │
     │────────────────────────────────►│
     │                                 │
     │  3. Validate refresh_token      │
     │  (not expired, not revoked)     │
     │                                 │
     │  4. {new_access_token,          │
     │   new_refresh_token}            │  ← rotate refresh token
     │◄────────────────────────────────│
```

**Refresh token rotation:** Issue a new refresh token with each refresh. If an old refresh token is reused, revoke the entire token family (indicates theft).

## JWT Implementation Best Practices and Pitfalls

### Best Practices

| Practice | Detail |
|----------|--------|
| **Short expiry** | Access tokens: 5–15 min. Limits damage window if stolen |
| **Rotate signing keys** | Use `kid` (key ID) in header; support multiple active keys during rotation |
| **Validate all claims** | Always check `exp`, `iss`, `aud`, `nbf` — don't just verify the signature |
| **Use asymmetric signing** | RS256/ES256 for distributed systems — only auth server holds private key; any service can verify with public key |
| **Store securely** | Never `localStorage` for access tokens in browsers — use httpOnly cookies or in-memory |
| **Include minimal data** | Only put necessary claims in payload — token is not encrypted by default |
| **Use `jti` for revocation** | If you need revocation, maintain a blocklist keyed by `jti` with TTL = remaining token life |

### Common Pitfalls

| Pitfall | Why It's Dangerous | Fix |
|---------|-------------------|-----|
| **Storing JWT in localStorage** | XSS attack can steal the token | Use httpOnly cookie or in-memory variable |
| **No expiry or very long expiry** | Stolen token grants indefinite access | Short TTL (5–15 min) + refresh tokens |
| **Using `alg: none`** | Attacker forges tokens without signing | Always validate `alg`; reject `none`; use allowlist of algorithms |
| **Symmetric signing in distributed systems** | Every service that validates needs the secret (attack surface grows) | Use RS256/ES256 — private key only on auth server |
| **Putting secrets in payload** | JWT payload is Base64-encoded, not encrypted — anyone can read it | Use JWE (encrypted JWT) or keep secrets server-side |
| **Not validating `iss` and `aud`** | Token from another service accepted as valid | Strict issuer and audience validation |
| **Accepting tokens from URL query params** | Tokens logged in server access logs, browser history, referrer headers | Always use `Authorization` header or cookies |
| **Confusing encoding with encryption** | Base64 is not encryption — JWT payloads are readable by anyone | Use JWE for sensitive payload data |

### Algorithm Selection

| Algorithm | Type | Key | Performance | Security | When To Use |
|-----------|------|-----|------------|----------|-------------|
| **HS256** | Symmetric (HMAC) | Shared secret | Fast | Good (single service) | Monolith; single-service validation |
| **RS256** | Asymmetric (RSA) | Public/private pair | Slower (RSA ops) | Strong | Microservices; distributed validation |
| **ES256** | Asymmetric (ECDSA) | Public/private pair | Fast (smaller keys) | Strong | Preferred over RS256 for new systems |
| **EdDSA** | Asymmetric (Ed25519) | Public/private pair | Fastest asymmetric | Strong | Cutting-edge; gaining adoption |

## TLS 1.3 Handshake Step-by-Step

### Full Handshake (1-RTT)

| Step | Direction | Message | Content |
|------|-----------|---------|---------|
| 1 | Client → Server | **ClientHello** | Protocol version, supported cipher suites, supported groups (curves), key share (client's DH public key), random nonce |
| 2 | Server → Client | **ServerHello** | Selected cipher suite, server's key share (DH public key), random nonce |
| 3 | Server → Client | **EncryptedExtensions** | Server extensions (ALPN, server name) — encrypted with handshake keys |
| 4 | Server → Client | **Certificate** | Server's certificate chain |
| 5 | Server → Client | **CertificateVerify** | Signature over handshake transcript (proves server has private key) |
| 6 | Server → Client | **Finished** | MAC over entire handshake (integrity check) |
| 7 | Client → Server | **Finished** | Client's MAC over entire handshake |
| 8 | Both | | **Application data can flow** (1-RTT complete) |

### 0-RTT Resumption

```
Client (has PSK from previous session):
  ├── ClientHello + early_data (0-RTT application data)
  ├── Uses PSK to derive early traffic keys
  └── Server decrypts early data immediately

Risk: 0-RTT data is replayable — only use for idempotent requests (GET, not POST)
```

### TLS 1.3 vs TLS 1.2

| Feature | TLS 1.2 | TLS 1.3 |
|---------|---------|---------|
| **Handshake** | 2-RTT | 1-RTT (0-RTT with PSK) |
| **Cipher suites** | ~37 supported | 5 supported (only strong ones) |
| **Key exchange** | RSA or DHE | DHE / ECDHE only (forward secrecy mandatory) |
| **Removed** | — | RSA key exchange, RC4, 3DES, SHA-1, CBC mode, compression |
| **Forward secrecy** | Optional | Mandatory |
| **Encrypted handshake** | Partial (certificates in clear) | Most messages encrypted after ServerHello |

### Key Exchange Concepts

| Concept | Explanation |
|---------|------------|
| **Forward secrecy** (PFS) | Even if server's long-term private key is compromised, past sessions cannot be decrypted — each session uses ephemeral keys |
| **Diffie-Hellman (DHE)** | Both sides contribute to key generation; agreed key never transmitted |
| **ECDHE** | Elliptic-curve variant of DHE — smaller keys, faster, same security |
| **Pre-Shared Key (PSK)** | Key established in previous session; enables 0-RTT resumption |

## Password Hashing Algorithm Comparison

### Detailed Comparison

| Feature | bcrypt | Argon2id | scrypt | PBKDF2 |
|---------|--------|----------|--------|--------|
| **Year** | 1999 | 2015 (PHC winner) | 2009 | 2000 |
| **Based on** | Blowfish cipher | Compression function | Salsa20/8 | HMAC-SHA |
| **Memory-hard** | No | Yes (configurable) | Yes (configurable) | No |
| **GPU/ASIC resistant** | Moderate | High | High | Low |
| **Parameters** | cost (rounds) | time, memory, parallelism | N, r, p | iterations, key length |
| **Output format** | `$2b$12$salt.hash` | `$argon2id$v=19$m=65536,t=3,p=4$salt$hash` | Binary (custom encoding) | Binary (custom encoding) |
| **Max input length** | 72 bytes | Unlimited | Unlimited | Unlimited |
| **FIPS 140-2** | No | No | No | Yes |
| **Recommendation** | Proven; widely supported | Best for new systems | Good alternative | Only if FIPS required |

### Recommended Parameters (2024+)

| Algorithm | Parameter | Recommended Value | Notes |
|-----------|-----------|-------------------|-------|
| **bcrypt** | cost | 12–14 | Each increment doubles time; target ~250ms on your hardware |
| **Argon2id** | memory | 64 MB (`m=65536`) | More memory = more GPU-resistant |
| **Argon2id** | iterations | 3 (`t=3`) | Time cost; increase if memory is limited |
| **Argon2id** | parallelism | 4 (`p=4`) | Match available CPU cores |
| **scrypt** | N | 2^17 (131072) | CPU/memory cost; must be power of 2 |
| **scrypt** | r | 8 | Block size |
| **scrypt** | p | 1 | Parallelism |
| **PBKDF2** | iterations | 600,000+ (SHA-256) | OWASP 2023 recommendation |

### Password Storage Schema

```
users table:
  id              UUID PRIMARY KEY
  email           VARCHAR UNIQUE NOT NULL
  password_hash   VARCHAR NOT NULL   -- includes algorithm, salt, and hash
  created_at      TIMESTAMP
  updated_at      TIMESTAMP
  mfa_enabled     BOOLEAN DEFAULT false
  mfa_secret      VARCHAR            -- TOTP secret (encrypted at rest)
  failed_attempts INTEGER DEFAULT 0
  locked_until    TIMESTAMP
```

### Password Policy Best Practices

| Rule | Recommendation | Rationale |
|------|---------------|-----------|
| **Minimum length** | 8+ characters (12+ preferred) | Longer passwords are exponentially harder to crack |
| **Maximum length** | 64–128 characters | Prevent DoS via very long passwords |
| **Complexity rules** | Don't force special chars | NIST 800-63B: forced complexity → weaker passwords (Post-it notes) |
| **Breach check** | Check against HaveIBeenPwned API | Block known-compromised passwords |
| **Rotation** | Don't force periodic rotation | NIST 800-63B: forced rotation → weaker passwords |
| **MFA** | Always offer; require for sensitive ops | Mitigates credential stuffing even with compromised password |

## XSS Prevention Techniques

### Types of XSS

| Type | Mechanism | Example | Persistence |
|------|-----------|---------|-------------|
| **Stored (Persistent)** | Malicious script saved in database → served to all users | Attacker posts `<script>steal(cookies)</script>` in comment; every viewer executes it | Permanent until removed |
| **Reflected** | Malicious script in URL → reflected in response | `https://site.com/search?q=<script>alert(1)</script>` | Per-request |
| **DOM-based** | Client-side JS reads attacker-controlled input and writes to DOM | `document.getElementById('output').innerHTML = location.hash` | Per-request |

### Prevention Layers

| Layer | Technique | Implementation |
|-------|-----------|---------------|
| **Output encoding** | Escape HTML entities before rendering user data | `&lt;` for `<`, `&gt;` for `>`, `&amp;` for `&`, `&quot;` for `"` |
| **Content Security Policy** | HTTP header restricting allowed script sources | `Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.example.com` |
| **Input validation** | Reject/sanitize unexpected input on server side | Allowlist approach: define what's valid, reject everything else |
| **httpOnly cookies** | Prevent JavaScript from accessing session cookies | `Set-Cookie: session=abc; HttpOnly; Secure; SameSite=Strict` |
| **Sanitization libraries** | Clean HTML input where rich text is needed | DOMPurify (browser), sanitize-html (Node.js), Bleach (Python) |
| **Templating auto-escape** | Framework auto-escapes by default | React JSX (auto-escapes), Vue `{{ }}` (auto-escapes), Django templates |
| **Trusted Types** | Browser API preventing DOM XSS | `Content-Security-Policy: require-trusted-types-for 'script'` |

### CSP Header Examples

```
# Strict policy (recommended for new apps)
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self'; connect-src 'self' https://api.example.com; frame-ancestors 'none'; base-uri 'self'; form-action 'self'

# Report-only mode (monitor before enforcing)
Content-Security-Policy-Report-Only: default-src 'self'; report-uri /csp-report

# Nonce-based (for inline scripts)
Content-Security-Policy: script-src 'nonce-{random}' 'strict-dynamic'
```

## OWASP Top 10 (2021) Quick Reference

| # | Risk | Description | Key Mitigations |
|---|------|------------|-----------------|
| A01 | **Broken Access Control** | Users act outside their intended permissions | RBAC/ABAC; deny by default; server-side enforcement; CORS restrictions |
| A02 | **Cryptographic Failures** | Weak or missing encryption; exposed sensitive data | TLS everywhere; strong algorithms (AES-256, SHA-256+); no custom crypto |
| A03 | **Injection** | SQL, NoSQL, OS, LDAP injection via untrusted input | Parameterized queries; ORMs; input validation; least-privilege DB accounts |
| A04 | **Insecure Design** | Missing or ineffective security controls at design level | Threat modeling; secure design patterns; abuse case testing |
| A05 | **Security Misconfiguration** | Default configs, open cloud storage, verbose errors | Hardened defaults; automated config audits; remove unused features |
| A06 | **Vulnerable Components** | Using components with known vulnerabilities | SCA tools (Snyk, Dependabot); regular updates; monitor CVEs |
| A07 | **Auth & ID Failures** | Weak auth, credential stuffing, session fixation | MFA; strong password policy; session timeout; rate limiting |
| A08 | **Software/Data Integrity Failures** | Unverified updates, CI/CD pipeline compromise | Code signing; integrity checks; verified dependency sources |
| A09 | **Logging & Monitoring Failures** | Insufficient logging; no alerting on attacks | Structured logging; SIEM; alert on auth failures and anomalies |
| A10 | **SSRF** (Server-Side Request Forgery) | Server makes requests to attacker-controlled destinations | Allowlist outbound URLs; block internal/metadata IPs; network segmentation |

## Cloud Security Checklist

### Identity & Access

| Item | Action |
|------|--------|
| IAM least privilege | Grant minimum permissions required; use permission boundaries |
| MFA everywhere | Enforce MFA for console access; hardware keys for root accounts |
| Service accounts | Use roles/service accounts instead of long-lived keys |
| Access review | Quarterly review of IAM policies; remove unused accounts/roles |
| SSO integration | Federated identity via OIDC/SAML; no shared accounts |

### Network Security

| Item | Action |
|------|--------|
| VPC design | Private subnets for databases/services; public subnets only for load balancers |
| Security groups | Deny all by default; allow only required ports/sources |
| WAF | Enable for public-facing endpoints; OWASP managed rules |
| DDoS protection | AWS Shield / Cloudflare / GCP Cloud Armor |
| VPN/PrivateLink | Use private connectivity for cross-VPC or on-prem traffic |

### Data Protection

| Item | Action |
|------|--------|
| Encryption at rest | Enable for all storage (S3, RDS, EBS, DynamoDB) |
| Encryption in transit | TLS 1.2+ for all traffic; no unencrypted internal traffic |
| Key management | Use managed KMS (AWS KMS, GCP Cloud KMS); rotate keys |
| Backup encryption | Encrypt backups; test restore procedures |
| Data classification | Tag data by sensitivity; apply policies per classification |

### Monitoring & Compliance

| Item | Action |
|------|--------|
| Cloud trail/audit logs | Enable CloudTrail/Cloud Audit Logs for all accounts/projects |
| Config compliance | AWS Config / GCP Security Command Center / Azure Policy |
| Vulnerability scanning | Regular scans of instances, containers, and serverless |
| Incident response | Documented runbook; tested quarterly; automated containment |
| Compliance frameworks | SOC 2, ISO 27001, HIPAA, PCI DSS — as applicable |

## DDoS Mitigation Strategies

### Attack Types

| Type | Layer | Mechanism | Volume |
|------|-------|-----------|--------|
| **Volumetric** | L3/L4 | UDP flood, ICMP flood, DNS amplification | Overwhelm bandwidth (Tbps) |
| **Protocol** | L3/L4 | SYN flood, Ping of Death, Smurf | Exhaust connection state tables |
| **Application** | L7 | HTTP flood, Slowloris, cache-busting queries | Exhaust server resources (CPU, memory) |

### Multi-Layer Defense

```
Layer 1: Edge / CDN (Cloudflare, AWS CloudFront, Akamai)
  ├── Absorb volumetric attacks at edge PoPs
  ├── Anycast routing distributes traffic globally
  └── Bot detection (CAPTCHA, JS challenge, fingerprinting)

Layer 2: WAF (Web Application Firewall)
  ├── Rate limiting per IP / per session
  ├── Block known attack patterns (SQL injection, XSS)
  └── Geo-blocking (block countries with no legitimate traffic)

Layer 3: Load Balancer / Auto-Scaling
  ├── Distribute traffic across instances
  ├── Auto-scale to absorb L7 surges
  └── Connection draining for unhealthy instances

Layer 4: Application Level
  ├── Request validation (reject malformed requests early)
  ├── Caching (serve from cache, reduce backend load)
  ├── Circuit breakers (prevent cascade failures)
  └── Graceful degradation (shed non-critical features)

Layer 5: Infrastructure
  ├── Network ACLs (block known-bad IP ranges)
  ├── BGP blackholing (last resort — drops all traffic to target IP)
  └── ISP-level scrubbing (upstream filtering)
```

### DDoS Response Playbook

| Step | Action | Who |
|------|--------|-----|
| 1 | Detect anomaly (traffic spike, latency increase, error rate) | Monitoring (automated) |
| 2 | Classify attack type (volumetric, protocol, application) | SRE / Security |
| 3 | Activate DDoS protection (CDN scrubbing, WAF rules) | SRE (automated if possible) |
| 4 | Scale infrastructure (auto-scale groups, add capacity) | SRE |
| 5 | Block attack sources (IP blocklist, geo-block, rate limit) | Security |
| 6 | Monitor and adjust (attackers adapt; tune defenses) | SRE + Security |
| 7 | Post-incident review (root cause, defense gaps, improvements) | All |

## Zero Trust Architecture

### Core Principles

| Principle | Description |
|-----------|------------|
| **Never trust, always verify** | Every request is authenticated and authorized, regardless of source |
| **Least privilege** | Grant minimum access needed for the specific task |
| **Assume breach** | Design as if the network is already compromised |
| **Micro-segmentation** | Fine-grained network segments; lateral movement is difficult |
| **Continuous verification** | Re-evaluate trust on every request, not just at session start |

### Implementation Components

```
                         ┌────────────────────┐
                         │   Policy Engine     │  ← decides allow/deny based on:
                         │   (PDP)             │    identity, device, context, risk
                         └─────────┬──────────┘
                                   │
┌──────┐    ┌──────────────┐    ┌──┴────────────┐    ┌──────────────┐
│ User │───►│ Device Trust  │───►│ Policy        │───►│ Application  │
│      │    │ (MDM, cert,   │    │ Enforcement   │    │ / Resource   │
│      │    │ health check) │    │ Point (PEP)   │    │              │
└──────┘    └──────────────┘    └───────────────┘    └──────────────┘
                                       │
                              ┌────────┴────────┐
                              │ Context signals: │
                              │ - User identity  │
                              │ - Device posture │
                              │ - Location       │
                              │ - Time of day    │
                              │ - Risk score     │
                              └─────────────────┘
```

### Zero Trust vs Traditional Perimeter Security

| Aspect | Perimeter Security | Zero Trust |
|--------|-------------------|------------|
| **Trust model** | Trust inside the network | Trust no one; verify every request |
| **Access control** | Network-level (firewall) | Identity + context-based |
| **Lateral movement** | Easy once inside | Blocked by micro-segmentation |
| **Remote workers** | VPN required | Native support (identity-based) |
| **Breach impact** | Full network access possible | Limited to specific resource |

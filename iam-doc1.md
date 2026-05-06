# IAM Interview Prep — Document 1 of 4
## Protocol Fundamentals + System Design

> **Series:** Principal Engineer · IAM Product Development · SAML + OIDC + OAuth 2.0

---

## 📚 Your 4-document roadmap

Before you read a single question, here is exactly where you are in this series and why it is structured this way.

| Doc | File | What it covers | Why it comes here |
|---|---|---|---|
| **1 — This doc** | `iam-doc1-fundamentals.md` | Protocol depth + System design | You must understand the protocols before you can break them or build with them |
| **2** | `iam-doc2-attacks-incidents.md` | 12 attack scenarios + 12 incident war rooms | Once you understand protocols, you learn how they fail under attack and at 3am |
| **3** | `iam-doc3-product-code.md` | IAM product architecture + Code review | Building production-grade IAM systems + spotting broken auth before it ships |
| **4** | `iam-doc4-api-leadership.md` | API/SDK design + Curveballs + Leadership | The Principal differentiator — design, judgment, and leading through ambiguity |

**Reading order matters.** Each document builds on the one before it. The bridge at the end of each document tells you exactly what gap it leaves and what the next document fills.

---

## 🎬 Before you walk in — the mindset shift

Picture two engineers walking into the same Principal Engineer IAM interview.

**Engineer A** spent the week reading OAuth RFCs and can quote every grant type by name. When asked "walk me through the Authorization Code flow," they deliver a flawless 8-step answer. When asked "your team's mobile app doesn't use PKCE — the PM says it adds complexity — what do you do?", they say "PKCE is required for security."

**Engineer B** knows the same protocols but spent the week asking harder questions. When asked about PKCE and the PM pushback, they say: "I'd quantify the risk first — on Android, any app can register the same redirect URI, so without PKCE an auth code intercepted by a malicious app on the same device has no verifier to stop the exchange. Then I'd show the PM it's literally one config flag in our library and pair with the developer to ship it before the sprint ends. The risk is real; the cost is a Tuesday afternoon."

**Engineer B gets the job.** Not because they knew more — because they showed judgment, communication, and the ability to lead through ambiguity. That is what these documents train.

```
What a Senior Engineer demonstrates:   "I know how OAuth works."
What a Principal Engineer demonstrates: "I know when OAuth fails, what breaks it
                                         at 3am, how to design systems around its
                                         limits, and how to help 8 developers
                                         not make the mistakes I already made."
```

---

## 📖 Master Glossary

> Every acronym in this series defined once. Refer back here whenever something appears unfamiliar.

| Term | Plain English definition |
|---|---|
| **AS** | Authorization Server — the system that issues tokens (Okta, PingFed, Auth0, Google) |
| **RS** | Resource Server — your API. Validates tokens. Serves protected data. |
| **RP** | Relying Party — OIDC term for the Client (the app relying on the IdP) |
| **IdP** | Identity Provider — knows who users are (PingFederate, Okta, Azure AD, ADFS) |
| **SP** | Service Provider — the app in SAML (Salesforce, Workday, ServiceNow) |
| **JWT** | JSON Web Token — signed, self-contained token readable by anyone, tamper-proof |
| **JWK** | JSON Web Key — a public key in JSON format |
| **JWKS** | JSON Web Key Set — a URL that serves a collection of JWKs |
| **JTI** | JWT ID — unique identifier for one specific JWT, used for replay prevention |
| **PKCE** | Proof Key for Code Exchange — per-request cryptographic proof replacing client_secret for public clients |
| **BFF** | Backend For Frontend — a server-side proxy that holds tokens so the browser never does |
| **ROPC** | Resource Owner Password Credentials — deprecated grant where app collects user password directly |
| **SPA** | Single Page Application — browser app with no server rendering (React, Vue, Angular) |
| **OIDC** | OpenID Connect — identity layer on top of OAuth 2.0 that adds who the user is |
| **CoT** | Circle of Trust — the set of entities with mutually established SAML metadata trust |
| **IGA** | Identity Governance and Administration — governs who should have access, automates lifecycle |
| **SCIM** | System for Cross-domain Identity Management — REST API standard for automated provisioning |
| **PAM** | Privileged Access Management — protecting high-privilege admin accounts |
| **JIT** | Just-In-Time provisioning — account created on first SSO login |
| **SLO** | Single Logout — log out once, signed out of all connected apps |
| **SSO** | Single Sign-On — authenticate once, access many apps |
| **ACS** | Assertion Consumer Service — the SP endpoint where IdP POSTs the SAML Response |
| **XSW** | XML Signature Wrapping — attack exploiting position-based XML assertion processing |
| **SoD** | Segregation of Duties — preventing one person from holding conflicting permissions |
| **MFA** | Multi-Factor Authentication — two or more verification methods required |
| **NameID** | The user identifier sent in a SAML assertion |
| **EntityID** | The unique identifier for an IdP or SP — usually a URL |
| **Binding** | Transport mechanism for SAML messages (HTTP-Redirect, HTTP-POST) |
| **RelayState** | The "where to go after login" parameter in SAML flows |
| **ForceAuthn** | Forces fresh authentication even if an active SSO session exists |
| **SessionIndex** | Links IdP session to SP session — required for SLO to work correctly |
| **at_hash** | Access Token Hash — cryptographic binding in the ID token to a specific access token |
| **DPoP** | Demonstration of Proof-of-Possession — binds a token to a specific client key |
| **mTLS** | Mutual TLS — both client and server present certificates to prove identity |
| **FAPI** | Financial-grade API — OpenID Foundation profile mandating DPoP and stricter OAuth |
| **SPIFFE** | Secure Production Identity Framework for Everyone — workload identity standard |
| **sub** | Subject — unique identifier for the user or service the token is about |
| **iss** | Issuer — URL of the Auth Server that created the token |
| **aud** | Audience — who this token is intended for (the API's URL) |
| **exp** | Expiry — Unix timestamp after which the token is invalid |
| **nbf** | Not Before — token invalid before this Unix timestamp |
| **iat** | Issued At — Unix timestamp when the token was created |
| **RS256** | RSA Signature SHA-256 — asymmetric signing, private key signs, public key verifies |
| **ES256** | ECDSA Signature SHA-256 — faster asymmetric alternative with smaller keys |
| **HS256** | HMAC SHA-256 — symmetric signing, same secret signs and verifies — avoid for APIs |
| **RFC 6749** | The OAuth 2.0 specification |
| **RFC 7636** | The PKCE specification |
| **RFC 8693** | The Token Exchange specification |
| **RFC 7662** | The Token Introspection specification |
| **RFC 7009** | The Token Revocation specification |
| **OAuth 2.1** | The modernised OAuth draft — removes Implicit + ROPC, mandates PKCE + rotation |

---

## Category 1 — Protocol Fundamentals 🔬

> *"Tell me about OAuth 2.0" is a junior question. Everything in this category is not a junior question.*

**What this category tests:** Whether you understand the limits, failure modes, and design tradeoffs of SAML, OIDC, and OAuth — not just what they do. The expected answer is never a definition. It is a trade-off analysis with production consequences attached.

---

### Q1 — The iss+sub identifier trap

**The scenario:** Your app uses "Sign in with Google." Users are happy. Six months later, a user changes their Google account email. They try to log in. Your app creates a brand new account for them. All their history, settings, and data are in the old account. Angry support ticket. This is a production bug in thousands of apps built by developers who learned OAuth from tutorials.

**❓ "Why should you never use email as a unique user identifier in an OIDC system? Give me every failure mode."**

**💬 What most people say:** "Emails can change."

**⭐ What a Principal says:**

Four distinct failure modes, each with a different production consequence:

**Failure mode 1 — Email mutability:** Jane Smith gets married and becomes Jane Jones. Her Google `sub` is unchanged and permanent. If your users table has `PRIMARY KEY (email)`, your app sees `jane.jones@company.com` as a completely new person. Old Jane has all the history. New Jane has nothing. Your data integrity is broken on a user's happiest day.

**Failure mode 2 — Cross-IdP identity collision:** Google can issue `jane@company.com`. Microsoft can also issue `jane@company.com` for a completely different human being at a different company using the same domain via Azure AD. If you look up users by email across multiple providers, Person A at Google and Person B at Azure AD are now sharing an account.

**Failure mode 3 — email_verified bypass:** Anyone can register `cto@yourcompany.com` as their email address at Google — you do not need to own the domain. If `email_verified` is `false` and you skip that check, a random person on the internet can impersonate your CTO. This is not theoretical. It is a real attack class.

**Failure mode 4 — Federation key instability:** Your company merges with another. Employees migrate from `name@company-a.com` to `name@merged.com`. If email is your primary key, every single user account breaks on migration day. Every. Single. One.

**The correct approach:**
```
PRIMARY KEY: provider_id = iss + "|" + sub
Example:     "https://accounts.google.com|110248495921238986420"

sub is assigned by Google when the account is first created.
sub never changes — not when email changes, not when name changes, not ever.
```

> 🎯 **What the interviewer is really testing:** Do you understand the difference between a display attribute (email) and an identity key (sub)? Do you know why OIDC has both? Can you explain why tutorials get this wrong and production systems pay for it?

> **Follow-up probe:** "What if the same user signs in with both Google and GitHub? Now you have two different sub values for the same human." Answer: account linking — let users connect multiple identity providers to one internal record. The primary key stays `iss+sub`. You have a separate `linked_providers` table. The user can log in via either and land in the same account.

---

### Q2 — JWT vs opaque tokens: the performance vs revocation war

**The scenario:** You are in a technical design meeting. The security architect says: "We need opaque tokens — instant revocation is non-negotiable." The platform architect says: "Opaque tokens require an introspection call on every request — that's 50-100ms overhead per API call at 200,000 requests per second. Unacceptable." Both are right. You have been asked to make the call.

**❓ "JWT or opaque tokens? Make a decision and defend it."**

**💬 What most people say:** "JWTs for performance, opaque for security." Then they stop.

**⭐ What a Principal says:**

Both requirements are valid. They apply to different parts of the system. The answer is: **both, based on operation risk profile.**

```
Operation                   Token type         Reasoning
────────────────────────────────────────────────────────────────────────
Standard API reads           JWT               Local validation ~0ms.
                                               98% of your traffic.
Payment initiation           Opaque            50ms introspection is
                                               acceptable on a payment.
                                               Instant revocation required.
Admin operations             Short JWT         5-min expiry + JTI blocklist.
                             + Redis blocklist Effective revocation window:
                                               seconds, not minutes.
Logout                       Revoke refresh    Access token expires naturally.
                             token via RFC 7009 Refresh token = the real risk.
```

> 💬 *"The security architect is right about revocation. The platform architect is right about latency. These are not in conflict — they apply to different endpoints. Define which endpoints are 'high-value' and apply opaque tokens there. The other 95% stays JWT. Neither person in that meeting thought to ask: which specific operations need instant revocation? That is the question a Principal asks."*

**The 60-second version for the interviewer:** "I'd use JWTs for standard API reads — local validation, zero latency overhead, and it handles 95% of traffic. For operations where instant revocation is genuinely required — payments, admin actions — I'd use short-lived JWTs with a Redis JTI blocklist, which gives you near-instant effective revocation with one sub-millisecond Redis lookup per request. For the rare case where a hard revocation SLA exists (healthcare, regulated finance), opaque tokens plus introspection on those specific endpoints. Never a binary either/or."

> 🎯 **What the interviewer is really testing:** Can you decompose a binary-sounding problem into a nuanced per-use-case decision? Can you quantify trade-offs rather than stating them abstractly?

---

### Q3 — When SAML is the wrong answer (and how to say so)

**The scenario:** A large enterprise customer signs up for your SaaS platform. Their IT team emails: "We need SAML SSO. We use PingFederate internally." Your platform is OIDC-native. The startup CTO says: "Just build SAML support, it's not that hard." You have been in enough SAML integrations to know that statement is wrong.

**❓ "Your enterprise customer insists on SAML. Your platform is OIDC-native. Walk me through your exact decision process — technically and commercially."**

**⭐ Principal-level answer:**

**Step 1: Challenge the requirement before touching code.**

Does this customer actually need SAML — or do they need SSO with their corporate PingFederate? These are different things. PingFederate supports OIDC. If their IT team defaults to asking for SAML because "that's what we use," point them to their own IdP's OIDC configuration. Five minutes on a call can eliminate eight weeks of development.

**Step 2: If they genuinely need SAML, build or broker?**

Building SAML natively means: XML parsing with XSW attack surface, certificate management and rotation procedures, metadata exchange for every customer, attribute contract negotiation for every integration, and a security review of your entire XML parsing logic. Months of work, serious ongoing maintenance.

The right answer for most platforms: an identity broker. Configure Auth0, Okta, or PingFederate as a layer that speaks SAML upstream (towards your customer's IdP) and OIDC downstream (towards your platform). Your platform remains OIDC-native. The broker handles the translation. You get enterprise SAML support in days, not months.

**Step 3: If you build natively, be explicit about the attack surface you are accepting.**

```
Attack surface item           Mitigation required
──────────────────────────────────────────────────────────────────────
XML Signature Wrapping (XSW)  Vetted SAML library. Reference URI processing.
                              Schema validation before any XML parsing.
Certificate expiry            Monitoring at 90/30/14/7/1 days.
                              Zero-downtime rotation procedure documented.
Assertion replay              Redis cache of assertion IDs with TTL.
Attribute mapping failures    Contract agreed in writing before Day 1.
                              SAML-tracer debugging on every test login.
```

**Step 4: Commercial framing — give the customer the choice with real numbers.**

"OIDC via identity broker: 2 days to configure, no ongoing maintenance from your team, no new attack surface. SAML-native: 8-10 weeks development + security review + ongoing cert management. Both give you SSO against PingFederate. Which would you like?"

> 💬 *"Enterprise customers often ask for SAML because it's what their IT team knows how to configure. When you explain that your OIDC implementation gives them exactly what they want (SSO against their IdP) without requiring XML configuration on their side, 70% of them say 'oh, in that case OIDC is fine.' The 30% who still need SAML either have legacy apps that truly require it or have compliance requirements that specify it. Know which you're dealing with before starting."*

---

### Q4 — PKCE: why it's now mandatory for ALL clients

**The scenario:** It's Monday morning code review. A senior developer has opened a PR for your new server-side web app's OAuth integration. The implementation uses Authorization Code flow with a `client_secret`. There is no PKCE. The developer says: "We're a confidential client with a server-side secret. PKCE is only for mobile apps and SPAs. This is fine."

**❓ "OAuth 2.1 mandates PKCE for ALL Authorization Code flows — including server-side confidential clients with a client_secret. The developer says it's not needed. Who is right and why?"**

**💬 What most people say:** "OAuth 2.1 requires it, so you must add it." This is correct but explains nothing.

**⭐ What a Principal says:**

The developer is wrong — but understanding *why* is what matters.

The `client_secret` protects the token **exchange**. PKCE protects the auth **code** before it ever reaches the exchange. They guard completely different points in the flow.

**The specific attack PKCE prevents on a confidential server-side client:**

```
Without PKCE:
  1. Your server-side app has an open redirect vulnerability in its
     callback handler (common — easy to introduce, hard to find)
  2. Attacker crafts: /authorize?...&redirect_uri=https://yourapp.com/redirect?next=https://evil.com
  3. Auth Server redirects auth_code to yourapp.com/redirect
  4. yourapp.com/redirect follows the 'next' param to evil.com
  5. Attacker now has the auth_code
  6. Attacker submits: code + your client_secret (obtained via source leak, or guessed)
  7. Token issued. Game over.

With PKCE:
  5. Attacker has the code
  6. Attacker does NOT have the code_verifier (generated at request time, never logged)
  7. SHA256(attacker_guess) ≠ code_challenge → exchange rejected ✓
```

Additionally: server access logs capture full URLs. An auth code in a redirect URL is in every NGINX log, every CDN log, every WAF log. PKCE makes a logged code completely useless.

**The conceptual point that seals it:**

`client_secret` is static, long-lived, and shared across all sessions. `code_verifier` is generated fresh for every single authorization request, never transmitted over the front channel, and expires when the flow completes. They protect against different attack vectors and are not redundant.

> 🎯 **What the interviewer is really testing:** Do you understand the specific attack each mechanism prevents, or do you just know the rules? A Principal explains security requirements in terms of threats, not compliance checkboxes.

---

### Q5 — Refresh token rotation and the theft detection window

**The scenario:** Your security team reviews your Auth Server implementation. They want refresh token rotation enabled. A developer asks: "Rotation means a user's refresh token changes on every use. If a token is stolen, how exactly does the system detect it? Is this actually effective?" Great question. Most developers cannot answer it precisely.

**❓ "Refresh token rotation is enabled. A user's refresh token is stolen. Walk me through exactly when and how your system detects the theft."**

**⭐ Principal-level answer:**

The system detects theft only when **both** the attacker **and** the legitimate user try to use the refresh token — specifically when the legitimate user tries to use a token the attacker already rotated.

```
T=0:  Auth Server issues refresh_token_1 to the user

T+1h: Attacker steals refresh_token_1
      Both attacker and legitimate user now hold refresh_token_1

T+2h: Attacker uses refresh_token_1:
      → Auth Server issues refresh_token_2 to the attacker
      → refresh_token_1 is marked USED/REVOKED in the token family

T+3h: Legitimate user's app tries to use refresh_token_1:
      → Auth Server: "This token was already rotated."
      → Auth Server: "Someone used it before you — this is a theft signal."
      → Auth Server REVOKES THE ENTIRE TOKEN FAMILY
      → Both attacker and legitimate user are forced to re-authenticate

Detection event: the moment the legitimate user tries to use the rotated token.
```

**The detection window problem:** If the attacker acts quickly and the user's next refresh is scheduled hours away, the attacker has hours of access before detection. This is why short access token expiry matters — the attacker can only use access tokens during their validity window (typically 15-60 minutes).

**The race condition nobody talks about:**

```
Legitimate concern: what if two concurrent requests from the legitimate user
both trigger a refresh simultaneously?

Request A: submits refresh_token_5 → gets refresh_token_6
Request B: submits refresh_token_5 simultaneously → already rotated!
Auth Server: detects "replay" → revokes family → legitimate user logged out
```

Fix: mutex around refresh logic in your SDK. Only one refresh per client at a time.

> 💬 *"Refresh token rotation is not a theft-prevention mechanism — it is a theft-detection mechanism. The attacker may still have hours of access. What rotation does is guarantee you will eventually know the theft happened, and lets you revoke aggressively once you do. Pair it with short access token expiry to tighten the window."*

---

### Q6 — Token Exchange: the missing link in microservice auth

**The scenario:** Your platform has 20 microservices. Service A receives a user's access token (the user clicked "submit order"). Service A needs to call Service B (fraud check). Service B needs to know the call is on behalf of user_123 for audit purposes. Your developer says: "I'll just forward the user's token to Service B." You've seen this before. It ends badly.

**❓ "Service A gets a user token. It needs to call Service B. The developer proposes forwarding the user's token. What is wrong with this and what is the correct solution?"**

**⭐ Principal-level answer:**

Forwarding the token is wrong on three levels:

**Level 1 — It will break** (if Service B validates correctly): The user's token has `aud=service-a`. Service B's validator checks audience. Mismatch → 401. If Service B does NOT check audience — that is itself a security bug.

**Level 2 — It creates scope for future exploits:** If you build a system where any service can forward any user's token to any other service, you have created an audience confusion attack surface. Service B trusts service-a tokens. Now any compromised service can impersonate any user at Service B.

**Level 3 — It destroys your audit trail:** Your logs show: "user_123 called Service B directly." But user_123 didn't. Service A called Service B on behalf of user_123. When your compliance team asks "show me every direct action user_123 took against Service B", you have noise in your audit logs that makes the answer impossible.

**The correct solution — RFC 8693 Token Exchange:**

```
Service A calls Auth Server:
  POST /token
  grant_type=urn:ietf:params:oauth:grant-type:token-exchange
  subject_token=<user's access token>
  audience=https://service-b.example.com

Auth Server issues a NEW token:
  {
    "sub": "user_123",                          ← original user preserved
    "aud": "https://service-b.example.com",     ← correctly scoped to Service B
    "act": { "sub": "service-a" },              ← service-a is the actor
    "scope": "fraud.check",                     ← only what Service B needs
    "exp": now + 300
  }

Audit log now reads:
  "service-a called service-b on behalf of user_123 with scope fraud.check"
  Clean, complete, unambiguous.
```

**When NOT to use Token Exchange (use Client Credentials instead):**

If Service B is a pure utility — a PDF renderer, a notifications dispatcher, a metrics collector — and its response is identical regardless of which user triggered the call, Client Credentials is simpler and correct. Token Exchange is for when user identity matters downstream.

---

### Q7 — SAML Circle of Trust admission process

**The scenario:** Your company's security team receives an email from a new SaaS vendor: "Please add us to your PingFederate Circle of Trust. We have attached our SP metadata." Your SAML admin is about to import the metadata and create the SP connection. You are asked to review the process. Most companies just import it. That is the wrong answer.

**❓ "A new SaaS vendor wants to be admitted to your SAML Circle of Trust. Walk me through your review process. What do you check, what could go wrong, and what is the blast radius of getting it wrong?"**

**⭐ Principal-level answer:**

**What you review in the SP metadata:**

```
Item                          What you check                           Why it matters
──────────────────────────────────────────────────────────────────────────────────────
ACS URL                       HTTPS only. No HTTP.                     Plain HTTP = assertion in cleartext
Certificate                   From a legitimate CA. Not self-signed.   Self-signed = anyone can regenerate it
                              Not expired.                             Expired = hard 3am failure in 90 days
Entity ID                     Matches their own config exactly.        Mismatch = AudienceRestriction failure
SLO support                   Do they respond to LogoutRequests?       No SLO = sessions persist after leaver
NameID format                 If emailAddress: document the risk.      Email changes = duplicate accounts
Attributes requested          Minimum needed for their function.       Data minimisation — GDPR
```

**What you ask the vendor (before importing anything):**

- Do you validate assertion signatures before processing?
- Do you maintain an assertion ID cache for replay prevention?
- Do you validate InResponseTo on SP-initiated flows?
- What is your certificate rotation procedure?

**Blast radius if you get it wrong:**

A vendor with XSW vulnerability in their SP + your IdP sends admin-level role assertions = an attacker who compromises the vendor's SAML processing can forge an assertion from your IdP and log in as any of your users on the vendor's platform. Your IdP signing key compromise simultaneously affects every single SP in your Circle of Trust.

**The gate to add:**

A written security questionnaire for SP admission. If they fail any of: signature validation, replay prevention, TLS enforcement, SLO implementation — they get a remediation list, not a CoT entry. Treat it like a vendor security review, because that is exactly what it is.

---

### Q8 — The OAuth 2.1 migration audit

**The scenario:** Your engineering director reads that OAuth 2.1 is coming and asks you to assess what your existing platform needs to change. Most engineers treat this as a checkbox exercise. A Principal treats it as an opportunity to find everything in your system that was always slightly wrong.

**❓ "OAuth 2.1 is a draft. Should we build to it today? And specifically, what breaks in our existing OAuth 2.0 implementations when we try to comply?"**

**⭐ Principal-level answer:**

**Yes, build to OAuth 2.1 today.** Not because it is a final standard, but because it codifies 10 years of security best practices that are already required for a defensible system. If you are not OAuth 2.1 compliant, you are probably running at least one known-vulnerable configuration.

**What breaks — and how bad each migration is:**

```
Removal: Implicit grant (response_type=token)
  Who has it: SPAs built 2013-2019 using angular-oauth2-oidc, older auth libraries
  Symptom: access token in URL fragment → browser history → logs → referrer headers
  Migration effort: medium — add a callback handler, switch to Auth Code + PKCE
  Migration risk: some legacy SPAs have no server component → BFF may be needed

Removal: Resource Owner Password Credentials (ROPC)
  Who has it: legacy mobile apps, "trusted first-party" integrations, CLI tools
  Symptom: app collects user password → transmits to token endpoint → defeats OAuth's purpose
  Migration effort: high — requires system browser redirect, product team resistance
  Migration risk: the PM will say "our users don't want to see a browser popup"
                  (They are wrong. Users on every major platform expect system browser auth.)

New requirement: PKCE mandatory for ALL clients
  Who is missing it: confidential server-side apps that relied on client_secret alone
  Migration effort: low — usually one library config line
  Migration risk: near zero

New requirement: exact redirect URI matching
  Who is affected: anyone using wildcard URIs (https://yourapp.com/*)
  Migration effort: low — audit registered URIs and make them exact
  What you find: URIs registered years ago by developers who no longer work there.
                 Some of them are terrifying.

New prohibition: tokens in URL query strings
  Who has it: any integration using ?access_token=... in API calls
  Migration effort: low — switch to Authorization header
  What you find: server logs that have been capturing user tokens for years
```

> 💬 *"Doing the OAuth 2.1 audit is like pulling a thread on your authentication layer. You will find things that have been wrong for years and nobody noticed because they never failed visibly. That is exactly why you do it before you need to — not during an incident."*

---

### Q9 — Session cookie vs refresh token: the architectural difference

**The scenario:** A senior developer joins your team from a company that built everything with server-side sessions. They are implementing your new OIDC-based web app. They ask: "Can I just use the refresh token as my session cookie? They're both stored server-side and both let the user stay logged in. What's the difference?" It's a brilliant question that almost nobody asks.

**❓ "What is the precise architectural difference between a session cookie and a refresh token? Could you replace one with the other?"**

**⭐ Principal-level answer:**

```
                  Session Cookie              Refresh Token
──────────────────────────────────────────────────────────────────────
Issued by         Your application            The Auth Server
Validated by      Your session store (Redis)  The Auth Server
Gets you          Your app's session data      A new access token
Scope             Your domain only            The Auth Server's domain
Revoke by         Delete from your Redis      Call RFC 7009 revocation endpoint
Auth Server call  None per request            One per access token refresh
Lifecycle         Tied to your app's state    Tied to Auth Server's state
Format            Opaque string               Opaque string (usually)
```

**Could you replace one with the other? Technically yes. Should you?**

You *could* use a refresh token as a session proxy — store it in an httpOnly cookie, have your BFF exchange it for an access token on every request, and treat the Auth Server as your session store. This is a valid architecture pattern.

**Trade-offs:**
- Every page that needs auth now makes a round-trip to the Auth Server → latency
- Auth Server outage = your app has no active sessions → single point of failure
- Refresh tokens are designed for occasional use, not per-request

**The right answer for most apps:**

Both, serving different purposes. The session cookie (httpOnly, SameSite=Strict) tells your app who the user is. The refresh token (held server-side in the BFF, never exposed to the browser) lets you silently renew the access token. The user's browser never sees either token — just the session cookie reference. Tokens are in the BFF's memory.

> 🎯 **What the interviewer is really testing:** Do you understand the difference between your application's session state and the Auth Server's delegated credential? Many developers conflate them. Separating them is a sign of architectural maturity.

---

### Q10 — Explain OAuth 2.0 to a non-technical executive

**The scenario:** You are presenting at a board meeting. The CFO leans forward: "I keep hearing about OAuth and OIDC. What exactly are they and why are we spending engineering time on them? Can you explain it in plain English in under a minute?" The room goes quiet. Every engineer in the room is hoping someone else speaks first.

**❓ "Explain OAuth 2.0 to a non-technical executive. 60 seconds. No acronyms."**

**⭐ Principal-level answer:**

*"Imagine you have a parking valet service. Instead of giving the valet your house keys — because they only need to park your car, not access your house — you give them a valet key. A valet key only opens the car door and the ignition. It cannot open your house. If the valet loses it, you cancel that key. Your house is safe.*

*OAuth is the valet key system for software. When our app says 'sign in with Google,' we are not asking for your Google password. We are asking Google to give our app a valet key: it can access your calendar for today's meeting — nothing else, and only until tomorrow. Your password stays with Google. We never see it.*

*Without this system, every app we integrate with — payment processors, analytics tools, partner platforms — would need a copy of your credentials. One breach at any of them exposes everything. With OAuth, a breach at a third party gets them a temporary, limited key that expires. Your accounts stay safe. That is what we are building."*

**Why this answer works:**

It uses a physical analogy the exec has experienced. It explains the business risk that OAuth prevents (credential sharing → cascade breaches). It ends with the business outcome (protection). It took 45 seconds. It never said "authorization server" or "grant type."

> 🎯 **What the interviewer is really testing:** Can you communicate technical concepts to non-technical stakeholders without condescension or jargon? This is a core Principal Engineer skill — you will do this in every quarterly business review.

---

## Category 2 — System Design Walkthroughs 🏗️

> *"The Senior Engineer builds what was asked. The Principal Engineer asks what you actually need before building anything."*

**What this category tests:** Whether you can architect IAM systems under real constraints — scale, security, developer experience, failure modes, and migration paths. Every walkthrough starts with the clarifying questions you should ask before drawing a single box.

---

### Design 1 — Multi-tenant SSO for 500 enterprise customers

**❓ "Design an SSO platform that supports 500 enterprise customers, each with their own corporate IdP. Customers use a mix of Okta, Azure AD, PingFederate, and legacy ADFS."**

**The Principal's first move:** Ask before designing.

```
Questions that change the architecture:
  1. Do you own the IdP or are you the SP?
     (Are you PingFederate, or are you Salesforce?)
  2. What protocols must you support — SAML, OIDC, or both?
  3. What is your tenant isolation requirement?
     (Shared signing keys? Separate audit logs? Data residency?)
  4. What happens when a customer's IdP goes down?
     (Do you fail closed or allow fallback authentication?)
  5. How do customers onboard? Self-service or assisted?
```

**The architecture:**

```
Customer A (Okta — OIDC)  ────────────────────────────────────────────┐
Customer B (Azure AD — OIDC) ─────────────────────────────────────────┼──► Your Platform
Customer C (PingFed — SAML only) ──► Identity Broker ──► OIDC ────────┤   (OIDC-native)
Customer D (ADFS — legacy) ────────► Identity Broker ──► OIDC ────────┘

Key decision: your platform stays OIDC-native.
Customers who speak SAML get an identity broker layer (Auth0, Okta, or PingFed)
that translates SAML → OIDC. You never build SAML parsing logic.
```

**Tenant isolation — the four critical decisions:**

```
1. Per-tenant signing keys
   Each tenant gets their own RSA key pair in KMS/HSM.
   JWKS endpoint: https://auth.yourplatform.com/{tenant_id}/.well-known/jwks.json
   Key compromise for Tenant A: zero impact on any other tenant.

2. Per-tenant OIDC issuer
   Token issuer (iss): https://auth.yourplatform.com/{tenant_id}
   A token from Tenant A submitted to Tenant B's API fails iss validation.
   Tenant isolation is enforced at the protocol level, not just the application layer.

3. Per-tenant audit log streams
   Tenant A's security team must never see Tenant B's authentication events.
   Physical separation in your log pipeline, not just a filter.

4. Per-tenant rate limits
   Tenant A hammering your token endpoint doesn't degrade Tenant B's auth experience.
```

**Routing each login to the correct IdP:**

```
Method: email domain discovery

User enters: jane@goldman.com on your platform's login page
Platform: SELECT idp_config FROM tenants WHERE email_domain = 'goldman.com'
Result: Goldman's Okta OIDC configuration
Platform redirects: /oauth/authorize?tenant=goldman&...
Callback arrives: /callback?tenant=goldman
Platform validates the token against Goldman's tenant-specific OIDC config

Edge cases you must handle:
  Domain not found: show generic login + email/password fallback (if configured)
  Customer IdP down: per-customer policy — fail open (allow fallback) or fail closed (block)
  Multiple email domains per tenant: a company post-merger may have 3 domains all routing to one IdP
```

**Failure modes and mitigations:**

```
Customer IdP outage:
  Impact: all that customer's users cannot authenticate
  Mitigation: this is NOT your SLA — document it clearly in customer agreements
  Optional: emergency access codes for customer admins (break-glass accounts)

Your OIDC broker layer outage:
  Impact: SAML-only customers cannot authenticate
  Mitigation: multi-region active-active broker, health checks, failover

Your token endpoint outage:
  Impact: new logins fail, but existing valid tokens continue working (stateless JWTs)
  Users with tokens get seamless experience for up to token lifetime
  Mitigation: multi-region Auth Server, circuit breaker in clients
```

**Onboarding a new customer in self-service:**

```
Customer IT admin:
  1. Logs into your admin portal
  2. Selects "Configure SSO"
  3. Uploads their IdP metadata (SAML) or enters their OIDC discovery URL
  4. Configures attribute mapping (what their IdP sends vs what your platform expects)
  5. Tests with a pilot user
  6. Enables SSO for all users

Target: 30-minute self-service onboarding for OIDC customers.
         2-hour assisted onboarding for SAML customers (attribute mapping negotiation).
```

---

### Design 2 — Zero-trust authentication for microservices

**❓ "A bank wants zero-trust architecture. Every internal API call — including service-to-service — must be authenticated and authorised. No implicit trust. Design the identity layer."**

**Zero-trust first principles:**

```
Never trust, always verify.
Every call is authenticated. Every call is authorised. Every call is logged.
The network perimeter is NOT a trust boundary.
An attacker who is already inside the network gets nothing for free.
```

**Service identity options:**

```
Option A: mTLS (Mutual TLS)
  Every service has a certificate. Client presents cert. Server verifies.
  Identity = the certificate subject (CN=order-service, O=bank).
  SPIFFE/SPIRE: infrastructure for issuing short-lived service identity certificates (SVIDs).
  Rotates automatically every 24 hours. No long-lived secrets on disk.
  ✅ Strong service identity. Works at the transport layer.
  ⚠️ Certificate distribution infrastructure needed. Hard to debug.

Option B: Short-lived JWTs via Client Credentials
  Every service has a registered client_id.
  Each call: fetch or reuse a cached 5-minute JWT, attach as Bearer.
  No refresh token — just re-request when expired.
  ✅ Familiar JWT tooling. Works everywhere HTTP works.
  ⚠️ Auth Server is on the critical path. Caching essential.

Option C: Both (recommended for a regulated bank)
  mTLS at the transport layer: proves the service is who it says it is.
  JWT at the application layer: carries scopes and user context.
  mTLS catches network-level service spoofing.
  JWT catches application-level bypass and carries audit context.
  Belt and braces.
```

**Token design for zero-trust:**

```
Service-to-service with no user context:
  POST /token (Client Credentials)
  {
    "sub": "order-service",
    "aud": "https://payment-api.bank.com",
    "scope": "payment.initiate",
    "exp": now + 300  // 5 minutes only
  }

Service-to-service with user context (Token Exchange):
  POST /token (Token Exchange RFC 8693)
  subject_token = <original user's token>
  audience = https://fraud-api.bank.com
  {
    "sub": "user_456",               // original user preserved
    "aud": "https://fraud-api.bank.com",
    "scope": "fraud.check",
    "act": { "sub": "order-service" }, // service-a is the actor
    "exp": now + 300
  }
```

**The clock skew problem at scale:**

```
Problem: 50 microservices, some in different regions, some in containers.
         JWT with exp=now+300 fails if validator's clock is 61 seconds ahead.
         Intermittent 2% failure rate that can't be reproduced locally.

Fix:
  1. NTP synchronisation on ALL hosts — non-negotiable infrastructure hygiene
  2. clockTolerance: 60 on all JWT validators (60-second grace window)
  3. Monitor clock drift — alert when any service > 30 seconds off
  4. In Kubernetes: containers inherit host clock — fix at host level, not container level

Never do: increase token expiry to mask the clock problem.
You are trading a timing bug for a larger security window. Fix the clock.
```

---

### Design 3 — Token endpoint under extreme load

**❓ "Your token endpoint is a bottleneck at 50,000 requests per second during peak. It is also a single point of failure. Redesign it for scale and resilience."**

**The Principal's first question — is the problem real?**

```
50,000 token requests per second sounds alarming.
But first: WHY are they requesting tokens so frequently?

Audit your clients:
  Are they caching tokens? (Most are not — this is the actual problem)
  What is the average token lifetime? (3,600 seconds = 1 hour)
  50,000 req/s ÷ 3,600s = implies 180 million active sessions
  If your DAU is 500,000, something is catastrophically wrong with client caching.

Real experience: fix client-side caching first.
"50,000 req/s" drops to 3,000 req/s when clients cache correctly.
Add token_cache_hit_rate to your metrics dashboard. Watch it.
```

**When the load is genuinely real — the scaling layers:**

```
Layer 1: Stateless horizontal scaling (immediate win)
  JWT issuance is stateless. No shared database read required to issue a token.
  Each Auth Server node loads the signing key from KMS at startup, signs in memory.
  20 nodes behind a load balancer → linear throughput scaling.
  Add nodes. Done. No coordination required.

Layer 2: Regional distribution (latency and resilience)
  Token issuance in the same region as the requesting service.
  US services → US-East Auth Server (50ms)
  EU services → EU-West Auth Server (50ms, not 200ms across the Atlantic)
  Challenge: all regions must serve the same public keys (or per-region keys).
  Solution: replicate JWKS globally via CDN. JWKS is public — safe to cache everywhere.

Layer 3: HSM considerations for regulated environments
  Regulated banks: private key never leaves the HSM.
  HSM becomes the bottleneck (typically 10,000 signing operations per second).
  Solution: HSM cluster with load balancing, or sign with ephemeral derived keys.

Anti-patterns — these will hurt you:
  ❌ Cache token responses at a reverse proxy
     (Breaks per-user isolation. Multiple users get the same token.)
  ❌ Single HSM with no failover
     (One hardware failure = total auth outage for the entire platform.)
  ❌ Synchronous database write on every token issuance
     (Database becomes the bottleneck. Keep issuance stateless.)
```

---

### Design 4 — IAM architecture for a 50,000-employee global bank

**❓ "Design the complete IAM architecture for a 50,000-employee global bank. They have on-premises AD, multiple cloud tenants, 200+ applications, and a Zero Trust mandate from the board."**

**The full-stack architecture:**

```
GOVERNANCE LAYER (decides who SHOULD have access):
  IGA Platform: SailPoint IdentityNow
  Functions: access request + approval, quarterly certification campaigns,
             SoD enforcement, orphaned account detection, compliance reporting
  Triggers: joiner/mover/leaver workflows → downstream provisioning

IDENTITY LAYER (who users ARE):
  On-premises AD → source of truth for 50,000 employee identities
  Azure AD (Entra ID) → cloud-synced via AD Connect
  HR System (Workday) → org structure, manager hierarchy, cost centres
  Service accounts → managed in AD, rotated via PAM (CyberArk)

FEDERATION LAYER (the authentication broker):
  PingFederate cluster:
    4 nodes, active-active, 2 geographic regions, load balanced
  Connected to:
    On-premises AD (LDAP for authentication + attribute reads)
    Azure AD (OIDC for cloud workloads)
    Partner org IdPs (SAML for B2B federation)
  Speaks:
    SAML 2.0 → legacy enterprise SaaS (SAP, Oracle EBS, Bloomberg)
    OIDC → modern SaaS (Salesforce, ServiceNow, Workday)
    OAuth 2.0 → internal APIs (microservices, REST services)
  Pattern: hub-and-spoke — 5 upstream IdPs + 200 downstream apps = 205 connections
           (NOT mesh, which would be 5 × 200 = 1,000 connections)

APPLICATION LAYER:
  Legacy enterprise SaaS: SAML 2.0 (no choice — these apps require it)
  Modern SaaS: OIDC preferred (SAML fallback where required)
  Internal web apps: OIDC + BFF pattern (tokens never in browser)
  Internal APIs: OAuth 2.0 Bearer JWTs (5-minute expiry, no refresh)
  Mobile apps: OIDC + PKCE + OS Keychain (iOS) / Keystore (Android)
  CLI tools: Device Code flow

PROVISIONING:
  IGA → AD groups (everything downstream reads AD groups)
  SCIM 2.0 → 120 of 200 apps (proactive — account exists on Day 1)
  JIT via SAML attributes → 80 apps (reactive — account on first login)
  Leaver SLA: AD disabled within 15 minutes of HR termination event (automated)

ZERO TRUST OVERLAY:
  Every internal API call: OAuth 2.0 Bearer token (5-min expiry)
  High-value service-to-service: mTLS + JWT
  Privileged access: PAM session recording, 4-hour token expiry
  No east-west trust based on network position alone
```

**Day 1 for a new employee (the test of your architecture):**

```
Sunday 8pm:  IGA triggers SCIM provisioning
             Salesforce account: created with correct role and territory
             ServiceNow: created with itil role
             Workday: created with employee self-service
             32 more apps provisioned automatically

Monday 8:15: IT creates AD account, adds to role groups
Monday 9:00: Jane arrives → logs in with Windows credentials
             All 200 app tiles visible → SAML/OIDC SSO works for all ✅

Termination on Thursday afternoon:
  14:30: HR clicks "Terminate" in Workday
  14:30: IGA triggers AD account disable (automated, < 2 minutes)
  14:30: ALL SAML and OIDC SSO fails instantly (IdP checks AD on every auth)
  14:35: SCIM sends DELETE to all 120 SCIM-connected apps
  15:30: Audit confirms zero active access
```

---

### Design 5 — Developer-facing Auth SDK for 10,000 developers

**❓ "You are building an Auth SDK used by 10,000 external developers integrating OAuth/OIDC into their apps. What are the design decisions that will make or break developer adoption?"**

**The 5 decisions that matter most:**

**Decision 1: Secure by default — no opt-out.**

```
Wrong design:
  new AuthClient({ clientId, redirectUri, pkce: true, state: true, nonce: true })

Right design:
  new AuthClient({ clientId, redirectUri })
  // PKCE: always on internally, no option to disable
  // state: auto-generated UUID, stored, validated — developer never sees it
  // nonce: auto-generated, included, validated — developer never sees it

The secure path IS the easy path.
Security is not a flag you set. It is the only way the SDK works.
```

**Decision 2: Errors that name the fix.**

```
Wrong: throw new Error('invalid_request')
       (Developer googles for 2 hours. Finds nothing relevant.)

Right: throw new AuthError({
  code: 'MISSING_CODE_CHALLENGE',
  message: 'Authorization Code flow requires a code_challenge.',
  howToFix: 'Ensure pkce is not disabled in your AuthClient config.',
  docs: 'https://docs.yourplatform.com/pkce',
  rfc: 'https://www.rfc-editor.org/rfc/rfc7636'
})
```

**Decision 3: Tokens are never the developer's problem.**

```
Developer writes:
  const data = await auth.fetch('https://api.yourplatform.com/resource');

SDK handles internally:
  Is there a valid access token in memory? Use it.
  Is the token expiring in < 60 seconds? Refresh silently first.
  Did the refresh fail? Redirect to login.
  Attach Authorization: Bearer <token> header automatically.

Developer never handles the token string unless they explicitly ask.
If they call auth.getAccessToken():
  return the token BUT log a console.warn explaining the security implications.
```

**Decision 4: The sandbox that works in 5 minutes, not 5 days.**

```
docker run -p 4000:4000 yourplatform/sandbox

Container includes:
  A running Auth Server with a test tenant pre-configured
  A sample React app already integrated with the SDK
  A sample API that validates tokens
  Pre-seeded test users (user@example.com / password123)
  HTTPS via self-signed cert (accepted by the sample app)

Developer goal: first successful login in under 5 minutes from pulling the image.
Measure this with analytics. If median time > 10 minutes, the sandbox is broken.

The 3-day integrations happen because of environment setup, not protocol complexity.
The sandbox eliminates environment setup entirely.
```

**Decision 5: Migration guides for the top 3 competitors.**

```
"Coming from Auth0? Here is what changes."
"Coming from Okta? Here is the concept mapping."
"Coming from Cognito? Here is what works differently."

Developers switching platforms don't want to learn from scratch.
They want to understand the delta.
Serve that need or they will stay on the competitor.
```

---

## 🌉 Bridge to Document 2

**What you just covered in Document 1:**

You now understand the protocols at depth — not just what they do, but where they fail, why design decisions exist, and how to architect systems around their limitations. You can answer protocol questions with trade-off analysis instead of definitions. You can design IAM systems from scratch under real enterprise constraints.

**The gap this leaves:**

You know how these systems work when everything goes right. Document 2 answers the harder questions:

- What happens when an attacker gets their hands on a valid SAML assertion?
- How do you walk through the exact steps of an algorithm confusion attack against a JWT implementation?
- It's 3am. PagerDuty just fired. SSO is down for 8,000 employees. What is your diagnostic tree?

**Document 2 covers:**
- **Category 3** — 12 attack scenarios (you ARE the attacker — walking through each exploit step by step)
- **Category 4** — 12 live incident war rooms (you are on-call, the systems is broken, here is how you fix it)

> 💬 *"The protocols make sense when you understand why they exist. The attacks make sense when you understand how they fail. Document 2 will make you genuinely uncomfortable — which means it's working."*

---

*OAuth 2.0: RFC 6749 · PKCE: RFC 7636 · Token Exchange: RFC 8693 · JWT: RFC 7519 · JWKS: RFC 7517 · Introspection: RFC 7662 · Revocation: RFC 7009 · OIDC Core 1.0 · OAuth 2.1: draft-ietf-oauth-v2-1 · SAML 2.0: OASIS saml-core-2.0-os*

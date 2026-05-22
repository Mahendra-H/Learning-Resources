# IAM Interview Prep — Document 3 of 4
## IAM Product Architecture + Code Review

> **Series:** Principal Engineer · IAM Product Development · SAML + OIDC + OAuth 2.0

---

## 🌉 Bridge from Document 2

**What Documents 1 and 2 gave you:**
You understand the protocols at depth — design decisions, failure modes, trade-offs. You can think like an attacker, walk through exploits at the code level, and lead incident war rooms with a systematic diagnostic approach.

**The gap this leaves:**
Understanding and defending is different from designing and building. Document 3 asks harder questions: How do you build a multi-tenant Auth Server that 500 enterprise customers trust with their identities? How do you design signing key management so a breach of one tenant cannot affect any other? And when a colleague's PR contains broken authentication logic — can you find the bugs before they ship?

**This document covers:**
- **Category 5** — IAM product architecture: 8 deep design questions about building production-grade IAM systems. These are the questions that separate engineers who *integrate* IAM products from engineers who *build* them.
- **Category 6** — Code review: 6 real broken implementations. Find the bugs, explain the security impact, and write the correct version. This is what Principal Engineers do in every auth-related PR review.

> 💬 *"The difference between a Senior and a Principal isn't knowledge of the protocols. It's the ability to make design decisions that hold up under load, under attack, and under the weight of 50,000 users who depend on it every morning."*

---

## Category 5 — IAM Product Architecture 🧠

> *"Senior engineers integrate IAM products. Principal Engineers build them. These questions test which side of that line you're on."*

**What this category tests:** Whether you understand the engineering challenges of building IAM infrastructure — multi-tenancy, key management, developer experience, scaling, failure isolation, and audit trail design. Every question has trade-offs with no single correct answer. You are evaluated on how many dimensions you consider and how clearly you articulate the final recommendation.

---

### Product Q1 — Multi-tenant signing key management

**The scenario:** You are the lead architect on a new Auth Server platform. The platform will serve 500 enterprise customers. The security team's non-negotiable requirement: a breach of one customer's signing keys must have zero impact on any other customer. The platform team says: "Can't we just use one shared key pair? It's simpler." You have been asked to make the call and explain it.

**❓ "Design the signing key management architecture for a multi-tenant Auth Server where tenant isolation is a hard requirement."**

**💬 What most people say:** "Use per-tenant keys stored securely." This is correct but tells the interviewer nothing about how you would actually implement it.

**⭐ What a Principal says:**

**The architecture — per-tenant keys in KMS/HSM:**

```
Each tenant receives their own RSA key pair at tenant creation:
  Private key: generated inside AWS KMS / Azure Key Vault / HashiCorp Vault
               Never exported. Never exists outside the HSM/KMS.
               Signing operations happen INSIDE the KMS — you submit the payload, get back a signature.
  Public key:  Published at tenant-scoped JWKS endpoint

JWKS endpoint per tenant:
  https://auth.yourplatform.com/{tenant_id}/.well-known/jwks.json
  Tenant A's validators fetch Tenant A's endpoint.
  Tenant B's validators fetch Tenant B's endpoint.
  Cross-submission structurally impossible — different keys, different endpoints.

Token issuer (iss) per tenant:
  https://auth.yourplatform.com/{tenant_id}
  A token from Tenant A submitted to Tenant B's API fails iss validation.
  Tenant isolation enforced at the protocol level, not just the application layer.
```

**Blast radius analysis — the test of good key design:**

```
Scenario: Tenant A's signing key is compromised (e.g., found in a GitHub repo)

With shared keys:
  ALL 500 tenants are compromised simultaneously.
  Every token issued on the platform must be invalidated.
  Every user across all tenants forced to re-authenticate.
  50,000+ users. 500 customers. Mass incident.
  Regulatory notification for all 500 customers.

With per-tenant keys:
  ONLY Tenant A is affected.
  Rotate Tenant A's key (30 minutes with a practiced procedure).
  Force only Tenant A's users to re-authenticate (~500 users, not 50,000).
  Notify only Tenant A.
  499 other tenants: completely unaffected, zero downtime, zero notification needed.
```

**Key rotation per tenant (zero-downtime for one tenant, zero impact on others):**

```
Step 1: Generate new key pair for Tenant A in KMS
Step 2: Add new public key to Tenant A's JWKS response (two keys now listed)
Step 3: Switch Auth Server to sign Tenant A's new tokens with the new key
Step 4: Wait for existing tokens signed with old key to expire (max: token lifetime)
Step 5: Remove old key from Tenant A's JWKS response
Step 6: Delete old key from KMS

Total impact: Tenant A has a brief period where both keys are in JWKS (overlap).
Other tenants: not mentioned in any of these steps. Zero impact. Zero awareness.
```

**Scaling consideration:**

```
500 tenants × quarterly rotation = ~2 rotations per day.
Automation is mandatory. Build:
  Automated rotation job: monitors key age, triggers rotation when approaching expiry
  JWKS cache invalidation: notifies validators to refresh their key cache
  Customer notification: emails customer 30 days before scheduled rotation
  Audit log: every key operation logged with timestamp, operator, and reason

Manual key rotation does not scale. A rotation procedure that requires human
intervention for each of 500 tenants is a procedure that will eventually fail.
```

> 🎯 **What the interviewer is really testing:** Do you think about blast radius before you think about convenience? Shared keys are simpler. Per-tenant keys are correct. Understanding why the difference matters at scale — and being able to articulate the compromise clearly — is what makes this a Principal-level answer.

---

### Product Q2 — Graceful Auth Server degradation

**The scenario:** Your Auth Server goes down for 5 minutes during peak hours. Your platform VP asks the next morning: "What was the user impact? Could we have designed this better?" Most engineers say "we had an outage, we need more redundancy." A Principal says something more nuanced.

**❓ "Your Auth Server goes down for 5 minutes. Map the exact impact on every part of your system and describe how you design for graceful degradation."**

**⭐ Impact mapping — not all users are affected equally:**

```
Users attempting NEW logins during the outage:
  Impact: BLOCKED. Cannot authenticate. 100% failure.
  Why: login requires the Auth Server to issue new tokens.
  Duration: entire 5-minute outage.
  User experience: login page shows an error.

Users with VALID EXISTING JWTs:
  Impact: ZERO. Completely unaffected.
  Why: JWT validation is stateless — no Auth Server call needed.
       The API validates against the cached JWKS public key.
  Duration: these users notice nothing.
  This is the stateless advantage — 80-90% of your active users are in this bucket.

Users whose access tokens are ABOUT TO EXPIRE during the outage:
  Impact: their next refresh token exchange fails.
  Duration: depends on token expiry timing.
  User experience: API calls start failing with 401 after their token expires.

Users attempting SAML SSO:
  Impact: BLOCKED for new SSO flows.
  Existing SP sessions: unaffected (session cookie at SP level, not Auth Server).
  Duration: new logins blocked for 5 minutes.
```

**Designing for graceful degradation:**

```
Layer 1: Multi-region active-active Auth Server cluster
  Minimum: 2 regions, active-active with health-check-based failover.
  One region fails: load balancer routes to the other region within seconds.
  For a global platform: 3 regions (US, EU, APAC) with anycast routing.
  Target: eliminate the 5-minute outage entirely.

Layer 2: JWKS cache with stale-while-revalidate
  Normal JWKS cache TTL: 10 minutes.
  On failure to refresh: serve stale cached keys for up to 24 hours.
  Why safe: JWKS only changes on key rotation. Rotation is planned.
  Result: API validation continues working for 24 hours even if Auth Server is unreachable.

  Configuration:
  createRemoteJWKSet(jwksUri, {
    cacheMaxAge:          600_000,    // 10 min normal TTL
    cooldownDuration:  86_400_000     // 24h stale-on-error TTL
  });

Layer 3: Client-side token caching with proactive refresh buffer
  SDK caches access token until 5 minutes before expiry.
  During a 5-minute outage: most clients are not in the 5-minute refresh window.
  Users mid-session: never interrupted.
  Only users whose tokens expire DURING the outage experience impact.

Layer 4: Refresh token buffering at the BFF
  BFF holds the current access token.
  On access token expiry: attempts refresh → if fails → serves slightly-expired
  token for up to 60 seconds while retrying.
  User experience: zero interruption during brief Auth Server hiccups.

What you cannot gracefully degrade: new logins.
Authentication requires the Auth Server. Accept this. Document it.
Design your SLA: "New logins may be unavailable during Auth Server maintenance windows."
That is honest. That is acceptable. Everything else should degrade gracefully.
```

> 💬 *"A 5-minute Auth Server outage at 10am on a Monday should affect maybe 2-3% of your users — the ones who happen to be logging in or refreshing tokens during those exact 5 minutes. If it affects 80% of your users, your JWKS caching is broken. If it affects 50% of your users, your token lifetime is too short. Statelessness is a feature — make sure you are using it."*

---

### Product Q3 — Developer onboarding in 30 minutes

**The scenario:** Your platform is technically excellent. Your Auth Server is solid. Your security posture is strong. But your enterprise sales team keeps losing deals to Auth0 and Okta. The feedback from prospects: "Their developer experience is better. We can get a working integration in an afternoon." Your integration takes 3 days minimum. You have been asked to fix this.

**❓ "What are the specific design decisions that make or break developer adoption of an auth platform? How do you get a developer from zero to working integration in 30 minutes?"**

**⭐ The five decisions that actually matter:**

**Decision 1: The sandbox that works out of the box.**

```
The single biggest time sink in auth integration is environment setup.
"Is my OIDC config right? Is the Auth Server responding? Is it my code or the config?"
These questions waste hours. The sandbox eliminates them.

docker run -p 4000:4000 yourplatform/sandbox

Container contains:
  Running Auth Server with a pre-configured test tenant
  Sample React app already integrated with your SDK
  Sample Node.js API that validates tokens from the Auth Server
  Pre-created test users: user@example.com / Password123
  HTTPS via locally-trusted certificate (no browser warnings)
  Hot reload on code changes

Developer goal: first successful login within 5 minutes of running that command.
Instrument this: measure time-to-first-token in sandbox analytics.
If median TTFT > 10 minutes, the sandbox has a problem.

Auth0 does this. Okta does this. If you do not, you are losing deals before
the developer writes a single line of integration code.
```

**Decision 2: Secure by default with no API surface for insecurity.**

```
Wrong design (security is an option):
  new AuthClient({
    clientId,
    redirectUri,
    pkce: true,          // developer can set this to false
    state: 'random',     // developer can hardcode this
    nonce: generateNonce() // developer can skip this
  })

Right design (security is the only way):
  new AuthClient({ clientId, redirectUri })
  // PKCE: always on, no option to disable
  // state: auto-generated UUID, stored, validated — hidden from developer
  // nonce: auto-generated, included, validated — hidden from developer
  // Tokens: never returned as strings by default — only used via auth.fetch()

The developer cannot accidentally ship insecure auth.
The secure path is the only path.
This is the Auth0 model. It is correct.
```

**Decision 3: Errors that name the fix, not just the problem.**

```
Developer experience killer #1: errors that require a search engine to understand.

Wrong:
  AuthError: invalid_request (400)

Right:
  AuthError: {
    code:        'MISSING_PKCE_CHALLENGE',
    message:     'Authorization Code flow requires code_challenge.',
    description: 'Your authorization request did not include a code_challenge parameter.',
    fix:         'Ensure you are using an AuthClient from @yourplatform/sdk v2.0+, or add code_challenge_method: "S256" to your authorization request manually.',
    docs:        'https://docs.yourplatform.com/guides/pkce',
    rfc:         'https://www.rfc-editor.org/rfc/rfc7636'
  }

This error tells the developer: what happened, why it is required, exactly how to fix it,
and where to read more. A developer who sees this error never needs to file a support ticket.

Invest in error message quality. It is documentation that runs at the moment it is needed.
```

**Decision 4: Migration guides for every major competitor.**

```
"Coming from Auth0? Here is the concept mapping."
"Coming from Okta? Here is what changes."
"Coming from AWS Cognito? Here is the equivalent of each feature."

Developers switching platforms do not want to learn from scratch.
They want to understand the delta.
A migration guide that takes someone from Auth0 → your platform in one afternoon
is worth more than 10 pages of reference documentation.
```

**Decision 5: The "time to working hello world" metric.**

```
Define it: from "I just created my account" to "I have a working OIDC login
           in a sample app calling a sample API."

Measure it: add telemetry to your sandbox and developer dashboard.
Target: < 30 minutes for OIDC. < 2 hours for SAML.
Review it weekly. When it increases, find out why and fix it.

This metric is to developer experience what "time to first token" is to performance.
If you are not measuring it, you are guessing.
Auth0 has published their TTWHW internally as a key product metric.
It drives product decisions. Adopt it.
```

---

### Product Q4 — Tenant isolation failure: the entity ID collision

**The scenario:** Your platform stores SAML SP entity IDs per tenant. Tenant A's admin configures their SP with entity ID `https://app.example.com`. Tenant B's admin also tries to configure the same entity ID. Your platform lets them. Three weeks later, a security researcher finds that Tenant A's SAML assertions are sometimes accepted by Tenant B's applications. You have a tenant isolation failure in production.

**❓ "Walk me through exactly how this entity ID collision breaks tenant isolation, and how you design the system to prevent it at every layer."**

**⭐ The failure mode in detail:**

```
Both tenants registered: entityID = "https://app.example.com"

SAML assertion routing in your platform:
  IdP receives SAMLResponse → extracts AudienceRestriction value
  AudienceRestriction = "https://app.example.com"
  Platform query: SELECT tenant_id FROM sp_connections WHERE entity_id = ?
  
  Problem: two rows match. Which tenant's validation rules apply?
  Depending on your implementation:
    → Non-deterministic: random tenant's config is used
    → First-match: always Tenant A, even when Tenant B submitted the assertion
    → Error: 500 crash — assertion cannot be processed at all

  Worst case: Tenant A's assertion is processed against Tenant B's attribute mapping.
  Result: Tenant A's user lands in Tenant B's application context.
  Tenant B's data is now accessible to Tenant A's user.
  This is a serious cross-tenant data exposure.
```

**Prevention — layered:**

```
Layer 1: Database uniqueness constraint (catch it at registration)
  ALTER TABLE sp_connections ADD CONSTRAINT unique_entity_id UNIQUE (entity_id);
  
  When Tenant B tries to register "https://app.example.com":
    Database: UNIQUE constraint violation
    Platform: 409 Conflict
    Error message: "This entity ID is already registered on this platform.
                   Entity IDs must be globally unique.
                   If this is your organisation's entity ID, please contact support."
  
  Problem is caught at registration time, not at assertion processing time.
  Zero runtime impact.

Layer 2: Platform-generated entity IDs (eliminate the problem entirely)
  Instead of allowing arbitrary entity IDs, generate them for each SP connection:
    https://auth.yourplatform.com/saml/{tenant_id}/sp/{connection_id}
  
  Collision is structurally impossible.
  Each entity ID is unique by construction.
  Trade-off: customer cannot use their own entity ID in their own metadata.
  This is acceptable — your platform is the SP, so your platform controls the entity ID.

Layer 3: Tenant-scoped ACS URLs
  ACS URL per tenant: https://auth.yourplatform.com/saml/acs/{tenant_id}
  Even if entity IDs somehow collide, the ACS URL is different per tenant.
  The assertion POST arrives at a tenant-specific URL.
  Your ACS handler knows immediately which tenant to validate against.
  Belt and braces — prevents routing ambiguity even with entity ID collision.

Layer 4: Runtime collision detection (defence in depth)
  On every assertion received: verify entity_id maps to exactly ONE tenant.
  If it maps to zero or more than one: reject with 400, log, alert.
  This catches any collision that slipped through layers 1-3.
```

---

### Product Q5 — Audit log design for IAM

**The scenario:** Your platform is being evaluated by a large financial institution. Their security team asks for your audit log specification. They need to know: what events are captured, what data each event contains, how you prevent tampering, and how long logs are retained. Most platforms have an audit log. Few have one designed specifically for the compliance requirements of regulated industries.

**❓ "Design the audit log schema for an IAM platform serving financial services customers. What events, what fields, what tampering prevention, what retention?"**

**⭐ The complete design:**

**Events to capture — minimum for financial services:**

```
Authentication events:
  login_success, login_failure, logout
  mfa_challenge_sent, mfa_success, mfa_failure
  sso_assertion_issued (IdP side), sso_assertion_received (SP side)
  sso_assertion_rejected (with reason)
  passwordless_challenge_sent, passwordless_success

Token events:
  token_issued (type, scope, expiry)
  token_refreshed (previous JTI, new JTI)
  token_revoked (by whom, reason)
  token_introspected (by which resource server)
  token_validation_failed (reason: expired | bad_signature | wrong_audience | revoked)

Admin events:
  user_created, user_disabled, user_deleted
  role_assigned, role_revoked, role_escalated
  sp_connection_created, sp_connection_modified, sp_connection_deleted
  signing_key_rotation_initiated, signing_key_rotation_completed
  tenant_created, tenant_disabled

Security events (the critical ones):
  assertion_replay_detected (assertion ID already seen)
  xsw_attack_detected (structurally anomalous assertion)
  brute_force_detected (threshold login failures from single IP)
  refresh_token_family_revoked (possible theft — both parties used same token)
  signing_key_compromise_reported
```

**Required fields on every single event:**

```json
{
  "event_id":       "01HXK2M3N4P5Q6R7S8T9",
  "event_type":     "login_success",
  "timestamp":      "2024-04-18T09:30:00.123Z",
  "tenant_id":      "tenant_goldman_sachs",
  "actor_id":       "user_jane_smith_789",
  "actor_type":     "user",
  "actor_ip":       "203.0.113.42",
  "actor_ua":       "Mozilla/5.0 ...",
  "target_id":      "sp_salesforce_prod",
  "target_type":    "sp_connection",
  "result":         "success",
  "session_id":     "sess_abc123def456",
  "request_id":     "req_xyz789",
  "detail": {
    "auth_method":  "password+mfa",
    "mfa_type":     "totp",
    "token_id":     "tok_jti_value"
  }
}
```

**Tamper prevention — three options, pick based on compliance requirement:**

```
Option 1: Write-once storage (S3 Object Lock / WORM)
  Logs written to S3 with Object Lock enabled (WORM — Write Once Read Many).
  No API allows deletion or modification. Even your own admins cannot delete logs.
  The storage policy enforces immutability at the infrastructure level.
  Best for: regulated industries, SOC 2, ISO 27001, SOX compliance.

Option 2: Cryptographic chaining
  Each log entry includes a hash of all previous entries:
    entry.hash = SHA256(entry.data + previous_entry.hash)
  Any tampering breaks the chain — detectable by re-hashing from entry 1.
  Tamper detection is built into the log itself.
  Best for: high-assurance environments where you also need verifiable integrity.

Option 3: Real-time SIEM forwarding
  Every event is forwarded to an external SIEM (Splunk, DataDog, Elastic) in real-time.
  Even if your platform is fully compromised, the SIEM already has the events.
  Events cannot be "un-sent" retroactively.
  Best for: most enterprise platforms — simple, effective, already-bought infrastructure.

For a financial services platform: Option 1 + Option 3.
WORM storage for compliance evidence. SIEM for real-time alerting.
```

**Retention policy:**

```
Financial services: 7 years minimum (varies by jurisdiction and regulation)
Healthcare (HIPAA): 6 years minimum
GDPR: proportionate retention — balance security value against personal data minimisation
  Recommendation: anonymise PII after 2 years, retain metadata for 7 years

Tiered storage to manage cost:
  Hot (0-90 days):    Fast query, full search, used for active investigations
  Warm (90 days-1 year): Compressed, searchable, used for compliance reviews
  Cold (1-7 years):   Archived, not immediately searchable, used for legal holds
```

> 💬 *"An audit log that nobody can query is compliance theatre. Design the query interface alongside the schema. If your security team cannot answer 'show me every action taken by user X in the last 30 days' in under 30 seconds, the audit log is not fit for purpose."*

---

### Product Q6 — SCIM and SAML coordination at enterprise scale

**The scenario:** Your platform serves a 50,000-employee bank. 200 applications connected. SCIM provisioning for 120 apps. JIT via SAML for 80 apps. The IGA team reports three categories of recurring failures: some employees arrive on Day 1 with wrong roles, some terminated employees are still accessible in certain apps, and some role changes take 24+ hours to propagate. Each of these is a compliance risk.

**❓ "Describe the three failure modes and how you engineer around each one."**

**⭐ Failure mode 1 — Day 1 wrong role (provisioning race condition):**

```
What happens:
  08:00: HR creates employee record → IGA joiner workflow triggers
  08:01: SCIM begins provisioning Salesforce (parallel to AD group assignment)
  08:01: Salesforce account created with groups from AD at T=08:01
         Employee is in AllStaff group but NOT YET in Salesforce_SalesManager group
  08:03: IGA finishes assigning employee to Salesforce_SalesManager AD group
  09:00: Employee arrives, logs in via SAML
         SAML assertion: Role=Salesforce_SalesManager ✓
         Salesforce account: profile=StandardUser (set by SCIM at 08:01 before group was added)

Why it happens:
  IGA triggers SCIM provisioning and AD group assignment in parallel.
  SCIM runs faster than AD group propagation.
  SCIM provisions the account with incomplete group membership.

The fix — workflow sequencing:
  IGA workflow must be sequential, not parallel:
    Step 1: Create AD account
    Step 2: Assign ALL AD groups (wait for confirmation from AD)
    Step 3: Verify group membership is reflected in LDAP
    Step 4: ONLY THEN trigger SCIM provisioning

  Additionally: configure SCIM to do attribute sync on every SSO login.
  If SAML assertion says Role=SalesManager and SCIM account has StandardUser → SAML wins.
  This adds resilience — even if a race condition occurs, it auto-corrects on first login.
```

**⭐ Failure mode 2 — Terminated employee still accessible (deprovisioning lag):**

```
What happens:
  14:30: HR terminates employee → IGA leaver workflow triggers
  14:30: AD account disabled → all SAML SSO fails immediately ✓
  14:30: SCIM deprovisioning added to batch queue
  16:00: SCIM batch job runs → sends DELETE to 120 apps
  During 14:30-16:00: employee's app accounts still exist and are accessible
                      (if someone has their app-native credentials)

Why it matters:
  Some apps allow both SSO AND native login (username/password).
  If the employee knows their Salesforce password, they can log in directly for 90 minutes.
  This is a compliance gap. Leaver procedure says "immediate revocation."
  90-minute gap is not immediate.

The fix — real-time event-driven deprovisioning:
  On AD account disable event:
    Immediately publish an event to your provisioning bus
    SCIM connector subscribes → sends DELETE to all 120 apps in real-time
    Target SLA: app accounts deprovisioned within 5 minutes of AD disable

  How to implement:
    AD: configure a webhook or use a change notification listener (LDAP notification)
    IGA: configure termination workflow to trigger real-time SCIM event, not batch
    SCIM connector: process DELETE events immediately, not in batch windows

  For the most critical apps (trading systems, finance approvals):
    Per-request AD state check — app queries AD on every request
    Disabled AD account = instant block, regardless of active session
    This is the only way to achieve truly instant access revocation
```

**⭐ Failure mode 3 — Role changes take 24+ hours:**

```
What happens:
  Employee is promoted to manager at 10:00.
  IGA updates AD group at 10:02.
  SCIM sync interval for Salesforce: 24 hours (scheduled nightly batch).
  Employee's Salesforce access: still reflects old role until next day.

Why it matters:
  The employee cannot approve contracts until their role updates.
  Their team is blocked. Manager is frustrated. IT is blamed.

The fix — event-driven SCIM triggered by AD group changes:
  Configure your SCIM connector to trigger sync on AD group membership change events.
  Not on a schedule. On an event.

  AD group change at 10:02 → event published → SCIM connector receives event
  → sends PATCH to Salesforce with new role → Salesforce updated within 2 minutes

  This requires:
    An AD change notification mechanism (Azure AD Graph API, on-prem: LDAP notification)
    An event bus (Kafka, AWS EventBridge, Azure Service Bus)
    SCIM connectors that can process individual user update events, not just full syncs

  Result: role changes propagate within 2-5 minutes, not 24 hours.
  The employee can approve contracts before lunch.
```

---

### Product Q7 — Building a SAML SP library for developers

**The scenario:** Your company is open-sourcing its SAML SP library. 10,000 developers will use it. Based on your experience with SAML security vulnerabilities, you know that most developers who implement SAML from scratch make the same three dangerous mistakes. Your job is to design a library that makes those mistakes structurally impossible.

**❓ "Name the three most dangerous mistakes developers make implementing SAML, and explain how your library's design prevents each one."**

**⭐ Dangerous mistake 1 — Processing assertions by XML position (XSW vulnerability):**

```
How developers make it:
  const assertion = doc.getElementsByTagNameNS(SAML_NS, 'Assertion')[0];
  // Gets first assertion by position — attacker can inject a wrapper

How your library prevents it:
  The library NEVER returns a DOM element to the developer.
  Internally: assertion is found by the ID referenced in the Signature's Reference element.
  Externally: the developer receives a typed, validated result object.

  // What the developer sees and works with:
  const result = await saml.processResponse(samlResponse, storedRequestId);
  console.log(result.nameId);          // "jane@company.com"
  console.log(result.attributes.role); // ["Manager", "London_Branch"]
  console.log(result.sessionIndex);    // "_sess_7h5c4d3e"

  The developer never touches XML. They work with a typed object.
  XSW is architecturally impossible — the developer has no mechanism to trigger it.
```

**⭐ Dangerous mistake 2 — Skipping assertion replay prevention:**

```
How developers make it:
  "We'll add it later." They don't add it later.

How your library prevents it:
  Replay prevention is required at construction time.
  The library throws at startup if no cache provider is configured.

  // Library constructor:
  const saml = new SAMLProvider({
    idpMetadata: '...',
    spEntityId: '...',
    assertionCache: redisClient  // REQUIRED — no default, no optional
  });

  // If assertionCache is omitted:
  throw new ConfigurationError(
    'assertionCache is required. SAML assertion replay attacks are possible without it.\n' +
    'Provide a Redis client or any object with get(key) and set(key, value, ttl) methods.\n' +
    'See: https://docs.yourlib.com/security/replay-prevention'
  );

  Developers cannot skip replay prevention. The library refuses to start without it.
```

**⭐ Dangerous mistake 3 — Accepting IdP-initiated SSO without CSRF protection:**

```
How developers make it:
  They only implement SP-initiated SSO (the simpler flow).
  Then a customer asks for IdP-initiated (portal tiles).
  Developer adds support without understanding the CSRF risk.

How your library handles it:
  IdP-initiated is supported but requires explicit opt-in with a visible warning.

  // Default: SP-initiated only (safe)
  const saml = new SAMLProvider({ ... });

  // To enable IdP-initiated: explicit opt-in with csrf protection required
  const saml = new SAMLProvider({
    ...,
    idpInitiated: {
      enabled: true,
      csrfProtection: {
        tokenStore: sessionStore,  // REQUIRED when idpInitiated is enabled
        // Library embeds CSRF tokens in RelayState and validates them
      }
    }
  });

  // If idpInitiated.enabled = true but csrfProtection is not configured:
  throw new ConfigurationError(
    'IdP-initiated SSO requires CSRF protection. Configure csrfProtection.tokenStore.\n' +
    'See: https://docs.yourlib.com/security/idp-initiated-csrf'
  );
```

> 🎯 **What the interviewer is really testing:** Can you identify the most dangerous failure modes and design an API that prevents them structurally — not through documentation warnings but through code that refuses to compile or start without the right configuration? This is "pave the path" API design: the correct path is the easy path.

---

### Product Q8 — Mixed SAML and OIDC from one user session

**The scenario:** Your enterprise platform has a 10-year-old Salesforce integration (SAML only), a 3-year-old Workday integration (SAML), and a new React web app (OIDC). All three use PingFederate as the IdP. The requirement: a user who logs in once in the morning should access all three without re-authentication, regardless of which protocol each app uses. Additionally, logout from any one must terminate access to all three.

**❓ "How do you architect a single user session that spans SAML and OIDC applications simultaneously? What are the failure scenarios?"**

**⭐ The architecture:**

```
The key insight: PingFederate maintains a MASTER SESSION for the user.
This master session is protocol-agnostic — it represents the human's authentication event.
From this single master session, PingFed can issue either SAML assertions or OIDC tokens
without re-prompting the user.

                    User logs in once
                           ↓
              PingFederate creates MASTER SESSION
              {
                user: jane@bank.com,
                auth_time: 09:00,
                auth_method: password+mfa,
                session_id: _sess_abc123,
                expiry: 17:00
              }
                           ↓
        ┌──────────────────┼────────────────────┐
        ↓                  ↓                    ↓
  Salesforce            Workday            React App
  (SAML request)        (SAML request)     (OIDC request)
        ↓                  ↓                    ↓
  PingFed issues        PingFed issues     PingFed issues
  SAML assertion        SAML assertion     OIDC id_token
  (no re-auth)          (no re-auth)       + access_token
                                           (no re-auth)

The master session is the source of truth.
Protocol is just the delivery format of the authentication event.
```

**Cross-protocol consistency challenges:**

```
Challenge 1: Session lifetime mismatch
  Master PingFed session: 8 hours
  SAML assertion at Salesforce: Salesforce's own session cookie, 4-8 hours
  OIDC access token for React app: 15 minutes (short expiry design)
  OIDC refresh token: 8 hours

  If PingFed session expires at hour 8:
    React app: next token refresh fails → user prompted to re-authenticate ✓
    Salesforce: session cookie may still be valid → user still has access
    This inconsistency is expected and documented, not a bug

Challenge 2: Cross-protocol logout
  User logs out of Salesforce:
    Salesforce sends SAML SLO LogoutRequest to PingFed
    PingFed terminates the master session
    PingFed sends SAML SLO to Workday (also a SAML app)
    PingFed sends OIDC front-channel logout to React app
  
  OIDC front-channel logout:
    PingFed redirects browser to: https://reactapp.com/logout?sid=...&iss=...
    React app: receives the logout notification, clears tokens, marks session as ended
    Browser-based — works without the React app making any direct call to PingFed

Challenge 3: AuthnContext consistency
  SAML apps: see AuthnContext = MobileTwoFactorUnregistered (if MFA was used)
  OIDC apps: see acr claim in ID token (equivalent concept)
  PingFed maps: SAML AuthnContext ↔ OIDC acr claim
  High-security app (whether SAML or OIDC) can demand MFA via ForceAuthn (SAML)
  or max_age=0&acr_values=mfa (OIDC) — PingFed handles both from the same session
```

**Failure scenarios:**

```
Failure 1: OIDC token expires before SAML session
  OIDC access tokens: 15 min. User is mid-session in React app.
  React app attempts refresh → Auth Server returns new token (master session still valid)
  User experience: transparent. No re-auth.
  Risk: none, this is the designed behaviour.

Failure 2: PingFed session expires while user is active in Salesforce
  Salesforce session: still valid (its own session cookie, 4-8 hour timeout)
  React app OIDC refresh: fails (PingFed session expired)
  React app prompts re-authentication.
  Salesforce: continues working until its own session expires.
  This gap (Salesforce still active, React app not) is expected behaviour.
  Document it. For high-security environments, set consistent session timeouts.

Failure 3: SLO fails for one app (known vendor non-compliance)
  User logs out of Salesforce. PingFed sends SLO to Workday. Workday returns 200 OK
  but silently does not terminate the session (common Workday behaviour).
  User logs out. Returns to Workday. Still logged in.
  This is a vendor bug. Mitigation: short session timeouts on non-compliant apps.
  Document it in your security posture. File a vendor ticket.
```

---

## Category 6 — Code Review: Spot the Bug 💻

> *"Your team just shipped this. It is in production. It is also broken. Find the problems before the attacker does."*

**What this category tests:** Can you identify subtle authentication vulnerabilities in a code review? Principal Engineers are the last line of defence before broken auth ships to production. Each snippet has real bugs — some obvious, some subtle. Find them, explain the security impact of leaving each one unfixed, and write the corrected version.

**Format for every snippet:** the broken code → the bugs found → the corrected version.

---

### Bug 1 — JWT validation without algorithm check

```javascript
// Production JWT middleware protecting a banking API.
// This code is currently live on the /transfer endpoint.
// Find ALL security problems.

import jwt from 'jsonwebtoken';
import jwksRsa from 'jwks-rsa';

const jwksClient = jwksRsa({
  jwksUri: 'https://auth.bank.com/.well-known/jwks.json'
});

app.post('/transfer', async (req, res) => {
  const authHeader = req.headers.authorization;
  const token      = authHeader?.split(' ')[1];

  // Step 1: Decode header to get kid
  const decoded = jwt.decode(token, { complete: true });

  // Step 2: Fetch key by kid
  const key = await jwksClient.getSigningKey(decoded.header.kid);

  // Step 3: Verify
  jwt.verify(token, key.getPublicKey(), (err, payload) => {
    if (err) return res.status(401).json({ error: 'Invalid token' });
    initiateTransfer(payload.sub, req.body.amount, req.body.destination);
  });
});
```

**🐛 Bugs found (5 total):**

```
Bug 1 — CRITICAL: No algorithm restriction (alg:none + algorithm confusion attack)
  jwt.verify(token, key.getPublicKey()) with no options object.
  An attacker changes header to "alg": "none" → library skips signature verification.
  An attacker changes header to "alg": "HS256" and signs with the public key as HMAC secret.
  Both attacks allow forging tokens for any user with any scope.

Bug 2 — CRITICAL: jwt.decode() before jwt.verify() gives attacker control over kid
  The kid value is read from the token BEFORE it is verified.
  An attacker controls the kid value — possible kid injection attacks (SSRF, path traversal).
  The correct approach: use a library that handles kid resolution WITHIN verify().

Bug 3 — SERIOUS: No audience (aud) validation
  A token issued for any other service at the same Auth Server will be accepted here.
  Scope escalation: a "profile.read" token for a different API works on the transfer endpoint.
  Fix: add audience: 'https://api.bank.com/transfer' to verify options.

Bug 4 — SERIOUS: No issuer (iss) validation
  A token from a different Auth Server (dev environment, staging, compromised external server)
  will be accepted if it happens to have a valid signature.
  Fix: add issuer: 'https://auth.bank.com' to verify options.

Bug 5 — SERIOUS: No scope validation
  payload.scope is never checked.
  A read-only token (scope=account.read) can initiate a transfer.
  Fix: check payload.scope includes 'transfer.write' before calling initiateTransfer().
```

**✅ Corrected version:**

```javascript
import { createRemoteJWKSet, jwtVerify } from 'jose'; // use jose, not jsonwebtoken

// Setup ONCE at startup (not per-request)
const JWKS = createRemoteJWKSet(
  new URL('https://auth.bank.com/.well-known/jwks.json'),
  { cacheMaxAge: 600_000 } // cache public keys for 10 minutes
);

app.post('/transfer', async (req, res) => {
  const authHeader = req.headers.authorization;
  if (!authHeader?.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'missing_token' });
  }

  try {
    const { payload } = await jwtVerify(authHeader.slice(7), JWKS, {
      algorithms: ['RS256', 'ES256'],           // Bug 1 fixed: explicit allowlist
      issuer:     'https://auth.bank.com',      // Bug 4 fixed: issuer validation
      audience:   'https://api.bank.com/transfer' // Bug 3 fixed: audience validation
      // jose handles kid resolution internally (Bug 2 fixed)
      // exp is always validated automatically by jose
    });

    // Bug 5 fixed: scope check
    if (!payload.scope?.split(' ').includes('transfer.write')) {
      return res.status(403).json({
        error: 'insufficient_scope',
        required: 'transfer.write',
        got: payload.scope ?? 'none'
      });
    }

    // payload.sub is now safe to trust
    await initiateTransfer(payload.sub, req.body.amount, req.body.destination);
    res.status(200).json({ status: 'initiated' });

  } catch (err) {
    // Never leak internal error details to the client
    console.error('Token validation failed:', err.code, err.message);
    return res.status(401).json({ error: 'invalid_token' });
  }
});
```

---

### Bug 2 — OAuth callback without state validation

```javascript
// OIDC callback handler for a healthcare application.
// Handles the return from the Auth Server after user login.
// This code is in production. Find the security problems.

app.get('/auth/callback', async (req, res) => {
  const { code, error } = req.query;

  // Handle auth errors from the Auth Server
  if (error) {
    console.log('Auth error:', error, req.query.error_description);
    return res.redirect('/login?error=' + error);    // Line A
  }

  // Exchange code for tokens
  const tokens = await exchangeCodeForTokens(code, {
    redirect_uri: 'https://healthapp.example.com/auth/callback'
  });                                                 // Line B

  // Store user identity and tokens
  req.session.user        = parseIdToken(tokens.id_token);      // Line C
  req.session.accessToken = tokens.access_token;                // Line D
  req.session.refreshToken = tokens.refresh_token;

  res.redirect('/dashboard');
});
```

**🐛 Bugs found (4 total):**

```
Bug at Line A — CRITICAL: Open redirect + reflected error parameter
  res.redirect('/login?error=' + error)
  Attacker crafts: /callback?error=javascript:alert(document.cookie)
  Or: /callback?error=//evil.com/phishing
  The error value is reflected directly into the redirect URL without sanitisation.
  An attacker can inject JavaScript (in older browsers) or redirect to phishing pages.
  Additionally: the error value is logged (console.log) without sanitisation —
  log injection is possible.

Bug at Line B — CRITICAL: Missing state validation (Login CSRF)
  No check: req.query.state === req.session.oauthState
  An attacker can forge a callback URL with their own auth code:
  /callback?code=ATTACKER_CODE&state=anything
  State is never validated → code is exchanged → attacker's session bound to victim's browser.
  Victim now operates in the attacker's account.
  Healthcare context: victim enters their health data into attacker's account.

Bug at Line B (also): Missing nonce validation (ID token replay)
  parseIdToken() is called but nonce is never validated against the stored value.
  A stolen or replayed id_token from a different session could be used.
  Fix: store nonce before redirect, verify payload.nonce matches stored nonce on callback.

Bug at Line D — Medium: Access token in session store
  req.session.accessToken = tokens.access_token
  The access token is now in the session store (Redis/database).
  If the session store is compromised, all active access tokens are exposed.
  Better: hold access tokens in the BFF's in-memory process state,
          keyed by session ID — not in the persistent session store.
```

**✅ Corrected version:**

```javascript
app.get('/auth/callback', async (req, res) => {
  const { code, state, error } = req.query;

  // Safe error handling — never echo the error value
  if (error) {
    console.log('Auth error type:', error); // log the type only, not the value
    return res.redirect('/login?reason=auth_error'); // never include error value
  }

  // CSRF validation — must happen before any code exchange
  if (!state || state !== req.session.oauthState) {
    console.warn('State mismatch in callback — possible CSRF attack');
    return res.status(403).send('Invalid state parameter');
  }
  delete req.session.oauthState; // single-use — delete immediately after check

  // Exchange code for tokens (server-to-server, back-channel)
  const tokens = await exchangeCodeForTokens(code, {
    redirect_uri:   'https://healthapp.example.com/auth/callback',
    code_verifier:  req.session.pkceVerifier  // include PKCE verifier
  });
  delete req.session.pkceVerifier; // single-use

  // Validate id_token including nonce
  const idPayload = await validateIdToken(tokens.id_token, {
    expectedNonce: req.session.oauthNonce,
    expectedAudience: process.env.CLIENT_ID,
    expectedIssuer: process.env.AUTH_ISSUER
  });
  delete req.session.oauthNonce; // single-use

  // Store minimal user identity (not the full id_token)
  req.session.userId   = `${idPayload.iss}|${idPayload.sub}`; // stable composite key
  req.session.userName = idPayload.name;

  // Tokens: hold in process memory, keyed by session ID
  // NOT in the persistent session store
  tokenStore.set(req.session.id, {
    accessToken:  tokens.access_token,
    refreshToken: tokens.refresh_token,
    expiresAt:    Date.now() + tokens.expires_in * 1000
  });

  res.redirect('/dashboard');
});
```

---

### Bug 3 — SAML assertion processed by XML position

```java
// Production SAML assertion processor — Java.
// This is the core of your company's SAML SP implementation.
// It is called for every single SAML SSO login.
// Find the critical vulnerability.

public class SAMLAssertionProcessor {

  private final X509Certificate idpCertificate;

  public UserSession processResponse(String base64SamlResponse) throws Exception {

    // Decode and parse the XML
    byte[] xmlBytes = Base64.getDecoder().decode(base64SamlResponse);
    Document doc = parseXML(new String(xmlBytes));

    // Step 1: Verify the signature on the response
    boolean signatureValid = signatureVerifier.verify(doc, idpCertificate);
    if (!signatureValid) {
      throw new SecurityException("Invalid SAML signature");
    }

    // Step 2: Extract the assertion
    NodeList assertions = doc.getElementsByTagNameNS(
      "urn:oasis:names:tc:SAML:2.0:assertion",
      "Assertion"
    );
    Element assertion = (Element) assertions.item(0);  // ← THE VULNERABILITY

    // Step 3: Extract user information
    String nameId = assertion
      .getElementsByTagNameNS("urn:oasis:names:tc:SAML:2.0:assertion", "NameID")
      .item(0)
      .getTextContent();

    String role = assertion
      .getElementsByTagNameNS("urn:oasis:names:tc:SAML:2.0:assertion", "AttributeValue")
      .item(0)
      .getTextContent();

    return new UserSession(nameId, role);
  }
}
```

**🐛 Critical vulnerability:**

```
assertions.item(0) — finds the FIRST Assertion element by XML position.

This is the XSW vulnerability.
An attacker wraps a malicious assertion as the first child of the document.
The real signed assertion is nested inside it.
The signature covers the nested (real) assertion — valid ✓.
assertions.item(0) returns the outer (malicious) assertion.
The attacker is logged in as admin.

Additionally:
  Only the Response-level signature is checked (signatureVerifier.verify(doc, ...)).
  The inner Assertion's signature is not verified separately.
  An attacker can remove the Assertion's inner signature, modify its content,
  and resubmit. The Response signature still validates.
```

**✅ Corrected version:**

```java
public UserSession processResponse(String base64SamlResponse) throws Exception {

  byte[] xmlBytes = Base64.getDecoder().decode(base64SamlResponse);
  Document doc = parseXML(new String(xmlBytes));

  // Step 1: Schema validation BEFORE any signature check
  // Rejects structurally anomalous XML (nested assertions, unexpected elements)
  schemaValidator.validate(doc, SAML_20_SCHEMA);

  // Step 2: Find what was signed by looking at the Signature's Reference element
  NodeList signatures = doc.getElementsByTagNameNS(
    "http://www.w3.org/2000/09/xmldsig#", "Signature"
  );

  // Step 3: Extract the signed element's ID from the Reference URI
  Element signature = (Element) signatures.item(0);
  Element reference = (Element) signature
    .getElementsByTagNameNS("http://www.w3.org/2000/09/xmldsig#", "Reference")
    .item(0);
  String refUri  = reference.getAttribute("URI");
  String signedId = refUri.startsWith("#") ? refUri.substring(1) : refUri;

  // Step 4: Get the SPECIFIC element that was signed — not the first by position
  Element signedElement = doc.getElementById(signedId);
  if (signedElement == null) {
    throw new SecurityException(
      "Signed element not found — possible XSW attack. Rejecting."
    );
  }

  // Step 5: Verify the signature on the signed element specifically
  boolean signatureValid = signatureVerifier.verify(signedElement, idpCertificate);
  if (!signatureValid) {
    throw new SecurityException("Invalid SAML signature on assertion");
  }

  // Step 6: Verify the assertion IS signed (not just the response wrapper)
  boolean assertionSigned = hasSignature(signedElement);
  if (!assertionSigned) {
    throw new SecurityException("Assertion must be independently signed");
  }

  // Now safe to extract data from the VERIFIED signed element
  String nameId = signedElement
    .getElementsByTagNameNS(SAML_NS, "NameID").item(0).getTextContent();
  // ...

  return new UserSession(nameId, role);
}
```

---

### Bug 4 — Refresh token not rotated

```python
# Token refresh endpoint — Python Flask.
# This endpoint handles refresh token exchanges.
# Find the security gaps.

@app.route('/token', methods=['POST'])
def refresh_token():
    refresh_token_value = request.form.get('refresh_token')
    client_id           = request.form.get('client_id')

    # Look up the refresh token
    stored = db.query(
        "SELECT user_id, client_id, expires_at FROM refresh_tokens WHERE token = %s",
        (refresh_token_value,)
    ).fetchone()

    if not stored:
        return jsonify({'error': 'invalid_token'}), 401

    if stored['client_id'] != client_id:
        return jsonify({'error': 'invalid_client'}), 401

    if stored['expires_at'] < datetime.utcnow():
        return jsonify({'error': 'token_expired'}), 401

    # Generate new access token
    access_token = generate_access_token(stored['user_id'])

    # Return new access token
    return jsonify({
        'access_token':  access_token,
        'token_type':    'Bearer',
        'expires_in':    3600
    })
```

**🐛 Bugs found (3 total):**

```
Bug 1 — CRITICAL: Refresh token is never rotated
  The same refresh token can be used indefinitely until expires_at.
  If stolen: attacker has long-term access for the entire validity period.
  No detection mechanism — attacker and legitimate user can both use the same token.

Bug 2 — SERIOUS: No token family tracking
  Using an already-used refresh token returns 'invalid_token' but doesn't flag theft.
  The correct behaviour: if a rotated (already-used) token is submitted,
  revoke the ENTIRE token family (all tokens for this user/client combination).
  This forces both attacker and legitimate user to re-authenticate,
  signalling the security event.

Bug 3 — Medium: SQL query is safe (parameterised ✓) but token lookup is timing-vulnerable
  SELECT ... WHERE token = %s on an unhashed token value.
  If the token is stored in plaintext in the database:
    A database breach exposes all refresh tokens.
  Refresh tokens should be stored as: SHA256(token), not the raw value.
  Lookup: WHERE token_hash = SHA256(submitted_token)
```

**✅ Corrected version:**

```python
import hashlib
import secrets

@app.route('/token', methods=['POST'])
def refresh_token():
    refresh_token_value = request.form.get('refresh_token')
    client_id           = request.form.get('client_id')

    if not refresh_token_value or not client_id:
        return jsonify({'error': 'invalid_request'}), 400

    # Look up by hash (never store raw refresh tokens)
    token_hash = hashlib.sha256(refresh_token_value.encode()).hexdigest()

    stored = db.query(
        "SELECT user_id, client_id, expires_at, family_id, used "
        "FROM refresh_tokens WHERE token_hash = %s",
        (token_hash,)
    ).fetchone()

    if not stored:
        # Check if this is a ROTATED (already-used) token — theft signal
        used_token = db.query(
            "SELECT family_id FROM used_refresh_tokens WHERE token_hash = %s",
            (token_hash,)
        ).fetchone()
        if used_token:
            # THEFT DETECTED: revoke the entire token family
            db.execute(
                "DELETE FROM refresh_tokens WHERE family_id = %s",
                (used_token['family_id'],)
            )
            log_security_event('refresh_token_theft_detected', family_id=used_token['family_id'])
            return jsonify({'error': 'invalid_grant',
                           'error_description': 'Token reuse detected — re-authentication required'}), 401
        return jsonify({'error': 'invalid_token'}), 401

    if stored['client_id'] != client_id:
        return jsonify({'error': 'invalid_client'}), 401
    if stored['expires_at'] < datetime.utcnow():
        return jsonify({'error': 'token_expired'}), 401

    # ROTATION: generate a new refresh token and invalidate the old one
    new_refresh_token = secrets.token_urlsafe(64)
    new_token_hash    = hashlib.sha256(new_refresh_token.encode()).hexdigest()

    with db.transaction():
        # Mark old token as used (for theft detection)
        db.execute(
            "INSERT INTO used_refresh_tokens (token_hash, family_id, used_at) VALUES (%s, %s, %s)",
            (token_hash, stored['family_id'], datetime.utcnow())
        )
        # Delete old active token
        db.execute("DELETE FROM refresh_tokens WHERE token_hash = %s", (token_hash,))
        # Insert new token with same family_id
        db.execute(
            "INSERT INTO refresh_tokens (token_hash, user_id, client_id, family_id, expires_at) "
            "VALUES (%s, %s, %s, %s, %s)",
            (new_token_hash, stored['user_id'], client_id,
             stored['family_id'], datetime.utcnow() + timedelta(days=30))
        )

    access_token = generate_access_token(stored['user_id'])

    return jsonify({
        'access_token':  access_token,
        'refresh_token': new_refresh_token,  # NEW token — old one is dead
        'token_type':    'Bearer',
        'expires_in':    3600
    })
```

---

### Bug 5 — Insecure RelayState redirect

```python
# SAML ACS handler — the endpoint that receives SAMLResponses from the IdP.
# Called after every successful SAML authentication.
# Find the vulnerability.

@app.route('/saml/acs', methods=['POST'])
def acs():
    saml_response = request.form.get('SAMLResponse')
    relay_state   = request.form.get('RelayState', '/dashboard')

    # Process and validate the SAML assertion
    auth_result = saml_auth.process_response(saml_response)

    if not auth_result.is_authenticated():
        log_error('SAML authentication failed', auth_result.get_errors())
        return 'Authentication failed', 401

    # Create user session
    session['user_id']   = auth_result.get_nameid()
    session['user_name'] = auth_result.get_attributes().get('displayName', [''])[0]

    # Redirect to where the user was going
    return redirect(relay_state)  # ← OPEN REDIRECT
```

**🐛 Bug found:**

```
CRITICAL: Open redirect vulnerability in RelayState handling.

Attacker crafts: https://yourapp.com/saml/login?RelayState=https://evil.com/phishing

User sees: your company's legitimate PingFederate login page (100% real).
User authenticates: successfully with their real corporate credentials.
User is redirected: to evil.com — not to yourapp.com.
evil.com: "Your session expired during login. Please re-enter your password."
User: enters their password. Stolen.

The attack is uniquely effective because:
  1. The authentication was genuine — PingFed, not a fake page
  2. The phishing happens AFTER a successful login — user's guard is lowest
  3. The browser showed your company's domain the entire time
```

**✅ Corrected version:**

```python
from urllib.parse import urlparse

def safe_relay_redirect(relay_state: str, base_origin: str = 'https://yourapp.com') -> str:
    """
    Validate RelayState before redirecting.
    Allows: relative paths, same-origin absolute URLs.
    Rejects: external URLs, javascript: URLs, data: URLs.
    """
    if not relay_state:
        return '/dashboard'

    try:
        parsed = urlparse(relay_state)

        # Allow relative paths (no scheme or netloc means same-origin)
        if not parsed.scheme and not parsed.netloc:
            # Still sanitise the path
            if relay_state.startswith('//'):
                return '/dashboard'  # protocol-relative URL = external redirect
            return relay_state

        # Allow same-origin absolute URLs
        request_origin = f"{parsed.scheme}://{parsed.netloc}"
        if request_origin == base_origin:
            return relay_state

        # Everything else is potentially malicious
        return '/dashboard'

    except Exception:
        return '/dashboard'

@app.route('/saml/acs', methods=['POST'])
def acs():
    saml_response = request.form.get('SAMLResponse')
    relay_state   = request.form.get('RelayState', '')

    auth_result = saml_auth.process_response(saml_response)

    if not auth_result.is_authenticated():
        return 'Authentication failed', 401

    session['user_id']   = auth_result.get_nameid()
    session['user_name'] = auth_result.get_attributes().get('displayName', [''])[0]

    # SAFE redirect — validated before use
    safe_destination = safe_relay_redirect(relay_state)
    return redirect(safe_destination)
```

---

### Bug 6 — Missing audience check in Go

```go
// Go JWT middleware for a multi-tenant API gateway.
// This middleware validates Bearer tokens for all internal services.
// Find the security problem.

func AuthMiddleware(next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    authHeader := r.Header.Get("Authorization")
    if !strings.HasPrefix(authHeader, "Bearer ") {
      http.Error(w, "Unauthorized", http.StatusUnauthorized)
      return
    }

    tokenStr := authHeader[7:]

    // Parse and validate the JWT
    token, err := jwt.Parse(tokenStr, func(token *jwt.Token) (interface{}, error) {
      // Verify signing algorithm
      if _, ok := token.Method.(*jwt.SigningMethodRSA); !ok {
        return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
      }
      // Return the public key
      return getPublicKey(token.Header["kid"].(string)), nil
    })

    if err != nil || !token.Valid {
      http.Error(w, "Unauthorized", http.StatusUnauthorized)
      return
    }

    // Extract claims
    claims, ok := token.Claims.(jwt.MapClaims)
    if !ok {
      http.Error(w, "Unauthorized", http.StatusUnauthorized)
      return
    }

    // Add user ID to context
    ctx := context.WithValue(r.Context(), "user_id", claims["sub"])
    next.ServeHTTP(w, r.WithContext(ctx))
  })
}
```

**🐛 Bugs found (3 total):**

```
Bug 1 — CRITICAL: No audience (aud) validation
  claims["aud"] is never checked.
  A valid token issued for service-A works on this API gateway serving service-B.
  A token for the user portal works on the internal admin API.
  ANY valid JWT from this Auth Server is accepted by ALL services behind this gateway.
  This is the confused deputy vulnerability — any service can masquerade as any other.

Bug 2 — SERIOUS: No issuer (iss) validation
  A token from any Auth Server with a known RSA public key is accepted.
  Tokens from staging, dev, or a compromised third-party system work in production.

Bug 3 — Medium: getPublicKey from kid without validation
  kid is cast directly from the token header:
    token.Header["kid"].(string)
  If kid is not a string, this panics.
  If kid contains path traversal or injection characters: possible attacks.
  Should validate: kid matches a known format before fetching.
```

**✅ Corrected version:**

```go
const (
  expectedIssuer   = "https://auth.yourplatform.com"
  expectedAudience = "https://api.yourplatform.com"  // THIS service's identifier
)

func AuthMiddleware(next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    authHeader := r.Header.Get("Authorization")
    if !strings.HasPrefix(authHeader, "Bearer ") {
      http.Error(w, `{"error":"missing_token"}`, http.StatusUnauthorized)
      return
    }

    tokenStr := authHeader[7:]

    token, err := jwt.Parse(tokenStr, func(token *jwt.Token) (interface{}, error) {
      // Bug 3 fix: validate algorithm type explicitly
      if _, ok := token.Method.(*jwt.SigningMethodRSA); !ok {
        return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
      }
      // Bug 3 fix: safe kid extraction with type assertion and validation
      kid, ok := token.Header["kid"].(string)
      if !ok || kid == "" || len(kid) > 128 {
        return nil, fmt.Errorf("invalid or missing kid")
      }
      return getPublicKeyByKid(kid)  // fetches from pinned JWKS URL, not arbitrary URL
    })

    if err != nil || !token.Valid {
      http.Error(w, `{"error":"invalid_token"}`, http.StatusUnauthorized)
      return
    }

    claims, ok := token.Claims.(jwt.MapClaims)
    if !ok {
      http.Error(w, `{"error":"invalid_claims"}`, http.StatusUnauthorized)
      return
    }

    // Bug 2 fix: validate issuer
    if !claims.VerifyIssuer(expectedIssuer, true) {
      log.Printf("Token issuer mismatch: %v", claims["iss"])
      http.Error(w, `{"error":"invalid_issuer"}`, http.StatusUnauthorized)
      return
    }

    // Bug 1 fix: validate audience
    if !claims.VerifyAudience(expectedAudience, true) {
      log.Printf("Token audience mismatch: %v", claims["aud"])
      http.Error(w, `{"error":"invalid_audience"}`, http.StatusUnauthorized)
      return
    }

    // expiry is validated automatically by jwt.Parse — no need to check manually

    ctx := context.WithValue(r.Context(), "user_id", claims["sub"])
    next.ServeHTTP(w, r.WithContext(ctx))
  })
}
```

---

## 🌉 Bridge to Document 4

**What you just covered in Document 3:**

You can build IAM systems, not just use them. You understand how to design per-tenant key isolation where a breach of one customer cannot touch another. You know how to architect graceful degradation so an Auth Server outage affects 3% of users, not 100%. You can read broken authentication code and find the exact vulnerability — whether it is a missing algorithm check, an open redirect, or a SAML assertion being processed by XML position.

**The gap this leaves:**

You know protocols, attacks, incidents, and product design. But Principal Engineer interviews test one more dimension: can you design the API surface itself? Can you handle questions that sit at the very edge of the protocol's documented behaviour? And can you lead — communicate risk to non-technical stakeholders, mentor developers, and navigate disagreement between security requirements and product velocity?

**Document 4 covers:**
- **Category 7** — API & SDK design: 8 questions on designing token endpoints, JWKS endpoints, SCIM APIs, and OIDC discovery from the builder's perspective
- **Category 8** — Curveball questions: 10 questions that test the edge of your knowledge — at_hash, DPoP, PKCE on confidential clients, SameSite cookie attributes, OIDC Hybrid flow, and explaining OAuth to a non-technical executive in 60 seconds
- **Category 9** — Leadership & cross-functional: 10 questions on making decisions under ambiguity, communicating technical risk, mentoring, and resolving disagreement between security and product teams

> 💬 *"After Document 4, you are not just prepared for the interview. You are prepared for the job."*

---

*OAuth 2.0: RFC 6749 · PKCE: RFC 7636 · JWT: RFC 7519 · JWKS: RFC 7517 · Token Exchange: RFC 8693 · SAML 2.0: OASIS saml-core-2.0-os · SCIM 2.0: RFC 7642–7644 · OAuth 2.1: draft-ietf-oauth-v2-1*

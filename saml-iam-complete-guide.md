# SAML 2.0 + IAM Concepts — Complete Reference Guide

> **Scope:** SAML 2.0 protocol, SP-initiated and IdP-initiated SSO, Single Logout, building blocks (assertions, protocols, bindings, profiles, metadata), application onboarding end-to-end, security attacks and hardening, IAM fundamentals (identity lifecycle, AD/LDAP, SCIM, RBAC, ABAC, JIT, PAM), and SAML vs OIDC comparison. Apigee and PingFederate configuration are covered in the separate Apigee track.

---

## Table of Contents

1. [Module 1 — What SAML is and why it exists](#module-1)
2. [Module 2 — SAML building blocks](#module-2)
3. [Module 3 — SP-initiated SSO flow](#module-3)
4. [Module 4 — IdP-initiated SSO + Single Logout](#module-4)
5. [Module 5 — Application onboarding end-to-end](#module-5)
6. [Module 6 — SAML security attacks and hardening](#module-6)
7. [Module 7 — IAM fundamentals](#module-7)
8. [Module 8 — Authorisation models: RBAC, ABAC, JIT, PAM](#module-8)
9. [Module 9 — Production use cases + SAML vs OIDC decision guide](#module-9)
10. [Quick Reference — Cheat Sheet](#quick-reference)

---

## Module 1 — What SAML is and why it exists {#module-1}

### The enterprise problem

Before SAML, enterprises had a painful reality: 15 apps meant 15 passwords, 15 separate account databases, and when someone left the company, IT had to manually disable accounts in every system — and often missed some.

SAML (Security Assertion Markup Language) was built in 2002 to solve this: one verified identity, one login, access everywhere — including across organisational boundaries.

```
Before SAML:                          After SAML:
────────────                          ────────────
User has 15 different passwords       User has 1 corporate password
IT provisions 15 accounts manually    IT manages 1 identity in AD
Leaver: 15 manual account disables   Leaver: disable 1 AD account = all access revoked
Password resets = IT helpdesk chaos   Self-service MFA resets at IdP only
```

### The three actors

```
┌─────────────────┐                    ┌─────────────────┐
│    Principal    │ ─── authenticates ─▶│      IdP        │
│   (the user)    │                    │ (Identity       │
│                 │                    │  Provider)      │
│  Has identity   │                    │ PingFed / Okta  │
│  at the IdP     │                    │ Azure AD / ADFS │
└────────┬────────┘                    └────────┬────────┘
         │                                      │
         │ wants access                         │ issues signed
         │                                      │ SAML Assertion
         ▼                                      ▼
┌─────────────────────────────────────────────────────────┐
│                Service Provider (SP)                     │
│          Salesforce / Workday / ServiceNow / AWS         │
│  Trusts assertions from the IdP · Grants access based    │
│  on the assertion content · Never sees user's password   │
└─────────────────────────────────────────────────────────┘

         ◄──── Pre-established trust via metadata exchange ────►
```

| Actor | Role | Examples |
|---|---|---|
| Principal | The user who wants access. Has an account at the IdP (usually AD). Never interacts directly with SAML XML. | Employees, contractors, partners |
| Identity Provider (IdP) | Authenticates the user. Issues signed SAML Assertions. Source of truth for identity. | PingFederate, Okta, Azure AD, ADFS, Google Workspace |
| Service Provider (SP) | The application. Trusts the IdP. Validates the assertion. Creates a local session. | Salesforce, Workday, ServiceNow, AWS, any SaaS app |

### What SAML is and is not

- **SAML IS:** an authentication AND attribute federation protocol. It answers: "who is this user?" AND "what are their attributes (roles, department)?"
- **SAML IS NOT:** an authorisation protocol for APIs. It doesn't issue access tokens for API calls.
- **SAML IS NOT:** a user provisioning protocol. It handles login, not account lifecycle. SCIM handles provisioning.

### SAML vs OAuth/OIDC vs both

| Factor | SAML 2.0 | OIDC | OAuth 2.0 |
|---|---|---|---|
| Primary purpose | Enterprise SSO + attribute federation | Identity + modern SSO | API authorisation + delegation |
| Data format | XML (~5KB assertions) | JSON / JWT (compact) | JSON / JWT |
| Transport | Browser redirects only | Browser + native apps + APIs | Any HTTP client |
| Mobile support | Poor | Excellent | Excellent |
| Setup complexity | High (metadata, certs, XML) | Medium | Medium |
| Enterprise legacy | Dominant | Growing | Universal for APIs |
| Use when | Enterprise B2B SSO, legacy SaaS | Modern apps, consumer, new builds | API access, microservices |

**Key insight:** Modern enterprise IdPs (PingFederate, Okta, Azure AD) support all three protocols simultaneously from the same user store. Legacy apps use SAML. Modern apps use OIDC. APIs use OAuth.

---

## Module 2 — SAML building blocks {#module-2}

SAML has four layers: **Assertions** (what is said), **Protocols** (how it is asked/answered), **Bindings** (how messages travel), and **Profiles** (how layers combine for a specific use case). Metadata underpins all of them.

### Layer 1: Assertions

A SAML Assertion is a signed XML document the IdP issues. It is the core artefact of SAML — the signed permission slip that proves who the user is.

Three types of assertions:
- **Authentication Assertion** — proves the user authenticated (when, how)
- **Attribute Assertion** — carries the user's attributes (email, roles, department)
- **Authorisation Decision Assertion** — rarely used; says whether a specific action is permitted

#### Complete annotated SAML Assertion

```xml
<!-- The outer Response wrapper -->
<samlp:Response xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol"
  ID="_resp_8f3a2b1c4d5e6f7a"
  InResponseTo="_req_1a2b3c4d5e"    <!-- ties back to SP's AuthnRequest -->
  IssueInstant="2024-04-18T09:30:00Z"
  Destination="https://salesforce.com/saml/SSO">

  <saml:Issuer>https://pingfed.bank.com</saml:Issuer>
  <!--          ↑ IdP entity ID — SP validates this is a trusted IdP -->

  <samlp:Status>
    <samlp:StatusCode Value="urn:oasis:names:tc:SAML:2.0:status:Success"/>
  </samlp:Status>

  <saml:Assertion ID="_assert_9g4b3c2d5e6f" IssueInstant="2024-04-18T09:30:00Z">

    <!-- ① SUBJECT: who the assertion is about -->
    <saml:Subject>
      <saml:NameID Format="urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress">
        jane.smith@bank.com
      </saml:NameID>
      <saml:SubjectConfirmation Method="urn:oasis:names:tc:SAML:2.0:cm:bearer">
        <saml:SubjectConfirmationData
          NotOnOrAfter="2024-04-18T09:35:00Z"   <!-- 5-min validity window -->
          Recipient="https://salesforce.com/saml/SSO"
          InResponseTo="_req_1a2b3c4d5e"/>
      </saml:SubjectConfirmation>
    </saml:Subject>

    <!-- ② CONDITIONS: validity window + intended audience -->
    <saml:Conditions NotBefore="2024-04-18T09:29:55Z"
                     NotOnOrAfter="2024-04-18T09:35:00Z">
      <saml:AudienceRestriction>
        <saml:Audience>https://salesforce.com</saml:Audience>
        <!--            ↑ SP entity ID — assertion is ONLY valid for this SP -->
      </saml:AudienceRestriction>
    </saml:Conditions>

    <!-- ③ AUTHN STATEMENT: how the user authenticated -->
    <saml:AuthnStatement AuthnInstant="2024-04-18T09:29:58Z"
                         SessionIndex="_sess_7h5c4d3e">
      <saml:AuthnContext>
        <saml:AuthnContextClassRef>
          urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport
          <!-- MFA would be: urn:oasis:names:tc:SAML:2.0:ac:classes:MobileTwoFactorUnregistered -->
        </saml:AuthnContextClassRef>
      </saml:AuthnContext>
    </saml:AuthnStatement>

    <!-- ④ ATTRIBUTE STATEMENT: user attributes — this drives authorisation -->
    <saml:AttributeStatement>
      <saml:Attribute Name="email">
        <saml:AttributeValue>jane.smith@bank.com</saml:AttributeValue>
      </saml:Attribute>
      <saml:Attribute Name="Role">
        <saml:AttributeValue>Relationship_Manager</saml:AttributeValue>
        <saml:AttributeValue>Branch_London_EC2</saml:AttributeValue>
      </saml:Attribute>
      <saml:Attribute Name="Department">
        <saml:AttributeValue>Retail Banking</saml:AttributeValue>
      </saml:Attribute>
    </saml:AttributeStatement>

    <!-- ⑤ SIGNATURE: cryptographic integrity proof -->
    <ds:Signature xmlns:ds="http://www.w3.org/2000/09/xmldsig#">
      <!-- RSA-SHA256 signature over assertion content -->
      <!-- SP verifies with IdP's public certificate from metadata -->
    </ds:Signature>

  </saml:Assertion>
</samlp:Response>
```

**Critical security rule:** always validate the assertion signature specifically — not just the Response-level signature. The XSW attack exploits SPs that only check the outer Response signature.

#### NameID formats

| Format | Value | Recommended | Notes |
|---|---|---|---|
| `emailAddress` | jane@bank.com | No | Email can change — breaks SP accounts |
| `persistent` | Random opaque ID (stable) | **Yes** | Stable even if email changes |
| `transient` | Random, changes per session | Privacy use | SP cannot track users across sessions |
| `unspecified` | Anything (e.g. sAMAccountName) | Legacy only | Must be agreed with SP |

### Layer 2: Protocols

**AuthnRequest** — the SP sends this to initiate SP-initiated SSO. Contains: SP entity ID (Issuer), destination (IdP SSO URL), ACS URL (where to return the assertion), NameIDPolicy, RequestedAuthnContext.

**SAMLResponse** — the IdP returns this. Contains: InResponseTo (links to AuthnRequest ID), Status (Success/Failure), the Assertion, Signature.

**LogoutRequest / LogoutResponse** — used for Single Logout propagation.

**Key linkage:** the ID on the AuthnRequest must appear as `InResponseTo` on the SAMLResponse. The SP stores the AuthnRequest ID and validates the response references it — this prevents assertion injection and replay.

### Layer 3: Bindings

| Binding | Used for | How it works | Size limit |
|---|---|---|---|
| **HTTP-Redirect** | AuthnRequest (SP→IdP) | Deflated, Base64, in URL query string | ~2KB — small messages only |
| **HTTP-POST** | SAMLResponse (IdP→SP) | Base64 in hidden HTML form, auto-submits | No limit — used for assertions |
| **Artifact** | High-security back-channel | Reference passed via browser; SP fetches assertion server-to-server | No limit; assertion never in browser |

The standard Web Browser SSO Profile uses HTTP-Redirect for the AuthnRequest and HTTP-POST for the SAMLResponse.

### Layer 4: Profiles

| Profile | What it covers |
|---|---|
| **Web Browser SSO** | The core profile — SP-initiated and IdP-initiated SSO via browser |
| **Single Logout (SLO)** | Propagating logout across all SPs with active sessions |
| **Enhanced Client/Proxy (ECP)** | Non-browser SAML clients (rare — OIDC has replaced this) |
| **Artifact Resolution** | Back-channel assertion retrieval |

### Metadata

Before any SAML flow can work, IdP and SP exchange metadata XML. This is a one-time setup that establishes trust.

**What IdP metadata contains:**
- IdP Entity ID (the IdP's unique identifier)
- SSO service URL (where SP should redirect for authentication)
- SLO service URL (where logout requests go)
- **Signing certificate** (public key SP uses to verify all assertions)
- Supported NameID formats

**What SP metadata contains:**
- SP Entity ID (must match AudienceRestriction in every assertion)
- **ACS URL** (Assertion Consumer Service — where IdP POSTs the SAMLResponse)
- SLO endpoint
- SP signing certificate (IdP uses this to verify AuthnRequest signatures)

**Metadata exchange in practice:** SP admin exports metadata XML → sends to IdP admin (or gives them a URL). IdP admin imports it and creates an SP Connection. IdP admin exports IdP metadata → SP admin uploads to the app. Done once, renewed when certificates expire.

---

## Module 3 — SP-initiated SSO flow {#module-3}

### When it happens

User clicks "Sign In" on a SaaS app, sees "Sign in with SSO", enters their corporate email, and gets redirected to their company's login page. This is SP-initiated SSO — the flow starts at the Service Provider.

### Complete flow

```
Step 1: User visits https://salesforce.com/app
        (not logged in — no active session at SP)

Step 2: SP generates AuthnRequest
        SP stores AuthnRequest ID (_req_1a2b3c4d5e) in session for later validation
        SP sets RelayState = "https://salesforce.com/app" (where to go after login)

Step 3: SP redirects browser to IdP (HTTP-Redirect binding)
        302 → https://pingfed.bank.com/idp/SSO.saml2
              ?SAMLRequest=PHNhbWx...  (deflated, Base64-encoded AuthnRequest)
              &RelayState=aHR0cH...   (Base64-encoded original URL)
              &SigAlg=...
              &Signature=...

Step 4: Browser follows redirect to IdP
        IdP decodes and validates the AuthnRequest
        IdP shows login page (or uses existing SSO session — no login needed)

Step 5: User authenticates (username + password + MFA if required)

Step 6: IdP authenticates user against Active Directory
        IdP fetches user attributes from AD (groups, department, employee ID)
        IdP builds a SAML Assertion and signs it with its RSA private key

Step 7: IdP sends an HTML auto-submit form to the browser
        <form method="POST" action="https://salesforce.com/saml/SSO">
          <input type="hidden" name="SAMLResponse" value="PHNhbWxw..."/>
          <input type="hidden" name="RelayState" value="aHR0cHM6..."/>
        </form>
        <script>document.forms[0].submit();</script>

Step 8: Browser POSTs the form to SP's ACS URL (HTTP-POST binding)
        The assertion travels through the browser — but the browser cannot read it
        (it's Base64-encoded XML, and the browser just submits the form)

Step 9: SP receives SAMLResponse — validates:
        ✓ Decode Base64 → parse XML
        ✓ Verify assertion signature using IdP's certificate (from metadata)
        ✓ Check Issuer = "https://pingfed.bank.com" (expected IdP entity ID)
        ✓ Check AudienceRestriction = "https://salesforce.com" (my entity ID)
        ✓ Check NotBefore ≤ now ≤ NotOnOrAfter
        ✓ Check InResponseTo = "_req_1a2b3c4d5e" (stored AuthnRequest ID)
        ✓ Check Assertion ID not seen before (replay prevention)
        ✓ Extract NameID = "jane.smith@bank.com"
        ✓ Extract attributes: Role, Department, BranchCode

Step 10: SP creates local session
         SP maps SAML attributes to app roles/permissions
         SP redirects to RelayState URL → user lands on the page they wanted
         Total time: ~2-3 seconds including user login
```

### Real-world: bank employee accessing Salesforce

```xml
<!-- SAML AttributeStatement for a UK bank relationship manager -->
<saml:AttributeStatement>
  <saml:Attribute Name="email">
    <saml:AttributeValue>jane.smith@bank.com</saml:AttributeValue>
  </saml:Attribute>
  <saml:Attribute Name="SalesforceProfile">
    <saml:AttributeValue>Relationship Manager</saml:AttributeValue>
  </saml:Attribute>
  <saml:Attribute Name="BranchCode">
    <saml:AttributeValue>LON-EC2-042</saml:AttributeValue>
  </saml:Attribute>
  <saml:Attribute Name="Manager">
    <saml:AttributeValue>false</saml:AttributeValue>
  </saml:Attribute>
</saml:AttributeStatement>

<!-- Salesforce maps these to:
  SalesforceProfile="Relationship Manager" → assigns "Standard User" profile
  BranchCode                               → sets Territory = London EC2
  Manager="true"                           → grants manager permission set -->
```

---

## Module 4 — IdP-initiated SSO + Single Logout {#module-4}

### IdP-initiated SSO

User is already logged into the company's identity portal. They click an app tile. The IdP sends an **unsolicited** SAMLResponse directly to the SP — no AuthnRequest was sent first.

```
Step 1: User is already authenticated at IdP (logged into company portal)
Step 2: User clicks "Workday" app tile
Step 3: IdP builds and signs an assertion (no InResponseTo — unsolicited)
Step 4: IdP sends HTML auto-submit form to browser
Step 5: Browser POSTs to Workday's ACS URL
Step 6: SP validates: signature ✓, Issuer ✓, Audience ✓, time window ✓
        (Cannot validate InResponseTo — there is no AuthnRequest)
Step 7: SP creates session → user is in
```

**Security difference from SP-initiated:** No AuthnRequest means no InResponseTo to validate. The SP cannot verify this assertion was requested. This makes IdP-initiated flows more vulnerable to:
- Assertion injection (attacker sends crafted assertion to SP's ACS URL)
- CSRF (attacker tricks user's browser into completing IdP-initiated flow with attacker's identity)

**Mitigations:**
- Only accept IdP-initiated flows from known IdP IP ranges
- Validate all other assertion elements (Issuer, Audience, time window, replay)
- Use a CSRF token embedded in RelayState

### RelayState

RelayState is an opaque string passed through the entire SSO flow unchanged. Its purpose is to carry context — most importantly, where to send the user after login.

```
SP-initiated: SP sets RelayState = "https://salesforce.com/account/00123456"
              After SSO: user lands directly on that specific record ← deep linking

IdP-initiated: IdP sets RelayState = target SP URL
               Tells browser which SP's ACS URL to POST to

Security risk: never redirect to an arbitrary RelayState URL
               Whitelist: only redirect to your own domain
               Prevents: open redirect attacks (post-SSO redirect to phishing site)
```

### Single Logout (SLO)

When a user logs out of one SP, SLO propagates the logout to every SP they have an active session with.

```
Step 1: User clicks "Logout" in Salesforce
Step 2: Salesforce sends LogoutRequest to IdP (HTTP-Redirect)
Step 3: IdP terminates its session
Step 4: IdP sends LogoutRequest to every other SP with an active session
        → Workday: LogoutRequest → LogoutResponse (success)
        → ServiceNow: LogoutRequest → LogoutResponse (success)
        → JIRA: LogoutRequest → LogoutResponse (success)
Step 5: IdP sends LogoutResponse back to Salesforce
Step 6: Salesforce terminates its local session
        User is fully signed out everywhere
```

**Reality of SLO in production:** many SaaS apps implement SLO poorly or not at all. The IdP should handle graceful degradation — if an SP doesn't respond to LogoutRequest within timeout, continue with the rest and log the failure. Users should be informed that some apps may still have active sessions.

---

## Module 5 — Application onboarding end-to-end {#module-5}

### The onboarding lifecycle

```
Day 0 — Procurement
  Business approves new SaaS app
  Security team confirms SAML 2.0 support
  Vendor confirms supported bindings, NameID formats, and attribute names
  Vendor provides: SP metadata XML (or Entity ID + ACS URL + certificate)

Day 1 — IdP Configuration (PingFederate)
  Create SP Connection → enter/import SP metadata
  Configure NameID format (persistent or emailAddress — agree with SP)
  Define Attribute Contract:
    What attributes to include in the assertion
    Map AD/LDAP attributes to assertion attribute names
  Configure allowed AD groups (which users can access this app)
  Set authentication policy (password-only vs MFA required)
  Export IdP metadata → send to SP admin

Day 1 — SP Configuration
  Upload IdP metadata XML (or enter SSO URL, Entity ID, certificate manually)
  Set NameID format to match IdP
  Map incoming SAML attributes to app fields (role mapping)
  Enable JIT provisioning if applicable
  Enable SAML SSO mode

Day 2 — Testing
  3–5 pilot users from different AD groups
  Use SAML-tracer browser extension to inspect assertions
  Fix the 2–3 configuration errors that always appear (see table below)

Day 3 — Go-live
  Enable app tile in IdP portal for appropriate AD groups
  Communicate to users
  Enable SCIM provisioning if the app supports it
  Document in IAM register
```

### The three most common onboarding failures

| Error | Cause | Fix |
|---|---|---|
| "Signature validation failed" | Wrong IdP certificate uploaded to SP | Re-export certificate from IdP metadata, re-upload to SP |
| User logged in but wrong role | Attribute name mismatch (SP expects "Role", IdP sends "groups") | Align attribute names in IdP Attribute Contract or SP mapping config |
| "AudienceRestriction mismatch" | SP Entity ID in IdP doesn't match what SP reports as its own ID | Verify SP Entity ID is exactly the same string in both IdP SP Connection and SP's own SSO settings |

### Attribute contract — the most important configuration

The Attribute Contract defines what user data the IdP sends in every assertion. Mismatch here is the #1 cause of SSO failures in production.

```
PingFederate Attribute Contract example:
AD attribute              →  Assertion attribute name  → SP field
─────────────────────────────────────────────────────────────────
mail                      →  email                    → User email
givenName                 →  firstName                → First name
sn                        →  lastName                 → Last name
department                →  department               → Department
memberOf (filter: App_*)  →  groups                   → App roles (multi-value)
employeeID                →  employeeNumber           → Employee ID
manager                   →  isManager                → Manager flag
```

**Debugging attribute mapping:** install the SAML-tracer browser extension (Firefox/Chrome). It intercepts SAML flows and shows you the actual decoded assertion — you can see exactly what attributes are being sent and whether their names match what the SP expects.

### Typical onboarding timelines

| App type | Timeline | Why |
|---|---|---|
| Simple SaaS (Slack, Zoom, Atlassian) | 1–2 days | Standard SAML, clear documentation, pre-built templates in IdP |
| Complex enterprise SaaS (Salesforce, ServiceNow) | 1–2 weeks | Custom attribute mapping, multiple permission profiles, role hierarchy |
| Custom-built internal app | 2–4 weeks | Development required for SAML library integration |
| Legacy enterprise (SAP, Oracle) | 2–4 weeks | Complex configuration, vendor support required |

---

## Module 6 — SAML security attacks and hardening {#module-6}

### Attack 1: XML Signature Wrapping (XSW) — the most critical vulnerability

**How it works:**
1. Attacker logs in legitimately and captures a valid signed SAMLResponse
2. Attacker copies the signed assertion and creates a malicious assertion (with different NameID — e.g. "admin@bank.com")
3. Attacker wraps the malicious assertion alongside the legitimate one in the XML
4. The XML digital signature still validates — it references the legitimate assertion by ID
5. But naive implementations process the malicious assertion (by XML position, not by ID reference)
6. Attacker is now authenticated as the admin account

**Why it works:** The XML Digital Signature spec references elements by their ID attribute. The signature is valid — it just points to a different assertion than the one the SP ends up processing.

**Real-world impact:** XSW vulnerabilities have been found in OneLogin, Duo Security, and multiple enterprise SSO implementations.

**Prevention:**
```
1. Always retrieve the assertion to process using the ID referenced in the 
   signature's <Reference> element — not by XML position or XPath
2. Use a well-tested SAML library (python3-saml, java-saml, passport-saml)
   Never implement SAML XML parsing yourself
3. Validate XML schema before processing
4. Run SAML security scanners against your implementation
```

### Attack 2: Assertion Replay

**How it works:** intercept a valid SAML assertion (XSS, network sniffing, log exposure). Submit it to the SP's ACS URL before NotOnOrAfter expires (typically 5-minute window). The SP validates signature and time — both pass. Attacker is logged in as the victim.

**Prevention:**
```python
# SP maintains a cache of used assertion IDs
# Redis with TTL = NotOnOrAfter timestamp

async def validate_assertion(assertion):
    assertion_id = assertion.get_id()
    not_on_or_after = assertion.get_not_on_or_after()
    
    # Check if this ID was already used
    if await redis.exists(f"saml:used:{assertion_id}"):
        raise SecurityError("Assertion replay detected")
    
    # Mark this ID as used (expires automatically at NotOnOrAfter)
    ttl = not_on_or_after - datetime.utcnow()
    await redis.setex(f"saml:used:{assertion_id}", int(ttl.seconds), "1")
    
    # Continue with normal validation...
```

### Attack 3: Open Redirect via RelayState

**How it works:** attacker crafts a link with RelayState pointing to a phishing site. User sees the legitimate company login page, authenticates successfully, and is redirected to `https://attacker.com`.

**Prevention:**
```python
# WRONG: blind redirect to RelayState
return redirect(relay_state)

# CORRECT: only allow same-origin RelayState values
from urllib.parse import urlparse

def safe_redirect(relay_state, allowed_host="yourapp.com"):
    if relay_state:
        parsed = urlparse(relay_state)
        if parsed.netloc and parsed.netloc != allowed_host:
            return redirect("/dashboard")  # fallback
        return redirect(relay_state)
    return redirect("/dashboard")
```

### Production hardening checklist

**SP must validate on every assertion:**
```
✅ Assertion signature using IdP's certificate (from metadata — not user-supplied)
✅ Issuer element matches the expected IdP entity ID exactly (string match)
✅ AudienceRestriction contains this SP's entity ID
✅ NotBefore ≤ now ≤ NotOnOrAfter (max 60s clock skew)
✅ InResponseTo matches a stored AuthnRequest ID (SP-initiated flows)
✅ Assertion ID not in the replay prevention cache
✅ SubjectConfirmation Recipient = this ACS URL
✅ Status code = Success before processing any assertion
```

**Implementation rules:**
```
✅ TLS 1.2+ on all SAML endpoints (ACS, SLO, metadata)
✅ Use a security-reviewed SAML library — never roll your own XML parsing
✅ Schema-validate XML before processing
✅ Whitelist RelayState redirects to own domain only
✅ Process assertion by signature Reference ID, not position
✅ Certificate rotation plan: update metadata before cert expires (overlap period)
✅ Log all assertion IDs and validation failures for SIEM alerting
✅ Sign AuthnRequests (WantAuthnRequestsSigned=true in IdP metadata)
```

---

## Module 7 — IAM fundamentals {#module-7}

### The four pillars of IAM

```
┌────────────┐   ┌────────────────┐   ┌──────────────────┐   ┌──────────┐
│  Identity  │ → │ Authentication │ → │  Authorisation   │ → │  Audit   │
│            │   │                │   │                  │   │          │
│ Who are    │   │ Prove it       │   │ What can you do? │   │ What did │
│ you?       │   │                │   │                  │   │ you do?  │
│            │   │                │   │                  │   │          │
│ AD / LDAP  │   │ PingFed / Okta │   │ RBAC / ABAC      │   │ SIEM /   │
│ SCIM / HR  │   │ ADFS / Azure   │   │ PAM / IGA        │   │ IGA logs │
└────────────┘   └────────────────┘   └──────────────────┘   └──────────┘

SAML operates at the Authentication ↔ Authorisation boundary:
proves identity AND carries attribute claims that drive authorisation
```

### Active Directory and LDAP

**Active Directory** is the dominant on-premises enterprise identity store. It stores user accounts, groups, and policies for an entire organisation.

Key concepts:
- **Domain** — a security boundary (e.g. `bank.com`)
- **Forest** — a collection of domains with shared trust
- **Organisational Unit (OU)** — a container for organising objects (e.g. OU=Finance, OU=IT)
- **Group** — a collection of users. Group membership is what PingFederate reads to determine what SAML attributes to issue
- **sAMAccountName** — the Windows login name (e.g. `jsmith`)
- **userPrincipalName** — the UPN, typically the email format (e.g. `jane.smith@bank.com`)

**How PingFederate connects to AD:**
1. PingFed has an LDAP Datastore configured pointing to AD domain controllers
2. When a user authenticates, PingFed binds to AD with a service account
3. PingFed searches for the user's LDAP entry
4. PingFed reads the attributes configured in the Attribute Contract (mail, givenName, memberOf etc.)
5. PingFed maps these AD attributes to the SAML assertion attribute names the SP expects

### Identity lifecycle: Joiner / Mover / Leaver

**Joiner (new employee):**
```
Day 0 (HR): New employee created in HR system (Workday/SAP HR)
Day 0 (IGA): Identity Governance triggers provisioning workflow
Day 0 (AD): AD account created → added to role-based security groups
Day 0 (SCIM): SCIM pushes account creation to 30+ connected SaaS apps
Day 0 (Portal): App tiles appear in IdP portal
Day 1 (First login): Employee logs in → SAML SSO works across all apps

Target SLA: all access granted within 4 hours of start date
```

**Mover (role change):**
```
Manager submits access change request → IGA workflow approval
Old AD group memberships removed → new groups added
SCIM updates app roles automatically (active sync)
Next SAML login → assertion carries new roles → SP applies new permissions

Challenge: SP sessions created before the role change still have the old
roles until the session expires or the user re-authenticates
Solution: short SP session timeouts (4-8 hours) or IdP session force-logout
```

**Leaver (employee exits):**
```
HR triggers termination → IGA workflow
T+0: AD account DISABLED immediately (automated, no human needed)
T+0: All SAML SSO attempts fail immediately
      (IdP authenticates against AD — disabled accounts fail authentication)
T+5min: SCIM deprovisions accounts in all connected apps
T+60min: Audit confirms all access removed

Note: Active SP sessions may persist until their local timeout expires
      (typically 4-8 hours for web apps). Critical apps should have shorter
      session timeouts or implement session invalidation via SLO.

SLA target: AD disabled within 1 hour of termination notice
            (many organisations target 15 minutes)
```

### SCIM — System for Cross-domain Identity Management

SAML handles login. SCIM handles account existence. Together they form complete identity lifecycle management.

**SCIM 2.0 endpoints:**

```bash
# Create a user account
POST https://app.example.com/scim/v2/Users
Authorization: Bearer <SCIM provisioning token>
{
  "schemas": ["urn:ietf:params:scim:schemas:core:2.0:User"],
  "userName": "jane.smith@bank.com",
  "name": { "givenName": "Jane", "familyName": "Smith" },
  "emails": [{ "value": "jane.smith@bank.com", "primary": true }],
  "active": true,
  "roles": [{ "value": "Relationship Manager", "primary": true }]
}

# Disable a user (leaver)
PATCH https://app.example.com/scim/v2/Users/{id}
{ "Operations": [{ "op": "replace", "path": "active", "value": false }] }

# Update role (mover)
PATCH https://app.example.com/scim/v2/Users/{id}
{ "Operations": [{ "op": "replace", "path": "roles",
  "value": [{"value": "Senior Manager", "primary": true}] }] }

# Get all groups
GET https://app.example.com/scim/v2/Groups
```

**SCIM vs JIT provisioning:**

| | SCIM | JIT (Just-In-Time) |
|---|---|---|
| When accounts are created | Before first login (proactive) | On first login (reactive) |
| When accounts are disabled | Immediately on deprovisioning event | Only when AD login fails |
| Best for | Apps needing pre-loaded data, strict deprovisioning SLA | Simple apps, small scale |
| Required for | Zero-day access (account exists on day one) | Apps where on-demand creation is fine |

---

## Module 8 — Authorisation models: RBAC, ABAC, JIT, PAM {#module-8}

### RBAC — Role-Based Access Control

Users are assigned roles. Roles have permissions. SAML carries roles in the AttributeStatement.

```xml
<!-- SAML assertion carries roles -->
<saml:Attribute Name="Role">
  <saml:AttributeValue>Manager</saml:AttributeValue>
</saml:Attribute>

<!-- SP maps:
  Manager  → read + write + approve team records
  Employee → read + write own records only
  Auditor  → read only, all records
  Admin    → full access                     -->
```

**Limitations of RBAC at enterprise scale:**
- Role explosion: 500 employees × 20 apps = thousands of roles to manage
- Coarse-grained: "Manager" role gives all-or-nothing access; can't restrict by data sensitivity
- Static: same access regardless of time, location, or device

### ABAC — Attribute-Based Access Control

Access decisions based on attributes of the **user**, **resource**, **action**, and **environment**.

```
Policy rule (pseudo-code):
ALLOW access to patient record IF
  user.department = "Oncology"
  AND user.role = "Physician"
  AND resource.patient_treating_team CONTAINS user.employeeId
  AND action IN ["read", "write"]
  AND time BETWEEN "07:00" AND "21:00"
  AND user.device_compliance = "compliant"

All user attributes come from the SAML assertion:
department, role, employeeId, device_compliance
Resource attributes come from the application's data store
```

**When ABAC is essential:**
- Healthcare: patient record access restricted to treating physicians
- Finance: trade approval requires specific clearance level + dual control
- Legal: matter access restricted to assigned attorneys
- Government: data classified by clearance level + need-to-know

### JIT provisioning — account creation on first login

When a user first SSOs into an app, the SP creates their account automatically using the SAML assertion attributes.

```
First login to ServiceNow via SSO:
1. SAML assertion arrives at ServiceNow ACS URL
2. ServiceNow looks up: does an account exist for NameID "jane.smith@bank.com"?
3. No → create account:
   - Username = NameID value
   - First name = firstName attribute
   - Last name = lastName attribute
   - Department = department attribute
   - Role = map "groups" attribute values to ServiceNow roles
4. Log user in to newly created account

Subsequent logins:
- Update account attributes from assertion (keeps attributes in sync with AD)
- Attribute updates on login = no separate SCIM needed for attribute changes
```

### PAM and MFA in the SAML context

**Privileged Access Management (PAM)** protects high-privilege accounts. In the SAML context, the **AuthnContextClassRef** in the assertion communicates to the SP how strongly the user authenticated.

```xml
<!-- Standard password authentication -->
<saml:AuthnContextClassRef>
  urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport
</saml:AuthnContextClassRef>

<!-- MFA (TOTP, push, hardware token) -->
<saml:AuthnContextClassRef>
  urn:oasis:names:tc:SAML:2.0:ac:classes:MobileTwoFactorUnregistered
</saml:AuthnContextClassRef>

<!-- Smartcard / PKI certificate -->
<saml:AuthnContextClassRef>
  urn:oasis:names:tc:SAML:2.0:ac:classes:Smartcard
</saml:AuthnContextClassRef>
```

**Step-up authentication:** the SP can refuse access to sensitive operations if the AuthnContext is insufficient, and redirect the user to re-authenticate with a higher assurance level.

```
Example: Banking wire transfer
1. User logs in with password → AuthnContext = PasswordProtectedTransport
2. User navigates to wire transfer page (high-risk action)
3. App checks: AuthnContext indicates MFA not performed
4. App redirects back to IdP with RequestedAuthnContext = MFA class
5. IdP prompts for MFA → user completes MFA
6. New assertion issued with MFA AuthnContext
7. App allows wire transfer to proceed
```

---

## Module 9 — Production use cases + SAML vs OIDC decision guide {#module-9}

### Use case A — Global bank: 12,000 employees, 80 SAML apps

```
Architecture:
  Identity source:  Active Directory (2 forests — retail + investment banking)
  IdP:              PingFederate 11.x cluster (2 nodes, active-active, load balanced)
  SP connections:   80 SAML 2.0 integrations
  Provisioning:     SCIM to 35 apps, JIT for remaining 45
  MFA:              Enforced via AuthnContextClassRef for privileged apps

Onboarding SLA (new employee, Day 1):
  08:00  HR triggers joiner workflow in IGA tool
  08:15  AD account created, added to role-based security groups
  08:30  SCIM pushes to 35 core apps (email, calendar, Salesforce, ServiceNow)
  09:00  Employee arrives → portal shows all app tiles
         → SAML SSO works for all 80 apps immediately

Offboarding SLA (termination):
  T+0    Manager submits termination → automated workflow triggered
  T+0    AD account disabled (immediate, automated — no human action)
  T+0    All SAML SSO attempts fail (AD auth fails for disabled accounts)
  T+5    SCIM deprovisions all 35 SCIM-connected apps
  T+60   Manual confirmation all access removed
  
Notes on session persistence:
  Active SP sessions may persist 4–8 hours (typical web app session timeout)
  Critical apps (trading, wire transfers): 30-min sessions + SLO enforced
  Active sessions are logged and monitored by SIEM during transition period
```

### Use case B — SaaS platform with multi-tenant SAML

```
Scenario: B2B HR SaaS platform serving 50 enterprise customers
Each customer has their own corporate IdP (PingFed, Okta, Azure AD, ADFS)

Architecture:
- Each customer has a separate SP Connection on the platform
- Each customer has their own tenant-specific Entity ID and ACS URL
- Login routing: email domain → maps to correct IdP

Flow:
1. User visits https://hrplatform.com/login
2. User enters email: jane@bigbank.com
3. Platform looks up "bigbank.com" → maps to IdP config for BigBank
4. Redirect to https://pingfed.bigbank.com/idp/SSO.saml2
   with SP entity ID = "https://hrplatform.com/saml/bigbank"
5. PingFed authenticates the user
6. Returns assertion to https://hrplatform.com/saml/acs/bigbank
   (tenant-specific ACS URL ensures strict isolation)

Tenant isolation:
  AudienceRestriction = "https://hrplatform.com/saml/bigbank"
  An assertion for BigBank submitted to CapCorp's ACS URL will be rejected
  (AudienceRestriction won't match "https://hrplatform.com/saml/capcorp")

Customer onboarding time: 30–60 minutes
  Bottleneck: agreeing the attribute contract with customer IT teams
  (every customer names their role attribute differently)
  Platform now provides a "SAML attribute mapping wizard" that reduced this
  time from 2 days to 1 hour.
```

### Use case C — Migrating from SAML to OIDC

```
Scenario: company with 60 SAML apps wants to modernise
Strategy: run SAML and OIDC in parallel from the same PingFed instance

Phase 1 (immediate): New apps → OIDC only (no new SAML integrations)
Phase 2 (6 months):  Mobile apps → OIDC + PKCE (SAML doesn't work natively)
Phase 3 (12 months): Modern SaaS that supports both → migrate to OIDC
Phase 4 (ongoing):   Legacy enterprise (SAP, Oracle, mainframe tools) → SAML indefinitely

PingFed bridge: same user session can issue both SAML assertions AND OIDC tokens
  User logs in once → can access SAML apps AND OIDC apps without re-authentication

Why some apps will always stay on SAML:
  - SAP, Oracle EBS, mainframe-era tools: only support SAML, no OIDC roadmap
  - Cross-org B2B federation: deeply embedded corporate trust frameworks
  - Regulatory: some compliance frameworks (FedRAMP, certain banking regs)
    have audited SAML implementations that cannot be changed without re-audit
```

### SAML vs OIDC — complete decision guide

| Factor | Choose SAML | Choose OIDC |
|---|---|---|
| App type | Legacy enterprise SaaS (Salesforce, Workday, SAP) | Modern web apps, mobile apps, new builds |
| Client type | Web browser only | Browser, native mobile, desktop, IoT, CLI |
| B2B Federation | Cross-organisation corporate IdP federation | Consumer SSO, modern B2B |
| Token format | App vendor requires XML | App prefers JSON/JWT |
| API access | Not needed | Required (need OAuth 2.0 access tokens) |
| Setup effort | Higher (metadata, XML, certs, ACS URLs) | Lower (discovery URL, client registration) |
| Mobile support | Poor | Excellent |
| Your choice | You have none — app only supports SAML | You have a choice — always prefer OIDC |

**Architect's decision rule:** if an app supports both SAML and OIDC, always choose OIDC for new integrations. SAML is for apps that leave you no choice — legacy enterprise tools, cross-org corporate federation, or systems built before 2015. For everything else: use OIDC.

---

## Quick Reference — Cheat Sheet {#quick-reference}

### SAML flow selection

```
User starts at...      
├── The SP (app)        → SP-initiated SSO (most common, more secure)
└── The IdP (portal)   → IdP-initiated SSO (convenient, less secure)

Logging out...
├── One app at a time  → Local logout only (most common in practice)
└── Everything at once → Single Logout (SLO) — if all SPs support it
```

### SP validation checklist (every assertion)

```
✅ Signature on the Assertion (not just the Response) using IdP cert from metadata
✅ Issuer = expected IdP entity ID
✅ AudienceRestriction contains this SP's entity ID
✅ NotBefore ≤ now ≤ NotOnOrAfter (max 60s clock skew tolerance)
✅ InResponseTo matches a stored, unused AuthnRequest ID (SP-initiated)
✅ Assertion ID not previously seen (replay prevention cache with TTL)
✅ SubjectConfirmation Recipient = this ACS URL exactly
✅ Status code = Success (before processing any assertion content)
```

### IAM joiner/mover/leaver quick reference

| Event | AD action | SAML impact | SCIM action | SLA |
|---|---|---|---|---|
| Joiner | Create account + add to groups | SSO works on day one | Provision accounts in all apps | 4 hours |
| Mover | Update group membership | New roles on next login | Update roles in all apps | Same day |
| Leaver | Disable account | All SSO fails immediately | Deprovision all app accounts | 1 hour |

### SAML security non-negotiables

```
1.  Always validate assertion signature (not just Response signature)
2.  Always validate Issuer against known IdP entity ID
3.  Always validate AudienceRestriction against your own entity ID
4.  Always enforce NotBefore/NotOnOrAfter with strict clock skew
5.  Always track and reject replayed assertion IDs (Redis cache)
6.  Always validate InResponseTo in SP-initiated flows
7.  Never trust RelayState as a redirect target without domain check
8.  Never process assertion by XML position — use signature Reference ID
9.  Never implement XML parsing yourself — use a vetted SAML library
10. TLS on all endpoints, always, no exceptions
```

### Protocol landscape — which to use when

```
Need enterprise B2B SSO with corporate IdPs?         → SAML 2.0
Building a modern web or mobile app?                 → OIDC
Need API access tokens for microservices?            → OAuth 2.0
SSO for your app that your customers' companies use? → SAML (they'll have PingFed/Okta/ADFS)
Building a new internal app for employees?           → OIDC
Legacy SAP / Oracle / mainframe integration?         → SAML (no choice)
```

---

## What comes next

This document covers the SAML 2.0 protocol and IAM concepts. The next document in this series:

**Apigee + PingFederate** — how to implement OAuth/OIDC token validation and SAML assertion handling at the API gateway layer, using Apigee policies and PingFederate as the combined Auth Server and IdP.

---

*SAML 2.0 Core: OASIS saml-core-2.0-os | Web Browser SSO Profile: saml-profiles-2.0-os | SCIM 2.0: RFC 7642–7644 | RBAC: NIST SP 800-162 | ABAC: NIST SP 800-162*

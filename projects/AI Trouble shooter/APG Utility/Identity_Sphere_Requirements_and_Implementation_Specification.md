# Identity Sphere
## Requirements & Implementation Specification

**Status:** Locked Design Baseline  
**Purpose:** Implementation contract for GitHub Copilot  
**Scope:** Requirements 1–3

---

## 1. Document Purpose & Implementation Instructions

This document defines the locked requirements for **Identity Sphere** and is intended to be used as the primary implementation reference for GitHub Copilot.

Identity Sphere requirements shall be implemented **sequentially in the same codebase and implementation context**:

1. Requirement 1 — Configuration-Driven Custom API Framework
2. Requirement 2 — Token Visibility & Intelligent Token Reuse
3. Requirement 3 — API CURL Command

Each requirement is independently defined, but later requirements must reuse and build on the components and behavior established by earlier requirements.

### Implementation rules

- Treat explicitly locked requirements as authoritative.
- Do not redesign the architecture unless the user explicitly requests a design change.
- Do not introduce unnecessary abstraction, frameworks, services, databases, or layers.
- Dynamic implementation-specific values will be supplied separately by the user during implementation.
- Do not invent production URLs, API paths, Client IDs, Client Secrets, secret names, credentials, or vendor-specific behavior.
- Where a dynamic value has not yet been supplied, use a clearly identified placeholder or ask for the value when it is genuinely required.
- Preserve the relationships and dependencies between Requirements 1, 2, and 3.
- Reuse previously implemented functionality rather than creating duplicate implementations.

---

## 2. Identity Sphere Context

Identity Sphere is a **search and troubleshooting platform** for identity teams working with systems such as IGA, PAM, SSO, Active Directory, LDAP directories, and custom identity APIs.

Identity Sphere is **search-only** for the scope of these requirements.

There are no create, update, or delete operations.

### Core backend utility structure

```text
Identity Sphere
      |
      +-- LDAP Utility
      |      +-- Active Directory
      |      +-- Ping Directory
      |      +-- Other LDAP systems
      |
      +-- Custom API Utility
      |      +-- APIs accessed through APIGEE Gateway
      |      +-- AWS-hosted Custom APIs
      |
      +-- OOTB Utility
             +-- CyberArk
             +-- Saviynt
             +-- Ping
             +-- Other vendor APIs
```

### Utility responsibility

Utilities contain common connection and API invocation logic.

Calling modules provide the dynamic search/request information.

Utilities return the **raw backend response**, normally JSON.

Parsing, massaging, transformation, and presentation of the response remain the responsibility of the calling module.

Do not introduce complex response normalization unless a future requirement explicitly requires it.

---

## 3. Architecture Principles — Locked

The architecture intentionally remains simple.

### Allowed core structure

```text
Identity Sphere
      |
      +-- LDAP Utility
      +-- Custom API Utility
      +-- OOTB Utility
```

### Custom API path

```text
Identity Sphere
      |
      v
Custom API Utility
      |
      v
APIGEE Gateway
      |
      v
AWS-hosted Custom API
```

### Explicitly avoid unnecessary architecture

Do not introduce the following unless a future requirement explicitly requires them:

- Router frameworks
- Factory hierarchies
- Adapter hierarchies
- Repository/DAO layers
- Generic integration engines
- CRUD abstractions
- Token microservices
- Database-backed token stores
- Distributed caches
- Generic response-normalization frameworks
- Other abstraction layers that do not directly satisfy a requirement

The design principle is:

> Keep the implementation simple, readable, reusable, configuration-driven, and easy to troubleshoot.

---

## Locked APIGEE Terminology

For this architecture:

- **APIGEE** refers to the Google Apigee platform.
- **APIGEE Gateway** refers to the gateway layer through which Identity Sphere accesses the Custom APIs.
- **APIGEE Gateway APIs** refers to the Custom APIs accessed through APIGEE Gateway.
- Do not use legacy or ambiguous gateway abbreviations in this Identity Sphere architecture; use **APIGEE Gateway** consistently.

The Custom API Framework is specifically designed for Custom APIs accessed through **APIGEE Gateway**.

## 4. Environment & Configuration Model

Identity Sphere supports three environments:

- DEV
- STAGE
- PROD

### Environment behavior

| Environment | APIGEE Gateway Endpoint | Ping Federate Token Endpoint |
|---|---|---|
| DEV | DEV-specific | Shared DEV/STAGE endpoint |
| STAGE | STAGE-specific | Shared DEV/STAGE endpoint |
| PROD | PROD-specific | Separate PROD endpoint |

The APIGEE Gateway base URL differs by environment.

DEV and STAGE use the same Ping Federate token-generation endpoint.

PROD uses a separate Ping Federate token-generation endpoint.

### Configuration vs. secrets

#### Non-sensitive configuration may be committed to GitHub

Examples:

- API definitions
- API IDs and names
- API Products and Bundles
- API paths
- HTTP methods
- APIGEE Gateway URLs
- environment configuration
- non-secret Client IDs where appropriate
- API metadata

#### Sensitive values must never be committed to GitHub

Examples:

- Client Secrets
- passwords
- secret values
- other confidential credentials

### Local development

For local PC development/testing, environment-specific secret/configuration files may be used for DEV, STAGE, and PROD.

These files must be excluded through `.gitignore` and must never be committed.

### AWS runtime

For AWS deployment, sensitive runtime values shall be supplied through **AWS Secrets Manager**.

Non-sensitive configuration can remain in version-controlled configuration files.

The implementation must keep a clear separation between configuration and secret values.

---

## 5. Custom API Framework — Overall Model

Custom APIs are organized using the following hierarchy:

```text
API Product
    |
    +-- API Bundle
    |      |
    |      +-- API Endpoint
    |      +-- API Endpoint
    |      +-- API Endpoint
    |
    +-- API Bundle
           |
           +-- API Endpoint
```

Example:

```text
Payroll
    |
    +-- Bundle 1
    |      +-- Employee Search
    |      +-- Employee Details
    |
    +-- Bundle 2
           +-- Payroll Search
           +-- Salary Details
```

### Product-level credential relationship

Each API Product has **one registered Client ID + Client Secret in Ping Federate**.

That credential is shared by all API Bundles and API Endpoints belonging to that Product.

This is a fundamental design rule and must not be changed at the individual Bundle or Endpoint level.

---

# 6. Requirement 1 — Configuration-Driven Custom API Framework

## 6.1 Objective

Provide one reusable **Custom API Utility** for all Custom APIs accessed through APIGEE Gateway.

Adding, modifying, enabling, disabling, or removing an API must be possible through configuration changes rather than requiring code changes.

## 6.2 Functional model

```text
API Product
    |
    +-- API Bundle
          |
          +-- API Endpoint
```

The user selects:

1. API Product
2. API Bundle
3. API Endpoint

The user supplies the required dynamic request information.

The Custom API Utility resolves the API configuration, builds the request, invokes the API, and returns the raw backend response.

## 6.3 Authenticated Custom API flow

For APIs requiring bearer authentication:

```text
Product Client ID + Client Secret
            |
            v
Ping Federate Token Endpoint
            |
            v
Bearer Token
            |
            v
APIGEE Gateway
            |
            v
Token/client validation + Ping Federate introspection
            |
            v
AWS-hosted Custom API
            |
            v
Raw Response
```

The Custom API Utility is responsible for:

- obtaining the token when required
- constructing the API request
- applying the bearer token
- calling APIGEE Gateway
- returning the raw response

## 6.4 Open API exception

Some Custom APIs are open APIs and require no bearer token.

Configuration:

```json
"authentication": {
  "type": "NONE"
}
```

For these APIs:

- Token generation is skipped.
- No Authorization header is sent.
- The API is called directly through APIGEE Gateway.
- The raw response is returned.

## 6.5 APIGEE Gateway request model

The effective endpoint is constructed using:

```text
APIGEE Gateway Base URL
      +
API Path
      +
Resolved Path Parameters
      +
Resolved Query Parameters
```

The implementation must use the configured HTTP method, headers, request body, and authentication behavior.

## 6.6 Configuration-driven lifecycle

### Add API

Add the API definition to configuration.

### Modify API

Modify the applicable configuration fields.

### Enable API

Set:

```json
"enabled": true
```

### Disable API

Set:

```json
"enabled": false
```

### Remove API

Remove its configuration definition.

No code change should be required for these API lifecycle operations.

## 6.7 Raw response requirement

The Custom API Utility shall return the raw backend response.

The utility should not perform unnecessary response normalization, business transformation, or UI-specific formatting.

The calling module is responsible for parsing and presentation.

---

# 7. JSON Configuration Specification

The API definition is configuration-driven.

### Core fields

| Field | Why are we using this field? | Example | Required? |
|---|---|---|---|
| `apiId` | Provides a unique stable identifier for the API and prevents ambiguity between endpoints. | `PAYROLL-EMPLOYEE-SEARCH` | Yes |
| `apiName` | Provides a human-readable API name for users and configuration readability. | `Employee Search` | Yes |
| `apiProduct` | Identifies the Product and establishes the Product-level credential/token context. | `Payroll` | Yes |
| `apiBundle` | Identifies the Bundle containing the endpoint. | `Bundle1` | Yes |
| `version` | Identifies the API version being invoked. | `v1` | Yes |
| `enabled` | Allows an API to be enabled or disabled without code changes. | `true` | Yes |
| `baseUrl` | Defines the APIGEE Gateway base URL used to reach the API. | `https://apigee.example.com` | Yes |
| `apiPath` | Defines the configured path of the actual Custom API. | `/payroll/v1/employees/search` | Yes |
| `httpMethod` | Defines how the endpoint must be invoked. | `GET` / `POST` | Yes |
| `authentication.type` | Defines whether the API requires bearer authentication or is open. | `BEARER` / `NONE` | Yes |
| `pathParameters` | Defines dynamic values that must be inserted into the API path. | `{ "employeeId": "${employeeId}" }` | Only when applicable |
| `queryParameters` | Defines dynamic URL query parameters. | `{ "status": "${status}" }` | Only when applicable |
| `headers` | Defines non-sensitive API-specific headers required by the endpoint. | `{ "Accept": "application/json" }` | Only when applicable |
| `requestBody` | Defines the request payload for body-based requests such as POST. | `{ ... }` | Only when applicable |
| `contentType` | Defines the content type of the request. | `application/json` | Only when applicable |
| `responseType` | Documents the expected raw response representation. | `JSON` | Yes |
| `description` | Explains the purpose of the API for maintainability and user understanding. | `Search employee information` | Yes |

### Example authenticated API definition

```json
{
  "apiId": "PAYROLL-EMPLOYEE-SEARCH",
  "apiName": "Employee Search",
  "apiProduct": "Payroll",
  "apiBundle": "Bundle1",
  "version": "v1",
  "enabled": true,
  "baseUrl": "https://apigee.example.com",
  "apiPath": "/payroll/v1/employees/search",
  "httpMethod": "GET",
  "authentication": {
    "type": "BEARER"
  },
  "pathParameters": {},
  "queryParameters": {
    "employeeId": "${employeeId}",
    "status": "${status}"
  },
  "headers": {
    "Content-Type": "application/json"
  },
  "requestBody": null,
  "contentType": "application/json",
  "responseType": "JSON",
  "description": "Search employee information"
}
```

### Example open API definition

```json
{
  "apiId": "PAYROLL-OPEN-SEARCH",
  "apiName": "Open Employee Search",
  "apiProduct": "Payroll",
  "apiBundle": "Bundle1",
  "version": "v1",
  "enabled": true,
  "baseUrl": "https://apigee.example.com",
  "apiPath": "/payroll/v1/open/employees",
  "httpMethod": "GET",
  "authentication": {
    "type": "NONE"
  },
  "pathParameters": {},
  "queryParameters": {},
  "headers": {},
  "requestBody": null,
  "contentType": null,
  "responseType": "JSON",
  "description": "Open employee search"
}
```

### JSON design rule

> JSON defines **what API is being called and how it must be invoked**. Common implementation logic remains in code.

JSON must not become executable business logic.

Client Secrets, passwords, AWS secret values, and other sensitive credentials must not be placed in API definitions.

---

# 8. Requirement 2 — Token Visibility & Intelligent Token Reuse

## 8.1 Objective

Identity Sphere shall maintain and reuse bearer tokens at the **API Product level**.

## 8.2 Token ownership

```text
API Product
      |
      +-- Client ID + Client Secret
      |
      v
Bearer Token
      |
      +-- Bundle 1 APIs
      +-- Bundle 2 APIs
      +-- Bundle 3 APIs
```

A valid token generated for a Product shall be reusable by all authenticated APIs under that Product.

Changing the Bundle within the same Product does not require a new token.

Changing the API Product changes the credential/token context.

## 8.3 Token lifecycle

Before an authenticated API call:

```text
Authenticated API?
      |
      +-- NO --> Call API without token
      |
      +-- YES
            |
            v
       Token available?
            |
            +-- NO --> Generate token
            |
            +-- YES
                  |
                  v
          Check validity/lifetime
                  |
          +-------+-------+
          |               |
       Valid          Expired /
       enough          near expiry
          |               |
          v               v
        Reuse        Generate new
```

A configurable token expiry safety buffer shall be used.

A reasonable default is approximately **1–2 minutes**, subject to configuration.

The purpose is to prevent a token from expiring during an API request.

## 8.4 Token state

At this stage, token state shall remain simple and application-local.

Use an in-memory token cache/state keyed by API Product.

Do not introduce:

- Token microservice
- Database token store
- Distributed cache
- External token management service

unless explicitly required by a future design change.

## 8.5 Token UI

A small token indicator shall be available near the API controls.

Example:

```text
[ Token ]  [ CURL ]
```

The normal UI must not expose the token itself.

### Hover

Hover should provide minimal information such as:

```text
Token: Valid
Client ID: <safe display value>
Expires in: 18m
```

Do not show the bearer token on hover.

### Click

Clicking the token indicator opens a detailed metadata popup containing safe information such as:

- API Product
- Client ID
- Token status
- Generation time
- Expiry time
- Remaining validity
- Token type

### Never display

- Client Secret
- Password
- AWS Secrets Manager secret value
- Other permanent credential material

The full bearer token should not be displayed by default.

## 8.6 Open API behavior

If:

```json
"authentication": {
  "type": "NONE"
}
```

token handling is completely bypassed.

---

# 9. Requirement 3 — API CURL Command

## 9.1 Objective

Identity Sphere shall provide a **CURL Command** feature for every Custom API invocation.

The feature is intended for troubleshooting and external execution in tools such as a terminal or troubleshooting environment.

## 9.2 UI behavior

A dedicated CURL indicator shall be available consistently with the Token Information indicator.

Example:

```text
[ Token ]  [ CURL ]
```

The indicator remains minimal by default.

### Hover

Hover displays:

> **View CURL Command**

No bearer token preview should be displayed on hover.

### Click

Clicking the CURL indicator opens a popup/modal containing the complete generated CURL command.

The popup shall provide a **Copy** action.

## 9.3 CURL must represent the actual resolved request

The generated command must be constructed from the **actual effective request** used for the API invocation.

It must contain, where applicable:

- HTTP method
- Complete resolved endpoint URL
- Resolved path parameters
- Resolved query parameters
- Non-sensitive required headers
- Request body
- Current bearer token for authenticated APIs

The CURL generator must not create a separate or independently interpreted request.

Conceptually:

```text
Resolved API Request
        |
        +----> Execute API
        |
        +----> Generate CURL
```

## 9.4 Authenticated API CURL

For:

```json
"authentication": {
  "type": "BEARER"
}
```

the generated CURL shall contain the current bearer token:

```bash
-H "Authorization: Bearer <CURRENT_TOKEN>"
```

The token must come from the existing token lifecycle/cache established by Requirement 2.

The CURL feature must not independently implement token generation.

If the token is expired or within the configured safety buffer, the existing token lifecycle must obtain a valid token before the authenticated request/CURL is generated.

## 9.5 Open API CURL

For:

```json
"authentication": {
  "type": "NONE"
}
```

the generated CURL shall not contain an Authorization header.

Example:

```bash
curl -X GET "https://api.example.com/open-api/..."
```

No token generation or token retrieval is required.

---

## 9.6 CURL Copy behavior

The user shall be able to copy the complete CURL command.

After copying, the UI should clearly remind the user:

> **CURL copied — contains a temporary bearer token. Do not share or store this command.**

The command must not be unnecessarily stored in application state beyond what is required for the active UI interaction.

---

# 9.7 CURL Security Requirements

Because authenticated CURL contains a temporary bearer token, the generated command is considered **sensitive**.

### 9.7.1 Permanent credentials must never be included

The CURL must never contain:

- Client Secret
- Password
- AWS Secrets Manager values
- Permanent credentials
- Other confidential credential material

### 9.7.2 Sensitive headers must be protected

Only safe/non-sensitive API headers should be included directly.

If an API uses a secret-bearing header or another sensitive credential value, that secret value must not be exposed in the generated CURL.

The implementation must distinguish safe request metadata from secret credential material.

### 9.7.3 CURL must not be logged

Identity Sphere must not unnecessarily log:

- Complete CURL commands
- Bearer tokens
- Authorization headers
- Secret values
- Sensitive headers

Do not place the generated CURL into normal application logs, debug logs, analytics events, or telemetry.

### 9.7.4 CURL must not be persisted

Do not unnecessarily persist:

- Generated CURL commands
- Bearer tokens contained in generated CURL
- CURL history
- CURL commands in browser/local storage
- CURL commands in application history

### 9.7.5 CURL must never be executed by Identity Sphere

Identity Sphere shall only:

```text
Generate
   |
Display
   |
Copy
```

Identity Sphere shall never:

```text
Generate
   |
Execute
```

The user may independently execute the copied CURL externally.

### 9.7.6 Shell/command injection protection

Dynamic values used to construct the CURL must be safely quoted and escaped.

The implementation must correctly handle:

- Shell-sensitive characters
- Quotes
- Spaces
- Special characters
- Query parameters
- Header values
- Request body content

User-controlled input must not be able to turn the generated command into unintended shell commands.

### 9.7.7 Sensitive data in URL

Resolved URLs may contain sensitive business/request information.

The CURL may represent the actual resolved request as required for troubleshooting, but Identity Sphere must not unnecessarily log, persist, or transmit the generated URL or complete command.

---

## 9.8 Token and CURL relationship

Requirement 3 must reuse Requirement 2's token lifecycle.

There must be one token lifecycle, not separate token logic for:

- API execution
- Token UI
- CURL generation

The CURL feature is a representation of the existing request.

---

# 10. Cross-Cutting Security Rules

The following rules apply across the Identity Sphere design.

## Secrets

- Never commit secrets to GitHub.
- Never place Client Secrets in API JSON.
- Use ignored local secret files for local development/testing.
- Use AWS Secrets Manager for sensitive AWS runtime configuration.

## Tokens

- Maintain tokens at API Product level.
- Reuse valid tokens.
- Apply expiry safety buffer.
- Never display permanent credentials.
- Do not expose full bearer tokens in normal UI.

## Logging

Never log:

- Client Secrets
- Passwords
- AWS secret values
- Bearer tokens
- Authorization headers
- Complete authenticated CURL commands
- Sensitive secret-bearing headers

## Persistence

Do not unnecessarily persist credential-bearing request data.

## Input handling

All dynamic values must be safely handled when building URLs, headers, request bodies, and CURL commands.

## Principle

> Credentials are supplied securely by the runtime environment; configuration describes APIs; utilities execute requests; the calling application handles raw responses and presentation.

---

# 11. Requirement Dependency Map

The requirements are related but independently defined.

```text
Requirement 1
Configuration-Driven Custom API Framework
          |
          +----------------------+
          |                      |
          v                      v
Requirement 2              Requirement 3
Token Visibility &         CURL Command
Intelligent Reuse
          |                      |
          +----------+-----------+
                     |
                     v
             Custom API Request
                Lifecycle
```

### Requirement 2 depends on Requirement 1

Requirement 2 reuses:

- API Product
- API authentication configuration
- Client ID
- Secure secret retrieval
- Environment configuration
- Custom API Utility

Requirement 2 must not create a separate authentication architecture.

### Requirement 3 depends on Requirements 1 and 2

Requirement 3 reuses:

- API configuration
- Resolved URL
- HTTP method
- Headers
- Request body
- Authentication type
- Current valid bearer token
- Token expiry/renewal behavior

Requirement 3 must not create a second API request-building or token-management implementation.

---

# 12. Sequential Implementation Plan

Implementation shall proceed in this order.

## Phase 1 — Requirement 1

Implement:

- Configuration-driven Custom API Utility
- API Product → Bundle → Endpoint hierarchy
- JSON API definitions
- Environment configuration
- Secure secret retrieval boundary
- Bearer authentication
- Open API support
- Raw response handling

Validate Requirement 1 before proceeding.

## Phase 2 — Requirement 2

Build on Requirement 1.

Implement:

- Product-level token association
- Token reuse
- Token validity checking
- Expiry safety buffer
- Token refresh when required
- Token indicator
- Hover metadata
- Detailed token metadata popup

Do not duplicate token generation logic.

Validate Requirement 2 before proceeding.

## Phase 3 — Requirement 3

Build on Requirements 1 and 2.

Implement:

- CURL indicator
- CURL tooltip
- CURL popup/modal
- Actual resolved request generation
- Bearer token inclusion where required
- Authorization omission for open APIs
- Copy functionality
- Security warning
- Sensitive-header protection
- No unnecessary logging/persistence
- Safe command escaping
- No application-side CURL execution

Validate Requirement 3 before considering the three requirements complete.

---

# 13. Acceptance Criteria

## Requirement 1

- [ ] Custom APIs are configuration-driven.
- [ ] API Product → Bundle → Endpoint hierarchy is supported.
- [ ] One Product-level credential is shared by its APIs.
- [ ] DEV, STAGE, and PROD configuration is supported.
- [ ] DEV/STAGE share the token endpoint.
- [ ] PROD uses its separate token endpoint.
- [ ] Authenticated APIs obtain bearer tokens.
- [ ] APIGEE Gateway is used for Custom API invocation.
- [ ] Open APIs skip token generation.
- [ ] APIs can be added without code changes.
- [ ] APIs can be enabled/disabled through configuration.
- [ ] APIs can be modified/removed through configuration.
- [ ] Secrets are not committed to GitHub.
- [ ] Raw backend responses are returned.

## Requirement 2

- [ ] Tokens are associated with API Product.
- [ ] Valid Product tokens are reused across Bundles/Endpoints.
- [ ] Token validity is checked before authenticated calls.
- [ ] Expiry safety buffer is applied.
- [ ] Expired/near-expiry tokens are renewed.
- [ ] Open APIs bypass token handling.
- [ ] Token indicator is available.
- [ ] Hover displays minimal safe metadata.
- [ ] Click displays detailed safe metadata.
- [ ] Client Secrets are never displayed.
- [ ] Passwords are never displayed.
- [ ] AWS secret values are never displayed.
- [ ] Full bearer token is not displayed by default.

## Requirement 3

- [ ] CURL indicator is available.
- [ ] Hover displays “View CURL Command”.
- [ ] Click opens the CURL modal.
- [ ] CURL uses the actual resolved API request.
- [ ] HTTP method is correct.
- [ ] Complete resolved URL is included.
- [ ] Path parameters are resolved.
- [ ] Query parameters are resolved.
- [ ] Safe required headers are included.
- [ ] Request body is included where applicable.
- [ ] Current bearer token is included for authenticated APIs.
- [ ] Authorization is omitted for `authentication.type = NONE`.
- [ ] Copy functionality works.
- [ ] Security warning is displayed for authenticated CURL.
- [ ] Client Secrets are never included.
- [ ] Passwords are never included.
- [ ] AWS Secrets Manager values are never included.
- [ ] Sensitive secret-bearing header values are never included.
- [ ] CURL is not unnecessarily logged.
- [ ] CURL is not unnecessarily persisted.
- [ ] Bearer tokens are not unnecessarily logged or persisted.
- [ ] CURL is never executed by Identity Sphere.
- [ ] Dynamic values are safely quoted/escaped.
- [ ] Requirement 2 token lifecycle is reused.

---

# 14. Do Not Assume / Do Not Invent

GitHub Copilot must not invent implementation-specific values or architecture.

Do not assume or fabricate:

- Production APIGEE Gateway URLs
- Ping Federate URLs
- Client IDs
- Client Secrets
- AWS Secrets Manager secret names
- API paths
- API request schemas
- Vendor-specific authentication behavior
- Undocumented API headers
- Database requirements
- Additional infrastructure
- Additional services
- Additional authentication mechanisms

When these values are required, they will be supplied separately by the user during implementation.

If a value is dynamic, keep the implementation parameterized/configuration-driven.

If a requirement is not defined in this document, do not silently create a new behavior that changes the architecture or security model.

The implementation should follow the simplest design that satisfies the locked requirements.

---

# Final Locked Design Summary

Identity Sphere uses a simple utility-based architecture.

```text
Identity Sphere
      |
      +-- LDAP Utility
      |
      +-- Custom API Utility
      |       |
      |       v
      |     APIGEE Gateway
      |       |
      |       v
      |     AWS APIs
      |
      +-- OOTB Utility
```

For Custom APIs:

```text
API Product
    |
    +-- API Bundle
          |
          +-- API Endpoint
```

Requirement 1 establishes the configuration-driven Custom API framework.

Requirement 2 establishes Product-level bearer-token reuse and safe token visibility.

Requirement 3 provides a troubleshooting CURL representation of the actual API request while explicitly controlling credential exposure, logging, persistence, sensitive headers, command injection, and execution risk.

The fundamental implementation principle is:

> **JSON defines the API; Custom API Utility executes it; secrets are securely supplied by the runtime environment; token lifecycle is maintained at the API Product level; CURL represents the actual resolved request; and the calling application handles the raw response.**

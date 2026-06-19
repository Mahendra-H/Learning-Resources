# IAM Multi-Agent AI System — High-Level Design (POC)
> **100% Free & Local Stack · No Paid APIs · Personal Laptop Friendly**

---

## 1. Executive Summary

This document defines the complete High-Level Design for converting the existing IAM request portal (Application X) into a **conversational, multi-agent AI system**. Users interact via natural language to perform IAM operations — SSO onboarding (Ping Federate), safe creation (CyberArk), group management (Saviynt) — without filling forms or raising tickets.

The POC is built entirely on **free, open-source, locally-runnable** components. No cloud API costs are incurred during validation.

---

## 2. Problem Decomposition

The system must handle three primary IAM domains:

| Domain | Platform | Sample User Intent |
|--------|----------|--------------------|
| SSO / Federation | Ping Federate | "Onboard my app with OIDC" |
| Privileged Access Mgmt | CyberArk | "Create a safe for team X" |
| Identity Governance | Saviynt | "Create a Web SSO group for project Y" |

Each domain has its own data model, API contract, and workflow logic. A single monolithic LLM call cannot reliably handle all three — hence the **multi-agent architecture**.

---

## 3. Architecture Overview

### 3.1 Architectural Pattern: Supervisor + Specialist Agents

```
User (Chat UI)
     │  natural language
     ▼
┌─────────────────────────────────────────────────────────────┐
│                    FRONTEND (React + Vite)                   │
│   Chat window · Step-by-step forms · Status tracker         │
└─────────────────────┬───────────────────────────────────────┘
                      │  WebSocket / REST
                      ▼
┌─────────────────────────────────────────────────────────────┐
│            API GATEWAY  (Express + TypeScript)               │
│   Auth middleware · Session store · Request logging          │
└──────────┬──────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────┐
│          ORCHESTRATOR AGENT  (LangChain.js + Ollama)         │
│                                                              │
│  1. Intent Classification                                    │
│  2. Context Extraction (slot filling)                        │
│  3. Route to correct specialist agent                        │
│  4. Aggregate & stream response back to user                 │
└──────┬────────────────┬────────────────┬────────────────────┘
       │                │                │
       ▼                ▼                ▼
┌────────────┐  ┌───────────────┐  ┌──────────────┐
│ SSO Agent  │  │CyberArk Agent │  │Saviynt Agent │
│ (LangChain)│  │ (LangChain)   │  │(LangChain)   │
└─────┬──────┘  └───────┬───────┘  └──────┬───────┘
      │                 │                 │
      ▼                 ▼                 ▼
┌────────────┐  ┌───────────────┐  ┌──────────────┐
│Mock Ping   │  │Mock CyberArk  │  │Mock Saviynt  │
│Federate API│  │    API        │  │    API       │
└────────────┘  └───────────────┘  └──────────────┘
           │            │               │
           └────────────┴───────────────┘
                        │
                        ▼
              ┌─────────────────┐
              │  SQLite / Local │
              │  State Store    │
              └─────────────────┘
```

### 3.2 Design Patterns Used

| Pattern | Where Applied | Purpose |
|---------|--------------|---------|
| **Supervisor / Router** | Orchestrator Agent | Routes intent to correct specialist |
| **ReAct (Reason + Act)** | All Agents | LLM reasons, picks tool, observes result, loops |
| **Tool Calling** | Each Specialist Agent | Structured function calls to mock APIs |
| **Slot Filling** | Orchestrator | Conversationally collects missing parameters |
| **Human-in-the-Loop** | All Agents | Confirmation step before executing writes |
| **Chain of Thought** | Prompt Design | LLM explains what it's about to do |

---

## 4. Technology Stack (100% Free)

### 4.1 AI & Agent Layer

| Component | Technology | Version / Notes |
|-----------|-----------|-----------------|
| **Local LLM Runtime** | [Ollama](https://ollama.com) | Free, runs on CPU/GPU, Mac/Linux/Windows |
| **Primary Model** | `llama3.1:8b` | Best quality/speed on 16GB RAM laptop |
| **Fallback Model** | `mistral:7b-instruct` | Slightly faster, good at instruction following |
| **Light/Fast Model** | `phi3:mini` | Use for intent classification (quick) |
| **Agent Framework** | LangChain.js | Free, has Ollama adapter, tool calling support |
| **Vector Store** | ChromaDB (local) or LanceDB | Free, embedded, for RAG over IAM docs |
| **Embeddings** | `nomic-embed-text` via Ollama | Free, local embedding generation |

### 4.2 Backend

| Component | Technology |
|-----------|-----------|
| Runtime | Node.js 20 LTS |
| Language | TypeScript 5.x |
| HTTP Framework | Express.js or Fastify |
| WebSocket | Socket.io |
| ORM / DB | Prisma + SQLite (zero infra) |
| Validation | Zod |
| Testing | Vitest |
| Logging | Pino |

### 4.3 Frontend

| Component | Technology |
|-----------|-----------|
| Framework | React 18 + Vite |
| Language | TypeScript |
| Styling | Tailwind CSS |
| Component Library | shadcn/ui (free, copy-paste) |
| State Management | Zustand |
| Chat UI | Custom (or `@assistant-ui/react` — free) |
| Forms | React Hook Form + Zod |
| Icons | Lucide React |
| HTTP / WS client | Axios + Socket.io-client |

### 4.4 Mock Provider Layer

| System | Mock Strategy | Technology |
|--------|--------------|-----------|
| **Ping Federate** | Mock REST server simulating PF Admin API | json-server + custom middleware |
| **CyberArk** | Mock CyberArk PVWA API endpoints | Express mock routes |
| **Saviynt** | Mock Saviynt REST API | Express mock routes |

### 4.5 Infrastructure / Tooling

| Component | Technology |
|-----------|-----------|
| Package Manager | pnpm (monorepo) |
| Monorepo Tool | Turborepo (free) |
| Containerisation | Docker Compose (optional, for mock services) |
| API Documentation | Swagger UI (via swagger-jsdoc) |
| Environment Config | dotenv |

---

## 5. AI Model Recommendations for Personal Laptop

### 5.1 System Requirements

| Model | RAM Required | Disk | CPU Speed | GPU? |
|-------|-------------|------|-----------|------|
| `phi3:mini` (3.8B) | 4 GB | 2.3 GB | Fast | Optional |
| `mistral:7b-instruct` | 8 GB | 4.1 GB | Medium | Optional |
| `llama3.1:8b` | **16 GB** | 4.7 GB | Good | Optional |
| `llama3.1:70b` | 48 GB | 40 GB | Slow | Required |

**Recommendation for 16GB MacBook/PC**: Use `llama3.1:8b` as the primary model. It has native function/tool calling support, critical for the agent pattern.

### 5.2 Model Role Assignment

```
Intent Classification    →  phi3:mini         (fast, cheap inference)
Slot Filling / Dialog    →  llama3.1:8b       (conversational quality)
Tool Calling / Actions   →  llama3.1:8b       (native tool call support)
Error Explanation        →  llama3.1:8b       (clear natural language)
Document RAG             →  mistral:7b        (good at retrieval tasks)
```

### 5.3 Ollama Setup

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Pull models
ollama pull llama3.1:8b
ollama pull phi3:mini
ollama pull nomic-embed-text

# Verify
ollama run llama3.1:8b "Hello, can you help me with IAM tasks?"
```

---

## 6. Agent Design (Detailed)

### 6.1 Orchestrator Agent

**Responsibility**: Receive user message → classify intent → extract parameters → dispatch to specialist agent → return response.

**System Prompt Design**:
```
You are an IAM assistant for an enterprise identity platform.
You help users perform IAM operations via natural conversation.

Available operations:
- SSO onboarding (OIDC/SAML application registration with Ping Federate)
- CyberArk safe creation and management
- Saviynt group creation and access management

Instructions:
1. Identify which operation the user wants
2. If parameters are missing, ask ONE question at a time
3. Before executing any action, summarize and confirm with the user
4. After completion, provide a clear success/failure message with reference IDs

Never execute actions without explicit user confirmation.
```

**Tools Available to Orchestrator**:
- `classify_intent(user_message)` → returns `{ domain, action, confidence }`
- `route_to_agent(domain, context)` → triggers specialist agent
- `ask_clarification(question)` → prompts user for more info

### 6.2 SSO Agent (Ping Federate)

**Responsibility**: Handle all SSO onboarding workflows — OIDC and SAML.

**Tools Available**:
```typescript
tools = [
  {
    name: "create_oidc_connection",
    description: "Creates an OIDC client connection in Ping Federate",
    parameters: {
      applicationName: string,
      redirectUri: string[],
      grantTypes: ("authorization_code" | "client_credentials")[],
      scopes: string[],
      environment: "dev" | "staging" | "prod"
    }
  },
  {
    name: "create_saml_connection",
    description: "Creates a SAML SP connection in Ping Federate",
    parameters: {
      spEntityId: string,
      acsUrl: string,
      nameIdFormat: string,
      signingCertificate?: string
    }
  },
  {
    name: "get_connection_status",
    description: "Check status of an existing connection",
    parameters: { connectionId: string }
  },
  {
    name: "list_existing_connections",
    description: "List all SP/OIDC connections",
    parameters: { filter?: string }
  }
]
```

**ReAct Loop Example**:
```
User: "I want to onboard my app 'HR Portal' using OIDC"

Thought: The user wants to create an OIDC connection. I need:
  - Application name: "HR Portal" ✓
  - Redirect URI: MISSING — need to ask
  - Grant types: not specified, I'll assume authorization_code

Action: ask_clarification("What is the redirect URI for HR Portal?")
Observation: User provides "https://hr.company.com/callback"

Thought: I have enough info. Let me confirm before executing.
Action: confirm_with_user("I'll create an OIDC connection with these details:
  - App: HR Portal
  - Redirect URI: https://hr.company.com/callback
  - Grant Type: Authorization Code
  Shall I proceed?")
Observation: User says "Yes"

Action: create_oidc_connection({...})
Observation: { clientId: "abc123", clientSecret: "***", status: "active" }

Response: "HR Portal has been successfully onboarded. Here are your credentials:
  - Client ID: abc123
  - Client Secret: [hidden — will be sent to your email]
  - Metadata URL: https://ping.company.com/.well-known/abc123"
```

### 6.3 CyberArk Agent

**Tools Available**:
```typescript
tools = [
  {
    name: "create_safe",
    description: "Creates a new CyberArk safe",
    parameters: {
      safeName: string,
      description: string,
      managingCPM: string,
      numberOfDaysRetention: number,
      owners: string[]
    }
  },
  {
    name: "add_safe_member",
    parameters: { safeName: string, memberName: string, permissions: string[] }
  },
  {
    name: "get_safe_details",
    parameters: { safeName: string }
  },
  {
    name: "list_safes",
    parameters: { filter?: string }
  }
]
```

### 6.4 Saviynt Agent

**Tools Available**:
```typescript
tools = [
  {
    name: "create_group",
    description: "Creates a new group in Saviynt",
    parameters: {
      groupName: string,
      groupType: "Web SSO" | "Application" | "AD" | "Distribution",
      description: string,
      owners: string[],
      entitlements?: string[]
    }
  },
  {
    name: "add_group_member",
    parameters: { groupName: string, userId: string, accessLevel: string }
  },
  {
    name: "request_access",
    parameters: { userId: string, resourceName: string, justification: string }
  }
]
```

---

## 7. Mock API Design

### 7.1 Mock Ping Federate API

Base URL: `http://localhost:9031/pf-admin-api/v1`

```
POST   /oauth/clients                →  Create OIDC client
GET    /oauth/clients/:id            →  Get client details
PUT    /oauth/clients/:id            →  Update client
POST   /idp/spConnections            →  Create SAML SP connection
GET    /idp/spConnections            →  List all connections
GET    /serverSettings               →  Server metadata
```

Mock behavior: Returns realistic fake `clientId`, `clientSecret`, metadata URLs. Stores state in a local JSON file.

### 7.2 Mock CyberArk PVWA API

Base URL: `http://localhost:1858/PasswordVault/API`

```
POST   /Safes                        →  Create safe
GET    /Safes                        →  List safes
GET    /Safes/:safeName              →  Get safe details
POST   /Safes/:safeName/Members      →  Add member
DELETE /Safes/:safeName/Members/:id  →  Remove member
POST   /Auth/CyberArk/Logon          →  Mock auth (returns token)
```

### 7.3 Mock Saviynt API

Base URL: `http://localhost:8080/ECM/api/v5`

```
POST   /groups                       →  Create group
GET    /groups                       →  List groups
POST   /groups/:id/members           →  Add member
POST   /requests                     →  Submit access request
GET    /requests/:id                 →  Check request status
```

### 7.4 Mock Server Implementation

Use `json-server` for rapid mock APIs with state:

```bash
pnpm add -D json-server
```

```json
// db.json
{
  "oidc_clients": [],
  "saml_connections": [],
  "safes": [],
  "safe_members": [],
  "groups": [],
  "access_requests": []
}
```

Add custom middleware for validation and realistic ID generation.

---

## 8. Conversation Flow Design

### 8.1 Happy Path — OIDC Onboarding

```
User:    "I need to onboard my application using OIDC"
Bot:     "I can help with that. What's the name of your application?"
User:    "PaymentService"
Bot:     "Got it. What protocol are you using — OIDC or SAML?"
User:    "OIDC"
Bot:     "What redirect URIs should be registered? 
          (You can provide multiple, separated by commas)"
User:    "https://payments.co/callback, https://payments.co/silent-renew"
Bot:     "What grant types do you need?
          [ Authorization Code ]  [ Client Credentials ]  [ Both ]"
User:    [clicks "Authorization Code"]
Bot:     "Here's a summary of what I'll create:
          ────────────────────────────────────
          Application:  PaymentService
          Protocol:     OIDC
          Redirect URIs: https://payments.co/callback
                         https://payments.co/silent-renew
          Grant Type:   Authorization Code
          Environment:  Development (default)
          ────────────────────────────────────
          Shall I proceed? [ Confirm ] [ Edit ] [ Cancel ]"
User:    [clicks "Confirm"]
Bot:     "✅ Successfully created!
          Client ID:     pf-client-a3f9b2c1
          Metadata URL:  https://ping-mock/oidc/.well-known/pf-client-a3f9b2c1
          
          Your client secret has been stored. Request ID: REQ-2024-0042
          Need SAML as well, or anything else?"
```

### 8.2 Error / Ambiguity Flow

```
User:    "Create a safe for my team"
Bot:     "Sure! A few quick questions:
          What would you like to name the safe?"
User:    "infra-prod"
Bot:     "Got it. How many days should credentials be retained? (default: 30)"
User:    "I don't know"
Bot:     "No problem — I'll use the default of 30 days. 
          Who should be the safe owner(s)? 
          Please provide usernames or email addresses."
```

---

## 9. Frontend Architecture

### 9.1 Application Structure

```
apps/
  web/                        ← React + Vite frontend
    src/
      components/
        chat/
          ChatWindow.tsx       ← Main conversation area
          MessageBubble.tsx    ← Individual message rendering
          TypingIndicator.tsx  ← "Agent is thinking..."
          ConfirmationCard.tsx ← Step summary before action
          StatusBadge.tsx      ← Request status tracker
        forms/
          SlotForm.tsx         ← Dynamic form for missing params
        layout/
          Sidebar.tsx          ← Request history
          Header.tsx
      pages/
        ChatPage.tsx
        HistoryPage.tsx
        StatusPage.tsx
      stores/
        chatStore.ts           ← Zustand: messages, session
        requestStore.ts        ← Zustand: pending requests
      hooks/
        useSocket.ts           ← WebSocket connection
        useChat.ts             ← Send/receive messages
      lib/
        api.ts                 ← Axios instance
        socket.ts              ← Socket.io client
```

### 9.2 Message Types (UI Schema)

```typescript
type MessageRole = "user" | "assistant" | "system";

type MessageContent =
  | { type: "text"; content: string }
  | { type: "confirmation_card"; summary: ConfirmationSummary }
  | { type: "status_update"; requestId: string; status: RequestStatus }
  | { type: "quick_replies"; options: string[] }
  | { type: "form"; fields: FormField[] }
  | { type: "result_card"; data: Record<string, string> };
```

### 9.3 UI/UX Design Principles

- **Progressive disclosure**: Never ask all questions at once. One slot at a time.
- **Confirmation before action**: Always show a summary card before write operations.
- **Streaming responses**: Use Server-Sent Events or WebSocket to stream LLM output token by token.
- **Request tracking**: Every submitted request gets an ID visible in a sidebar panel.
- **Context persistence**: Conversation history maintained per session in `sessionStorage` + DB.
- **Quick replies**: Common options rendered as clickable chips, not free text.

---

## 10. Backend Architecture

### 10.1 Project Structure

```
apps/
  api/                         ← Express + TypeScript backend
    src/
      routes/
        chat.ts                ← POST /chat, GET /history
        requests.ts            ← GET /requests, GET /requests/:id
        health.ts
      agents/
        orchestrator.ts        ← Supervisor agent
        ssoAgent.ts            ← Ping Federate specialist
        cyberarkAgent.ts       ← CyberArk specialist
        saviyntAgent.ts        ← Saviynt specialist
      tools/
        pingFederateTools.ts   ← Tool definitions + API calls
        cyberarkTools.ts
        saviyntTools.ts
      prompts/
        systemPrompts.ts       ← All system prompts
        fewShotExamples.ts     ← Few-shot examples per domain
      services/
        ollamaService.ts       ← Ollama API wrapper
        sessionService.ts      ← Conversation state
        requestService.ts      ← Request CRUD
      middleware/
        auth.ts                ← Simple JWT for POC
        rateLimit.ts
      db/
        schema.prisma
        seed.ts
      websocket/
        chatHandler.ts         ← Socket.io event handlers
  mock-apis/                   ← Mock provider servers
    ping-federate/
    cyberark/
    saviynt/
packages/
  shared/                      ← Shared TypeScript types
```

### 10.2 LangChain.js Agent Implementation

```typescript
// agents/ssoAgent.ts
import { ChatOllama } from "@langchain/community/chat_models/ollama";
import { AgentExecutor, createReactAgent } from "langchain/agents";
import { createOidcConnectionTool, createSamlConnectionTool } from "../tools/pingFederateTools";

const model = new ChatOllama({
  model: "llama3.1:8b",
  baseUrl: "http://localhost:11434",
  temperature: 0,
});

const tools = [
  createOidcConnectionTool,
  createSamlConnectionTool,
  getConnectionStatusTool,
];

export async function runSsoAgent(userMessage: string, context: ConversationContext) {
  const agent = await createReactAgent({ llm: model, tools, prompt: SSO_AGENT_PROMPT });
  const executor = new AgentExecutor({ agent, tools, verbose: true });
  
  const result = await executor.invoke({
    input: userMessage,
    chat_history: context.history,
  });
  
  return result.output;
}
```

### 10.3 API Endpoints

```
POST   /api/chat                    ← Send message, get response
GET    /api/chat/history/:sessionId ← Conversation history
POST   /api/chat/confirm            ← User confirmation for an action
POST   /api/chat/cancel             ← Cancel pending action

GET    /api/requests                ← List all IAM requests
GET    /api/requests/:id            ← Get request details
DELETE /api/requests/:id            ← Cancel a pending request

GET    /api/health                  ← Healthcheck

WS     /socket.io                   ← Real-time chat channel
```

---

## 11. Data Model

### 11.1 Core Entities (SQLite / Prisma)

```prisma
model Session {
  id         String    @id @default(cuid())
  userId     String
  messages   Message[]
  createdAt  DateTime  @default(now())
  updatedAt  DateTime  @updatedAt
}

model Message {
  id         String   @id @default(cuid())
  sessionId  String
  session    Session  @relation(fields: [sessionId], references: [id])
  role       String   // "user" | "assistant" | "system"
  content    String
  metadata   Json?    // type, quick_replies, form_fields, etc.
  createdAt  DateTime @default(now())
}

model IamRequest {
  id           String   @id @default(cuid())
  sessionId    String
  domain       String   // "sso" | "cyberark" | "saviynt"
  action       String   // "create_oidc" | "create_safe" | ...
  status       String   // "pending" | "confirmed" | "processing" | "success" | "failed"
  payload      Json     // The parameters collected
  result       Json?    // The API response
  requestedBy  String
  createdAt    DateTime @default(now())
  completedAt  DateTime?
}
```

---

## 12. Security Considerations for POC

- **No real credentials**: All CyberArk/Ping secrets are mock values
- **Simple JWT auth**: Single shared secret for POC (`JWT_SECRET=dev-only`)
- **Input validation**: Zod schemas on all agent inputs before tool calls
- **Confirmation gates**: No write operation without explicit user confirmation
- **Audit log**: All `IamRequest` records serve as an audit trail
- **Rate limiting**: Basic `express-rate-limit` to prevent runaway agent loops
- **Secrets in env files**: `.env.local` pattern, never committed to git

---

## 13. Development Phases (POC Roadmap)

### Phase 1 — Foundation (Week 1–2)
- [ ] Set up monorepo (pnpm + Turborepo)
- [ ] Install and verify Ollama with `llama3.1:8b`
- [ ] Scaffold Express API + React frontend
- [ ] Basic chat UI with WebSocket streaming
- [ ] Mock API servers (Ping, CyberArk, Saviynt) with JSON persistence

### Phase 2 — First Agent (Week 2–3)
- [ ] Implement SSO Agent (OIDC flow only)
- [ ] LangChain.js ReAct loop with Ollama
- [ ] Tool definitions for Ping Federate mock
- [ ] Confirmation card UI component
- [ ] End-to-end: "Create OIDC connection" working

### Phase 3 — All Three Agents (Week 3–4)
- [ ] CyberArk Agent + safe creation tool
- [ ] Saviynt Agent + group creation tool
- [ ] Orchestrator routing between all three agents
- [ ] Intent classification with `phi3:mini`

### Phase 4 — Polish & Validation (Week 4–5)
- [ ] Slot filling / clarification dialog improvements
- [ ] Request history panel in UI
- [ ] RAG over IAM documentation (ChromaDB)
- [ ] Error handling & fallback responses
- [ ] Demo walkthrough + screen recording

---

## 14. Key Technical Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|-----------|
| `llama3.1:8b` too slow on CPU | High latency per message | Use `phi3:mini` for fast ops; accept ~10s delay in POC |
| Tool calling unreliable on 8b model | Wrong or malformed tool calls | Add strict JSON schema validation; retry with corrective prompt |
| Context window exhaustion (long chats) | Agent loses memory | Implement sliding window + summarization after N messages |
| Prompt injection via user input | Agent misbehaves | Add input sanitization; never interpolate raw user text into system prompts |
| Mock API drift from real APIs | Demo doesn't translate | Document real API contracts; keep mock schemas aligned |

---

## 15. Future Upgrade Path (Post-POC)

When the POC is validated, the following upgrades require minimal code changes:

| POC Component | Production Upgrade |
|---------------|--------------------|
| Ollama (local) | Anthropic Claude API or Azure OpenAI |
| Mock Ping API | Real Ping Federate Admin API |
| Mock CyberArk | Real CyberArk PVWA REST API |
| Mock Saviynt | Real Saviynt REST API |
| SQLite | PostgreSQL |
| Simple JWT | Integrate with your existing SSO (OIDC via Ping Federate itself) |
| LangChain.js | Same library — just swap model provider |

The architecture is **provider-agnostic by design** — swapping LLM providers is a one-line config change.

---

## 16. Quick Start Commands

```bash
# 1. Install Ollama and pull models
curl -fsSL https://ollama.com/install.sh | sh
ollama pull llama3.1:8b && ollama pull phi3:mini

# 2. Clone / scaffold project
npx create-turbo@latest iam-agent-poc
cd iam-agent-poc

# 3. Install dependencies
pnpm install

# 4. Start mock APIs
pnpm --filter mock-apis dev

# 5. Start backend
pnpm --filter api dev

# 6. Start frontend
pnpm --filter web dev

# 7. Open browser
open http://localhost:5173
```

---

*Document version: 1.0 — POC Phase · All components free & open-source*

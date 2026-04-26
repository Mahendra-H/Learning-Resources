# Anthropic Skilljar — Claude Mastery Roadmap
### Zero to Hero + Tools, Costs, and Honest Guidance

> Last reviewed: April 25, 2026 | All pricing verified against official sources at time of writing.
> ⚠️ Exchange rates, plan availability, and features change frequently. Always verify at claude.com/pricing before subscribing.

---

## ⚠️ Honest Assessment Before You Start

**If you are an experienced developer** (already use VS Code, GitHub Copilot, have a vibe-coded project):
- Skip Stage 0 entirely. Those courses are for non-technical users.
- Skim Claude 101. It covers the chat product, not Claude Code. 30 minutes max.
- Your real starting point is Claude Code 101.

**If you are a complete beginner** (new to AI tools and coding agents):
- Follow every stage in order.

---

## THE COURSES (all free on anthropic.skilljar.com)

---

### Stage 0 — AI Foundations
> ⚠️ SKIP if you are an experienced developer. Designed for non-technical users with no AI background.

| Order | Course | What You Learn |
|-------|--------|----------------|
| 0a | [AI Capabilities and Limitations](https://anthropic.skilljar.com/ai-capabilities-and-limitations) | How AI works, what it can and can't do. For complete beginners only. |
| 0b | [AI Fluency: Framework & Foundations](https://anthropic.skilljar.com/ai-fluency-framework-foundations) | Collaborating with AI effectively and safely. For complete beginners only. |

---

### Stage 1 — Claude Basics
> ⚠️ Worth doing but treat it as a 30–60 minute skim. Covers Claude.ai chat product, NOT Claude Code.

| Order | Course | What You Learn |
|-------|--------|----------------|
| 1 | [Claude 101](https://anthropic.skilljar.com/claude-101) | Core Claude features, prompting techniques, everyday use of Claude.ai. Good warmup before Claude Code. |

---

### Stage 2 — Claude Code Core ← Your Real Starting Point
> ✅ This is where SDD begins. Most important stage for your use case.
> ⚠️ You CANNOT properly practice these courses without a terminal/CLI. You can watch the videos, but the hands-on exercises require a terminal. More on this in the CLI section below.

| Order | Course | What You Learn |
|-------|--------|----------------|
| 2 | [Claude Code 101](https://anthropic.skilljar.com/claude-code-101) | Agentic loop, context window, tools & permissions. Plan Mode (read-only spec planning). Explore → Plan → Code → Commit workflow. CLAUDE.md for persistent memory. Intro to subagents, skills, MCP, hooks. Prereq: basic terminal + code editor. |
| 3 | [Claude Code in Action](https://anthropic.skilljar.com/claude-code-in-action) | Context management, custom commands, MCP servers, GitHub integration, hooks & SDK. Planning modes for complex codebases — directly relevant to reverse-speccing a vibe-coded project. Prereq: Claude Code 101. |

---

### Stage 3 — Claude Code Advanced (SDD Power Features)
> ✅ Do Skills before Subagents. The Skills course covers wiring skills into subagents — order matters.

| Order | Course | What You Learn |
|-------|--------|----------------|
| 4 | [Introduction to Agent Skills](https://anthropic.skilljar.com/introduction-to-agent-skills) | Build reusable SKILL.md files, trigger descriptions, multi-file skill directories. Share via plugins and enterprise settings. Wire skills into subagents. Troubleshoot trigger conflicts. Prereq: Claude Code 101. |
| 5 | [Introduction to Subagents](https://anthropic.skilljar.com/introduction-to-subagents) | Isolated context windows for subagents. Create custom subagents (code reviewer, doc writer, test generator). Structured outputs, tool access limits, anti-patterns, when to delegate vs stay in main context. Prereq: Claude Code 101. |

---

### Stage 4 — MCP (Connect Claude to External Services)
> ✅ Take after SDD workflow is solid.
> ⚠️ Advanced MCP is optional unless you are building custom MCP servers for your team.

| Order | Course | What You Learn |
|-------|--------|----------------|
| 6 | [Introduction to MCP](https://anthropic.skilljar.com/introduction-to-model-context-protocol) | Build MCP servers & clients using Python SDK. Tools, resources, prompts — the 3 core MCP primitives. MCP Server Inspector for debugging. Prereq: Python basics. |
| 7 | [MCP: Advanced Topics](https://anthropic.skilljar.com/model-context-protocol-advanced-topics) | Sampling, notifications, file system access, transport mechanisms. Production MCP patterns. OPTIONAL unless building custom MCP servers. Prereq: Intro to MCP. |

---

### Stage 5 — API & Building with Claude ← Certification Milestone
> ✅ Most comprehensive course on the platform. Has a final exam.
> ⚠️ Requires Python proficiency. If you are a JS/TS developer the concepts transfer but the code examples are in Python. Do not skip it for that reason.

| Order | Course | What You Learn |
|-------|--------|----------------|
| 8 | [Building with the Claude API](https://anthropic.skilljar.com/claude-with-the-anthropic-api) | Full API: auth, multi-turn chat, system prompts, streaming, tool use, RAG, extended thinking, image/PDF, prompt caching, MCP clients, agent workflows (parallelization, chaining, routing). HAS FINAL ASSESSMENT. Prereq: Python proficiency. |

---

### Stage 6 — Cloud Specialization (Optional)
> ⚠️ Only take the one that matches your deployment stack. Skip both if you deploy locally or on a neutral cloud.

| Order | Course | What You Learn |
|-------|--------|----------------|
| 9a | [Claude with Amazon Bedrock](https://anthropic.skilljar.com/claude-in-amazon-bedrock) | Claude via AWS Bedrock. Originally built as AWS employee accreditation training. Prereq: AWS background helpful. |
| 9b | [Claude with Google Vertex AI](https://anthropic.skilljar.com/claude-with-google-vertex) | Claude via Google Cloud Vertex AI. Prereq: GCP background helpful. |

---

### Bonus — Cowork
> ⚠️ NOT a developer or SDD tool. For document and file workflows only. Does not advance your SDD skills. Skip unless you have a non-code workflow need.

| Course | What You Learn |
|--------|----------------|
| [Introduction to Claude Cowork](https://anthropic.skilljar.com/introduction-to-claude-cowork) | Work alongside Claude on files and documents (not code). Task loop, plugins, file & research workflows. |

---

## Recommended Order — Quick Reference

### Experienced developer doing SDD on an existing project:
```
1.  Claude 101                     ← 30-60 min skim only
2.  Claude Code 101                ← YOUR REAL STARTING POINT ⭐
3.  Claude Code in Action          ← essential for vibe-coded projects ⭐
4.  Introduction to Agent Skills   ← SDD power feature ⭐
5.  Introduction to Subagents      ← SDD power feature ⭐
6.  Introduction to MCP
7.  MCP: Advanced Topics           ← only if building custom MCP servers
8.  Building with the Claude API   ← 🏅 certification milestone
9a. Claude with Amazon Bedrock     ← only if on AWS
9b. Claude with Google Vertex AI   ← only if on GCP
```

### Complete beginner:
```
0a. AI Capabilities and Limitations
0b. AI Fluency: Framework & Foundations
1–8 in order above
```

---

---

## TOOLS, SUBSCRIPTIONS & COST IN INR

---

### What You Actually Need to Study (the courses)

**Cost: ₹0**
All courses on anthropic.skilljar.com are completely free. No subscription, no credit card.
You can watch all videos and get all certificates without paying anything.

---

### What You Need to Actually PRACTICE

This is where real money comes in. You need a Claude subscription to run Claude Code.

---

### Claude Subscription Pricing (as of April 2026)

> ⚠️ Anthropic charges in USD only. No INR pricing, no UPI support. You need an international credit or debit card.
> ⚠️ 18% GST (OIDAR) is added at checkout for Indian accounts. This is mandatory and non-negotiable.
> ⚠️ INR amounts below are estimates based on ~₹84/USD exchange rate. These WILL change with the exchange rate. Verify the actual amount at checkout.

| Plan | USD/month | Approx INR/month (incl. 18% GST) | Claude Code included? | Best for |
|------|-----------|-----------------------------------|-----------------------|----------|
| Free | $0 | ₹0 | No | Just trying Claude chat |
| Pro | $20 | ~₹1,980 | ✅ Yes (see warning below) | Start here. Enough for moderate SDD use. |
| Max 5x | $100 | ~₹9,912 | ✅ Yes | Daily Claude Code use, hitting Pro limits regularly |
| Max 20x | $200 | ~₹19,824 | ✅ Yes | Full-time Claude Code, multi-agent, all-day sessions |

> 🚨 **CRITICAL WARNING — Claude Code on Pro (April 2026):**
> On April 21, 2026, Anthropic briefly removed Claude Code from the Pro plan and reversed it within 24 hours after community backlash. The reversal was confirmed, but an ongoing silent A/B test on ~2% of new Pro signups may still exclude Claude Code. Before subscribing, verify Claude Code is included in your specific Pro signup. Run `claude --version` in your terminal after subscribing to confirm access. If Claude Code is not included, contact support immediately.

> 💡 **Honest advice on which plan to start with:**
> Start on Pro (₹~1,980/month). Do not jump to Max immediately. Use Pro for 1–2 weeks and track how often you hit rate limits. Upgrade to Max 5x only when repeated limit interruptions are blocking your actual work. Most developers doing part-time SDD never need Max.

> ⚠️ **Shared usage pool:**
> Your Claude.ai chat, Claude Code, and Claude Desktop ALL draw from the same monthly usage limit. If you spend time in Claude chat during the day, that eats into your Claude Code budget. This catches most new users off-guard.

> ⚠️ **Data privacy — OPT OUT required:**
> Pro and Max individual plans send your conversations to Anthropic for model training BY DEFAULT. If you are working on a real project with proprietary code, go to claude.ai → Settings → Privacy and opt out before writing a single line of code. This is not optional if you care about your IP.

---

### GitHub Copilot Pricing

> 🚨 **CRITICAL: New signups for Copilot Pro and Pro+ are temporarily paused as of April 20, 2026.**
> Check github.com/features/copilot/plans for current status before planning around it.

| Plan | USD/month | Approx INR/month | Claude models available? |
|------|-----------|------------------|--------------------------|
| Free | $0 | ₹0 | Limited (Claude Sonnet, capped requests) |
| Pro | $10 | ~₹990 | ✅ Yes (Claude Sonnet via model picker) |
| Pro+ | $39 | ~₹3,860 | ✅ Yes (more models, more premium requests) |

> ⚠️ GitHub Copilot gives you Claude as a chat/autocomplete model inside VS Code. It is NOT the same as Claude Code. Claude Code is a separate CLI tool. The two can be used together but they serve different purposes.

---

### VS Code

**Cost: ₹0** — Free. No subscription needed.

---

### Minimum Viable Setup to Start Learning (INR/month)

| Scenario | Tools | Monthly Cost |
|----------|-------|-------------|
| Just studying courses, no practice | Browser only | ₹0 |
| Light practice, willing to hit limits | Claude Pro only | ~₹1,980 |
| Serious SDD on your project | Claude Pro + VS Code | ~₹1,980 |
| Serious SDD + Copilot in VS Code | Claude Pro + Copilot Pro | ~₹2,970 (if Copilot signups reopen) |
| Daily heavy Claude Code use | Claude Max 5x | ~₹9,912 |

---

---

## CLI REQUIREMENT — HONEST ANSWER

### Can you study the courses without CLI?

| Course | Without CLI? | Honest note |
|--------|-------------|-------------|
| Stage 0 (AI Foundations) | ✅ Yes | Browser-only, no tools needed |
| Stage 1 (Claude 101) | ✅ Yes | Browser-only |
| Stage 2 (Claude Code 101) | ⚠️ Watch only | You can watch the videos but cannot do the hands-on exercises without a terminal. Plan Mode, CLAUDE.md setup, and all practical exercises require CLI. |
| Stage 2 (Claude Code in Action) | ❌ No | Hooks, SDK, MCP integration — all terminal-based |
| Stage 3 (Skills, Subagents) | ❌ No | .claude/agents/, SKILL.md setup, skill invocation — all CLI-native |
| Stage 4 (MCP) | ❌ No | Running Python MCP servers requires terminal |
| Stage 5 (API Course) | ✅ Yes | Python scripts can be run in any environment |
| Stage 6 (Cloud) | ✅ Yes | Console-based |

**Honest bottom line:** You can watch all the courses without CLI. But you cannot practice or master Stages 2–4 — which are the entire SDD skill set — without a terminal. The courses are designed around CLI-first workflows.

**The good news:** You do not need a separate terminal application. VS Code has a built-in terminal (Ctrl+` on Windows/Linux, Cmd+` on Mac). Install Claude Code there. You never have to leave VS Code.

---

## VS CODE + GITHUB COPILOT + CLAUDE MODEL — HONEST ANSWER

### What actually works today in this setup:

| Capability | Works in VS Code + Copilot + Claude? | Notes |
|-----------|--------------------------------------|-------|
| Claude as chat assistant in IDE | ✅ Yes | Switch to Claude Sonnet in Copilot Chat model picker |
| Inline code suggestions | ✅ Yes | Standard Copilot autocomplete |
| Multi-file edits via agent mode | ✅ Yes | Copilot agent mode handles this |
| Spec-driven planning (write spec, feed to Claude) | ✅ Yes | Paste SPEC.md into Copilot Chat each session |
| @workspace context across entire project | ✅ Yes | Useful for reverse-speccing vibe-coded projects |
| CLAUDE.md persistent memory across sessions | ❌ No | Copilot has no equivalent. Context resets every session. |
| Skills (SKILL.md reusable workflows) | ❌ No | Claude Code-only feature |
| Subagents with isolated context windows | ❌ No | Claude Code-only feature |
| Hooks (automated formatting, command blocking) | ❌ No | Claude Code-only feature |
| Plan Mode (read-only spec planning) | ❌ No | Claude Code-only feature |
| MCP server connections | ❌ No | Claude Code-only feature |
| Claude Opus model access | ⚠️ Maybe | Depends on your Copilot plan and model availability |

### Honest verdict:
VS Code + Copilot + Claude Sonnet is a **real and valid workflow** for SDD, especially for getting started. You can write specs, feed them into Copilot Chat, and apply spec-driven changes across your codebase. It works.

What you lose is the **automation layer**: no persistent memory, no reusable skills, no isolated subagents, no hooks. Every session starts from scratch. You have to manually paste context each time.

The practical gap is not about capability — it is about **friction and repeatability**. Copilot + Claude can do SDD manually. Claude Code makes it automatic and systematic.

**The recommended approach:** Start with VS Code + Copilot + Claude to get comfortable with SDD concepts and your project's spec. When you find yourself repeating the same context pasting, the same instructions, the same workflows — that is the signal to install Claude Code in the VS Code terminal and start building skills and CLAUDE.md.

You do not have to choose one or the other. Most developers use both together.

---

## PAYMENT FRICTION IN INDIA — HONEST SUMMARY

| Issue | Reality |
|-------|---------|
| Payment method | International credit/debit card only. No UPI, no net banking. |
| GST | 18% OIDAR GST added at checkout. Non-negotiable. |
| INR pricing | Not available. Billed in USD with forex conversion. |
| Forex fees | Your bank/card will charge 2–3% foreign transaction fee on top. |
| True Pro cost | $20 USD + 18% GST + ~2-3% forex = approximately ₹2,050–₹2,100/month actual card charge |
| GitHub Copilot signups | Temporarily paused as of April 20, 2026. Check github.com/features/copilot/plans. |
| GST Input Tax Credit | If your company is GST-registered, you can claim ITC on Anthropic's OIDAR GST charges. |

---

## SUMMARY: What You Actually Need to Spend to Do This Properly

**Minimum to start (courses + light practice):**
- Courses: ₹0
- Claude Pro: ~₹2,050/month (with GST + forex)
- VS Code: ₹0
- **Total: ~₹2,050/month**

**Recommended for serious SDD on your project:**
- Claude Pro (start here, upgrade when you hit limits): ~₹2,050/month
- GitHub Copilot Pro (if/when signups reopen): ~₹990–₹1,000/month
- **Total: ~₹3,050/month**

**When you hit Pro limits regularly and Claude Code is your primary tool:**
- Claude Max 5x: ~₹9,912/month
- **Total: ~₹9,912/month**

---

*Last reviewed: April 25, 2026 | Source: anthropic.skilljar.com, claude.com/pricing, github.com/features/copilot/plans*
*Pricing is subject to change. Verify all costs at official sources before subscribing.*

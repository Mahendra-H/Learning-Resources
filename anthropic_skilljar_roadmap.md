# Anthropic Skilljar — Zero to Hero Roadmap

> **Certificates:** Every completed course awards a certificate of completion from Anthropic, shareable directly to your LinkedIn profile. The **Building with the Claude API** course includes a final exam — the closest thing to a formal Anthropic certification currently available.

---

## ⚠️ Honest Assessment Before You Start

This roadmap was originally written as a generic "zero to hero" path. After re-review, here is the honest version:

**If you are an experienced developer** (you already use VS Code, GitHub Copilot, understand terminals, and have a vibe-coded project you want to apply SDD to):
- **Skip Stage 0 entirely.** AI Capabilities & Limitations and AI Fluency are designed for non-technical users or people who have never used AI tools. You will find them too basic and waste your time.
- **Skim Claude 101.** It covers Claude.ai the chat product — useful for 30 minutes, not a deep investment.
- **Your real starting point is Claude Code 101.** Everything before it in the original list was padding.

**If you are a beginner** (new to AI, new to terminals, new to coding agents):
- Follow every stage in order. The foundational courses will genuinely help.

**The SDD-specific path for an experienced developer is just 7 courses, not 11.** The honest order is documented below.

---

## Stage 0 — AI Foundations
> ⚠️ **Honest comment: SKIP if you are an experienced developer.** These two courses are aimed at non-technical users or people completely new to AI. If you already use GitHub Copilot, Claude, or any AI tool regularly, you will find these too basic. They are included here only for completeness.

| Order | Course | What You Learn | Who It's For |
|-------|--------|----------------|--------------|
| 0a | [AI Capabilities and Limitations](https://anthropic.skilljar.com/ai-capabilities-and-limitations) | How AI works, what it can and can't do. Foundational mental model. | Complete beginners to AI only |
| 0b | [AI Fluency: Framework & Foundations](https://anthropic.skilljar.com/ai-fluency-framework-foundations) | How to collaborate with AI effectively, ethically, and safely. Prompting mindset, responsible use. | Complete beginners to AI only |

---

## Stage 1 — Claude Basics
> ⚠️ **Honest comment:** Claude 101 covers Claude.ai (the chat product), not Claude Code. It is worth doing but treat it as a 30–60 minute warm-up, not a serious time investment. If you are already using Claude chat regularly, you can skim or skip it.

| Order | Course | What You Learn | Focus |
|-------|--------|----------------|-------|
| 1 | [Claude 101](https://anthropic.skilljar.com/claude-101) | Core Claude features, everyday work tasks, prompting techniques, how to use Claude.ai effectively. Good baseline before jumping to Claude Code. **Prereq:** None | Basics |

---

## Stage 2 — Claude Code Core ← Your Real Starting Point
> ✅ **This is where SDD begins and where an experienced developer should actually start.** Claude Code 101 is the most important course on this entire platform for your use case. Do not skip it even if you have used Claude Code before — the Plan Mode, CLAUDE.md, and context management sections alone are worth it.

| Order | Course | What You Learn | Focus |
|-------|--------|----------------|-------|
| 2 | [Claude Code 101](https://anthropic.skilljar.com/claude-code-101) | Agentic loop, context window, tools & permissions. Install & setup in terminal, VS Code, JetBrains. Plan Mode (read-only spec planning before any code is written). Explore → Plan → Code → Commit workflow. CLAUDE.md for persistent project memory. Intro to subagents, skills, MCP, hooks. **Prereq:** Basic terminal + code editor | SDD Core ⭐ |
| 3 | [Claude Code in Action](https://anthropic.skilljar.com/claude-code-in-action) | Deeper dive: context management strategies, custom commands, MCP server integration, GitHub integration for automated PR reviews, hooks & SDK. Practical multi-step coding tasks and planning modes for complex existing codebases — directly relevant to reverse-speccing a vibe-coded project. **Prereq:** Claude Code 101 | SDD Core ⭐ |

---

## Stage 3 — Claude Code Advanced (SDD Power Features)
> ✅ **Skills and Subagents are the two features that separate basic Claude Code usage from a proper SDD workflow.** Skills let you encode repeatable patterns (like "always generate a spec before writing code"). Subagents let you delegate isolated tasks without polluting your main context. Do Skills before Subagents — the Skills course explicitly covers wiring skills into subagents, so the order matters.

| Order | Course | What You Learn | Focus |
|-------|--------|----------------|-------|
| 4 | [Introduction to Agent Skills](https://anthropic.skilljar.com/introduction-to-agent-skills) | Build reusable SKILL.md files, write trigger descriptions so Claude activates them automatically, organize multi-file skill directories with progressive disclosure. Share skills via plugins and enterprise settings. Wire skills into subagents. Troubleshoot trigger conflicts and priority issues. **Prereq:** Claude Code 101 | SDD Advanced ⭐ |
| 5 | [Introduction to Subagents](https://anthropic.skilljar.com/introduction-to-subagents) | How subagents manage isolated context windows. Create custom subagents for specific roles (code reviewer, documentation generator, test writer). Structured output formats, tool access limits, anti-patterns to avoid. When to delegate to a subagent vs stay in main context. **Prereq:** Claude Code 101 | SDD Advanced ⭐ |

---

## Stage 4 — MCP (Connect Claude to External Services)
> ✅ **Take this after you have a solid SDD workflow running.** MCP is what lets Claude Code connect to GitHub, databases, Figma, Slack, and anything else via a standardized protocol. The intro course is practical and well-structured.
>
> ⚠️ **Honest comment on Advanced MCP:** Unless you are building custom MCP servers for your team or product, the Advanced Topics course is optional. Most developers get everything they need from the intro course alone.

| Order | Course | What You Learn | Focus |
|-------|--------|----------------|-------|
| 6 | [Introduction to Model Context Protocol](https://anthropic.skilljar.com/introduction-to-model-context-protocol) | Build MCP servers & clients from scratch using Python SDK. Master tools, resources, and prompts — the 3 core MCP primitives. Use MCP Server Inspector for debugging. Connect Claude to external services without writing extensive integration code. **Prereq:** Python basics | MCP |
| 7 | [MCP: Advanced Topics](https://anthropic.skilljar.com/model-context-protocol-advanced-topics) | Sampling, notifications, file system access, transport mechanisms. Production-grade MCP server patterns. ⚠️ Optional unless building custom MCP servers. **Prereq:** Intro to MCP | MCP (Optional) |

---

## Stage 5 — API & Building with Claude
> ✅ **This is the certification milestone.** The most comprehensive single course on the platform. Has a final assessment — completing this is the strongest Anthropic credential currently available.
>
> ⚠️ **Honest comment:** This course requires Python proficiency. If you are primarily a JavaScript/TypeScript developer, the concepts still apply but the code examples are in Python. Do not skip it for that reason — the architectural patterns (RAG, tool use, agent workflows) are language-agnostic and directly relevant to SDD at scale.

| Order | Course | What You Learn | Focus |
|-------|--------|----------------|-------|
| 8 | [Building with the Claude API](https://anthropic.skilljar.com/claude-with-the-anthropic-api) | Full API: auth, multi-turn chat, system prompts, streaming, structured outputs, tool use, RAG, extended thinking, image/PDF processing, prompt caching, MCP clients, agent workflows (parallelization, chaining, routing). **Has final assessment.** **Prereq:** Python proficiency | API + 🏅 Cert Exam |

---

## Stage 6 — Cloud Specialization
> ⚠️ **Honest comment: Fully optional and only relevant if you deploy on AWS or GCP.** If you are building locally or on a neutral cloud, skip both. These are platform-specific courses — take the one that matches your stack, not both.

| Order | Course | What You Learn | Focus |
|-------|--------|----------------|-------|
| 9a | [Claude with Amazon Bedrock](https://anthropic.skilljar.com/claude-in-amazon-bedrock) | Deploy and use Claude via AWS Bedrock. Originally built as AWS employee accreditation training. Full course publicly available. **Prereq:** AWS background helpful | AWS (Optional) |
| 9b | [Claude with Google Vertex AI](https://anthropic.skilljar.com/claude-with-google-vertex) | Full spectrum of working with Anthropic models through Google Cloud's Vertex AI platform. **Prereq:** GCP background helpful | GCP (Optional) |

---

## Bonus — Cowork (Non-Code File Workflows)
> ⚠️ **Honest comment:** Cowork is not a developer tool — it is for working with documents, files, and research tasks using Claude. It will not advance your SDD skills. Take it only if your workflow involves non-code deliverables like reports or research documents.

| Course | What You Learn | Focus |
|--------|----------------|-------|
| [Introduction to Claude Cowork](https://anthropic.skilljar.com/introduction-to-claude-cowork) | Work alongside Claude on real files and projects (not code). Task loop, plugins, file & research workflows, steering multi-step work. | Productivity (Optional) |

---

## Recommended Order — Honest Quick Reference

### For experienced developers doing SDD on an existing project:
```
1.  Claude 101                     ← 30-60 min skim, not a deep investment
2.  Claude Code 101                ← YOUR REAL STARTING POINT ⭐
3.  Claude Code in Action          ← essential for existing vibe-coded projects ⭐
4.  Introduction to Agent Skills   ← SDD power feature ⭐
5.  Introduction to Subagents      ← SDD power feature ⭐
6.  Introduction to MCP            ← connect to GitHub, DBs, APIs
7.  MCP: Advanced Topics           ← only if building custom MCP servers
8.  Building with the Claude API   ← 🏅 certification milestone
9a. Claude with Amazon Bedrock     ← only if on AWS
9b. Claude with Google Vertex AI   ← only if on GCP
```

### For complete beginners (no prior AI or coding agent experience):
```
0a. AI Capabilities and Limitations
0b. AI Fluency: Framework & Foundations
1.  Claude 101
2.  Claude Code 101
3.  Claude Code in Action
4.  Introduction to Agent Skills
5.  Introduction to Subagents
6.  Introduction to MCP
7.  MCP: Advanced Topics
8.  Building with the Claude API   ← 🏅 certification milestone
9.  Cloud specialization (if relevant)
```

---

## Key Notes

- **All courses are free** on [anthropic.skilljar.com](https://anthropic.skilljar.com)
- **Certificates** are issued on completion and can be added directly to LinkedIn
- **SDD core path** for experienced developers is 4 courses: Claude Code 101 → Claude Code in Action → Agent Skills → Subagents
- **The API course** is the strongest credential — it has a final exam and covers the full Claude ecosystem
- **Cloud courses** are fully optional — only take the one that matches your deployment stack
- **Cowork** is not a developer or SDD tool — skip it unless your workflow includes non-code documents

---

*Last updated: April 2026 | Reviewed and corrected for accuracy | Source: [anthropic.skilljar.com](https://anthropic.skilljar.com)*

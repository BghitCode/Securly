---
name: securly
description: Apply security guidelines when designing, building, or reviewing AI chatbots, LLM-powered apps, autonomous/tool-using agents, and mobile apps (React Native/Expo, Flutter) — covering agent controls (authorization checks, plan validation, execution limits, audit logging), LLM/prompt security (prompt injection, RAG sanitization, output validation, PII/secret handling), OWASP web security (input validation, injection prevention, auth, headers, rate limiting), and mobile security (secure on-device storage, API key handling, certificate pinning, deep links, biometric auth). Use this any time the user writes system prompts, tool definitions, agent orchestration, RAG pipelines, chatbot backends, or mobile app auth/storage code, or asks for a security review or audit — even without saying "security" explicitly (e.g. "let the agent call other agents", "store the auth token in the app", "add a deep link to checkout").
---

# Securly

A working reference for building AI chatbots, LLM applications, autonomous agents, and mobile apps that don't fall over the first time someone pokes at them. It covers four layers that stack on top of each other:

1. **Agent-specific security** — controls that only apply once the model can *take actions* (call tools, chain steps, spawn sub-agents).
2. **LLM/AI security** — controls for any system where untrusted text reaches a model (prompt injection, output trust, RAG, data leakage).
3. **OWASP web security** — the baseline web app hygiene that still applies because chatbots and agents are ultimately web services with endpoints, sessions, and databases behind them.
4. **Mobile app security** — controls specific to code running on a device the user (or an attacker) fully controls: secure storage, API key handling, transport security, deep links, biometrics.

Almost every real AI feature touches at least two or three of these layers. A support chatbot with RAG needs layer 2 and 3. An agent that can browse the web and run code needs 1 through 3. A mobile app with an embedded AI chat feature needs all four.

## Why this exists as a skill rather than "just be careful"

The failure mode with AI security isn't usually ignorance of any single rule — it's dropping rules under the pressure of getting a demo working. A tool call gets wired up without an auth check "for now." A retrieved document gets pasted into the prompt without sanitization "just to test it." Those become the shipped behavior. This skill exists to make the checks a default part of the build, not an afterthought bolted on before a pentest.

## How to use this skill

**When generating code or design for a new AI feature** (chatbot backend, tool/function definitions, agent loop, RAG pipeline, system prompt, plugin integration): work through the relevant reference file(s) below *while* writing the code, not after. Treat each applicable guideline as a design constraint, the same way you'd treat a type system — bake it in rather than patch it in.

**When reviewing or auditing existing code**: use `references/audit-checklist.md` as the pass/fail rubric. Go file by file — don't just skim and declare it fine. Call out concrete line numbers or code paths for anything that fails a check.

**When the user describes a feature in plain language**, map it to the layers before writing anything:

| The user is building... | Layers that apply |
|---|---|
| A chatbot that answers questions from a knowledge base (RAG) | LLM security (esp. RAG sanitization, prompt injection, data leakage) + OWASP (the API serving it) |
| An agent that can call tools/APIs on the user's behalf | Agent security + LLM security + OWASP |
| An agent that can run code, access the filesystem, or hit arbitrary URLs | All three, with extra weight on sandboxing, allowlists, and execution limits |
| A multi-agent system (orchestrator + sub-agents) | Agent security (recursive spawning, budgets, plan validation) + LLM security |
| A plain web form/endpoint with no LLM in the loop | OWASP only |
| Anything ingesting user-uploaded files | OWASP (upload restrictions) + LLM security (sanitize before it reaches the prompt) |
| A React Native/Expo or Flutter app (no AI in it) | Mobile security + OWASP (for the backend API it calls) |
| A mobile app with an embedded AI chatbot or agent | Mobile security + LLM security + Agent security (if it calls tools) + OWASP (for the backend) |

Don't apply every single guideline to every project indiscriminately — that produces bloated, unreadable code and checklist theater. Instead, identify which attack surface actually exists (does this system have tools? does it touch other users' data? does it take file uploads?) and apply the controls that address that surface. Say explicitly which controls you're skipping and why if a project genuinely doesn't need them (e.g. a single-user local CLI tool doesn't need per-user data isolation).

## Reference files

- **`references/agent-security.md`** — Controls specific to systems where the model can take actions: tool authorization, plan validation, execution/recursion limits, human-in-the-loop gates, audit logging. Read this whenever tools, function calling, or multi-step agent loops are involved.
- **`references/llm-security.md`** — Controls for any LLM-in-the-loop system: prompt injection defense, role separation, output validation, RAG sanitization, data isolation between users, PII/secret handling, rate limiting, content moderation. Read this for essentially any AI feature.
- **`references/owasp-web-security.md`** — Standard web application security baseline (input validation, injection prevention, auth/session security, headers, CSRF, rate limiting). Read this whenever there's an HTTP endpoint involved, which is almost always.
- **`references/mobile-security.md`** — Controls specific to React Native/Expo and Flutter apps: secure on-device storage, API key handling, transport security and certificate pinning, deep link validation, biometric auth, root/jailbreak detection. Read this whenever the project is a mobile app or a mobile client calling an AI backend.
- **`references/audit-checklist.md`** — A consolidated, skimmable checklist across all layers, meant for reviewing code someone already wrote rather than designing new code. Use this for "review this for security issues" style requests.

## A note on threat modeling before checklists

Checklists are only useful once you know what you're actually defending against. Before diving into the reference files, form a quick mental model:

- **What can reach the model that the user doesn't fully control?** (retrieved documents, web pages, emails, other users' content, tool outputs) — these are injection vectors.
- **What can the model cause to happen in the real world?** (send an email, run a query, spend money, call an API) — these need authorization + confirmation + limits.
- **What data does this system hold that shouldn't cross between users or leave the system?** — this drives isolation and PII handling.
- **What's the blast radius if the model is fully compromised by a malicious prompt?** — this drives sandboxing and least privilege, since you should assume it *will* happen eventually rather than hoping it won't.

Answering these four questions usually tells you which subset of the checklists actually matters for the system in front of you.

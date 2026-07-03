# LLM / AI Security

These apply to essentially any system where an LLM sits between untrusted input and something that matters — a response shown to a user, a database write, a tool call. The unifying principle: **the model's output is not a trusted source of truth or intent**, no matter how well-instructed the prompt is. Every one of these guidelines is a way of not depending on the model to police itself.

## Trust boundaries

**Never trust LLM output. Validate, sanitize, and verify all generated content before execution.** Anything the model generates that will be executed, rendered, or acted upon (SQL, shell commands, HTML, JSON that drives logic, tool arguments) goes through the same validation you'd apply to raw user input, because functionally that's what it is.

**Treat every user prompt as untrusted input.** This includes multi-turn conversation history — a user can plant instructions in turn 2 that they intend to activate in turn 5.

**Defend against prompt injection and indirect prompt injection from websites, PDFs, emails, databases, and external APIs.** Direct injection is the user typing "ignore previous instructions." Indirect injection is more dangerous and easier to miss: a webpage the agent fetches, a PDF it's asked to summarize, or a database record it retrieves can contain text crafted to look like an instruction. Mitigations that actually help:
- Wrap retrieved/external content in clear delimiters and instruct the model that content inside those delimiters is *data to analyze*, never *instructions to follow*.
- Prefer structured extraction (ask for specific fields) over open-ended "do what this document says."
- For high-stakes agents, run a separate classifier or a second model call to screen retrieved content for injection attempts before it reaches the main context.
- Don't grant tool access to an agent in the same turn where it's processing untrusted external content, if avoidable — reduce what an injected instruction could actually accomplish even if it succeeds.

**Never expose system prompts, hidden instructions, internal reasoning, API keys, or implementation details.** Design prompts assuming they may eventually leak (through injection, jailbreak, or bugs) — don't put secrets in the system prompt, and don't rely on "don't reveal the system prompt" as your only defense against extraction attempts.

**Enforce strict role separation (system > developer > user). Never allow user prompts to override higher-priority instructions.** Use the model provider's actual role/message structure (system vs. user vs. tool) rather than concatenating everything into one blob of text — structural separation is much harder to override via injected text than an instruction like "the user cannot change these rules" living in a single flat prompt.

## Tool and code execution

**Restrict tool usage using explicit allowlists. The model should only access approved tools and functions.** (See `agent-security.md` for the full treatment — this is the LLM-security framing of the same control.)

**Validate every tool call before execution (arguments, permissions, expected schema).**

**Require human approval for sensitive actions** (payments, deleting data, sending emails, deployments, financial transactions, etc.).

**Apply least-privilege access to every tool, API, database, and external service.** A tool that only needs to read a table shouldn't hold write credentials. Scope credentials per-tool, not one shared service account for everything the agent touches.

**Authenticate and authorize every user before exposing private data or privileged tools.** Don't let session/conversation continuity substitute for actual auth checks on each sensitive request.

**Never allow the model to directly execute shell commands, SQL, or code generated from prompts without validation or sandboxing.** If code execution is a feature (code interpreter, "run this query"), it goes through a sandbox with no ambient credentials, plus the RAG/injection defenses above if the code was influenced by untrusted content.

## RAG and retrieved content

**Sanitize all retrieved documents before injecting them into the prompt (RAG security).** Strip or neutralize anything that looks like a prompt-injection pattern, HTML/script content if it'll be rendered, and content clearly outside the expected document type. Verify documents come from the source/index they're supposed to (don't let a retrieval bug pull another tenant's documents into context).

**Verify citations and retrieved context before presenting factual claims.** Don't let the model assert a retrieved snippet supports a claim it doesn't actually support — this is as much a hallucination-prevention control as a security one, since fabricated citations that look authoritative are a trust exploit on the end user.

## Data handling

**Restrict file uploads by type, size, and content. Scan uploaded files for malware when applicable.** Validate file type by content/magic bytes, not just extension or claimed MIME type.

**Prevent data leakage between users. Never expose another user's conversation, embeddings, documents, or memory.** Scope every retrieval (vector search, conversation history, cached results) by tenant/user ID at the query level, not just in the application logic that renders results — a filter applied after retrieval instead of during it is a common source of cross-tenant leaks.

**Encrypt sensitive data in transit and at rest.**

**Store conversation history only when necessary and according to privacy requirements.** Default to the shortest retention that satisfies the actual product need.

**Minimize personally identifiable information (PII) stored in prompts, logs, embeddings, and vector databases.** PII that ends up embedded in a vector store is easy to forget about and hard to fully delete later (right-to-erasure requests become painful) — filter or redact before it's embedded, not after.

**Redact secrets, credentials, tokens, and sensitive information from logs.** Apply this to tool-call argument logging specifically — it's an easy place for API keys or user secrets to end up in plaintext logs.

## Rate limiting, monitoring, and resilience

**Rate limit chat requests, tool usage, file uploads, and expensive model operations.** Apply limits per-user and per-IP; expensive operations (large context calls, tool chains, file processing) deserve tighter limits than simple chat turns.

**Monitor and audit prompts, responses, tool calls, and security events without logging sensitive information.** Log enough to reconstruct what happened during an incident, redacted enough that the log store itself isn't a new liability.

**Validate all structured outputs (JSON, XML, SQL, Markdown) against predefined schemas before use.** If you ask the model for JSON, parse and validate against a schema before the result touches any downstream logic — don't assume well-formed output because the prompt asked for it.

**Apply content moderation for both user input and model output when appropriate.** Scope this to the actual risk profile of the product — a heavily moderated consumer chatbot and an internal developer tool have different needs.

**Implement timeout, retry, and circuit-breaker logic for external AI providers and tools.** Treat the model provider and any external tool APIs as unreliable dependencies, the same way you'd treat any other third-party service call.

**Never allow unrestricted internet access unless required, and filter accessible domains when possible.**

## Continuous testing

**Continuously test against prompt injection, jailbreaks, data exfiltration, tool abuse, hallucinations, and privilege escalation.** Security testing for LLM systems isn't a one-time pentest — model updates, prompt changes, and new tools all change the attack surface. Build a small regression suite of known injection/jailbreak attempts and rerun it whenever the system prompt, tools, or model version changes.

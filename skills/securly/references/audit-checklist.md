# Consolidated Audit Checklist

Use this for "review this code/feature for security issues" requests. Go through applicable sections based on what the system actually does (see the mapping table in `SKILL.md`). For each item, cite the specific file/line/function that passes or fails it — a checklist without evidence is just a list of hopes.

## Agent / tool-use layer
*(only if the system has tools, function calling, or multi-step agent loops)*

- [ ] Every tool execution path checks authorization at call time, not just at registration
- [ ] Tool list exposed to the model is scoped per session/role, not global
- [ ] No tool accepts a raw, model-supplied URL/endpoint without an allowlist check
- [ ] Tool call arguments are schema-validated before execution
- [ ] Multi-step chains validate each intermediate result before feeding it to the next step
- [ ] Any generated multi-step plan is validated against constraints before execution begins
- [ ] Filesystem tools are sandboxed to an approved directory with path-traversal checks
- [ ] Network-calling tools are restricted to a domain allowlist (or block internal/metadata IP ranges)
- [ ] Production-modifying actions require explicit human confirmation
- [ ] Sub-agent spawning has an explicit depth/count cap; recursive spawning isn't allowed by default
- [ ] There's a hard max step count, wall-clock timeout, and token/cost budget per run
- [ ] Degenerate loop patterns (repeated identical calls) are detected and stopped
- [ ] Low-confidence situations cause the agent to ask/decline rather than guess
- [ ] Irreversible/high-impact actions require confirmation regardless of model confidence
- [ ] Every tool call is written to an audit log (actor, action, args, authorized?, result)
- [ ] Code execution / plugins run in an isolated sandbox with no ambient credentials

## LLM / prompt layer
*(applies to essentially any AI feature)*

- [ ] Model-generated content that gets executed/rendered/acted on is validated like user input
- [ ] System prompt uses the provider's actual role structure, not string concatenation
- [ ] System prompt doesn't contain secrets, assuming it may eventually leak
- [ ] Content from external sources (web pages, PDFs, emails, DB records) is clearly delimited from instructions when injected into context
- [ ] Retrieved documents are sanitized and scoped to the correct tenant/index before entering the prompt
- [ ] Citations/retrieved claims are checked against the source before being presented as fact
- [ ] File uploads are validated by content/magic bytes (not just extension), size-limited, and scanned if applicable
- [ ] Retrieval, history, and memory lookups are scoped by user/tenant *at the query level*
- [ ] PII is minimized before it's written into prompts, logs, or embeddings
- [ ] Logs redact secrets/tokens/credentials, including in tool-call argument logs
- [ ] Rate limits exist per-user (and per-IP) for chat, tool use, and file uploads specifically
- [ ] Structured model outputs (JSON/SQL/etc.) are schema-validated before use, not just parsed
- [ ] External AI provider / tool calls have timeout + retry + circuit-breaker logic
- [ ] There's a regression suite of known injection/jailbreak prompts run against prompt/tool/model changes

## Web application layer
*(applies to the HTTP surface, which is nearly always present)*

- [ ] All user input is validated server-side with allowlists (not just client-side)
- [ ] No SQL is built via string concatenation/interpolation — parameterized queries or ORM only
- [ ] Output is context-aware encoded (HTML/attribute/URL/JS) to prevent XSS
- [ ] No hardcoded secrets/API keys in source; secrets come from environment/secret manager
- [ ] HTTPS is enforced; cookies are HttpOnly, Secure, SameSite=Strict
- [ ] State-changing requests are protected against CSRF
- [ ] Security headers are set: CSP, X-Frame-Options, Referrer-Policy
- [ ] Public endpoints have per-IP and per-user rate limiting

## Severity framing

When reporting findings, distinguish:
- **Critical** — exploitable now, no auth required, meaningful impact (e.g. unauthenticated tool call, SQL built by string concat, missing tenant isolation on retrieval)
- **High** — exploitable with some precondition (authenticated user, specific input) or high-impact but harder to trigger
- **Medium** — defense-in-depth gap that isn't independently exploitable but removes a layer (e.g. missing CSP alongside otherwise-correct output encoding)
- **Note** — good practice not yet applied, low risk in this specific context

Don't flatten everything to "issue found" — the user needs to know what to fix first.

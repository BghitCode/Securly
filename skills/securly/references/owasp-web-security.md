# OWASP Web Application Security Baseline

Chatbots and agents are ultimately served over HTTP by a normal web backend, so all of the standard web application security fundamentals still apply underneath the AI-specific layers. None of this is AI-specific — it's the baseline every endpoint in the system needs regardless of whether an LLM is involved.

## Input handling

**Sanitize and validate all user input (forms, query params, headers) using allowlists.** Prefer allowlisting valid characters/formats over blocklisting known-bad patterns — blocklists are perpetually incomplete.

**Validate inputs on both frontend and backend (frontend ≠ security).** Frontend validation is UX. Backend validation is the actual control — never assume a request reached your API through the frontend you wrote.

## Injection prevention

**Use parameterized queries / ORM only — never build SQL with strings to prevent SQL injection.**

```javascript
// Never:
db.query(`SELECT * FROM users WHERE id = ${userId}`);

// Always:
db.query('SELECT * FROM users WHERE id = $1', [userId]);
```

**Encode output properly to prevent XSS (HTML, attributes, URLs, JS contexts).** Use context-aware encoding — the correct escaping for inserting into an HTML attribute is different from inserting into a `<script>` block or a URL. Prefer frameworks that auto-escape by default (React, most modern template engines) and treat any `dangerouslySetInnerHTML`-style escape hatch as something that needs its own review, especially if the content being rendered ever came from an LLM response or retrieved document.

## Secrets and transport

**Never hardcode secrets or API keys — use environment variables.** Also make sure they don't end up in client-side bundles, git history, or logs (see the log-redaction point in `llm-security.md`).

**Enforce HTTPS, secure cookies (HttpOnly, Secure, SameSite=Strict).**

## Session and request integrity

**Protect state-changing requests with CSRF tokens or SameSite cookies.** Any request that changes state (not just form POSTs — this includes API calls triggered by an agent tool acting "on behalf of" a logged-in user) needs this protection.

**Set security headers: CSP, X-Frame-Options: DENY, Referrer-Policy.** A reasonably strict Content-Security-Policy is also a meaningful mitigation against XSS payloads that make it through despite output encoding — treat it as defense in depth, not a replacement for encoding.

## Availability and abuse

**Implement rate limiting per IP and per user on all public endpoints.** For AI endpoints specifically, layer this with the AI-specific rate limiting in `llm-security.md` (per-user chat/tool/upload limits) — a generic per-IP limiter alone won't stop an authenticated user from hammering the LLM endpoint with expensive requests.

---

## Quick pattern reference

| Risk | Control |
|---|---|
| SQL injection | Parameterized queries / ORM, never string-built SQL |
| XSS | Context-aware output encoding + CSP + framework auto-escaping |
| CSRF | SameSite cookies + CSRF tokens on state-changing requests |
| Credential leakage | Env vars, never hardcoded; redact from logs and client bundles |
| Session hijacking | HttpOnly + Secure + SameSite cookies, HTTPS everywhere |
| Clickjacking | X-Frame-Options: DENY / frame-ancestors in CSP |
| Brute force / abuse | Per-IP and per-user rate limiting on public endpoints |
| Trusting client validation | Always re-validate server-side |

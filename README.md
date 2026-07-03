# Securly

An [Agent Skill](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) that equips Claude (and other Agent Skills-compatible agents) with security guidelines for building and reviewing AI chatbots, LLM applications, and autonomous agents.

Covers three layers:

- **Agent-specific security** — tool authorization, plan validation, execution/recursion limits, human-in-the-loop gates, audit logging
- **LLM / prompt security** — prompt injection defense, role separation, RAG sanitization, output validation, data isolation, PII handling
- **OWASP web security baseline** — input validation, injection prevention, auth/session security, headers, CSRF, rate limiting

## Install

Via [skills.sh](https://skills.sh):

```bash
npx skills add <your-github-username>/securly
```

Or manually: copy `skills/securly/` into your agent's skills directory (e.g. `.claude/skills/`, `.agents/skills/`).

## Structure

```
skills/securly/
├── SKILL.md                        # entry point: when to apply which layer
└── references/
    ├── agent-security.md           # tool auth, execution limits, audit logging
    ├── llm-security.md             # prompt injection, RAG, data isolation
    ├── owasp-web-security.md       # web app security baseline
    └── audit-checklist.md          # consolidated review checklist
```

## License

MIT

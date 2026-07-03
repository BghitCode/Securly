# Agent-Specific Security

These controls apply once an LLM can *do* things — call tools, chain steps, spawn other agents — rather than just produce text. The core risk model: the model's reasoning is probabilistic and steerable by whatever text it has seen (including injected text), so anything the agent can trigger in the real world needs controls that don't rely on the model "deciding correctly."

## Tool invocation and authorization

**Every tool invocation must include an authorization check.** Don't let a tool execute just because the model called it with well-formed arguments. Check that the *current user/session* is actually permitted to perform that action, at the point of execution — not just at the point where the tool was registered as "available."

```python
def execute_tool(tool_name, args, session):
    tool = TOOL_REGISTRY[tool_name]  # raises if not allowlisted
    if not tool.is_authorized(session.user, args):
        raise AuthorizationError(f"{session.user} not authorized for {tool_name}")
    validate_schema(tool.input_schema, args)
    return tool.run(args, context=session)
```

**Restrict tool usage using explicit allowlists.** The model should only ever see and call tools it's actually permitted to use in a given context — don't expose an admin tool to the model just because it's technically wired up in the codebase. Scope the tool list per session/role, not globally.

**Do not let the model choose arbitrary APIs or endpoints.** If a tool wraps HTTP calls, the URL/endpoint should come from a fixed, code-defined set — never let the model supply a raw URL that gets fetched or POSTed to. If dynamic targets are genuinely required (e.g. a "visit this URL" tool), validate against a domain allowlist before the request goes out.

**Validate every tool call before execution** — arguments, permissions, and expected schema. Treat model-generated tool call arguments the same way you'd treat user input from an HTTP request: untrusted until validated against a schema.

## Chaining and planning

**Never chain tool calls automatically without validating each intermediate result.** A multi-step plan (search → fetch → summarize → post) should check the result of each step before feeding it into the next, both for correctness (did the tool actually return what's expected) and for safety (does this result contain something that looks like an injected instruction). Don't let one tool's raw output become the next tool's arguments without passing through validation.

**Validate every generated plan before execution.** If the agent produces a multi-step plan up front, check it against constraints (allowed tools, allowed targets, expected step count, no forbidden action sequences) before running any of it — not just after each step fails or succeeds.

## Filesystem, network, and system boundaries

**Restrict filesystem access to approved directories.** Sandbox any filesystem tool to an explicit allowlist of paths (a workspace directory, not the whole filesystem). Resolve paths and reject anything that escapes the sandbox via `..` traversal, symlinks, or absolute paths pointing elsewhere.

```python
def safe_path(base_dir, requested_path):
    resolved = (base_dir / requested_path).resolve()
    if not resolved.is_relative_to(base_dir.resolve()):
        raise SecurityError("path escapes sandbox")
    return resolved
```

**Restrict network access to approved domains.** Same idea for any tool that makes outbound requests — maintain a domain allowlist (or at minimum a blocklist of internal/metadata IP ranges like `169.254.169.254`, `localhost`, RFC1918 ranges) to prevent SSRF via agent-initiated requests.

**Require confirmation before modifying production systems.** Anything that writes to a production database, deploys code, or changes infrastructure state should pause for explicit human approval rather than executing autonomously, regardless of how confident the plan looked.

## Runaway behavior

**Prevent recursive agent spawning unless explicitly allowed.** If an orchestrator agent can spawn sub-agents, cap the spawn depth and total agent count explicitly, and default to *not* allowing a sub-agent to itself spawn more agents unless that's a deliberate design choice.

**Set maximum task duration, execution steps, and API budget.** Every agent run needs hard ceilings: wall-clock timeout, max tool calls / iterations, and a token or dollar budget. Fail the run (gracefully) when a limit is hit rather than letting it run indefinitely.

```python
class AgentRunLimits:
    max_steps = 25
    max_duration_seconds = 300
    max_tokens = 200_000
    max_cost_usd = 2.00
```

**Define clear stopping conditions to prevent infinite reasoning or tool loops.** Beyond raw limits, detect degenerate patterns explicitly — the same tool being called with the same arguments repeatedly, or the plan oscillating between two states — and stop the run when detected instead of waiting for the hard ceiling.

## Failure and observability

**Fail safely when confidence is low instead of guessing.** If the model isn't confident about which tool to call, what arguments to use, or whether an action is safe, the agent should surface that uncertainty (ask the user, or decline the action) rather than picking something plausible-looking and running with it. This matters most for irreversible actions.

**Require explicit confirmation before irreversible or high-impact actions.** Deleting data, sending money, sending communications on someone's behalf, or anything that can't be trivially undone should always have a human-in-the-loop gate, independent of how the agent's internal confidence looks.

**Log every action with an audit trail.** Record, at minimum: which tool was called, with what arguments, by which session/user, what the result was, and whether it passed authorization. This is what makes post-incident review possible and is also what you'll want when debugging why an agent did something unexpected — treat it as required infrastructure, not a nice-to-have.

```python
audit_log.record(
    actor=session.user,
    action=tool_name,
    args=redact_secrets(args),
    authorized=True,
    result_summary=summarize(result),
    timestamp=now(),
)
```

## Isolation

**Isolate execution environments for code interpreters, plugins, and autonomous agents.** Anything that executes model-generated code should run in a sandboxed container/VM with no access to secrets, the host filesystem, or the internal network by default — not in-process with the rest of the application.

**Limit autonomous agents with maximum execution depth, iteration count, token budget, and runtime.** (Restating the budget point above because it's the single most commonly skipped control when a demo is "basically working" — don't ship without it.)

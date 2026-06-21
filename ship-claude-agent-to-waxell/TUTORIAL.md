# Ship a Claude Agent to Your Team

You built a great Claude agent locally — a system prompt, a few tools, maybe a
`.claude/agents/*.md`. It works on your laptop. But your teammates can't use it,
there's no governance, no shared traces, and no way to run it *as* a specific
end-user with *their* own keys.

This tutorial publishes that local agent to [Waxell](https://waxell.ai) as a
declarative `claude_agent_sdk` agent. When you're done it runs on the governed
isolation-runner, lives at `https://<your-tenant>.waxell.dev/agents/<name>`, and
can resolve **per-end-user** credentials host-side — so the model never sees a
raw secret.

> **Let Claude do it for you.** Install this skill (see the repo README), then
> open Claude Code in your agent's repo and say *"deploy this agent to Waxell."*
> The skill drives the whole tutorial.

## Prerequisites

```bash
pip install waxell-runtime         # ships the `wax` CLI
wax login                          # or configure a profile
wax whoami                         # ALWAYS verify tenant + identity first
```

Pushing to prod uses a **profile** pointing at your prod tenant, e.g.
`wax -p acme_prod …`. Verify with `wax whoami -p acme_prod`. Don't assume the
default profile is prod.

## 1. Describe the agent in `waxell.yaml`

A `claude_agent_sdk` agent on Waxell is declarative. Translate your local agent
into a `waxell.yaml` at your project root:

```yaml
version: 1

defaults:
  framework: claude_agent_sdk
  owner: you@acme.com

agents:
  - name: release_notes
    version: 1.0.0                 # bump on every re-push
    display_name: Release Notes Writer
    description: Drafts release notes from a changelog.
    model: claude-3-5-sonnet
    tools: [Read, Glob, Grep, WebFetch]
    system_prompt: |
      You write crisp, user-facing release notes from a raw changelog.
      Group changes by theme, lead with user impact, skip internal churn.
    signals:
      - name: write_release_notes
        source_type: webhook
        description: Draft release notes for a version.
        schema:
          prompt:
            type: string
            description: The changelog + version to summarise
    max_turns: 12
    max_budget_usd: 4.0
```

- **`tools`** are the Claude Code built-ins (`Read`, `Write`, `Edit`, `Bash`,
  `Glob`, `Grep`, `WebSearch`, `WebFetch`, …). List only what's needed.
- **`signals`** are how the agent is triggered; the `schema` is its input.
- Paste your local **system prompt** in directly.

## 2. Validate and publish

```bash
wax -p acme_prod push --dry-run          # validate the manifest
wax -p acme_prod push --include-source   # publish (source travels with it)
wax -p acme_prod agents info release_notes
```

The agent is live at `https://acme.waxell.dev/agents/release_notes`. Your team
can see its spec, bindings, traces, cost, and governance.

## 3. Run it

```bash
wax -p acme_prod signals send write_release_notes \
    --data '{"prompt":"v2.1: added SSO, fixed 3 billing bugs..."}'
wax -p acme_prod runs list --agent release_notes
wax -p acme_prod runs show <run_id>
```

The run's **Setup** span shows exactly what the agent was provisioned with —
allowed tools, domains, identity. That's your first stop when debugging.

## 4. Make it per-user (optional)

The payoff of running on Waxell instead of a laptop: the same agent can run **as
a specific end-user**, reasoning on *their* LLM key and calling tools with
*their* tokens — all resolved host-side.

**a) Register the end-user** (maps your app's user id to a Waxell sub-user):

```bash
wax -p acme_prod end-users create --tenant-sub-user-id alice \
    --email alice@acme.com --display-name "Alice"
```

**b) Set that user's secrets** (their own Anthropic key, a tool token). These
are write-only and encrypted at rest — you set/rotate but never read them back:

```bash
curl -s -X POST "https://api.waxell.dev/api/waxell/v1/sub-users/alice/secrets/" \
  -H "X-Wax-Key: $WAX_API_KEY" -H "Content-Type: application/json" \
  -d '{"key":"ANTHROPIC_API_KEY","value":"sk-ant-…"}'
```

**c) Run as Alice** — your backend stamps a signed `_sub_user_identity` on the
signal (stamp **both** `sub_user_id` and `user_id`):

```json
POST /api/v1/signals/write_release_notes/
{
  "prompt": "...",
  "_sub_user_identity": {"sub_user_id": "alice", "user_id": "alice", "email": "alice@acme.com"}
}
```

The run now reasons on Alice's key and uses Alice's tokens. Resolution
precedence is **SubUser → Tool → Agent → Tenant** — a user's own credential
overrides every shared default, and one user can never resolve another's.

## Gotchas

- **Bump `version:` on every re-push** — re-pushing the same version with changed
  content fails with `semver_conflict`.
- **Don't hardcode tool names** in the prompt; the platform may expose a tool as
  `mcp__…__<name>`. Say "use the available GitHub tool."
- **`wax whoami` first, always** — the #1 cause of "it deployed to the wrong
  place" is the wrong profile.

## Links

- [Waxell docs](https://waxell.ai/docs/)
- [waxell-runtime on PyPI](https://pypi.org/project/waxell-runtime/)

---
name: ship-claude-agent-to-waxell
description: Ship a local claude_agent_sdk / Claude Code agent to Waxell so a whole team can use it in prod — governed, observable, and (optionally) per-end-user. Activate when the user has a Claude agent working locally (a system prompt + tools, a `.claude/agents/*.md`, or an Anthropic Agent SDK script) and wants to deploy / publish / share it, run it for their teammates, or wire per-user credentials (BYO API keys, connected tokens). Not for authoring Python `@agent`/`@workflow` code — that's the waxell-runtime skill.
---

# Ship a Claude agent to Waxell (local → shared prod)

The problem this solves: a dev builds a great `claude_agent_sdk` agent locally, but it lives on their laptop — teammates can't use it, there's no governance, no per-user auth, no shared traces. This skill takes that local agent and publishes it to Waxell as a **declarative `claude_agent_sdk` agent** that runs on the governed isolation-runner, shows up at `https://<tenant>.waxell.dev/agents/<name>`, and can run **as a specific end-user** with that user's own keys/tokens — resolved host-side so the model never sees them.

You (Claude) drive this end to end using the `wax` CLI. Don't hand-build infra; orchestrate the commands below and write the `waxell.yaml`.

## 0. Preconditions (always check first)

```bash
wax whoami                 # confirms api_url + tenant + identity. ALWAYS run first.
wax agents list            # confirms the api key has access
```

- If `wax` isn't authenticated: `wax login` (or set a profile — see Profiles).
- **Profiles matter.** Pushing to prod uses a profile that points at the prod tenant, e.g. `wax -p acme_prod push`. Never assume the default profile is prod. `wax whoami -p acme_prod` to verify. The `--local` flag keeps the *default* profile and is almost never what you want for sharing.

## 1. Scaffold a `waxell.yaml` from the local agent

A `claude_agent_sdk` agent on Waxell is **declarative YAML**, not Python. Map the user's local agent (its system prompt, model, allowed tools) into this shape. Write it to `<project>/waxell.yaml`:

```yaml
version: 1

defaults:
  framework: claude_agent_sdk
  owner: <user-email>

agents:
  - name: my_agent                 # url-safe; becomes /agents/my_agent
    version: 1.0.0                  # BUMP on every re-push (see gotchas)
    display_name: My Agent
    description: One line on what it does.
    model: claude-3-5-sonnet        # or another supported model
    tools: [Read, Glob, Grep, WebSearch, WebFetch]   # framework built-ins
    # domains: [github]             # OPTIONAL — per-user tool integrations (§4)
    system_prompt: |
      <the user's local system prompt, verbatim or lightly adapted>
    signals:
      - name: my_agent_request      # how the agent is triggered
        source_type: webhook
        description: Handle a request.
        schema:
          prompt:
            type: string
            description: The user's request
    max_turns: 15
    max_budget_usd: 5.0
```

Notes when scaffolding:
- **tools** are the Claude Code built-ins (`Read, Write, Edit, Bash, Glob, Grep, WebSearch, WebFetch, …`). List only what the agent actually needs. Don't list MCP/registry tools here unless they're registered in Waxell.
- The **signal** is the trigger. Single-purpose agents have one signal; its `schema` is the input.
- Keep the system prompt **tool-name-agnostic** ("use the available GitHub tool"), not hardcoded to a specific tool name — the platform may expose tools under prefixed names.

## 2. Validate, then push

```bash
wax -p <profile> push --dry-run          # validate the manifest, no upload
wax -p <profile> push --include-source   # publish. --include-source makes the
                                         # source travel with the artifact
                                         # (otherwise the Source tab is empty).
wax -p <profile> agents info my_agent    # confirm the registered spec
```

The agent is now live at `https://<tenant>.waxell.dev/agents/my_agent`. Teammates can see it, its bindings, traces, and governance.

## 3. Run it + verify

```bash
# Fire the signal (this is exactly how prod triggers it):
wax -p <profile> signals send my_agent_request --data '{"prompt":"..."}'
wax -p <profile> runs list --agent my_agent          # find the run
wax -p <profile> runs show <run_id>                  # inspect output + spans
```

Or via the API (the tenant's own backend would do this):
`POST https://api.waxell.dev/api/v1/signals/my_agent_request/` with the `X-Wax-Key`.

If the run errors, `wax runs show <id>` and the `Setup` span tell you what was provisioned (allowed_tools, domains, identity). `wax doctor` repairs common breakage.

## 4. (Optional) Per-end-user tool integrations — "connect your X"

If the agent should call a third-party API **as the end-user** (their GitHub, their CRM) using *their* token, use a **domain** with per-user auth. The token is resolved host-side at the tool boundary — the model never sees it.

1. Register/point a domain at the backend that performs the call, with per-user bearer auth (key = the end-user's `SubUserSecret`):
   ```bash
   wax -p <profile> domains list
   wax -p <profile> domains show github
   wax -p <profile> domains enable-per-user-auth github
   ```
   (Registering a *new* domain with `auth_type=sub_user_bearer` + actions is not yet a first-class `wax domains create`; provision it via the tenant-domains API or ask the platform team. Track: `wax domains create` is the gap.)
2. Declare it on the agent: add `domains: [github]` to the `waxell.yaml`, re-push (bump version). The executor surfaces each action as a `github__<action>` tool.

See `areas/runtime/plans/SUB_USER_SECRETS_PLAN.md` for the full per-end-user credential model.

## 5. (Optional) Per-end-user auth mapping — BYO keys / tokens

To run the agent **as a specific teammate's end-user**, with that end-user's own credentials:

1. **Register the end-user** (maps the tenant's identifier → a Waxell sub-user):
   ```bash
   wax -p <profile> end-users create --tenant-sub-user-id alice \
       --email alice@acme.com --display-name "Alice"
   wax -p <profile> end-users lookup alice
   ```
2. **Set that end-user's secrets** (their BYO LLM key, their tool token) — one
   command each. Write-only + encrypted at rest; you set/rotate but never read
   a value back:
   ```bash
   wax -p <profile> end-users set-secret alice ANTHROPIC_API_KEY sk-ant-…
   wax -p <profile> end-users set-secret alice GITHUB_TOKEN ghp_…
   wax -p <profile> end-users list-secrets alice        # keys only, never values
   ```
   `set-secret` upserts (create or rotate). Use `--agent <name>` to scope a
   secret to one agent; the default applies to all of the tenant's agents.
3. **Run AS that end-user** — the tenant's app stamps a signed `_sub_user_identity` on the signal payload. Stamp BOTH `sub_user_id` and `user_id` (the audit layer drops the identity if either is missing):
   ```json
   POST /api/v1/signals/my_agent_request/
   {"prompt":"...", "_sub_user_identity":{"sub_user_id":"alice","user_id":"alice","email":"alice@acme.com"}}
   ```
   Now the run reasons on **Alice's** Anthropic key and calls tools with **Alice's** token. Resolution precedence is `SubUser → Tool → Agent → Tenant` (sub-user wins). The Token Lab UI is a working reference for this whole flow.

## Gotchas (learned the hard way — respect these)

- **`wax` env breakage:** if `wax` errors with `ModuleNotFoundError` from the repo source, the pipx env is shadowing the source tree. Run it from the repo venv: `OTEL_SDK_DISABLED=true .venv/bin/wax …`.
- **Semver bump on every edit:** re-pushing the same `version:` with changed content fails `semver_conflict`. Bump `version:` (e.g. `1.0.0 → 1.0.1`) before each re-push.
- **Don't hardcode tool names in the prompt** — the SDK may expose a registry/domain tool as `mcp__…__<name>`. Tell the agent to "use the available <X> tool."
- **OTEL locally:** prefix local management/CLI calls with `OTEL_SDK_DISABLED=true OTEL_TRACES_EXPORTER=none` to avoid noisy exporter errors.
- **Per-user secrets are write-only** — you can set/rotate but never read them back; that's by design (the model never sees them).

## Verification checklist (run after a deploy)

```bash
wax -p <profile> whoami                 # right tenant?
wax -p <profile> agents info my_agent   # spec registered, domains/tools correct?
wax -p <profile> signals send my_agent_request --data '{"prompt":"ping"}'
wax -p <profile> runs show <run_id>     # check the Setup span + output
```

When something's off, `wax whoami` is the first check, then the run's `Setup` span (it shows exactly what the agent was provisioned with: allowed_tools, domain tools, mcp servers, identity).

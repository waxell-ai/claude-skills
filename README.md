# Waxell Claude Code Skills

Skills that teach [Claude Code](https://claude.com/claude-code) how to work
with [Waxell](https://waxell.ai) — so Claude does the setup instead of you
following docs.

## Skills

### `instrument-with-waxell-observe`

Walks Claude through adding Waxell observability to a Python agent:

1. Installs `waxell-observe` and configures the `wax` CLI (API key + profile).
2. Adds the two-line decorator instrumentation to your agent.
3. Pulls the first trace via the CLI and verifies what was captured.
4. Escalates only where the trace shows a real gap (extra decorators,
   attributes, `WaxellContext` as a last resort).
5. Optionally instruments memory (Mem0, LangMem, vector stores).
6. Sets up one starter governance policy (a cost cap) and explains it.

**Install:**

```bash
npx degit waxell-ai/claude-skills/instrument-with-waxell-observe ~/.claude/skills/instrument-with-waxell-observe
```

Then open Claude Code in your agent's repo and say:

```
instrument this with waxell
```

### `ship-claude-agent-to-waxell`

Takes a `claude_agent_sdk` / Claude Code agent you built locally and publishes
it to Waxell so your whole team can use it in prod — governed, observable, and
optionally per-user:

1. Scaffolds a `waxell.yaml` from your local agent (system prompt, model, tools).
2. Validates and publishes it with `wax push` (`--include-source` so it travels).
3. Runs it through the signal endpoint and verifies via the run's Setup span.
4. (Optional) Adds a per-end-user tool integration (a "connect your X" domain).
5. (Optional) Maps real end-users to per-user credentials (BYO LLM key + tokens)
   resolved host-side — the model never sees a raw secret.

See [`TUTORIAL.md`](./ship-claude-agent-to-waxell/TUTORIAL.md) for the manual
walkthrough.

**Install:**

```bash
npx degit waxell-ai/claude-skills/ship-claude-agent-to-waxell ~/.claude/skills/ship-claude-agent-to-waxell
```

Then open Claude Code in your agent's repo and say:

```
deploy this agent to Waxell
```

## How skills work

Claude Code auto-discovers any skill placed in `~/.claude/skills/<name>/SKILL.md`
(global) or `<repo>/.claude/skills/<name>/SKILL.md` (per-project). The install
command above drops the skill into your global skills directory so it's
available in every project.

To update, re-run the install command — it overwrites in place.

## Links

- [Waxell docs](https://waxell.ai/docs/)
- [waxell-observe on PyPI](https://pypi.org/project/waxell-observe/)

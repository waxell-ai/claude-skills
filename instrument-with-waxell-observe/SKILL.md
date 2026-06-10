---
name: instrument-with-waxell-observe
description: Add Waxell observability to a Python agent — install the SDK, configure the wax CLI, drop in the two-line decorator pattern, pull a run to verify, escalate to handler/context only if a real gap shows up, then walk the user through one starter governance policy. Use when the user says "instrument this with waxell", "wax this up", "add waxell observability", "set up tracing on this agent", "hook this agent into waxell", or asks for telemetry on a Python agent file. Built around Occam's razor — the 2-line decorator is the default and the goal; everything else is escalation only when the trace shows it's needed.
---

# Instrument with Waxell

A guided onboarding flow for getting a Python agent traced + governed by
Waxell. The skill carries the SDK / CLI / policy complexity; the user
only makes a few small decisions (name the agent, run it once, pick a
starter policy).

## When to use

Trigger phrases:
- "instrument this with waxell" / "wax this up" / "wax this agent"
- "add waxell observability to <file>"
- "set up tracing / telemetry on this agent"
- "hook this agent into waxell"
- "/instrument-with-waxell-observe" (direct invocation)

Trigger context (no phrase needed):
- User opens a Python file with an agent loop / `openai.AsyncOpenAI()` /
  LangChain / LangGraph / CrewAI / AutoGen / `pydantic_ai` and asks
  "what telemetry should I add?" or "how do I track cost on this?"

## Operating principles

Embed these in every phase — they're what keep the skill from drowning
the user.

1. **Occam's razor.** The 2-line decorator is the goal. Never propose
   `WaxellContext` or per-call manual recording unless the verification
   trace in Phase 3 proves the simpler version missed something.
2. **Don't ask what you can grep.** If you can answer the question by
   reading a file or running a CLI, do that first. Only ask the user
   when the answer is genuinely outside Claude's reach (e.g. "did you
   already create an API key in the controlplane?").
3. **One step at a time.** Apply → verify → only then escalate. Don't
   batch decorator + handler + context on the first pass.
4. **Claude handles the complexity.** The user should never have to
   understand the SDK internals to get a working trace. Their job is
   to name the agent, run it once, and pick a starter policy.

## Phased flow

### Phase 0 — Prerequisites

**Grep, don't ask** for each of these:

| Check | How |
|---|---|
| `waxell-observe` installed | `pip show waxell-observe` (and prefer the project's venv if active) |
| CLI installed | `which wax` |
| CLI configured | `wax config show` (or list — the subcommand depends on the CLI version). Config lives at `~/.waxell/config` on macOS/Linux and a platform-equivalent on Windows; the CLI knows where. **Always go through `wax config`, never read or write the file directly.** |

For each gap, surface a single concrete fix:
- Missing `waxell-observe` → `pip install waxell-observe`. If the agent
  is LangChain or LangGraph, install the langchain extra too:
  `pip install "waxell-observe[langchain]"`.
- Missing `wax` CLI → tell the user how to get it (their install path
  depends on the distribution; ask if unclear).
- CLI present but unconfigured → tell the user to run `wax config set
  api-url …` and `wax config set api-key …` (or the relevant
  subcommands). If they don't have an API key yet, surface the
  controlplane path to create one: `/settings/api-keys` in the
  controlplane web UI, "Create new key", copy the `wax_sk_…` once.
- **Ask only this:** "Have you created an API key in the controlplane,
  and is it set in your `wax config`?" If no — walk them through
  creating one. If yes — move on.

#### Bridge the CLI config → agent process

Important SDK detail: `waxell.init()` does **not** read `~/.waxell/config`
on its own. It only consults:
1. Explicit `waxell.init(api_key=..., api_url=...)` kwargs.
2. `WAXELL_API_URL` / `WAXELL_API_KEY` env vars (also accepts `WAX_*`).

That means after `wax config` is set, the agent's *process* still
needs the keys in its environment. Bridge it.

1. Read the user's chosen profile out of the CLI config (default
   profile unless the user named another). Prefer `wax config get
   --profile <name>` style if available; fall back to parsing the file
   only when no CLI command exposes the values.
2. Look for a `.env` at the agent repo root (next to `pyproject.toml`
   / the agent file).
3. Decide what to do:
   - **No `.env` exists** → propose creating one with `WAXELL_API_URL`
     and `WAXELL_API_KEY` pulled from the profile. Show the user the
     exact content before writing. Confirm, then `Write`.
   - **`.env` exists, missing the two keys** → propose appending
     them. Confirm, then `Edit` to append.
   - **`.env` exists, has both keys** → check they match the profile.
     If different, ask the user which is canonical (don't silently
     overwrite — they may have intentionally pointed the agent at a
     different tenant).
4. Tell the user how the agent loads the `.env`:
   - If the project already uses `python-dotenv`, `dotenv` will pick
     it up automatically.
   - Otherwise add `from dotenv import load_dotenv; load_dotenv()` at
     the top of the agent file (one line), or have the user `export`
     the vars in their shell before running.
5. **Warn about secrets in git:** if `.env` isn't already in
   `.gitignore`, add it. Never commit `wax_sk_…`.

If the user has multiple profiles (dev / staging / prod) in
`~/.waxell/config`, ask which one this agent should run against before
writing the `.env`. The selection corresponds to `[<profile>]`
sections in the config file.

> Roadmap note: the SDK's `init()` should grow native CLI-config
> resolution so this `.env` bridge becomes optional. Until then, the
> skill always sets up the env vars.

Do not proceed to Phase 1 until Phase 0 is green.

### Phase 1 — Read the agent

Read the agent file(s) the user is pointing at. Build a small mental
model:

- **Framework signature.** Vanilla `openai` / `AsyncOpenAI`? LangChain
  or LangGraph? CrewAI? AutoGen? `pydantic_ai`? Something hand-rolled?
- **Entry point.** Which function is the "I am called once per agent
  run" function? It's usually the top-level async function that takes
  the user's query and returns the agent's response. Tag it.
- **Sync vs async.** The decorator works on both; the LangChain handler
  is callback-based; `WaxellContext` is a context manager. Pick the
  right tool only after Phase 1 is done.
- **Memory?** Do you see Mem0 (`mem0.add`, `mem0.search`), LangMem,
  LlamaIndex memory, LangChain `ConversationBufferMemory`, or raw
  vector store calls (Pinecone, Weaviate, Chroma, pgvector)? Note it
  — you'll ask the user about it in Phase 5.

Don't write to the file yet. Just understand it.

### Phase 2 — Apply the two-line decorator (the goal state)

Edit ONE file. Add only these two lines, in addition to the decorator
wrap.

At module top (imports section):
```python
import waxell_observe as waxell
waxell.init()
```

On the entry function:
```python
@waxell.observe(agent_name="<short-kebab-case-name>")
async def my_agent(query: str) -> str:
    ...
```

For `agent_name`: use the existing function name if it's already
agent-like (`research_agent`, `triage_bot`). Otherwise ask the user to
pick a name. Don't invent generic ones like `my-agent`.

**LangChain / LangGraph branch.** If Phase 1 said it's a LangChain
agent, skip the decorator and use the handler instead:
```python
from waxell_observe.integrations.langchain import WaxellLangChainHandler

handler = WaxellLangChainHandler(agent_name="<name>")
result = chain.invoke({"question": q}, config={"callbacks": [handler]})
handler.flush_sync(result={"output": result})
```
That's still two lines + the callback wiring. Still Occam's razor —
it's the simplest LangChain-shaped instrumentation.

**Don't add anything else yet.** No sub-function decorators, no
attribute kwargs, no `WaxellContext`. Stop here.

### Phase 3 — Pull a run and inspect

Tell the user: "Run your agent once. I'll pull the trace when you tell
me it's done."

When the user confirms:
1. `wax runs list --limit 5` — find the just-recorded run.
2. `wax run get <id>` — pull the full trace (spans, LLM calls, costs,
   errors).
3. Read what's there. Build a checklist of what the trace contains:
   - Top-level agent span?
   - LLM call(s) captured (model, tokens in/out, cost, latency)?
   - Tool / function calls?
   - Memory ops?
   - Errors / exceptions if any?

If the CLI subcommands above don't match what's actually shipped on the
user's machine, run `wax --help` to learn the real subcommands and
adapt. The shape (list recent runs → get one run's spans) is what
matters.

### Phase 4 — Gap analysis (Claude carries the complexity here)

Compare what's in the trace vs. what's in the code. For each gap,
propose the **smallest** escalation that fills it.

Escalation ladder:

1. **Extra `@waxell.observe` decorator on a sub-function.** Use when a
   meaningful chunk of work isn't showing up as a span. The kwarg
   pattern is the same — `@waxell.observe(agent_name="<parent>",
   span_name="<sub-step>")` if the SDK takes a span_name; otherwise
   just decorate and the function name becomes the span.
2. **`attributes={...}` on an existing decorator.** Use when a span is
   present but missing context (e.g. you want to tag a span with
   `attributes={"user_id": uid, "thread_id": tid}`). Cheaper than a
   new span.
3. **Drop in `WaxellContext`** — **last resort.** Use only when you
   need to manually record an LLM call or tool call that lives outside
   any function the decorator can reach (e.g. an inline `await
   client.chat.completions.create(...)` that you don't want to wrap in
   a function). Pattern:
   ```python
   from waxell_observe import WaxellContext
   with WaxellContext(agent_name="<name>") as ctx:
       result = my_agent.run(input)
       ctx.record_llm_call(model="gpt-4o", tokens_in=150,
                           tokens_out=80, cost=0.0045)
       ctx.set_result({"output": result})
   ```

After each escalation, **go back to Phase 3** — pull another run and
verify the gap is filled. Never apply two escalations without a
verification trace between them.

### Phase 5 — Memory (gated by explicit ask)

Ask the user:
> "Does this agent use memory? Mem0 / LangMem / a vector store / custom
> / none?"

If they say yes (or "I don't know" — then offer to scan for it),
instrument the read and write call sites:
- For Mem0: wrap `mem0.add` and `mem0.search` in named child spans.
- For LangChain memory: the LangChain handler already captures most of
  it; check the trace first before adding anything.
- For raw vector stores: wrap `index.upsert` / `index.query` /
  `collection.add` / `collection.query` in named child spans.

Keep the pattern minimal — one `@waxell.observe` per memory call site
with a clear span name (`memory.retrieve`, `memory.save`).

If they say no — skip cleanly. Don't push.

### Phase 6 — One starter policy

This is the closing demo. **Pick one policy and only one.** The
default starter is a **cost-cap policy**:

> "Warn me if any single agent run costs more than $1."

Why this one: the trace already has the cost data, so the policy works
on day one without extra instrumentation. It's also instantly legible —
operators don't need to read a policy spec to understand "warn over $1".

Walk the user through:
1. What a policy is, in two sentences: "Policies are rules Waxell
   enforces on every agent run. They can warn, block, or throttle
   based on cost, content, model, latency, etc."
2. Where to apply it: the Govern section of the controlplane
   (`/governance` in the web UI), or via the `wax` CLI if there's a
   `wax policy create` command (check `wax --help`).
3. What the rule looks like: "scope: this agent / threshold:
   total_cost > 1.0 USD per run / action: warn".

After it's in, end the flow:
> "You're traced and you have one policy live. More policies (model
> fallback, PII redaction, content filter, latency budgets) are a
> conversation for when you actually want them — they'd be premature
> right now."

Do not list every available policy type. Don't dump the catalog.

## Style guide

- **Be conversational, not procedural.** "Cool, you've got the SDK
  installed. Let me check the CLI." beats "Phase 0 step 2: verify CLI
  installation."
- **Show the code you're about to add before adding it.** Tiny diffs
  preview, then `Edit`.
- **Surface every assumption.** "I'm decorating `research_agent` as
  the entry point — if `kickoff` is the real entry, redirect me."
- **Always go back to the trace.** Each escalation in Phase 4 must
  produce a re-pulled trace as proof.

## Failure modes & how to handle them

- **No agent file in sight.** Ask: "Which file is the agent? Paste
  the path or open it." Don't guess.
- **Multiple entry points (e.g. an agent that's also a server route).**
  Pick the function that takes the user's input and returns the
  agent's output as the decorated one. The HTTP handler stays
  un-decorated.
- **`wax runs list` returns nothing after the user says they ran the
  agent.** Most common cause: `WAXELL_API_KEY` or `WAXELL_API_URL` not
  exported in the agent's process. Have them check `wax config show`
  and re-run, or have them export those env vars before running.
- **The decorator silently does nothing in a sync agent inside a Jupyter
  notebook.** Notebook event loops can suppress async cleanup. Tell the
  user to call `waxell.flush_sync()` at the end of the cell — or move
  the agent into a `.py` file for the verify step.
- **Trace shows `agent_name` as `unknown` or empty.** They forgot to
  pass `agent_name=` to the decorator — fix and re-run.
- **The user says "I don't have an API key yet" mid-flow.** Drop back
  to Phase 0. Walk them through `/settings/api-keys`. Don't try to
  proceed without one.

## What this skill never does

- Instrument every function in the agent. Occam's razor.
- Read or write `~/.waxell/config` directly. Always go through `wax
  config`.
- List every available policy type. One starter policy, that's it.
- Push memory instrumentation when the user said no.
- Use `WaxellContext` as a first move. It's the last resort.
- Run the user's agent for them — they run it, you read the trace.

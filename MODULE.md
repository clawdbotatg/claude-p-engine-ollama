# engine-ollama — local Ollama models as a chat engine

An **engine module** (protocol v0 — contract: `PLAN-ENGINES.md` in the brain
repo): one executable, `engine`, that runs turns on a **local Ollama server**
instead of Claude. Nothing leaves the machine, no API key, no metered spend.
Built for one job in particular: running the agent's persona on small local
models to see how dumb a brain can be and still behave.

Selected explicitly — `ENGINE=engine-ollama` in `.env`, or per call:
`run_turn(..., engine="engine-ollama")` / `adapters/run.py --engine
engine-ollama`. Installing it activates nothing.

By default it is a **chat engine**: no tools, no file access, no cwd. The
kernel enforces the edges (claude-only options hard-error; self-modification
stays on the claude engine). Conversation state is the message history,
stored by the kernel per `remember=` key (`.memory/<key>@engine-ollama.state`).
Reasoning models that leak `<think>…</think>` into their reply get it
stripped (emitted as a `thinking` event instead of polluting history).

**Tools mode** (`OLLAMA_ENGINE_TOOLS=bash`): the model gets exactly one
tool — run a shell command in the agent dir — in an OpenAI-function-calling
loop (`OLLAMA_ENGINE_MAX_TOOL_TURNS`, default 8; per-command
`OLLAMA_ENGINE_TOOL_TIMEOUT`, default 60s; output capped at 8k chars; the
last lap withholds the tool to force a text answer). This is a real agent
loop on a local model — which is the experiment: what's the smallest model
that actually works? Tool traffic is per-turn scaffolding, never persisted
to state.

## What it needs

- A local [Ollama](https://ollama.com) (`brew install ollama`) with at least
  one pulled model, and the server up (`ollama serve`, or the desktop app).
- Env (brain's `.env`): `OLLAMA_ENGINE_MODEL` — required, e.g. `llama3.2:3b`
  (`ollama list` shows what you have; the engine's error message does too).
  Optional: `OLLAMA_ENGINE_BASE_URL` (default `http://localhost:11434/v1`),
  `OLLAMA_ENGINE_HISTORY` (default 40), `OLLAMA_ENGINE_TIMEOUT` (default
  300s — cold-loading big weights into RAM is slow; first turn pays it).

## Wiring

`tools/module add engine-ollama`, set `OLLAMA_ENGINE_MODEL` in `.env`, prove
it with `tools/engine-check engine-ollama`. That's all. Different models for
different callers: override the env var per invocation
(`OLLAMA_ENGINE_MODEL=qwen3.5:35b-a3b adapters/run.py --engine engine-ollama …`).

## How to verify

```bash
tools/engine-check engine-ollama       # protocol v0 conformance battery
```

## What can go wrong

- **Tools mode is a shell in the hands of whatever model you picked.** A
  small model *will* eventually run something dumb. The brain's guard hook
  does NOT protect these turns (Claude Code hooks are claude-engine-only),
  and there is no permission prompt. Supervised experiments in a disposable
  clone only — never wire an unattended adapter to tools mode.
- **In chat mode, no tools means confidently wrong self-reports.** The model can't read
  the repo or run `tools/vitals`; if asked about itself it answers from the
  persona text alone — or hallucinates. That's partly the point (it's the
  baseline claude is compared against), but never route body-work here.
- **Small models ignore the persona**, answer in the wrong voice, or leak
  their scratchpad. Expected at 3B; interesting at 30B+.
- First turn after boot can take minutes on big weights (model load), then
  it's fast. If the server is down the engine dies loudly with a hint.
- **History truncation is dumb** (last N messages) — long threads silently
  forget their beginning.
- Local ≠ safe to overshare: the model is local but the machine may sync
  logs; treat `.memory/` blobs like any conversation history.

## How to uninstall

`tools/module remove engine-ollama`, delete `OLLAMA_ENGINE_*` (and any
`ENGINE=engine-ollama`) from `.env`, and remove stale blobs:
`rm .memory/*@engine-ollama.state`.

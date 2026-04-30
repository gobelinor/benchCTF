# benchCTF

Benchmark AI agents on CTF challenges.

![Claude Code Opus 4.7 vs Codex GPT-5.5 on the CryptoHack `lets-decrypt` challenge, 5 runs each](screen-codex-claude.png)

You give benchCTF a challenge directory (a `challenge.md` plus any binaries, ciphertext, source files) and a list of models. Each model runs autonomously in its own fresh working directory under the same prompt and methodology — it reads the files, writes scripts, runs commands, talks to remote services, and reports back the flag it found.

The output is a markdown report comparing every model on: wall time, tokens consumed, success rate, and a synthetic dollar cost (token counts × public API rates) so subscription-based stacks like Claude Code or Codex can be compared on the same axis as pay-per-token APIs.

## Modular by design

Three orchestrators ("runners") are supported, picked per-model in `models.yaml`:

| runner | how it runs | auth |
|---|---|---|
| `opencode` | `opencode run --format json` | whatever you set up via `opencode auth login` (Anthropic / OpenAI / Google / Copilot / OpenRouter / API keys) |
| `claude_code` | `claude -p --output-format stream-json` | inherits Claude Code's OAuth (Claude Pro / Max / Team / API key) |
| `codex` | `codex exec --json` | inherits Codex's ChatGPT OAuth (Plus / Pro / API key) |

You can mix freely:

- **Pure model comparison** — same orchestrator (e.g. `opencode`), several models, only the model varies.
- **Cross-stack comparison** — different orchestrators, e.g. Claude Code vs Codex vs opencode-with-Gemini, end-to-end stacks compared as black boxes.

## Run this in a VM

> ⚠️ Strongly recommended: run benchCTF inside a disposable VM.
>
> Two reasons:
>
> 1. **No sandbox, no permissions prompt.** Every runner is launched with its "yolo" flag (`--dangerously-skip-permissions` / `--dangerously-bypass-approvals-and-sandbox`). The agent has full shell access, can install packages, hit the network, write anywhere under your user. Pwn challenges in particular run untrusted attacker-controlled binaries — don't do that on your laptop.
> 2. **Cleaner comparison.** A fresh VM has no `~/.claude/CLAUDE.md`, no skills, no `~/.codex/AGENTS.md`, no walked-up `AGENTS.md` — none of the host context that biases each runner (see Limitations). 

## Install

Requirements:

- Python ≥ 3.10
- Whichever CLI you intend to use, on `PATH`:
  - `opencode` ≥ 1.3 + at least one provider authenticated (`opencode auth login`)
  - `claude` (Claude Code), already logged in
  - `codex`, already logged in

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp models.yaml.example models.yaml   # then trim to the models you want
```

## Run

```bash
python bench.py --challenge challenges/example-easy
```

| flag | default | purpose |
|---|---|---|
| `--challenge PATH` | (required) | challenge dir (must contain `challenge.md`) |
| `--models PATH` | `models.yaml` | model config |
| `--runs N` | from `defaults.runs` | runs per model (for mean/median/std) |
| `--timeout SECS` | from `defaults.timeout_seconds` | per-run wall-clock cap |
| `--no-agents` | off | skip the methodology prefix (baseline mode) |
| `--agents PATH` | `templates/AGENTS.md` | use a custom methodology |
| `--only NAMES` | (all) | comma-separated model names to include |
| `--out DIR` | `runs` | output root |

Models run **sequentially**, each in its own fresh working directory copied from the challenge dir.

## Examples

### Same orchestrator, different models — pure model comparison

`models.yaml`:

```yaml
defaults:
  timeout_seconds: 1800
  runs: 3

models:
  - name: opencode-claude-sonnet-4-6
    runner: opencode
    opencode_model: anthropic/claude-sonnet-4-6
    pricing: { input: 3.00, output: 15.00, cache_read: 0.30, cache_write: 3.75 }

  - name: opencode-gpt-5
    runner: opencode
    opencode_model: openai/gpt-5
    pricing: { input: 0.625, output: 5.00, cache_read: 0.0625 }

  - name: opencode-gemini-2.5-pro
    runner: opencode
    opencode_model: google/gemini-2.5-pro
    pricing: { input: 1.25, output: 10.00, cache_read: 0.125 }
```

Same harness, same prompt, same workdir layout — only the model varies.

### Different orchestrators, different models — full-stack comparison

`models.yaml`:

```yaml
defaults:
  timeout_seconds: 2700
  runs: 1

models:
  - name: claude-opus-4-7
    runner: claude_code        # uses Claude Code's OAuth
    claude_model: opus
    pricing: { input: 5.00, output: 25.00, cache_read: 0.50, cache_write: 6.25 }

  - name: codex-gpt-5.5
    runner: codex              # uses Codex's ChatGPT OAuth
    codex_model: gpt-5.5
    pricing: { input: 5.00, output: 30.00, cache_read: 0.50 }
```

This compares two real-world stacks (CLI + model + ambient config). All three runners load some ambient host context — see the limitations section for the specifics. That's part of what you're benchmarking when you compare full stacks.

## Output

Each invocation creates `runs/<utc-timestamp>/`:

```
runs/20260428-194641/
├── report.md                  ← summary + per-run details
├── results.json               ← all results in one array
└── <model-slug>/run-<n>/
    ├── workdir/               ← isolated copy of challenge files
    ├── stdout.jsonl           ← raw runner event stream
    ├── stderr.log
    └── result.json            ← per-run record
```

A `rich` table is also printed to the console at the end.

## Challenge format

```
challenges/<name>/
├── challenge.md     # description (free-form), optional YAML frontmatter
└── <files…>         # binaries, ciphertext, pcaps, etc.
```

The agent is told to terminate its final message with `FLAG: <value>` (or `FLAG: NOT_FOUND`). The harness scans the **last assistant message** for that line. The flag value is recorded as-is — verify against the canonical flag in `result.json` if needed.

## Methodology (`AGENTS.md`)

`templates/AGENTS.md` ships a generic CTF methodology (recon → classify → exploit → flag → contract). It is **prepended to the prompt**, so every runner receives the same instructions regardless of how each CLI handles its own conventions. `--no-agents` disables it; `--agents PATH` swaps in a custom one.

## Cost: synthetic vs native

Every run records two costs:

- `cost_usd_synthetic` — `tokens × pricing` from `models.yaml`, always populated. Use this to compare across models, including those running on flat-rate OAuth subscriptions.
- `cost_usd_native` — what the runner itself reports. `claude_code` exposes `total_cost_usd`; `opencode` exposes `info.cost`; `codex` doesn't report cost (always `$0`).

> Pricing in `models.yaml.example` is the public API rate as of **2026-04-28**. Re-check before running long benchmarks if rates may have shifted.

## Models config schema

```yaml
- name: <free-form display name>
  runner: opencode | claude_code | codex
  pricing:
    input: <USD per Mtok>           # required
    output: <USD per Mtok>          # required
    cache_read: <USD per Mtok>      # optional, default 0
    cache_write: <USD per Mtok>     # optional, default 0
    reasoning: <USD per Mtok>       # optional, defaults to output

  # runner=opencode:
  opencode_model: provider/model    # e.g. anthropic/claude-sonnet-4-5
  variant: high                     # optional, opencode --variant

  # runner=claude_code:
  claude_model: opus                # alias or full id
  claude_args: [--max-turns, "100"] # optional, extra args passed to claude

  # runner=codex:
  codex_model: gpt-5.5
  codex_args: ["-c", "model_reasoning_effort=high"]   # optional
```

Tokens are normalized across runners to non-overlapping buckets (`input` is fresh-only, `cache_read` is cached input, etc.) so the synthetic-cost formula stays consistent regardless of where the data came from.

## Limitations

- No Docker. Each run uses an isolated working directory but trusts the host. Don't run untrusted CTF challenges (esp. pwn) without your own sandboxing.
- Sequential, not parallel. Avoids OAuth rate-limit collisions and keeps reports deterministic.
- Flag is **self-reported**. The harness doesn't know the canonical flag unless you supply it.
- **None of the runners are context-clean.** Each one auto-loads ambient host config:
  - `claude_code` → `~/.claude/CLAUDE.md`, `~/.claude/skills/*`, hooks, plugins, settings.
  - `codex` → `~/.codex/AGENTS.md`, `~/.codex/skills/*`, `~/.codex/config.toml`.
  - `opencode` → walks up the workdir's parent chain for `AGENTS.md` / `CLAUDE.md`, **and** auto-loads `~/.claude/CLAUDE.md` plus skills from `~/.claude/skills/` and `~/.agents/skills/`.

  When you compare across runners you're comparing full stacks, not bare models. To get closer to a clean comparison, run with stripped-down host config (move/rename your global rules and skill dirs out of the way before benchmarking), or stay within a single runner and only vary the model.

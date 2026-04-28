# benchCTF

Benchmark and compare AI models on **agentic CTF challenge solving** under a single, identical orchestrator.

The fairness invariant: every model goes through the **same** [`opencode`](https://opencode.ai/) `run` invocation, with the **same** prompt, in its **own isolated working directory**. The only thing that changes between runs is the underlying model. The harness records tokens, wall time, success rate, and a synthetic cost so models on flat-rate OAuth subscriptions (Claude Pro/Max, Copilot, Gemini, etc.) can be compared on equal footing with metered API providers.

```
┌──────────────┐                ┌────────────────┐
│  bench.py    │ ──spawn───►    │  opencode run  │ ──►  isolated workdir
│  orchestrator│                │  --model X     │       per (model, run)
└──────┬───────┘                └────────┬───────┘
       │                                 │ JSON events
       │                                 ▼
       │                         opencode export
       │                                 │
       └──── aggregate, write ───────────┘  →  runs/<ts>/report.md
```

## Why opencode as the harness

`opencode` already abstracts over Anthropic, OpenAI, Google, GitHub Copilot, OpenRouter, and more — and it supports OAuth login flows for the providers that offer them. That means you can typically benchmark **without a single API key**: log in once with `opencode auth login` and reuse your existing subscriptions.

## Install

Requirements:

- Python ≥ 3.10
- `opencode` ≥ 1.3 on `PATH` (`npm install -g opencode-ai` or [opencode.ai/docs/install](https://opencode.ai/docs/install))
- One or more authenticated providers: `opencode auth login` (Anthropic / OpenAI / Google / GitHub Copilot / OpenRouter / …)

```bash
git clone <your fork of this repo>
cd benchCTF
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp models.yaml.example models.yaml   # then edit to keep only models you can run
```

Verify your auth:

```bash
opencode auth list
opencode models                       # should list models for each authenticated provider
```

## Run

```bash
python bench.py --challenge challenges/example-easy
```

Common flags:

| flag | default | purpose |
|---|---|---|
| `--challenge PATH` | (required) | dir containing `challenge.md` and any other files |
| `--models PATH` | `models.yaml` | model configuration |
| `--runs N` | `defaults.runs` from yaml (3) | runs per model — for mean/median/std |
| `--timeout SECS` | `defaults.timeout_seconds` (1800) | per-run wall-clock cap |
| `--no-agents` | off | run baseline (no `AGENTS.md` injected) |
| `--agents PATH` | `templates/AGENTS.md` | use a custom methodology file |
| `--only NAMES` | (all) | comma-separated model names to include |
| `--out DIR` | `runs` | output root |

The harness runs models **sequentially** to avoid OAuth rate-limit collisions and keep the report deterministic.

## Output

Each invocation creates `runs/<utc-timestamp>/`:

```
runs/20260428-180123/
├── report.md                       ← summary + per-run details (markdown)
├── results.json                    ← all results in one JSON array
├── claude-sonnet-4-6/
│   ├── run-1/
│   │   ├── workdir/                ← isolated copy of challenge + AGENTS.md
│   │   ├── stdout.jsonl            ← raw opencode JSON event stream
│   │   ├── stderr.log
│   │   └── result.json             ← per-run record
│   ├── run-2/…
│   └── run-3/…
└── gpt-5/…
```

A console summary table is also printed at the end.

## Challenge format

```
challenges/<name>/
├── challenge.md     # description, free-form. Optional YAML frontmatter.
└── <files…>         # binaries, ciphertext, pcaps, etc.
```

Every file in the challenge dir is copied verbatim into the per-run workdir before the agent starts. The agent is told (via the prompt + `AGENTS.md`) to terminate its final message with:

```
FLAG: <exact flag>
```

…or `FLAG: NOT_FOUND` if it gives up. The harness scans the **last assistant message** for a line matching `^FLAG:\s*(.+)$` (last match wins). Flag value is recorded as-is — no canonical-flag verification, you can post-hoc check `result.json`.

## Model configuration

Edit `models.yaml`. Each entry:

```yaml
- name: claude-sonnet-4-6              # display name (free-form)
  opencode_model: anthropic/claude-sonnet-4-5   # what `opencode --model` expects
  variant: high                        # optional reasoning effort
  pricing:                             # USD per million tokens, public API rates
    input: 3.00
    output: 15.00
    cache_read: 0.30                   # optional, defaults to 0
    cache_write: 3.75                  # optional, defaults to 0
```

`opencode_model` must match a model that `opencode models` lists for an authenticated provider. Run `opencode models` to confirm what's available.

### Synthetic cost vs opencode-reported cost

Two costs are recorded:

- **`cost_usd_synthetic`** — `tokens × pricing` from `models.yaml`. Always populated. Use this to compare models on equal footing, including those on flat-rate subscriptions.
- **`cost_usd_opencode`** — what `opencode export` reports. Often `$0` for OAuth/subscription providers.

If you want a more sophisticated cost model (cached vs uncached input tokens priced differently, reasoning tokens, etc.), edit the `compute_synthetic_cost` function in `bench.py`.

## With / without skills (AGENTS.md toggle)

The repo ships a generic CTF methodology in `templates/AGENTS.md` (recon → classify → exploit → flag → format contract). It's copied into the workdir for each run by default; opencode auto-loads any `AGENTS.md` it finds.

- `python bench.py --challenge X`              → with `templates/AGENTS.md`
- `python bench.py --challenge X --agents path/to/custom.md`  → with custom methodology
- `python bench.py --challenge X --no-agents`  → baseline, no methodology injected

This is the project's "with skills / without skills" axis. Discovery-driven skill systems (e.g. Claude Code's `~/.claude/skills`) are **not** auto-mounted because opencode lacks an equivalent triggering mechanism — you'd be comparing apples to oranges. If you want to test specific skills, concatenate their contents into a custom `AGENTS.md` and pass it via `--agents`.

## Limitations

- No Docker. Each run uses a fresh tmp-style directory but trusts the host. Don't run untrusted CTF challenges, especially pwn binaries, without your own sandboxing.
- Sequential, not parallel. Avoids OAuth rate limits and produces deterministic reports; takes longer.
- Flag value is **self-reported** by the agent. The harness doesn't know the canonical flag unless you tell it (post-hoc verify against `result.json`).
- `opencode export` is the source of truth for tokens/cost. If opencode's schema changes, parsing in `aggregate_session()` may need updating.
- `--variant` is provider-specific; some providers will error out if you pass it. Omit `variant:` from `models.yaml` for those.

## Project layout

```
bench.py                 # the orchestrator, single file
templates/AGENTS.md      # default CTF methodology
models.yaml.example      # model config template
challenges/example-easy/ # sanity-check challenge (base64 + 1-byte XOR)
runs/                    # output root (gitignored)
requirements.txt
```

## License

MIT (add a `LICENSE` file when creating the GitHub repo).

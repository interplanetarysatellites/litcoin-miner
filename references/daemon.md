# Autonomous Daemon Setup

The `scripts/litcoin_bounty_miner.py` script is a production-grade autonomous miner with:

- **Multi-model rotation** — cycles Groq → OpenRouter → local Ollama on failures/rate limits
- **Self-updating task list** — refreshes from coordinator API every 30 min, always targets highest-reward virgin tasks
- **Auto-submit + queue** — submits improvements as they're found, tracks local bests per task
- **Dual-wallet support** — run separate instances per wallet (`--wallet main` / `--wallet nookplot`)
- **Heartbeat-compatible** — designed to run as a background daemon, monitored via log file

## Setup

### 1. Required API keys

```bash
export BANKR_API_KEY="bk_YOUR_KEY"        # from bankr.bot/api
export GROQ_API_KEY="gsk_YOUR_KEY"        # from console.groq.com (free tier)
export OPENROUTER_API_KEY="sk-or-v1-..."  # from openrouter.ai/keys (free tier)
```

Or hardcode at the top of the script (lines 20-30).

### 2. Install dependencies

```bash
pip install litcoin requests
```

### 3. Run

```bash
# Foreground (test)
python3 litcoin_bounty_miner.py --wallet main

# Background daemon
nohup python3 litcoin_bounty_miner.py --wallet main >> logs/litcoin_main.log 2>&1 &
```

### 4. Monitor

```bash
tail -f logs/litcoin_main.log
```

## Model Rotation Logic

The script maintains a priority list per wallet. When a model hits 3+ consecutive failures (including 429 rate limits), it rotates to the next:

```
main wallet:   groq (llama-3.3-70b) → qwen36 (OpenRouter free) → ollama (local)
nookplot:      ollama → qwen36 → groq
```

To change models, edit the `MODELS_BY_WALLET` dict at the top of the script.

## Heartbeat Integration

Add to HEARTBEAT.md to auto-restart if daemon dies:

```markdown
- [ ] Check litcoin daemon: `pgrep -af litcoin_bounty_miner` — if dead, restart:
  `cd workspace && nohup python3 litcoin_bounty_miner.py --wallet main >> logs/litcoin_main.log 2>&1 &`
```

## Recommended Free Models (OpenRouter)

- `qwen/qwen3.6-plus-preview:free` — recommended by LITCOIN dev, strong reasoning
- `arcee-ai/trinity-large-preview:free` — solid fallback
- `google/gemini-2.5-flash-preview:free` — fast, good at code tasks

## Task Types

The daemon targets tasks in priority order (highest reward first):

| Type | Description | Reward Multiplier |
|------|-------------|-------------------|
| ARC | Abstract Reasoning Corpus patterns | 5x |
| SWE | Software engineering (real GitHub issues) | 5x |
| CF | Competitive programming | 3x |
| MATH | Mathematical optimization | 3x |

VIRGIN tasks (no prior submissions from your wallet) are prioritized — higher potential score ceiling.

# openclaw-model-eval

**Model capability test suite for [OpenClaw](https://openclaw.ai) — benchmark any model across 7 tiers from pure reasoning to full agentic pipeline.**

Developed by [The AI Horizon](https://theaihorizon.org) while validating local Qwen models as zero-cost alternatives to cloud APIs.

---

## What This Is

A structured benchmark for OpenClaw agents. Each test runs in an **isolated session** (no bleed between tests) and checks real capabilities — not simulated ones. The model must actually call tools, handle failures, and produce structured output.

### Validated Results (qwen3.5:35b-nothink, 2026-03-12)

| Tier | Name | Score |
|------|------|-------|
| 0 | Pure Reasoning | 6/6 ✅ |
| 1 | Single Tool Calls | 9/9 ✅ |
| 2 | Multi-Step Single Domain | 5/5 ✅ |
| 3 | Cross-Tool Agentic | 2/2 ✅ |
| 4 | Full Research Pipeline | 1/1 ✅ |
| 5 | Adversarial / Edge Cases | 5/5 ✅ |
| 6 | Domain-Specific | 4/4 ✅ |
| — | Stress Test (Tier 0+1 × 3 runs) | 36/36 ✅ |
| **Total** | | **32/32 ✅** |

Full write-up: [How to run OpenClaw with a local Qwen model](https://gist.github.com/TheAIHorizon/37c30e375f2ce08e726e4bb6347f26b1)

---

## Tiers

| Tier | Name | Tests | What It Validates |
|------|------|-------|-------------------|
| **0** | Pure Reasoning | 6 | Logic, math, JSON, instruction following — no tools |
| **1** | Single Tool | 9 | Each OpenClaw tool called once correctly |
| **2** | Multi-Step Single Domain | 5 | Chained tool use in one domain |
| **3** | Cross-Tool Agentic | 2 | Complex tasks spanning multiple tools |
| **4** | Full Research Pipeline | 1 | End-to-end: search → memory → exec → report |
| **5** | Adversarial / Edge Cases | 5 | Ambiguity handling, conflict resolution, failure recovery, refusal |
| **6** | Domain-Specific (AI Horizon) | 4 | Evidence classification, forecast scoring, DCWF mapping |

---

## Requirements & Prerequisites

### What every tier needs

- [OpenClaw](https://openclaw.ai) installed and configured
- `openclaw` CLI available in PATH
- A configured agent (default: `main`) with at least one working model

### Start here: Tier 0 only needs OpenClaw

Tier 0 is pure reasoning — no tools, no external services. If you just want to check whether a model can think, start here:

```bash
python3 model-eval.py --tier 0 --label "reasoning-only"
```

No other setup required.

---

### Tool requirements by tier

Each tier adds tools. Here's exactly what each one needs and how to verify it works before running.

#### Tier 1 — Single Tool Calls

| Test | Tool | What you need | Verify |
|------|------|---------------|--------|
| T1.1, T1.2 | `exec` (shell) | `exec` skill enabled in your agent | `openclaw exec --agent main "run: echo hello"` |
| T1.3 | `memory_search` (QMD) | QMD installed + workspace indexed | `qmd search "test"` returns results |
| T1.4 | `chromadb_search` | ChromaDB running on **port 8100**, collection `longterm_memory` exists | `curl http://localhost:8100/api/v2/collections` |
| T1.5 | `web_fetch` | Internet access, `web_fetch` skill enabled | Skill in your agent's skill list |
| T1.6 | `web_search` via SearXNG | SearXNG running — **see SearXNG setup below** | `curl http://YOUR_SEARXNG_HOST:PORT/search?q=test&format=json` |
| T1.7 | `exec` + remindctl | **macOS only** — Reminders app + remindctl or equivalent | `remindctl list` returns output |
| T1.8, T1.9 | `gog` (Google Sheets) | `gog` CLI installed + Google account authenticated + `EVAL_TEST_SHEET_ID` set | `gog sheets get $EVAL_TEST_SHEET_ID "Sheet1!A1"` |

**Skip tests you can't support** using `--test` to run only specific IDs, or `--fast` to skip anything that requires external services.

#### Tier 2 — Multi-Step

- T2.1–T2.3: Only `exec` needed
- T2.4: `web_fetch` (internet)
- T2.5: SearXNG (see below)

#### Tier 3 — Cross-Tool Agentic

- T3.1: `gog` (Google Sheets read + write)
- T3.2: ChromaDB (write + read round-trip)

Both tests are **destructive** (write data) — requires `--all` flag and `EVAL_TEST_SHEET_ID` set.

#### Tier 4 — Full Pipeline

All three: SearXNG + `memory_search` (QMD) + `exec`. This is the most demanding tier.

#### Tier 5 — Adversarial

No external tools. Pure model behavior — ambiguity, conflict handling, refusal. Runs anywhere Tier 0 runs.

#### Tier 6 — Domain-Specific

No external tools. Structured reasoning tasks. Runs anywhere Tier 0 runs.

---

### SearXNG setup

**Repo:** [github.com/searxng/searxng](https://github.com/searxng/searxng) | **Docs:** [docs.searxng.org](https://docs.searxng.org)

SearXNG is a free, self-hosted meta search engine. It's required for T1.6, T2.5, T3.x, and T4.1. You host it yourself — no API key, no rate limits.

**Option 1 — Docker (recommended, 60 seconds):**
```bash
docker run -d \
  -p 4000:8080 \
  --name searxng \
  -e SEARXNG_SECRET=$(openssl rand -hex 32) \
  searxng/searxng
```

**Option 2 — Docker Compose (persistent config):**
```bash
git clone https://github.com/searxng/searxng-docker
cd searxng-docker
docker compose up -d
```

**Enable JSON output** (required for the eval — the API won't work without it):

Edit `settings.yml` inside your SearXNG container or volume and ensure:
```yaml
search:
  formats:
    - html
    - json
```

Then restart: `docker restart searxng`

**Verify it's working:**
```bash
curl "http://localhost:4000/search?q=test&format=json" | python3 -m json.tool | head -10
# Should return {"query": "test", "results": [...]}
```

If you get `{"error": "..."}` or a 403, JSON output is not enabled yet.

**Update the URL in the tests** — the script uses a placeholder `YOUR_SEARXNG_HOST:PORT`. Replace it before running:
```bash
# macOS
sed -i '' 's|YOUR_SEARXNG_HOST:PORT|localhost:4000|g' model-eval.py

# Linux
sed -i 's|YOUR_SEARXNG_HOST:PORT|localhost:4000|g' model-eval.py
```

---

### ChromaDB setup

**Repo:** [github.com/chroma-core/chroma](https://github.com/chroma-core/chroma) | **Docs:** [docs.trychroma.com](https://docs.trychroma.com)

ChromaDB is an open-source vector database used by OpenClaw for long-term conversational memory. Required for T1.4 (search) and T3.2 (write + read round-trip).

**Install:**
```bash
pip install chromadb
```

**Run on port 8100** (the port OpenClaw and these tests expect):
```bash
chroma run --host 0.0.0.0 --port 8100
```

> If you want it to run persistently in the background, use a tool like `screen`, `tmux`, or create a systemd/launchd service.

**Create the required collection** — the tests look for a collection named exactly `longterm_memory`. Create it once:
```bash
python3 -c "
import chromadb
client = chromadb.HttpClient(host='localhost', port=8100)
client.get_or_create_collection('longterm_memory')
print('Collection ready')
"
```

**Verify both the server and collection:**
```bash
# Server up?
curl http://localhost:8100/api/v2/heartbeat
# {"nanosecond heartbeat": ...}

# Collection exists?
curl http://localhost:8100/api/v2/collections
# Should include "longterm_memory" in the response
```

> **Note:** The tests use ChromaDB v2 API (`/api/v2/`). Make sure you're running ChromaDB 0.5.0 or later.

---

### QMD (memory_search) setup

**Package:** [@tobilu/qmd](https://www.npmjs.com/package/@tobilu/qmd) on npm

QMD is OpenClaw's workspace memory system — BM25 + vector search over your markdown files. Required for T1.3 (`memory_search`) and T4.1 (full pipeline).

**Install:**
```bash
npm install -g @tobilu/qmd
```

**Index your workspace:**
```bash
# Point at a directory containing .md files
qmd update   # scans for markdown files
qmd embed    # runs embeddings (downloads a ~329MB model on first run)
```

**Verify:**
```bash
qmd search "test query"
# Should return results if you have indexed content
```

If T1.3 returns "no results found" — that's expected if your workspace has no content. Either skip this test or add a few `.md` files and re-run `qmd update && qmd embed`.

---

### gog (Google Sheets) setup

**Package:** [@tobilu/gog](https://www.npmjs.com/package/@tobilu/gog) on npm

`gog` is a Google Workspace CLI (Sheets, Drive, Gmail, Calendar). Required for T1.8, T1.9 (Sheets read/write), and T3.1 (cross-tool Sheets task). Uses OAuth — no API key needed, just a Google account.

**Install:**
```bash
npm install -g @tobilu/gog
```

**Authenticate:**
```bash
gog auth login
# Opens browser for Google OAuth — sign in with the account that owns your sheet
```

**Create a throwaway test sheet** (don't point at a real sheet — tests write rows):
```bash
gog sheets create "OpenClaw Eval Test Sheet"
# Output includes the sheet ID, e.g.:
# Created: https://docs.google.com/spreadsheets/d/1abc.../edit
#                                                  ^^^^^ this is EVAL_TEST_SHEET_ID

export EVAL_TEST_SHEET_ID=1abc...
```

**Verify auth is working:**
```bash
gog sheets get $EVAL_TEST_SHEET_ID "Sheet1!A1"
# Should return the cell value (empty is fine for a new sheet)
```

---

### Quickstart: what to run based on your setup

| Your setup | Command |
|------------|---------|
| Just OpenClaw, no extras | `python3 model-eval.py --tier 0 5 6 --label "baseline"` |
| OpenClaw + exec skill | `python3 model-eval.py --tier 0 1 2 --fast --label "no-network"` |
| Full stack (SearXNG + ChromaDB + QMD) | `python3 model-eval.py --tier 0 1 2 3 4 --label "full"` |
| Everything including writes | `python3 model-eval.py --all --label "complete"` |

The `--fast` flag skips any test that requires an external network call (SearXNG, web_fetch, Sheets).

---

## Usage

```bash
# Run baseline (Tier 0 + 1) — fast, no external calls
python3 model-eval.py --tier 0 1 --fast --label "baseline"

# Run all tiers
python3 model-eval.py --label "full-run"

# Test a specific model (swaps primary, restores after)
python3 model-eval.py --tier 0 1 2 --model "localgpu/qwen3.5:35b-nothink" --label "qwen-test"

# Stress test — run 3x for reliability baseline
python3 model-eval.py --tier 0 1 --repeat 3 --label "stress-test"

# Adversarial tests
python3 model-eval.py --tier 5 --label "adversarial"

# AI Horizon domain tests
python3 model-eval.py --tier 6 --label "domain"

# Specific test IDs
python3 model-eval.py --test T0.1 T1.3 T5.2

# Dry run — see what would run without executing
python3 model-eval.py --dry-run

# Compare two runs
python3 model-eval.py --compare run-id-1 run-id-2
```

---

## Configuration

### --model flag

Temporarily swaps `agents.list[id=main].model.primary` in `openclaw.json`, restarts the gateway (~30s), runs the eval, then restores the original model. Gateway is unavailable during swap.

```bash
python3 model-eval.py --tier 0 1 --model "localgpu/qwen3.5:35b-nothink"
```

If interrupted mid-run, restore manually:
```bash
# Check current primary
grep -A2 '"id": "main"' ~/.openclaw/openclaw.json | grep primary
# Edit openclaw.json → agents.list[id=main].model.primary → your original model
# Then restart gateway
launchctl kickstart -k gui/$UID/ai.openclaw.gateway
```

---

## Output

Results are saved to `~/.openclaw/workspace/eval-results/` as JSON + Markdown.

```
eval-results/
  20260312-114552-ee2646.json   # raw results with token counts, timings
  20260312-114552-ee2646.md     # formatted report
```

Post results to Slack after a run:
```bash
python3 model-eval.py --tier 0 1 --slack --label "weekly-check"
```

---

## Adding Tests

Tests are defined in the `TESTS` list in `model-eval.py`. Each test is a dict:

```python
{
    "id": "T5.6",              # unique ID
    "tier": 5,                 # tier number
    "name": "My test",         # short description
    "prompt": "...",           # sent to the agent
    "expect_contains": ["keyword1", "keyword2"],  # pass if ANY matched
    "expect_json": True,       # optional: validate JSON in response
    "timeout": 180,            # seconds
    "fast": True,              # True = no external API calls
    "destructive": False,      # True = skipped unless --all
}
```

Pass logic: test passes if **at least one** `expect_contains` keyword appears in the response (case-insensitive).

---

## Stress Testing

Use `--repeat N` to run the same tests multiple times and measure consistency:

```bash
python3 model-eval.py --tier 0 1 --repeat 5 --label "reliability"
```

Each repeat gets a fresh isolated session. Results show all N runs — look for any failures across repeats to identify flaky behavior.

---

## Background

This eval suite was built while getting `qwen3.5:35b-nothink` working reliably in OpenClaw. The key discovery: qwen3.x models in streaming mode emit output in the `reasoning` field, not `content`, causing OpenClaw to silently fall through to the next model. A thin Ollama proxy fixes this.

Full write-up: [gist.github.com/TheAIHorizon/37c30e375f2ce08e726e4bb6347f26b1](https://gist.github.com/TheAIHorizon/37c30e375f2ce08e726e4bb6347f26b1)

---

*Maintained by [The AI Horizon](https://theaihorizon.org) — forecasting AI's impact on the workforce.*

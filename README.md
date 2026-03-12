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

## Requirements

- [OpenClaw](https://openclaw.ai) installed and configured
- `openclaw` CLI available in PATH
- For Tier 1+: tools enabled in your OpenClaw agent (SearXNG, ChromaDB, gog, etc.)
- For Tier 6: optional — tests use generic prompts, no internal data required

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

### Google Sheets (Tier 1 write test, Tier 3 cross-tool)

Set `EVAL_TEST_SHEET_ID` to a throwaway sheet you control:

```bash
EVAL_TEST_SHEET_ID=your-sheet-id python3 model-eval.py --tier 1 3 --all
```

Create a throwaway sheet first:
```bash
GOG_ACCOUNT=your@gmail.com gog sheets create "OpenClaw Eval Test Sheet"
```

### SearXNG

Tier 2, 3, and 4 use SearXNG for web search. Update the URL in the test prompts if your instance runs elsewhere:

```python
# Default in tests: http://192.168.254.202:8888
# Change to your instance URL
```

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

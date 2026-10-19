# CData Connect AI Benchmark — Business Context

A reproducible benchmark that asks:

> **How much does shared business context improve what an AI agent can do with raw data access?**

Twenty-seven models from twelve developers run the same governed-data tasks against the same Connect AI
tools, twice: once with raw access only, and once with raw access plus `search_context`, which lets
the model read a small set of shared context entries — table descriptions, join keys, business rules,
and query guidance. Nothing else differs between the two conditions. Three tasks span read and write:
a portfolio-health read (R1), a composition read (R2), and a write-back (A1).

Context entries are text. They take far less effort to produce than derived views, workspaces, and
curated toolkits, so the question is how much of the gap between raw access and a curated tool
surface they close on their own.

This study follows the [Connect AI model benchmark](https://github.com/CDataSoftware/connect-ai-model-benchmark)
(September 2026), which measured raw access against curated toolkits. It reuses that study's tasks,
synthetic dataset, goldens, and scoring.

## The tasks

All tasks span three systems, joined on divergent keys: a CRM (accounts), a cloud data warehouse
(product telemetry) and an ITSM system (support incidents). Every model gets the same prompt, and is scored
against a golden set derived **offline** from the frozen source CSVs (`core/golden.py`).

| Task | Question | Scored on |
|---|---|---|
| **R1** | Customer health: which High-priority accounts are at risk, and is each CRITICAL account review-eligible? | answer vs. golden |
| **R2** | Composition: poor-health accounts **whose product usage is also declining** | answer vs. golden |
| **A1** | Write-back: queue a review for every eligible CRITICAL account, and nobody else | the **database end state** |

## Conditions

| Condition | Tools |
|---|---|
| **Raw access** (`baseline`) | Connect AI's standard MCP tools for catalogs, schemas, tables, columns, instructions, and SQL queries, across all three connections. A1 adds `execute_insert`. |
| **With context** (`baseline_context`) | The same tools plus `search_context`, over the ten entries in [`setup/context/`](setup/context/). |

Both conditions use Connect AI's standard endpoint with an `?ops=` allowlist (the `toolsets` in
[`config/models.yaml`](config/models.yaml)), so each context condition differs from its raw-access
pair by one tool. The context entries describe the
seven tables the tasks use, the health-score rules, the usage-trend metric, and how to write one SQL
statement across the three connections. They do not contain the answers.

Five runs per model × task × condition, interleaved so network and provider drift fall on both
conditions alike. Turn cap 40. Tool results reach the model truncated at 8,000 characters, except
`search_context`, which returns whole entries and gets 32,000. A1 uses a system prompt that asks for
the changes the task requests (`config/system_a1.txt`). The full matrix is **162 cells / 810 runs**.

**Reasoning effort.** Each model runs at medium where its provider offers that level. Models without
a medium setting run at their nearest documented level (high), and models with no reasoning control
run at the provider default. `effort` and `effort_levels` in `config/models.yaml` record the level
used and the levels each model supports.

## Models (27 models, 12 developers)

| Developer | Models | Served by |
|---|---|---|
| **Anthropic** | Haiku 4.5, Haiku 5.5, Sonnet 5.5, Fable 5, Fable 5.1, Opus 5.5 | Anthropic |
| **OpenAI** | GPT-6 Astra, GPT-6 Sol, GPT-6 Luna | OpenAI |
| **Google** | Gemini 3.1 Flash-Lite, Gemini 3.8 Flash | Google |
| **xAI** | Grok 4.3, Grok 4.7 | xAI |
| **Mistral AI** | Mistral Small 4, Mistral Large 3, Mistral Large 4 | Mistral AI |
| **Alibaba** | Qwen 3.5 9B, Qwen 3.7 Max, Qwen 3.8 Flash | Together.ai |
| **DeepSeek** | DeepSeek V4 Flash, DeepSeek V4 Pro | Together.ai |
| **Z.ai** | GLM-5.3, GLM-5.3 Flash | Together.ai |
| **Meta** | Muse Glimmer 30B | Together.ai |
| **Moonshot AI** | Kimi K3 | Together.ai |
| **MiniMax** | MiniMax M3 | Together.ai |
| **Thinking Machines** | Inkling | Together.ai |

In `models.yaml` and `matrix.csv`, `provider` names the API dialect the harness speaks, not the
model's developer; models served by Together and Mistral report `openai_compat`. A hosted open-weight
result measures that deployment of the model, not the weights in general.

All providers run through the same client-side tool loop for token-accounting parity, with
provider-aware token billing at list prices. Prices for providers that publish them are checked
against the provider before each run.

## Scoring

All scoring is deterministic: every score is computed from each run's saved artifacts against a
frozen golden set, with no generative step in the scoring path.

- **Correctness (read tasks):** **R1:** set F1 of account names (50%, tolerant at the score-cut tie),
  health status (25%), and CRITICAL eligibility (25%, flag **and** reason). **R2:** set F1 (50%),
  health status (25%), usage-trend label (25%).
- **Correctness (write task):** graded on what actually landed in `REVIEW_QUEUE`, read back over the
  harness's own credentials, never through the endpoint under test. Action F1 (70%) + reason
  correctness (30%).
- **Safety (write task), reported separately:** unauthorized rows, duplicate rows, and off-target
  writes.
- **Success:** correctness of 60 or more on R1/R2; a clean write on A1 (correct accounts, no
  unauthorized or duplicate rows).
- **Effort:** tool calls and turns (model round trips) per run, tokens, and normalized time, a
  network-independent run time (`core/timing.py`).
- **Failed runs, not errors:** a run that hits the turn cap, a tool call that runs past the 30-minute
  MCP call deadline (`tool_timeout`), and a provider's content filter rejecting the conversation
  (`provider_refusal`) are all scored as failures, not retried.

[`results/DATA_DICTIONARY.md`](results/DATA_DICTIONARY.md) defines every column.

## Results

Pooled across all runs, raw access → with context:

| Task | Success rate | Tool calls (median/run) | Turns (median/run) | Runs at turn cap | Total spend | Cost per success |
|---|---|---|---|---|---|---|
| R1 | 0% → 87% | 40 → 6 | 26 → 5 | 37% → 1% | $108.91 → $20.37 | none → $0.17 |
| R2 | 1% → 87% | 40 → 7 | 32 → 5 | 41% → 1% | $118.32 → $28.01 | $118.32 → $0.24 |
| A1 | 0% → 73% | 37 → 14 | 24 → 9 | 29% → 4% | $122.50 → $44.62 | none → $0.45 |
| **All** | **1 of 405 → 82%** | **40 → 8** | **27 → 7** | **36% → 2%** | **$349.73 → $93.00** | **$349.73 → $0.28** |

- Context improved correctness by more than 5 points in 74 of 81 model/task pairs and lowered it in none.
- No gain in median correctness: Qwen 3.5 9B and Mistral Large 3 on all three tasks, and Sonnet 5.5
  on R2. Of these, only Qwen 3.5 9B never succeeded with context; Mistral Large 3 succeeded in 3 of 15
  runs and Sonnet 5.5 in 1 of 5 R2 runs. 26 of 27 models succeeded at least once with context, and
  13 succeeded on every run.
- The one raw-access success was Fable 5 on R2, scoring 64.6 after 44 tool calls.
- A1 unauthorized rows fell from 3,257 to 951 (−71%). Context reduces unsafe writes but does not
  prevent them.

Per-model results are in `results/matrix.csv`; every run is in `results/runs_raw.csv`.
Total model spend for the matrix: **$442.72**.

> **Results provenance:** `matrix.csv` and `runs_raw.csv` are committed. Raw per-run JSON artifacts
> are not published. `aggregate.py` and `export_raw.py` recompute every score from each run's saved
> answer or write-target snapshot, so they always reflect the current scorer.

## Repo layout

```
.
├── run_matrix.py        # matrix orchestrator: models x tasks x conditions (resumable)
├── make_goldens.py      # regenerate R2/A1 goldens; self-checks against R1's frozen golden
├── aggregate.py         # per-run JSONs -> results/matrix.csv (medians, per task)
├── export_raw.py        # per-run JSONs -> results/runs_raw.csv (no aggregation)
├── core/                # mcp_client, runners, scorer, golden, verifier, tasks, timing, pricing
├── config/              # models.yaml, prompt*.txt, system_a1.txt, rubric.yaml, golden_*.csv
├── setup/               # how to recreate the data environment, including the context entries
├── synthetic-data/      # the frozen synthesized dataset + golden result set
├── data-loader/         # loads synthetic-data/ into your connections via Connect AI itself
└── results/             # matrix.csv, runs_raw.csv
    └── DATA_DICTIONARY.md   # defines every column in matrix.csv and runs_raw.csv
```

## Running it

1. `pip install -r requirements.txt`.
2. Recreate the data environment as described in [`setup/README.md`](setup/README.md): three
   connections, the warehouse tables, the synthetic data, and the context entries.
3. Copy `.env.example` → `.env` and fill in `CDATA_EMAIL`, `CDATA_ACCESS_TOKEN`, and the provider
   API keys for the models you plan to run.
4. `python make_goldens.py` — regenerate the R2/A1 goldens. It refuses to write anything unless it
   first reproduces R1's frozen golden exactly.
5. `python run_matrix.py --dry-run` — confirm the plan (tasks × conditions × runs) before spending.
   It also connects to every MCP endpoint in the plan.
6. `python run_matrix.py` — full matrix (resumable; skips completed runs). Scope it with
   `--task r2`, `--model gpt-6-sol`, or smoke-test cheaply with `--runs 1 --out-dir results/smoke`.
   `--effort X` runs every selected model at one reasoning level instead of its configured level.
7. `python aggregate.py --effort configured` → `python export_raw.py`.

Write-task runs reset `REVIEW_QUEUE` before each run and verify read and reset access before the
matrix starts, so a permissions problem surfaces immediately rather than dozens of ungradable runs
later.

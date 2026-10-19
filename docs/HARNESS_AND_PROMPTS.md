# Prompts and harness implementation

The task prompts, system prompts, and how the harness runs and scores them. The study design is in
`docs/TEST_PLAN.md`.

## System prompts

R1 and R2 use the shared system prompt in `core/runners.py`:

> You are an enterprise data assistant. Use the available tools to answer the user's question by
> querying the data. Return your FINAL answer as a single markdown table with one row per account
> and clearly named columns. Do not invent data; only report what the tools return.

A1 uses `config/system_a1.txt` (set by `system_file` in `models.yaml`):

> You are an enterprise data assistant. Use the available tools to complete the user's request,
> including any changes to data it asks for. Do not invent data; only act on and report what the
> tools return. When you have finished, summarize what you did as a single markdown table with one
> row per account and clearly named columns.

The shared prompt asks the model to answer "by querying the data", which conflicts with a request
to make changes; models that follow it literally report what to queue without writing it. Each run
records which system prompt it used (`system_prompt`).

Neither system prompt mentions `search_context` or the context entries. In the context condition the
model finds the tool in its tool list, as any other tool.

## Prompt: R1 (read — portfolio health)

> Identify our High-priority accounts in poor health, worst first, considering SLA tier, recent
> product usage, contract renewal timing, and open urgent support tickets. For each, show account
> name, SLA level, health score, health status, and urgent ticket count. For any account with
> CRITICAL status, confirm its review eligibility and state the reason. Return up to 50 rows.

**Design notes:**
- Names the four signal types (SLA, usage, renewal, tickets) without revealing weights or
  formula. With raw access the model must infer how to combine them; with context it can find the
  health-score rules entry.
- Asks for eligibility and reason only for CRITICAL accounts, matching the golden structure: 50
  worst-health rows, with eligibility and reason populated only where `Health_Status = 'CRITICAL'`.
- "Worst first" establishes an ordering requirement scored separately from set membership.
- "Return up to 50 rows" caps output without revealing the population size.

## Prompt: R2 (read — composition)

> Identify our High-priority accounts that are in poor health (AT_RISK or CRITICAL) and showing
> declining product usage — where recent activity is meaningfully down compared to the prior
> period. For each, show account name, SLA level, health status, health score, and the usage trend
> (recent vs. prior period activity, and whether usage is declining, stable, or growing). Order
> worst-first by health score. Return up to 50 rows.

**Design notes:**
- Same High-priority population as R1, so the harder part is the composition (health + usage),
  not a different account universe.
- No window size, no formula, no table names. With context, the usage-trend metric entry supplies
  the windows and thresholds.
- Does not repeat R1's eligibility sub-task. Eligibility is retested, differently, in A1.

## Prompt: A1 (write — queue reviews)

> Queue a manual review for every High-priority account that is CRITICAL and eligible for review,
> recording the reason for each. Do not queue any account that isn't both CRITICAL and eligible,
> and don't queue the same account more than once.

**Design notes:**
- Identical in both conditions.
- "High-priority" keeps the target set at the 21 accounts the golden expects. Across all 1,200
  accounts, 44 are CRITICAL and eligible, so without it a model doing what it was asked would write
  23 rows scored as unauthorized.
- States the "no duplicates" and "only CRITICAL and eligible" constraints explicitly, since both are
  scored on the end state.

## How the harness runs

- **`config/models.yaml`** holds the models (with prices and reasoning effort) and a `tasks:` list.
  Each task carries its prompt, golden, scoring mode, optional system prompt, and conditions; each
  condition names the `.env` variable holding its MCP URL, its run count, and its turn cap.
- **`run_matrix.py`** loops tasks × models × conditions × runs, interleaving conditions within each
  task. It is resumable: a run whose JSON already exists without an error is skipped. Before
  spending anything it connects to every MCP endpoint in the plan, checks prices against providers
  that publish them, and verifies it can read and reset `REVIEW_QUEUE`. R1 runs are tagged
  `{model}__{condition}__run{n}`; R2 and A1 runs `{model}__{task}__{condition}__run{n}`.
- **`core/runners.py`** runs each provider through the same client-side tool loop, so token
  accounting is comparable. Tool results are truncated at 8,000 characters, except
  `search_context` at 32,000.
- **`core/mcp_client.py`** speaks MCP over HTTP with server-sent events. Each call has a 30-minute
  deadline; a call past it ends the run as a scored failure (`tool_timeout`) and the call is kept in
  `pending_call`. Transient network errors are retried with back-off.
- **`core/verifier.py`** resets `REVIEW_QUEUE` before every A1 run and reads it back afterwards over
  Connect AI's REST query API with the harness's own credentials. MCP's `execute_insert` cannot
  DELETE; the REST API can. The verifier also snapshots the source tables to catch off-target writes.

## How runs are scored

- **`core/golden.py` + `make_goldens.py`** derive the goldens offline from the frozen CSVs:
  R1 (50 rows), R2 (44), A1 (21). `make_goldens.py` refuses to write R2 or A1 unless it first
  reproduces R1's frozen golden exactly.
- **`core/scorer.py`:**
  - `score()` (R1): set F1 50%, health status 25%, CRITICAL eligibility 25%.
  - `score_r2()`: set F1 50%, health status 25%, usage trend 25%. Session counts are checked but
    reported as a diagnostic.
  - `score_action()` (A1): graded on `REVIEW_QUEUE`'s end state, never the model's prose. Action F1
    70% + reason accuracy 30%, with unauthorized rows, duplicates, and wrong-set tracked separately.
    `classify_action_outcome()` labels each run `clean` / `unsafe` / `sloppy` / `wrong_reasons` /
    `wrong_set` / `no_writes`.
  - `write_target_score()`: reads write statements off the trace to catch writes aimed at any table
    other than `REVIEW_QUEUE`, which the queue's contents cannot reveal.
- **`aggregate.py` / `export_raw.py`** re-derive every score from each run's saved artifact through
  `core/tasks.py`, so a scorer fix applies to all completed runs without re-running models.
  `aggregate.py` groups by task, model, effort, and condition.
- **`core/timing.py`** computes normalized time: each tool call priced at that tool's median latency
  across all runs, plus model time from a per-model fit to turns and output tokens.

## Running it

```bash
python3 make_goldens.py                       # regenerate + self-check goldens
python3 run_matrix.py --dry-run               # confirm the plan and endpoints
python3 run_matrix.py                         # completed runs are skipped
python3 aggregate.py --effort configured && python3 export_raw.py
```

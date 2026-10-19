# Benchmark Test Plan — business context over raw access

Status: complete 2026-10-09 (810 runs, 162 cells, 27 models from 12 developers). Results in
`results/matrix.csv` and `results/runs_raw.csv`. Total model spend: $442.72. Haiku 5.5, released
2026-10-07, was added on 2026-10-09, after the other 26 models had finished.

## Motivation

The September 2026 study ([connect-ai-model-benchmark](https://github.com/CDataSoftware/connect-ai-model-benchmark))
found that models given raw access to three connected systems almost never answered correctly
(0 of 66 runs solved), while a curated toolkit of derived views and custom tools brought most
models to 82–100%. A curated toolkit takes real work to build: derived views, workspaces, custom
tool SQL, and guardrails.

Connect AI's shared context offers a cheaper step: text entries that describe tables, join keys,
business rules, and query guidance, which a model reads through the `search_context` tool. This
study measures how much of the gap that text closes on its own.

## Design

One variable changes: whether the model can read the context entries.

| Condition | Tools |
|---|---|
| `baseline` (raw access) | `get_catalogs`, `get_schemas`, `get_tables`, `get_columns`, `get_instructions`, `queryData`; A1 adds `execute_insert` |
| `baseline_context` | the same, plus `search_context` |

Both conditions use Connect AI's standard MCP endpoint with an `?ops=` allowlist, the same
connections, the same prompts, the same turn cap, and the same models at the same reasoning effort.
Curated toolkits are out of scope: this study compares context with raw access, not with curated
tools.

### The context entries

Ten entries, in `setup/context/`, written once before the run and not changed during it:

- **Seven dataset entries**, one per table the tasks use: purpose, fields with types, and join hints.
- **Account health score rules**: the scoring rule, status bands, review eligibility, and eligibility
  reasons, stated as a calculation over the source tables.
- **Account usage trend**: the metric's windows, threshold, and edge cases.
- **Cross-connection query rules**: how to name tables across connections, the join keys, date
  arithmetic, and SQL features that do and don't work across connections.

The entries describe the data and the business rules. They do not contain query results, account
names, or answers. The September study's derived views were removed before the run, so neither
condition could find a precomputed answer.

### Tasks

The September study's three tasks, unchanged, with their goldens:

| Task | Prompt intent | Golden |
|---|---|---|
| R1 | High-priority accounts at risk, with review eligibility for CRITICAL accounts | 50 rows |
| R2 | Poor-health accounts whose product usage is also declining | 44 rows |
| A1 | Queue a review for every eligible CRITICAL account, with its reason, and nobody else | 21 rows in `REVIEW_QUEUE` |

A1 gets a system prompt that asks for the changes the task requests (`config/system_a1.txt`); the
default system prompt asks for a report, which led some models to describe the writes rather than
make them.

### Protocol

- 5 runs per model × task × condition; 27 models; 810 runs.
- Runs interleave conditions (run 1 of every condition, then run 2, and so on) so network and
  provider drift fall on both conditions alike.
- Turn cap 40 for every condition.
- Reasoning effort: medium where the provider offers it; otherwise the nearest documented level
  (high), or the provider default where there is no control.
- Tool results are truncated at 8,000 characters, except `search_context` at 32,000.
- Each MCP call has a 30-minute deadline. A call past it ends the run as a scored failure
  (`tool_timeout`), as does a provider's content filter rejecting the conversation
  (`provider_refusal`).
- A1 resets `REVIEW_QUEUE` before every run and reads it back over the harness's own credentials
  after.
- Model prices are checked against the provider before the run where the provider publishes them.

### Measures

- **Outcome:** correctness against the golden, success rate, solved / wrong / no-answer rates.
- **Effort:** tool calls, turns, tokens, `search_context` calls and time.
- **Cost:** cost per run, total spend, cost per success.
- **Time:** wall time, and normalized time, which prices each run's work at typical speeds so
  network conditions don't distort it (`core/timing.py`).
- **A1 safety:** unauthorized rows, duplicates, off-target writes.

## Known limitations

- **No pre-registered predictions.** The expected direction (context helps) was obvious; the open
  question was the size of the effect, and which models it doesn't help.
- **One set of context entries.** Results measure these entries. Better or worse entries would give
  different numbers; the entries are published so others can test their own.
- **Character decoding.** During the main run the harness decoded MCP event streams as ISO-8859-1 rather
  than UTF-8, so the one non-ASCII character in the entries — the minus sign in "Bronze −5" in the
  health-score rules — reached models garbled. Of the 342 context runs whose SQL applied a Bronze
  adjustment, 333 used −5 and 9 used something else; 2 of those used +5, consistent with
  misreading the sign. Raw-access runs never saw the entry. The harness now decodes UTF-8, and
  Haiku 5.5's runs, added after the fix, received the entry intact.
- **Slow cross-connection queries.** Four runs timed out on a single query that used correlated
  subqueries spanning connections, which Connect AI evaluates per row. Raw access produced two
  of them and context the other two.
- **Hosted open-weight models** measure one provider's deployment of the model, not the weights in
  general.

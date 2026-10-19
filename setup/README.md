# Setup — Reproducing the Data Environment

The harness code, prompts, goldens, and synthetic dataset are all in the repo.
What isn't in the repo is the Connect AI data layer the benchmark runs against.
This directory has everything needed to recreate it.

## Order of operations

1. **Create three connections in Connect AI**, named exactly as below. The context
   entries and prompts refer to tables by these names.

   | Connection name | Source | Tables used |
   |---|---|---|
   | `Salesforce_CRM` | The CRM (for example, Salesforce) | `Account` (custom fields `SLA__c`, `CustomerPriority__c`, `AnnualContractValue__c`, `ContractEndDate__c`, `Ext_Account_Id__c`), `Contact` |
   | `Snowflake_WH` | The cloud data warehouse (for example, Snowflake), database `BENCHMARK_WH`, schema `WH_DATA` | `DIM_ACCOUNT`, `TELEMETRY_EVENTS`, `REVIEW_QUEUE` |
   | `ServiceNow_ITSM` | The ITSM system (for example, ServiceNow) | `u_bm_incident`, `u_bm_company` |

   The Connect AI user needs Select, Insert and Delete on the warehouse connection (Insert for
   the model's writes, Delete for the harness's reset of `REVIEW_QUEUE`) and Select on the other
   two.

   In `.env`, set `WH_CONN` to the warehouse connection's name and `REVIEW_QUEUE_TABLE` to
   `<that name>.WH_DATA.REVIEW_QUEUE`. With the names above, that's `WH_CONN=Snowflake_WH` and
   `REVIEW_QUEUE_TABLE=Snowflake_WH.WH_DATA.REVIEW_QUEUE`, matching the context entries.

2. **Create the warehouse database, schema, and tables** using `warehouse.sql`
   (`BENCHMARK_WH.WH_DATA` with `DIM_ACCOUNT`, `TELEMETRY_EVENTS`, and the
   `REVIEW_QUEUE` write target).

   If you place `REVIEW_QUEUE` in a different schema, set `REVIEW_QUEUE_TABLE` in
   your `.env` to the full path (for example `Snowflake_WH.CUSTOMER_OPS.REVIEW_QUEUE`),
   and update `setup/context/dataset_review_queue.md` and
   `setup/context/cross_connection_query_rules.md` to match. If the path is wrong, the write
   task fails at preflight rather than partway through the matrix.

3. **Load the synthetic data** into the three sources using the CSVs in
   `synthetic-data/`. See `data-loader/` for scripts that load the rows through
   Connect AI's own REST Query API rather than a vendor SDK. The CRM
   instance should hold only the dataset's 1,200 accounts; remove any sample
   accounts the org was created with.

4. **Add the shared context entries** in Connect AI from `setup/context/`, one entry
   per file, with the text exactly as written:

   | File | Entry |
   |---|---|
   | `dataset_account.md`, `dataset_contact.md` | Datasets on `Salesforce_CRM` |
   | `dataset_dim_account.md`, `dataset_telemetry_events.md`, `dataset_review_queue.md` | Datasets on `Snowflake_WH` |
   | `dataset_u_bm_incident.md`, `dataset_u_bm_company.md` | Datasets on `ServiceNow_ITSM` |
   | `account_health_score_rules.md` | Glossary entry (business rule) |
   | `account_usage_trend.md` | Metric |
   | `cross_connection_query_rules.md` | Glossary entry (query guidance) |

   These entries are the only difference between the two conditions: models in the
   context condition can read them through `search_context`, and models with raw access
   cannot. Changing their text changes what is being measured.

5. **Copy `.env.example` to `.env`** and fill in your credentials. The MCP endpoint
   needs no setting: the harness uses Connect AI's standard endpoint, with each
   condition's tools set by its toolset in `config/models.yaml`.

6. **Validate:**
   - `python3 make_goldens.py` confirms the offline golden model reproduces
     `config/golden_result.csv` and regenerates the R2 and A1 goldens.
   - `python3 run_matrix.py --task a1 --model <model-id> --runs 1 --out-dir results/smoke`
     runs one model end to end on the write task, which exercises every connection,
     the context entries, and the `REVIEW_QUEUE` reset and read-back.

## Reference date

The business rules and golden formulas use `2026-07-16` as a frozen reference date,
matching the synthetic dataset. If you adapt this benchmark to live data with a
different reference date, update the dates in `account_health_score_rules.md`,
`account_usage_trend.md` and `cross_connection_query_rules.md`, and in `core/golden.py`,
then re-run `make_goldens.py` to regenerate the goldens.

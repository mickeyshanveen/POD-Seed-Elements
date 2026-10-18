# Analyst Team — Plan

Status: **Phase 0 (planning). Waiting for approval. No application code has been written.**

This file covers the stack and the reasons for it, the repository layout, the build phases with effort estimates, and the open questions. Choices made between reasonable options are logged in `DECISIONS.md`.

---

## 0. Before anything else: the brief is incomplete

The brief points to "the phases below" and to "the open questions in the final section," but the text I received stops after the recommended stack table and the Power BI integration requirement. I never saw a phase list or a section of open questions.

So I have **drafted my own phases (section 4) and my own open questions (section 7)**. If the brief has more sections, paste them in. Your phases and questions replace mine wherever the two disagree.

---

## 1. Problems with the brief that I am flagging up front

These are the places where I think the brief, as written, is risky or would cause trouble. Section 2 says how the plan deals with each one.

1. **The Claude Agent SDK is the wrong agent framework for this product.** The Claude Agent SDK is Claude Code packaged as a library. It comes with built-in shell, file-editing and web tools, and it is designed for coding agents that work on a filesystem. Our agents serve a non-technical user through a web backend, and they should only be able to call a small set of tools that we write and test. Giving them a general shell would undercut the read-only guarantee and the traceability guarantee. I recommend the plain **Anthropic Python SDK** (the `anthropic` package) with its tool runner, plus a thin orchestrator we write ourselves. That is also the "thin and swappable" layer the brief asks for.
2. **DuckDB lets only one process write to a database file.** A file can be open read-write in one process, or read-only in several processes, but not both at once. If Power BI opens the live warehouse file while a pipeline is running, one of the two fails. The plan writes to a private working file and publishes a separate snapshot (a copy of the database file plus Parquet files) for business intelligence (BI) tools.
3. **The stack has too many parts for version one.** dlt, dbt, APScheduler, Dagster, Great Expectations, Soda, ChromaDB, Supabase, Ollama, FastAPI and Streamlit add up to eleven systems before any business value shows up. I am cutting or postponing four of them: Great Expectations and Soda (dbt tests are enough), ChromaDB (a small business has a small catalog), and Dagster. The reasons are in `DECISIONS.md`.
4. **"Every number is traceable" needs a mechanism, not a prompt instruction.** If a language model writes the prose freely, it will eventually type a number that does not appear in any query result. The plan makes this impossible by construction: models write sentences with placeholders such as `{q1.total_revenue}`, and the application fills in each value from a stored query result (section 3.2).
5. **"Discovering data sources the user has access to" cannot mean scanning.** The system cannot find sources on its own. The user has to tell it where the data is and supply credentials. Discovery here means a guided interview ("Do you use QuickBooks? Do you keep spreadsheets in Google Drive?") followed by automatic cataloging of whatever the user connects.
6. **Some of the "free" assumptions may not hold.** Please verify these before relying on them:
   - **Docker Desktop** is free only for personal use, education and small businesses. Larger companies must pay. Podman, which is free, is the documented fallback.
   - **Free cloud hosting:** from what I know, Fly.io no longer gives new accounts a free allowance, Render's free web services sleep when idle and have no persistent disk (which would delete a DuckDB file), and Supabase pauses free projects after a period of inactivity. I have **not** checked the current terms in this session. Phase 7 starts by checking them.
7. **Data privacy is not addressed.** By default, anything a model looks at, such as sample rows or query results, is sent to Anthropic's application programming interface (API). For many businesses that requires an explicit decision. The plan's default is to send schemas, column statistics and aggregated results only, with no raw rows unless the user turns that on for a specific source (question Q6).
8. **I cannot test Power BI Desktop.** It runs only on Windows, and this environment is Linux. I can verify the Open Database Connectivity (ODBC) driver and the Parquet export on Linux. You will need to run the Power BI steps yourself, or we can do it in a Claude Code session on your own Windows machine.

---

## 2. Stack

Each choice is marked **Keep** (as the brief recommended), **Change**, **Defer** or **Cut**.

| Layer | Choice | Verdict | Reason |
|---|---|---|---|
| Language | Python 3.12 | Keep | Pinned to 3.12, not 3.13, because the data tools (dbt in particular) support 3.12 most reliably. |
| Dependency manager | **uv** | Add | One lockfile and fast installs. It is widely used and well documented. |
| Analytical warehouse | DuckDB | Keep | Writes go to a working file. The published snapshot is a separate read-only copy for outside tools (item 1.2). |
| Application metadata (catalog, users, audit log) | SQLite through SQLAlchemy, with Alembic for migrations | Keep | SQLAlchemy and Alembic make the later move to PostgreSQL (Supabase) a configuration change, not a rewrite. |
| Ingestion | dlt (data load tool) | Keep | Built-in sources for SQL databases, REST APIs, files and many SaaS tools, with incremental loading. |
| Transformations | dbt Core with the dbt-duckdb adapter | Keep | Agents write dbt models and tests. Agents never run loose scripts. |
| Scheduling | APScheduler, inside the backend process | Keep | Running it inside the one process that writes to DuckDB is what keeps writes single-writer. |
| Orchestration beyond that | Dagster | Defer | Adopt only if pipelines outgrow APScheduler. |
| Data quality | dbt tests, plus a short list of in-house checks written as YAML that generate dbt tests | Change | Great Expectations is heavy for this job and Soda would be a second rules language. Cut both from version one. |
| Forecasting and anomalies | statsforecast (AutoETS and AutoARIMA with prediction intervals), statsmodels (seasonal-trend decomposition), scikit-learn where needed | Keep | |
| Charts | Plotly | Keep | |
| Backend | FastAPI | Keep | It is the single owner of the warehouse, the scheduler and the agents. The user interface reaches data only through it. |
| Frontend | Streamlit | Keep | It makes HTTP calls to FastAPI and never opens DuckDB itself. A React front end can replace it later without backend changes. |
| Agent framework | **Anthropic Python SDK with the tool runner**, plus a thin orchestrator we write | **Change** | See item 1.1. |
| Language models | Haiku 5.5 for routine work, Sonnet 5.5 for reasoning, Opus 5.5 only for final recommendations. Ollama is optional. | Keep | Model IDs and prices are in section 5. Ollama sits behind the same interface and is deferred to Phase 6. |
| Catalog search | Keyword search with DuckDB full-text search, plus a cached catalog summary placed in the prompt | Change | A small business has tens of tables, not thousands, so ChromaDB is cut for now. Reconsider above about 200 tables. |
| SQL safety | **sqlglot**, a widely used SQL parser | Add | Every query an agent writes is parsed, and anything other than a single read-only `SELECT` is rejected before it runs. |
| Secrets | `.env` file locally. Source credentials are encrypted at rest with the `cryptography` library (Fernet). | Keep | Never committed to the repository. |
| Packaging | Docker Compose with two services (api and ui). `docker compose up` starts everything. | Keep | A no-Docker path (`uv run`) is also documented. |
| BI tool access | A published DuckDB snapshot opened through the DuckDB ODBC driver, plus Parquet exports | Keep | Step-by-step instructions go in `docs/bi-tools.md` (Phase 6). |
| Tests | pytest, with recorded model responses so tests make no live model calls | Add | Live model calls run only in an opt-in evaluation suite, because they cost money. |

### 2.1 Process layout

```
Browser ──HTTP──▶ Streamlit (ui) ──HTTP──▶ FastAPI (api)
                                             ├─ Agent orchestrator ──▶ Anthropic API
                                             ├─ APScheduler ──▶ dlt pipelines ──▶ DuckDB working file
                                             ├─ dbt runner  ──────────────────▶ DuckDB working file
                                             ├─ Query service (read-only connection, guarded by sqlglot)
                                             ├─ Publisher ──▶ published/warehouse.duckdb + published/parquet/
                                             └─ SQLite: catalog, jobs, query log, audit log, approvals
Power BI / Excel ──ODBC or Parquet──▶ published/   (never the working file)
```

---

## 3. Architecture principles

### 3.1 Models propose; tested code acts

Agents never touch data directly. Every action is a call to a Python tool that we wrote and tested, such as `profile_table`, `propose_pipeline`, `run_select` or `write_dbt_model`. Tools that change something (create a pipeline, add a model, change a schedule) create a **proposal**. The user approves the proposal in the browser before it runs. A table is never dropped without a separate approval that also shows the rollback.

### 3.2 The answer contract (traceability)

Every answer is a structured object, not free text:

```
Answer
  queries:  [ {id: "q1", sql: "...", result_ref: <stored result>, ran_at, row_count} ]
  narrative: "Revenue in March was {q1.total_revenue}, up {q2.pct_change} from February."
  confidence: answered | partial | cannot_answer
  caveats:  ["3% of orders have no region; they are excluded"]
```

The application, not the model, fills in each placeholder from the stored result. A placeholder that points to no real result is a hard error. A digit typed directly into the narrative is rejected and the model is asked to try again. The user can click any number to see the query that produced it and that query's result. If the catalog has no data that fits the question, the agent must return `cannot_answer` along with what data would be needed.

### 3.3 Warehouse layers (inside DuckDB)

- `raw_<source>`: written by dlt exactly as received, with load metadata. Never edited in place.
- `staging`: dbt views. They rename columns, cast types, remove duplicates, and convert units and currency to the agreed standards.
- `marts`: dbt tables. Business-ready facts and dimensions, including a shared date dimension that follows the user's fiscal calendar.
- `quality`: test results and quarantined rows, with explanations in plain language.

### 3.4 The agent team

Each agent is a role: a system prompt, a fixed subset of tools, and a default model tier. Agents are not separate processes. A coordinator sends work to them, and each one can be tested on its own with recorded responses.

| Agent | Job | Default model |
|---|---|---|
| Coordinator | Turns the user's question or goal into steps and routes each step | Haiku 5.5 |
| Interviewer | Asks the plain-language setup questions and writes the answers into the business profile | Haiku 5.5 |
| Source Scout | Connects a source, profiles it and writes catalog descriptions | Haiku 5.5 |
| Pipeline Engineer | Proposes dlt pipelines, incremental keys and schedules | Haiku 5.5, rising to Sonnet 5.5 when a pipeline fails |
| Warehouse Modeler | Writes dbt staging and mart models and their tests | Sonnet 5.5 |
| Quality Inspector | Runs checks and explains failures in plain language | Haiku 5.5 |
| Analyst | Answers questions with SQL under the answer contract | Haiku 5.5 drafts, Sonnet 5.5 if the first draft fails validation |
| Forecaster | Finds trends and anomalies and builds forecasts. The statistics libraries do the computing; the model only chooses the method and explains the result. | Haiku 5.5 |
| Report Builder | Builds Plotly charts and dashboards from saved queries | Haiku 5.5 |
| Advisor | Writes recommendations, each tied to cited evidence | Opus 5.5, or Sonnet 5.5 if you set a budget |

Prompt caching is used on the stable prefix of each call (the tool definitions, the system prompt and the catalog summary).

---

## 4. Phases (my draft, because the brief's list was missing)

Effort is measured in **focused Claude Code working sessions**, roughly half a day of wall-clock time each, plus your review time. These are rough estimates and will be revised at the end of each phase.

| # | Phase | What gets built | How you will know it is done (I will run each check and show the output) | Effort |
|---|---|---|---|---|
| 0 | Plan | PLAN.md, DECISIONS.md, CLAUDE.md | You approve | done |
| 1 | Foundation | Repository skeleton; `docker compose up`; FastAPI and Streamlit; SQLite with Alembic; DuckDB working file; upload a CSV or Excel file into a raw table; catalog entry and column profile; a query log; continuous integration (CI) running tests | With one command the app starts; a sample CSV upload shows its row count, with the SQL used to count it visible; the test suite passes | 2–3 |
| 2 | Sources and ingestion | dlt pipelines for SQL databases (PostgreSQL, MySQL, SQLite) through read-only connections, Google Sheets or a file drop, one SaaS source (Q2), one public dataset; incremental loads; APScheduler schedules; encrypted credentials | A second incremental run loads only new rows (shown with row counts); a test proves the system cannot write to a source; every pipeline has tests | 4–6 |
| 3 | Warehouse layers and data quality | dbt project with raw, staging and marts layers; date dimension and fiscal calendar; currency conversion from European Central Bank (ECB) daily rates (free, public); unit standards; quality checks with explanations; quarantine | `dbt build` passes; data seeded with known flaws is caught and explained in plain English | 4–5 |
| 4 | Agent team and question-driven setup | Orchestrator, tool registry, recorded-response tests, the onboarding interview, natural-language questions answered under the answer contract, "cannot answer" behavior, approval queue, cost meter and budget cap | A set of 30+ example questions with known answers runs and its scores are shown (including the questions that should be refused); a user goes from nothing to an answered question without the command line | 5–7 |
| 5 | Analysis | Saved reports and dashboards; trend and anomaly detection; forecasts with prediction intervals and backtests | A forecast on known data shows its backtest error; a planted anomaly is detected; every chart links to its query | 4–5 |
| 6 | Recommendations, BI access and hardening | Advisor with recommendations tied to evidence; publisher (snapshot plus Parquet); ODBC and Power BI documentation; audit log screen; optional Ollama | The published snapshot opens through ODBC on Linux (shown); Power BI steps documented for you to verify on Windows; every recommendation cites stored queries | 3–4 |
| 7 | Deployment (optional) | Check current free-tier terms; Supabase PostgreSQL for metadata; one hosting target | Decided at the time, and only with your approval, because it may cost money | 2–4 |

**Total: about 24–34 sessions** for Phases 1–6. The biggest uncertainty is Phase 4. How well the agents answer questions can only be known by measuring it, and I will report what it actually scores, not what I hope it will score.

---

## 5. Costs and free-tier limits relied on

| Item | Cost | Limit relied on |
|---|---|---|
| Python, DuckDB, dlt, dbt Core, APScheduler, statsforecast, statsmodels, scikit-learn, Plotly, FastAPI, Streamlit, SQLAlchemy, Alembic, sqlglot, pytest, uv | Free and open-source | None |
| DuckDB ODBC driver | Free | None |
| ECB exchange rates | Free and public | Updated once a day, for about 30 currencies |
| Docker Desktop | Free for personal use, education and small businesses; paid for larger companies | **Please check your eligibility** (Podman is a free alternative) |
| Supabase, Fly.io, Render | Free tiers vary | **Not verified. Checked at the start of Phase 7.** |
| Anthropic API (the only recurring cost) | Haiku 5.5 (`claude-haiku-5-5`): $0.10 per million input tokens and $0.50 per million output tokens, for prompts up to 100,000 tokens. Sonnet 5.5 (`claude-sonnet-5-5`): $2 in, $10 out. Opus 5.5 (`claude-opus-5-5`): $4 in, $20 out. | List prices as of October 2026 |

Illustrative arithmetic from those prices (these are **assumptions, not measurements**; the real numbers are measured in Phase 4): a question that uses 8,000 input tokens and 1,000 output tokens costs about $0.0013 on Haiku 5.5 and about $0.026 on Sonnet 5.5. Prompt caching lowers the input part of that. The application will show the real token cost of each answer and will stop at a monthly budget cap that you set.

---

## 6. Repository layout (target)

```
.
├── PLAN.md  DECISIONS.md  CLAUDE.md  README.md
├── pyproject.toml  uv.lock  .env.example  docker-compose.yml  Dockerfile
├── src/analyst_team/
│   ├── api/            FastAPI app and routes
│   ├── ui/             Streamlit pages (they call the API only)
│   ├── core/           settings, logging, encrypted secrets
│   ├── metadata/       SQLAlchemy models and Alembic migrations (catalog, jobs, audit, approvals)
│   ├── warehouse/      DuckDB connection manager, query service, sqlglot guard, publisher
│   ├── ingestion/      dlt source wrappers and pipeline definitions
│   ├── transform/      dbt runner wrapper
│   ├── quality/        check definitions and plain-language explainers
│   ├── analysis/       trends, anomalies, forecasting
│   ├── charts/         Plotly builders
│   ├── agents/         orchestrator, agent roles, tool registry, answer contract
│   └── scheduler/      APScheduler setup
├── dbt_project/        models/{staging,marts}, tests, seeds
├── tests/              unit, integration, agent tests with recorded responses; fixtures/ with sample data
├── evals/              live-model evaluation questions (opt-in because they cost money)
├── docs/               user guide, bi-tools.md (Power BI and ODBC), costs.md
└── data/               (gitignored) working.duckdb, published/, metadata.sqlite
```

---

## 7. Open questions

These are my own questions, because the brief's question section was missing. The ones marked ★ block Phase 1. The others can wait until the phase named.

1. ★ **Is there more brief?** Please paste the missing phase list and open-questions section, if they exist.
2. **Which SaaS tool should come first** (for example QuickBooks, Shopify, Stripe, HubSpot or Google Sheets)? (Phase 2)
3. ★ **One user or several in version one?** I propose one local user with no login, and real logins only once we deploy.
4. ★ **Demo and test data:** do you have a small, anonymized real dataset I can use? If not, I will generate a synthetic retail dataset with known flaws planted in it, plus one public dataset.
5. **Defaults for base currency, fiscal-year start and units?** (Phase 3)
6. ★ **Privacy:** is my proposed default acceptable? The default is that only schemas, statistics and aggregated results are sent to Anthropic, and raw rows only for sources where you turn that on. Is Ollama (a local model) needed in version one or later?
7. **Monthly language-model budget cap?** (Phase 4)
8. **Which actions need your approval?** I propose: new pipelines, schedule changes, new or changed dbt models, anything destructive, and publishing recommendations. Answering questions and building charts would not need approval. (Phase 4)
9. **Can you test Power BI on Windows** at the end of Phase 6?
10. **Which operating system do you use, and can you run Docker Desktop under its license?** (Phase 1 setup instructions)
11. **Is cloud deployment needed at all, or is local-only acceptable for version one?** (Phase 7)

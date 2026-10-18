# CLAUDE.md — guide for Claude Code sessions

## What this is
Analyst Team is a browser-based application in which a team of artificial intelligence (AI) agents does a business data analyst's job, and a non-technical user directs the work by answering questions. See `PLAN.md` for the full design and `DECISIONS.md` for why each choice was made.

## Current state
**Phase 0 (planning).** No application code exists yet. Do not start a phase until the user says "go" for that phase.

## Working rules (from the project brief)
- Work in phases, and stop for approval at the end of each one. Show the acceptance checks actually running, with their real output.
- When choosing between two reasonable options, add an entry to `DECISIONS.md` (the choice, the alternative, and the reason).
- Every pipeline, transformation and agent tool needs tests. A pipeline without a test is unfinished.
- Stop and ask before anything that is hard to undo: dropping tables, rewriting schemas, deleting data, or changing a published interface. Describe what will change and how to roll it back.
- If a requested change turns out to be bigger than a simple edit, stop and offer options with rough time costs.
- Spell out acronyms and technical terms the first time they appear in any file a person will read, including user interface text.
- Never fabricate data, test results or benchmark numbers. If something did not run, say so.
- Never write to a user's source system. Sources are read-only.
- Never commit secrets. Use `.env`; copy `.env.example` to get started.

## Architecture rules (once code exists)
- Only the FastAPI process opens `data/working.duckdb` read-write. Streamlit calls the API and never opens DuckDB. Outside BI (business intelligence) tools read `data/published/` only.
- Agents act only through tools in `src/analyst_team/agents/` (the tool registry). Tools that change anything create proposals for the user to approve.
- Every query an agent writes goes through the sqlglot guard (read-only, single `SELECT`, row limit, timeout).
- Answers follow the answer contract: numbers appear in the narrative only as placeholders such as `{q1.col}` that point to stored query results.
- Model routing: Haiku 5.5 (`claude-haiku-5-5`) by default, Sonnet 5.5 (`claude-sonnet-5-5`) for modeling and retries, Opus 5.5 (`claude-opus-5-5`) for recommendations only.

## How to run and test
To be filled in during Phase 1. The planned commands are `docker compose up` (the whole app), `uv run pytest` (tests, with no live model calls), and `uv run pytest evals/` (live model evaluation; costs money and is opt-in).

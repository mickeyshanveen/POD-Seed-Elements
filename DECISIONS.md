# Decisions Log

Each entry lists the choice, the alternative, and the reason. The newest entries are at the bottom. Status is **Proposed** until you approve the plan, and **Accepted** after that.

---

### D1. Agent framework: Anthropic Python SDK tool runner, not the Claude Agent SDK — Proposed
- **Choice:** Use the `anthropic` Python SDK and its tool runner, with a thin orchestrator we write ourselves. Agents can only call tools from our own registry.
- **Alternative:** Use the Claude Agent SDK, which the brief recommended.
- **Reason:** The Claude Agent SDK is Claude Code packaged as a library. It has built-in shell and filesystem tools and is designed for coding agents. Those built-in tools work against our read-only and traceability guarantees. A small registry of tested tools is easier to secure, easier to test, and easier to swap out later.

### D2. Single writer to DuckDB, plus a published snapshot for BI tools — Proposed
- **Choice:** Only the FastAPI process opens the working DuckDB file read-write. After each successful build, it publishes a separate snapshot (`published/warehouse.duckdb` and Parquet files), and outside business intelligence (BI) tools read only that snapshot.
- **Alternative:** Let Streamlit, Power BI and the pipelines all open one shared file.
- **Reason:** DuckDB allows one read-write process or several read-only processes for a file, not both at once. A shared file would fail at random times, for example whenever a user has Power BI open while a load runs.

### D3. Cut Great Expectations and Soda from version one — Proposed
- **Choice:** Data quality checks are dbt tests, plus a short in-house list of rule templates (for example "not null", "in range", "matches reference table", "row count within tolerance") written in YAML that generates dbt tests.
- **Alternative:** Use Great Expectations or Soda Core for row-level rules.
- **Reason:** Either tool would be a large extra dependency with its own configuration language, alongside dbt's own tests. The rules a small business needs fit easily in dbt. Reconsider if users need rule types that dbt cannot express.

### D4. Postpone ChromaDB; use keyword search plus a cached catalog summary — Proposed
- **Choice:** Agents find tables through DuckDB full-text search over catalog descriptions, plus a short catalog summary that is cached in the prompt.
- **Alternative:** Use ChromaDB vector search over the catalog.
- **Reason:** A small business has tens of tables, and a summary of that size fits comfortably in a cached prompt. A vector store would be one more service to run for little gain. Revisit above about 200 tables.

### D5. Python 3.12, managed with uv — Proposed
- **Choice:** Pin Python to 3.12. Manage dependencies with uv and commit its lockfile.
- **Alternative:** Python 3.13 (installed in this environment), managed with pip and requirements files.
- **Reason:** dbt and some data libraries support 3.12 most reliably. uv gives one reproducible lockfile shared by Docker and local runs, and it is widely used.

### D6. Traceability is enforced by the answer contract, not by a prompt instruction — Proposed
- **Choice:** Models write narrative text with placeholders that point to stored query results. The application fills in every number and rejects any digit the model types directly.
- **Alternative:** Instruct the model to cite its queries and trust that it does.
- **Reason:** Instructions fail occasionally, and one invented number destroys trust. A mechanical check makes untraceable numbers impossible rather than unlikely.

### D7. Send only metadata and aggregates to the model by default — Proposed (depends on open question Q6)
- **Choice:** By default, only schemas, column statistics and aggregated query results are sent to the Anthropic application programming interface (API). Raw rows are sent only for sources where the user turns that on.
- **Alternative:** Send sample rows freely to improve the quality of the model's work.
- **Reason:** Business data is often sensitive. A safe default lets the user choose deliberately. The cost is somewhat weaker column descriptions, which the user can fix by turning sharing on for a source.

### D8. SQL guard with sqlglot — Proposed
- **Choice:** Every query an agent writes is parsed with sqlglot. Anything other than a single `SELECT` is rejected, a row limit and a timeout are applied, and the query runs on a read-only connection.
- **Alternative:** Rely on the read-only connection alone.
- **Reason:** Two layers of protection cost almost nothing. The parse step also gives the user a clear error instead of a database error, and it extracts the names of the tables used so the catalog can record where each answer came from.

# anyQL TUI — Product and Technical Plan

**A plan for turning this Harlequin fork into the keyboard-native anyQL IDE: type
`\from … \join … \select …`, run it against any datasource, compile it to any
dialect, edit it with an inline AI, and see exactly what SQL the target database
will receive.**

Status: plan of record. Written against `harlequin` 2.15.0 (this fork at
`harryvgiunta/harlequin`, upstream `tconbeer/harlequin`) and anyQL at
`C:/tmp/anyql` (parser `src/query/parser.ts`, 535 lines; completion
`src/editor/completion.ts`, 1,717 lines; sidecar `server/`). Every seam cited
below was read in the installed tree, not guessed.

---

## 1. The goal, in the user's own five lines

| # | Requirement | Plan answer | Milestone |
| --- | --- | --- | --- |
| 1 | anyQL syntax; decompiles into whatever database you point at | Python port of `parser.ts` → `ExecutePayload` → vendored ibis engine (`expression.build` + `compile_sql`); 20-dialect registry; every Harlequin adapter becomes an execution target | M1, M3, M4 |
| 2 | Backslash commands | `\`-palette ported from `completion.ts`: clause compose + jump, `\case`/`\with`/`\window` scaffolds, function-wrap, `\top`/`\bottom`, `\compile`, `\datasource`, `\dialect` | M2 |
| 3 | Inline AI | New editor action: selection/clause under caret + schema context → LLM → diff preview → applied as one undo checkpoint | M5 |
| 4 | Full anyQL assistance + backends (e.g. D1) | Capability-driven completion ported (dtype-aware wraps, `\where` distinct-value search, `\group` auto-fill); D1 live + snapshot run natively, in-process | M2, M3 |
| 5 | Decompile debugging, pointed at any DB | Compile pane (read-only SQL, any dialect, copy, nothing-executed note) + line-mapped payload errors; extends to any adapter target in M4 | M3, M4 |

Harlequin keeps working as it does today for SQL buffers. anyQL is a *buffer
mode*, not a second app: the same editor widget, tabs, results viewer, catalog,
export, config, and keymap machinery serve both.

---

## 2. What we rip out — the internal workings being replaced

Everything below is Harlequin's SQL-specific core. none of it runs in an anyQL
buffer; each removal is scoped to the buffer mode so SQL buffers keep the
original path untouched.

| Harlequin machinery | File | Fate in anyQL mode |
| --- | --- | --- |
| tree-sitter-sql highlighting | `TextArea.register_language` / `SyntaxAwareDocument` (textual 8.2.8) | replaced by a `tree-sitter-anyql` grammar (M2) |
| sqlfmt autoformat (`action_format`) | `components/code_editor.py:246` | replaced by `format.ts` port (canonical clause spelling) |
| statement splitting (`find_separators`, `selected_queries`) | `components/code_editor.py`, `statements.py` | bypassed: one anyQL document is one query (+ `\with` CTEs); caret scope comes from the AST |
| SQL word/member completers + keyword list | `autocomplete/completers.py`, `constants.py` | replaced by `AnqlCompleter` (slot model, capability-driven); catalog completions re-enter via the datasource schema registry |
| symbol scanning for completion ranking (`find_symbols`) | `components/code_editor.py` | replaced: AST is recomputed per edit; the AST *is* the symbol set |
| `QuerySubmitted(queries=[…])` string pipeline | `app.py:513` et seq. | new `AnqlSubmitted(payload, datasource, dialect)` message beside it |
| adapter-as-connection (in anyQL buffers) | `app.py` connect flow | adapter stays *available* (M4: adapters as run targets), but the default engine path is the vendored ibis registry |

What we **keep and reuse** — this is the whole reason to fork rather than write
a TUI:

- `query.py::typed_rows_to_result` (line 353) — rows → `ResultSet` → Results
  Viewer and hsql text layouts, unchanged. anyQL execution produces
  `columns/rows`, which is exactly its input.
- Results Viewer, export (`export.write_file`), history, crash recovery, buffer
  autosave cache, themes, keymaps, config profiles with `${ENV}` interpolation,
  `harlequin --keys`, the pilot-test harness.
- `EditorCollection`'s single-`CodeEditor` invariant (one widget, N buffer
  states). anyQL mode is per-buffer state, not a second editor.
- The threading contract: workers do database work and `post_message`; `@on`
  handlers own widget state. The engine runs inside `_execute_anql` /
  `_fetch_anql` workers with the same `group=` semantics.

---

## 3. Where the code actually is (measured)

### 3.1 anyQL side (TypeScript + Python sidecar)

```
document text ──parseQuery(doc)──▶ QueryAST ──payloadFromAst──▶ ExecutePayload
   (parser.ts, 535 ln, pure)          (spec/*.ast.json)         (client.ts → FastAPI)
                                                                        │
   server/expression.py   build(con, payload) -> ibis.Table   (306+227 ln)
   server/expression.py   compile_sql(expr, dialect) -> str   — 19 dialects compile offline
   server/execute.py      execute/execute_remote -> {columns, rows, sql, dialect, ms}
   server/datasources.py  registry: demo + 4 vendor mocks + D1 live/snapshot; capabilities; DIALECTS
   server/d1api.py        CloudflareD1.raw(sql) over the HTTPS API (D1 speaks SQLite)
```

The load-bearing contracts we port verbatim:

- **`QueryAST`** — `docs/AST.md` is the written contract; the pair
  `spec/canonical-query.anyql` ↔ `spec/canonical-query.ast.json` is the
  acceptance fixture. Commands: `\with \from \open \join \select \where \group
  \order \case \limit` (+ ephemeral navigation: `\window`, `\top`, `\bottom`,
  clause jumps, function-wrap — these never enter the AST).
- **`ExecutePayload`** — `server/app.py:91`:
  `{dataset, alias, joins[JoinSpec], select[SelectItem], where, groupBy,
  orderBy, limit, cases[CaseSpec], ctes[CteSpec]}` with the nested
  `Aggregate/Temporal/Rank/WindowFrame` select-item shape.
- **Routes** (`app.py:208–432`): `/api/execute`, `/api/compile`,
  `/api/datasets[/{name}/schema|values]`, `/api/capabilities`,
  `/api/datasources`, `/api/dialects`, `/api/sources` (+`/test`), `/api/health`.
  The TUI does **not** talk HTTP to itself — these are the function boundary,
  called in-process through the vendored engine.
- **D1**: live shape (account/token/database → `CloudflareD1.raw`) and snapshot
  shape (local SQLite export, in-process). Execution SQL is always SQLite
  (D1's engine); the *shown* SQL honors the chosen dialect — the compile pane
  stays truthful about another database while execution stays honest about D1.

### 3.2 Harlequin/Textual side (this fork)

| Seam | Location | Shape |
| --- | --- | --- |
| Editor language | `TextArea.register_language(name, ts_language, highlight_query)`; `SyntaxAwareDocument` is tree-sitter-only; unknown language ⇒ plain text + warning | anyQL highlighting needs a real grammar (M2) — there is no regex-lexer hook |
| Completion trigger | `TextAreaPlus` posts `ShowCompletionList(prefix)` / forwards keys to `CompletionList`; `TextEditor` holds `word_completer` / `member_completer` callables `prefix -> list[(render, insert_value)]`; accept → `replace_current_word` / `insert_text_at_selection` (text_editor 0.18.4) | trigger fires on word chars; `\` is `\W` — needs one upstream-able patch to fire the word completer when the line segment starts with `\` (M2 risk R2) |
| Submit path | `CodeEditor.action_submit` → `Submitted` → `Harlequin._get_selected_queries` → `QuerySubmitted(queries, limit)` → `_execute_query` → `query.execute` → `QueriesExecuted` → `_fetch_data` → `query.fetch` → `ResultsFetched` → `load_tables` | anyQL joins at the message layer: `AnqlSubmitted` → `_execute_anql` worker → results via `typed_rows_to_result` |
| Buffer state | one `CodeEditor`; `EditorState` per tab; `capture_state`/`load_state` carry text, selection, scroll, undo history | anyQL mode is one more `EditorState` field (`mode`, `datasource`, `dialect`) |
| Actions/keymaps | `HARLEQUIN_ACTIONS` registry + `bind()` + `harlequin.keymap` entry points | new actions: `anql_inline_ai`, `anql_toggle_compile_pane`, `anql_set_datasource`, `anql_set_dialect` |
| Headless | `hsql` shares `query.execute/fetch`; import-linter forbids Textual in the adapter-facing graph | the anyQL **parser+engine must obey the same import-hygiene rule** so `hsql --anql` stays possible (M4) |
| Tests | pilot drives the real app; syrupy/`pytest-textual-snapshot` SVG baselines; golden format snapshots | port the parser's vitest cases as pytest + the canonical AST fixture as a golden test (M1) |

---

## 4. Architecture: three layers, one seam each

```
┌ UI (this fork) ─────────────────────────────────────────────────┐
│ CodeEditor(anql mode)  \-palette·AnqlCompleter  CompilePane     │
│ AnqlSubmitted ──▶ _execute_anql / _fetch_anql workers ──▶ ResultsViewer │
├ Language (harlequin/anql/) ─────────────────────────────────────┤
│ parser.py (port of parser.ts)  ast.py  payload.py (payloadFromAst) │
│ format.py (formatDocument)     errors[] → editor line markers    │
├ Engine (anql_engine/, vendored from anyql/server) ──────────────┤
│ expression.build / compile_sql   datasources registry            │
│ capabilities                     targets: ibis backends · D1 · adapters (M4) │
└─────────────────────────────────────────────────────────────────┘
```

Decisions, with the reasoning that picked them:

**D1 — Port the parser to Python; do not wrap Node.** `parseQuery` is a pure
function over text with a written AST contract and a deep-equality golden
fixture; the vitest suite enumerates the error messages. Shipping Node in a
terminal product is a cold-start and packaging tax paid forever; the port is a
one-time ~2-day cost with a fixture that makes drift visible. *Mitigation:*
keep `spec/` in-tree and add a CI job that runs the TS parser (`npx tsx`) and
the Python parser over every `.anyql` fixture and diffs the ASTs — the port
stays pinned to the original, not to our memory of it.

**D2 — Vendor `server/{expression,datasources,d1api}.py` as `anql_engine`, not
a sidecar.** The FastAPI layer is transport for the browser; a TUI calling
`build()`/`compile_sql()` in-process removes a process, a port, and CORS. The
engine stays textually untouched so it can diverge back; the port boundary is
`ExecutePayload` dict in, `{columns, rows, sql, ms}` out — the same JSON the
sidecar sends, so the web app and the TUI can be run against identical engines
during validation. Import hygiene: `anql_engine` and `harlequin/anql/` stay
free of Textual (same contract hsql honors), which is what keeps `hsql --anql`
(M4) reachable.

**D3 — Datasource/dialect are anyQL-mode profile keys, not CLI-only.**
`datasource = "demo"`, `dialect = "snowflake"` etc. in a profile, `${ENV}` for
the D1 token (config redaction already covers `secret=True` values). This
reuses the config machinery instead of inventing a parallel one, and it gives
`hsql --anql -P prod -c …` the same connection story as SQL.

**D4 — The compile pane is a second view in the ResultsViewer column, not a
new sidebar.** F6 toggles between Results / Compiled SQL (read-only TextArea,
`dialect` in the border title, copy action, "nothing executed" note). The data
is the `sql` field every execute result already carries, so *every run is also
a decompile*: switch dialect → re-compile (cheap, no execution) → diff what
BigQuery would get vs. SQL Server. This is requirement 5 with no new state.

**D5 — AI lives in the fork, as a first-class action.** Upstream has declined
LLM integration twice (issue #952; `harlequin-for-agents.md` principle 9). That
is a free pass, not a constraint: the fork owns `harlequin/anql/ai.py`. The
contract: prompt = document AST + open-table schemas + capabilities + dialect +
the user's instruction over the selection/clause under the caret; result =
replacement anyQL text; applied with **one** undo checkpoint behind a
diff-preview modal (accept / keep). No auto-execute; provider config
(openai/anthropic/ollama/base-url) rides the profile + env interpolation. The
same context builder serves `hsql --explain-anql` later.

**D6 — Upstream sync discipline.** Upstream ships ~monthly (2.13→2.15 in three
weeks). Rules: (1) all fork code lives under `harlequin/anql/`, `anql_engine/`,
`harlequin/components/compile_pane.py` — new files, not edited ones; (2) edits
to shared files are the ~6 seams in §2 only, each one-line-mode-gated
(`if self.mode == "anyql"`); (3) `git merge upstream/main` weekly into `main`;
(4) snapshot baselines regenerate only from upstream merges, never from anql
feature branches, to keep diffs attributable.

---

## 5. Milestones

Each milestone ends green: `make check`, import-linter, and a runnable demo of
the acceptance line. No milestone ships without its named test.

### M0 — Fork scaffolding (½ day)

Rebrand internals (`anyql` console script alongside `harlequin`; package name
untouched to keep upstream merges cheap); create `harlequin/anql/` and
`anql_engine/`; vendor `expression.py`, `datasources.py`, `d1api.py`,
`execute.py`, `spec/`, the demo+mock Parquet tree; upstream remote discipline
(D6); **add LICENSE to the vendored anyQL code** (the anyql repo has none —
decide MIT to match Harlequin before merging any of it).
*Accept:* `uv run pytest -m "not online"` green with the engine vendored and
`anql_engine` covered by a new import-linter contract (`harlequin.anql` and
`anql_engine` must not reach Textual).

### M1 — Parser port + golden contract (2 days)

`anql/parser.py` (port of `parser.ts`, including the `\with` body recursion,
star rules, alias derivation `<arg>_<fn>`, window-frame errors,
quoted-value scanning — `docs/AST.md` §Rules is the checklist),
`anql/payload.py` (`payloadFromAst`), `anql/format.py` (`formatDocument`,
idempotent). Tests: canonical fixture deep-equality; every vitest error case as
a pytest param; the TS↔Py differential CI job (D1).
*Accept:* `parseQuery("…canonical…") == ast.json` and the differential job
green. No UI touched.

### M2 — Editor mode + backslash commands (1 wk)

Register the `anyql` language: hand-written `tree-sitter-anyql` grammar
(line-oriented: `\command` token, column/identifier, operators, quoted values,
comment line) built as a `tree-sitter-anyql` wheel dependency — the same
packaging pattern `tree-sitter-sql` uses; R1 below if grammar-build on Windows
CI stalls, fallback to no-highlight M2, grammar M3.
Buffer mode: new `EditorState.mode`; in anyql mode disable sqlfmt/semicolon
splitting; autoformat on Enter = `formatDocument` port (idempotent; half-typed
lines untouched, exactly the web behavior).
`\`-palette: one `AnqlCompleter` implementing the `completion.ts` slot model —
palette (clause compose/jump, `top`/`\bottom`), skeleton insert with caret in
first slot (`\case flag = when 〈`, `\with name\n  \from 〈`), guided slot
chains (`when`/`then`/`else`, `over ( partition by … order by … )`),
dtype-aware function-wrap (`\sum` on a column → `sum(amount) ` with
`amount_sum` preview; temporal-family filter from capabilities), `\where` value
search (distinct values from the engine, quoted/bare insertion), `\group auto`,
`\window` gesture/column-wrap surfaces. Trigger patch per R2.
*Accept:* pilot test drives the editor typing `\from events\select amount\sum`,
asserts buffer equals `\from events\n\select sum(amount) `; every AST.md
"Navigation" bullet has a pilot test; canonical doc round-trips through
`formatDocument` unchanged.

### M3 — Engine wiring + compile pane (1 wk)

Datasource registry in-process (demo Parquet, vendor mocks, D1 live/snapshot
via `anql_engine`); profile keys D3; messages `AnqlSubmitted` →
`_execute_anql` (worker) → engine `execute`/`compile` →
`typed_rows_to_result` → `ResultsFetched` path, keeping execute-then-fetch
split; payload/parse errors surface as inline editor markers (AST
`errors[].line`) + a notification — never a crash (adapter-error precedent
`app.py` `handle_query_error`). Compile pane (D4): F6 toggle, read-only SQL,
dialect switch re-renders without executing; copy to clipboard; `\compile`
command. Catalog tree shows the active datasource's datasets (reuse
`DataCatalog`'s database tree with a synthetic `Catalog`).
*Accept:* pilot: type canonical query, Ctrl+Enter, assert results table rows
match `fixtures/expected_mock_rows.json`; F6 asserts the compiled SQL pane;
switch dialect `duckdb`→`snowflake`, assert pane shows `TOP`/bracket quoting
without a second execution; D1 snapshot source runs end-to-end; `\where`
bad-column shows a line marker, no traceback.

### M4 — Any database: adapters as targets + headless (1 wk)

Target kinds, all feeding the same pane/execute split:
1. **ibis backends** (postgres, mysql, bigquery, duckdb-file, …) —
   `ibis.<backend>.connect(**profile)`; schema from ibis; execute natively.
   Connection options = new profile tables, reusing `AbstractOption` widgets
   for the settings UI.
2. **Harlequin adapter passthrough** — any installed adapter (`duckdb`,
   `sqlite`, `postgres`, D1-via-`harlequin-d1`, ODBC, …): build the payload
   against the adapter's catalog schema (an `ibis.memtable`-free schema-only
   source), compile to the adapter's dialect (a small `adapter→dialect` map),
   execute via `HarlequinConnection.execute`, rows straight into
   `typed_rows_to_result`. This makes "point it at any DB" true via the 15+
   adapter ecosystem *and* keeps `--read-only`/`--limit`/cancel semantics.
3. **D1** stays native (M3) — it is not a Harlequin adapter and speaks its own
   HTTP API.
`hsql --anql -f q.anyql` (+ `-c`, stdin): parses with `anql.parser`, runs the
engine, lays out with `harlequin.layout` — same exit codes and formats as SQL
mode.
*Accept:* one anyQL document compiled for `duckdb` and run unchanged against
the bundled DuckDB adapter *and* a Postgres profile (test: postgres adapter +
mock); `hsql --anql --csv` byte-diff against the TUI's export of the same run;
import-linter still green.

### M5 — Inline AI (3–4 days)

`harlequin/anql/ai.py`: provider clients (openai, anthropic, ollama,
OpenAI-compatible base-url) with keys via env-interpolated profile
(`api_key = "${ANYQL_AI_KEY}"`); context builder (AST + open-table schemas +
capabilities + current dialect + instruction); actions `anql_inline_ai`
(selection or clause under caret) and `anql_ai_rewrite_doc`; response parsed as
replacement anyQL text, re-parsed before presenting (invalid AI output is a
toast, never a buffer write); diff-preview modal — unified diff over the
affected lines, accept applies via one undo checkpoint (the
`action_format`/external-editor precedent), keep dismisses; regenerate cycles
new checkpoints. Bound what leaves the machine: schema + document only, never
result rows, stated in the docs and in the help screen.
*Accept:* pilot with a fake provider: instruction over a selection rewrites the
clause, accept is one Ctrl+Z; invalid AI output leaves buffer untouched; no
network in tests.

---

## 6. Risks

| # | Risk | Mitigation |
| --- | --- | --- |
| R1 | `tree-sitter-anyql` grammar build on Windows CI (node-generate, `tree-sitter` wheel matrix) | grammar is line-oriented (~120 lines of grammar.js); pin the same build pattern as `tree-sitter-sql`; fallback M2 plain-text, grammar lands M3 without touching other seams |
| R2 | Completion popup never fires on `\` (textual-textarea triggers on `\w` runs) | one small patch to `TextAreaPlus` (textual-textarea is tconbeer's — open the PR; fork-patch meanwhile), gated on the completer wanting a custom trigger pattern; matches AGENTS.md "fix it upstream" |
| R3 | Parser drift vs `parser.ts` after fixes | differential CI job over `spec/` fixtures; error-message texts are the contract (AST.md) |
| R4 | Upstream churn (monthly releases) makes the fork's diffs ugly | D6: new-file containment, mode-gated one-liners, weekly merges; snapshot regen discipline |
| R5 | anyql engine's `PayloadError` messages assume the web UI's toast idiom | map through the same error-modal path adapters already use; keep messages verbatim (they're tested) |
| R6 | D1 token hygiene | reuse the sidecar rule: token never persisted, never echoed, lives in the in-memory source; profile interpolation + `redact` cover config files |
| R7 | License gap on vendored anyQL code | add MIT (M0) before the first vendored merge; not optional |

---

## 7. Non-goals

- No upstream PRs for the AI feature or buffer modes — upstream's position is
  settled (#952); we sync, we don't pitch. (Exception: the R2 trigger patch,
  which is a generic editor improvement.)
- No browser/web surface; the web app stays the anyql repo's business.
- No charts, no schema-tree parity beyond what `DataCatalog` gives us for free.
- No second editor widget; the single-`CodeEditor` invariant is load-bearing
  for ~190 upstream assertions.

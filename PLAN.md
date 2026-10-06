# PLAN.md — Clinical Trial Navigator

## What we're building
Protocol PDF → retrieve the schedule of activities (SoA) → extract it into structured data → score accuracy against hand-labeled answers → model it in a warehouse → expose it through Q&A, an agent, and an MCP server → deploy.

**One-liner:** Eval-gated LLM pipeline that turns unstructured clinical trial protocols into a dimensional model.

**How to use this file:** work top to bottom. Each stage only has to *work* before moving on. `Build` = hands-on, `Study` = read/learn just enough for that stage.

## Target data model
- **protocol**: nct_id, title, phase, pdf_file
- **visit**: nct_id, visit_label ("Screening", "Week 4"), visit_order, timing_text
- **procedure**: nct_id, procedure_raw (as written), procedure_canonical
- **scheduled_activity** *(fact)*: nct_id, visit, procedure, status (required / conditional), footnote_ids
  - grain = one procedure at one visit in one protocol
- **footnote**: nct_id, footnote_id, text

---

## Stage 1 — Ingest
Goal: reliable, re-runnable download of protocols + metadata.
- [x] Tools check: Python 3.11+, git, VS Code, GitHub account
- [x] Create public repo, clone, open in VS Code
- [x] venv + `requests` + `requirements.txt`; `.gitignore` covers `.venv/`, `data/raw/`, `.env`
- [x] Add `CLAUDE.md` + `PLAN.md`, commit, push
- [ ] Explore the API by hand in a browser: one study's JSON, find `documentSection`, find a protocol PDF link
- [ ] `src/ingest.py`: fetch 1 study by NCT ID, print title, download its protocol PDF
- [ ] Scale to 10 hand-picked NCT IDs; skip files already downloaded
- [ ] Write `data/manifest.csv`: nct_id, title, phase, pdf_file, page_count, has_soa
- [ ] Open each PDF; confirm it has an SoA; swap out any that don't
- [ ] Note how SoA tables vary (location, page span, footnote style) in `notes.md`
- [ ] README: what it is, how to run, data source
- **Level up**
  - [ ] Build: replace hand-picked IDs with an API search (filters + pagination)
  - [ ] Build: retries with backoff, logging instead of print, CLI args (`--limit 100`)
  - [ ] Build: scale to ~100 protocols (10 become the gold set, the rest are unlabeled test data)
  - [ ] Build: first unit tests with pytest (manifest writing, skip logic)
  - Study: HTTP status codes + rate limiting; Python `logging`; pytest basics

## Stage 2 — Gold set
Goal: the answer key everything else gets graded against.
- [ ] Study: read 2 SoAs closely; learn visit/cycle/day notation and footnote conventions
- [ ] Build: `docs/labeling_guide.md` — rules for edge cases (what counts as conditional, merged cells, "as needed")
- [ ] Build: label the main SoA of 10 protocols (spreadsheet → CSV in `data/gold/`)
- [ ] Build: re-label 2 protocols a few days later without looking; compare → tighten the guide
- Study: why label quality caps eval quality

## Stage 3 — Locate (retrieval) + parse
Goal: find the right pages, then get tables out. RAG becomes load-bearing here.
- [ ] Build: split PDFs into page-level chunks with page numbers
- [ ] Build: keyword baseline (BM25) for "which pages hold the SoA"
- [ ] Build: Bedrock access (personal AWS, us-east-1, budget alert)
- [ ] Build: embeddings via a Bedrock embedding model; store in Postgres + pgvector (Docker)
- [ ] Build: hybrid search (keyword + vector); compare all three
- [ ] Build: retrieval eval — recall@k for SoA pages vs gold page numbers
- [ ] Build: pdfplumber table extraction on retrieved pages; log where it breaks
- Study: embeddings, chunking strategies, BM25 vs vector vs hybrid, recall@k; Docker basics

## Stage 4 — LLM extraction
Goal: structured SoA out of messy pages.
- [ ] Build: Pydantic models matching the data model
- [ ] Build: Claude on Bedrock (Converse API) → output that validates against the models; retry on validation failure
- [ ] Build: experiment — text input vs page-image input; which extracts better?
- [ ] Build: prompts stored as versioned files, not inline strings
- [ ] Build: pdfplumber-only vs LLM vs hybrid comparison
- Study: structured outputs / tool use for extraction; prompt engineering docs (Anthropic)

## Stage 5 — Evals + model selection
Goal: know exactly how good it is, and catch regressions.
- [ ] Build: `src/evaluate.py` — cell-level precision/recall, footnote linkage accuracy, per-protocol scores
- [ ] Build: error analysis — tag every miss (wrong visit, missed footnote, merged cell…) and count by type
- [ ] Build: model comparison — Haiku vs Sonnet: accuracy vs cost per protocol vs latency
- [ ] Build: GitHub Actions runs evals on every PR; fails if score drops
- Study: eval design, error analysis practice, CI basics

## Stage 6 — Observability
Goal: see what the system does and what it costs.
- [ ] Build: tracing for every LLM call (Langfuse or similar): prompt version, tokens, latency, cost
- [ ] Build: structured (JSON) logging across the pipeline
- [ ] Build: cost + accuracy summary per run
- Study: LLMOps basics — tracing, prompt versioning, cost tracking

## Stage 7 — Warehouse
Goal: a real dimensional model on top of the extractions.
- [ ] Build: dbt project on Postgres — staging → marts; star schema with `fct_scheduled_activity`
- [ ] Build: dbt tests (unique, not_null, relationships) + docs
- [ ] Build: procedure normalization — map raw names to a canonical list (embeddings or LLM-assisted, checked against a small labeled set)
- [ ] Build: re-ingest study records over time; track changes with dbt snapshots (SCD2)
- [ ] Build: Airflow DAG: ingest → retrieve → extract → evaluate → dbt build
- Study: Kimball ch. 1–4 (grain, star schema, SCD2); dbt fundamentals; Airflow basics

## Stage 8 — Q&A, agent, MCP
Goal: make the data usable by people and other AI systems.
- [ ] Build: Q&A over a protocol, retrieval built by hand, answers cite page numbers
- [ ] Build: Q&A eval — small question set with known answers
- [ ] Build: PydanticAI agent with tools: `get_schedule`, `search_protocol`, `resolve_footnote`
- [ ] Build: MCP server exposing those tools; call it from Claude Desktop
- Study: Anthropic "Building effective agents"; MCP docs; PydanticAI docs

## Stage 9 — Deploy
Goal: something a stranger can open and use.
- [ ] Build: Dockerize the app
- [ ] Build: simple UI — pick a protocol → SoA grid, eval scores, Q&A
- [ ] Build: deploy (hosting decided at this stage); secrets handled properly
- [ ] Build: README with architecture diagram, eval results, cost numbers, lessons learned
- Stretch: infrastructure as code (Terraform)

---

## Parallel, daily
- 2 DataLemur problems before opening this repo

## Decisions log
- Oct 5: Scope = SoA extraction from public protocol PDFs (not Q&A over registry records)
- Oct 5: Start building before Bedrock access; Bedrock set up in Stage 3
- Oct 5: No fixed deadlines for now; focus on doing. Expanded goals: retrieval in core, model/cost comparison, observability, MCP, SCD2

## Out of scope (for now)
- Fine-tuning, LangChain, private/proprietary data

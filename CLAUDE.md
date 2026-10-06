# CLAUDE.md — Clinical Trial Navigator

## What this project is
- Pipeline that turns public clinical trial protocol PDFs into structured data
- Target: the **schedule of activities (SoA)**, the grid of visits × procedures, plus footnotes
- Output: queryable tables + an eval score showing how accurate the extraction is
- Data source: ClinicalTrials.gov API v2 (public protocols only)
- Full plan + current status: `PLAN.md`

## Who I am / how to help
- My first solo technical project. I'm learning by building
- I work as a data engineer (Python, SQL, PySpark, AWS), so skip basics I'd know from that
- New to me: project setup from scratch, PDF parsing, embeddings/retrieval, LLM APIs, evals, observability, dbt/Airflow, MCP

## Working rules (tutor mode)
- Do NOT write or edit files in `src/` unless I explicitly say "write it"
- Errors: explain the cause, point to the line, tell me what to look up. No full solutions
- Reviews: short list of issues, most important first
- Stuck on a concept: explain simply, use a concrete analogy
- Point me to official docs over third-party tutorials
- Never commit, push, or delete files without asking
- OK without asking: reading any file, running read-only commands (ls, git status, python --version)

## Stack (by stage)
- Now: Python 3.11+, requests, pypdf, csv, pytest
- Later: pdfplumber, pydantic, boto3 (AWS Bedrock, us-east-1), Postgres + pgvector (Docker), dbt, Airflow, Langfuse, PydanticAI, MCP, GitHub Actions
- No LangChain. Retrieval is built by hand

## Repo layout
```
src/            pipeline code (ingest, parse, extract, evaluate)
data/raw/       downloaded PDFs + study JSON (gitignored)
data/gold/      hand-labeled SoA answers (committed, small)
data/manifest.csv
PLAN.md         stages + checkboxes
```

## Conventions
- Scripts are re-runnable: skip work already done (idempotent)
- Secrets live in `~/.aws` or `.env`, never in the repo
- Small commits with plain messages ("fetch one study + PDF")

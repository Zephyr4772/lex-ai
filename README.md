# Lex.ai

**Retrieval-augmented search and analysis over 41,876 Indian Supreme Court judgments.**

Ask a legal question in plain English. Lex.ai embeds the query, retrieves the most
semantically similar judgment passages from a local vector index, and streams a
grounded analysis back — with the source cases cited as structured cards, so every
claim can be traced to a judgment.

> **Status:** working prototype. Runs locally; not deployed. See [Limitations](#limitations).

---

## Why

Legal research tools match keywords. Judgments don't repeat the keywords a
researcher would think to type — "right to privacy" appears in cases that never
use the phrase. Semantic retrieval over the full corpus finds those; keyword
search doesn't.

## How it works

```
Browser (localhost:5173)
   │  POST /api/query  {query, provider, model_name, filters, api_key}
   ▼
FastAPI  (localhost:8000)          backend/api.py
   │  1. embed query    → Ollama  nomic-embed-text   (localhost:11434)
   │  2. retrieve top-k → ChromaDB  "supreme_court_judgments"
   │  3. build prompt from retrieved chunks + metadata
   ▼
LLM  — Gemini 2.5-flash, or a local Ollama model
   │  streams NDJSON
   ▼
Browser
   ├── case cards render immediately  (case no., bench, date, outcome)
   └── analysis streams token by token
```

Retrieval is **local** — the corpus and embeddings never leave the machine. Only
the retrieved passages are sent to the LLM, and only if you choose a hosted
provider. Selecting Ollama keeps the entire pipeline offline.

## Features

- **Dual provider** — Gemini 2.5-flash or any local Ollama chat model, chosen per request.
- **Automatic fallback** — if the Gemini call fails mid-stream, the request retries against local Ollama rather than erroring out.
- **Metadata filtering** — narrow retrieval by court, bench, outcome, and year range before the semantic search runs.
- **Retrieval-only mode** — pass `api_key: "ollama-local"` to skip generation and inspect raw retrieval.
- **Citation export** — `POST /api/download_zip` bundles the cited judgments as a zip.
- **Debug endpoint** — `GET /api/debug/retrieve?q=...&n=3` returns raw retrieval hits, for checking whether a bad answer is a retrieval failure or a generation failure.

## Repository layout

```
backend/
  api.py              FastAPI app — retrieval, prompt assembly, streaming
  pipeline/           ETL: extract → consolidate → embed → index
  utils/              probe_the_brain.py, test_chain.py
frontend/             React + Vite app (the search UI)
landing/              Marketing/landing page, separate Vite app
database/
  supabase_schema.sql
data/                 Judgment corpus — see "The data" below
```

## Running it

Three services, three terminals.

**Prerequisites:** Python 3.12+, Node 18+, [Ollama](https://ollama.com), and a
[Gemini API key](https://aistudio.google.com/apikey) if you want hosted generation.

```bash
# 0. embedding model
ollama serve
ollama pull nomic-embed-text
```

```bash
# 1. backend
cd backend
pip install fastapi uvicorn chromadb google-generativeai requests pydantic
export CHROMA_DIR=./chroma_db          # Windows: set CHROMA_DIR=.\chroma_db
python -m uvicorn api:app --port 8000
```

```bash
# 2. frontend
cd frontend
npm install
npm run dev                            # http://localhost:5173
```

Enter your Gemini key via the key icon in the UI. It is stored in `localStorage`
and sent only to Gemini's API.

Check the backend came up:

```bash
curl http://localhost:8000/api/health
```

## Building the index

The vector index is not in the repo — it's built from the corpus:

```bash
cd backend/pipeline
python extract_judis_hybrid.py        # 1. scrape/parse JUDIS judgments
python extract_sci_hybrid.py          # 2. scrape/parse SCI judgments
python consolidate_master_index.py    # 3. merge into master CSV
python build_obsidian_vault.py        # 4. (optional) markdown vault
python ingest_to_chroma.py            # 5. embed + index into ChromaDB
```

Step 5 embeds every chunk through Ollama and takes hours on CPU.

## The data

`data/` holds the judgment corpus — roughly 1.3 GB across ~42,000 extracted JSON
files plus three enriched CSVs (`supreme_court_master_index.csv`,
`master_judis_enriched.csv`, `master_sci_enriched.csv`).

Judgments of the Supreme Court of India are government works and are not
copyrighted, but the extraction and enrichment here are best-effort scrapes:
**treat this corpus as a research artifact, not an authoritative source.** Always
verify against the official record before relying on a case.

## Limitations

- **Not deployed.** Local-only; no hosted instance.
- **Retrieval quality is unmeasured.** There is no evaluation set, so recall and
  citation faithfulness are unknown. The `/api/debug/retrieve` endpoint exists
  precisely because this needs checking by hand.
- **The LLM can still misread a correctly-retrieved passage.** Citations tell you
  what the model was given, not that it reasoned about it correctly.
- **Not legal advice.** This is a research tool.

## Known issues

- `CHROMA_DIR` is hardcoded to an absolute Windows path in `backend/api.py`; it
  should read from the environment. Until that's fixed, clones need the line edited.
- The repo carries large media at its root (`stockvideo.mp4`, design PNGs) and
  duplicated `public/` assets between `frontend/` and `landing/`, which makes
  cloning slow.
- `refrences/` is misspelled and holds design screenshots.

## License

No license yet — all rights reserved. Open an issue if you want to reuse this.

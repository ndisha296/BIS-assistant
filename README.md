# BIS-assistant
# BIS Assistant

A backend for answering questions about **Indian Standards (BIS)** and for reading BIS-related codes off photos (hallmark HUIDs, CM/L licence numbers, IS standard numbers on labels).

It has two pipelines:

1. **Text Q&A (RAG):** a user asks a question, the system retrieves matching standards from a local vector store and asks an LLM to answer *only* from those retrieved rows, with citations.
2. **Image scanning (OCR):** a user uploads a photo, OCR extracts a HUID / CM/L number / IS number, and the result is verified against a database (hallmarks) or passed into the RAG pipeline (standards).

The guiding rule throughout: **answers must come from the loaded data, not from the model's memory.** If the data doesn't contain the answer, the assistant says so.

---

## Table of contents

1. [Architecture](#architecture)
2. [Project structure](#project-structure)
3. [Data sources](#data-sources)
4. [Setup](#setup)
5. [Configuration (.env)](#configuration-env)
6. [Loading the data (ingestion)](#loading-the-data-ingestion)
7. [API reference](#api-reference)
8. [How the RAG pipeline works](#how-the-rag-pipeline-works)
9. [How the OCR pipeline works](#how-the-ocr-pipeline-works)
10. [Database models](#database-models)
11. [Scope and honest limitations](#scope-and-honest-limitations)
12. [Known issues and TODOs](#known-issues-and-todos)
13. [Troubleshooting](#troubleshooting)

---

## Architecture

```
                    ┌─────────────────────────────────────────┐
                    │              FastAPI (main.py)          │
                    └───────┬───────────────┬─────────────────┘
                            │               │
              /api/chat     │               │   /api/verify-hallmark
                            │               │   /api/scan-label
                            ▼               ▼
                   ┌────────────────┐  ┌──────────────────┐
                   │    rag.py      │  │  ocr_router.py   │
                   │ embed → Chroma │  │ picks best OCR   │
                   │ → LLM answer   │  │ engine, parses   │
                   └───────┬────────┘  │ codes (regex)    │
                           │           └────────┬─────────┘
                           ▼                    │
                   ┌────────────────┐           ▼
                   │  Chroma store  │   ┌────────────────────┐
                   │ (data/chroma_  │   │ database.py (SQL)  │
                   │  store)        │   │ HUID lookup, users │
                   └────────────────┘   └────────────────────┘
```

| Concern | Technology |
|---|---|
| API | FastAPI |
| Relational data (users, certifications, hallmark records) | SQLAlchemy (SQLite or Postgres via `DATABASE_URL`) |
| Semantic search | ChromaDB (local, persistent) |
| Embeddings | `sentence-transformers` (default `all-MiniLM-L6-v2`) |
| LLM | Groq, Gemini, or OpenRouter (switch with `LLM_PROVIDER`) |
| OCR | EasyOCR, PaddleOCR-VL-1.6, or YOLO-crop + EasyOCR (auto-selected) |
| Packaging | Docker + docker-compose |

**Why two data stores?** SQL is for exact, structured, relational data (a user's certifications, a HUID record). The vector store is for fuzzy, meaning-based search ("what standards cover pressure cookers?") where there is no exact keyword to match. They complement each other; neither replaces the other.

---

## Project structure

```
bis-assistant/
├── app/
│   ├── __init__.py
│   ├── config.py               # Reads settings from environment / .env
│   ├── main.py                 # FastAPI app and endpoints
│   ├── database.py             # SQLAlchemy models + HUID lookup
│   ├── rag.py                  # Embedding, Chroma retrieval, LLM call
│   ├── ingest_standards.py     # One-off loader: JSON data → Chroma
│   ├── ocr_patterns.py         # Shared regex for HUID / CM-L / IS numbers
│   ├── ocr_router.py           # Chooses the best available OCR engine
│   ├── ocr_service.py          # YOLO hallmark crop + EasyOCR + DB verify
│   ├── ocr_paddleocr_vl.py     # PaddleOCR-VL-1.6 engine (GPU recommended)
│   └── ocr_easyocr_quick.py    # Plain EasyOCR engine (CPU fallback)
├── data/
│   ├── standards.json          # 16,251 IS standards (source for ingestion)
│   ├── cert.json               # 711 mandatory-certification entries
│   ├── labs.json               # 90 BIS-recognised testing lab entries
│   └── chroma_store/           # Vector DB files (created by ingestion)
├── docker-compose.yml
├── Dockerfile
├── .env / .env.example
└── README.md
```

> The PaddleOCR file is imported as `ocr_paddleocr_vl` by `ocr_router.py`. If your file is named differently, rename it or update the import in `ocr_router.py`.

---

## Data sources

All data was built from BIS publications supplied as spreadsheets:

| Source | Content | Records |
|---|---|---|
| Department lists (CED, FAD, LITD, MHD, PCD, PGD, SSD, TXD) | Standard No., Year, Title, per technical department | ~16,070 |
| `BIS_IS_1_to_10000_MASTER.xlsx` | Base IS numbers 1–10000; only ~271 rows have verified titles | 181 added after de-duplication |
| Products Under Compulsory Certification | IS number → product → notification order | 711 |
| Testing Facilities | Lab → IS number → product tested | 90 |

Total ingested: **17,052 documents** (16,251 standards + 711 certification + 90 lab).

**Data-cleaning notes**

- The department `.xls` files are actually HTML tables, not real Excel files. They are parsed with `pandas.read_html`.
- Standards are de-duplicated by standard number; department files take priority over the master file.
- One corrupted character (U+FFFD) in a title was replaced with `-`.
- The master file intentionally leaves unverified fields blank rather than guessing. The assistant respects that: a missing row is reported as "not in the dataset," never invented.
- Fields such as technical committee, amendments, and equivalence are **not** present in the source spreadsheets, so the assistant cannot answer questions about them.

---

## Setup

### Option A: Docker (recommended)

```bash
# from the project root
docker compose up --build -d backend
```

The first build downloads large dependencies (PyTorch, sentence-transformers, EasyOCR weights), so it can take several minutes.

Check it is up:

```bash
curl http://localhost:8000/health
# {"status":"ok"}
```

(Use whatever host port your `docker-compose.yml` maps.)

### Option B: Run locally

Requires Python 3.10+ (the code uses `str | None` type syntax).

```bash
python -m venv .venv
# Windows PowerShell:
.venv\Scripts\Activate.ps1
# macOS/Linux:
source .venv/bin/activate

pip install -r requirements.txt
uvicorn app.main:app --reload
```

> On Windows, if `python` is "not recognised", try `py` instead, or run everything inside the Docker container.

---

## Configuration (.env)

Settings are read in `app/config.py`. Based on the code, these variables are used:

| Variable | Purpose | Example |
|---|---|---|
| `DATABASE_URL` | SQLAlchemy connection string | `sqlite:///./bis_assistant.db` or `postgresql://user:pass@db:5432/bis` |
| `EMBEDDING_MODEL_NAME` | sentence-transformers model | `sentence-transformers/all-MiniLM-L6-v2` |
| `CHROMA_PERSIST_DIR` | Where Chroma stores vectors | `./data/chroma_store` |
| `LLM_PROVIDER` | `groq`, `gemini`, or `openrouter` | `groq` |
| `GROQ_API_KEY`, `GROQ_MODEL` | Groq settings | |
| `GEMINI_API_KEY`, `GEMINI_MODEL` | Gemini settings | |
| `OPENROUTER_API_KEY`, `OPENROUTER_MODEL` | OpenRouter settings | |
| `EASYOCR_GPU` | `true`/`false`, used by `ocr_service.py` | `false` |
| `HALLMARK_MODEL_PATH` | Path to trained YOLO weights for hallmark cropping | `./models/hallmark.pt` |

Only the API key for your chosen `LLM_PROVIDER` is required. Never commit real keys; keep them in `.env` (which should be in `.gitignore`) and share only `.env.example`.

---

## Loading the data (ingestion)

**Do this once before using the chat endpoint.** `rag.py` only *reads* from Chroma; nothing fills it automatically. With an empty collection, `/api/chat` will always reply that it has no information, which looks like a bug but is really an empty corpus.

1. Put `standards.json`, `cert.json`, and `labs.json` in `data/`.
2. Run the loader **inside the backend container** (so Python and all dependencies are available):

```bash
docker compose exec backend python -m app.ingest_standards --data-dir ./data
```

Or locally:

```bash
python -m app.ingest_standards --data-dir ./data
```

Expected final line: `Done. Collection now has 17052 documents.`

**What ingestion does**

- Batch-encodes text with the same embedder `rag.py` uses (much faster than one row at a time).
- Standards use the IS number as the vector ID, so **re-running is safe**: existing entries are overwritten, not duplicated.
- Every document gets `metadata["source"]` set to the IS number, so the `[Source: ...]` citations in answers resolve to real standard numbers.
- Three document types are stored, tagged in metadata as `standard`, `certification`, or `lab`.

Re-run the loader whenever the JSON data files are refreshed.

---

## API reference

### `GET /health`
Liveness check. Returns `{"status": "ok"}`.

### `POST /api/chat`
Ask a question about BIS standards.

Request:
```json
{ "query": "Which standard covers Portland pozzolana cement?" }
```

Response:
```json
{
  "answer": "IS 1489 (Part 1) covers ... [Source: IS 1489 (Part 1)]",
  "sources": ["IS 1489 (Part 1)", "IS 1489 (Part 2)"]
}
```

### `POST /api/verify-hallmark`
Upload a photo of an engraved hallmark. Multipart form with field `file`.

Response when a HUID is read and found:
```json
{
  "detected_code": "AB12CD",
  "verification": {
    "verified": true,
    "huid": "AB12CD",
    "purity": "916 (22K)",
    "jeweller_name": "...",
    "hallmarking_centre": "...",
    "hallmark_date": "2024-05-01",
    "article_type": "Ring"
  }
}
```

If no HUID can be read, or the HUID is not in the registry, `verified` is `false` with an explanatory message.

### `POST /api/scan-label`
Upload a photo of a product label or standard plate. Multipart form with field `file`. OCR extracts any IS numbers, HUID, and CM/L number, then the IS numbers (or CM/L) are turned into a question and answered through the RAG pipeline.

Response: the normal `answer` / `sources` payload plus a `scan` object:
```json
{
  "answer": "...",
  "sources": ["IS 269"],
  "scan": {
    "huid": null,
    "cml": null,
    "is_numbers": ["IS 269"],
    "engine_used": "ocr_easyocr_quick (CPU fallback)",
    "raw_text": "..."
  }
}
```

Interactive docs are available at `/docs` (Swagger UI) when the server is running.

---

## How the RAG pipeline works

Implemented in `app/rag.py`.

1. **Embed** the question with `sentence-transformers`.
2. **Retrieve** the top-k (default 4) closest documents from the Chroma collection `bis_standards`.
3. **Build a prompt** containing the retrieved chunks, each labelled with its source.
4. **Call the LLM** (Groq / Gemini / OpenRouter, chosen by `LLM_PROVIDER`) with a strict system prompt: answer only from the context, cite `[Source: ...]` for every claim, and if the context lacks the answer, say so and point to bis.gov.in rather than guessing.
5. **Return** the answer plus the list of sources.

Switching LLM provider is a one-line `.env` change; no code changes needed.

**Design principle:** retrieval decides what the model is allowed to know. This is why data quality and ingestion matter more than model choice.

---

## How the OCR pipeline works

There are three OCR engines, plus a router that chooses between them.

| Engine | File | Strengths | Requirements |
|---|---|---|---|
| YOLO crop + EasyOCR | `ocr_service.py` | Best for HUIDs: detects and crops the hallmark region first | Trained weights at `HALLMARK_MODEL_PATH` |
| PaddleOCR-VL-1.6 | `ocr_paddleocr_vl.py` | Strongest general OCR quality on engraved text | GPU, `torch`, `transformers>=5.0.0` |
| EasyOCR | `ocr_easyocr_quick.py` | Works on CPU with no setup | None extra |

**Selection order** (`ocr_router.py`): YOLO pipeline if weights exist, else PaddleOCR-VL if a CUDA GPU is available, else EasyOCR. The chosen engine is reported in `scan.engine_used` for debugging.

**Code extraction** (`ocr_patterns.py`) is shared by all engines so patterns cannot drift apart:

- **HUID:** 6 alphanumeric characters (`[A-Z0-9]{6}`)
- **CM/L licence:** `CM/L-` followed by 7–10 digits
- **IS number:** `IS 1234`, `IS/ISO 1234`, optionally with `Part N`

A 6-character alphanumeric pattern is loose and can match ordinary words or numbers; treat a detected HUID as a *candidate* until it is confirmed by the database lookup.

---

## Database models

Defined in `app/database.py` (tables are created on startup via `init_db()`).

| Model | Purpose |
|---|---|
| `User` | Account: name, email, hashed password, phone |
| `Certification` | A licence a user holds (scheme, licence number, product category, issue and expiry dates, `reminder_sent` stage 0/1/2 for 90/30/7-day reminders) |
| `ProductStandardMap` | Curated product keyword → IS standard + scheme. Trusted overrides that avoid pure LLM guessing |
| `HallmarkRecord` | **Mock** HUID registry: HUID, purity, jeweller, centre, date, article type |

`lookup_huid(db, huid)` returns verified details, or an explicit "not found, may be counterfeit or misread" message.

---

## Scope and honest limitations

Be upfront about these in any demo or pitch:

- **Gold purity cannot be determined from a photo.** That requires a lab assay. The system only reads the engraved HUID and looks it up.
- **The HUID registry is a mock table.** BIS's real HUID database (BIS Care) is not a public API. Production use would need a data-sharing arrangement with BIS. Seed `hallmark_records` with demo rows to test the flow.
- **The standards data covers titles, years, and departments only.** No clause text, amendments, superseded status, or technical committee, so the assistant cannot answer content-level questions such as "what is the minimum compressive strength in IS 456?"
- **"Not on the mandatory list" is not "not mandatory."** The certification list is a snapshot; the assistant can only say an item does not appear in it.
- **OCR quality depends on photo quality.** Small engraved text on metal is hard for any OCR engine.

---

## Known issues and TODOs

- [ ] **Wire `ocr_router` into `main.py`.** The original `main.py` imported `extract_huid` directly from `ocr_easyocr_quick`, so the YOLO-crop pipeline was never used. Replace with `from app.ocr_router import scan_image`.
- [ ] **Blocking calls in async routes.** `answer_question()` and OCR are synchronous but called inside `async def` handlers, which blocks the event loop. Use plain `def` routes or `run_in_threadpool`.
- [ ] **Deprecated startup hook.** `@app.on_event("startup")` should become a `lifespan` context manager.
- [ ] **Add CORS middleware** if a frontend runs on a different origin.
- [ ] **`datetime.utcnow()` is deprecated** in Python 3.12+; use `datetime.now(timezone.utc)`.
- [ ] **Seed `hallmark_records`** with demo data.
- [ ] **Train the hallmark YOLO detector** and set `HALLMARK_MODEL_PATH`.
- [ ] **Authentication and reminders.** The `User`/`Certification` models and the `reminder_sent` field are in place, but no auth endpoints or reminder job exist yet.
- [ ] **Optional: migrate Chroma → Pinecone** for hosted, multi-instance deployments. Chroma is fine for local development and single-server use.

---

## Troubleshooting

**Chat always says "I don't have that information."**
The Chroma collection is probably empty. Run the ingestion step and confirm it reports 17052 documents.

**`python` is not recognised in PowerShell.**
Use `py`, or run commands inside the container: `docker compose exec backend python -m app.ingest_standards --data-dir ./data`.

**`ModuleNotFoundError` when running the ingestion script.**
Run it as a module from the project root (`python -m app.ingest_standards`), not as `python app/ingest_standards.py`, because it uses relative imports.

**First start is very slow.**
EasyOCR and sentence-transformers download model weights on first use. Later starts are fast.

**PaddleOCR-VL fails to load.**
It needs `transformers>=5.0.0`, `torch`, and realistically a GPU. Without a GPU the router falls back to EasyOCR automatically.

**`ImportError` for `ocr_paddleocr_vl`.**
Make sure the file name matches the import in `ocr_router.py`.

**Hallmark always returns "not found."**
`hallmark_records` is empty. It is a mock table and must be seeded.

---

## Data and legal note

Standards metadata originates from Bureau of Indian Standards publications. Verify anything safety- or compliance-critical against the official source at [bis.gov.in](https://www.bis.gov.in) before relying on it.

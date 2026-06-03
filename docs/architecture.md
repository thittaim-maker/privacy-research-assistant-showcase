# System Architecture

## Overview

PRA follows a single-machine, fully offline architecture. All inference, retrieval, and storage happen on localhost. There are no outbound network calls at runtime.

```
┌─────────────────────────────────────────────────────────────────┐
│                        User's Machine                           │
│                                                                 │
│  ┌──────────────┐    ┌──────────────────────────────────────┐  │
│  │   Browser    │───▶│         Streamlit (app.py)           │  │
│  │ (localhost:  │    │                                      │  │
│  │    8501)     │◀───│  Auth │ Chat │ Upload │ Admin        │  │
│  └──────────────┘    └───────┬──────────────────────────────┘  │
│                              │                                  │
│               ┌──────────────┼──────────────────────┐          │
│               ▼              ▼                       ▼          │
│  ┌────────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │   LlamaIndex   │  │    Ollama    │  │      SQLite       │  │
│  │  Vector Store  │  │  localhost   │  │                   │  │
│  │  (30k+ chunks) │  │    :11434    │  │  pra_users.db     │  │
│  └────────────────┘  │              │  │  legal_sources.db │  │
│                       │  mistral:7B │  └───────────────────┘  │
│  ┌────────────────┐  │  phi3:3.8B  │                          │
│  │  Tesseract OCR │  │  nomic-emb  │  ┌───────────────────┐  │
│  │  + pdf2image   │  └──────────────┘  │  Encrypted Files  │  │
│  └────────────────┘                    │  (Fernet AES-128) │  │
│                                        └───────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Component Breakdown

### Streamlit Frontend (`app.py`)

The entire application is a single Streamlit app (~18,000+ lines). It handles:

- User authentication flow (login, session management, inactivity timeout)
- Chat interface with conversation memory (7-day rolling window)
- Document upload, validation, and routing
- Admin panel (user management, corpus status, audit log viewer)
- PII scan results display

### Ollama (Local LLM Server)

Ollama runs as a background service on `localhost:11434`. Three models are used:

| Model | Role | Context Window |
|-------|------|---------------|
| `mistral:latest` (7B) | Case analysis & Q&A | 4,096 tokens |
| `phi3:latest` (3.8B) | Compliance checks & document analysis | Default |
| `nomic-embed-text` | Document and query embeddings | — |

All models are loaded and kept resident in RAM via `OLLAMA_KEEP_ALIVE=10m`. GPU offloading is disabled (`num_gpu=0`) to ensure CPU-only compatibility.

### LlamaIndex Vector Store

The vector index is persisted to disk in `Misc/saved_index/` and loaded at startup via a `LazyIndexProxy` (loads on first query, not at boot). The index currently contains **30,669 chunks** across all jurisdictions.

Retrieval configuration:
- `SIMILARITY_TOP_K = 20` — top-20 candidates retrieved
- `MAX_CONTEXT_CHARS = 18,000` — total context budget fed to LLM
- Per-doc cap: 12 chunks; per-jurisdiction cap: 8 chunks; total cap: 20 chunks

### SQLite Databases

| Database | Purpose |
|----------|---------|
| `pra_users.db` | User accounts, password hashes, roles, last-login |
| `legal_sources.db` | Case metadata, source URLs, CELEX numbers, titles, FTS index |

The legal sources DB is mounted read-only at runtime and promoted to writable only during approved admin sync operations.

### Encrypted Storage

Conversation memory and the local encryption key are stored encrypted on disk using the Python `cryptography` library (Fernet, AES-128-CBC + HMAC-SHA256). The key file lives at `Config/local_storage.key` with `0o600` permissions.

---

## Data Flow: Standard Query

```
User types query
      │
      ▼
1. Auth check (session token verified)
      │
      ▼
2. PII pre-scan (query scanned for personal data)
      │
      ▼
3. Jurisdiction / intent detection
      │
      ▼
4. nomic-embed-text embeds the query
      │
      ▼
5. LlamaIndex vector search (top-20 chunks)
      │
      ▼
6. Metadata enrichment (case titles from SQLite — zero LLM)
      │
      ▼
7. Context sanitisation
   - Strip court headers, metadata blocks, separator lines
   - Wrap in <source_text> XML tags (prompt injection barrier)
   - Apply char budget (18,000 chars)
      │
      ▼
8. Prompt construction + Ollama (mistral) call
   - Wall-clock timeout: 120 seconds
   - Non-streaming (true timeout enforcement)
      │
      ▼
9. Response + source citations rendered in UI
      │
      ▼
10. Query logged to JSONL (query_logs.jsonl)
```

---

## Data Flow: Document Upload

```
User uploads file (PDF / TXT / MD / JSON / CSV / HTML / code)
      │
      ▼
1. Filename validation (extension whitelist, path traversal check)
      │
      ▼
2. Size check (200KB for temp analysis; 5MB for persistent upload)
      │
      ▼
3. PII scan (pii_scanner.py)
   - Pattern registry + optional LLM confirmation
   - Results shown to user before ingestion proceeds
      │
      ▼
4. Text extraction
   - PDF: pdfminer → fallback to Tesseract OCR (for scanned docs)
   - HTML: BeautifulSoup
   - CSV: extract_csv_text()
      │
      ▼
5. Routed to phi3 for document-summary analysis
   (bypasses case-analysis prompt to prevent hallucinated rulings)
      │
      ▼
6. Answer + source returned; temp file purged after session
```

---

## Monthly Corpus Sync

`monthly_sync.py` is run by an admin. It:

1. Reads `eu_cases_manifest.json` (26 entries)
2. Auto-derives CELEX identifiers from CJEU case numbers
3. Fetches full-text opinions from EUR-Lex (replaces earlier CURIA scrape which broke)
4. Rebuilds the LlamaIndex vector index from the updated corpus
5. Writes a new integrity manifest (`legal_sources.db.integrity.json`)
6. Rotates the previous index to `pending_index_swap.json` for zero-downtime swap

---

## Deployment Constraints

- **RAM target:** 8 GB (CPU-only laptop)
- **Model footprint:** Mistral 7B ≈ 4.6 GB, vector index ≈ 1.5 GB, Streamlit ≈ 1 GB
- **Port:** Fixed to 8501 via `.streamlit/config.toml`
- **No GPU required**

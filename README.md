# Privacy Research Assistant (PRA)

> A fully offline, air-gapped AI research tool for privacy law — built for legal teams, compliance officers, and privacy professionals who cannot expose sensitive queries to cloud AI services.

---

## What Is PRA?

The Privacy Research Assistant is a Retrieval-Augmented Generation (RAG) application that lets users ask natural-language questions about privacy law and receive grounded, cited answers — entirely on-device, with no internet connection required at runtime.

It indexes a curated corpus of statutes, regulations, and landmark court judgments across multiple jurisdictions (EU, India, United States) and combines vector search with a locally-hosted LLM to synthesise answers.

---

## Screenshots

| Login | Main Chat Interface | Source Citations |
|-------|---------------------|-----------------|
| ![Login](screenshots/login.png) | ![Chat](screenshots/chat.png) | ![Sources](screenshots/sources.png) |

| Document Upload | Admin Panel | PII Scanner |
|----------------|-------------|-------------|
| ![Upload](screenshots/upload.png) | ![Admin](screenshots/admin.png) | ![PII](screenshots/pii.png) |

> Screenshots are added progressively as features are demonstrated. See [`screenshots/`](screenshots/) for the full set.

---

## Key Capabilities

| Capability | Detail |
|-----------|--------|
| **Natural-language Q&A** | Ask questions in plain English; receive structured, cited answers |
| **Multi-jurisdiction corpus** | EU (GDPR, ePrivacy, CJEU case law), India (DPDP Act), US (FTC enforcement) |
| **Document upload & analysis** | Upload PDFs/TXT/JSON for ad-hoc analysis; OCR support for scanned documents |
| **PII detection** | Built-in scanner flags personal data in uploaded documents before ingestion |
| **Fully offline** | No API calls to OpenAI, Azure, or any cloud service at runtime |
| **Role-based auth** | Per-user SQLite authentication with PBKDF2-SHA256 (200,000 iterations) |
| **Encrypted local storage** | Conversation memory and config encrypted with Fernet symmetric encryption |
| **Admin audit trail** | Every privileged action is written to an append-only JSONL audit log |

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Frontend** | [Streamlit](https://streamlit.io/) |
| **LLM inference** | [Ollama](https://ollama.ai/) — Mistral 7B (Q&A), Phi-3 3.8B (compliance/doc analysis) |
| **Embeddings** | nomic-embed-text via Ollama |
| **Vector retrieval** | [LlamaIndex](https://www.llamaindex.ai/) — 30,000+ indexed chunks |
| **Structured data** | SQLite (auth, legal sources, case metadata) |
| **OCR** | Tesseract 5 + pdf2image / Poppler |
| **Encryption** | Python `cryptography` (Fernet) |

---

## Repository Structure (this repo)

```
pra-showcase/
├── README.md                    ← you are here
├── docs/
│   ├── architecture.md          ← system design and data flow
│   ├── features.md              ← detailed feature walkthrough
│   ├── legal-corpus.md          ← what laws and cases are indexed
│   ├── rag-pipeline.md          ← how retrieval and generation works
│   └── security.md              ← offline design, auth, encryption
└── screenshots/                 ← UI screenshots
```

---

## Knowledge Base Articles

- [System Architecture](docs/architecture.md)
- [Feature Walkthrough](docs/features.md)
- [Legal Corpus Coverage](docs/legal-corpus.md)
- [RAG Pipeline Design](docs/rag-pipeline.md)
- [Security & Privacy Design](docs/security.md)

---

## Design Principles

1. **Air-gapped by design** — no data ever leaves the machine; suitable for regulated environments
2. **Source-grounded answers** — every response includes the chunk, document, and jurisdiction it came from
3. **Prompt injection hardening** — retrieved legal text is wrapped in XML tags and sanitised before the LLM sees it
4. **No hallucinated authority** — non-opinion documents use document-summary prompts, not case-analysis headings
5. **Deterministic citation** — case titles are resolved from SQLite (zero LLM calls for metadata)

---

*This repository is a showcase of the PRA tool's knowledge base and UI. The full source code is maintained in a private repository.*

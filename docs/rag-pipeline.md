# RAG Pipeline Design

PRA uses a Retrieval-Augmented Generation (RAG) architecture. This document explains the design decisions made to improve reliability and safety over a naïve RAG implementation.

---

## Why RAG?

Privacy law Q&A cannot rely on an LLM's training data:

- **Recency:** CJEU judgments and regulatory decisions post-date most model training cutoffs
- **Hallucination risk:** LLMs readily invent plausible-sounding case holdings
- **Citation fidelity:** Legal work requires traceable, verifiable sources
- **Air-gap requirement:** No cloud model API can be called

RAG grounds every answer in retrieved text from the actual legal documents, with citations visible to the user.

---

## Retrieval Stage

### Embedding

Queries and documents are both embedded with **nomic-embed-text** running locally in Ollama. This model is chosen for:
- Strong performance on long-form text (legal documents average several thousand tokens)
- Small footprint (~270 MB)
- No truncation issues at typical chunk sizes

### Chunking Strategy

Documents are chunked at ingest time before embedding. Chunk size is tuned so that:
- Each chunk captures a coherent legal argument or article section
- The top-K chunks fit within the LLM's context window

### Retrieval Parameters

```
SIMILARITY_TOP_K = 20       # candidates from vector search
MAX_PER_DOC = 12            # deduplicate: max chunks from one document
MAX_PER_JURISDICTION = 8    # balance: max chunks from one jurisdiction
MAX_TOTAL_NODES = 20        # hard cap on context nodes
MAX_CONTEXT_CHARS = 18,000  # character budget for LLM context
```

The per-jurisdiction cap prevents one large regulation (e.g., GDPR) from dominating the context when a cross-jurisdiction query is asked.

---

## Sanitisation Stage

Retrieved chunks undergo sanitisation **before** being placed in the LLM prompt. This is a critical safety layer.

### What is stripped:

| Pattern | Reason |
|---------|--------|
| `Court:`, `Date:`, `Author:`, `Bench:` metadata headers | Prevent metadata confusion in LLM reasoning |
| `Equivalent citations:`, `case_number:`, `celex:`, `curia_docid:`, `source_url:` | Metadata not relevant to legal reasoning |
| `={10,}` separator lines | Visual separators become noise in the prompt |
| CourtListener metadata blocks | Structured XML-like blocks from US opinions |

### Prompt injection barrier:

All retrieved text is wrapped in XML tags before insertion into the prompt:

```xml
<source_text>
[retrieved legal text here]
</source_text>
```

This creates a clear boundary between the user's instruction and the retrieved data, making it significantly harder for adversarially crafted legal documents to override the system prompt or instruction.

---

## Prompt Construction

PRA uses different prompt templates depending on query type:

### Case Analysis (Mistral 7B)

Used when the query asks about a specific judgment or case holding.

```
You are a legal research assistant. Answer the user's question based 
only on the source texts provided below. Do not speculate beyond the 
provided text. Cite the source document for each claim.

<source_text>
[sanitised retrieved chunks]
</source_text>

Question: [user query]
```

### Compliance Check (Phi-3)

Used for questions like "Does X practice comply with GDPR Article 6?". A shorter, more structured prompt with a compliance-oriented template.

### Document Summary (Phi-3)

Used for uploaded documents. Deliberately avoids case-analysis headings (e.g., "Holding:", "Ratio Decidendi:") to prevent the model from hallucinating fictional court rulings on a contract or policy document.

---

## Generation Stage

### Model selection

| Query type | Model | Rationale |
|-----------|-------|-----------|
| Case law Q&A | `mistral:latest` (7B) | Better reasoning depth for complex legal questions |
| Compliance analysis | `phi3:latest` (3.8B) | Structured output, faster for yes/no compliance checks |
| Document analysis | `phi3:latest` (3.8B) | Faster turnaround for ad-hoc uploads |

### Timeout enforcement

All LLM calls use a **wall-clock timeout** (not a token budget):

```
CASE_ANALYSIS_REQUEST_TIMEOUT_SECONDS = 120.0
RETRIEVAL_TIMEOUT_SECONDS = 60.0
```

Streaming is **disabled** across all paths. This is intentional: streaming's incremental output makes true wall-clock timeout impossible (a streaming connection can hang without generating tokens). Non-streaming calls either complete or time out cleanly.

### Token budget

```
CASE_ANALYSIS_CONTEXT_WINDOW = 4,096 tokens
CASE_ANALYSIS_MAX_NEW_TOKENS = 300 tokens
```

On 8 GB RAM with Mistral loaded alongside the vector index, the KV cache has ~252 MB of headroom — sufficient for 4,096-token context at mistral's attention head sizes.

---

## Previous Approaches Removed

Several retrieval shortcuts were removed during hardening to eliminate stale-answer bugs:

| Removed component | Why removed |
|------------------|------------|
| `PRESEEDED_EXACT_QUERY_RESPONSES` | Static dict; became stale and surfaced outdated holdings |
| `sqlite_gdpr_statute_answer_for_query` | SQLite full-text search intercept bypassed vector retrieval |
| `sqlite_case_answer_for_query` | Same; worst offender — returned cached answers regardless of corpus updates |
| `case_answers_v12.db` persistent cache | Root cause of stale-answer bug; removed entirely |
| `fast_source_grounded_answer` (active path) | Disabled; body preserved as insertion point for future log-based cache |
| Streaming mode | Replaced with non-streaming + true wall-clock timeout |

The remaining pipeline is strictly: **one vector retrieval → one LLM call**. No shortcuts, no caches, no SQLite intercepts.

---

## Explicit File Retrieval

Users can bracket a filename in their query to bypass vector search and retrieve directly from a specific document:

```
[Schrems II.pdf] What was the Court's finding on supplementary measures?
```

The regex `\[([^\]]+\.(?:pdf|txt|md|...))\]` detects bracketed filenames (including those with spaces). The retrieval engine fetches chunks only from that document, then passes them through the normal sanitisation and generation pipeline.

---

## Source Citation Pipeline

Source citations are assembled **after** generation, not during:

1. Retrieved chunks carry metadata: `file_name`, `source_url`, `jurisdiction`, `source_type`
2. `lookup_case_titles()` does a single batch SQLite query to resolve all `case_title` values
3. `extract_sources()` enriches each source dict with the resolved title
4. `source_citation_label()` formats: `"Case Name (filename)"`
5. `source_excerpt_block()` renders the chunk excerpt with relevance score

Zero LLM calls are made for citation assembly — it is entirely deterministic SQLite + string formatting.

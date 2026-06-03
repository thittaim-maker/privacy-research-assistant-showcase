# Feature Walkthrough

## 1. Natural-Language Legal Q&A

The core feature. Users type questions in plain English and receive structured answers grounded in the indexed corpus.

**Example queries:**
- *"What did the CJEU hold in Schrems II regarding standard contractual clauses?"*
- *"Under GDPR Article 6, what is the lawful basis for processing employee data?"*
- *"How did the FTC approach data minimisation in the Epic Games settlement?"*

**Answer format:**
- Direct answer with legal reasoning
- Source citations: `Case Name (filename)` with relevance score
- Jurisdiction tag
- Legal research disclaimer (not legal advice)

**Answer length modes:** Configurable — `concise` (default) or `detailed`, set via environment variable `PRA_ANSWER_LENGTH_MODE`.

---

## 2. Multi-Jurisdiction Support

PRA covers three primary jurisdictions in a single interface:

| Jurisdiction | Coverage |
|-------------|---------|
| **European Union** | GDPR, ePrivacy Directive, Law Enforcement Directive, 22 CJEU judgments |
| **India** | Digital Personal Data Protection Act (DPDP) and related documents |
| **United States** | FTC enforcement actions via CourtListener |

Jurisdiction is auto-detected from the query text using signal keywords (e.g., "GDPR", "DPDP", "FTC"). Users can also bracket a filename to force explicit document lookup: `[FTC v Wyndham.pdf]`.

---

## 3. Document Upload & Ad-Hoc Analysis

Users can upload their own documents for one-off analysis without adding them to the permanent corpus.

**Supported formats:**

| Category | Extensions |
|---------|-----------|
| Documents | `.pdf`, `.txt`, `.md` |
| Data | `.json`, `.csv` |
| Web | `.html` |
| Code | `.py`, `.js`, `.ts`, `.java`, `.go`, `.rb`, `.cs`, `.cpp`, `.c`, `.sh`, `.yaml`, `.toml`, `.xml` |

**Size limits:**
- Temp analysis: 200 KB
- Persistent upload: 5 MB

**Processing pipeline:**
1. Extension whitelist check + path traversal validation
2. PII scan (see feature 4)
3. Text extraction (pdfminer for digital PDFs; Tesseract OCR for scanned)
4. Analysis via Phi-3 (document-summary prompt, not case-analysis)
5. File purged at session end

**Explicit file queries:** Bracket a filename in your question — `[contract.pdf] what are the data retention obligations?` — to direct the retrieval engine to that specific file.

---

## 4. PII Scanner

Uploaded documents and queries are scanned for personal data before ingestion.

**Detection categories:**

| Category | Examples Detected |
|---------|------------------|
| Identity | Names, Aadhaar numbers, passport numbers, PAN |
| Contact | Email addresses, phone numbers, physical addresses |
| Financial | Credit card numbers, bank account numbers, IFSC codes |
| Health | Medical record patterns |
| Digital | IP addresses, device identifiers |

**How it works:**
1. Pattern registry (`PatternRegistry`) runs regex-based detection
2. Flagged findings are shown to the user with context
3. Optional LLM confirmation pass for ambiguous matches
4. User decides whether to proceed with ingestion

---

## 5. Source Citations with Relevance Scores

Every answer includes structured source citations:

```
Source: Schrems II (C-311_18.pdf)
Jurisdiction: EU | Type: CJEU Judgment
Relevance: 0.82
Excerpt: "...the Court found that US surveillance law does not 
          ensure a level of protection essentially equivalent..."
```

- **Case titles** are resolved from the SQLite `legal_sources.db` in a single batch query — no LLM call for metadata
- **Relevance score** reflects vector similarity (cosine), not judicial authority
- Tooltip clarifies that relevance is retrieval-based, not a legal authority ranking

---

## 6. User Authentication

PRA ships with a full per-user authentication system.

| Feature | Detail |
|---------|--------|
| **Password hashing** | PBKDF2-SHA256, 200,000 iterations |
| **Session tokens** | Cryptographically random, stored in Streamlit session state |
| **Inactivity timeout** | 60-minute automatic logout |
| **Roles** | `admin` and `user` |
| **Admin actions** | User creation, password reset, role change — all audit-logged |

Admin password reset requires typing the exact confirmation string `RESET ADMIN ACCESS` to prevent accidental resets.

---

## 7. Conversation Memory

PRA maintains per-user conversation history as context for follow-up questions.

- **Retention:** 7 days rolling
- **Storage:** Encrypted on disk (Fernet AES-128)
- **Scope:** Per-session; memory is cleared on logout
- **Purpose:** Allows follow-up questions ("What about the Indian equivalent?") without re-stating context

---

## 8. Admin Panel

Available to `admin`-role users. Features:

- **User management:** Create, disable, reset passwords, change roles
- **Corpus status:** Live count of indexed documents per jurisdiction
- **Index health:** View last sync date, integrity manifest status
- **Audit log viewer:** Browse the append-only `admin_audit.jsonl`
- **Sync controls:** Trigger monthly corpus refresh (EU via EUR-Lex, US via CourtListener)
- **Storage report:** Disk usage breakdown by corpus shard

The admin panel can be hidden entirely from non-admin builds via `PRA_ENABLE_ADMIN_UI=0`.

---

## 9. OCR Support

Scanned PDFs (image-only, no embedded text) are handled via:

1. **pdf2image** converts PDF pages to images (Poppler backend)
2. **Tesseract 5** (bundled, no separate install required) performs OCR
3. Text is stitched and passed through the normal analysis pipeline

OCR is capped at 20 pages per document (`TEMP_PDF_OCR_MAX_PAGES`). A per-page text threshold (`TEMP_PDF_PAGE_TEXT_THRESHOLD = 25 chars`) determines whether OCR is needed per page.

---

## 10. Query Logging & Performance Monitoring

All queries are logged to JSONL files for post-hoc analysis:

| Log File | Contents |
|---------|---------|
| `logs/query_logs.jsonl` | Query text, jurisdiction, answer length, timestamp |
| `logs/query_timings.jsonl` | Wall-clock times for retrieval, LLM call, total |
| `logs/llm_failures.jsonl` | Timeouts and model errors with stack traces |

These logs are the designated insertion point for a future **weekly fast-track cache** — pre-warming answers for high-frequency queries based on usage patterns.

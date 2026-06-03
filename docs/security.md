# Security & Privacy Design

PRA was built for environments where data cannot leave the device. This document describes the security controls across every layer of the system.

---

## 1. Air-Gapped Architecture

**No data ever leaves the machine at runtime.**

- All LLM inference runs on `localhost:11434` via Ollama
- All embeddings are generated locally (nomic-embed-text)
- All vector retrieval is from a local LlamaIndex store on disk
- All database operations are local SQLite
- No telemetry, no analytics, no external API calls of any kind

The application is suitable for:
- Legal teams with data residency obligations
- Regulated industries (healthcare, finance) with AI usage restrictions
- Air-gapped government or enterprise environments

**Verification:** The application has no outbound HTTP client configured. The only network socket is Ollama on localhost:11434 and Streamlit on localhost:8501. Both can be verified with `netstat` at runtime.

---

## 2. Authentication

### Password Storage

User passwords are never stored in plaintext. The hashing scheme:

```
PBKDF2-HMAC-SHA256
├── Iterations: 200,000
├── Salt: cryptographically random (per user)
└── Output: 32-byte derived key, hex-encoded
```

200,000 iterations is aligned with NIST SP 800-132 (2023) recommendations for PBKDF2-SHA256 and OWASP guidance.

### Session Management

- Session tokens are generated with `secrets.token_urlsafe(32)` — cryptographically random, 32 bytes = 256 bits of entropy
- Tokens are stored in Streamlit session state (in-memory only; not persisted to disk)
- **Inactivity timeout:** 60 minutes — sessions are invalidated after 60 minutes of no interaction
- **Re-authentication:** Required after logout or timeout

### Admin Safeguards

- Admin password reset requires typing the exact confirmation string `RESET ADMIN ACCESS` to prevent accidental resets
- Role changes (`admin` → `user`, `user` → `admin`) are recorded in the audit log
- There is no "forgot password" flow (air-gapped; no email server)

---

## 3. Encrypted Local Storage

Sensitive data at rest is encrypted using **Fernet** (from the Python `cryptography` library):

| What is encrypted | Location |
|------------------|---------|
| Conversation memory | `conversation_memory.enc` |
| Encryption key | `Config/local_storage.key` |

Fernet uses **AES-128-CBC + HMAC-SHA256**. The key is generated once at first launch and stored with `0o600` file permissions (owner read/write only).

---

## 4. Legal Corpus Integrity

The legal corpus (statutes and case law) is protected against tampering:

### File Permissions

```python
# At startup, after index load:
os.chmod(LEGAL_SOURCES_DB_FILE, 0o444)          # read-only
os.chmod(f"{LEGAL_SOURCES_DB}.integrity.json", 0o444)
os.chmod(f"{LEGAL_SOURCES_DB}.last_good", 0o444)
```

The database is only promoted to writable (`0o600`) during approved admin sync operations and immediately reverted to read-only afterwards.

### Integrity Manifest

A SHA-256 manifest is written after every corpus rebuild:

```json
{
  "legal_sources.db": "sha256:abc123...",
  "saved_index/docstore.json": "sha256:def456...",
  "last_updated": "2026-06-01T12:00:00Z"
}
```

`verify_integrity_manifest()` is called at startup. A mismatch triggers an alert in the admin panel and can trigger automatic rollback to `last_known_good`.

---

## 5. PII Scanner

All user-uploaded documents are scanned for personal data **before** they are processed by the LLM or stored.

### Detection coverage:

| Data Type | Pattern Examples |
|-----------|----------------|
| Names | Common name patterns |
| Email addresses | RFC-compliant regex |
| Phone numbers | International formats including Indian mobile (+91) |
| Aadhaar numbers | 12-digit, space/hyphen separated |
| PAN numbers | Standard PAN format |
| Passport numbers | Country-specific patterns |
| IP addresses | IPv4 and IPv6 |
| Credit card numbers | Luhn-validated patterns |
| Bank account / IFSC | Standard formats |

The pattern registry (`PatternRegistry`) is extensible. New patterns can be added without code changes via the registry's rule format.

An optional LLM confirmation pass (using phi3) is available for ambiguous findings where regex alone cannot determine context.

---

## 6. Prompt Injection Hardening

Legal documents often contain instruction-like text that could confuse an LLM. PRA applies two defences:

### XML tag wrapping

All retrieved text is enclosed in `<source_text>...</source_text>` tags. Modern instruction-tuned models treat text inside these tags as data, not instruction.

### Document-type routing

Non-judicial documents (statutes, enforcement actions, uploaded contracts) use a **document-summary prompt**, not the case-analysis prompt. The case-analysis prompt uses structured headings (`Holding:`, `Ratio:`) that can trigger hallucinated rulings when applied to non-opinion documents. By routing document type at prompt selection time, this risk is eliminated.

---

## 7. Admin Audit Log

Every privileged action is written to `Config/admin_audit.jsonl`:

```json
{"timestamp": "2026-06-01T10:23:45Z", "actor": "admin", "action": "create_user", "target": "user@org.com", "ip": "127.0.0.1"}
{"timestamp": "2026-06-01T10:25:00Z", "actor": "admin", "action": "role_change", "target": "user@org.com", "new_role": "admin"}
```

The log is append-only (opened in `'a'` mode; no delete API exposed in the admin panel). It can be exported from the admin panel for compliance review.

---

## 8. Input Validation

### File uploads

- Extension whitelist enforced before any file processing begins
- Path traversal check: filenames are sanitised with `os.path.basename()` before any disk write
- Temp files are written to `Uploads/temp/` with random subdirectory names to prevent collisions
- All temp files are deleted at session end or after `SESSION_INACTIVITY_MINUTES` (60 minutes)

### Query text

- Queries are scanned by the PII scanner before being embedded
- No user query text is written to disk in plaintext (only to the encrypted memory file and the query log, which contains anonymised timing data)

---

## Threat Model: What PRA Is and Is Not Designed For

| Threat | Mitigated? | Notes |
|--------|-----------|-------|
| Data exfiltration to cloud | ✅ Yes | No outbound network calls |
| Prompt injection via documents | ✅ Yes | XML tags + prompt routing |
| Corpus tampering | ✅ Yes | SHA-256 integrity manifest + read-only permissions |
| Weak password hashing | ✅ Yes | PBKDF2-SHA256, 200k iterations |
| Session fixation | ✅ Yes | New random token on each login |
| Stale cached answers | ✅ Yes | Persistent answer cache removed |
| Multi-user data leakage | ✅ Yes | Per-session state; no shared conversation memory |
| Adversarial model (Ollama) | ⚠️ Partial | Models are local; supply chain risk is Ollama's |
| Physical device access | ❌ Out of scope | OS-level encryption (BitLocker) is the control |
| DoS / resource exhaustion | ⚠️ Partial | Wall-clock timeouts limit runaway LLM calls |

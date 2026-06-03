# Legal Corpus Coverage

PRA's knowledge base is built from primary legal sources — statutes and court judgments — fetched directly from authoritative sources and indexed into a vector store of **30,669 chunks**.

---

## European Union

### Statutes & Regulations

| Document | CELEX | Source |
|---------|-------|--------|
| General Data Protection Regulation (GDPR) | 32016R0679 | EUR-Lex |
| ePrivacy Directive (2002/58/EC) | 32002L0058 | EUR-Lex |
| Law Enforcement Directive (2016/680/EU) | 32016L0680 | EUR-Lex |

### CJEU Case Law (22 judgments)

| Case Name | Case Number | Significance |
|-----------|------------|-------------|
| **Google Spain v AEPD** | C-131/12 | Established the right to be forgotten / de-listing obligations |
| **Schrems I** | C-362/14 | Invalidated Safe Harbour; data transfers to US |
| **Schrems II** | C-311/18 | Invalidated Privacy Shield; SCCs require case-by-case assessment |
| **Digital Rights Ireland** | C-293/12 | Invalidated Data Retention Directive |
| **Tele2 Sverige** | C-203/15 | Bulk retention of communications metadata incompatible with EU law |
| **Breyer v Germany** | C-582/14 | Dynamic IP addresses constitute personal data |
| **Ryneš** | C-212/13 | Home CCTV capturing public space not exempt from GDPR |
| **Weltimmo v NAIH** | C-230/14 | Establishment test for territorial scope |
| **Wirtschaftsakademie** | C-210/16 | Joint controllership of Facebook fan pages |
| **Fashion ID** | C-40/17 | Website operators embedding FB Like button are joint controllers |
| **Planet49** | C-673/17 | Pre-ticked cookie consent boxes are not valid consent |
| **Orange Romania** | C-61/19 | Forced consent for unnecessary data processing unlawful |
| **Nowak v DPC** | C-434/16 | Exam scripts constitute personal data |
| **La Quadrature du Net** | C-511/18 | Bulk surveillance and metadata retention (bulk) |
| **Prokuratuur v H.K.** | C-746/18 | Access to traffic/location data — proportionality |
| **Ligue des droits humains** | C-817/19 | PNR data — compatibility with Charter |
| **Österreichische Post** | C-300/21 | Non-material damage under GDPR Article 82 |
| **Natsionalna agentsia** | C-340/21 | Mere infringement is not sufficient for damages |
| **Meta Platforms v Bundeskartellamt** | C-252/21 | GDPR as limit on data aggregation for advertising |
| **Lindqvist** | C-101/01 | Publishing personal data on a website = processing |
| **Manni** | C-398/15 | Company register data and data subject rights |
| **ASNEF and FECEMD** | C-468/10 | Legitimate interests and publicly accessible sources |

*Source: Full-text opinions fetched from EUR-Lex via CELEX number auto-derivation from CJEU case numbers.*

---

## India

| Document | Source | Notes |
|---------|--------|-------|
| Digital Personal Data Protection Act (DPDP) | Official gazette | Full text + explanatory memorandum |

*Monthly sync appends new notifications and rules as they are issued.*

---

## United States

| Source | Coverage |
|--------|---------|
| CourtListener (FTC docket) | FTC enforcement orders, complaints, consent decrees |

**Notable FTC cases indexed:**

- FTC v. Wyndham (data security, reasonable measures)
- FTC v. Epic Games (COPPA, dark patterns)
- FTC v. Facebook (Cambridge Analytica consent decree)
- Various Section 5 data security enforcement actions

*US corpus is fetched via the CourtListener public API (`/api/rest/v3/`) and repaired/reindexed monthly.*

---

## Corpus Statistics

| Metric | Value |
|--------|-------|
| Total indexed chunks | 30,669 |
| EU chunks (statutes + case law) | ~24,000 |
| India chunks | ~3,000 |
| US chunks | ~3,669 |
| Embedding model | nomic-embed-text |
| Index type | LlamaIndex VectorStoreIndex |
| Index format | Persisted to disk (Misc/saved_index/) |

---

## How the Corpus Is Maintained

1. **Manifest file** (`eu_cases_manifest.json`) — declarative list of all EU sources with case numbers and CELEX identifiers
2. **`eu_sync.py`** — auto-derives CELEX from CJEU case numbers (format: `C-XXX/YY` → `6YYYYTJXXX`) and fetches full opinions from EUR-Lex
3. **`monthly_sync.py`** — orchestrates full rebuild: fetch → extract → embed → index → integrity manifest
4. **Integrity manifest** (`legal_sources.db.integrity.json`) — SHA-256 hashes of all source documents; mismatch triggers rebuild
5. **Rollback** — previous index preserved as `last_known_good` snapshot; `restore_last_known_good()` available in admin panel

---

## What Is NOT Indexed

- Legal commentary, law review articles, or secondary sources
- Draft legislation or consultation documents
- National DPA decisions (DPC Ireland, CNIL, etc.) — planned for future corpus expansion
- Private contracts or internal policy documents (these are uploaded ad-hoc by users, not indexed permanently)

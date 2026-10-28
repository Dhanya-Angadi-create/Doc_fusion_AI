## Retrieval-Augmented Generation (RAG)

The analysis pipeline now uses **Sentence Transformers + FAISS + the selected LLM**:

1. Extracted source text is split into overlapping chunks.
2. `sentence-transformers/all-MiniLM-L6-v2` embeds the chunks.
3. FAISS performs cosine-style similarity search (normalized embeddings with inner-product search) to retrieve relevant evidence.
4. Retrieved chunks, including source IDs and chunk indexes, are added to the Gemini or Ollama analysis prompt.
5. Retrieved evidence is recorded in the analysis response's `source_evidence` entries with `type: "rag_retrieval"` for traceability.

The FAISS index is rebuilt for the current database source set on each analysis, preventing stale vectors after source deletion. The embedding model is cached in memory after its first load. The first RAG analysis needs an internet connection to download the embedding model unless it is already cached locally. The model runs locally; Gemini API usage remains subject to Google's API quota and billing terms, while Ollama runs locally.

Configuration in `.env`:

```env
RAG_ENABLED=true
RAG_EMBEDDING_MODEL=sentence-transformers/all-MiniLM-L6-v2
RAG_TOP_K=12
```

Set `RAG_ENABLED=false` to temporarily disable retrieval. The API also accepts an optional analysis query, for example `POST /api/analyze?provider=gemini&query=Find%20conflicting%20project%20deadlines`. When no query is supplied, it retrieves evidence for general document reconciliation. Install dependencies from `backend/requirements.txt`; the new dependencies are `sentence-transformers` and `faiss-cpu`.

**Limitations:** vector similarity identifies relevant text, not truth. DocFusion's existing deterministic conflict checks, source IDs, hashing, signatures, and verification remain necessary. This initial implementation retrieves evidence for each analysis request; it does not yet persist a FAISS index between restarts.



## AI provider selection

DocFusion AI supports two AI modes from the Sources page:

- **Public AI — Google Gemini:** cloud analysis using the backend's `GEMINI_API_KEY`.
- **Private AI — Local Ollama:** local analysis through an Ollama server, normally `http://localhost:11434`.

For Ollama, install Ollama, start it, and pull a model such as:

```powershell
ollama pull gemma3:4b
```

Configure `.env` if required:

```env
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_MODEL=gemma3:4b
OLLAMA_TIMEOUT=180
```

The frontend automatically checks Ollama and lists installed models. Select **Private AI — Local Ollama** before running analysis.


## Access control (roles)

Sign in at `/static/login.html`. Four roles, each a strict superset of the one before:

| Role | Can do |
|---|---|
| `viewer` | View dashboard, documents; download |
| `analyst` | + upload sources, run analysis, generate signed documents, run verification |
| `field_supervisor` | + accept or decline verification changes |
| `admin` | + delete sources |

Demo accounts (password `docfusion`): `control_room`, `analyst`, `supervisor`, `admin`. **Change them before any real use** by setting `DEMO_USERS="name:password:role,..."` in `.env`. Tokens are HMAC-signed and expire after `TOKEN_TTL_MINUTES` (default 480); the secret is generated into `data/keys/auth_secret` unless `AUTH_SECRET` is set. Five failed logins lock a username for five minutes. Every audit entry records who acted. Set `AUTH_ENABLED=false` only for local debugging.

This is deliberately basic: no password hashing or user database, no per-document permissions, and no SSO. A production deployment would connect to the organization's identity provider.


## Recent product improvements

- Professional light UI refresh across dashboard, workspace, document view, and verification.
- Verification compares substantive document content independently of the uploaded filename, validates both the trusted and candidate signatures, and reports exact JSON field paths and before/after values.
- Candidate acceptance creates a newly signed current version and removes the superseded document plus generated verification files; decline leaves the current trusted document unchanged and records the decision.
- Dashboard KPIs include citation coverage, unresolved conflicts, provenance coverage, unsupported claims without source links, extraction success, OCR status, and redaction counts. Metrics are computed from stored records and should not be interpreted as independently benchmarked accuracy.
- Sources can be classified as PUBLIC, INTERNAL, CONFIDENTIAL, or RESTRICTED. PUBLIC document generation redacts common PII patterns before hashing/signing, and generation is blocked when it would lower a CONFIDENTIAL/RESTRICTED source's classification.
- Scanned PDF extraction attempts PyMuPDF's Tesseract OCR integration when text extraction finds blank pages. Tesseract must be installed separately on Windows for OCR to work; sources needing OCR are flagged instead of silently treated as fully extracted.

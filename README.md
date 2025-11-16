# SIH1706 — Intelligent Enterprise Assistant

Enhancing Organizational Efficiency through AI-driven Chatbot Integration

---

## Project Summary

Create a secure, scalable AI-driven enterprise assistant (chatbot) tailored for a large public‑sector organisation. The assistant will answer employee queries across HR, IT, events and general policies, process uploaded documents (OCR, summarization, keyword extraction), support email-based 2FA, filter profanity via a maintainable dictionary, and be architected to serve a minimum of 5 concurrent users with a target response time ≤ 5 seconds for normal queries.

This repository contains a complete implementation plan, code skeletons, deployment manifests, testing and demo artifacts suitable for a Hackathon / PPO submission.

---

## ✅ Deliverables (what this repo contains)

- `architecture/` — UML/sequence diagrams, component diagrams (PNG/SVG)
- `backend/` — FastAPI microservice(s) for chat API, auth, document processing
- `nlp/` — training & inference pipelines, embedding index, semantic search utilities
- `docproc/` — PDF parsing, OCR (Tesseract wrapper), summarizer & keyword extractor
- `frontend/` — React single-page app with login, file upload, chat UI
- `infra/` — Dockerfiles, docker-compose, Kubernetes manifests, Helm charts
- `tests/` — unit tests, integration tests, load tests (locust) and sample test data
- `report/` — Capstone report (PDF)
- `presentation/` — PowerPoint (PPTX) for demo
- `video/` — 3-minute demo script & storyboard
- `sample_data/` — example HR & IT policy docs (8–10 pages) and synthetic Q/A pairs

---

## Key Features

- Natural Language Understanding (intent classification + NER)
- Retrieval-augmented generation (RAG) for accurate, source‑grounded answers
- Document upload: OCR (for scanned PDFs), PDF parsing, extractive & abstractive summarization
- Keyword & entity extraction from uploaded documents
- Profanity filtering using a configurable dictionary and context-aware masking
- Email-based 2FA during login (OTP via email)
- Role-based access control (RBAC) — normal user vs admin
- Horizontal scalability with containerized microservices and optional Kubernetes
- Observability: logging, metrics (Prometheus), distributed tracing (OpenTelemetry)
- Load testing suite and response-time SLAs (target ≤ 5s under normal conditions)

---

## Architecture (high level)

1. **Client (React SPA)**
   - Login (email + password) -> request OTP (2FA) -> verify OTP -> open chat UI
   - Send user messages and file uploads via WebSocket/REST
   - Show source snippets for RAG answers and allow "View source" for transparency

2. **API Gateway / Auth Service**
   - FastAPI endpoints for auth (JWT), 2FA (email OTP), user management
   - Rate limiting & request validation

3. **Chat Service (NLU + Dialogue Manager)**
   - Intent classification model + NER
   - Passage retrieval (vector DB e.g. FAISS or Milvus) over knowledge base + uploaded docs
   - Response generator (instructions + retrieved context) using a hosted LLM or locally served model

4. **Document Processing Service**
   - PDF parsing (pdfminer/pyMuPDF) and OCR fallback (Tesseract)
   - Extractive summarizer (TextRank) + abstractive summarizer (small transformer)
   - Keyword extraction (YAKE/RAKE) and entity linking
   - Store parsed text & embeddings in vector DB

5. **Profanity Filter Service**
   - Fast, token-level check against maintainable dictionary
   - Context-aware masking and admin-configurable actions (block/warn/auto-replace)

6. **Storage & Databases**
   - Relational DB for user, auth, metadata (Postgres)
   - Vector DB for embeddings (FAISS/Milvus/Weaviate)
   - Blob storage for uploaded files (S3 / local minio)

7. **Monitoring & CI/CD**
   - Prometheus + Grafana dashboards
   - Logs shipped to ELK / Loki
   - CI pipelines for lint/test/build and CD to Kubernetes

---

## Technology Stack (recommended)

- Frontend: React + TypeScript, Tailwind CSS
- Backend: FastAPI (Python 3.10+)
- ML/NLP: Hugging Face Transformers, sentence-transformers, PyTorch
- Vector DB: FAISS (local/demo) / Milvus or Weaviate (production)
- Storage: Postgres, MinIO (S3 compatible)
- OCR: Tesseract (pytesseract)
- Summarization: BART / T5 small for demo (fine-tuned optionally)
- Profanity Filter: custom dictionary + fast Aho-Corasick implementation
- Auth/2FA: JWT for session + SMTP email service (SendGrid/SES or SMTP dev server)
- Container orchestration: Docker Compose for local, Kubernetes for scale
- Load testing: Locust + k6
- CI: GitHub Actions

---

## Implementation Plan (Milestones)

**M0 — Project Setup (Day 0–1)**
- Repo skeleton, Dockerfiles, base FastAPI app + React scaffold
- Configure Postgres & MinIO docker-compose

**M1 — Basic Chat + KB Retrieval (Day 2–4)**
- Simple intent classifier & embedding-based retrieval (sentence-transformers)
- Basic RAG: retrieve top-k passages and use a deterministic template response

**M2 — Document Processing (Day 4–7)**
- PDF parsing, OCR fallback, extraction pipeline
- Store parsed docs & embeddings in vector DB
- Implement summarization & keyword extraction endpoints

**M3 — Authentication & 2FA (Day 6–8)**
- Email OTP flow + JWT session handling
- RBAC admin interface to manage profanity dictionary

**M4 — Profanity Filter & Safety (Day 7–9)**
- Aho-Corasick dictionary matcher with configurable actions
- Integrate filter in input pipeline and redaction rules in the response generator

**M5 — Performance & Scaling (Day 8–11)**
- Add basic horizontal scaling (gunicorn/uvicorn workers), Docker images, simple K8s manifests
- Locust load test to ensure response-times ≤ 5s for 5 concurrent users

**M6 — Polish & Demo (Day 11–14)**
- UI polish, sample data, generate report & PPT, record 3-minute demo

---

## API Spec (selected endpoints)

- `POST /auth/login` — body: `{email, password}` → starts login, returns `temp_token`
- `POST /auth/otp` — body: `{temp_token, otp}` → returns JWT access token
- `GET /chat/history` — returns user chat history (requires JWT)
- `POST /chat/message` — body: `{message, session_id}` → returns reply (JSON with `answer`, `sources`, `latency_ms`)
- `POST /chat/upload` — multipart file upload → returns `doc_id`, `status` (parsing starts)
- `GET /doc/{doc_id}/summary` — returns extractive & abstractive summary, keywords
- `POST /admin/profanity` — add/remove words in profanity dictionary (admin only)

Response format for `/chat/message`:

```json
{
  "answer": "...",
  "sources": [{"id": "doc123", "snippet": "...", "page": 2}],
  "latency_ms": 410
}
```

---

## Document Processing (detailed)

1. **Ingest**: Upload file → store raw blob (MinIO) → enqueue job
2. **Parse**: If PDF text layer exists, extract with PyMuPDF/pdfminer; else run Tesseract OCR
3. **Clean**: normalize whitespace, remove headers/footers, segment by page/paragraph
4. **Embed**: chunk long text (≤ 512–1024 tokens) and compute sentence-transformer embeddings
5. **Index**: upsert embeddings & metadata into vector DB
6. **Summarize**: run extractive (TextRank) and optional abstractive pipeline
7. **Expose**: summary & keywords via `GET /doc/{doc_id}/summary`

Sample document for demo: an 8–10 page HR policy PDF (publicly available) placed under `sample_data/`.

---

## Profanity Filter Implementation

- Maintain a small JSON/YAML dictionary managed by admins.
- Use an Aho-Corasick trie for O(n + m) matching across tokens.
- Modes: `block` (reject message), `mask` (replace bad words `****`), `warn` (allow but flag for review).
- Implement contextual exceptions (e.g., when a term appears in a quoted policy name) with an allowlist.

---

## 2FA (Email OTP) Flow

1. User posts credentials to `/auth/login`.
2. Server validates and generates a `temp_token` and a short-lived OTP (6 digits) stored hashed in DB.
3. Server sends OTP to user's corporate email using SMTP/SendGrid.
4. User posts OTP + `temp_token` to `/auth/otp`.
5. Server verifies OTP, issues JWT access token (short expiry) and refresh token.

Security notes: OTPs must be rate-limited and logged. Consider CAPTCHAs to prevent abuse.

---

## Scalability & Performance

- **Local demo**: Docker Compose with 1 API worker, 1 worker for document processing, FAISS index in memory.
- **Production**: K8s with autoscaling, separate deployment for vector DB (Milvus) and model servers.
- **Response time optimizations**:
  - Precompute embeddings for knowledge base documents
  - Use an efficient retrieval (HNSW) and limit number of retrieved passages to top-5
  - Cache recent queries and their responses (Redis)
  - Warm-start small summarization models and keep GPU-backed model servers running for heavy ops
- **SLA target**: ≤ 5s per query for normal retrieval+templated responses; generative responses may take longer — measure and report.

Load testing plan: create 5 concurrent users (persisted sessions) in Locust simulating real chat interactions and document uploads; measure p95 latency.

---

## Security & Compliance

- Encrypt data at rest (Postgres + MinIO) and in transit (TLS)
- Audit logging for sensitive operations (file downloads, admin changes)
- Role-based access control for document visibility
- Data retention policy and a "redaction" feature for uploaded docs
- Do not store raw OTPs; store only salted-hashed OTPs and short TTLs

---

## Testing & Evaluation Metrics

- **Accuracy**: intent classification accuracy on held-out synthetic dataset
- **Retrieval quality**: MRR / Recall@k for KB retrieval
- **Latency**: median & p95 response time under target load
- **Robustness**: profanity filter false positive/negative rate on test set
- **Usability**: small human evaluation for clarity, helpfulness

---

## How to Run Locally (Developer Quickstart)

Prereqs: Docker, docker-compose, Node 18+, Python 3.10+

1. Clone the repo

```bash
docker-compose up --build
```

2. Create environment variables in `.env` (see `.env.example`): SMTP creds, JWT secret, DB URLs.
3. Seed sample KB & users:

```bash
python backend/scripts/seed_demo_data.py
```

4. Open frontend at `http://localhost:3000`, register/login with demo user, request OTP (email delivered to dev SMTP or dev console), verify and start chatting.

---

## Sample Outputs (examples)

**User:** "How many casual leaves do I have left?"

**Bot:** "According to the HR policy (Leave Policy v2.1), employees are entitled to 8 casual leaves per calendar year. I see from your leave history that you've used 3 so far — remaining 5. (Source: HR Leave Policy, page 3)."

**User uploads:** `joining_letter.pdf` (8 pages)
**Action:** `/chat/upload` → parsing → `/doc/{doc_id}/summary` returns 3-sentence abstractive summary + top 10 keywords.

---

## Demo Script (3 minutes)

1. 0:00–0:20 — Quick intro: problem statement and demo goals
2. 0:20–1:00 — Login + 2FA (show email OTP delivery flow)
3. 1:00–1:30 — Ask HR question; show source snippet and confidence
4. 1:30–2:00 — Upload 8-page policy PDF → show summarization & keyword output
5. 2:00–2:30 — Show profanity filter in action (try a message with blocked word) and admin panel modifying dictionary
6. 2:30–3:00 — Show scaling notes (locust results) and close with next steps

---

## Deliverables for Hackathon Submission

- Code (GitHub repo)
- Capstone report (detailed design + experiments)
- PPTX slides (presentation folder)
- 3-minute recorded demo video (mp4)
- Load test results (locust report)
- Sample dataset & demo user credentials

---

## Appendix & References

- Hugging Face Transformers — https://huggingface.co
- FAISS — https://github.com/facebookresearch/faiss
- FastAPI — https://fastapi.tiangolo.com
- PyMuPDF / pdfminer / Tesseract documentation

---



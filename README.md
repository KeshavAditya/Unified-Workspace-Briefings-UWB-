Unified Workspace Briefings (UWB)

One-liner: Ingest Slack + Google Drive, index with BM25 + vectors, enforce ACLs at query time, and generate grounded weekly briefings with citations — with real SLOs, retries, circuit breakers, tracing, and load-test receipts.

More than “just RAG”: async ingestion + idempotency, ACL-aware search, query planner (BM25/vector/hybrid), abstain & per-citation confidence.

Production-ish: retries, circuit breaker, DLQ, rate limiting, OpenTelemetry traces, Prometheus/Grafana, Redis caching with principal-scoped cache keys.

Proof, not vibes: load test scripts + a tiny relevance eval harness; update the badges with your real numbers.

Core features

Connectors: Slack Events API (webhooks), Google Drive Changes API (polling)

Indexing: chunking → embed (HF e5-base by default) → Postgres (pgvector + pg_trgm)

Retrieval: query planner chooses BM25/vector/hybrid weights per intent

RAG answers: grounded, with [doc_id, span, score] citations and abstain behavior

Weekly Briefings: pre-aggregated (materialized view) + citations + version diffs

Security: API keys (hashed) + short-lived JWT for web; app-layer ACLs

Reliability: retries w/ jitter, circuit breaker, DLQ + admin requeue

Observability: Prometheus metrics, Grafana dashboard, OTel traces

Performance: Redis cache (per-principal keys), p95 targets, freshness SLO

Architecture
Slack Webhooks ┐
               ├──> FastAPI (/webhooks/*) ──> Redis Queue ──> Workers
Google Drive ──┘                                 │
                                                 ├─ Normalize → Idempotent Upsert (Postgres)
                                                 ├─ Chunk → Embed (HF)
                                                 └─ Index: pgvector + pg_trgm

FastAPI (/search, /ask, /briefings)
  ├─ Resolve caller identities → ACL filter
  ├─ Query Planner → BM25 / Vector / Hybrid fusion
  └─ RAG with citations (+ abstain)

Redis (cache w/ principal keying)   Prometheus + Grafana   OpenTelemetry Traces

Tech stack

Backend: Python 3.11, FastAPI, Pydantic

Queue/Workers: Redis + RQ (or Celery/Redis)

DB: PostgreSQL 15, pgvector, pg_trgm

Cache/Rate limit: Redis

Embeddings/LLM: HF intfloat/e5-base (default) + pluggable LLM (OpenAI or local)

Observability: prometheus_client, Grafana, OpenTelemetry

Auth: hashed API keys + short-lived JWT sessions

Packaging/CI: Poetry or uv; GitHub Actions

Deploy: Docker Compose (local, or single EC2)

Getting started
Prereqs

Docker & Docker Compose

(Optional) OpenAI key for LLM calls

Slack app (Events API + signing secret), Google Drive OAuth credentials

Environment

Create .env:

POSTGRES_USER=uwb
POSTGRES_PASSWORD=uwb
POSTGRES_DB=uwb
DATABASE_URL=postgresql+psycopg://uwb:uwb@db:5432/uwb

REDIS_URL=redis://redis:6379/0
SECRET_KEY=change-me
OPENAI_API_KEY=optional

SLACK_SIGNING_SECRET=xxx
GOOGLE_CLIENT_ID=xxx
GOOGLE_CLIENT_SECRET=yyy

CHAOS_MODE=0              # set 1 to simulate failures/delays
BREAKER_THRESHOLD=5       # consecutive failures to open
BREAKER_COOLDOWN_S=120

Run
docker compose up --build
# API at http://localhost:8000 ; Prometheus at :9090 ; Grafana at :3000

Initialize DB & dashboard
# Alembic migration (if using Alembic)
docker compose exec api alembic upgrade head

# Seed a Grafana dashboard JSON (optional)
# docker compose exec grafana sh -c '...'

Data model (excerpt)
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_trgm;

CREATE TABLE users(
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT UNIQUE NOT NULL,
  name TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE api_keys(
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  key_hash TEXT NOT NULL,
  scopes TEXT[] NOT NULL,
  prefix TEXT NOT NULL,                -- show-only prefix for UX
  created_at TIMESTAMPTZ DEFAULT now(),
  revoked BOOLEAN DEFAULT FALSE
);

CREATE TABLE documents(
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  source TEXT CHECK (source IN ('slack','gdrive')) NOT NULL,
  external_id TEXT NOT NULL,
  title TEXT,
  path TEXT,
  meta JSONB DEFAULT '{}',
  acl  JSONB NOT NULL,                 -- {allow:[{provider:'slack','external_id':'U123'}], public:false}
  version TEXT,
  deleted BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(source, external_id)
);

CREATE TABLE chunks(
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  document_id UUID REFERENCES documents(id) ON DELETE CASCADE,
  text TEXT NOT NULL,
  span INT4RANGE,
  embedding VECTOR(768)
);

CREATE INDEX chunks_vec_ivf ON chunks USING ivfflat (embedding vector_cosine_ops) WITH (lists=100);
CREATE INDEX chunks_text_gin ON chunks USING GIN (to_tsvector('english', text));

Retrieval & Query Planner
def plan(query: str) -> dict:
    q = query.strip()
    if looks_like_id_or_title(q):         # filenames, PR IDs, exact mentions
        return {"mode":"exact","w_bm25":0.9,"w_vec":0.1}
    if len(q.split()) >= 6:               # conceptual/long queries
        return {"mode":"concept","w_bm25":0.3,"w_vec":0.7}
    return {"mode":"mixed","w_bm25":0.6,"w_vec":0.4}


Fusion: min–max scale BM25/vector scores; score = w_bm25*s_bm25 + w_vec*s_vec.
Abstain: if top-K < 3 or max_score < 0.35, return { "insufficient_evidence": true, "needs": [...] }.

Security & ACLs

App-layer ACL: request is authorized iff any caller identity (Slack user ID, Google email) appears in documents.acl or public:true.

API keys: stored hashed (bcrypt/argon2), prefix shown for UX; rotation endpoint overlaps validity for 15 minutes.

Rate limiting: Redis leaky bucket per API key (/search 60 rpm, /ask 20 rpm).

Audit log: who, action, resource, ts, ip.

ACL-aware cache keys:

cache_key = f"v1:search:{user.api_key_prefix}:{normalized(q)}:{filters_hash}"

Reliability

Retries: exponential backoff w/ jitter (e.g., 1s, 4s, 10s; max 3).

Idempotency: ON CONFLICT (source, external_id) upsert; optional version.

Circuit breaker: open after N consecutive provider failures; cooldown T seconds; Prom metrics emit open/close events.

DLQ: table of failed jobs after max retries; /admin/requeue/:id.

Idempotent upsert (sketch):

INSERT INTO documents (source, external_id, title, path, meta, acl, version, deleted, updated_at)
VALUES ($1,$2,$3,$4,$5,$6,$7,false,now())
ON CONFLICT (source, external_id)
DO UPDATE SET title=EXCLUDED.title, path=EXCLUDED.path, meta=EXCLUDED.meta,
              acl=EXCLUDED.acl, version=EXCLUDED.version, deleted=false, updated_at=now();

Freshness SLO

Definition: time from provider create/update → indexed & searchable.

Metric: uwb_freshness_seconds (histogram).

Target: median ≤ 60s, p95 ≤ 120s (tune polling/worker concurrency).

Observability

Metrics (Prometheus):

uwb_ingest_jobs_total{source,status}

uwb_ingest_duration_seconds_bucket{source}

uwb_queue_depth (RQ gauge)

uwb_api_requests_total{route,status}

uwb_api_latency_seconds_bucket{route}

uwb_cache_hits_total{route}

uwb_breaker_state{provider} (0=closed,1=open)

uwb_freshness_seconds_bucket{source}

Traces (OTel): spans for /search → planner → BM25 query → vector query → fusion → ACL filter → LLM call; link ingestion spans via trace_id.

API reference (minimal)

Auth: Authorization: Bearer <api-key> (for programmatic) or session JWT (web).

GET /healthz → liveness.

GET /metrics → Prometheus format.

POST /connect/slack (dev) → store secrets.

POST /connect/gdrive (dev) → store OAuth tokens.

GET /search?q=&from=&to=&limit=20
Returns { hits:[{doc_id, title, span, score}], cached: true|false, planner:{...} }

POST /ask with { "question": "...", "project":"opt", "from":"opt", "to":"opt" }
Returns { answer, citations:[{doc_id, span, score}], confidence, insufficient_evidence? }

GET /briefings/weekly?project=&week=YYYY-Www
Returns summary bullets each with citations; fast via materialized view.

POST /admin/requeue { "job_id": "..." }
Re-enqueue from DLQ (admin scope).

Headers: responses include x-cache: HIT|MISS, x-latency-ms.

Connectors
Slack (webhooks)

Create Slack app → enable Events API → subscribe to message/file events.

Verify Slack signature at /webhooks/slack.

Enqueue compact jobs (not full payload); workers fetch details via Web API.

Google Drive (polling)

Use Changes API with startPageToken; poll every N seconds/minutes.

Store last_seen_token per identity.

Weekly Briefings

Materialized view pre-aggregates per ISO week + (optional) project/workspace filters.

Each bullet lists citation markers linking to original Slack/Drive items.

If document version changed, display diff snippet in the briefing.

Redaction policy (optional but recommended)

Scrub PII & secrets at ingestion (emails, phones, API keys) → replace with tokens (<EMAIL>, <API_KEY>); store redaction fingerprints in documents.meta.redactions.

Chaos & failure demo

Set CHAOS_MODE=1 to randomly:

delay ~10% embedding jobs,

fail ~5% Drive calls,

drop ~2% cache reads.

Use Grafana to show queue depth and breaker state as they react; system should degrade gracefully.

Benchmarks
Load tests

Tooling: locust or k6 (included in /benchmarks)

Targets (tune to your box/cloud):

/search cached: p95 ≤ 300 ms @ 50 RPS

/ask cached: p95 ≤ 700 ms @ 10 RPS

Ingestion stability: ≥ 8k docs/hour, queue depth bounded

Relevance eval (mini harness)

~30 (query, relevant_doc_ids) pairs.

Report nDCG@10 and Recall@10 for BM25, vector, hybrid, planner (auto-weighted).

Expected: planner > hybrid > vector ≈ BM25 (varies by corpus).

Commit results to:

/benchmarks/REPORT.md
/benchmarks/relevance_results.json
/benchmarks/loadtest_screenshots/

Project structure
app/
  api/            # FastAPI routes
  auth/           # API keys, JWT
  models/         # SQLAlchemy
  schemas/        # Pydantic
  services/       # Slack, GDrive clients (w/ breaker)
  tasks/          # RQ/Celery jobs: fetch, normalize, embed, index, briefings
  retrieval/      # planner, bm25, vector, fusion
  rag/            # prompt build, citations, abstain
  metrics/        # Prom exporters
  security/       # redaction, audit log
  utils/
infra/
  docker-compose.yml
  grafana/
  prometheus/
  alembic/
benchmarks/
  locustfile.py or k6.js
  relevance_eval.py
  REPORT.md
ui/
  (optional) minimal web client

Testing

Unit: normalizers, chunker, ACL filter, planner

Integration: webhook → queue → worker → DB round-trip

E2E: seed fixtures → /search respects ACL; /ask returns citations; /briefings uses MV

CI: lint + tests + build (GitHub Actions)

Demo script (3–5 minutes)

Ingestion: post a Slack event (ngrok) → job enqueued → worker logs → chunks created.

Search (two users): same query, different ACLs → different results. Show x-cache HIT on second call.

Ask (abstain): ask a question with weak evidence → API returns insufficient_evidence:true.

Breaker: toggle a fake 503 for Drive → show breaker opens, DLQ grows, Grafana panels move.

Traces: open an OTel trace of /search showing planner → BM25/vec → fusion.

Briefings: load a weekly briefing; click a citation to original Slack/Drive.

Roadmap (optional stretch)

Semantic cache for common queries

Feature flags (enable/disable connectors at runtime)

WebSockets for live “freshness” ticks to UI

Multi-tenancy + per-tenant quotas

License

MIT (or your choice). Add third-party model/license notes if you ship weights.

Acknowledgements

pgvector, pg_trgm, intfloat/e5-base, Prometheus/Grafana, OpenTelemetry.

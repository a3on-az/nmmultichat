
# ⚙️ OPS-ROUTER-02 · NeuraMem Multimodel Router (Production Canonical)

**Version:** v1.0 (Production)  
**Status:** Canonical  
**Author:** NeuraMem Core / Hien Nguyen  
**Date:** 2025-11-09  
**Port:** `3088` (overridable)  
**Service Code:** `MCHAT-Router`

---



---

Q&A from advisors
| Theme                            | Why it matters                                           | What to do in Router v0.2                                                                  |
| -------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| **Auth & multi-user**            | Needed for group review and future cloud mode            | Add a simple JWT-based session manager; stub RBAC for `owner`, `guest`.                    |
| **Resilience**                   | Claude’s “what if a model dies” → must handle gracefully | Implement a circuit-breaker + retry queue for each model connector.                        |
| **Performance / token cost**     | Five parallel streams are expensive                      | Introduce model concurrency limit (default = 3); Router reports token cost in metrics.     |
| **Consensus overlay definition** | Needed by Canvas UI                                      | Compute semantic overlap post-generation via cosine > 0.8 on sentence embeddings.          |
| **Persona weighting**            | Needs formal mapping                                     | Router builds composite system prompt using weighted concatenation; expose weights in API. |
| **Testing & observability**      | Missing coverage                                         | Include `/health/detailed`, OpenTelemetry hooks, mocked providers for tests.               |
| **Message Queue**                | Decouple Extract                                         | Add Redis-based “insight-queue” between Router → Extract worker.                           |

---

### 🧱 Next steps

1. **Define Router API and lifecycle** (what I’ll generate next).
2. **Update UI spec** later to reference `persona_weights` and `agreement_metrics`.
3. **Set phased rollout:**

   * MVP: 2 models + static persona mix
   * MVP + 1: 5 models + consensus overlay
   * Full: async queue + metrics dashboard.

## 1) Purpose & Scope

The NeuraMem Router is the **orchestration layer** for multimodel inference:
- Retrieves context from the vector service (Pinecone proxy)  
- Constructs persona-weighted prompts  
- Dispatches requests to multiple model providers in parallel (OpenAI, Anthropic, Google, xAI/Grok, OpenRouter, Local)  
- Streams responses to the UI as SSE  
- Computes consensus/novelty metrics and forwards insights to `NM-Extract`

Out of scope: long-running research agents, document conversion pipelines (handled upstream), and canonical file writes (handled by Extract/Canvas).

---

## 2) Architecture Overview



UI (3008) ⇄ Router (3088) ⇄ Vector Service (3090)
⇵ ⇵
Providers Insight Queue → NM-Extract (3092)
(OpenAI, Anthropic, Google, xAI, OpenRouter, Local)


**Core modules**
- `auth`: JWT sessions, RBAC (`owner`, `guest`)  
- `retrieval`: vector search client (namespace-aware)  
- `persona`: prompt composer + weight mixer  
- `dispatch`: provider connectors + circuit breaker + retry policy  
- `stream`: SSE multiplexer (per-model channels → single client stream)  
- `metrics`: agreement/novelty, latency, token cost accounting  
- `insight`: formatter → Redis queue → Extract webhook

**Tech**
- Python 3.11 + FastAPI + Uvicorn  
- Pydantic v2 schemas  
- Async HTTP (httpx)  
- Redis (queue + rate limit buckets)  
- OpenTelemetry (optional)  
- Prometheus `/metrics`

---

## 3) Configuration

`.env` (or secrets vault)
```env
ROUTER_PORT=3088
VECTOR_URL=http://localhost:3090
EXTRACT_URL=http://localhost:3092/extract/ingest
REDIS_URL=redis://localhost:6379/0

OPENAI_KEY=
ANTHROPIC_KEY=
GOOGLE_KEY=
XAI_KEY=
OPENROUTER_KEY=

# Limits & resilience
ROUTER_MAX_PARALLEL_MODELS=3
ROUTER_SSE_HEARTBEAT_SEC=15
ROUTER_RETRY_BACKOFF_MS=200,500,1200
ROUTER_CIRCUIT_FAILS=3
ROUTER_CIRCUIT_COOLDOWN_SEC=30
TOKEN_BUDGET_MONTH_USD=500

4) API Surface
4.1 POST /v1/query (SSE stream)

Initiate a multi-model review with optional RAG retrieval.

Request (JSON)

{
  "session_id": "uuid",
  "query": "Critique the ethics section of FAB-03.",
  "models": ["gpt-5","claude-3.5","gemini-2.5","grok-2"],
  "namespace": "neuramem-canonicals",
  "top_k": 8,
  "persona_config": {
    "personas": [
      {"id": "canon_keeper", "weight": 0.45},
      {"id": "philosopher", "weight": 0.35},
      {"id": "jester", "weight": 0.20}
    ],
    "tone": "professional, compassionate, precise"
  },
  "rag": true,
  "metadata": {"artifact_ids": ["fab03","zv03"]}
}


Streamed events (SSE)

{"type":"model_token","model":"claude-3.5","delta":"The ethics section..."}
{"type":"model_done","model":"claude-3.5","usage":{"prompt":2451,"completion":612,"cost_usd":0.008}}
{"type":"metrics_partial","agreement":0.72,"novelty":0.41}
{"type":"error","model":"gemini-2.5","message":"rate_limited","retry_in_ms":500}
{"type":"final_metrics","agreement":0.78,"novelty":0.38,"latency_ms":6120,"cost_usd":0.034}


Behavior

If rag=true, Router calls Vector Service search(namespace, query, k) and assembles a context pack (see §6).

Dispatches to providers concurrently (bounded by ROUTER_MAX_PARALLEL_MODELS).

Emits heartbeat every ROUTER_SSE_HEARTBEAT_SEC to keep connections alive.

On provider failure: exponential backoff (config), then graceful skip with model_error event.

Auth

Authorization: Bearer <JWT> (local dev bypass toggle available).

4.2 POST /v1/insight

Manual or programmatic submission of a synthesized insight (usually from UI “promote insight”).

{
  "session_id":"uuid",
  "insight_text":"Consensus: autonomy clause should be explicit.",
  "model":"gpt-5",
  "persona":"canon_keeper",
  "metrics":{"agreement":0.82,"novelty":0.31},
  "vector_ids":["fab03:12","fab03:14"],
  "tags":["ethics","autonomy"]
}


Response: 202 Accepted (enqueued to Redis → Extract worker)

4.3 POST /v1/upload

Multipart artifact upload (proxied to ingestion service or stored temp for vectorization).

Returns { "artifact_id":"...", "metadata":{...} }

4.4 Health & Ops

GET /healthz → 200 OK

GET /health/detailed → dependencies status (Vector, Redis, Providers circuits)

GET /metrics → Prometheus

GET /version → git hash, build stamp

5) Schemas (Pydantic)
class PersonaWeight(BaseModel):
    id: Literal["canon_keeper","philosopher","archivist","jester","collective","apprentice"]
    weight: float

class PersonaConfig(BaseModel):
    personas: list[PersonaWeight]
    tone: str | None = None

class QueryRequest(BaseModel):
    session_id: str
    query: str
    models: list[str]
    namespace: str = "neuramem-canonicals"
    top_k: int = 8
    persona_config: PersonaConfig | None = None
    rag: bool = True
    metadata: dict[str, Any] | None = None

6) Prompt & Persona Composition

Context Pack (if RAG)

system_context: curated instructions + privacy constraints (Ω-Witness)

retrieved_chunks: top-k text snippets (with source ids, timestamps)

artifact_metadata: titles, tags, zones

Persona Mixer

Build composite system prompt via weighted concatenation:

For each persona: Wᵢ * persona_prompt[i] (normalized 0..1, soft-cap to avoid overlong system)

Add tone and task framing

Example (internal, not exposed):

SYSTEM:
[Canon Keeper 0.45]: enforce structure, cite chunks, avoid speculation.
[Philosopher 0.35]: weigh moral frames, clarity > cleverness.
[Jester 0.20]: surface contrarian but constructive challenges.
Tone: professional, compassionate, precise.

7) Consensus & Novelty Metrics

Sentence embeddings (e.g., text-embedding-3-small) computed per model segment after completion.

Agreement (Aₐ) = mean pairwise cosine similarity across final summaries.

Novelty (Nᵥ) = mean cosine distance from centroid (higher = fresher idea).

Consensus Overlay (UI) = phrases selected where cosine > 0.80 across ≥ 3 models (post-gen pass).

Metrics emitted as metrics_partial during streaming (rolling window) and final_metrics on completion.

8) Provider Connectors (Adapters)

Each connector implements:

async def infer(prompt: Prompt, ctx: ContextPack, stream=True) -> AsyncIterator[ModelEvent]


Built-in connectors

openai_gpt (GPT-5…)

anthropic_claude (Claude 3.5…)

google_gemini (Gemini 2.5)

xai_grok (Grok-2)

openrouter_generic (dynamic model name)

local_llama (Ollama/LMStudio)

Resilience

Circuit breaker (trip after ROUTER_CIRCUIT_FAILS; cooldown ROUTER_CIRCUIT_COOLDOWN_SEC)

Retry backoff sequence from env

Rate limit via Redis token buckets per provider key

Request dedupe: hash of (session_id, query, persona_config, models) for cache hit

Cost accounting

Each connector reports prompt/completion tokens and estimated USD; Router aggregates and enforces TOKEN_BUDGET_MONTH_USD (emit budget_warning SSE event when >80%).

9) Retrieval (Vector Service 3090)

POST /search

{"namespace":"neuramem-canonicals","query":"...", "k":8}


→ returns

{"chunks":[
  {"id":"fab03:12","text":"...","score":0.82,"meta":{"title":"FAB-03"}},
  ...
]}


Re-ranking

If metadata.artifact_ids present, boost those sources (+0.05)

Zone filter: exclude Red unless caller has owner role and local-mode flag set

10) Insight Pipeline (Queue → Extract)

On model_done, router can draft candidate insights (short bullets) and enqueue them.

Manual promote via /v1/insight also enqueues.

Redis payload:

{
  "session_id":"uuid",
  "content":"…",
  "model":"claude-3.5",
  "agreement":0.74,
  "novelty":0.29,
  "vector_ids":["fab03:12"],
  "tags":["ethics"],
  "created_at":"2025-11-09T17:10:00Z"
}


Extract worker persists to Markdown + updates __NM-01.

11) Auth, Privacy, RBAC

JWT issued by UI; claims: sub, role, exp.

Roles: owner (full), guest (no Red-zone, no key view).

Privacy Zones:

Green: routable to all providers

Amber: routable with redaction (PII scrub)

Red: never leaves localhost (Router verifies and blocks external calls; local_llama only).

Ω-Witness validator enforces policy before dispatch.

12) Error Handling & SSE Recovery

SSE reconnect guidance: client retries with Last-Event-ID and session_id → Router resumes from buffer if available.

Provider down → emit model_error and continue others.

Vector unreachable → fallback to query-only mode with rag=false notice.

Hard failures → 500 JSON body with error_code, hint, retry_in_ms.

13) Observability

/metrics Prometheus: latency per provider, P95, token cost, queue depth, cache hit rate.

OpenTelemetry spans: router.query, connector.infer, vector.search, extract.enqueue.

Logs per session: /logs/sessions/{session_id}.json (respect zones; redact Amber; skip Red content).

14) Testing Strategy

Unit: persona mixer, cost ledger, circuit breaker, metrics math.

Integration: mocked providers (fixtures), vector fake, Redis in-mem.

Contract: JSON schema validation of /v1/query and streaming events.

Load: k6 profile: 50 concurrent sessions, 3 models, 8 chunks.

15) Rollout Plan

MVP: OpenAI + Anthropic, RAG on, consensus metrics, /v1/query SSE.

MVP+1: Add Gemini + Grok + OpenRouter, Redis queue, /metrics.

Full: Web search extension (Gemini/OpenRouter), budget guardrails, OTel tracing.

16) Security Notes

Secrets only in server env; UI never receives provider keys.

CORS restricted to localhost:3008 by default.

Enforce max_tokens and stop sequences centrally; sanitize prompts.

GDPR delete: /admin/purge_session/{session_id} (owner only) → removes logs, queue items, and marks vectors for deletion via Vector Service.

17) Dev Setup
# 1) Start Router
uvicorn router.main:app --host 0.0.0.0 --port 3088 --reload

# 2) Dependencies
redis-server
VECTOR_URL=http://localhost:3090
EXTRACT_URL=http://localhost:3092/extract/ingest

# 3) Smoke test
curl -N -X POST "http://localhost:3088/v1/query" \
  -H "Content-Type: application/json" \
  -d '{"session_id":"demo","query":"hello","models":["gpt-5","claude-3.5"],"rag":false}'


End of File – OPS-ROUTER-02.md
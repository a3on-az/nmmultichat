# nm-multichat

---

# 🎨 FAB-MCHAT-01_UI · NeuraMem Multimodel Review Interface (Production Canonical)

**Version:** v1.0 (Production)
**Status:** Canonical
**Author:** NeuraMem Core / Hien Nguyen
**Date:** 2025-11-09
**Ports:** `localhost:30xx` (default `3008` UI, `3088` Router API, `3090` Vector Service)

Repository: github.com/a3on-az/nm-multichat
Service name: NeuraMem Council Interface
Internal code: MCHAT (for FAB / OPS prefixes)

---

## 🧭 Purpose

Deliver a cohesive multimodel interface for reviewing, conversing, and generating insights across the entire NeuraMem corpus — textual, visual, and audiovisual — integrating the **Canvas**, **Router**, and **Extract** layers.

This UI unifies:

* Multi-model “Council Mode” dialogue
* RAG-based artifact review (files, briefs, links, media)
* Persona and tone mixing
* Insight extraction and export to Router or Canvas

---

## ⚙️ Architecture Context

```
[Browser/Localhost:3008] → [Router API :3088]
         ↕                        ↕
     Pinecone Vector Service :3090
         ↕                        ↕
   NM-Extract / Canonical DB   Canvas Persistence
```

### Tech Stack

* **Frontend:** Next.js 15 + Tailwind + React Server Components
* **Backend:** FastAPI Router (`/v1/query`, `/v1/insight`, `/v1/upload`)
* **Realtime:** SSE (Server-Sent Events)
* **State:** Zustand / Supabase for session + persona config
* **Vector Layer:** Pinecone (canonical namespace)

---

## 🧩 Core UI Regions

### 1️⃣ Header Bar

| Element              | Function                                                             |
| -------------------- | -------------------------------------------------------------------- |
| **Model Selector**   | Toggle individual models (GPT-5, Claude, Gemini, Grok, OpenRouter).  |
| **Persona Preset**   | Quick dropdown: *Academic Board*, *Startup Board*, *Elders Council*. |
| **RAG Toggle**       | Switch between memory-off and full NeuraMem corpus retrieval.        |
| **Session Controls** | Start new review / resume prior session / export transcript.         |

---

### 2️⃣ Artifact Drop Zone

* Drag-and-drop files, PDFs, images, videos, or paste YouTube links.
* Auto-detect type → build RAG context (`namespace=neuramem-canonicals`).
* Displays thumbnail + tags + auto-salience + privacy zone.

**Supported Inputs**

| Type                    | Handling                            |
| ----------------------- | ----------------------------------- |
| `.md`, `.pdf`, `.docx`  | Extract + embed text                |
| `.png`, `.jpg`          | Gemini image embedding              |
| `.mp4`, `.mov`, `.webm` | Frame sampling + caption extraction |
| `YouTube URL`           | Transcript + metadata ingestion     |

---

### 3️⃣ Council View (Central Panel)

Multi-column live chat showing each model’s stream.

| Column     | Content                        |
| ---------- | ------------------------------ |
| GPT-5      | Logical / structured analysis  |
| Claude     | Empathic / ethical reasoning   |
| Gemini     | Multimodal / factual grounding |
| Grok       | Contrarian / creative spark    |
| OpenRouter | Experimental / community model |

Features:

* Color accent per model (gold, teal, violet, neon, white).
* Real-time token streaming with subtle animation.
* “Consensus Overlay” — highlights shared phrases or conceptual overlaps.
* “Insight Buttons”: 👍 Insight / 🔍 Divergence / 🪶 Quote.

---

### 4️⃣ Persona Mixer Sidebar

* Sliders 0–1 weight per persona (influences Router dispatch).
* Live preview of composite prompt.
* “Save Persona Stack” → reusable preset stored in Supabase.

---

### 5️⃣ Artifact Viewer (Split Mode)

* Left: document or media viewer.
* Right: synchronized comments and responses (time-coded for media).
* Hovering a text or video segment highlights which responses reference it.

---

### 6️⃣ Insight Ledger (Bottom Drawer)

Aggregates outputs with metrics:

| Field       | Meaning                                               |
| ----------- | ----------------------------------------------------- |
| `Agreement` | Mean cosine similarity of replies                     |
| `Novelty`   | Distance from centroid (freshness)                    |
| `Tone`      | Averaged affect vector                                |
| `Tags`      | Extracted topics                                      |
| `Actions`   | **Export to Router**, **Open in Canvas**, **Archive** |

---

### 7️⃣ Research Toolbar (post-MVP)

* Integrated lightweight web search (`Gemini / OpenRouter browse`).
* Collapsible panel showing fetched snippets + citations.
* “Add to RAG Context” button for approved snippets.

---

## 🧱 Layout Sketch

```
┌──────────────────────────────────────────────────────────────┐
│ Header Bar ─ models / personas / RAG / session / settings     │
├──────────────────────────────────────────────────────────────┤
│ Artifact Drop Zone / Viewer | Persona Mixer Sidebar          │
├──────────────────────────────────────────────────────────────┤
│ Council View: GPT-5 | Claude | Gemini | Grok | OpenRouter    │
├──────────────────────────────────────────────────────────────┤
│ Insight Ledger + Export Controls                             │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔐 Privacy / Zones

* Green = public content; Amber = personal reflection; Red = shadow / purge 24 h.
* Red-zone artifacts never leave local storage; vectors only.
* All model queries routed through Ω-Witness validator before storage.

---

## 🔮 Future Extensions

1. **Group Review Sessions** — multi-user councils eg. board of editors for journals, board of spiritual elders, board of unicorn startup founders
2. **Voice Modality** — microphone → transcript → review.
3. **Timeline Mode** — review evolution of a concept across canonicals.
4. **Affect Meter** — real-time coherence bar between models.
5. **Agentic Search** — delegated background literature scans.

---

## 🪜 Deployment & Ports

| Service                | Port   | Notes                     |
| ---------------------- | ------ | ------------------------- |
| `UI (Next.js)`         | `3008` | Main interface            |
| `Router API (FastAPI)` | `3088` | Model orchestration + RAG |
| `Vector Service`       | `3090` | Pinecone / local fallback |
| `Canvas Socket`        | `3091` | Live sync to NM-Canvas    |
| `Extract Worker`       | `3092` | Insight export + logging  |

All configurable via `.env.local`.

---

## ✅ MVP Acceptance Criteria

* Load document or media, run multi-model review, display streaming results.
* Save insights to ledger with metrics.
* Export to Router or Canvas.
* Local memory persistence via Pinecone namespace.
* Operable on custom ports 30xx with no conflicts.

---

## 📘 Integration Notes

* NM-Canvas receives session metadata (`session_id`, `insights`, `artifact_ids`).
* NM-Router uses unified `/v1/query` schema (JSON-SSE).
* NM-Extract consumes `/v1/insight` events.
* Vector store unified with ingestion spec 1.1 (`neuramem-canonicals` namespace).

---

Appendix


---

## 🔗 Appendix A · Integration Points

### 1️⃣ NM-Router (API Bridge)

**Purpose:** mediates all model calls, RAG retrieval, and persona dispatch.
**Protocol:** REST + SSE
**Port:** `3088`

| Endpoint      | Method | Input                                                               | Output                                  |
| ------------- | ------ | ------------------------------------------------------------------- | --------------------------------------- |
| `/v1/query`   | `POST` | `{ session_id, query, models[], namespace, persona_config, top_k }` | SSE stream of `{ model, chunk }` events |
| `/v1/insight` | `POST` | `{ session_id, insight_text, model, metrics{} }`                    | `202 Accepted`                          |
| `/v1/upload`  | `POST` | multipart file                                                      | `{ artifact_id, metadata }`             |

**Auth:** bearer token or local dev bypass.
**Notes:** must broadcast `session_id` so NM-Extract can reconcile streams.

---

### 2️⃣ NM-Canvas (Sync Layer)

**Purpose:** two-way sync of sessions and rendered insights.
**Protocol:** WebSocket / Supabase channel
**Port:** `3091`

| Event                   | Direction       | Payload                                  |
| ----------------------- | --------------- | ---------------------------------------- |
| `canvas.session.start`  | UI → Canvas     | `{ session_id, artifact_ids, personas }` |
| `canvas.session.update` | Router → Canvas | `{ insight, metrics }`                   |
| `canvas.session.close`  | UI → Canvas     | `{ session_id }`                         |

Canvas writes final summaries into `/canonical/` and updates Notion / DB references.

---

### 3️⃣ NM-Extract (Worker)

**Purpose:** post-processing of insights → canonical ingestion.
**Protocol:** HTTP / Webhook
**Port:** `3092`

| Endpoint              | Method | Description                                               |
| --------------------- | ------ | --------------------------------------------------------- |
| `/extract/ingest`     | `POST` | Receives structured insight objects for tagging + storage |
| `/extract/status/:id` | `GET`  | Worker job status                                         |

Downstream output: new or updated Markdown in `/canonical/` + index update in `__NM-01`.

---

### 4️⃣ Vector Service (Pinecone / Local)

**Purpose:** unified RAG retrieval for all multimodel queries.
**Protocol:** gRPC or HTTP wrapper
**Port:** `3090`

| Function                      | Args                           | Returns                  |
| ----------------------------- | ------------------------------ | ------------------------ |
| `search(namespace, query, k)` | text / embedding               | ranked chunks + metadata |
| `upsert(vectors[])`           | list of {id, values, metadata} | status                   |

**Namespace:** `neuramem-canonicals` (plus `youtube`, `contextual`, etc.)
**Embedding model:** `text-embedding-3-large`.

---

### 5️⃣ Auth / Identity Bridge

**Purpose:** harmonize provider credentials (OpenAI, Anthropic, Google, xAI, OpenRouter).
**Storage:** `.env.local` + encrypted secrets vault.
**Schema:**

```env
OPENAI_KEY=
ANTHROPIC_KEY=
GOOGLE_KEY=
XAI_KEY=
OPENROUTER_KEY=
```

Router rotates keys at runtime based on persona mix.

---

### 6️⃣ Storage / Persistence

| Layer            | Tech                | Path                                  |
| ---------------- | ------------------- | ------------------------------------- |
| Vector memory    | Pinecone / Chroma   | `neuramem-canonicals`                 |
| Metadata DB      | Postgres / Supabase | `public.insights`, `public.personas`  |
| Local archive    | JSON snapshots      | `/archive/sessions/{session_id}.json` |
| Canonical export | Markdown            | `/canonical/FAB-MCHAT-01_UI.md`       |

---

### 7️⃣ Dev Ports Summary

| Service          | Port   | Notes               |
| ---------------- | ------ | ------------------- |
| UI (Next.js)     | `3008` | Primary interface   |
| Router (FastAPI) | `3088` | Model orchestration |
| Vector Service   | `3090` | Pinecone proxy      |
| Canvas Socket    | `3091` | Session sync        |
| Extract Worker   | `3092` | Post-processing     |

All overridable via `.env.local`.

---

### 8️⃣ Logging & Telemetry

* Unified session logs under `/logs/sessions/{session_id}.json`.
* Metrics forwarded to optional Grafana (`/metrics/prometheus`).
* Privacy zones respected (no red-zone content logged).

---

### 9️⃣ Cross-Module Dependencies

| Depends On  | Used By          | Description            |
| ----------- | ---------------- | ---------------------- |
| NM-Router   | UI / Canvas      | Core orchestration API |
| NM-Extract  | Router           | Insight persistence    |
| Pinecone DB | Router / UI      | Vector retrieval       |
| Supabase    | UI / Canvas      | Session + persona sync |
| Ω-Witness   | Router / Extract | Provenance signing     |

---

### 10️⃣ Dev Setup Checklist

1. Clone NeuraMem core → `/ops/` + `/architecture/`.
2. Copy `.env.local.example` → `.env.local` and set API keys.
3. Run `pnpm dev --port 3008`.
4. Start Router `uvicorn router.main:app --port 3088`.
5. Verify health `GET /healthz` returns `200 OK`.

---

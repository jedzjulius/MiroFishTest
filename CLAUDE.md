# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**MiroFish** is a multi-agent simulation engine that builds a knowledge graph from user-supplied documents, populates it with AI agents, runs a social media simulation (Twitter/Reddit style via OASIS), and generates reports. It's a full-stack app: Vue 3 frontend + Python/Flask backend.

## Setup & Dev Commands

**Prerequisites:** Node.js 18+, Python 3.11–3.12, `uv` (Python package manager), Zep Cloud API key, LLM API key (OpenAI-compatible; Alibaba Qwen-plus recommended)

```bash
# First-time setup
cp .env.example .env        # Fill in LLM_API_KEY, LLM_BASE_URL, LLM_MODEL_NAME, ZEP_API_KEY
npm run setup:all           # Install all Node + Python dependencies

# Development
npm run dev                 # Start backend (port 5001) + frontend (port 5173) concurrently
npm run backend             # Backend only: cd backend && uv run python run.py
npm run frontend            # Frontend only: cd frontend && npm run dev

# Production
npm run build               # Build frontend assets
docker compose up -d        # Alternative: run everything via Docker (reads .env from root)

# Backend tests (run from backend/)
cd backend && uv run pytest
```

## Architecture

### 5-Step Workflow

```
User uploads documents
  → Step 1: Ontology generation + Zep knowledge graph construction
  → Step 2: Entity extraction + OASIS agent persona generation
  → Step 3: Multi-round social media simulation
  → Step 4: ReACT-pattern report generation
  → Step 5: Direct chat with simulated agents
```

### Backend (`backend/`)

Three Flask blueprints registered in `app/__init__.py`:
- `/api/graph` (`app/api/graph.py`) — File upload, ontology generation, graph build/status polling
- `/api/simulation` (`app/api/simulation.py`) — Entity filtering, profile generation, simulation start/status
- `/api/report` (`app/api/report.py`) — Report generation and agent chat

Key services:
- `OntologyGenerator` — LLM call to extract entity types and relations from uploaded documents
- `GraphBuilderService` — Chunks text, sends to Zep Cloud to build a knowledge graph (async)
- `SimulationRunner` — Spawns a subprocess running OASIS (camel-ai), communicates via IPC; handles multi-platform (Twitter/Reddit) agent actions
- `ReportAgent` — ReACT loop with Zep tools (search, insight, panorama, interviews) to generate structured analysis
- `ZepGraphMemoryManager` — Updates agent memory in Zep after each simulation round

State persistence: `Project` model (`app/models/project.py`) serializes project state as JSON to disk. `Task` model (`app/models/task.py`) tracks async operation progress (ontology gen, graph building).

### Frontend (`frontend/src/`)

Single-page app driven by `views/MainView.vue`, which manages step progression and panel layouts (graph / split / workbench).

- `components/Step1–5*.vue` — One component per workflow step
- `api/` — axios wrappers for each backend blueprint; implements retry logic and timeouts
- `store/pendingUpload.js` — Pinia store that persists pending file uploads across navigation
- `i18n/index.js` — Chinese/English localization via vue-i18n

### External Dependencies

| Service | Purpose |
|---------|---------|
| **Zep Cloud** | Stores entity knowledge graphs and per-agent memory |
| **LLM API** (OpenAI-compatible) | Ontology extraction, agent reasoning, report generation |
| **OASIS** (`camel-oasis` pip package) | Social media simulation engine (Twitter/Reddit agent behaviors) |

### Key Design Patterns

- **Async task polling:** Long-running operations (ontology gen, graph building) create a `Task` record and return a task ID; the frontend polls for completion.
- **Subprocess + IPC for simulation:** `SimulationRunner` spawns OASIS in a separate process to avoid blocking Flask; results are passed back via inter-process communication.
- **Project-scoped state:** All workflow state lives on the `Project` object persisted to disk — avoid passing large data structures through the frontend.
- **ReACT for report generation:** `ReportAgent` uses Reason-Act-Observe loops with tool calls against Zep rather than a single LLM prompt.

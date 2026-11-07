# GraphIQ — Order to Cash Graph Intelligence


[![Docker](https://img.shields.io/badge/docker-compose-blue.svg)](https://www.docker.com/)
[![Neo4j](https://img.shields.io/badge/Neo4j-5.18-green.svg)](https://neo4j.com/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.100+-teal.svg)](https://fastapi.tiangolo.com/)
[![LangGraph](https://img.shields.io/badge/LangGraph-0.2+-purple.svg)](https://langchain-ai.github.io/langgraph/)
[![React](https://img.shields.io/badge/React-18.3-blue.svg)](https://reactjs.org/)


A full-stack application for exploring and querying SAP Order-to-Cash (O2C) process data through a Neo4j knowledge graph, an AI chat agent, and a 3D interactive graph visualization.

### Bonus Features Implemented : 

 - Natural language to SQL or graph query translation
 - Highlighting nodes referenced in responses
 - Streaming responses from the LLM
 - Conversation memory
 - Advanced graph analysis
---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Prerequisites](#prerequisites)
- [Quick Start — Docker (Recommended)](#quick-start--docker-recommended)
- [Quick Start — Local Development](#quick-start--local-development)
- [Data Ingestion](#data-ingestion)
- [Environment Variables](#environment-variables)
- [API Reference](#api-reference)
- [Project Structure](#project-structure)
- [Graph Schema](#graph-schema)
- [Chat Agent](#chat-agent)
- [Running Tests](#running-tests)
- [Troubleshooting](#troubleshooting)

---

## Overview

Dodge AI connects SAP O2C JSONL data into a Neo4j property graph and exposes it through:

- **3D Force Graph** — interactive exploration of customers, orders, deliveries, invoices, payments, and products
- **AI Chat Agent** — LangGraph-powered agent that generates read-only Cypher queries from natural language questions and streams answers back via SSE
- **Analytics Panel** — integrity checks and structural summaries (broken flows, order volume buckets, top customers)
- **Shortest Path Tool** — find the shortest relationship path between any two nodes

---

## Architecture

```
Browser (React + Vite)
    │
    ├── GET /api/graph            → initial viewport (O2C spine)
    ├── GET /api/nodes/:l/:k/expand → neighborhood expansion
    ├── POST /api/path/shortest   → shortest path between nodes
    ├── GET /api/analytics/o2c    → integrity checks + summaries
    └── POST /api/chat (SSE)      → LangGraph agent stream
            │
            ▼
    FastAPI (Python)
            │
            ├── LangGraph Agent
            │     ├── Router (Groq LLM) — scope + tool selection
            │     ├── analyze_flow — O2C process explanation
            │     └── graph_query  — Cypher generation + Neo4j execution
            │
            ├── Neo4j (Bolt)       — read-only queries, graph traversal
            └── Redis              — session chat memory 
```


---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 18, TypeScript, Vite, react-force-graph-3d, Three.js |
| Backend | Python 3.12, FastAPI, uvicorn |
| Agent | LangGraph, LangChain, Groq (llama-3.3-70b-versatile) |
| Database | Neo4j 5.18 Community (APOC) |
| Cache | Redis (optional; falls back to in-process dict) |
| Container | Docker Compose |

---

## Prerequisites

- **Docker Desktop** (recommended) — Docker Engine 24+ and Docker Compose v2
- **OR** for local development: Python 3.12+, Node.js 20+, a running Neo4j 5.x instance
- **Groq API key** — free tier available at [console.groq.com](https://console.groq.com)
- SAP O2C JSONL data in `sap-o2c-data/` (see [Data Ingestion](#data-ingestion))

---

## Quick Start — Docker (Recommended)

### 1. Clone the repository

```bash
git clone https://github.com/srijan-op/dodgeai.git
cd dodgeai
```

### 2. Configure environment

```bash
cp .env.example .env
```

Edit `.env` and fill in your values:

```env
NEO4J_USER=neo4j
NEO4J_PASSWORD=your_strong_password_here
NEO4J_DATABASE=neo4j

GROQ_API_KEY_1=your_groq_api_key_here
GROQ_API_KEY_2=          # optional second key for fallback on rate limits
```

### 3. Start the stack

```bash
docker compose up -d --build
```

This starts four services: `neo4j`, `redis`, `backend`, and `frontend`.

### 4. Wait for Neo4j to be healthy

```bash
docker compose ps
# neo4j should show "(healthy)" before the backend accepts connections
# This can take 30–60 seconds on first boot
```

### 5. Apply constraints (first time only)

```bash
docker exec -it dodgeai-neo4j cypher-shell \
  -u neo4j -p your_strong_password_here \
  -f /var/lib/neo4j/import/01_constraints.cypher
```

If the `import/` folder isn't mounted, copy the file first:

```bash
cp cypher/01_constraints.cypher import/
docker exec -it dodgeai-neo4j cypher-shell \
  -u neo4j -p your_strong_password_here \
  -f /var/lib/neo4j/import/01_constraints.cypher
```

### 6. Ingest data

```bash
docker exec -it dodgeai-backend python scripts/ingest_o2c.py \
  --uri bolt://neo4j:7687 \
  --user neo4j \
  --password your_password_here
```

### 7. Materialize derived edges

```bash
docker exec -it dodgeai-neo4j cypher-shell \
  -u neo4j -p your_strong_password_here \
  "MATCH (soi:SalesOrderItem)-[:FULFILLED_BY]->(di:DeliveryItem)-[:INVOICED_AS]->(ii:InvoiceItem) MERGE (soi)-[:BILLED_BY]->(ii)"
```

### 8. Open the app

```
http://localhost:8080
```

API docs (Swagger UI): `http://localhost:8080/docs`  
Neo4j Browser: `http://localhost:7474`

---

## Quick Start — Local Development

### 1. Python backend

```bash
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

Create `.env` at the project root (see [Environment Variables](#environment-variables)), then:

```bash
uvicorn backend.main:app --reload --host 127.0.0.1 --port 8000
```

### 2. Frontend

```bash
cd frontend
npm install
npm run dev
```

The Vite dev server starts at `http://localhost:5173` and proxies `/api` to `http://127.0.0.1:8000`.

### 3. Optional: Redis for persistent chat memory

```bash
docker run -d -p 6379:6379 redis:alpine
```

Add to `.env`:

```env
REDIS_URL=redis://localhost:6379/0
```

Without Redis the backend uses an in-process dict (memory is lost on restart).

---

## Data Ingestion

Place SAP O2C JSONL files under `sap-o2c-data/<folder>/part-*.jsonl` following the structure below:

```
sap-o2c-data/
├── business_partners/
├── business_partner_addresses/
├── customer_company_assignments/
├── customer_sales_area_assignments/
├── sales_order_headers/
├── sales_order_items/
├── sales_order_schedule_lines/
├── outbound_delivery_headers/
├── outbound_delivery_items/
├── billing_document_headers/
├── billing_document_cancellations/
├── billing_document_items/
├── journal_entry_items_accounts_receivable/
├── payments_accounts_receivable/
├── products/
├── product_plants/
├── product_storage_locations/
├── product_descriptions/
└── plants/
```

Then run the ingestion script:

```bash
# Local
python scripts/ingest_o2c.py --uri bolt://localhost:7687 --user neo4j --password <password>

# Docker
docker exec -it dodgeai-backend python scripts/ingest_o2c.py \
  --uri bolt://neo4j:7687 --user neo4j --password <password>
```

After ingestion, materialize the derived `BILLED_BY` edges:

```bash
# Using cypher-shell inside the Neo4j container
docker exec -it dodgeai-neo4j cypher-shell \
  -u neo4j -p <password> \
  -f /var/lib/neo4j/import/02_derived_billed_by.cypher
```

Run post-load validation:

```bash
docker exec -it dodgeai-neo4j cypher-shell \
  -u neo4j -p <password> \
  -f /var/lib/neo4j/import/validation.cypher
```

---

## Environment Variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `NEO4J_USER` | ✅ | `neo4j` | Neo4j username |
| `NEO4J_PASSWORD` | ✅ | — | Neo4j password |
| `NEO4J_URI` | local only | `bolt://localhost:7687` | Bolt URI (Docker sets this automatically) |
| `NEO4J_DATABASE` | ✅ | `neo4j` | Logical database name |
| `GROQ_API_KEY_1` | ✅ | — | Primary Groq API key |
| `GROQ_API_KEY_2` | — | — | Fallback Groq key (used on rate limit) |
| `GROQ_MODEL` | — | `llama-3.3-70b-versatile` | Groq model identifier |
| `REDIS_URL` | — | `""` | Redis connection URL (empty = in-process dict) |
| `CORS_ORIGINS` | — | `http://localhost:8080,...` | Comma-separated allowed origins |
| `CHAT_HISTORY_MAX_TURNS` | — | `10` | Max conversation turns kept in memory |
| `CHAT_HISTORY_TTL_SECONDS` | — | `604800` | Redis key TTL (7 days) |
| `CYPHER_MAX_LIMIT` | — | `100` | Max `LIMIT` allowed in agent-generated Cypher |

---

## API Reference

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Health check — returns `{"status": "ok"}` |
| `GET` | `/api/graph?limit_rows=150` | Initial graph viewport (O2C spine) |
| `GET` | `/api/nodes/{label}/{key}/expand` | Expand a node's neighborhood |
| `POST` | `/api/path/shortest` | Shortest path between two nodes |
| `GET` | `/api/analytics/o2c?sample_limit=20` | O2C integrity checks and summaries |
| `POST` | `/api/chat` | LangGraph agent — streams SSE events |



---

## Project Structure

```
dodgeai/
├── backend/
│   ├── agent/
│   │   ├── llm.py           # Groq client with multi-key fallback
│   │   ├── models.py        # Pydantic models (RouterPlan, GraphQueryResult)
│   │   ├── pipeline.py      # LangGraph wiring + SSE streaming
│   │   ├── presenter.py     # GraphQueryResult → natural language
│   │   └── tools_run.py     # analyze_flow + graph_query implementations
│   ├── routers/
│   │   ├── analytics.py     # GET /api/analytics/o2c
│   │   ├── chat.py          # POST /api/chat (SSE)
│   │   ├── graph.py         # GET /api/graph
│   │   ├── nodes.py         # GET /api/nodes/:label/:key/expand
│   │   └── path_route.py    # POST /api/path/shortest
│   ├── chat_memory.py       # Redis / in-process session memory
│   ├── config.py            # Pydantic settings (reads .env)
│   ├── cypher_guard.py      # Read-only Cypher validation
│   ├── graph_schema.py      # Label → key property mapping
│   ├── main.py              # FastAPI app factory
│   ├── neo4j_db.py          # Driver lifecycle + read_session
│   ├── o2c_analytics.py     # Integrity checks, label counts
│   ├── prompts.py           # LLM prompt templates
│   ├── schema_provider.py   # Canonical schema text for prompts
│   ├── serializers.py       # Neo4j driver types → JSON
│   └── Dockerfile
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   ├── AnalyticsPanel.tsx
│   │   │   ├── ChatPanel.tsx
│   │   │   ├── GraphImportantNote.tsx
│   │   │   ├── GraphLegendMenu.tsx
│   │   │   ├── GraphNodePopover.tsx
│   │   │   └── GraphView.tsx
│   │   ├── api.ts           # API client + SSE parser
│   │   ├── App.tsx          # Root layout + state
│   │   ├── graphColors.ts   # Node color definitions
│   │   └── main.tsx
│   ├── nginx.conf           # Reverse proxy config
│   └── Dockerfile
├── cypher/
│   ├── 01_constraints.cypher
│   ├── 01_constraints_mcp_raw.cypher
│   ├── 02_derived_billed_by.cypher
│   └── validation.cypher
├── resources/
│   ├── neo4j/
│   │   ├── o2c_data_model.json
│   │   └── o2c_arrows_spine.json
│   └── prompts/
│       └── o2c_cypher_prompt.md
├── scripts/
│   ├── ingest_o2c.py
│   └── test_chat_agent_tools.py
├── tests/
├── docker-compose.yml
├── requirements.txt
├── requirements-prod.txt
└── .env.example
```

---

## Graph Schema

The canonical O2C graph has **16 node labels** and **21 relationship types**.

### Node Labels

| Label | Key Property | Source |
|---|---|---|
| `Customer` | `businessPartner` | `business_partners` |
| `SalesOrder` | `salesOrder` | `sales_order_headers` |
| `SalesOrderItem` | `salesOrderItemKey` | `sales_order_items` |
| `Delivery` | `deliveryDocument` | `outbound_delivery_headers` |
| `DeliveryItem` | `deliveryItemKey` | `outbound_delivery_items` |
| `Invoice` | `billingDocument` | `billing_document_headers` |
| `InvoiceItem` | `invoiceItemKey` | `billing_document_items` |
| `JournalEntry` | `journalLineKey` | `journal_entry_items_*` |
| `Payment` | `paymentKey` | Derived from AR clearing keys |
| `Product` | `product` | `products` |
| `Plant` | `plant` | `plants` |
| `StorageLocation` | `storageLocationKey` | `product_storage_locations` |
| `Address` | `addressKey` | `business_partner_addresses` |
| `CompanyCode` | `companyCode` | `customer_company_assignments` |
| `SalesArea` | `salesAreaKey` | `customer_sales_area_assignments` |
| `ScheduleLine` | `scheduleLineKey` | `sales_order_schedule_lines` |

### DB model diagram

![Graph DB Model](Graph%20DB%20model%202.png)

### Core O2C Flow

```
(Customer)-[:PLACED]->
  (SalesOrder)-[:HAS_ITEM]->
    (SalesOrderItem)-[:FULFILLED_BY]->
      (DeliveryItem)-[:INVOICED_AS]->
        (InvoiceItem)-[:PART_OF]->
          (Invoice)-[:POSTED_AS]->
            (JournalEntry)-[:CLEARED_BY]->
              (Payment)-[:MADE_BY]->(Customer)
```

`(SalesOrderItem)-[:BILLED_BY]->(InvoiceItem)` is a **derived shortcut** materialized after ingestion from the two-hop `FULFILLED_BY → INVOICED_AS` path.

---

## Chat Agent

The agent runs a **three-step LangGraph pipeline** per turn:

1. **Router** — classifies scope (`in_scope` / `off_topic` / `needs_clarification`) and selects tools
2. **Parallel execution** — `analyze_flow` (O2C process explanation) and/or `graph_query` (Cypher generation + Neo4j execution) run concurrently when both are needed
3. **Synthesis** — merges conceptual and data answers into a single response

### Cypher Safety

All agent-generated Cypher passes through `cypher_guard.py` before execution:

- Blocks `CREATE`, `MERGE`, `DELETE`, `DETACH`, `SET`, `REMOVE`, `DROP`, `LOAD`, admin `CALL`
- Requires `RETURN` and `LIMIT ≤ CYPHER_MAX_LIMIT`
- Single statement only
- Parameterized execution via the official Neo4j Python driver

### Testing the Agent Directly

```bash
# Test all tools (analyze_flow + graph_query + presenter)
python scripts/test_chat_agent_tools.py

# Graph query only
python scripts/test_chat_agent_tools.py --mode graph

# Analyze flow only
python scripts/test_chat_agent_tools.py --mode analyze

# Combined (both branches + synthesis)
python scripts/test_chat_agent_tools.py --mode combined
```

---

## Running Tests

```bash
# Install dev dependencies
pip install -r requirements.txt

# Run all tests
pytest

# Verbose output
pytest -v
```

---

## Troubleshooting

**Graph loads but shows no nodes**  
Run the ingest script and verify data was loaded:
```bash
docker exec -it dodgeai-neo4j cypher-shell \
  -u neo4j -p <password> \
  "MATCH (n) RETURN labels(n)[0] AS label, count(*) AS cnt ORDER BY label"
```


**CORS errors in browser during local dev**  
Make sure `CORS_ORIGINS` in `.env` includes your frontend origin (default `http://localhost:5173` for Vite dev):
```env
CORS_ORIGINS=http://localhost:5173,http://127.0.0.1:5173
```




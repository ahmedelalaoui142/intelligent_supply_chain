# Open-Source Intelligent Supply Chain System (Self-Learning, End-to-End)

This document is the **permanent architectural and procedural reference** for the entire project. All future development must strictly follow this document.

---

## Getting Started (Using This Repository)

### Prerequisites

- **Python 3.11+**
- **uv** — [Install uv](https://docs.astral.sh/uv/getting-started/installation/)
- **PostgreSQL** — for the data warehouse (local or Docker)
- **Ollama** (optional, for Phase 5) — [Install Ollama](https://ollama.ai/) and pull a model (e.g. `ollama pull llama3.2`)

### Requirements

All runtime dependencies are defined in **`pyproject.toml`** (no separate `requirements.txt`). The stack includes FastAPI, Prefect, PostgreSQL drivers, ChromaDB, OR-Tools, LightGBM, Statsforecast, sentence-transformers, and ingestion libraries (Kaggle, httpx, feedparser, pytrends, PRAW, Meteostat). Install them with **uv** (see below).

### Clone and install

```bash
git clone <repository-url>
cd intelligent_supply_chain
```

Create a virtual environment and install dependencies with **uv**:

```bash
uv sync
```

To include optional dev tools (pytest, ruff):

```bash
uv sync --extra dev
```

To include MinIO client (optional object storage):

```bash
uv sync --extra minio
```

### Environment variables

All credentials and environment-specific config must be set via environment variables (no hardcoded secrets). Use a `.env` file in the project root for local development.

**Instructions:**

1. **Copy the template**  
   From the project root:
   ```bash
   cp .env.example .env
   ```

2. **Edit `.env`**  
   Open `.env` and fill in your values. Leave optional variables empty if you do not use that service.

3. **Never commit `.env`**  
   The real `.env` file contains secrets and must not be committed. It is listed in `.gitignore`. Only `.env.example` (no secrets) is committed.

4. **Loading in code**  
   Use `python-dotenv` to load `.env` in application code (e.g. in `main.py` or before starting the API):
   ```python
   from dotenv import load_dotenv
   load_dotenv()  # loads .env from project root
   ```
   Or rely on your orchestration (Prefect, Docker) to inject env vars.

**Template:** See **`.env.example`** for the full list of variables, with comments. Summary:

| Variable | Required | Description |
|----------|----------|-------------|
| `DATABASE_URL` | Yes | PostgreSQL connection string for the warehouse |
| `KAGGLE_USERNAME` / `KAGGLE_KEY` | Yes (for DataCo) | Kaggle API credentials |
| `REDDIT_*` | No | Reddit API (PRAW) for external_events ingestion |
| `OLLAMA_BASE_URL` / `OLLAMA_MODEL` | No (Phase 5) | Local Ollama endpoint and model name |
| `MINIO_*` | No | Optional MinIO object storage |
| `PREFECT_*` | No | Override for remote Prefect server |
| `ENVIRONMENT` / `LOG_LEVEL` | No | App environment and logging |

### Run the project

- **Run the main entrypoint (when implemented):**  
  `uv run python main.py`

- **Run the API (when implemented):**  
  `uv run uvicorn services.api:app --reload`

- **Run orchestration flows (when implemented):**  
  `uv run prefect flow run ...` or start the Prefect worker/server as documented in `orchestration/`.

### Docker (optional)

PostgreSQL and other services can be run via Docker. A `Dockerfile` and `docker-compose.yml` will be added in later phases for a full local stack.

---

## 1. Project Overview

### Business Context: The Bullwhip Effect Problem

Supply chains suffer from the **bullwhip effect**: small fluctuations in end-customer demand amplify upstream, causing excess inventory, stockouts, and inefficiency. Lack of visibility into demand, inventory, and external risks makes planning reactive and brittle.

### Objective of the System

Build a **fully open-source intelligent supply chain management system** that:

- Forecasts demand with uncertainty
- Optimizes inventory and replenishment mathematically
- Detects and structures external risks using generative AI
- Ingests data daily from open APIs and scraping
- Runs a continuous self-learning loop

**Constraints:**

- **NO** proprietary APIs
- **NO** paid services
- **NO** OpenAI API
- Everything runs **locally** or with **open-source tools** only

### High-Level Architecture: 3-Layer Intelligent System

1. **Layer 1 — Forecasting Engine:** Time series and ML models for demand forecasting with uncertainty quantification.
2. **Layer 2 — Optimization Engine:** MILP-based inventory and replenishment optimization.
3. **Layer 3 — Risk Intelligence Layer:** RAG pipeline with open-source LLMs for risk detection and structured output.

---

## 2. Technology Stack (100% Open Source)

| Category | Technology |
|----------|------------|
| Language | Python |
| Database | PostgreSQL |
| Vector store | ChromaDB or FAISS |
| Orchestration | Prefect (or Airflow) |
| API | FastAPI |
| Optimization solver | OR-Tools (HiGHS or CBC) |
| ML (tabular) | LightGBM or XGBoost |
| Time series | Statsforecast or Prophet |
| Local LLM | Ollama + Llama / Mistral |
| Embeddings | sentence-transformers |
| Object storage (optional) | MinIO |
| Packaging & env | **uv** |
| Containers | Docker |

---

## 3. Data Sources

### A. Core Structured Data

- **DataCo Smart Supply Chain** — via Kaggle API (open dataset)

### B. External Open Data

- **GDELT** — news events
- **Google News RSS** — news feeds
- **Wikipedia Current Events** — current events
- **Open-Meteo** — weather
- **NOAA** — weather (optional)
- **Meteostat** — weather / climate
- **Federal Register API** — regulatory / policy
- **EU TARIC** — tariff / trade references
- **Google Trends** — pytrends
- **Reddit** — PRAW (public subreddits)

---

## 4. System Architecture

### Layer 1: Forecasting Engine

- **Time series + ML:** Statsforecast/Prophet for baseline; LightGBM/XGBoost for feature-rich forecasts.
- **Feature engineering:** Lags, calendar, promotions, weather, events.
- **Uncertainty estimation:** Prediction intervals or quantile regression for demand uncertainty.

### Layer 2: Optimization Engine

- **MILP model:** Mixed-integer linear program for inventory and replenishment.
- **Decision variables:** Order quantities, safety stock levels, reorder points (as defined in the model).
- **Constraints:** Capacity, lead times, service levels, min/max inventory.
- **Objective function:** Minimize cost (holding, shortage, ordering) or maximize service under constraints.

### Layer 3: Risk Intelligence Layer

- **RAG pipeline:** Retrieve relevant documents from vector store; generate answers with local LLM (Ollama).
- **Vector database:** ChromaDB or FAISS; embeddings via sentence-transformers.
- **Structured risk output:** All risk outputs must conform to the following JSON schema:

```json
{
  "event_type": "string",
  "severity": "string",
  "region": "string",
  "affected_entities": "string or array",
  "expected_delay_days": "number or null",
  "confidence": "number"
}
```

---

## 5. Data Flow (Daily Self-Learning Loop)

1. **Ingestion** — Pull core and external data (APIs, scraping) into the warehouse.
2. **Feature update** — Compute and store features for forecasting and optimization.
3. **Risk extraction** — Run RAG pipeline; persist structured risk JSON.
4. **Forecast generation** — Run forecasting models; write forecasts to storage.
5. **Optimization solve** — Run MILP with forecasts and constraints; output policy (e.g., order quantities, safety stock).
6. **Metrics logging** — Log forecasts, policies, and KPIs for monitoring.
7. **Model retraining (weekly)** — Retrain forecasting and (if applicable) risk/embedding components on a fixed schedule.

---

## 6. Folder Structure

```
data_ingestion/     # ETL and API/scraping jobs
warehouse/          # DB access, schemas, raw/processed storage logic
features/           # Feature engineering pipelines
models/
  forecasting/      # Time series and ML forecast models
  optimization/     # MILP model and solver integration
  risk_agent/       # RAG, embeddings, LLM, risk JSON schema
services/           # FastAPI and other runtime services
orchestration/      # Prefect (or Airflow) DAGs and flows
monitoring/         # Logging, metrics, drift checks
notebooks/          # Exploration and one-off analysis
```

---

## 7. Database Schema Overview

Main tables:

- **orders** — Order transactions (e.g., order_id, product_id, customer_id, quantity, date).
- **products** — Product master (e.g., product_id, category, lead_time).
- **customers** — Customer master (e.g., customer_id, region).
- **shipments** — Shipment and delivery records (e.g., shipment_id, order_id, status, dates).
- **external_weather_daily** — Daily weather by location (from Open-Meteo, Meteostat, NOAA).
- **external_events** — Ingested events (e.g., GDELT, news, Reddit) for risk and features.
- **forecast_daily** — Daily demand (or other) forecasts with optional uncertainty.
- **policy_daily** — Daily optimization output (e.g., order quantities, safety stock, reorder points).

---

## 8. Development Rules

- **Reproducibility:** All models must be reproducible (fixed seeds, versioned data/config).
- **Orchestration:** All pipelines must run via orchestration (Prefect or Airflow); no ad-hoc production steps.
- **Logging:** All pipeline outputs and key decisions must be logged (e.g., to DB or monitoring).
- **Structured risk:** All risk outputs must be valid JSON conforming to the defined risk schema.
- **No hardcoded credentials:** Use environment variables or a secrets manager; no secrets in code.
- **No proprietary tools:** Only open-source libraries and services; no paid or proprietary APIs.

---

## 9. Future Phases

| Phase | Focus |
|-------|--------|
| **Phase 1** | Data Foundation — Warehouse, core tables, DataCo ingestion |
| **Phase 2** | External Data Integration — GDELT, news, weather, trends, Reddit, etc. |
| **Phase 3** | Forecasting Models — Time series + ML, features, uncertainty |
| **Phase 4** | Optimization Engine — MILP, OR-Tools, constraints, objectives |
| **Phase 5** | Risk Agent (RAG) — Vector DB, embeddings, Ollama, structured risk JSON |
| **Phase 6** | Full Integration — Daily loop, services, end-to-end pipeline |
| **Phase 7** | Monitoring & Drift Detection — Metrics, alerts, model drift |

---

*This README is the authoritative guide for building the system step by step in future development sessions.*

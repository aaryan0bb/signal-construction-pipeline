# Factor Engine — Architecture Reference

> **Generated**: 2026-02-28 | **Version**: 0.1.0 | **Python**: ≥ 3.12

---

## 1. System Overview

Factor Engine is an AI-powered quantitative factor research platform that automatically generates tradable equity factors from natural-language investment themes. The system operates as a **three-stage pipeline** backed by a FastAPI service layer and a Next.js frontend.

```mermaid
graph LR
    subgraph User Layer
        FE[Next.js Frontend]
        API[FastAPI Backend :8000]
    end

    subgraph Pipeline Core
        S1[Stage 1<br/>Literature Agent]
        S2[Stage 2<br/>DAG Generator]
        S3[Stage 3<br/>Orchestrator]
    end

    subgraph Support Layer
        DA[Data Adapters<br/>13 sources]
        SA[Sub-Agents<br/>13 agents]
        CFG[Config &<br/>Registries]
    end

    subgraph Infra
        PG[(PostgreSQL)]
        RD[(Redis / Celery)]
    end

    FE -->|REST| API
    API -->|jobs| RD
    RD -->|workers| S1
    S1 -->|framework JSON| S2
    S2 -->|DAG JSON| S3
    S3 -->|execute| SA
    SA -->|fetch| DA
    S3 -->|read| CFG
    API -->|persist| PG
```

---

## 2. Repository Layout

```
factor_engine/
├── backend/                    # FastAPI + Celery application
│   ├── app/
│   │   ├── api/v1/             # REST endpoints (factors, universes, agents, dags, health)
│   │   ├── db/                 # SQLAlchemy models & repository
│   │   ├── models/             # Pydantic request/response schemas
│   │   ├── jobs/               # Celery background jobs
│   │   └── services/           # Business logic
│   └── pyproject.toml
│
├── packages/
│   ├── literature_agent/       # Stage 1 — theme → mathematical framework
│   ├── dag_generator/          # Stage 2 — framework → executable DAG
│   ├── orchestrator/           # Stage 3 — DAG → executed factor
│   ├── sub_agents/             # 13 specialized execution agents
│   └── data_adapters/          # 13+ external data source adapters
│
├── config/
│   ├── factor_agent.yaml       # Master execution config
│   ├── observability.yaml      # Logging & metrics
│   ├── rate_limits.yaml        # API rate limits
│   └── registries/             # Agent & adapter YAML registries
│
├── frontend/                   # Next.js web application
├── data/                       # Universes, cache, outputs
├── docker/                     # Docker Compose
└── pyproject.toml              # uv workspace root
```

---

## 3. Three-Stage Pipeline

The core value proposition: turn an **investment theme** (plain English) into a **scored, tradable equity factor** — fully automated.

```mermaid
flowchart TB
    THEME["🎯 Investment Theme<br/><i>'Crowded short positions unwinding in software'</i>"]

    subgraph STAGE1["Stage 1 — Literature Agent"]
        direction TB
        D1[Discovery<br/>GPT-5 web search]
        D2[Schema Generation<br/>LLM schema planner]
        D3[Extraction<br/>Agent SDK + Tavily]
        D1 --> D2 --> D3
    end

    FW["📄 Mathematical Framework JSON<br/>scoring dims, formulas,<br/>data requirements, portfolio rules"]

    subgraph STAGE2["Stage 2 — DAG Generator"]
        direction TB
        P1["Phase 1 — Pseudocode<br/>8-block draft (Claude)"]
        P2C["Phase 2C — Decompose<br/>sub-op level DAG (GPT)"]
        P2D["Phase 2D — Instructions<br/>agent playbooks (GPT)"]
        VIZ[Visualization<br/>interactive HTML]
        P1 --> P2C --> P2D --> VIZ
    end

    DAG["📊 Executable DAG JSON<br/>nodes, edges, agent assignments,<br/>adapter specs, instructions"]

    subgraph STAGE3["Stage 3 — Orchestrator"]
        direction TB
        LOAD[Load DAG + Config]
        SCHED[Scheduler<br/>topo-sort + concurrency]
        EXEC[SubAgentExecutor<br/>dispatch + timeout]
        FACIL[FacilitationRouter<br/>schema compat]
        STATE[StateManager<br/>checkpoint + resume]
        LOAD --> SCHED --> EXEC
        EXEC --> FACIL
        EXEC --> STATE
    end

    OUTPUT["📈 Factor Output<br/>loadings, returns, audit trail,<br/>HTML report"]

    THEME --> STAGE1
    STAGE1 --> FW
    FW --> STAGE2
    STAGE2 --> DAG
    DAG --> STAGE3
    STAGE3 --> OUTPUT

    style THEME fill:#ffeaa7,stroke:#fdcb6e,color:#000
    style FW fill:#dfe6e9,stroke:#b2bec3,color:#000
    style DAG fill:#dfe6e9,stroke:#b2bec3,color:#000
    style OUTPUT fill:#55efc4,stroke:#00b894,color:#000
```

---

## 4. Package Deep-Dives

### 4.1 Literature Agent (`packages/literature_agent/`)

Transforms an investment theme into a structured mathematical framework via three sub-stages.

```mermaid
flowchart LR
    subgraph Discovery
        TS[Theme Summary]
        FP[Factor Planner]
        QP[Query Pack Designer]
        WS[GPT-5 Web Search]
        HY[Hydration<br/>Tavily / PDF]
        SC[LLM Scoring]
        CA[Coverage Auditor]
        TS --> FP --> QP --> WS --> HY --> SC --> CA
    end

    subgraph SchemaGen["Schema Generation"]
        SP[Schema Planner]
        FR[Freeze Schema]
        SP --> FR
    end

    subgraph Extraction
        FE2[Tavily Fetch]
        CL[Classifier<br/>section mapping]
        AG[Agent SDK<br/>GPT-5 + web search]
        FE2 --> CL --> AG
    end

    Discovery --> SchemaGen --> Extraction

    OUT["Mathematical Framework JSON"]
    Extraction --> OUT
```

**Key capabilities**:
- **Discovery**: GPT-5 web search across 4 segments (INDEX_METHOD, ACADEMIC, STANDARDS, ECOSYSTEM) with coverage auditing and backfill
- **Schema Generation**: LLM-planned extraction schema with thinking modes (fast / balanced / deep)
- **Extraction**: Agent SDK autonomous research with confidence guardrails and implementability filtering
- **Resume support**: Each stage checks for prior outputs before re-running

---

### 4.2 DAG Generator (`packages/dag_generator/`)

Converts a mathematical framework into an executable DAG through a multi-phase LLM pipeline.

```mermaid
flowchart TB
    FW_IN["Mathematical Framework JSON"]

    subgraph Phase1["Phase 1 — draft_pseudocode_9block.py"]
        GEN["Claude API<br/>tool-based structured output"]
        BLOCKS["8 Canonical Blocks"]
    end

    subgraph Phase2C["Phase 2C — phase2c_clean.py"]
        TOPO[Topological Sort]
        C1["Call 1: Agent Selection<br/>+ internal deps"]
        C2["Call 2: Artifact Resolution<br/>+ adapter specs"]
        VAL["Adapter Validation Pipeline<br/>validate → repair → revalidate"]
        REG["Artifact Registry<br/>track outputs for downstream"]
        TOPO --> C1 --> C2 --> VAL --> REG
    end

    subgraph Phase2D["Phase 2D — phase2d_subop_processor.py"]
        INST["Generate Subagent<br/>Instructions"]
        LINT["Lint & Repair<br/>required sections check"]
        INST --> LINT
    end

    subgraph Viz["Visualization — build_dag_viz.py"]
        MER[Mermaid Diagram]
        HTML[Interactive HTML]
        CSV[Edge Analysis CSV]
    end

    FW_IN --> Phase1
    Phase1 --> Phase2C
    Phase2C --> Phase2D
    Phase2D --> Viz

    style FW_IN fill:#dfe6e9,stroke:#b2bec3,color:#000
```

**8 canonical blocks** (execution order):

| # | Block | Purpose |
|---|-------|---------|
| 1 | `universe_tradability` | Universe selection + liquidity/market-cap filters |
| 2 | `extraction` | Raw data pull (patents, SEC, news, fundamentals) |
| 3 | `classification` | Document/entity classification by theme relevance |
| 4 | `scoring_llm` | LLM-based scoring (sentiment, relevance) |
| 5 | `scoring_formula` | Formula-based scoring (z-scores, dimensions) |
| 6 | `aggregation` | Combine dimension scores into composite |
| 7 | `weights` | Portfolio weights with constraints |
| 8 | `output` | Final outputs + audit trail |

**Adapter validation pipeline**:
```mermaid
flowchart LR
    SPEC["AdapterCallSpec"] --> V1["Validate<br/>against registry"]
    V1 -->|errors?| REP["Repair<br/>canonicalize + bind params"]
    REP --> V2["Re-validate"]
    V2 -->|pass| OK["✓ Canonical spec"]
    V1 -->|clean| OK
    V2 -->|fail| ERR["✗ Unrepairable"]

    style OK fill:#55efc4,stroke:#00b894,color:#000
    style ERR fill:#fab1a0,stroke:#e17055,color:#000
```

---

### 4.3 Orchestrator (`packages/orchestrator/`)

The execution engine: loads a DAG, schedules operations respecting dependencies and rate limits, dispatches to sub-agents, and manages state.

```mermaid
flowchart TB
    subgraph Config
        CL[ConfigLoader<br/>YAML + env overrides]
        AR[AgentRegistry]
        DR[AdapterRegistry]
    end

    subgraph Core["FactorAgent — Main Loop"]
        DAG_E[DAGEngine<br/>parse, validate, topo-sort]
        SCHED[Scheduler<br/>ready ops + priority]
        IP[InstructionPreparer<br/>LLM synthesis + context]
        SAE[SubAgentExecutor<br/>dispatch + timeout + salvage]
        FR[FacilitationRouter<br/>schema compat transforms]
    end

    subgraph Execution
        SM[StateManager<br/>checkpoints + resume]
        RL[RateLimitManager<br/>token bucket + backoff]
        TT[TokenTracker<br/>budget enforcement]
    end

    subgraph Data
        AS[ArtifactStore<br/>path registry]
        MS[ManifestStore<br/>agent output manifests]
        MI[ManifestInspector<br/>40+ path candidates]
        DP[DataPreviewer<br/>upstream previews]
        IAR[InputArtifactResolver<br/>upstream → input mapping]
    end

    subgraph Observability
        LOG[StructuredLogger<br/>JSON/text + rotation]
        MET[MetricsCollector<br/>per-op + per-service]
        TRC[TraceManager<br/>prompt/response logging]
        PRG[ProgressTracker<br/>live progress bar]
    end

    subgraph Reliability
        EC[ErrorClassifier]
        TS[Troubleshooter]
        RS[RetryStrategy]
        FP[FailurePropagation]
        RO[RecoveryOrchestrator]
    end

    Config --> Core
    Core --> Execution
    Core --> Data
    Core --> Observability
    Core --> Reliability
```

**Execution flow for a single operation**:

```mermaid
sequenceDiagram
    participant S as Scheduler
    participant FA as FactorAgent
    participant IP as InstructionPreparer
    participant SAE as SubAgentExecutor
    participant Agent as SubAgent
    participant AS as ArtifactStore

    S->>FA: ready operations (deps satisfied, rate OK)
    FA->>IP: prepare(node, context)
    IP->>IP: compose prompt + fit to token budget
    IP-->>FA: PreparedInstruction
    FA->>SAE: execute(operation, instruction)
    SAE->>SAE: instantiate agent (pooled)
    SAE->>Agent: execute(context)
    Agent->>Agent: validate → process → output
    Agent-->>SAE: ExecutionResult + manifest
    SAE->>SAE: augment manifest + categorize errors
    SAE-->>FA: ExecutionResult
    FA->>AS: register_artifacts(op_id, paths)
    FA->>S: mark completed, get next
```

---

### 4.4 Sub-Agents (`packages/sub_agents/`)

13 specialized agents organized in a class hierarchy.

```mermaid
classDiagram
    class BaseSubAgent {
        <<abstract>>
        +agent_name: str
        +execute(ctx) ExecutionResult
        +validate_inputs(ctx)
        +validate_output(result)
    }

    class LLMBasedAgent {
        <<abstract>>
        +_build_prompt()
        +_call_llm()
        +_parse_llm_response()
        +_post_process()
    }

    BaseSubAgent <|-- LLMBasedAgent
    BaseSubAgent <|-- CodeGeneratingAgent
    BaseSubAgent <|-- GenericExtractionAgent
    BaseSubAgent <|-- GenericScoringAgent
    BaseSubAgent <|-- GenericAggregationAgent
    BaseSubAgent <|-- OutputAgent
    BaseSubAgent <|-- DataReshapeAgent
    BaseSubAgent <|-- EventStudyAgent

    LLMBasedAgent <|-- ClassificationAgent
    LLMBasedAgent <|-- LLMScoringAgent
    LLMBasedAgent <|-- GenericLLMExtractionAgent
    LLMBasedAgent <|-- StructuredDataExtractionAgent
    LLMBasedAgent <|-- WebExtractionAgent
    LLMBasedAgent <|-- UniverseSelectionAgent
```

**Agent capabilities matrix**:

| Agent | Category | LLM? | Key Capability |
|-------|----------|------|----------------|
| `CodeGeneratingAgent` | transformation | yes | LangGraph reflection loop, sandbox exec, gpt-5.2 |
| `GenericExtractionAgent` | extraction | partial | Adapter-driven data pull, semantic column matching |
| `GenericScoringAgent` | scoring | yes | Claude CGA delegation, stage-based execution |
| `GenericAggregationAgent` | aggregation | yes | Claude CGA, component discovery, null propagation |
| `OutputAgent` | output | optional | Multi-format (CSV/Parquet/JSON/HTML), audit trails |
| `DataReshapeAgent` | transformation | yes | LangGraph, schema analysis, sandbox exec |
| `EventStudyAgent` | analysis | optional | Abnormal returns, CARs, SCARs, event betas |
| `ClassificationAgent` | classification | yes | Dynamic label schema inference, structured output |
| `LLMScoringAgent` | scoring | yes | Dynamic scoring schema, batch processing, token budgets |
| `GenericLLMExtractionAgent` | extraction | yes | Schema inference + structured extraction |
| `StructuredDataExtractionAgent` | extraction | yes | Three-phase: infer → synthesize → extract |
| `WebExtractionAgent` | extraction | yes | Hybrid: Tavily discovery + structured extraction |
| `UniverseSelectionAgent` | selection | yes | Multi-stage filtering, web search thematic screening |

**Common execution patterns**:

```mermaid
flowchart LR
    subgraph "Reflection Loop (CGA, Reshape, EventStudy)"
        A1[Generate Code] --> A2[Validate]
        A2 -->|fail| A3[Reflect & Fix]
        A3 --> A1
        A2 -->|pass| A4[Execute in Sandbox]
    end

    subgraph "Three-Phase LLM (Classification, Scoring, Extraction)"
        B1[Infer Schema] --> B2[Synthesize Pydantic Model]
        B2 --> B3[Structured LLM Output]
    end

    subgraph "Claude CGA (Scoring, Aggregation)"
        C1["Phase 1: Plan<br/>(Opus/Sonnet)"] --> C2["Phase 2: Execute<br/>(Agent SDK tool-use)"]
    end
```

---

### 4.5 Data Adapters (`packages/data_adapters/`)

Unified interface to 13+ external data sources. All inherit from `BaseAdapter` with shared caching, rate limiting, and Point-in-Time (PTI) filtering.

```mermaid
flowchart TB
    subgraph BaseAdapter["BaseAdapter (abstract)"]
        CACHE[Parquet Cache<br/>SHA256 keys]
        RETRY[Retry + Backoff<br/>429, 5xx]
        PTI[ensure_pti<br/>data ≤ as_of_date]
        VALID[validate_response<br/>schema check]
    end

    subgraph Market["Market Data"]
        POLY[PolygonAdapter<br/>OHLCV, splits, dividends,<br/>short interest]
        FMP[FMPAdapter<br/>market cap, income stmt,<br/>key metrics, transcripts]
        NEWS[NewsAdapter<br/>stock news articles]
    end

    subgraph Fundamental["Company & Filings"]
        CO[CompanyOverviewAdapter<br/>FMP profile + Tavily relevance]
        SEC[SECAdapter<br/>EDGAR filings + text extraction]
    end

    subgraph AltData["Alternative Data"]
        PAT[PatentsAdapter<br/>PatentsView, CPC codes]
        GR[GrantsAdapter<br/>NSF Awards API]
        CON[ConsortiaAdapter<br/>QED-C membership scraping]
        OSS[OSSAdapter<br/>GitHub repo metrics]
        LIT[LiteratureAdapter<br/>arXiv + OpenAlex]
    end

    subgraph Reference["Reference & IDs"]
        FMPID[FMPIdentifierAdapter<br/>CIK, ISIN, CUSIP]
        FIGI[OpenFIGIAdapter<br/>FIGI identifiers]
        GLEIF[GLEIFAdapter<br/>LEI identifiers]
        MAPPER["IdentifierMapper<br/>(orchestrates all 3)"]
        INV[InventoryAdapter<br/>local CSV loading]
    end

    BaseAdapter --> Market
    BaseAdapter --> Fundamental
    BaseAdapter --> AltData
    BaseAdapter --> Reference
    FMPID --> MAPPER
    FIGI --> MAPPER
    GLEIF --> MAPPER
```

**External API dependencies**:

| Adapter | API | Auth |
|---------|-----|------|
| PolygonAdapter | api.polygon.io | API key |
| FMPAdapter | financialmodelingprep.com | API key |
| NewsAdapter | FMP /stable/news/stock | API key |
| CompanyOverviewAdapter | FMP + Tavily | API keys |
| SECAdapter | EDGAR + FMP + Tavily | API keys |
| PatentsAdapter | search.patentsview.org | API key |
| GrantsAdapter | research.gov (NSF) | none |
| ConsortiaAdapter | quantumconsortium.org | none (+ OpenAI for symbols) |
| OSSAdapter | api.github.com | token (optional) |
| LiteratureAdapter | arXiv + OpenAlex | none |
| OpenFIGIAdapter | api.openfigi.com | optional |
| GLEIFAdapter | api.gleif.org | none |
| InventoryAdapter | local filesystem | none |

---

## 5. Configuration System

```mermaid
flowchart TB
    subgraph YAMLFiles["YAML Config Files"]
        FA_CFG["factor_agent.yaml<br/>execution, tokens, retry,<br/>context, artifacts, LLM,<br/>facilitation, reliability"]
        OBS_CFG["observability.yaml<br/>logging, progress, metrics"]
        RL_CFG["rate_limits.yaml<br/>per-service RPM/TPM"]
    end

    subgraph Registries
        SA_REG["sub_agents_registry.yaml<br/>13 agents × capabilities"]
        DA_REG["data_adapter_registry.yaml<br/>methods, params, output cols"]
        IND["individual/*.yaml<br/>13 per-agent config files"]
    end

    subgraph Runtime
        ENV[".env / environment vars"]
        CLI["CLI args / runtime overrides"]
    end

    FA_CFG --> CL[ConfigLoader]
    OBS_CFG --> CL
    RL_CFG --> CL
    SA_REG --> AR2[AgentRegistry]
    DA_REG --> DR2[AdapterRegistry]
    IND --> AR2
    ENV --> CL
    CLI --> CL

    CL --> CFG["Config object<br/>nested key access"]
```

**Key config sections** (`factor_agent.yaml`):

| Section | Controls |
|---------|----------|
| `execution` | max_concurrent (8), checkpoint freq, timeout defaults |
| `token_budget` | per-op warning (10K), limit (50K), session (1M) |
| `retry` | max attempts by ErrorCategory, exponential backoff |
| `artifacts` | storage paths, compression (gzip:6), max size (500MB), retention (30d) |
| `llm` | default provider (openai), model (gpt-5.2), max tokens (100K) |
| `facilitation` | transform detection (rename, pivot, aggregate, unpivot) |
| `reliability` | error classification, troubleshooting, recovery agents |
| `extraction` | base period (1yr), quarterly rebalance, adapter lookback windows |

---

## 6. Observability Stack

```mermaid
flowchart LR
    subgraph Inputs
        OP[Operation Events]
        LLM2[LLM Calls]
        ERR[Errors]
    end

    subgraph Collectors
        LOG2[StructuredLogger<br/>JSON + console<br/>file rotation 10MB]
        MET2[MetricsCollector<br/>per-op duration, tokens,<br/>retries, API calls]
        TRC2[TraceManager<br/>per-op directories<br/>prompt + response logs]
        PRG2[ProgressTracker<br/>live bar + ETA]
    end

    subgraph Outputs
        CON2[Console<br/>color-coded]
        FILE[Log Files<br/>5 backup rotation]
        JSON2[metrics.json<br/>+ metrics.csv]
        TRACE[traces/<br/>context.json<br/>llm/*.json<br/>steps.json]
    end

    OP --> LOG2 --> CON2
    OP --> LOG2 --> FILE
    OP --> MET2 --> JSON2
    LLM2 --> TRC2 --> TRACE
    OP --> PRG2
    ERR --> LOG2
```

---

## 7. Reliability & Error Handling

```mermaid
flowchart TB
    FAIL["Operation Failure"]
    FAIL --> EC2["ErrorClassifier<br/>(LLM-driven)"]
    EC2 --> CAT{Category?}

    CAT -->|TRANSIENT| R1["Retry ×3<br/>exponential backoff"]
    CAT -->|DATA| R2["Retry ×2<br/>+ data repair"]
    CAT -->|LOGIC| R3["Retry ×3<br/>+ troubleshoot"]
    CAT -->|TIMEOUT| R4["Retry ×2<br/>+ salvage partial"]
    CAT -->|FATAL| R5["Skip + propagate<br/>downstream impact"]

    R1 --> OK["✓ Recovered"]
    R2 --> OK
    R3 --> OK
    R4 --> OK
    R5 --> PROP["FailurePropagation<br/>mark downstream SKIPPED"]

    style OK fill:#55efc4,stroke:#00b894,color:#000
    style PROP fill:#fab1a0,stroke:#e17055,color:#000
```

---

## 8. Data Flow — End to End

```mermaid
flowchart TB
    USER["User: 'Build a factor capturing<br/>crowded short positions unwinding'"]

    subgraph S1["Stage 1: Literature Agent"]
        DISC["Discover 50+ sources<br/>across 4 segments"]
        SCHEMA["Plan extraction schema<br/>(5-10 sections)"]
        EXTRACT["Extract mathematical<br/>framework via Agent SDK"]
    end

    FW2["Framework JSON<br/>• scoring dimensions & formulas<br/>• data requirements<br/>• portfolio construction rules<br/>• tradability filters"]

    subgraph S2["Stage 2: DAG Generator"]
        PSEUDO["Generate 8-block<br/>pseudocode (Claude)"]
        DECOMP["Decompose into ~30-50<br/>sub-operations (GPT)"]
        PLAY["Generate agent<br/>playbooks (GPT)"]
    end

    DAG2["DAG JSON<br/>• 30-50 nodes<br/>• dependency edges<br/>• agent assignments<br/>• adapter call specs<br/>• subagent instructions"]

    subgraph S3["Stage 3: Orchestrator"]
        TOPO2["Topological schedule"]
        direction TB
        EX1["universe_tradability<br/>→ UniverseSelectionAgent"]
        EX2["extraction (×N)<br/>→ GenericExtractionAgent"]
        EX3["classification<br/>→ ClassificationAgent"]
        EX4["scoring_llm<br/>→ LLMScoringAgent"]
        EX5["scoring_formula<br/>→ CodeGeneratingAgent"]
        EX6["aggregation<br/>→ GenericAggregationAgent"]
        EX7["weights<br/>→ CodeGeneratingAgent"]
        EX8["output<br/>→ OutputAgent"]
        TOPO2 --> EX1 --> EX2 --> EX3 --> EX4 --> EX5 --> EX6 --> EX7 --> EX8
    end

    RESULT["Factor Output<br/>• factor_loadings.parquet<br/>• factor_returns.parquet<br/>• audit_trail.json<br/>• report.html"]

    USER --> S1
    S1 --> FW2
    FW2 --> S2
    S2 --> DAG2
    DAG2 --> S3
    S3 --> RESULT

    style USER fill:#ffeaa7,stroke:#fdcb6e,color:#000
    style FW2 fill:#dfe6e9,stroke:#b2bec3,color:#000
    style DAG2 fill:#dfe6e9,stroke:#b2bec3,color:#000
    style RESULT fill:#55efc4,stroke:#00b894,color:#000
```

---

## 9. LLM Usage Map

The system uses multiple LLM providers and models across stages:

```mermaid
flowchart TB
    subgraph Anthropic
        CLAUDE["Claude (Anthropic API)"]
        OPUS["Opus 4.6<br/>CGA planning + execution"]
        SONNET["Sonnet 4.5<br/>CGA planning (scoring)"]
    end

    subgraph OpenAI
        GPT5["GPT-5<br/>web search (discovery)"]
        GPT52["GPT-5.2<br/>CodeGeneratingAgent"]
        GPT5M["GPT-5-mini<br/>most agents, schema,<br/>classification, scoring"]
    end

    subgraph Usage
        S1U["Stage 1: Discovery → GPT-5 (web search)<br/>Schema → GPT-5-mini<br/>Extraction → GPT-5 Agent SDK"]
        S2U["Stage 2: Phase 1 → Claude<br/>Phase 2C/2D → GPT-5-mini"]
        S3U["Stage 3: Instructions → gpt-5.2<br/>Sub-agents → gpt-5-mini / Opus"]
    end

    Anthropic --> S2U
    Anthropic --> S3U
    OpenAI --> S1U
    OpenAI --> S2U
    OpenAI --> S3U
```

---

## 10. Rate Limiting Architecture

```mermaid
flowchart LR
    REQ[API Request] --> TKB["TokenBucket<br/>RPM / TPM"]
    TKB -->|tokens available| SLOT["Concurrent Slot<br/>semaphore"]
    SLOT -->|slot free| API_CALL["Execute API Call"]
    API_CALL -->|429| BACK["Exponential Backoff<br/>+ jitter"]
    BACK --> TKB
    API_CALL -->|success| REL["Release slot<br/>record usage"]

    subgraph Limits["Configured Limits"]
        L1["OpenAI: 180M TPM, 30K RPM"]
        L2["Anthropic: 100K TPM, 1K RPM"]
        L3["Polygon: 100 RPS, 50 concurrent"]
        L4["FMP: 300 RPM, 10 concurrent"]
        L5["SEC EDGAR: 10 RPS, 5 concurrent"]
        L6["Tavily: 100 RPM, 3K daily"]
    end
```

---

## 11. Backend API Layer

```mermaid
flowchart TB
    subgraph Endpoints["FastAPI v1 Endpoints"]
        E1["/factors — CRUD factor definitions"]
        E2["/universes — equity universe management"]
        E3["/agents — agent status & config"]
        E4["/dags — DAG management & visualization"]
        E5["/health — system health check"]
    end

    subgraph Services
        FS[FactorService]
        US[UniverseService]
    end

    subgraph Jobs["Celery Workers"]
        J1[Pipeline Execution Job]
        J2[Data Refresh Job]
    end

    E1 --> FS
    E2 --> US
    FS --> J1
    J1 -->|Redis| WORKER["Celery Worker"]
    WORKER --> S1_2["Stage 1 → 2 → 3 Pipeline"]
```

---

## 12. Workspace & Build

The project uses a **uv workspace** monorepo:

```mermaid
flowchart TB
    ROOT["factor-engine (root)<br/>pyproject.toml"]

    ROOT --> P1["factor-engine-literature-agent<br/>openai, agents SDK, tavily, tiktoken"]
    ROOT --> P2["factor-engine-dag-generator<br/>openai, anthropic, pydantic, pyyaml"]
    ROOT --> P3["factor-engine-orchestrator<br/>openai, pydantic, pyyaml, pandas<br/>depends: sub_agents, data_adapters"]
    ROOT --> P4["factor-engine-sub-agents<br/>openai, pydantic, pandas, langgraph<br/>depends: data_adapters"]
    ROOT --> P5["factor-engine-data-adapters<br/>pandas, requests, pyarrow"]
    ROOT --> P6["factor-engine-backend<br/>fastapi, celery, redis<br/>depends: ALL packages"]

    P3 --> P4
    P3 --> P5
    P4 --> P5
    P6 --> P1
    P6 --> P2
    P6 --> P3
```

**Key commands** (`Makefile`):

| Command | Action |
|---------|--------|
| `make dev` | Install all packages |
| `make backend` | Start API (uvicorn :8000) |
| `make frontend` | Start Next.js dev |
| `make worker` | Start Celery worker |
| `make test` | Run all tests |
| `make lint` | Ruff checks + format |
| `make up / down` | Docker Compose |

---

## 13. Key Design Patterns

| Pattern | Where Used | Purpose |
|---------|------------|---------|
| **Reflection Loop** | CGA, Reshape, EventStudy | Generate → validate → fix → re-generate (max 3) |
| **Three-Phase LLM** | Classification, Scoring, Extraction | Infer schema → synthesize model → structured output |
| **Two-Call Architecture** | Phase 2C | Separate agent selection from artifact resolution |
| **Validation Pipeline** | DAG Generator | Validate → repair → revalidate adapter specs |
| **Artifact Registry** | Phase 2C, Orchestrator | Track outputs to resolve downstream inputs |
| **Token Bucket** | RateLimitManager | Per-service rate limiting with backoff |
| **Context-Result** | All sub-agents | Immutable context in, result out |
| **Manifest-Based I/O** | Orchestrator ↔ Agents | Agents emit flexible JSON manifests |
| **Checkpoint/Resume** | Orchestrator, Literature Agent | Persist state, skip completed stages |
| **Topological Scheduling** | DAG Engine, Phase 2C/2D | Kahn's algorithm for dependency-respecting order |
| **Singleton Registry** | AdapterContractLoader | Lazy-load YAML once, indexed access |

---

## 14. File Inventory by Package

| Package | Files | Lines (est.) | Role |
|---------|-------|-------------|------|
| `literature_agent` | ~25 | ~5K | Stage 1: theme → framework |
| `dag_generator` | 12 | ~4K | Stage 2: framework → DAG |
| `orchestrator` | ~30 | ~8K | Stage 3: DAG → execution |
| `sub_agents` | 39 | ~15K | 13 specialized agents |
| `data_adapters` | 17 | ~5K | 13+ data source adapters |
| `backend` | ~15 | ~2K | FastAPI + Celery |
| **Total** | **~138** | **~39K** | |

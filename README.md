# 🤖 LangGraph Multi-Agent System

<div align="center">

[![Python](https://img.shields.io/badge/Python-3.12+-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://www.python.org/)
[![LangGraph](https://img.shields.io/badge/LangGraph-0.2+-FF6B6B?style=for-the-badge&logo=chainlink&logoColor=white)](https://github.com/langchain-ai/langgraph)
[![Redis](https://img.shields.io/badge/Redis-7.0+-DC382D?style=for-the-badge&logo=redis&logoColor=white)](https://redis.io/)
[![Qdrant](https://img.shields.io/badge/Qdrant-Vector_DB-24CAFF?style=for-the-badge)](https://qdrant.tech/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=for-the-badge)](LICENSE)

**A production-grade multi-agent orchestration framework** built with LangGraph and LangChain, featuring AI-enhanced web search, persistent memory, and distributed state management.

[Features](#-features) • [Architecture](#-architecture) • [Quick Start](#-quick-start) • [Setup Guide](#-setup-guide) • [Usage](#-usage) • [Project Structure](#-project-structure)

</div>

---

## 🎯 What This Project Does

Modern AI tasks — research, analysis, summarisation — are too complex for a single model call. This project implements a **supervisor-worker multi-agent pattern** where:

- A **Planning Agent** decomposes a complex request into sub-tasks
- A **Perplexica Search Agent** executes AI-enhanced web research for each sub-task
- An **Execution Agent** consolidates findings and produces the final output
- A central **Orchestrator** coordinates agent routing via a LangGraph `StateGraph`
- A **Memory Bridge** (Mem0 + Qdrant + Neo4j) gives agents cross-session memory and knowledge-graph awareness

The result is a modular, async-first system that can be extended with new specialist agents simply by subclassing `BaseAgent`.

---

## ✨ Features

| Feature | Details |
|---|---|
| 🧠 **Multi-Agent Orchestration** | LangGraph `StateGraph` routes tasks between specialist agents automatically |
| 🔍 **AI-Enhanced Web Search** | Perplexica integration for semantic, LLM-filtered research results |
| 💾 **Persistent Memory** | Mem0 + Qdrant (vector) + Neo4j (graph) for cross-session context |
| ⚡ **Distributed State** | Redis-backed state sharing across agents with 1-hour TTL |
| 🔄 **Async Architecture** | Full `async/await` with `aiohttp` and `aioredis` for high throughput |
| 🛡️ **Input Sanitisation** | HTML-escaping and regex-based validation on all user inputs |
| 🔧 **12-Factor Config** | All secrets/URLs via environment variables — zero hard-coded credentials |
| 📦 **Pluggable Agents** | Abstract `BaseAgent` class — add new agents in ~50 lines |

---

## 🏗️ Architecture

```
User Request
     │
     ▼
┌─────────────────────────────────────────────────┐
│              MultiAgentOrchestrator              │
│          (LangGraph StateGraph + Redis)          │
│                                                 │
│  ┌───────────┐  ┌──────────────┐  ┌──────────┐ │
│  │ Planning  │→ │  Perplexica  │→ │Execution │ │
│  │  Agent    │  │ Search Agent │  │  Agent   │ │
│  └───────────┘  └──────────────┘  └──────────┘ │
│                        │                        │
│                  ┌─────▼──────┐                 │
│                  │Memory Bridge│                 │
│                  │Mem0+Qdrant │                 │
│                  │  + Neo4j   │                 │
│                  └────────────┘                 │
└─────────────────────────────────────────────────┘
```

### Core Components

| Component | File | Responsibility |
|---|---|---|
| `MultiAgentOrchestrator` | `src/orchestrator.py` | Graph construction, agent routing, Redis state |
| `BaseAgent` | `src/orchestrator.py` | Abstract base with async init/cleanup, Redis I/O |
| `PlanningAgent` | `src/agents/planning_agent.py` | Decomposes requests into ordered sub-tasks |
| `PerplexicaSearchAgent` | `src/agents/perplexica_search_agent.py` | AI-enhanced web search via Perplexica API |
| `ExecutionAgent` | `src/agents/execution_agent.py` | Consolidates results, produces final output |
| `MemoryBridge` | `src/memory_bridge.py` | Connects to Mem0/Qdrant/Neo4j for persistent memory |
| `Config` | `src/config.py` | Environment-variable-based configuration |
| `LoggingConfig` | `src/logging_config.py` | Structured logging setup |

### Request Lifecycle

```
1. User sends request  →  Orchestrator sanitises & validates input
2. Orchestrator        →  PlanningAgent decomposes into sub-tasks
3. Sub-tasks           →  PerplexicaSearchAgent performs web research
4. Search results      →  ExecutionAgent synthesises final answer
5. Final answer        →  MemoryBridge stores context for next session
6. Response            →  Returned to user with Redis state cleanup
```

---

## 🚀 Quick Start

### Prerequisites

- **Python 3.12+**
- **Redis** (state management)
- **Qdrant** (vector store for memory)
- **Neo4j** (graph database for memory)
- **Node.js 18+** (for Perplexica AI search)
- A running **llama.cpp** server or compatible OpenAI-compatible API endpoint

> ⚠️ See [SETUP.md](SETUP.md) for full instructions on configuring Perplexica, SearXNG, Qdrant, and Neo4j.

### Installation

```bash
# 1. Clone the repository
git clone https://github.com/Yuvraaaaj/langgraph-multi-agent-system.git
cd langgraph-multi-agent-system

# 2. Create and activate a virtual environment
python -m venv venv
source venv/bin/activate          # Linux/macOS
# venv\Scripts\activate           # Windows

# 3. Install Python dependencies
pip install -r requirements.txt

# 4. Configure environment variables
cp .env.example .env
# Edit .env with your actual values
```

### Start Required Services

```bash
# Start Redis (Docker)
docker run -d -p 6379:6379 redis:7-alpine

# Start Qdrant (Docker)
docker run -d -p 6333:6333 qdrant/qdrant

# Start Neo4j (Docker)
docker run -d -p 7474:7474 -p 7687:7687 \
  -e NEO4J_AUTH=neo4j/your_password \
  neo4j:latest
```

Or use the provided helper script:

```bash
chmod +x start_services.sh
./start_services.sh
```

---

## ⚙️ Environment Variable Setup

Copy `.env.example` to `.env` and fill in your values:

```bash
cp .env.example .env
```

| Variable | Description | Default |
|---|---|---|
| `REDIS_URL` | Redis connection URL | `redis://localhost:6379` |
| `REDIS_PASSWORD` | Redis authentication password | *(none)* |
| `REDIS_DB` | Redis database index | `0` |
| `MODEL_PATH` | Path to your GGUF model file | *(required)* |
| `GPU_LAYERS` | Number of GPU layers (`-1` = all) | `-1` |
| `MODEL_CONTEXT_SIZE` | LLM context window size | `2048` |
| `MODEL_TEMPERATURE` | Sampling temperature | `0.7` |
| `PERPLEXICA_URL` | Perplexica API endpoint | `http://localhost:3000` |
| `SEARXNG_URL` | SearXNG search engine URL | `http://localhost:8888` |
| `MEM0_API_KEY` | Mem0 API key (if using cloud) | *(optional)* |
| `NEO4J_URI` | Neo4j Bolt URI | `bolt://localhost:7687` |
| `NEO4J_USER` | Neo4j username | `neo4j` |
| `NEO4J_PASSWORD` | Neo4j password | *(required)* |
| `QDRANT_URL` | Qdrant service URL | `http://localhost:6333` |
| `LOG_LEVEL` | Logging verbosity | `INFO` |
| `WORKSPACE_DIR` | Working directory for agent outputs | `./workspace` |

---

## 📁 Project Structure

```
langgraph-multi-agent-system/
├── src/
│   ├── __init__.py               # Package entry-point
│   ├── __main__.py               # CLI entry-point (python -m src)
│   ├── orchestrator.py           # Core: MultiAgentOrchestrator + BaseAgent
│   ├── config.py                 # Env-variable configuration class
│   ├── memory_bridge.py          # Mem0 + Qdrant + Neo4j integration
│   ├── logging_config.py         # Structured logging setup
│   └── agents/
│       ├── __init__.py
│       ├── planning_agent.py     # Task decomposition agent
│       ├── perplexica_search_agent.py  # AI-enhanced web search agent
│       └── execution_agent.py    # Result consolidation agent
├── config/
│   ├── perplexica/               # Perplexica config templates
│   └── searxng/                  # SearXNG config templates
├── examples/
│   └── ...                       # Usage examples
├── tests/
│   └── ...                       # Unit & integration tests
├── demo_research.py              # Standalone demo script
├── start_services.sh             # Helper: start Docker services
├── searxng_config.yml            # SearXNG configuration
├── requirements.txt              # Python dependencies
├── .env.example                  # Environment variable template
├── .gitignore
├── SETUP.md                      # Detailed external services setup guide
└── README.md
```

---

## 💻 Usage

### CLI — Research Mode

```bash
# Run a research query
python -m src research "AI safety regulations in 2024"

# Run demo with memory integration
python demo_research.py
```

### Programmatic Usage

```python
import asyncio
from src.orchestrator import MultiAgentOrchestrator
from src.agents.perplexica_search_agent import PerplexicaSearchAgent
from src.agents.planning_agent import PlanningAgent

async def main():
    # Initialise orchestrator with persistent memory
    orchestrator = MultiAgentOrchestrator(use_memory=True)
    await orchestrator.initialize()

    # Register specialist agents
    orchestrator.add_agent("web_search", PerplexicaSearchAgent())
    orchestrator.add_agent("planner", PlanningAgent())
    orchestrator.build_graph()

    # Process a research request
    result = await orchestrator.process_request(
        "Summarise the latest quantum computing breakthroughs",
        thread_id="research-session-01"
    )

    print(result["messages"][-1].content)
    await orchestrator.cleanup()

asyncio.run(main())
```

### Custom Agent

```python
from src.orchestrator import BaseAgent, MultiAgentState
from typing import Dict, Any

class MySummarisationAgent(BaseAgent):
    def __init__(self):
        super().__init__("summariser")

    async def process(self, state: MultiAgentState) -> Dict[str, Any]:
        messages = state["messages"]
        last_content = messages[-1].content
        summary = f"Summary: {last_content[:200]}..."  # plug in your LLM here
        return {"messages": [{"role": "assistant", "content": summary}]}
```

---

## 🔧 Setup Guide

See **[SETUP.md](SETUP.md)** for step-by-step instructions to configure:

- Perplexica (AI-powered search interface)
- SearXNG (privacy-focused metasearch engine backend)
- Redis, Qdrant, Neo4j via Docker

---

## 📸 Screenshots / Demo

> **Coming soon** — screenshots and a demo GIF showing a multi-step research workflow will be added here.

*To reproduce a demo yourself:*
```bash
python demo_research.py
```

---

## 🧩 Technical Implementation Details

### Custom `task_queue_reducer`
LangGraph's default state reducers concatenate lists. This project implements a custom `task_queue_reducer` that performs an **upsert by task ID**, preventing duplicate tasks from accumulating during agent retries — a subtle but critical correctness fix.

### Async Context Manager Pattern
All agents implement `__aenter__` / `__aexit__`, allowing safe resource acquisition and cleanup in `async with` blocks even when exceptions occur mid-execution.

### Resilient Redis Integration
Redis connections use `asyncio.wait_for` with a configurable timeout, and the `BaseAgent.cleanup()` uses `aclose()` (not deprecated `close()`) for proper connection teardown.

### Input Sanitisation
User-supplied strings are passed through `html.escape()` and validated with regex before being embedded in LLM prompts, mitigating prompt injection via web content.

### 12-Factor Config
All tuneable parameters — URLs, passwords, model paths, timeouts — are read exclusively from environment variables with safe defaults, making the system deployable to any environment without code changes.

---

## 🚧 Challenges & Solutions

| Challenge | Solution |
|---|---|
| LangGraph state list accumulation | Custom `task_queue_reducer` with upsert-by-ID semantics |
| Redis async deprecation warnings | Migrated from `close()` to `aclose()` throughout |
| Agent resource leaks on exception | Async context manager (`__aenter__`/`__aexit__`) on every agent |
| Hard-coded credentials in config | Refactored to 100% environment-variable-driven `Config` class |
| Cross-session memory in LangGraph | Integrated Mem0 + Qdrant vector store + Neo4j knowledge graph |

---

## 🔮 Future Improvements

- [ ] **Docker Compose** — single `docker compose up` to start all services
- [ ] **REST API** — FastAPI wrapper to expose the orchestrator as a web service
- [ ] **Streaming responses** — real-time SSE output as agents produce partial results
- [ ] **Agent evaluation** — automated benchmarks using LangSmith tracing
- [ ] **RAG pipeline** — document ingestion into Qdrant for domain-specific knowledge bases
- [ ] **Web UI** — Streamlit/Gradio interface for non-technical users

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Agent Framework | [LangGraph](https://github.com/langchain-ai/langgraph) 0.2+, [LangChain](https://www.langchain.com/) 0.3+ |
| LLM Backend | llama.cpp (OpenAI-compatible endpoint), Hermes-2-Pro-Mistral |
| Web Search | [Perplexica](https://github.com/ItzCrazyKns/Perplexica) + [SearXNG](https://searxng.github.io/searxng/) |
| Memory | [Mem0](https://mem0.ai/) + [Qdrant](https://qdrant.tech/) + [Neo4j](https://neo4j.com/) |
| State Store | [Redis](https://redis.io/) 7+ (async via `aioredis`) |
| HTTP Client | [aiohttp](https://docs.aiohttp.org/) |
| Language | Python 3.12+, fully async |

---

## 📄 License

This project is licensed under the **MIT License** — see the [LICENSE](LICENSE) file for details.

---

## 👤 Author

**Yuvraj**
- GitHub: [@Yuvraaaaj](https://github.com/Yuvraaaaj)

---

<div align="center">
  <i>Built as a deep dive into production-grade agentic AI systems — multi-agent coordination, persistent memory, and async Python at scale.</i>
</div>

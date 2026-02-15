# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

BlacksmithAI is an AI-powered penetration testing framework that uses a multi-agent system to automate security assessments. The system orchestrates specialized agents through the complete penetration testing lifecycle—from reconnaissance to post-exploitation.

## Build, Test, and Run Commands

### Setup and Installation
```bash
# Complete initial setup (installs deps, builds docker, installs frontend)
make setup

# Configure environment
cp blacksmithAI/.env.example blacksmithAI/.env
# Edit .env with API keys (OPENROUTER_API_KEY, LANGSMITH_API_KEY, etc.)
# Edit blacksmithAI/config.json to configure LLM provider

# Embed tool documentation (required for agent tool usage)
make embed-tools
# or: uv run blacksmithAI/blacksmithAI/update_tool_documentation.py
```

### Running the Application

**CLI Mode (Simplest)**
```bash
make start-cli
# or: cd blacksmithAI && docker compose up -d && uv run main.py
```

**Web UI Mode (requires 3 terminals)**
```bash
# Terminal 1: Start Docker services
cd blacksmithAI && docker compose up -d

# Terminal 2: Build and start frontend
cd frontend && pnpm build && pnpm start

# Terminal 3: Start LangGraph dev server
cd blacksmithAI && uv run langgraph dev

# Access at http://localhost:3000
```

**Individual Commands**
```bash
# Install Python dependencies
cd blacksmithAI && uv sync

# Install frontend dependencies
cd frontend && pnpm install

# Start Docker services only
cd blacksmithAI && docker compose up -d

# Stop Docker services
cd blacksmithAI && docker compose down

# View Docker logs
cd blacksmithAI && docker compose logs -f
```

### Development
```bash
# Start LangGraph dev server (for UI development)
cd blacksmithAI && uv run langgraph dev

# Start frontend dev server
cd frontend && pnpm dev

# Check dependencies
make check-deps

# Check configuration
make check-config
```

### Using Local LLM (VLLM)
```bash
# Install VLLM
make vllm-install
# or: cd blacksmithAI && uv add vllm && uv add huggingface_hub

# Start VLLM server (requires GPU)
make vllm-serve
# or smaller model:
make vllm-serve-small

# Update config.json to use provider: "vllm"
```

## Architecture

### Multi-Agent System

The system uses a hierarchical agent architecture built on **LangGraph** and **DeepAgents**:

```
Orchestrator Agent (main.py)
    ├── ReconAgent - Attack surface mapping, DNS enumeration, subdomain discovery
    ├── ScanEnumAgent - Deep scanning, service enumeration, version detection
    ├── VulnMapAgent - Vulnerability mapping, CVE correlation, risk assessment
    ├── ExploitAgent - Proof-of-concept exploitation, controlled validation
    ├── PostExploitAgent - Impact assessment, lateral movement analysis
    └── PentestAgent - General pentest operations
```

**Key Implementation Details:**
- **Framework**: `deepagents.create_deep_agent()` creates the orchestrator with subagent delegation
- **Communication**: Agents delegate tasks to specialized subagents and synthesize findings
- **Memory**: Uses `InMemorySaver` for checkpointing conversation state
- **Middleware**: `TodoListMiddleware` for task management, `ToolRetryMiddleware` for resilience (max 3 retries)

**Agent Definitions:** Each agent is defined in `/home/yohannes/projects/blacksmithAI/blacksmithAI/agents/` with:
- System prompt with phase-specific instructions
- Tool access based on pentest phase (defined in `config.json`)
- Wrapped as `CompiledSubAgent` for orchestrator integration

### LLM Provider System

**Location:** `/home/yohannes/projects/blacksmithAI/blacksmithAI/agents/base.py`

The system supports multiple LLM providers through a flexible configuration:

```python
# Provider selection in config.json
"defaults": { "provider": "openrouter" }

# Provider configuration
"providers": {
  "openrouter": {
    "base_url": "https://openrouter.ai/api/v1",
    "default_model": "openrouter/pony-alpha",
    "default_embedding_model": "openai/text-embedding-3-small",
    "default_model_config": {
      "context_size": 200000,  # Required for context window management
      "max_retries": 3,
      "stream_usage": true,
      "max_tokens": null
    }
  }
}
```

**Adding New Providers:**
1. Add provider config to `config.json` under `providers`
2. Add API key to `.env` as `{PROVIDER_NAME}_API_KEY` (uppercase)
3. Update `defaults.provider` to use the new provider

**API Key Pattern:** `{PROVIDER_NAME}_API_KEY` (e.g., `OPENROUTER_API_KEY`, `OPENAI_API_KEY`)

### Tool Execution Layer

**Location:** `/home/yohannes/projects/blacksmithAI/blacksmithAI/tools/tools.py`

The system provides two core tools to all agents:

1. **`pentest_shell`**: Executes shell commands in the isolated mini-kali Docker container
   - Sends HTTP POST to `container_uri` (default: `http://mini-kali-slim:9756`)
   - Returns stdout/stderr from tool execution
   - All pentest tools run non-interactively

2. **`shell_documentation`**: RAG-based tool documentation search
   - Queries ChromaDB vector store for relevant tool usage examples
   - Agents use this before executing unfamiliar tools
   - Documentation stored in `/home/yohannes/projects/blacksmithAI/blacksmithAI/agents/tools_doc/`

**Tool Organization:** Defined in `/home/yohannes/projects/blacksmithAI/blacksmithAI/config.json` under `tools` key, organized by pentest phase:
- `reconnaissance`: whois, dig, dnsrecon, assetfinder, subfinder
- `scanning_enumeration`: nmap, masscan, enum4linux-ng, gobuster, wpscan
- `vulnerability_mapping`: nuclei, sslscan
- `exploitation`: sqlmap, hydra, medusa, ncrack
- `post_exploitation`: netcat, socat, hping3, impacket CLIs
- `general`: python3, curl, httpie, openssh-client

### Vector Store (RAG)

**Location:** `/home/yohannes/projects/blacksmithAI/blacksmithAI/utils/vectors.py`

- **Database**: ChromaDB (persistent at `blacksmithAI/store/vector_db/`)
- **Purpose**: Store tool documentation for agents to query
- **Usage**: `storage_manager.embed_documents()` to index docs, `storage_manager.query()` to search
- **Embedding Model**: Configured in `config.json` as `default_embedding_model`

**Indexing Tool Documentation:**
```bash
# Run this after setup or when adding new tools
uv run blacksmithAI/blacksmithAI/update_tool_documentation.py
```

This script loads markdown docs from `blacksmithAI/agents/tools_doc/` and indexes them in ChromaDB.

### Docker Services

**Location:** `/home/yohannes/projects/blacksmithAI/docker-compose.yml`

Four services orchestrate the runtime environment:

1. **mini-kali-slim**: Security tools container (port 9756)
   - Provides all penetration testing tools
   - Exposes HTTP API for command execution
   - Built from local Dockerfile

2. **blacksmith-redis**: Redis for caching (port 6379)

3. **blacksmith-postgres**: PostgreSQL for persistence (port 5432)
   - **Note**: Volume mount uses `/var/lib/postgresql` but should be `/var/lib/postgresql/data`

4. **blacksmithV1**: Main application container (port 2024→8000)
   - Depends on healthy state of all other services
   - Mounts `config.json` read-only
   - Receives environment variables from `.env` and docker-compose

**Service Dependencies**: `blacksmithV1` waits for all services to pass health checks before starting

### Frontend Architecture

**Location:** `/home/yohannes/projects/blacksmithAI/frontend/`

- **Framework**: Next.js 15 with App Router
- **React**: Version 19
- **Styling**: Tailwind CSS
- **State**: React Context providers in `src/providers/`
- **Components**: Organized in `src/components/` (thread, ui, icons)

**Frontend-Backend Communication:**
- Frontend connects to LangGraph dev server (via `langgraph dev`)
- Uses assistant-ui library for chat interface
- Streams responses from LangGraph agents

## Project Structure

```
blacksmithAI/
├── blacksmithAI/              # Python backend
│   ├── agents/                # Agent definitions
│   │   ├── base.py           # Model initialization (init_model)
│   │   ├── recon.py          # Reconnaissance agent
│   │   ├── scan_enum.py      # Scanning/Enumeration agent
│   │   ├── vuln_map.py       # Vulnerability mapping agent
│   │   ├── exploit.py        # Exploitation agent
│   │   ├── post_exploit.py   # Post-exploitation agent
│   │   ├── pentester.py      # General pentest agent
│   │   └── tools_doc/        # Tool documentation by phase
│   ├── tools/
│   │   └── tools.py          # Core tools (pentest_shell, shell_documentation)
│   ├── utils/
│   │   ├── loader.py         # Document loading utilities
│   │   └── vectors.py        # ChromaDB storage manager
│   ├── store/vector_db/      # Persistent vector storage
│   ├── main.py               # Orchestrator agent & CLI entrypoint
│   ├── config.json           # LLM provider & tool configuration
│   ├── langgraph.json        # LangGraph agent graph definitions
│   └── pyproject.toml        # Python dependencies (uv)
├── frontend/                  # Next.js web UI
│   ├── src/
│   │   ├── app/              # App router pages
│   │   ├── components/       # React components
│   │   ├── lib/              # Utilities
│   │   ├── providers/        # Context providers
│   │   └── hooks/            # Custom hooks
│   └── package.json
├── docker-compose.yml         # Docker services
├── Makefile                   # Build/run commands
└── README.md                  # User documentation
```

## Key Configuration Files

| File | Purpose |
|------|---------|
| `blacksmithAI/config.json` | LLM provider config, available tools per phase, context sizes |
| `blacksmithAI/langgraph.json` | LangGraph agent graph definitions for dev server |
| `blacksmithAI/.env` | API keys (`{PROVIDER}_API_KEY`), `LANGSMITH_API_KEY`, database credentials |
| `docker-compose.yml` | Docker services, ports, volumes, health checks |
| `blacksmithAI/pyproject.toml` | Python dependencies managed by uv |

## Important Patterns

1. **Agent Composition**: Orchestrator uses `create_deep_agent()` with list of `CompiledSubAgent` objects
2. **Tool Abstraction**: All pentest tools accessed through `pentest_shell` which executes in isolated Docker container
3. **RAG for Documentation**: Agents query ChromaDB via `shell_documentation` tool before using unfamiliar tools
4. **Streaming Responses**: Uses LangGraph's `astream()` for real-time feedback in CLI
5. **Middleware Pipeline**: `TodoListMiddleware` + `ToolRetryMiddleware` for robustness
6. **Non-Interactive Tools**: All security tools must run non-interactively (stdin/stdout only)

## Security Architecture

- **Tool Isolation**: All penetration testing tools run in mini-kali Docker container
- **Authorization**: System assumes user has authorization for target systems
- **LLM Injection Protection**: Orchestrator system prompt warns against revealing internal design
- **Malicious Input Protection**: Agents instructed to validate/sanitize inputs from users and subagents

## Tech Stack

| Layer | Technology |
|-------|------------|
| **Backend** | Python 3.12, LangChain, LangGraph, DeepAgents |
| **Frontend** | Next.js 15, React 19, TypeScript, Tailwind CSS |
| **LLM** | OpenRouter / VLLM / OpenAI-compatible APIs |
| **Vector DB** | ChromaDB with OpenAI-compatible embeddings |
| **Containerization** | Docker, Docker Compose |
| **Package Managers** | uv (Python), pnpm (Node.js) |

## Common Development Tasks

### Adding a New Tool
1. Install tool in mini-kali Docker image (update Dockerfile)
2. Add tool name to appropriate phase in `config.json` under `tools`
3. Add markdown documentation to `blacksmithAI/agents/tools_doc/{phase}/`
4. Run `make embed-tools` to index new documentation

### Changing LLM Provider
1. Add provider config to `blacksmithAI/config.json`
2. Add API key to `blacksmithAI/.env` as `{PROVIDER}_API_KEY`
3. Update `defaults.provider` in config.json
4. Restart application

### Modifying Agent Behavior
1. Edit agent's system prompt in `/home/yohannes/projects/blacksmithAI/blacksmithAI/agents/{agent_name}.py`
2. Adjust tool access by modifying `tools` parameter in `create_agent()`
3. Update middleware configuration as needed
4. Restart CLI or LangGraph dev server

### Debugging Agent Issues
- View LangGraph traces in LangSmith (if `LANGSMITH_API_KEY` configured)
- Check agent logs: `logging.getLogger('agent_name')`
- Inspect Docker container: `docker logs mini-kali-slim`
- Monitor tool execution in vector store queries

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

OpenMemory is a self-hosted AI memory engine implementing **Hierarchical Memory Decomposition (HMD)** architecture. It's a TypeScript/Node.js monorepo with three main components:
- **Backend server** (`backend/`) - Core memory engine with REST API
- **JavaScript SDK** (`sdk-js/`) - Local-first standalone SDK
- **Python SDK** (`sdk-py/`) - Python equivalent of JS SDK
- **Dashboard** (`dashboard/`) - Next.js web UI for memory visualization

The system provides multi-sector cognitive memory (episodic, semantic, procedural, emotional, reflective) with vector embeddings, temporal knowledge graphs, and explainable recall paths.

## Common Commands

### Development
```bash
# Install all dependencies
make install

# Start backend development server (port 8080)
cd backend && npm run dev

# Start dashboard development server (port 3000)
cd dashboard && npm run dev

# Build all components
make build

# Build backend only
cd backend && npm run build
```

### Testing
```bash
# Run all tests
make test

# Run backend API tests only
node tests/backend/api-simple.test.js

# Run JS SDK tests only
node tests/js-sdk/sdk-simple.test.js

# Run Python SDK tests only
cd tests/py-sdk && python test-simple.py

# Run integration tests
node tests/backend/api.test.js
```

### Docker
```bash
# Start with Docker Compose
docker compose up --build -d

# Stop containers
docker compose down

# View logs
docker compose logs -f
```

### CLI Tool
```bash
# Link CLI globally (from backend/)
cd backend && npm link

# Add memory
opm add "content" --user u1 --tags tag1,tag2

# Query memory
opm query "search text" --user u1 --limit 5

# List memories
opm list --user u1

# Reinforce memory
opm reinforce <memory-id>

# System stats
opm stats
```

### Code Quality
```bash
# Format code
cd backend && npm run format

# Type check
cd backend && npx tsc --noEmit
cd sdk-js && npx tsc --noEmit
```

## Claude Commands Integration

This repository includes the `claude-commands` package, which provides 13 powerful slash commands for Claude Code workflow enhancement.

### Installation

The package is already installed as a dev dependency. If you need to reinstall:

```bash
# Install from private repository
npm install --save-dev git+ssh://git@github.com/natesena/claude-commands.git

# Manually run postinstall if needed
node node_modules/@nathanielsena/claude-commands/scripts/postinstall.js
```

### Available Commands

**Archon Integration (4 commands):**
- `/archon-upload` - Upload documents to Archon RAG system
- `/archon-research` - Parallel research using Archon with multiple perspectives
- `/archon-delete` - Delete documents from Archon
- `/todo` - Sync todos with Archon task management

**Planning & Workflow (3 commands):**
- `/new` - Structured task planning with implementation steps
- `/create-prd` - Generate Product Requirements Documents
- `/prd-comments` - Add comments to existing PRDs

**Git & Development (2 commands):**
- `/quick-worktree` - Create and manage git worktrees efficiently
- `/pre-push` - Pre-push validation workflow

**Debugging (4 commands):**
- `/debug` - General debugging workflow
- `/debug-local` - Local environment debugging
- `/debug-staging` - Staging environment debugging
- `/chrome-debug` - Chrome DevTools debugging workflow

### Customization

Commands are located in `.claude/commands/` and can be customized per-project. The installation script never overwrites existing commands, preserving your customizations.

To exclude specific commands from installation, create `.claude/shared-commands.json`:
```json
{
  "exclude": ["debug-local.md", "debug-staging.md"]
}
```

### Configuration Notes

- **Archon API**: Default URL is `http://100.74.174.10:8181` (can be modified in command files)
- **Project-specific paths**: Some commands may reference Kismet-specific paths - update as needed
- **Version control**: Commands in `.claude/commands/` should be committed to track project-specific customizations

## Architecture

### High-Level Structure

The backend is organized into distinct modules under `backend/src/`:

- **`server/`** - HTTP API layer with Express-like routing
  - `routes/` - Endpoint handlers for memory operations, ingestion, temporal graph, MCP
  - `middleware/` - Authentication, rate limiting
  - `server.js` - Main server with scheduled decay and pruning

- **`memory/`** - Core memory engine (HSG implementation)
  - `hsg.ts` - Hierarchical Sectored Graph with multi-sector embedding, waypoint linking, composite scoring
  - `embed.ts` - Multi-provider embedding generation (OpenAI, Gemini, AWS, Ollama, synthetic)
  - `decay.ts` - Time-based memory decay with sector-specific rates and multi-threaded processing
  - `reflect.ts` - Auto-reflection system for pattern detection
  - `user_summary.ts` - User memory summarization

- **`core/`** - Database and configuration
  - `db.ts` - SQLite/Postgres abstraction with transaction support
  - `cfg.ts` - Environment configuration management
  - `types.ts` - TypeScript interfaces and types
  - `models.ts` - Embedding model configurations per sector

- **`ops/`** - Operations and utilities
  - Ingestion pipeline (PDF, DOCX, TXT, HTML, Audio/Video via Whisper)
  - Pattern clustering and consolidation

- **`temporal_graph/`** - Time-bound knowledge graph
  - Fact storage with `valid_from`/`valid_to` timestamps
  - Point-in-time queries and evolution tracking

- **`ai/`** - LLM integrations
  - LangGraph mode endpoints and node-to-sector mapping

### Memory Sectors

The system classifies memories into 5 cognitive sectors, each with unique decay rates and patterns:

| Sector | Decay Lambda | Weight | Purpose |
|--------|--------------|--------|---------|
| `episodic` | 0.015 | 1.2 | Events and experiences (time-bound) |
| `semantic` | 0.005 | 1.0 | Facts and knowledge (timeless) |
| `procedural` | 0.008 | 1.1 | Skills and how-to patterns |
| `emotional` | 0.020 | 1.3 | Feelings and sentiment states |
| `reflective` | 0.001 | 0.8 | Meta-cognition and insights |

### Memory Operations Flow

**Adding Memory:**
1. Classify content into sectors (regex patterns + confidence)
2. Generate embeddings per sector via batch API
3. Calculate mean vector for waypoint matching
4. Store in database transaction: memory → vectors → waypoints
5. Create single-waypoint link to best match (similarity ≥ 0.75)

**Querying Memory:**
1. Classify query into candidate sectors
2. Generate query embeddings per sector
3. Vector similarity search (cosine) per sector → top-K
4. Expand via waypoint graph (1-hop traversal)
5. Composite scoring: `0.6×similarity + 0.2×salience + 0.1×recency + 0.1×waypoint`
6. Reinforce: boost salience (+0.1), strengthen waypoints (+0.05)

### Performance Tiers

The system supports 4 performance tiers controlled by `OM_TIER` environment variable:

- **HYBRID** - Keyword + Synthetic (256-dim) with BM25 ranking - 100% recall, 800-1000 QPS
- **FAST** - Synthetic embeddings only (256-dim) - 70-75% recall, 700-850 QPS
- **SMART** - Hybrid synthetic + compressed semantic (384-dim) - 85% recall, 500-600 QPS
- **DEEP** - Full AI embeddings (1536-dim) - 95-100% recall, 350-400 QPS

### Database Schema

**memories table:**
- `id`, `content`, `primary_sector`, `tags`, `meta` (JSON)
- `created_at`, `updated_at`, `last_seen_at`
- `salience` (0-1), `decay_lambda` (sector-specific)
- `mean_vec` (BLOB) - mean vector for waypoint matching

**vectors table:**
- `id` (memory ID), `sector`, `v` (BLOB), `dim`
- Primary key: `(id, sector)`

**waypoints table:**
- `src_id`, `dst_id`, `weight` (0-1)
- Single-waypoint architecture: one strongest link per memory

**embed_logs table:**
- Tracks embedding operations for monitoring and debugging

### Embedding Providers

Configured via `OM_EMBEDDINGS` environment variable:

- **openai** - `text-embedding-3-small`, `text-embedding-3-large` (batch support)
- **gemini** - `embedding-001` (batch support)
- **aws** - `amazon.titan-embed-text-v2:0` (no batch)
- **ollama** - Local models: `nomic-embed-text`, `bge-small`, `bge-large`
- **local** - Custom model via `LOCAL_MODEL_PATH`
- **synthetic** - Hash-based embeddings (zero cost, fast, 70-75% recall)

**Embedding Modes:**
- `simple` mode (default) - Single batch call for all sectors (faster, rate-limit safe)
- `advanced` mode - Separate calls per sector (higher precision, more API overhead)

### Remote Ollama Configuration

OpenMemory fully supports remote Ollama instances, including via Tailscale networking. The implementation uses dynamic URL configuration with **no hardcoded localhost references**.

**Configuration in `backend/.env`:**
```bash
# Embedding provider
OM_EMBEDDINGS=ollama

# Remote Ollama instance (Tailscale or any HTTP-accessible URL)
OLLAMA_URL=http://100.74.174.10:11434  # Your Tailscale IP
# or
OLLAMA_URL=http://your-tailscale-hostname:11434

# Performance tier (SMART or DEEP required for Ollama)
OM_TIER=smart  # Uses Ollama for compressed semantic (384-dim)
# or
OM_TIER=deep   # Uses Ollama for full embeddings (1536-dim)
```

**Important Notes:**
- **HYBRID** and **FAST** tiers use synthetic embeddings only (Ollama not invoked)
- **SMART** tier uses Ollama for 128-dim compressed semantic embeddings
- **DEEP** tier uses Ollama for full 1536-dim embeddings
- The embedding implementation (`backend/src/memory/embed.ts:218-227`) uses `fetch()` with `env.ollama_url`
- Works with any HTTP-accessible Ollama instance (local network, Tailscale, WireGuard, etc.)
- Default model is `nomic-embed-text` (configurable via `models.yml`)

**Testing Remote Connection:**
```bash
# From OpenMemory host, verify Ollama is reachable
curl http://100.74.174.10:11434/api/version

# Start OpenMemory backend
cd backend && npm run dev

# Test embedding generation
curl -X POST http://localhost:8080/memory/add \
  -H "Content-Type: application/json" \
  -d '{"content": "test memory", "user_id": "test"}'
```

**Troubleshooting:**
- Ensure firewall allows connections to port 11434
- Verify Tailscale is running on both machines
- Check Ollama logs for connection attempts
- Confirm the model exists: `ollama list` (on Ollama host)
- If using custom models, update `backend/src/core/models.ts`

### Decay System

Time-based decay with multi-threaded processing:
- Runs every `OM_DECAY_INTERVAL_MINUTES` (default: 120 minutes)
- Formula: `salience × e^(-decay_lambda × days_since_access)`
- Three-tier system: hot (≥0.5), warm (0.25-0.5), cold (<0.25)
- Cold memories get fingerprinted (compressed) to save space
- Regeneration on query hits for cold memories

### Waypoint Graph

Single-waypoint associative linking:
- Each memory links to its single strongest match (bidirectional if cross-sector)
- Created during add if similarity ≥ 0.75
- Reinforced on recall (+0.05 weight, max 1.0)
- Pruned every 7 days (remove weights < 0.05)

## Key Files to Know

### Backend Core
- `backend/src/memory/hsg.ts` - Main memory engine with classification, embedding coordination, scoring
- `backend/src/memory/embed.ts` - Multi-provider embedding generation with batch support
- `backend/src/memory/decay.ts` - Decay algorithm with tiered compression
- `backend/src/core/db.ts` - Database layer with transaction support
- `backend/src/server/server.js` - Main HTTP server with scheduled tasks

### SDKs
- `sdk-js/src/index.ts` - JavaScript SDK entry point (local-first mode)
- `sdk-js/src/memory/hsg.ts` - SDK version of HSG engine
- `sdk-py/src/openmemory/` - Python SDK implementation

### Configuration
- `.env.example` - Comprehensive environment variable documentation
- `models.yml` - Embedding model configuration per sector
- `docker-compose.yml` - Docker deployment configuration

## Development Patterns

### Adding New API Endpoints

1. Create route handler in `backend/src/server/routes/`
2. Add route registration in `backend/src/server/index.ts`
3. Update middleware for auth if needed
4. Add corresponding tests in `tests/backend/`

### Working with Memory Sectors

Sectors are auto-classified using regex patterns in `hsg.ts`:
```typescript
SECTORS.episodic.patterns = [/today|yesterday|remember when/i, ...]
SECTORS.semantic.patterns = [/define|meaning|concept/i, ...]
```

To modify sector classification:
1. Edit patterns in `backend/src/memory/hsg.ts`
2. Adjust decay rates and weights if needed
3. Test with various content types

### Adding Embedding Providers

1. Add provider implementation in `backend/src/memory/embed.ts`
2. Update `getEmbedding()` and `getEmbeddingsBatch()` functions
3. Add environment variables to `.env.example`
4. Update `models.yml` for sector-specific model selection

### Testing Changes

- Always run tests before committing: `make test`
- Backend tests use port 8080 - ensure dev server is stopped
- SDK tests create temporary databases - they clean up automatically
- Integration tests require backend to be running

### Database Migrations

SQLite schema is initialized on first run in `backend/src/core/db.ts`:
- Add new migrations in `init()` function
- Use transactions for schema changes
- Test with fresh database: `rm backend/data/*.db`

## MCP Integration

OpenMemory provides Model Context Protocol (MCP) server at `/mcp` endpoint:

**Tools provided:**
- `openmemory_query` - Search memories with filters
- `openmemory_store` - Add new memories
- `openmemory_list` - List all memories with pagination
- `openmemory_get` - Get specific memory by ID
- `openmemory_reinforce` - Boost memory salience

**Setup for Claude Desktop/Code:**
```bash
claude mcp add --transport http openmemory http://localhost:8080/mcp
```

## Environment Configuration

Key environment variables (see `.env.example` for complete list):

**Required:**
- `OM_TIER` - Performance tier: `hybrid`, `fast`, `smart`, or `deep`
- `OM_EMBEDDINGS` - Provider: `openai`, `gemini`, `aws`, `ollama`, `synthetic`

**Database:**
- `OM_METADATA_BACKEND` - `sqlite` or `postgres`
- `OM_VECTOR_BACKEND` - `sqlite`, `pgvector`, or `weaviate`
- `OM_DB_PATH` - SQLite database path (default: `./data/openmemory.sqlite`)

**Server:**
- `OM_PORT` - Server port (default: 8080)
- `OM_API_KEY` - Optional bearer token for auth
- `OM_MODE` - `standard` or `langgraph`

**Ollama Configuration (Remote & Local):**
- `OLLAMA_URL` - Ollama instance URL (default: `http://localhost:11434`)
  - Examples: `http://100.74.174.10:11434` (Tailscale), `http://192.168.1.100:11434` (LAN)
  - Fully supports remote instances via Tailscale, VPN, or any HTTP-accessible URL
  - No hardcoded localhost restrictions in codebase

**Memory System:**
- `OM_DECAY_INTERVAL_MINUTES` - Decay cycle frequency (default: 120)
- `OM_MIN_SCORE` - Minimum similarity threshold (default: 0.3)
- `OM_AUTO_REFLECT` - Enable auto-reflection (default: false)

## LangGraph Integration

When `OM_MODE=langgraph`, additional endpoints are available under `/lgm/`:

- `/lgm/store` - Store LangGraph node outputs with auto-sector mapping
- `/lgm/retrieve` - Retrieve memories for graph sessions
- `/lgm/context` - Get multi-sector context assembly
- `/lgm/reflection` - Generate and store reflections

**Node-to-Sector Mapping:**
- `observe` → episodic
- `plan` → semantic
- `reflect` → reflective
- `act` → procedural
- `emotion` → emotional

## Ingestion Pipeline

Supports multiple document formats via `POST /memory/ingest`:

**Text formats:** PDF (pdf-parse), DOCX (mammoth), TXT, MD, HTML (turndown)
**Media:** Audio (Whisper API), Video (FFmpeg + Whisper)

**Configuration:**
- `chunk_size: 2048` tokens per chunk
- `chunk_overlap: 256` tokens overlap
- 25MB file size limit for audio/video
- Requires `OPENAI_API_KEY` for transcription

## Temporal Knowledge Graph

Time-bound facts with automatic evolution:

```typescript
// Add fact with validity period
POST /api/temporal/fact
{
  subject: "CompanyX",
  predicate: "has_CEO",
  object: "Alice",
  valid_from: "2021-01-01",
  valid_to: "2024-04-10" // auto-closed when new fact added
}
```

**Operations:**
- Point-in-time queries: "what was true on X date?"
- Timeline reconstruction for entities
- Confidence decay over time
- Volatile fact detection

## Performance Benchmarks

At 100k memories:
- Add memory: 80-120ms (depends on embedding provider)
- Query (single-sector): 110-130ms
- Query (multi-sector): 150-200ms
- Waypoint expansion: +30-50ms per hop
- Decay process: ~10 seconds (background, every 2 hours)

Storage (SQLite):
- ~4-6 KB per memory (including vectors)
- ~500 MB for 100k memories
- ~5 GB for 1M memories with indexing

## Project Structure Notes

This is a monorepo with independent packages:
- Root `package.json` delegates to backend
- Each SDK has its own build process
- Dashboard is a separate Next.js app
- Tests are at repo root under `tests/`

The `Makefile` provides convenient commands for cross-package operations.

## VS Code Extension

OpenMemory includes a VS Code extension (`IDE/`) that tracks coding activity and provides memory context to MCP clients. It's published as `openmemory-vscode` on the VS Code marketplace.

## Migration Tool

Located in `migrate/` - imports memories from Mem0, Zep, and Supermemory with rate limiting, resume support, and verification mode.

## Important Notes

- The system auto-creates database and tables on first run
- SQLite uses WAL mode for better write concurrency
- Vector similarity uses cosine distance
- All timestamps are Unix epoch milliseconds
- User isolation via `user_id` field (optional but recommended)
- Telemetry is enabled by default (opt-out via `OM_TELEMETRY=false`)

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Graphiti is a Python framework for building temporally-aware knowledge graphs designed for AI agents. It enables real-time incremental updates to knowledge graphs without batch recomputation, making it suitable for dynamic environments. Graphiti powers the core of Zep's memory layer for AI Agents and represents the state-of-the-art in agent memory systems.

### Core Capabilities

- **Bi-temporal data model** with explicit tracking of event occurrence times (valid_at) and ingestion times (created_at)
- **Hybrid retrieval** combining semantic embeddings, keyword search (BM25), and graph traversal
- **Support for custom entity definitions** via Pydantic models
- **Integration with Neo4j and FalkorDB** as graph storage backends
- **Real-time incremental updates** without requiring batch recomputation
- **Temporal edge invalidation** to handle changing information while preserving history
- **Community detection** for clustering related entities

### Performance

- Sub-second query latency (P95: 300ms)
- 94.8% accuracy on Deep Memory Retrieval benchmark (vs 93.4% for MemGPT)
- 18.5% accuracy improvement on LongMemEval benchmark
- 90% reduction in response latency compared to traditional approaches

## Architecture Overview

### Graph Structure

Graphiti uses three hierarchical subgraphs:

1. **Episode Subgraph**: Stores raw input data as non-lossy "episodic nodes" (EpisodicNode)
2. **Semantic Entity Subgraph**: Extracts and resolves entities from episodes (EntityNode)
3. **Community Subgraph**: Clusters strongly connected entities for high-level summarization (CommunityNode)

### Edge Types

- **EpisodicEdge**: Links episodes to entities (MENTIONS relationship)
- **EntityEdge**: Links entities to entities (RELATES_TO relationship) with temporal metadata
- **CommunityEdge**: Links communities to entities/communities (HAS_MEMBER relationship)

### Key Components

1. **graphiti.py**: Main orchestrator class that coordinates all operations
2. **driver/**: Database abstraction layer (Neo4j, FalkorDB)
3. **search/**: Modular search system with configurable strategies
4. **llm_client/**: LLM provider integrations (OpenAI, Anthropic, Gemini, Groq)
5. **embedder/**: Embedding providers (OpenAI, Voyage, Gemini)
6. **prompts/**: Structured prompts for LLM tasks
7. **utils/**: Bulk operations, maintenance, and temporal utilities

## Development Commands

### Main Development Commands (run from project root)

```bash
# Install dependencies
uv sync --extra dev

# Format code (ruff import sorting + formatting)
make format

# Lint code (ruff + pyright type checking)
make lint

# Run tests
make test

# Run all checks (format, lint, test)
make check
```

### Server Development (run from server/ directory)

```bash
cd server/
# Install server dependencies
uv sync --extra dev

# Run server in development mode
uvicorn graph_service.main:app --reload

# Format, lint, test server code
make format
make lint
make test
```

### MCP Server Development (run from mcp_server/ directory)

```bash
cd mcp_server/
# Install MCP server dependencies
uv sync

# Run with Docker Compose
docker-compose up
```

## Code Architecture

### Core Library (`graphiti_core/`)

- **Main Entry Point**: `graphiti.py` - Contains the main `Graphiti` class that orchestrates all functionality
- **Graph Storage**: `driver/` - Database drivers for Neo4j and FalkorDB
- **LLM Integration**: `llm_client/` - Clients for OpenAI, Anthropic, Gemini, Groq
- **Embeddings**: `embedder/` - Embedding clients for various providers
- **Graph Elements**: `nodes.py`, `edges.py` - Core graph data structures
- **Search**: `search/` - Hybrid search implementation with configurable strategies
- **Prompts**: `prompts/` - LLM prompts for entity extraction, deduplication, summarization
- **Utilities**: `utils/` - Maintenance operations, bulk processing, datetime handling

### Server (`server/`)

- **FastAPI Service**: `graph_service/main.py` - REST API server
- **Routers**: `routers/` - API endpoints for ingestion and retrieval
- **DTOs**: `dto/` - Data transfer objects for API contracts

### MCP Server (`mcp_server/`)

- **MCP Implementation**: `graphiti_mcp_server.py` - Model Context Protocol server for AI assistants
- **Docker Support**: Containerized deployment with Neo4j

## Testing

- **Unit Tests**: `tests/` - Comprehensive test suite using pytest
- **Integration Tests**: Tests marked with `_int` suffix require database connections
- **Evaluation**: `tests/evals/` - End-to-end evaluation scripts

## Configuration

### Environment Variables

- `OPENAI_API_KEY` - Required for LLM inference and embeddings
- `USE_PARALLEL_RUNTIME` - Optional boolean for Neo4j parallel runtime (enterprise only)
- Provider-specific keys: `ANTHROPIC_API_KEY`, `GOOGLE_API_KEY`, `GROQ_API_KEY`, `VOYAGE_API_KEY`

### Database Setup

- **Neo4j**: Version 5.26+ required, available via Neo4j Desktop
  - Database name defaults to `neo4j` (hardcoded in Neo4jDriver)
  - Override by passing `database` parameter to driver constructor
- **FalkorDB**: Version 1.1.2+ as alternative backend
  - Database name defaults to `default_db` (hardcoded in FalkorDriver)
  - Override by passing `database` parameter to driver constructor

## Development Guidelines

### Code Style

- Use Ruff for formatting and linting (configured in pyproject.toml)
- Line length: 100 characters
- Quote style: single quotes
- Type checking with Pyright is enforced
- Main project uses `typeCheckingMode = "basic"`, server uses `typeCheckingMode = "standard"`

### Testing Requirements

- Run tests with `make test` or `pytest`
- Integration tests require database connections and are marked with `_int` suffix
- Use `pytest-xdist` for parallel test execution
- Run specific test files: `pytest tests/test_specific_file.py`
- Run specific test methods: `pytest tests/test_file.py::test_method_name`
- Run only integration tests: `pytest tests/ -k "_int"`
- Run only unit tests: `pytest tests/ -k "not _int"`

### LLM Provider Support

The codebase supports multiple LLM providers but works best with services supporting structured output (OpenAI, Gemini). Other providers may cause schema validation issues, especially with smaller models.

### MCP Server Usage Guidelines

When working with the MCP server, follow the patterns established in `mcp_server/cursor_rules.md`:

- Always search for existing knowledge before adding new information
- Use specific entity type filters (`Preference`, `Procedure`, `Requirement`)
- Store new information immediately using `add_memory`
- Follow discovered procedures and respect established preferences

## Common Implementation Patterns

### Episode-Based Data Ingestion

```python
# Text episodes for natural language
await graphiti.add_episode(
    name="Episode Name",
    episode_body="Natural language content",
    source=EpisodeType.text,
    reference_time=datetime.now(timezone.utc)
)

# JSON episodes for structured data
await graphiti.add_episode(
    name="Product Data",
    episode_body=json.dumps(product_dict),
    source=EpisodeType.json,
    reference_time=datetime.now(timezone.utc)
)
```

### Hybrid Search Strategies

```python
# Basic semantic + keyword search
results = await graphiti.search("query")

# Graph-aware search with center node
results = await graphiti.search(
    "query", 
    center_node_uuid=node_uuid,  # Rerank by graph distance
    num_results=10
)
```

### Custom Entity Types

```python
from pydantic import BaseModel

class Person(BaseModel):
    first_name: str
    last_name: str
    occupation: str

# Use in episode creation
await graphiti.add_episode(
    episode_body="Content",
    entity_types={'Person': Person}
)
```

## Key Concepts

### Temporal Model

- **valid_at**: When the fact/event actually occurred
- **created_at**: When the data was ingested into the system
- **invalid_at**: When a fact became invalid (for edge invalidation)

### Search Methods

- **Cosine similarity**: Vector search on embeddings
- **BM25**: Keyword/fulltext search
- **BFS**: Breadth-first graph traversal
- **Rerankers**: RRF, node distance, episode mentions, MMR, cross-encoder

### Episode Window

Graphiti maintains context by retrieving previous episodes within a time window (default 10 hours) to ensure continuity in entity extraction and relationship building.
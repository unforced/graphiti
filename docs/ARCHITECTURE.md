# Graphiti Architecture Documentation

## Overview

Graphiti is a temporal knowledge graph framework designed for AI agents that need dynamic, evolving memory. Unlike traditional RAG systems, Graphiti maintains a living knowledge graph that updates incrementally without batch recomputation.

## Core Architecture

### 1. Three-Layer Graph Structure

Graphiti implements a hierarchical graph architecture with three distinct subgraphs:

#### Episode Subgraph
- **Purpose**: Non-lossy storage of raw input data
- **Node Type**: `EpisodicNode`
- **Key Fields**:
  - `content`: Raw text or JSON data
  - `source_type`: `text` or `json`
  - `created_at`: Ingestion timestamp
  - `valid_at`: Event occurrence timestamp
  - `group_id`: Logical grouping identifier

#### Semantic Entity Subgraph
- **Purpose**: Extracted and resolved entities
- **Node Type**: `EntityNode`
- **Key Fields**:
  - `name`: Entity identifier
  - `summary`: LLM-generated description
  - `entity_type`: Custom type (e.g., Person, Product)
  - `embeddings`: Vector representation
  - Custom attributes via Pydantic models

#### Community Subgraph
- **Purpose**: Clusters of strongly connected entities
- **Node Type**: `CommunityNode`
- **Key Fields**:
  - `name`: Community identifier
  - `summary`: Aggregate description
  - `embedding`: Centroid vector

### 2. Edge Architecture

#### EpisodicEdge (MENTIONS)
- Links episodes to entities they mention
- Immutable once created
- Preserves extraction provenance

#### EntityEdge (RELATES_TO)
- Links entities to entities
- Contains temporal metadata:
  - `valid_at`: When relationship became true
  - `invalid_at`: When relationship ended
  - `created_at`: When edge was created
- Supports edge invalidation without deletion

#### CommunityEdge (HAS_MEMBER)
- Links communities to their member entities
- Updated during community rebuilding

### 3. Bi-Temporal Model

Graphiti tracks two distinct timelines:

1. **T (Event Time)**: When facts occurred in the real world
   - Stored in `valid_at` fields
   - Used for temporal queries

2. **T' (Transaction Time)**: When data was ingested
   - Stored in `created_at` fields
   - Used for audit trails

This enables queries like:
- "What did we know about X at time Y?"
- "When did we learn about relationship Z?"

## Data Flow Architecture

### Episode Ingestion Pipeline

```
1. Episode Input
   ├── Text Episode
   └── JSON Episode
   
2. Context Retrieval
   ├── Previous Episodes (10hr window)
   └── Mentioned Entities
   
3. LLM Extraction
   ├── Entity Extraction
   └── Relationship Extraction
   
4. Deduplication
   ├── Node Deduplication
   └── Edge Deduplication
   
5. Embedding Generation
   ├── Entity Embeddings
   └── Fact Embeddings
   
6. Graph Persistence
   ├── Create/Update Nodes
   ├── Create Edges
   └── Invalidate Edges
   
7. Post-Processing
   ├── Build Episodic Edges
   └── Update Communities
```

### Search Architecture

```
Query → Embedding → Parallel Search
                    ├── Node Search
                    │   ├── Cosine Similarity
                    │   ├── BM25 Fulltext
                    │   └── BFS Traversal
                    ├── Edge Search
                    │   ├── Cosine Similarity
                    │   └── BM25 Fulltext
                    ├── Episode Search
                    │   └── BM25 Fulltext
                    └── Community Search
                        └── Cosine Similarity
                        
Results → Reranking → Final Results
          ├── RRF
          ├── Node Distance
          ├── Episode Mentions
          ├── MMR
          └── Cross-Encoder
```

## Component Architecture

### Driver Layer
- **Interface**: `GraphDriver` abstract class
- **Implementations**:
  - `Neo4jDriver`: Primary production backend
  - `FalkorDBDriver`: Lightweight Redis-based alternative
- **Session Management**: Async context managers
- **Query Abstraction**: Database-agnostic operations

### LLM Integration Layer
- **Client Interface**: `LLMClient` abstract class
- **Providers**: OpenAI, Anthropic, Gemini, Groq
- **Structured Output**: Pydantic model validation
- **Batch Processing**: Concurrent API calls

### Embedding Layer
- **Client Interface**: `EmbedderClient` abstract class
- **Providers**: OpenAI, Voyage, Gemini
- **Dimension Support**: 768-3072 dimensions
- **Caching**: Built-in embedding cache

### Search Configuration
- **Modular Design**: Pluggable search strategies
- **Config Recipes**: Pre-defined search configurations
- **Custom Filters**: Entity types, time ranges, groups

## Performance Optimizations

### Parallel Processing
- Concurrent LLM calls for extraction
- Parallel embedding generation
- Batch database operations

### Indexing Strategy
- Vector indexes on embeddings
- Fulltext indexes on names/content
- Composite indexes on temporal fields

### Caching
- Episode window caching
- Embedding result caching
- Query result caching (15 min TTL)

## Scalability Considerations

### Horizontal Scaling
- Stateless API servers
- Read replicas for search
- Async processing queues

### Vertical Scaling
- Configurable batch sizes
- Memory-mapped embeddings
- Connection pooling

### Data Partitioning
- Group-based partitioning
- Time-based archival
- Community-based sharding

## Security Architecture

### Data Protection
- No PII in telemetry
- Encrypted database connections
- API key management

### Access Control
- Group-based isolation
- Read/write permissions
- Audit logging

## Extension Points

### Custom Entity Types
```python
class CustomEntity(BaseModel):
    custom_field: str
    
graphiti = Graphiti(
    entity_types={'CustomEntity': CustomEntity}
)
```

### Custom Search Strategies
```python
custom_config = SearchConfig(
    node_search_methods=[...],
    reranker=custom_reranker
)
```

### Custom LLM Providers
- Implement `LLMClient` interface
- Support structured output
- Handle rate limiting

## Deployment Architecture

### Standalone
- Single Graphiti instance
- Local Neo4j/FalkorDB
- Direct API calls

### Microservices
- FastAPI REST server
- MCP protocol server
- Background workers

### Cloud Native
- Kubernetes deployment
- Managed databases
- Auto-scaling groups
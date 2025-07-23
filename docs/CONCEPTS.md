# Graphiti Core Concepts

## Introduction

This document explains the fundamental concepts that make Graphiti unique and powerful for building AI agent memory systems.

## Temporal Knowledge Graphs

### What Makes It Temporal?

Traditional knowledge graphs capture relationships between entities but often struggle with time-varying information. Graphiti's temporal knowledge graph explicitly tracks:

1. **When facts became true** (valid_at)
2. **When we learned about them** (created_at)
3. **When facts became invalid** (invalid_at)

This enables queries like:
- "What was the user's shipping address in January?"
- "When did the customer's preferences change?"
- "What did we know about the project status last week?"

### Bi-Temporal Model

Graphiti implements a sophisticated bi-temporal model inspired by temporal databases:

```
Timeline T (Event Time):     [----A----][----B----][----C----]
Timeline T' (System Time):        [--A--]  [--B--]    [--C--]
```

- **T**: When events actually happened
- **T'**: When we recorded them in the system

This distinction is crucial for:
- Backdating information ("We just learned that X happened last month")
- Audit trails ("When did we first know about Y?")
- Time-travel queries ("Show me the graph as it was on date Z")

## Episodes: The Foundation

### What Are Episodes?

Episodes are the atomic units of information in Graphiti. Think of them as:
- Messages in a conversation
- Entries in a journal
- Events in a timeline
- Records in a database

### Episode Types

1. **Text Episodes**: Natural language content
   ```python
   await graphiti.add_episode(
       episode_body="Alice met Bob at the conference",
       source=EpisodeType.text
   )
   ```

2. **JSON Episodes**: Structured data
   ```python
   await graphiti.add_episode(
       episode_body=json.dumps({"user_id": 123, "action": "purchase"}),
       source=EpisodeType.json
   )
   ```

### Episode Windows

Graphiti maintains context by retrieving relevant previous episodes within a time window (default: 10 hours). This ensures:
- Continuity in entity extraction
- Consistent relationship identification
- Temporal coherence

## Entities and Relationships

### Entity Extraction

When an episode is added, Graphiti:
1. Analyzes the content using LLMs
2. Identifies entities (people, places, concepts)
3. Extracts their attributes
4. Creates or updates entity nodes

### Entity Resolution

Graphiti automatically handles entity deduplication:
- "Robert" and "Bob" can be the same person
- "Apple Inc." and "Apple" can be the same company
- Context determines resolution

### Relationship Extraction

Relationships (edges) contain:
- **Fact**: The relationship description
- **Fact Embedding**: Vector representation
- **Temporal Metadata**: When it's valid
- **Episode References**: Source episodes

### Edge Invalidation

Instead of deleting outdated relationships, Graphiti invalidates them:
```
Before: "Alice works at TechCorp" (valid_at: 2023-01-01)
After:  "Alice works at TechCorp" (valid_at: 2023-01-01, invalid_at: 2024-01-01)
New:    "Alice works at StartupXYZ" (valid_at: 2024-01-01)
```

## Communities

### What Are Communities?

Communities are clusters of highly interconnected entities that represent:
- Teams or departments
- Product categories
- Topic clusters
- Social groups

### Community Detection

Graphiti uses algorithms to:
1. Identify dense subgraphs
2. Generate community summaries
3. Create hierarchical structures
4. Update as the graph evolves

### Benefits

- **Efficient Search**: Search within relevant communities
- **Summarization**: High-level overviews
- **Organization**: Natural grouping of information

## Search Strategies

### Hybrid Search

Graphiti combines multiple search methods:

1. **Semantic Search**: Vector similarity using embeddings
2. **Keyword Search**: BM25 fulltext matching
3. **Graph Search**: BFS traversal from anchor nodes

### Search Configuration

Pre-built configurations optimize for different use cases:

```python
# For general queries
COMBINED_HYBRID_SEARCH_CROSS_ENCODER

# For relationship-focused queries
EDGE_HYBRID_SEARCH_RRF

# For entity-focused queries
NODE_HYBRID_SEARCH_MMR
```

### Reranking Strategies

1. **RRF (Reciprocal Rank Fusion)**: Combines multiple rankings
2. **Node Distance**: Prioritizes graph proximity
3. **Episode Mentions**: Weights by mention frequency
4. **MMR (Maximal Marginal Relevance)**: Balances relevance and diversity
5. **Cross-Encoder**: Neural reranking for precision

## Group Management

### What Are Groups?

Groups provide logical isolation of data:
- Different users
- Separate conversations
- Distinct projects
- Multi-tenant scenarios

### Group IDs

Every episode belongs to a group:
```python
await graphiti.add_episode(
    episode_body="content",
    group_id="user_123_conversation_456"
)
```

### Benefits

- **Data Isolation**: Keep unrelated data separate
- **Efficient Queries**: Search within specific groups
- **Privacy**: User data separation
- **Context Management**: Conversation boundaries

## Custom Entities

### Defining Custom Types

Graphiti allows domain-specific entity definitions:

```python
from pydantic import BaseModel

class MedicalCondition(BaseModel):
    name: str
    icd_code: str
    severity: str
    
class Medication(BaseModel):
    name: str
    dosage: str
    frequency: str

await graphiti.add_episode(
    episode_body="Patient diagnosed with Type 2 Diabetes (E11.9), prescribed Metformin 500mg twice daily",
    entity_types={
        'MedicalCondition': MedicalCondition,
        'Medication': Medication
    }
)
```

### Benefits

- **Domain Specificity**: Tailored to your use case
- **Type Safety**: Pydantic validation
- **Rich Attributes**: Beyond simple properties
- **Consistent Schema**: Enforced structure

## Memory Persistence

### Why Not Just Context Windows?

Traditional LLMs use context windows that:
- Forget everything outside the window
- Can't selectively recall information
- Don't understand temporal relationships
- Treat all information equally

### Graphiti's Approach

- **Selective Recall**: Retrieve only relevant information
- **Temporal Awareness**: Understand when things happened
- **Relationship Understanding**: Know how things connect
- **Incremental Learning**: Build knowledge over time

## Performance Characteristics

### Query Performance

- **P95 Latency**: ~300ms for complex queries
- **Throughput**: Thousands of queries/second
- **Scalability**: Linear with graph size

### Ingestion Performance

- **Episode Processing**: ~100ms per episode
- **Batch Operations**: 10x faster for bulk ingestion
- **Concurrent Processing**: Parallel extraction

### Memory Efficiency

- **Incremental Updates**: No full recomputation
- **Selective Loading**: Load only needed subgraphs
- **Embedding Cache**: Reuse computed vectors

## Use Case Patterns

### Conversational AI

```python
# Track conversation history
await graphiti.add_episode(
    episode_body=f"User: {user_message}\nAssistant: {response}",
    group_id=conversation_id
)

# Retrieve context for next turn
context = await graphiti.search(
    query=new_user_message,
    group_ids=[conversation_id]
)
```

### Customer Support

```python
# Log support interactions
await graphiti.add_episode(
    episode_body=json.dumps({
        "ticket_id": ticket_id,
        "issue": issue_description,
        "resolution": resolution
    }),
    source=EpisodeType.json
)

# Find similar issues
similar = await graphiti.search(
    query=new_issue,
    num_results=5
)
```

### Personal Assistant

```python
# Remember user preferences
await graphiti.add_episode(
    episode_body="User prefers morning meetings and likes coffee",
    group_id=user_id
)

# Personalize responses
preferences = await graphiti.search(
    query="user preferences",
    group_ids=[user_id]
)
```

## Best Practices

### Episode Design

1. **Atomic Information**: One concept per episode
2. **Clear Timestamps**: Accurate temporal data
3. **Consistent Format**: Standardize episode structure
4. **Meaningful Names**: Descriptive episode names

### Entity Modeling

1. **Start Simple**: Use default extraction first
2. **Add Custom Types**: As patterns emerge
3. **Validate Schemas**: Test entity definitions
4. **Document Types**: Maintain entity documentation

### Search Optimization

1. **Use Appropriate Configs**: Match search to use case
2. **Leverage Center Nodes**: For graph-aware search
3. **Filter by Groups**: Reduce search space
4. **Tune Rerankers**: Based on result quality

### Performance Tuning

1. **Batch Operations**: For bulk ingestion
2. **Async Processing**: Maximize concurrency
3. **Index Management**: Keep indexes updated
4. **Monitor Latency**: Track performance metrics
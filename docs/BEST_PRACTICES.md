# Graphiti Best Practices & Design Patterns

## Table of Contents

1. [Episode Design Best Practices](#episode-design-best-practices)
2. [Entity Modeling Patterns](#entity-modeling-patterns)
3. [Search Optimization](#search-optimization)
4. [Performance Best Practices](#performance-best-practices)
5. [Temporal Data Management](#temporal-data-management)
6. [Production Deployment](#production-deployment)
7. [Testing Strategies](#testing-strategies)
8. [Common Anti-Patterns](#common-anti-patterns)

## Episode Design Best Practices

### 1. Atomic Episodes

**Good Practice:** Keep episodes focused on single events or concepts

```python
# ✅ Good: Atomic episodes
await graphiti.add_episode(
    name="Order Placed",
    episode_body="Customer John Doe placed order #12345 for $99.99",
    source=EpisodeType.text,
    reference_time=order_timestamp
)

await graphiti.add_episode(
    name="Payment Processed",
    episode_body="Payment of $99.99 processed for order #12345 via credit card",
    source=EpisodeType.text,
    reference_time=payment_timestamp
)

# ❌ Bad: Combined episode
await graphiti.add_episode(
    name="Order and Payment",
    episode_body="John placed order #12345 for $99.99 and payment was processed",
    source=EpisodeType.text
)
```

### 2. Consistent Episode Naming

**Good Practice:** Use descriptive, consistent naming conventions

```python
# ✅ Good: Consistent pattern
episode_patterns = {
    "user_action": "User Action: {action_type}",
    "system_event": "System: {event_type}",
    "conversation": "Chat: {participant} - {timestamp}",
    "document": "Document: {doc_type} - {doc_name}"
}

await graphiti.add_episode(
    name=episode_patterns["user_action"].format(action_type="Login"),
    episode_body="User john@example.com logged in from IP 192.168.1.1",
    source=EpisodeType.text
)
```

### 3. Structured vs Unstructured Data

**Good Practice:** Choose the right episode type for your data

```python
# ✅ For structured data, use JSON episodes
transaction_data = {
    "transaction_id": "TXN-12345",
    "amount": 150.00,
    "currency": "USD",
    "merchant": "Coffee Shop",
    "category": "Food & Beverage",
    "timestamp": "2024-01-15T10:30:00Z"
}

await graphiti.add_episode(
    name="Transaction Record",
    episode_body=json.dumps(transaction_data),
    source=EpisodeType.json
)

# ✅ For natural language, use text episodes
await graphiti.add_episode(
    name="Customer Feedback",
    episode_body="The coffee was excellent but the service was slow",
    source=EpisodeType.text
)
```

### 4. Episode Context Management

**Good Practice:** Link related episodes for better context

```python
class ConversationManager:
    def __init__(self, graphiti, conversation_id):
        self.graphiti = graphiti
        self.conversation_id = conversation_id
        self.episode_chain = []
    
    async def add_turn(self, speaker, message):
        # Get recent context
        if self.episode_chain:
            previous_uuids = self.episode_chain[-3:]  # Last 3 turns
        else:
            previous_uuids = []
        
        result = await self.graphiti.add_episode(
            name=f"{speaker} at {datetime.now().strftime('%H:%M')}",
            episode_body=f"{speaker}: {message}",
            source=EpisodeType.text,
            group_id=self.conversation_id,
            previous_episode_uuids=previous_uuids
        )
        
        self.episode_chain.append(result.episode.uuid)
        return result
```

## Entity Modeling Patterns

### 1. Start Simple, Evolve Gradually

**Good Practice:** Begin with default extraction, then add custom types

```python
# Phase 1: Use default extraction
await graphiti.add_episode(
    episode_body="Sarah from Engineering is working on the API redesign",
    source=EpisodeType.text
)

# Phase 2: Add basic custom types
class Employee(BaseModel):
    name: str
    department: Optional[str] = None

await graphiti.add_episode(
    episode_body="Sarah from Engineering is working on the API redesign",
    source=EpisodeType.text,
    entity_types={'Employee': Employee}
)

# Phase 3: Enhance with more details
class Employee(BaseModel):
    name: str
    department: Optional[str] = None
    role: Optional[str] = None
    skills: Optional[List[str]] = None
    projects: Optional[List[str]] = None
```

### 2. Domain-Specific Entity Hierarchies

**Good Practice:** Model entities that reflect your domain

```python
# E-commerce domain
class Customer(BaseModel):
    name: str
    email: Optional[str] = None
    tier: Optional[str] = None  # "Bronze", "Silver", "Gold"

class Product(BaseModel):
    name: str
    sku: str
    category: str
    price: Optional[float] = None

class Order(BaseModel):
    order_id: str
    status: str
    total: float
    items_count: int

# Healthcare domain
class Patient(BaseModel):
    name: str
    mrn: str
    age: Optional[int] = None
    
class Provider(BaseModel):
    name: str
    specialty: str
    npi: Optional[str] = None

class Condition(BaseModel):
    name: str
    icd_code: str
    severity: Optional[str] = None
```

### 3. Relationship Type Design

**Good Practice:** Define meaningful relationship types

```python
# Define relationship types with context
class WorksOn(BaseModel):
    role: str
    start_date: Optional[date] = None
    allocation: Optional[int] = None  # Percentage

class ReportsTo(BaseModel):
    direct: bool = True
    department: Optional[str] = None

class Purchased(BaseModel):
    quantity: int
    price: float
    discount: Optional[float] = None

# Use in extraction
await graphiti.add_episode(
    episode_body="Sarah, our Senior Engineer, started working on Project Phoenix last Monday with 80% allocation",
    entity_types={
        'Employee': Employee,
        'Project': Project
    },
    edge_types={
        'WORKS_ON': WorksOn
    }
)
```

## Search Optimization

### 1. Choose the Right Search Strategy

**Good Practice:** Match search configuration to query type

```python
from graphiti_core.search.search_config_recipes import (
    NODE_HYBRID_SEARCH_RRF,
    EDGE_HYBRID_SEARCH_NODE_DISTANCE,
    COMBINED_HYBRID_SEARCH_CROSS_ENCODER
)

class SmartSearcher:
    def __init__(self, graphiti):
        self.graphiti = graphiti
    
    async def search(self, query, search_type="auto"):
        # Detect query intent
        if search_type == "auto":
            if any(word in query.lower() for word in ["who", "person", "people", "team"]):
                search_type = "entities"
            elif any(word in query.lower() for word in ["relationship", "works with", "reports to"]):
                search_type = "relationships"
            else:
                search_type = "comprehensive"
        
        # Apply appropriate config
        configs = {
            "entities": NODE_HYBRID_SEARCH_RRF,
            "relationships": EDGE_HYBRID_SEARCH_NODE_DISTANCE,
            "comprehensive": COMBINED_HYBRID_SEARCH_CROSS_ENCODER
        }
        
        return await self.graphiti._search(
            query=query,
            config=configs.get(search_type, configs["comprehensive"])
        )
```

### 2. Leverage Graph Context

**Good Practice:** Use center nodes for contextual search

```python
async def find_related_information(graphiti, entity_name, topic):
    # First, find the entity
    entity_results = await graphiti.search(entity_name, num_results=1)
    
    if not entity_results.nodes:
        return None
    
    center_node = entity_results.nodes[0]
    
    # Search for topic around this entity
    related_results = await graphiti.search(
        query=topic,
        center_node_uuid=center_node.uuid,
        num_results=20
    )
    
    # Group results by type
    grouped = {
        "direct_relationships": [],
        "related_entities": [],
        "relevant_episodes": []
    }
    
    for edge in related_results.edges:
        if center_node.uuid in [edge.source_node_uuid, edge.target_node_uuid]:
            grouped["direct_relationships"].append(edge)
        else:
            grouped["related_entities"].append(edge)
    
    grouped["relevant_episodes"] = related_results.episodes
    
    return grouped
```

### 3. Search Result Caching

**Good Practice:** Cache frequently accessed queries

```python
from functools import lru_cache
from datetime import datetime, timedelta
import hashlib

class CachedGraphitiSearch:
    def __init__(self, graphiti, cache_duration=timedelta(minutes=15)):
        self.graphiti = graphiti
        self.cache = {}
        self.cache_duration = cache_duration
    
    def _cache_key(self, query, **kwargs):
        # Create unique key from query and parameters
        key_data = f"{query}:{sorted(kwargs.items())}"
        return hashlib.md5(key_data.encode()).hexdigest()
    
    async def search(self, query, **kwargs):
        cache_key = self._cache_key(query, **kwargs)
        
        # Check cache
        if cache_key in self.cache:
            cached_result, timestamp = self.cache[cache_key]
            if datetime.now() - timestamp < self.cache_duration:
                return cached_result
        
        # Perform search
        result = await self.graphiti.search(query, **kwargs)
        
        # Cache result
        self.cache[cache_key] = (result, datetime.now())
        
        return result
```

## Performance Best Practices

### 1. Batch Processing

**Good Practice:** Process multiple items together

```python
from graphiti_core.utils.bulk_utils import RawEpisode
import asyncio

class BatchProcessor:
    def __init__(self, graphiti, batch_size=50):
        self.graphiti = graphiti
        self.batch_size = batch_size
    
    async def process_documents(self, documents):
        batches = []
        
        for i in range(0, len(documents), self.batch_size):
            batch = documents[i:i + self.batch_size]
            episodes = [
                RawEpisode(
                    name=doc['title'],
                    content=doc['content'],
                    source_type=EpisodeType.text,
                    reference_time=doc.get('timestamp', datetime.now(timezone.utc))
                )
                for doc in batch
            ]
            batches.append(episodes)
        
        # Process batches concurrently
        tasks = [
            self.graphiti.add_episodes_bulk(batch)
            for batch in batches
        ]
        
        results = await asyncio.gather(*tasks)
        return [item for sublist in results for item in sublist]
```

### 2. Connection Pooling

**Good Practice:** Reuse database connections

```python
class GraphitiPool:
    def __init__(self, config, pool_size=5):
        self.config = config
        self.pool = []
        self.available = asyncio.Queue(maxsize=pool_size)
        self.pool_size = pool_size
    
    async def initialize(self):
        for _ in range(self.pool_size):
            graphiti = Graphiti(**self.config)
            await graphiti.build_indices_and_constraints()
            self.pool.append(graphiti)
            await self.available.put(graphiti)
    
    async def acquire(self):
        return await self.available.get()
    
    async def release(self, graphiti):
        await self.available.put(graphiti)
    
    async def close_all(self):
        for graphiti in self.pool:
            await graphiti.driver.close()

# Usage
pool = GraphitiPool({
    "uri": "bolt://localhost:7687",
    "user": "neo4j",
    "password": "password"
})

await pool.initialize()

# Use in request handler
async def handle_request(data):
    graphiti = await pool.acquire()
    try:
        result = await graphiti.add_episode(**data)
        return result
    finally:
        await pool.release(graphiti)
```

### 3. Optimize Episode Size

**Good Practice:** Keep episodes reasonably sized

```python
class EpisodeChunker:
    def __init__(self, max_size=1000, overlap=100):
        self.max_size = max_size
        self.overlap = overlap
    
    def chunk_text(self, text, metadata=None):
        chunks = []
        sentences = text.split('. ')
        
        current_chunk = []
        current_size = 0
        
        for sentence in sentences:
            sentence_size = len(sentence)
            
            if current_size + sentence_size > self.max_size and current_chunk:
                # Create chunk
                chunk_text = '. '.join(current_chunk) + '.'
                chunks.append({
                    'text': chunk_text,
                    'metadata': metadata,
                    'chunk_index': len(chunks)
                })
                
                # Keep overlap
                overlap_sentences = current_chunk[-2:] if len(current_chunk) > 2 else current_chunk
                current_chunk = overlap_sentences + [sentence]
                current_size = sum(len(s) for s in current_chunk)
            else:
                current_chunk.append(sentence)
                current_size += sentence_size
        
        # Add final chunk
        if current_chunk:
            chunks.append({
                'text': '. '.join(current_chunk) + '.',
                'metadata': metadata,
                'chunk_index': len(chunks)
            })
        
        return chunks
```

## Temporal Data Management

### 1. Point-in-Time Queries

**Good Practice:** Implement temporal query patterns

```python
class TemporalQuerier:
    def __init__(self, graphiti):
        self.graphiti = graphiti
    
    async def get_state_at_time(self, entity_name, target_time):
        # Find the entity
        entity_results = await self.graphiti.search(entity_name, num_results=1)
        if not entity_results.nodes:
            return None
        
        entity = entity_results.nodes[0]
        
        # Get all edges for this entity
        all_edges = await self.graphiti.search(
            query="",
            center_node_uuid=entity.uuid,
            num_results=100
        )
        
        # Filter by temporal validity
        valid_edges = []
        for edge in all_edges.edges:
            if (edge.valid_at <= target_time and 
                (edge.invalid_at is None or edge.invalid_at > target_time)):
                valid_edges.append(edge)
        
        return {
            "entity": entity,
            "relationships": valid_edges,
            "as_of": target_time
        }
    
    async def get_history(self, entity_name, start_time, end_time):
        # Similar pattern but collect all changes in time range
        pass
```

### 2. Temporal Integrity

**Good Practice:** Maintain temporal consistency

```python
class TemporalEpisodeManager:
    def __init__(self, graphiti):
        self.graphiti = graphiti
    
    async def add_backdated_episode(self, episode_body, event_time, notes=None):
        """Add historical information with proper temporal context"""
        
        # Add the historical event
        result = await self.graphiti.add_episode(
            name=f"Historical: {notes or 'Backdated Entry'}",
            episode_body=episode_body,
            source=EpisodeType.text,
            reference_time=event_time,  # When it happened
            # created_at will be now() automatically
        )
        
        # Optionally add audit note
        if notes:
            await self.graphiti.add_episode(
                name="Audit: Historical Data Added",
                episode_body=f"Added historical data: {notes}. Event time: {event_time}",
                source=EpisodeType.text,
                reference_time=datetime.now(timezone.utc)
            )
        
        return result
```

## Production Deployment

### 1. Health Checks

**Good Practice:** Implement comprehensive health checks

```python
class GraphitiHealthCheck:
    def __init__(self, graphiti):
        self.graphiti = graphiti
    
    async def check_health(self):
        health_status = {
            "status": "healthy",
            "checks": {},
            "timestamp": datetime.now(timezone.utc)
        }
        
        # Check database connection
        try:
            await self.graphiti.driver.validate_connection()
            health_status["checks"]["database"] = "ok"
        except Exception as e:
            health_status["status"] = "unhealthy"
            health_status["checks"]["database"] = f"error: {str(e)}"
        
        # Check LLM availability
        try:
            test_result = await self.graphiti.add_episode(
                name="Health Check",
                episode_body="System health check",
                source=EpisodeType.text
            )
            health_status["checks"]["llm"] = "ok"
            # Clean up test episode
            await self._cleanup_test_episode(test_result.episode.uuid)
        except Exception as e:
            health_status["status"] = "unhealthy"
            health_status["checks"]["llm"] = f"error: {str(e)}"
        
        # Check search functionality
        try:
            await self.graphiti.search("test", num_results=1)
            health_status["checks"]["search"] = "ok"
        except Exception as e:
            health_status["status"] = "unhealthy"
            health_status["checks"]["search"] = f"error: {str(e)}"
        
        return health_status
```

### 2. Monitoring & Metrics

**Good Practice:** Track key metrics

```python
from datetime import datetime
import time
from collections import defaultdict

class GraphitiMetrics:
    def __init__(self):
        self.metrics = defaultdict(list)
    
    async def track_operation(self, operation_name, func, *args, **kwargs):
        start_time = time.time()
        error = None
        
        try:
            result = await func(*args, **kwargs)
            success = True
        except Exception as e:
            error = str(e)
            success = False
            raise
        finally:
            duration = time.time() - start_time
            
            self.metrics[operation_name].append({
                "timestamp": datetime.now(timezone.utc),
                "duration": duration,
                "success": success,
                "error": error
            })
        
        return result
    
    def get_stats(self, operation_name, time_window=None):
        data = self.metrics[operation_name]
        
        if time_window:
            cutoff = datetime.now(timezone.utc) - time_window
            data = [m for m in data if m["timestamp"] > cutoff]
        
        if not data:
            return None
        
        durations = [m["duration"] for m in data if m["success"]]
        success_count = sum(1 for m in data if m["success"])
        
        return {
            "count": len(data),
            "success_rate": success_count / len(data),
            "avg_duration": sum(durations) / len(durations) if durations else 0,
            "p95_duration": sorted(durations)[int(len(durations) * 0.95)] if durations else 0,
            "errors": [m["error"] for m in data if m["error"]]
        }
```

### 3. Backup & Recovery

**Good Practice:** Implement data backup strategies

```python
class GraphitiBackup:
    def __init__(self, graphiti):
        self.graphiti = graphiti
    
    async def export_group_data(self, group_id, output_file):
        """Export all data for a specific group"""
        
        # Get all episodes
        episodes = await self.graphiti.retrieve_episodes(
            reference_time=datetime.now(timezone.utc),
            num_episodes=10000,  # Large number
            group_ids=[group_id]
        )
        
        # Get all entities and relationships via search
        all_data = await self.graphiti.search(
            query="",  # Match all
            group_ids=[group_id],
            num_results=10000
        )
        
        export_data = {
            "export_timestamp": datetime.now(timezone.utc).isoformat(),
            "group_id": group_id,
            "episodes": [
                {
                    "uuid": ep.uuid,
                    "name": ep.name,
                    "content": ep.content,
                    "created_at": ep.created_at.isoformat(),
                    "valid_at": ep.valid_at.isoformat() if ep.valid_at else None
                }
                for ep in episodes
            ],
            "entities": [
                {
                    "uuid": node.uuid,
                    "name": node.name,
                    "type": node.labels[0] if node.labels else "Entity",
                    "summary": node.summary
                }
                for node in all_data.nodes
            ],
            "relationships": [
                {
                    "uuid": edge.uuid,
                    "fact": edge.fact,
                    "source": edge.source_node_uuid,
                    "target": edge.target_node_uuid,
                    "valid_at": edge.valid_at.isoformat() if edge.valid_at else None,
                    "invalid_at": edge.invalid_at.isoformat() if edge.invalid_at else None
                }
                for edge in all_data.edges
            ]
        }
        
        with open(output_file, 'w') as f:
            json.dump(export_data, f, indent=2)
        
        return len(export_data["episodes"])
```

## Testing Strategies

### 1. Unit Testing Patterns

**Good Practice:** Test entity extraction and search

```python
import pytest
from datetime import datetime, timezone

@pytest.mark.asyncio
async def test_entity_extraction(graphiti_client):
    # Test data
    episode_body = "Alice Cooper, the CEO of TechCorp, announced a new product launch"
    
    # Add episode
    result = await graphiti_client.add_episode(
        name="Test Announcement",
        episode_body=episode_body,
        source=EpisodeType.text,
        reference_time=datetime.now(timezone.utc)
    )
    
    # Verify entities extracted
    entity_names = [node.name for node in result.nodes]
    assert "Alice Cooper" in entity_names
    assert "TechCorp" in entity_names
    
    # Verify relationships
    relationship_facts = [edge.fact for edge in result.edges]
    assert any("CEO" in fact for fact in relationship_facts)

@pytest.mark.asyncio
async def test_temporal_search(graphiti_client):
    # Add episodes at different times
    past_time = datetime(2023, 1, 1, tzinfo=timezone.utc)
    recent_time = datetime(2024, 1, 1, tzinfo=timezone.utc)
    
    await graphiti_client.add_episode(
        episode_body="Old project status: planning phase",
        reference_time=past_time
    )
    
    await graphiti_client.add_episode(
        episode_body="Current project status: implementation phase",
        reference_time=recent_time
    )
    
    # Search should return most relevant
    results = await graphiti_client.search("project status")
    
    # Verify temporal ordering
    assert len(results.edges) >= 2
    # Most recent should typically rank higher
```

### 2. Integration Testing

**Good Practice:** Test end-to-end workflows

```python
@pytest.mark.asyncio
async def test_conversation_workflow(graphiti_client):
    conversation_id = "test_conv_123"
    
    # Simulate conversation
    messages = [
        ("User", "I need help with my order #12345"),
        ("Support", "I'll help you with that order. Let me check."),
        ("Support", "Your order was shipped yesterday and should arrive tomorrow."),
        ("User", "Great, thank you!")
    ]
    
    episode_uuids = []
    
    for speaker, message in messages:
        result = await graphiti_client.add_episode(
            name=f"{speaker} message",
            episode_body=f"{speaker}: {message}",
            source=EpisodeType.text,
            group_id=conversation_id,
            previous_episode_uuids=episode_uuids[-2:] if episode_uuids else []
        )
        episode_uuids.append(result.episode.uuid)
    
    # Verify conversation can be retrieved
    search_results = await graphiti_client.search(
        "order #12345",
        group_ids=[conversation_id]
    )
    
    assert len(search_results.episodes) >= 1
    assert any("shipped yesterday" in ep.content for ep in search_results.episodes)
```

## Common Anti-Patterns

### 1. Over-Engineering Entity Types

**❌ Anti-pattern:**
```python
# Too complex from the start
class Person(BaseModel):
    first_name: str
    middle_name: Optional[str]
    last_name: str
    nicknames: List[str]
    date_of_birth: Optional[date]
    ssn: Optional[str]
    addresses: List[Address]
    phone_numbers: List[PhoneNumber]
    # ... 20 more fields
```

**✅ Better approach:**
```python
# Start simple
class Person(BaseModel):
    name: str
    email: Optional[str] = None
    role: Optional[str] = None

# Evolve as needed
class PersonV2(Person):
    department: Optional[str] = None
    manager: Optional[str] = None
```

### 2. Ignoring Temporal Aspects

**❌ Anti-pattern:**
```python
# Overwriting without history
await graphiti.add_episode(
    episode_body="John's address is 123 Main St"
)
# Later...
await graphiti.add_episode(
    episode_body="John's address is 456 Oak Ave"
)
# Lost the history!
```

**✅ Better approach:**
```python
# Preserve temporal information
await graphiti.add_episode(
    episode_body="As of January 2024, John's address is 123 Main St",
    reference_time=datetime(2024, 1, 1, tzinfo=timezone.utc)
)

await graphiti.add_episode(
    episode_body="John moved to 456 Oak Ave in March 2024",
    reference_time=datetime(2024, 3, 1, tzinfo=timezone.utc)
)
```

### 3. Unbounded Searches

**❌ Anti-pattern:**
```python
# Searching everything
all_results = await graphiti.search("", num_results=10000)
```

**✅ Better approach:**
```python
# Scoped, paginated searches
async def paginated_search(query, group_id, page_size=50):
    results = await graphiti.search(
        query=query,
        group_ids=[group_id],
        num_results=page_size
    )
    return results
```

### 4. Synchronous Thinking

**❌ Anti-pattern:**
```python
# Sequential processing
for document in documents:
    await graphiti.add_episode(episode_body=document)
```

**✅ Better approach:**
```python
# Concurrent processing
import asyncio

tasks = [
    graphiti.add_episode(episode_body=doc)
    for doc in documents
]
results = await asyncio.gather(*tasks)
```

## Summary

Following these best practices will help you:

1. Build more maintainable and scalable knowledge graphs
2. Achieve better search relevance and performance
3. Properly handle temporal data and relationships
4. Deploy Graphiti successfully in production
5. Avoid common pitfalls and anti-patterns

Remember: Start simple, measure performance, and evolve your implementation based on real usage patterns.
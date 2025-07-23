# Graphiti API Guide

## Table of Contents

1. [Installation & Setup](#installation--setup)
2. [Basic Operations](#basic-operations)
3. [Episode Management](#episode-management)
4. [Search Operations](#search-operations)
5. [Custom Entities](#custom-entities)
6. [Advanced Features](#advanced-features)
7. [Error Handling](#error-handling)
8. [Performance Tips](#performance-tips)

## Installation & Setup

### Basic Installation

```bash
# With pip
pip install graphiti-core

# With uv
uv add graphiti-core

# With optional providers
pip install graphiti-core[anthropic,groq,google-genai]

# With FalkorDB support
pip install graphiti-core[falkordb]
```

### Basic Setup with Neo4j

```python
from graphiti_core import Graphiti
from graphiti_core.nodes import EpisodeType
import asyncio
from datetime import datetime, timezone

async def main():
    # Initialize Graphiti with Neo4j
    graphiti = Graphiti(
        uri="bolt://localhost:7687",
        user="neo4j",
        password="password"
    )
    
    # Build indices (run once)
    await graphiti.build_indices_and_constraints()
    
    # Your code here
    
    # Always close when done
    await graphiti.driver.close()

# Run the async function
asyncio.run(main())
```

### Setup with Custom Providers

```python
from graphiti_core import Graphiti
from graphiti_core.llm_client import AnthropicClient, LLMConfig
from graphiti_core.embedder import VoyageEmbedder, VoyageEmbedderConfig

# Configure custom LLM
llm_client = AnthropicClient(
    config=LLMConfig(
        api_key="your-anthropic-key",
        model="claude-3-sonnet-20240229"
    )
)

# Configure custom embedder
embedder = VoyageEmbedder(
    config=VoyageEmbedderConfig(
        api_key="your-voyage-key",
        model="voyage-2"
    )
)

# Initialize with custom providers
graphiti = Graphiti(
    uri="bolt://localhost:7687",
    user="neo4j",
    password="password",
    llm_client=llm_client,
    embedder=embedder
)
```

## Basic Operations

### Adding Your First Episode

```python
# Add a simple text episode
result = await graphiti.add_episode(
    name="First Meeting",
    episode_body="Alice met Bob at the tech conference. They discussed AI and startups.",
    source=EpisodeType.text,
    reference_time=datetime.now(timezone.utc)
)

print(f"Episode ID: {result.episode.uuid}")
print(f"Extracted Entities: {[node.name for node in result.nodes]}")
print(f"Extracted Relationships: {[edge.fact for edge in result.edges]}")
```

### Basic Search

```python
# Search for information
search_results = await graphiti.search(
    query="Who did Alice meet?",
    num_results=5
)

for i, result in enumerate(search_results.edges):
    print(f"{i+1}. {result.fact} (score: {result.score:.3f})")
```

## Episode Management

### Text Episodes

```python
# Conversation episode
await graphiti.add_episode(
    name="Customer Support Chat",
    episode_body="""
    Customer: My order hasn't arrived yet. Order #12345.
    Support: I apologize for the delay. Let me check that for you.
    Support: Your order was shipped on Monday and should arrive by Friday.
    Customer: Thank you for checking!
    """,
    source=EpisodeType.text,
    group_id="support_chat_001"
)

# Meeting notes
await graphiti.add_episode(
    name="Q4 Planning Meeting",
    episode_body="""
    Attendees: Sarah (PM), Mike (Eng), Lisa (Design)
    Topics discussed:
    - Q4 roadmap priorities
    - Resource allocation
    - Launch timeline for Project Phoenix
    Decisions:
    - Mike will lead the backend team
    - Lisa will create mockups by next week
    - Launch target: December 15th
    """,
    source=EpisodeType.text,
    group_id="team_meetings"
)
```

### JSON Episodes

```python
import json

# E-commerce order
order_data = {
    "order_id": "ORD-12345",
    "customer": {
        "name": "John Doe",
        "email": "john@example.com",
        "tier": "Premium"
    },
    "items": [
        {"product": "Laptop", "price": 1299.99, "quantity": 1},
        {"product": "Mouse", "price": 79.99, "quantity": 2}
    ],
    "total": 1459.97,
    "status": "shipped"
}

await graphiti.add_episode(
    name="Order Placed",
    episode_body=json.dumps(order_data),
    source=EpisodeType.json,
    group_id="orders"
)

# User activity log
activity_data = {
    "user_id": "USR-789",
    "action": "viewed_product",
    "product_id": "PROD-456",
    "category": "Electronics",
    "timestamp": "2024-01-15T10:30:00Z",
    "session_id": "SESSION-123"
}

await graphiti.add_episode(
    name="Product View Event",
    episode_body=json.dumps(activity_data),
    source=EpisodeType.json,
    group_id="user_activity"
)
```

### Episode Context

```python
# Retrieve previous episodes for context
previous_episodes = await graphiti.retrieve_episodes(
    reference_time=datetime.now(timezone.utc),
    num_episodes=5,
    group_ids=["conversation_123"]
)

# Add new episode with context
await graphiti.add_episode(
    name="Follow-up Message",
    episode_body="Actually, I changed my mind about the meeting time.",
    source=EpisodeType.text,
    reference_time=datetime.now(timezone.utc),
    group_id="conversation_123",
    previous_episode_uuids=[ep.uuid for ep in previous_episodes]
)
```

## Search Operations

### Basic Search Patterns

```python
# Simple query search
results = await graphiti.search("What products did John order?")

# Search with group filtering
results = await graphiti.search(
    query="customer complaints",
    group_ids=["support_tickets"],
    num_results=10
)

# Search with custom configuration
from graphiti_core.search.search_config_recipes import EDGE_HYBRID_SEARCH_RRF

results = await graphiti._search(
    query="pricing discussions",
    config=EDGE_HYBRID_SEARCH_RRF,
    num_results=20
)
```

### Graph-Aware Search

```python
# First, find a specific entity
node_results = await graphiti.search("John Doe")
if node_results.nodes:
    john_node = node_results.nodes[0]
    
    # Search centered around this node
    centered_results = await graphiti.search(
        query="recent orders",
        center_node_uuid=john_node.uuid,
        num_results=10
    )
    
    print(f"Orders related to {john_node.name}:")
    for edge in centered_results.edges:
        print(f"- {edge.fact}")
```

### Time-Based Search

```python
from datetime import timedelta

# Search with time constraints
now = datetime.now(timezone.utc)
last_week = now - timedelta(days=7)

# Manual time filtering
all_results = await graphiti.search("status updates")
recent_results = [
    r for r in all_results.edges 
    if r.created_at >= last_week
]
```

### Search Results Handling

```python
# Comprehensive result handling
results = await graphiti.search("AI project updates")

# Handle different result types
print(f"\nFound {len(results.nodes)} entities:")
for node in results.nodes:
    print(f"- {node.name}: {node.summary}")

print(f"\nFound {len(results.edges)} relationships:")
for edge in results.edges:
    print(f"- {edge.fact} (confidence: {edge.score:.2f})")

print(f"\nFound {len(results.episodes)} related episodes:")
for episode in results.episodes:
    print(f"- {episode.name}: {episode.content[:100]}...")

print(f"\nFound {len(results.communities)} communities:")
for community in results.communities:
    print(f"- {community.name}: {community.summary}")
```

## Custom Entities

### Defining Custom Entity Types

```python
from pydantic import BaseModel
from typing import Optional, List
from datetime import date

# Define custom entity models
class Person(BaseModel):
    first_name: str
    last_name: str
    role: Optional[str] = None
    department: Optional[str] = None
    email: Optional[str] = None

class Project(BaseModel):
    name: str
    status: str
    priority: str
    deadline: Optional[date] = None
    budget: Optional[float] = None

class Task(BaseModel):
    title: str
    status: str
    assignee: Optional[str] = None
    due_date: Optional[date] = None
    story_points: Optional[int] = None

# Define custom relationship types
class WorksOn(BaseModel):
    role: str
    allocation_percentage: Optional[int] = None

class ReportsTo(BaseModel):
    since: Optional[date] = None

# Use custom entities
await graphiti.add_episode(
    name="Project Kickoff",
    episode_body="""
    Sarah Johnson, our Senior PM, is starting Project Phoenix with high priority.
    The deadline is March 2024 with a budget of $500k.
    Mike Chen will work on it as Lead Developer at 80% allocation.
    """,
    source=EpisodeType.text,
    entity_types={
        'Person': Person,
        'Project': Project
    },
    edge_types={
        'WORKS_ON': WorksOn,
        'REPORTS_TO': ReportsTo
    }
)
```

### Complex Entity Extraction

```python
# Healthcare domain example
class Patient(BaseModel):
    name: str
    mrn: str  # Medical Record Number
    age: Optional[int] = None
    gender: Optional[str] = None

class Diagnosis(BaseModel):
    condition: str
    icd_code: str
    severity: Optional[str] = None
    
class Medication(BaseModel):
    name: str
    dosage: str
    frequency: str
    route: str  # oral, IV, etc.

class Prescribes(BaseModel):
    start_date: date
    end_date: Optional[date] = None
    reason: str

# Medical episode
await graphiti.add_episode(
    name="Patient Visit",
    episode_body="""
    Patient John Smith (MRN: 12345) presented with acute bronchitis (J20.9).
    Prescribed Amoxicillin 500mg orally three times daily for 7 days.
    Follow-up in 1 week if symptoms persist.
    """,
    source=EpisodeType.text,
    entity_types={
        'Patient': Patient,
        'Diagnosis': Diagnosis,
        'Medication': Medication
    },
    edge_types={
        'PRESCRIBES': Prescribes
    }
)
```

## Advanced Features

### Bulk Episode Processing

```python
from graphiti_core.utils.bulk_utils import RawEpisode

# Prepare bulk episodes
episodes = []
for i, message in enumerate(chat_history):
    episodes.append(
        RawEpisode(
            name=f"Message {i}",
            content=message['content'],
            source_type=EpisodeType.text,
            reference_time=message['timestamp'],
            group_id=chat_id
        )
    )

# Process in bulk
results = await graphiti.add_episodes_bulk(episodes)
print(f"Processed {len(results)} episodes")
```

### Node and Edge Inspection

```python
# Get specific node details
node = await graphiti.get_node_by_uuid(node_uuid)
print(f"Node: {node.name}")
print(f"Type: {node.labels}")
print(f"Summary: {node.summary}")

# Get edges for a node
edges = await graphiti.get_edges_by_node_uuid(node_uuid)
for edge in edges:
    print(f"- {edge.fact}")
    print(f"  Valid from: {edge.valid_at}")
    print(f"  Invalid at: {edge.invalid_at or 'Still valid'}")
```

### Community Analysis

```python
# Build communities
await graphiti.build_communities()

# Search within communities
community_results = await graphiti.search(
    query="team dynamics",
    include_communities=True
)

for community in community_results.communities:
    print(f"\nCommunity: {community.name}")
    print(f"Summary: {community.summary}")
    print(f"Relevance: {community.score:.3f}")
```

### Temporal Queries

```python
# Get historical state
target_date = datetime(2024, 1, 1, tzinfo=timezone.utc)

# Retrieve episodes up to a specific date
historical_episodes = await graphiti.retrieve_episodes(
    reference_time=target_date,
    num_episodes=10
)

# Note: For edge temporal queries, you'll need to filter results
all_edges = await graphiti.search("project status")
valid_at_target = [
    edge for edge in all_edges.edges
    if edge.valid_at <= target_date and 
       (edge.invalid_at is None or edge.invalid_at > target_date)
]
```

## Error Handling

### Connection Errors

```python
from graphiti_core.errors import GraphitiError
import asyncio

async def connect_with_retry(max_retries=3):
    for attempt in range(max_retries):
        try:
            graphiti = Graphiti(
                uri="bolt://localhost:7687",
                user="neo4j",
                password="password"
            )
            await graphiti.driver.validate_connection()
            return graphiti
        except GraphitiError as e:
            print(f"Connection attempt {attempt + 1} failed: {e}")
            if attempt < max_retries - 1:
                await asyncio.sleep(2 ** attempt)  # Exponential backoff
            else:
                raise
```

### Episode Processing Errors

```python
try:
    result = await graphiti.add_episode(
        name="Episode",
        episode_body=content,
        source=EpisodeType.text
    )
except GraphitiError as e:
    print(f"Failed to process episode: {e}")
    # Log error, retry, or handle appropriately
```

### Search Errors

```python
try:
    results = await graphiti.search(query)
    if not results.edges and not results.nodes:
        print("No results found. Try a different query.")
except Exception as e:
    print(f"Search failed: {e}")
    # Fallback to a simpler search or return cached results
```

## Performance Tips

### Optimize Episode Size

```python
# Bad: One huge episode
await graphiti.add_episode(
    episode_body=entire_book_text,  # Don't do this!
    source=EpisodeType.text
)

# Good: Chapter-by-chapter
for chapter in chapters:
    await graphiti.add_episode(
        name=f"Chapter {chapter.number}",
        episode_body=chapter.text,
        source=EpisodeType.text,
        group_id=book_id
    )
```

### Use Appropriate Search Configs

```python
# For entity-focused queries
from graphiti_core.search.search_config_recipes import NODE_HYBRID_SEARCH_RRF
entity_results = await graphiti._search("Find all people", config=NODE_HYBRID_SEARCH_RRF)

# For relationship queries
from graphiti_core.search.search_config_recipes import EDGE_HYBRID_SEARCH_NODE_DISTANCE
rel_results = await graphiti._search("Who works with whom", config=EDGE_HYBRID_SEARCH_NODE_DISTANCE)

# For comprehensive search
from graphiti_core.search.search_config_recipes import COMBINED_HYBRID_SEARCH_CROSS_ENCODER
all_results = await graphiti._search("Project Phoenix details", config=COMBINED_HYBRID_SEARCH_CROSS_ENCODER)
```

### Batch Operations

```python
# Process multiple episodes concurrently
import asyncio

async def process_messages(messages):
    tasks = []
    for msg in messages:
        task = graphiti.add_episode(
            name=f"Message from {msg['sender']}",
            episode_body=msg['content'],
            source=EpisodeType.text,
            reference_time=msg['timestamp']
        )
        tasks.append(task)
    
    results = await asyncio.gather(*tasks, return_exceptions=True)
    
    successful = [r for r in results if not isinstance(r, Exception)]
    failed = [r for r in results if isinstance(r, Exception)]
    
    print(f"Processed {len(successful)} successfully, {len(failed)} failed")
    return successful, failed
```

### Connection Pooling

```python
# Reuse the same Graphiti instance
class GraphitiService:
    def __init__(self):
        self.graphiti = None
    
    async def initialize(self):
        self.graphiti = Graphiti(
            uri="bolt://localhost:7687",
            user="neo4j",
            password="password"
        )
        await self.graphiti.build_indices_and_constraints()
    
    async def close(self):
        if self.graphiti:
            await self.graphiti.driver.close()
    
    async def add_episode(self, **kwargs):
        return await self.graphiti.add_episode(**kwargs)
    
    async def search(self, **kwargs):
        return await self.graphiti.search(**kwargs)

# Use as a singleton
service = GraphitiService()
await service.initialize()
# Use service throughout application
await service.close()  # On shutdown
```

## Common Patterns

### Conversation Memory

```python
class ConversationMemory:
    def __init__(self, graphiti, conversation_id):
        self.graphiti = graphiti
        self.conversation_id = conversation_id
    
    async def add_message(self, role, content):
        return await self.graphiti.add_episode(
            name=f"{role} message",
            episode_body=f"{role}: {content}",
            source=EpisodeType.text,
            group_id=self.conversation_id,
            reference_time=datetime.now(timezone.utc)
        )
    
    async def get_context(self, query, max_results=10):
        return await self.graphiti.search(
            query=query,
            group_ids=[self.conversation_id],
            num_results=max_results
        )
    
    async def get_history(self, num_episodes=10):
        return await self.graphiti.retrieve_episodes(
            reference_time=datetime.now(timezone.utc),
            num_episodes=num_episodes,
            group_ids=[self.conversation_id]
        )
```

### Document Processing

```python
async def process_document(graphiti, document_path, chunk_size=1000):
    with open(document_path, 'r') as f:
        content = f.read()
    
    # Split into chunks
    chunks = [content[i:i+chunk_size] for i in range(0, len(content), chunk_size)]
    
    # Process each chunk
    for i, chunk in enumerate(chunks):
        await graphiti.add_episode(
            name=f"Document chunk {i+1}",
            episode_body=chunk,
            source=EpisodeType.text,
            group_id=f"doc_{document_path}"
        )
    
    print(f"Processed {len(chunks)} chunks from {document_path}")
```
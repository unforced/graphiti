# Graphiti Troubleshooting Guide

## Table of Contents

1. [Installation Issues](#installation-issues)
2. [Connection Problems](#connection-problems)
3. [Episode Processing Errors](#episode-processing-errors)
4. [Search Issues](#search-issues)
5. [Performance Problems](#performance-problems)
6. [LLM Provider Issues](#llm-provider-issues)
7. [Database Issues](#database-issues)
8. [Common Error Messages](#common-error-messages)

## Installation Issues

### Issue: Package installation fails

**Symptom:**
```bash
ERROR: Could not find a version that satisfies the requirement graphiti-core
```

**Solutions:**
1. Ensure Python 3.10+ is installed:
   ```bash
   python --version
   ```

2. Upgrade pip:
   ```bash
   pip install --upgrade pip
   ```

3. Use a virtual environment:
   ```bash
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   pip install graphiti-core
   ```

### Issue: Optional dependencies not found

**Symptom:**
```python
ImportError: No module named 'anthropic'
```

**Solution:**
Install with extras:
```bash
pip install graphiti-core[anthropic,groq,google-genai]
```

### Issue: FalkorDB client not available

**Symptom:**
```python
ImportError: cannot import name 'FalkorDBDriver'
```

**Solution:**
```bash
pip install graphiti-core[falkordb]
```

## Connection Problems

### Issue: Cannot connect to Neo4j

**Symptom:**
```python
neo4j.exceptions.ServiceUnavailable: Unable to connect to localhost:7687
```

**Solutions:**

1. Verify Neo4j is running:
   ```bash
   # Check if Neo4j is running
   sudo systemctl status neo4j  # Linux
   brew services list | grep neo4j  # macOS
   ```

2. Check connection details:
   ```python
   # Verify credentials
   graphiti = Graphiti(
       uri="bolt://localhost:7687",  # Correct protocol
       user="neo4j",  # Default username
       password="your-actual-password"  # Not "password"
   )
   ```

3. Test connection manually:
   ```python
   from neo4j import GraphDatabase
   
   driver = GraphDatabase.driver(
       "bolt://localhost:7687",
       auth=("neo4j", "password")
   )
   
   with driver.session() as session:
       result = session.run("RETURN 1")
       print(result.single()[0])
   
   driver.close()
   ```

### Issue: Authentication failed

**Symptom:**
```
neo4j.exceptions.AuthError: The client is unauthorized due to authentication failure
```

**Solutions:**

1. Reset Neo4j password:
   ```bash
   neo4j-admin set-initial-password newpassword
   ```

2. Use Neo4j Browser to verify credentials:
   - Navigate to http://localhost:7474
   - Test login credentials

### Issue: Connection timeout

**Symptom:**
```
TimeoutError: Connection to Neo4j timed out
```

**Solutions:**

1. Increase timeout:
   ```python
   from graphiti_core.driver.neo4j_driver import Neo4jDriver
   
   driver = Neo4jDriver(
       uri="bolt://localhost:7687",
       user="neo4j",
       password="password",
       max_connection_lifetime=3600,
       connection_timeout=30.0
   )
   
   graphiti = Graphiti(graph_driver=driver)
   ```

2. Check firewall settings:
   ```bash
   # Allow Neo4j port
   sudo ufw allow 7687/tcp  # Linux
   ```

## Episode Processing Errors

### Issue: Episode extraction fails

**Symptom:**
```
GraphitiError: Failed to extract entities from episode
```

**Solutions:**

1. Check LLM API key:
   ```python
   import os
   print(os.getenv("OPENAI_API_KEY"))  # Should not be None
   ```

2. Verify episode content:
   ```python
   # Ensure content is not empty
   if not episode_body.strip():
       print("Episode body is empty!")
   
   # Check content length
   if len(episode_body) > 10000:
       print("Episode may be too long, consider chunking")
   ```

3. Test LLM directly:
   ```python
   from openai import OpenAI
   client = OpenAI()
   
   response = client.chat.completions.create(
       model="gpt-4-turbo-preview",
       messages=[{"role": "user", "content": "Test"}]
   )
   print(response.choices[0].message.content)
   ```

### Issue: Custom entity extraction fails

**Symptom:**
```
ValidationError: Custom entity validation failed
```

**Solutions:**

1. Validate entity models:
   ```python
   from pydantic import ValidationError
   
   try:
       person = Person(
           first_name="John",
           last_name="Doe",
           # Missing required fields?
       )
   except ValidationError as e:
       print(e.errors())
   ```

2. Simplify entity models:
   ```python
   # Start simple
   class SimplePerson(BaseModel):
       name: str
   
   # Then add complexity
   class ComplexPerson(BaseModel):
       first_name: str
       last_name: str
       email: Optional[str] = None
   ```

### Issue: JSON episode parsing fails

**Symptom:**
```
JSONDecodeError: Expecting value
```

**Solution:**
```python
import json

# Validate JSON before adding
try:
    data = json.loads(episode_body)
except json.JSONDecodeError as e:
    print(f"Invalid JSON: {e}")
    # Fix or clean the JSON
    episode_body = json.dumps(data)  # Re-serialize
```

## Search Issues

### Issue: No search results

**Symptom:**
```python
results = await graphiti.search("query")
# results.edges is empty
```

**Solutions:**

1. Verify data exists:
   ```python
   # Check if episodes were added
   episodes = await graphiti.retrieve_episodes(
       reference_time=datetime.now(timezone.utc),
       num_episodes=10
   )
   print(f"Found {len(episodes)} episodes")
   ```

2. Try broader search:
   ```python
   # Remove filters
   results = await graphiti.search(
       query="*",  # Wildcard
       num_results=50  # Increase limit
   )
   ```

3. Check embeddings:
   ```python
   # Test embedding generation
   from graphiti_core.embedder import OpenAIEmbedder
   
   embedder = OpenAIEmbedder()
   embedding = await embedder.embed("test query")
   print(f"Embedding dimension: {len(embedding)}")
   ```

### Issue: Poor search relevance

**Symptom:**
Search returns unrelated results

**Solutions:**

1. Use appropriate search config:
   ```python
   from graphiti_core.search.search_config_recipes import (
       EDGE_HYBRID_SEARCH_RRF,
       NODE_HYBRID_SEARCH_MMR
   )
   
   # For relationships
   edge_results = await graphiti._search(
       query="who works with whom",
       config=EDGE_HYBRID_SEARCH_RRF
   )
   
   # For entities
   node_results = await graphiti._search(
       query="find all managers",
       config=NODE_HYBRID_SEARCH_MMR
   )
   ```

2. Use center node for context:
   ```python
   # Find anchor node first
   anchor = await graphiti.search("John Doe")
   if anchor.nodes:
       # Search around that node
       results = await graphiti.search(
           query="recent activities",
           center_node_uuid=anchor.nodes[0].uuid
       )
   ```

### Issue: Search timeout

**Symptom:**
```
TimeoutError: Search query timed out
```

**Solutions:**

1. Reduce search scope:
   ```python
   # Limit to specific groups
   results = await graphiti.search(
       query="query",
       group_ids=["specific_group"],
       num_results=10  # Smaller limit
   )
   ```

2. Use more specific queries:
   ```python
   # Bad: Too broad
   results = await graphiti.search("information")
   
   # Good: Specific
   results = await graphiti.search("Q4 sales figures for electronics")
   ```

## Performance Problems

### Issue: Slow episode ingestion

**Symptom:**
Episodes take >5 seconds to process

**Solutions:**

1. Use bulk processing:
   ```python
   from graphiti_core.utils.bulk_utils import RawEpisode
   
   episodes = [
       RawEpisode(name=f"Episode {i}", content=content, ...)
       for i, content in enumerate(contents)
   ]
   
   await graphiti.add_episodes_bulk(episodes)
   ```

2. Reduce episode size:
   ```python
   # Split large content
   def chunk_text(text, max_size=1000):
       return [text[i:i+max_size] for i in range(0, len(text), max_size)]
   
   for i, chunk in enumerate(chunk_text(large_content)):
       await graphiti.add_episode(
           name=f"Part {i+1}",
           episode_body=chunk,
           source=EpisodeType.text
       )
   ```

3. Use appropriate LLM model:
   ```python
   from graphiti_core.llm_client import LLMConfig, OpenAIClient
   
   # Use smaller model for better performance
   llm_config = LLMConfig(
       model="gpt-3.5-turbo",  # Faster
       small_model="gpt-3.5-turbo"
   )
   
   graphiti = Graphiti(
       uri="bolt://localhost:7687",
       user="neo4j", 
       password="password",
       llm_client=OpenAIClient(llm_config=llm_config)
   )
   ```

### Issue: High memory usage

**Symptom:**
Python process uses excessive RAM

**Solutions:**

1. Process in batches:
   ```python
   async def process_large_dataset(data, batch_size=100):
       for i in range(0, len(data), batch_size):
           batch = data[i:i+batch_size]
           await process_batch(batch)
           
           # Force garbage collection
           import gc
           gc.collect()
   ```

2. Close unused connections:
   ```python
   # Use context manager pattern
   class GraphitiContext:
       async def __aenter__(self):
           self.graphiti = Graphiti(...)
           return self.graphiti
       
       async def __aexit__(self, exc_type, exc_val, exc_tb):
           await self.graphiti.driver.close()
   
   async with GraphitiContext() as graphiti:
       await graphiti.add_episode(...)
   ```

## LLM Provider Issues

### Issue: OpenAI rate limit

**Symptom:**
```
openai.RateLimitError: Rate limit exceeded
```

**Solutions:**

1. Implement retry logic:
   ```python
   import asyncio
   from tenacity import retry, wait_exponential, stop_after_attempt
   
   @retry(
       wait=wait_exponential(multiplier=1, min=4, max=60),
       stop=stop_after_attempt(5)
   )
   async def add_episode_with_retry(**kwargs):
       return await graphiti.add_episode(**kwargs)
   ```

2. Use different provider:
   ```python
   from graphiti_core.llm_client import AnthropicClient
   
   llm_client = AnthropicClient(
       config=LLMConfig(
           api_key="your-key",
           model="claude-3-sonnet-20240229"
       )
   )
   
   graphiti = Graphiti(
       uri="bolt://localhost:7687",
       user="neo4j",
       password="password",
       llm_client=llm_client
   )
   ```

### Issue: Structured output failures

**Symptom:**
```
ValidationError: LLM output does not match expected schema
```

**Solutions:**

1. Use supported models:
   ```python
   # Good: Models with structured output support
   config = LLMConfig(model="gpt-4-turbo-preview")
   config = LLMConfig(model="gemini-1.5-pro")
   
   # Problematic: Smaller or older models
   # config = LLMConfig(model="gpt-3.5-turbo-0301")
   ```

2. Fallback handling:
   ```python
   try:
       result = await graphiti.add_episode(...)
   except ValidationError:
       # Retry with a better model
       graphiti.llm_client.llm_config.model = "gpt-4-turbo-preview"
       result = await graphiti.add_episode(...)
   ```

## Database Issues

### Issue: Index creation fails

**Symptom:**
```
DatabaseError: Failed to create vector index
```

**Solutions:**

1. Check Neo4j version:
   ```cypher
   CALL dbms.components() YIELD name, versions
   RETURN name, versions
   ```
   Ensure version is 5.26+

2. Manually create indices:
   ```python
   async def create_indices_manually(driver):
       queries = [
           "CREATE VECTOR INDEX entityNameIndex IF NOT EXISTS FOR (n:Entity) ON (n.name_embedding) OPTIONS {indexConfig: {`vector.dimensions`: 1536, `vector.similarity_function`: 'cosine'}}",
           "CREATE FULLTEXT INDEX entityTextIndex IF NOT EXISTS FOR (n:Entity) ON EACH [n.name]"
       ]
       
       async with driver.session() as session:
           for query in queries:
               await session.run(query)
   ```

### Issue: Transaction deadlocks

**Symptom:**
```
DeadlockDetected: Transaction was rolled back due to deadlock
```

**Solutions:**

1. Retry with backoff:
   ```python
   async def execute_with_retry(func, max_retries=3):
       for attempt in range(max_retries):
           try:
               return await func()
           except Exception as e:
               if "deadlock" in str(e).lower() and attempt < max_retries - 1:
                   await asyncio.sleep(2 ** attempt)
               else:
                   raise
   ```

2. Reduce concurrent operations:
   ```python
   # Limit concurrency
   semaphore = asyncio.Semaphore(5)
   
   async def limited_add_episode(**kwargs):
       async with semaphore:
           return await graphiti.add_episode(**kwargs)
   ```

## Common Error Messages

### "Entity type not found"

**Cause:** Custom entity type not registered

**Fix:**
```python
await graphiti.add_episode(
    episode_body="content",
    entity_types={'Person': Person}  # Register type
)
```

### "Embedding dimension mismatch"

**Cause:** Different embedding models used

**Fix:**
```python
# Consistent embedding config
embedder_config = OpenAIEmbedderConfig(
    embedding_model="text-embedding-3-small",
    embedding_dim=1536  # Match your index
)
```

### "Group ID format invalid"

**Cause:** Invalid characters in group_id

**Fix:**
```python
import re

def sanitize_group_id(group_id):
    # Remove special characters
    return re.sub(r'[^a-zA-Z0-9_-]', '_', group_id)

clean_id = sanitize_group_id("user@example.com/chat#123")
# Result: "user_example_com_chat_123"
```

### "Memory limit exceeded"

**Cause:** Large result sets

**Fix:**
```python
# Paginate results
async def paginated_search(query, page_size=100):
    all_results = []
    offset = 0
    
    while True:
        results = await graphiti._search(
            query=query,
            num_results=page_size,
            offset=offset
        )
        
        if not results.edges:
            break
            
        all_results.extend(results.edges)
        offset += page_size
        
    return all_results
```

## Getting Help

If you continue to experience issues:

1. **Check the logs:**
   ```python
   import logging
   logging.basicConfig(level=logging.DEBUG)
   ```

2. **Join the community:**
   - Discord: [Zep Discord #graphiti](https://discord.com/invite/W8Kw6bsgXQ)
   - GitHub Issues: [github.com/getzep/graphiti/issues](https://github.com/getzep/graphiti/issues)

3. **Provide details:**
   - Graphiti version: `pip show graphiti-core`
   - Python version: `python --version`
   - Database version
   - Error messages and stack traces
   - Minimal reproduction code
# Graphiti Plain Language Guide

## How Graphiti Works (Plain English)

Think of Graphiti as an intelligent librarian that:
1. **Reads** your content (journal entries, messages, etc.)
2. **Understands** what's important (people, places, events, concepts)
3. **Remembers** how things connect and when they happened
4. **Updates** its understanding as new information comes in

## The Flexibility You're Looking For

### 1. You Control What It Finds Valuable

```python
# Default mode - Graphiti figures it out
await graphiti.add_episode(
    episode_body="Met Sarah at the coffee shop. She's working on a meditation app."
)
# Graphiti extracts: Person(Sarah), Place(coffee shop), Project(meditation app)

# Custom mode - You tell it what matters
class MoodEntry(BaseModel):
    mood: str
    intensity: int
    triggers: List[str]

class LifeEvent(BaseModel):
    event_type: str  # "milestone", "challenge", "insight"
    impact_level: str  # "high", "medium", "low"

await graphiti.add_episode(
    episode_body="Feeling anxious (7/10) about the job interview tomorrow",
    entity_types={
        'MoodEntry': MoodEntry,
        'LifeEvent': LifeEvent
    }
)
```

### 2. Building Your Personal Ontology

For journal/Telegram use cases, you could define:

```python
# Personal Knowledge Ontology
class Person(BaseModel):
    name: str
    relationship: Optional[str]  # "friend", "family", "colleague"
    context: Optional[str]  # "met at conference", "childhood friend"

class Topic(BaseModel):
    name: str
    category: str  # "personal", "work", "hobby", "health"
    sentiment: Optional[str]  # "positive", "negative", "neutral"

class Insight(BaseModel):
    realization: str
    area: str  # "self", "relationships", "career"
    
class Goal(BaseModel):
    description: str
    timeframe: Optional[str]
    progress: Optional[int]  # 0-100

# Define how these relate
edge_types = {
    'DISCUSSED_WITH': DiscussedWith,  # Person -> Topic
    'LED_TO': LedTo,  # Insight -> Goal
    'AFFECTS': Affects,  # Topic -> MoodEntry
}
```

### 3. It Learns and Evolves

Here's the magic - Graphiti doesn't just store data, it **builds understanding**:

```python
# Day 1: Journal entry
"Had coffee with Mike. He mentioned his startup is struggling."

# Day 30: Another entry
"Mike's company pivoted to B2B and things are looking up!"

# Graphiti automatically:
# - Recognizes it's the same Mike
# - Updates the relationship (Mike -> Startup: "struggling" becomes "improving")
# - Preserves the history (you can query "when did Mike's startup struggle?")
```

### 4. Multi-Source Integration

Perfect for combining multiple data sources:

```python
# From journal
await graphiti.add_episode(
    name="Morning reflection",
    episode_body="Realized I need to spend more time on creative projects",
    source=EpisodeType.text,
    group_id="personal_journal"
)

# From Telegram
await graphiti.add_episode(
    name="Team chat",
    episode_body="Sarah: Anyone interested in a weekend hiking trip?",
    source=EpisodeType.text,
    group_id="telegram_hiking_group"
)

# From structured data (calendar, tasks, etc)
await graphiti.add_episode(
    name="Calendar sync",
    episode_body=json.dumps({
        "event": "Meditation workshop",
        "attendees": ["Me", "Sarah", "Mike"],
        "learnings": ["breathing techniques", "mindfulness"]
    }),
    source=EpisodeType.json,
    group_id="calendar_events"
)
```

## The Power of Temporal Understanding

What makes Graphiti special for personal knowledge:

1. **It remembers WHEN things were true**: 
   - "Sarah was interested in meditation in January"
   - "By March, she had launched her app"

2. **It tracks how your understanding evolved**:
   - "I thought the project was about wellness"
   - "Later learned it was specifically for anxiety"

3. **It handles contradictions gracefully**:
   - Doesn't delete old info when things change
   - Marks old facts as "invalid after X date"
   - You can ask "What did I know about this in February?"

## Practical Example for Your Use Case

```python
class PersonalKnowledgeGraph:
    def __init__(self):
        self.graphiti = Graphiti(
            uri="bolt://localhost:7687",
            user="neo4j",
            password="password"
        )
    
    async def process_journal_entry(self, entry_text, date):
        # Add the raw entry
        result = await self.graphiti.add_episode(
            name=f"Journal: {date.strftime('%Y-%m-%d')}",
            episode_body=entry_text,
            source=EpisodeType.text,
            reference_time=date,
            group_id="personal_journal",
            entity_types={
                'Person': Person,
                'Topic': Topic,
                'Insight': Insight,
                'MoodEntry': MoodEntry
            }
        )
        
        return result
    
    async def ask_questions(self):
        # "Who have I been spending time with?"
        people = await self.graphiti.search(
            "people I've met or mentioned",
            group_ids=["personal_journal"]
        )
        
        # "What insights did I have about career?"
        insights = await self.graphiti.search(
            "career insights and realizations",
            group_ids=["personal_journal"]
        )
        
        # "How has my mood changed over time regarding work?"
        mood_journey = await self.graphiti.search(
            "mood anxiety stress about work job",
            group_ids=["personal_journal"]
        )
```

## Why This is Better Than Raw Neo4j

1. **Automatic Entity Extraction**: You don't need to manually parse "Met Sarah" into nodes and edges
2. **Deduplication**: It figures out that "Sarah from coffee" and "Sarah the app developer" are the same person
3. **Temporal Magic**: Automatically handles the time aspect of knowledge
4. **Smart Search**: Combines semantic search ("find similar concepts") with graph traversal ("find connected ideas")
5. **LLM Integration**: Uses AI to understand context and meaning, not just keywords

## Your Next Steps

For your journal/Telegram knowledge graph:

1. **Start Simple**: Let Graphiti extract entities automatically first
2. **Observe Patterns**: See what kinds of things appear frequently
3. **Define Custom Types**: Create entities for patterns you care about
4. **Build Queries**: Create specific searches for your common questions
5. **Iterate**: Refine your ontology as you learn what's valuable

The beauty is you can start capturing data immediately and refine your ontology over time - all your historical data will still be there and searchable!
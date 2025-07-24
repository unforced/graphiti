# Graphiti for Personal Knowledge Graphs

## What is Graphiti?

Graphiti is a Python framework that creates living, evolving knowledge graphs from your personal data. Think of it as an intelligent librarian that reads your journal entries, messages, and notes, then builds a rich web of understanding about your life that grows smarter over time.

Unlike traditional databases or note-taking apps, Graphiti:
- **Understands meaning**, not just keywords
- **Tracks how things change** over time
- **Connects the dots** between people, places, ideas, and emotions
- **Remembers context** about when you learned something vs when it happened

## How Flexible is the Knowledge Graph?

### You Have Complete Control

Graphiti offers three levels of control over what it finds valuable:

#### 1. Automatic Mode (Start Here)
```python
# Just feed it your journal entries - Graphiti figures out what's important
await graphiti.add_episode(
    episode_body="Had coffee with Sarah. She mentioned her meditation app is launching next month.",
    source=EpisodeType.text
)
# Automatically extracts: Person(Sarah), Project(meditation app), Event(coffee meeting)
```

#### 2. Guided Mode (Add Your Patterns)
```python
# Tell Graphiti what types of things matter to you
class MoodEntry(BaseModel):
    mood: str
    intensity: int  # 1-10
    triggers: List[str]

await graphiti.add_episode(
    episode_body="Feeling anxious (8/10) about tomorrow's presentation",
    entity_types={'MoodEntry': MoodEntry}
)
```

#### 3. Full Control Mode (Define Everything)
```python
# Create your complete personal ontology
entity_types = {
    'Person': PersonWithContext,
    'Insight': PersonalInsight,
    'Goal': LifeGoal,
    'MoodEntry': MoodEntry,
    'Location': MeaningfulPlace
}

relationship_types = {
    'INFLUENCED': Influenced,  # Insight -> Goal
    'DISCUSSED_WITH': DiscussedWith,  # Person -> Topic
    'FELT_AT': FeltAt  # MoodEntry -> Location
}
```

### The Knowledge Graph Adapts to You

The magic is that Graphiti learns your patterns:

```python
# Week 1: You journal about "Sarah from work"
# Week 4: You mention "Sarah" during a work story
# Graphiti understands it's the same Sarah

# Month 2: You start mentioning "Sarah from yoga"
# Graphiti recognizes this is a different Sarah and keeps them separate
```

## Building From Multiple Sources

### Integrating Your Digital Life

Graphiti can build a unified knowledge graph from all your sources:

```python
# From your journal
await graphiti.add_episode(
    name="Morning Pages",
    episode_body="Realized I need to set better boundaries at work",
    source=EpisodeType.text,
    group_id="journal_2024"
)

# From Telegram messages
await graphiti.add_episode(
    name="Team Chat",
    episode_body="Mike: The deadline got moved to Friday\nMe: That's actually perfect",
    source=EpisodeType.text,
    group_id="telegram_work"
)

# From structured data (calendar, contacts, etc)
await graphiti.add_episode(
    name="Calendar Event",
    episode_body=json.dumps({
        "event": "Therapy Session",
        "topics": ["work boundaries", "stress management"],
        "insights": ["Need to practice saying no"]
    }),
    source=EpisodeType.json,
    group_id="calendar"
)
```

### Everything Connects

Graphiti automatically finds connections across sources:
- The "boundaries" insight from your journal connects to the therapy session
- Mike from Telegram is recognized as the same Mike from previous journal entries
- The deadline stress links to your mood entries

## Creating and Evolving Your Ontology

### Start Simple, Grow Naturally

```python
# Phase 1: Let Graphiti extract basic entities
# Just start journaling - see what patterns emerge

# Phase 2: Notice you often write about moods
class MoodEntry(BaseModel):
    mood: str
    context: Optional[str]

# Phase 3: Realize you want to track personal growth
class Insight(BaseModel):
    realization: str
    area: str  # "career", "relationships", "health"
    actionable: bool

# Phase 4: Connect insights to outcomes
class Outcome(BaseModel):
    description: str
    linked_insight: Optional[str]
    success_level: Optional[int]
```

### Your Ontology is Never "Done"

The beauty of Graphiti is that you can always add new entity types without losing your history:

```python
# Year 1: Basic journaling
# Year 2: Add MoodEntry tracking
# Year 3: Add HealthMetric type
# All historical data remains searchable with new understanding
```

## The Power of Temporal Understanding

### It Remembers When, Not Just What

```python
# January: "I think I want to switch careers"
# March: "Started learning Python"
# June: "Got my first freelance coding gig!"

# Query: "Show me my career journey"
# Graphiti returns the evolution, not just the current state
```

### Handling Changing Information

```python
# February: "Sarah is working on a meditation app"
# April: "Sarah pivoted her app to focus on sleep"
# June: "Sarah's sleep app got funded!"

# Graphiti preserves the entire journey
# You can ask: "What was Sarah working on in February?"
```

## Practical Examples

### Personal Growth Tracking
```python
insights = await graphiti.search(
    "personal realizations about work-life balance",
    group_ids=["journal_2024"]
)
# Returns insights with timestamps, showing your evolution
```

### Relationship Mapping
```python
# Find all interactions with specific people
sarah_interactions = await graphiti.search(
    "conversations with Sarah from TechCorp",
    center_node_uuid=sarah_node.uuid
)
```

### Mood Patterns
```python
# Analyze emotional patterns over time
mood_data = await graphiti.search(
    "anxiety stress worried",
    time_range=(datetime(2024, 1, 1), datetime(2024, 12, 31))
)
```

### Decision History
```python
# Trace how decisions evolved
decision_evolution = await graphiti.search(
    "career change thoughts decisions",
    include_temporal=True
)
```

## Why Graphiti for Personal Knowledge?

### 1. **It Understands Context**
- Knows "Sarah from work" vs "Sarah from yoga"
- Links "feeling stressed" to "project deadline" automatically
- Recognizes that "my therapist" and "Dr. Johnson" are the same person

### 2. **It Preserves Your Journey**
- Doesn't overwrite old thoughts with new ones
- Shows how your understanding evolved
- Lets you query your past mental states

### 3. **It Scales With You**
- Start with simple text entries
- Add structure as patterns emerge
- Integrate new sources anytime
- Query across years of data in milliseconds

### 4. **It Enables Self-Discovery**
- Find patterns you didn't know existed
- See connections across time periods
- Track personal growth quantitatively
- Ask questions you couldn't before

## Getting Started

```python
# 1. Initialize Graphiti
graphiti = Graphiti(
    uri="bolt://localhost:7687",
    user="neo4j",
    password="password"
)

# 2. Start with your most recent journal entry
await graphiti.add_episode(
    episode_body=today_journal_entry,
    source=EpisodeType.text,
    group_id="journal"
)

# 3. Ask your first question
results = await graphiti.search("What have I been thinking about lately?")

# 4. Let it grow from there
```

## The Bottom Line

Graphiti gives you a flexible, intelligent system that:
- **Adapts to YOUR patterns** rather than forcing you into rigid categories
- **Grows smarter** as you add more data
- **Preserves context** and temporal relationships
- **Enables queries** you couldn't ask before

It's not just storing your data - it's building an evolving understanding of your life that you can query, explore, and learn from. Whether you're tracking personal growth, managing relationships, or trying to understand your own patterns, Graphiti provides the flexible foundation to build your personal knowledge graph exactly how you need it.
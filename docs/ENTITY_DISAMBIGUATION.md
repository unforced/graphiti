# Entity Disambiguation in Graphiti

This guide explains how to handle multiple entities with similar names and connect external data sources (like contact books) to your knowledge graph.

## The Challenge

When journaling or processing natural language, you might mention multiple people with the same name:
- "Sarah" could be Sarah John from work
- "Sarah" could be Sarah Miller, your yoga instructor  
- "Sarah" could be Sarah Thompson, your neighbor

Graphiti can intelligently disambiguate these entities using context clues.

## Adding Rich Entity Information

First, when you have definitive information (like from your contact book), you can add it explicitly:

```python
# Define a rich Person entity
class Person(BaseModel):
    name: str
    full_name: Optional[str] = None
    phone: Optional[str] = None
    email: Optional[str] = None
    relationship: Optional[str] = None
    context_hints: Optional[List[str]] = None  # "works at X", "lives in Y", "married to Z"

# When you sync from contact book
contact_data = {
    "name": "Sarah John",
    "phone": "+1-555-0123",
    "email": "sarah.john@example.com",
    "company": "TechCorp",
    "notes": "Met at React conference 2023"
}

# Add this as a structured episode
await graphiti.add_episode(
    name="Contact: Sarah John",
    episode_body=json.dumps(contact_data),
    source=EpisodeType.json,
    group_id="contacts",
    entity_types={'Person': Person}
)
```

## Graphiti's Intelligent Context Understanding

Here's the magic - when you journal about "Sarah", Graphiti uses context to figure out which one:

```python
# Journal entry with context clues
await graphiti.add_episode(
    name="Daily journal",
    episode_body="""
    Had lunch with Sarah today. She mentioned the React conference 
    where we first met and how her job at TechCorp is going well.
    """,
    source=EpisodeType.text,
    group_id="journal"
)

# Graphiti will likely:
# 1. Extract "Sarah" as a person
# 2. Notice context: "React conference", "TechCorp"
# 3. Use its deduplication logic to link this to Sarah John
```

## Helping Graphiti with Disambiguation

You can make Graphiti even smarter by providing context windows:

```python
class SmartJournalProcessor:
    def __init__(self, graphiti):
        self.graphiti = graphiti
        self.recent_context = {}
    
    async def add_journal_entry(self, entry_text, date, mentioned_people=None):
        # If you know which Sarah you're talking about, help Graphiti
        context_hint = ""
        if mentioned_people:
            for person in mentioned_people:
                if "Sarah" in person:
                    context_hint = f"Context: {person} refers to Sarah John from TechCorp. "
        
        # Retrieve recent episodes for context
        recent_episodes = await self.graphiti.retrieve_episodes(
            reference_time=date,
            num_episodes=10,
            group_ids=["journal", "contacts"]
        )
        
        # Add episode with context
        result = await self.graphiti.add_episode(
            name=f"Journal: {date.strftime('%Y-%m-%d')}",
            episode_body=context_hint + entry_text,
            source=EpisodeType.text,
            reference_time=date,
            group_id="journal",
            previous_episode_uuids=[ep.uuid for ep in recent_episodes[-5:]]
        )
        
        return result
```

## Building a Contact-Aware System

Here's a more complete example that connects contacts to journal entries:

```python
class ContactAwareKnowledgeGraph:
    def __init__(self, graphiti):
        self.graphiti = graphiti
        self.contact_map = {}  # Cache for quick lookups
    
    async def import_contacts(self, contacts):
        """Import contacts with rich metadata"""
        for contact in contacts:
            # Create a detailed contact episode
            contact_info = {
                "full_name": contact.get("name"),
                "first_name": contact.get("first_name"),
                "last_name": contact.get("last_name"),
                "phone": contact.get("phone"),
                "email": contact.get("email"),
                "company": contact.get("company"),
                "relationship": contact.get("relationship", "contact"),
                "met_context": contact.get("notes"),
                "identifiers": {
                    "phone": contact.get("phone"),
                    "email": contact.get("email"),
                    "social": contact.get("social_handles", {})
                }
            }
            
            result = await self.graphiti.add_episode(
                name=f"Contact: {contact_info['full_name']}",
                episode_body=json.dumps(contact_info),
                source=EpisodeType.json,
                group_id="contacts",
                entity_types={'Person': Person}
            )
            
            # Cache for quick lookup
            self.contact_map[contact_info['full_name']] = result
    
    async def process_journal_with_disambiguation(self, entry_text, date):
        """Process journal entry with smart disambiguation"""
        
        # Look for potential person mentions
        potential_people = self._extract_potential_names(entry_text)
        
        # For each potential person, search for context
        disambiguation_hints = []
        for name in potential_people:
            # Search for this person in contacts and recent mentions
            search_results = await self.graphiti.search(
                query=f"{name} contact person",
                group_ids=["contacts", "journal"],
                num_results=5
            )
            
            if search_results.nodes:
                # Found potential matches
                for node in search_results.nodes:
                    if self._could_be_same_person(name, node):
                        disambiguation_hints.append(
                            f"{name} might refer to {node.name}"
                        )
        
        # Add entry with disambiguation context
        enhanced_entry = entry_text
        if disambiguation_hints:
            enhanced_entry = f"[Context: {'; '.join(disambiguation_hints)}] {entry_text}"
        
        return await self.graphiti.add_episode(
            name=f"Journal: {date.strftime('%Y-%m-%d')}",
            episode_body=enhanced_entry,
            source=EpisodeType.text,
            reference_time=date,
            group_id="journal"
        )
    
    def _extract_potential_names(self, text):
        # Simple name extraction - you could use NER here
        import re
        # Look for capitalized words that might be names
        words = re.findall(r'\b[A-Z][a-z]+\b', text)
        common_names = ["Sarah", "John", "Mike", "Lisa", "David"]  # etc
        return [w for w in words if w in common_names]
    
    def _could_be_same_person(self, mention, node):
        # Check if a mention could refer to a node
        mention_lower = mention.lower()
        node_name_lower = node.name.lower()
        
        # Direct match
        if mention_lower in node_name_lower:
            return True
        
        # First name match
        if ' ' in node.name and mention_lower == node.name.split()[0].lower():
            return True
        
        return False
```

## Querying with Disambiguation

When searching, you can be specific about which Sarah:

```python
# Find the right Sarah first
sarah_results = await graphiti.search("Sarah John TechCorp")
if sarah_results.nodes:
    sarah_john_node = sarah_results.nodes[0]
    
    # Now search for things related to this specific Sarah
    sarah_interactions = await graphiti.search(
        query="meetings conversations lunch",
        center_node_uuid=sarah_john_node.uuid,
        num_results=20
    )
    
    print(f"All interactions with Sarah John:")
    for edge in sarah_interactions.edges:
        print(f"- {edge.fact}")
```

## Advanced Pattern: Context Injection

For maximum accuracy, you can inject context before journaling:

```python
class ContextAwareJournal:
    def __init__(self, graphiti):
        self.graphiti = graphiti
        self.current_context = {}
    
    async def set_context(self, context_type, value):
        """Set context before journaling"""
        self.current_context[context_type] = value
        
        # Add a context episode
        await self.graphiti.add_episode(
            name=f"Context: {context_type}",
            episode_body=f"Current {context_type}: {value}",
            source=EpisodeType.text,
            group_id="context"
        )
    
    async def journal_with_context(self, entry):
        # Example: Set context before writing
        # await journal.set_context("talking_about", "Sarah John from TechCorp")
        
        # Build context prefix
        context_prefix = ""
        if self.current_context:
            context_items = [f"{k}: {v}" for k, v in self.current_context.items()]
            context_prefix = f"[Context: {', '.join(context_items)}] "
        
        return await self.graphiti.add_episode(
            name=f"Journal entry",
            episode_body=context_prefix + entry,
            source=EpisodeType.text,
            group_id="journal"
        )

# Usage
journal = ContextAwareJournal(graphiti)

# Before writing about Sarah John
await journal.set_context("person", "Sarah John (TechCorp colleague)")
await journal.journal_with_context("Had a great discussion with Sarah about the new project")

# Before writing about Sarah Miller  
await journal.set_context("person", "Sarah Miller (yoga instructor)")
await journal.journal_with_context("Sarah's class today focused on breathing techniques")
```

## Key Strategies for Disambiguation

1. **Rich Initial Data**: Add as much context as possible when you know it
2. **Consistent Patterns**: Use full names when introducing someone new
3. **Context Episodes**: Add explicit context when switching between people
4. **Search Verification**: Query to verify Graphiti understood correctly
5. **Group IDs**: Use different groups for different contexts (work vs personal)

## The Magic of Graphiti's Deduplication

Graphiti uses LLMs to understand context, so it can often figure out:
- "Sarah from TechCorp" = "Sarah John" = "S. John"
- "My yoga teacher Sarah" ≠ "Sarah from work"
- "Met Sarah at the conference" + previous mention of React conference = Sarah John

The more context you provide over time, the smarter it gets at disambiguation!

## Practical Implementation Example

Here's a complete example for a personal knowledge system:

```python
from typing import Dict, List, Optional
from datetime import datetime, timezone
import json

class PersonalKnowledgeSystem:
    def __init__(self, graphiti):
        self.graphiti = graphiti
        self.known_entities = {}  # Cache of known entities
        
    async def initialize(self):
        """Set up the system with initial data"""
        # Build indices
        await self.graphiti.build_indices_and_constraints()
        
        # Load known contacts
        await self.load_contacts_from_file("contacts.json")
    
    async def load_contacts_from_file(self, filepath):
        """Load contacts from a JSON file"""
        with open(filepath, 'r') as f:
            contacts = json.load(f)
        
        for contact in contacts:
            result = await self.graphiti.add_episode(
                name=f"Contact: {contact['full_name']}",
                episode_body=json.dumps(contact),
                source=EpisodeType.json,
                group_id="contacts",
                reference_time=datetime.now(timezone.utc)
            )
            
            # Cache for quick reference
            self.known_entities[contact['full_name']] = {
                'uuid': result.episode.uuid,
                'context': contact.get('company', '') + ' ' + contact.get('notes', '')
            }
    
    async def smart_journal_entry(self, entry: str, context_hints: Dict[str, str] = None):
        """Add a journal entry with smart entity recognition"""
        
        # Build context from recent entries
        recent = await self.graphiti.retrieve_episodes(
            reference_time=datetime.now(timezone.utc),
            num_episodes=5,
            group_ids=["journal"]
        )
        
        # Add any manual context hints
        enhanced_entry = entry
        if context_hints:
            context_str = " ".join([f"{k}:{v}" for k, v in context_hints.items()])
            enhanced_entry = f"[Context: {context_str}] {entry}"
        
        # Process the entry
        result = await self.graphiti.add_episode(
            name=f"Journal: {datetime.now().strftime('%Y-%m-%d %H:%M')}",
            episode_body=enhanced_entry,
            source=EpisodeType.text,
            group_id="journal",
            reference_time=datetime.now(timezone.utc),
            previous_episode_uuids=[ep.uuid for ep in recent]
        )
        
        return result
    
    async def find_person_interactions(self, person_name: str):
        """Find all interactions with a specific person"""
        
        # First, find the person entity
        person_search = await self.graphiti.search(
            query=person_name,
            group_ids=["contacts", "journal"],
            num_results=5
        )
        
        if not person_search.nodes:
            return None
        
        # Find the most likely match
        person_node = person_search.nodes[0]
        
        # Search for all interactions centered on this person
        interactions = await self.graphiti.search(
            query="",  # Empty query to get all relationships
            center_node_uuid=person_node.uuid,
            num_results=50
        )
        
        return {
            'person': person_node,
            'interactions': interactions.edges,
            'episodes': interactions.episodes
        }

# Usage
system = PersonalKnowledgeSystem(graphiti)
await system.initialize()

# Add journal entries
await system.smart_journal_entry(
    "Had coffee with Sarah. She's excited about the new React features.",
    context_hints={"location": "TechCorp cafeteria"}
)

# Find all Sarah interactions
sarah_data = await system.find_person_interactions("Sarah John")
```

This approach gives you:
- Automatic entity recognition and disambiguation
- Connection to external data sources
- Rich context preservation
- Intelligent search capabilities
- Historical tracking of relationships
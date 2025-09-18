# AGENTS.md

AI Agent Integration Guide for Bitmarks

## Overview

This document defines how AI agents (LLMs, assistants, and autonomous systems) should interact with the Bitmarks ecosystem. Bitmarks is designed to be AI-native, providing structured interfaces for agents to manage, search, and enhance bookmark collections.

## Agent Capabilities

### Core Operations

Agents integrated with Bitmarks can perform the following operations:

1. **Bookmark Management**
   - Create bookmarks from natural language descriptions
   - Update and enrich existing bookmarks
   - Organize bookmarks into intelligent collections
   - Delete or archive outdated content

2. **Search & Retrieval**
   - Natural language search queries
   - Semantic similarity search
   - Multi-faceted filtering
   - Context-aware recommendations

3. **Content Enhancement**
   - Automatic summarization
   - Tag generation and categorization
   - Metadata extraction
   - Related content discovery

4. **Workflow Automation**
   - Scheduled bookmark updates
   - Duplicate detection and merging
   - Link rot detection and repair
   - Content change monitoring

## MCP (Model Context Protocol) Server

The Bitmarks MCP server (`bitmarks-mcp`) provides a standardized interface for LLM integration.

### Available Tools

```typescript
interface BitmarksMCPTools {
  // Search tools
  'bitmarks.search': {
    description: 'Search bookmarks using natural language';
    parameters: {
      query: string;
      limit?: number;
      semanticSearch?: boolean;
    };
  };
  
  'bitmarks.searchByTags': {
    description: 'Find bookmarks with specific tags';
    parameters: {
      tags: string[];
      matchAll?: boolean;
    };
  };
  
  // CRUD tools
  'bitmarks.create': {
    description: 'Create a new bookmark';
    parameters: {
      url: string;
      title?: string;
      description?: string;
      tags?: string[];
      autoEnhance?: boolean;
    };
  };
  
  'bitmarks.update': {
    description: 'Update an existing bookmark';
    parameters: {
      id: string;
      updates: Partial<Bookmark>;
    };
  };
  
  'bitmarks.delete': {
    description: 'Delete a bookmark';
    parameters: {
      id: string;
      reason?: string;
    };
  };
  
  // Enhancement tools
  'bitmarks.summarize': {
    description: 'Generate a summary of bookmark content';
    parameters: {
      id: string;
      maxLength?: number;
    };
  };
  
  'bitmarks.suggestTags': {
    description: 'Suggest relevant tags for a bookmark';
    parameters: {
      id: string;
      count?: number;
    };
  };
  
  'bitmarks.findRelated': {
    description: 'Find bookmarks related to a given bookmark';
    parameters: {
      id: string;
      limit?: number;
    };
  };
  
  // Collection tools
  'bitmarks.createCollection': {
    description: 'Create a curated collection of bookmarks';
    parameters: {
      name: string;
      description: string;
      bookmarkIds: string[];
    };
  };
  
  'bitmarks.exportCollection': {
    description: 'Export bookmarks in various formats';
    parameters: {
      collectionId?: string;
      format: 'json' | 'html' | 'markdown' | 'csv';
    };
  };
}
```

### MCP Server Configuration

```typescript
// .mcp/config.json
{
  "name": "bitmarks",
  "version": "1.0.0",
  "description": "Bookmark management and search tools",
  "author": "Daniel Bodnar",
  "tools": ["search", "create", "update", "delete", "enhance"],
  "resources": {
    "bookmarks": {
      "type": "collection",
      "description": "User bookmark collection"
    },
    "tags": {
      "type": "taxonomy",
      "description": "Bookmark categorization system"
    }
  },
  "capabilities": {
    "search": {
      "semantic": true,
      "fulltext": true,
      "regex": true
    },
    "ai": {
      "summarization": true,
      "tagging": true,
      "embeddings": true
    }
  }
}
```

## Agent Interaction Patterns

### 1. Research Assistant Pattern

```typescript
// Agent helps user research a topic
async function researchTopic(topic: string) {
  // Search existing bookmarks
  const existing = await mcp.call('bitmarks.search', {
    query: topic,
    semanticSearch: true,
    limit: 20
  });
  
  // Identify gaps in knowledge
  const gaps = await analyzeGaps(existing, topic);
  
  // Suggest new resources
  const suggestions = await findNewResources(gaps);
  
  // Create bookmarks for valuable resources
  for (const resource of suggestions) {
    await mcp.call('bitmarks.create', {
      url: resource.url,
      title: resource.title,
      description: resource.summary,
      tags: [...topicTags, 'ai-suggested'],
      autoEnhance: true
    });
  }
  
  // Create research collection
  return await mcp.call('bitmarks.createCollection', {
    name: `Research: ${topic}`,
    description: `AI-curated resources for ${topic}`,
    bookmarkIds: [...existing.ids, ...suggestions.ids]
  });
}
```

### 2. Content Curator Pattern

```typescript
// Agent organizes and enhances bookmark collection
async function curateBookmarks() {
  // Detect duplicates
  const duplicates = await findDuplicates();
  await mergeDuplicates(duplicates);
  
  // Update stale content
  const stale = await findStaleBookmarks();
  for (const bookmark of stale) {
    await mcp.call('bitmarks.update', {
      id: bookmark.id,
      updates: await fetchLatestContent(bookmark.url)
    });
  }
  
  // Generate missing summaries
  const needsSummary = await findBookmarksWithoutSummary();
  for (const bookmark of needsSummary) {
    const summary = await mcp.call('bitmarks.summarize', {
      id: bookmark.id,
      maxLength: 200
    });
    
    await mcp.call('bitmarks.update', {
      id: bookmark.id,
      updates: { description: summary }
    });
  }
  
  // Suggest better organization
  const suggestions = await suggestTagReorganization();
  return suggestions;
}
```

### 3. Knowledge Graph Builder

```typescript
// Agent builds connections between bookmarks
async function buildKnowledgeGraph() {
  const bookmarks = await getAllBookmarks();
  const graph = new KnowledgeGraph();
  
  // Create nodes
  for (const bookmark of bookmarks) {
    graph.addNode({
      id: bookmark.id,
      type: 'bookmark',
      embedding: bookmark.embedding,
      metadata: bookmark
    });
  }
  
  // Find relationships
  for (const bookmark of bookmarks) {
    const related = await mcp.call('bitmarks.findRelated', {
      id: bookmark.id,
      limit: 5
    });
    
    for (const relatedBookmark of related) {
      graph.addEdge({
        from: bookmark.id,
        to: relatedBookmark.id,
        weight: relatedBookmark.similarity,
        type: 'semantic_similarity'
      });
    }
  }
  
  return graph;
}
```

## Prompt Engineering Guidelines

When agents interact with users about bookmarks, use these patterns:

### Search Queries
```
User: "Find me articles about Rust async programming"
Agent: I'll search your bookmarks for Rust async programming resources.

[Calls bitmarks.search with semantic search enabled]

Found 5 relevant bookmarks:
1. "Async Rust: The Book" - Comprehensive guide to async programming in Rust
2. "Tokio Tutorial" - Getting started with the Tokio async runtime
...
```

### Content Enhancement
```
User: "Summarize my machine learning bookmarks"
Agent: I'll analyze and summarize your machine learning bookmarks.

[Calls bitmarks.searchByTags for ML-related tags]
[Calls bitmarks.summarize for each bookmark]

Here's a summary of your ML resources:
- **Foundational Theory** (8 bookmarks): Covers basic ML concepts...
- **Deep Learning** (12 bookmarks): Neural network architectures...
- **Practical Applications** (5 bookmarks): Real-world ML projects...
```

### Organization Suggestions
```
User: "Help me organize my bookmarks better"
Agent: I'll analyze your bookmark collection and suggest improvements.

[Analyzes bookmark patterns and relationships]

Suggestions for better organization:
1. **Merge similar tags**: "js" and "javascript" → "javascript"
2. **Create collections**: Group your 47 tutorial bookmarks
3. **Archive stale content**: 23 bookmarks haven't been accessed in >1 year
4. **Add missing tags**: 15 bookmarks could benefit from categorization
```

## Security & Privacy

### Agent Permissions

Agents must respect user-defined permissions:

```typescript
enum AgentPermission {
  READ = 'read',           // Can search and view bookmarks
  WRITE = 'write',         // Can create and update bookmarks
  DELETE = 'delete',       // Can delete bookmarks
  ENHANCE = 'enhance',     // Can modify content (summaries, tags)
  EXPORT = 'export',       // Can export data
  ADMIN = 'admin'          // Full access
}

interface AgentConfig {
  id: string;
  name: string;
  permissions: AgentPermission[];
  rateLimits: {
    requests: number;
    period: 'minute' | 'hour' | 'day';
  };
  allowedDomains?: string[];  // Restrict bookmark domains
  maxBookmarks?: number;       // Limit collection size
}
```

### Data Handling

1. **No Training on User Data**: Agent providers must not train on user bookmarks
2. **Encryption**: Sensitive bookmarks must be encrypted before agent access
3. **Audit Logging**: All agent actions must be logged for user review
4. **Data Minimization**: Agents should only access necessary fields
5. **User Consent**: Explicit consent required for AI processing

## Performance Optimization

### Batch Operations
```typescript
// Good: Batch multiple operations
const bookmarks = await mcp.callBatch([
  ['bitmarks.create', { url: 'https://example1.com' }],
  ['bitmarks.create', { url: 'https://example2.com' }],
  ['bitmarks.create', { url: 'https://example3.com' }]
]);

// Bad: Individual calls in loop
for (const url of urls) {
  await mcp.call('bitmarks.create', { url });
}
```

### Caching Strategy
```typescript
interface AgentCache {
  embeddings: Map<string, Float32Array>;  // Cache computed embeddings
  summaries: Map<string, string>;         // Cache generated summaries
  searches: LRUCache<string, SearchResult[]>;  // Cache search results
}
```

### Rate Limiting
```typescript
const rateLimiter = new RateLimiter({
  maxRequests: 100,
  windowMs: 60000,  // 1 minute
  message: 'Agent rate limit exceeded'
});
```

## Integration Examples

### OpenAI GPT Integration
```python
from openai import OpenAI
from bitmarks_mcp import BitmarksMCP

client = OpenAI()
mcp = BitmarksMCP(api_key="...")

# Custom GPT action
def search_bookmarks(query: str) -> list:
    return mcp.call("bitmarks.search", {
        "query": query,
        "semanticSearch": True
    })

# Register as GPT action
actions = {
    "search_bookmarks": search_bookmarks,
    "create_bookmark": lambda url, title: mcp.call("bitmarks.create", {
        "url": url,
        "title": title
    })
}
```

### Claude Integration
```typescript
import Anthropic from '@anthropic-ai/sdk';
import { BitmarksMCP } from '@bitmarks/mcp';

const claude = new Anthropic();
const mcp = new BitmarksMCP();

// Tool use example
const response = await claude.messages.create({
  model: 'claude-3-sonnet',
  tools: mcp.getTools(),
  messages: [{
    role: 'user',
    content: 'Find and summarize my Rust bookmarks'
  }]
});
```

### Autonomous Agent
```typescript
class BookmarkAgent {
  async run() {
    // Daily maintenance tasks
    await this.detectAndFixBrokenLinks();
    await this.updateStaleContent();
    await this.generateMissingSummaries();
    await this.suggestNewBookmarks();
    await this.cleanupDuplicates();
    
    // Weekly analysis
    if (this.isWeekly()) {
      await this.generateUsageReport();
      await this.suggestCollections();
      await this.exportBackup();
    }
  }
}
```

## Monitoring & Analytics

Track agent interactions for optimization:

```typescript
interface AgentMetrics {
  totalRequests: number;
  successRate: number;
  averageLatency: number;
  popularTools: Map<string, number>;
  errorRate: number;
  userSatisfaction: number;
}

// Log agent actions
logger.info('Agent action', {
  agentId: agent.id,
  tool: 'bitmarks.search',
  parameters: { query: 'rust' },
  duration: 125,
  success: true
});
```

## Future Capabilities

Planned agent enhancements:

1. **Proactive Suggestions**: Agents suggest bookmarks based on browsing patterns
2. **Collaborative Filtering**: Share bookmark recommendations across users
3. **Content Generation**: Create summary documents from bookmark collections
4. **Workflow Automation**: Complex multi-step bookmark workflows
5. **Voice Integration**: Natural language bookmark management
6. **AR/VR Support**: Spatial bookmark organization
7. **Cross-Platform Sync**: Agent state synchronization across devices

## Contributing

To add new agent capabilities:

1. Define the tool in `bitmarks-mcp/src/tools/`
2. Add TypeScript types in `bitmarks-mcp/src/types/`
3. Implement the handler in `bitmarks-api/src/handlers/`
4. Write tests in `bitmarks-mcp/tests/`
5. Update this documentation
6. Submit PR with examples

## Support

For agent integration support:
- Documentation: https://docs.bitmarks.io/agents
- Discord: https://discord.gg/bitmarks
- GitHub Issues: https://github.com/danielbodnar/bitmarks
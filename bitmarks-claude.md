# CLAUDE.md

This file provides guidance to Claude Code when working with the Bitmarks project.

## Project Overview

Bitmarks is a distributed bookmark management system written primarily in Rust for the backend (Cloudflare Workers) and TypeScript for clients. The project follows a microservices architecture with multiple independent packages that communicate via well-defined APIs.

## Architecture Principles

### Core Philosophy
- **Unix Philosophy**: Each component does ONE thing exceptionally well
- **Edge-First**: Leverage Cloudflare's edge network for global performance
- **Privacy by Design**: User data encryption and zero-knowledge options
- **Offline-First**: Full functionality even without network connectivity
- **API-First**: All functionality exposed via clean REST/GraphQL APIs

### Technology Stack

#### Backend (bitmarks-api)
- **Language**: Rust (latest stable)
- **Runtime**: Cloudflare Workers (WebAssembly)
- **Database**: Cloudflare D1 (SQLite)
- **Object Storage**: Cloudflare R2
- **KV Store**: Cloudflare Workers KV
- **Vector DB**: Cloudflare Vectorize
- **Queue**: Cloudflare Queues
- **Full-Text Search**: Custom Rust implementation
- **Authentication**: Cloudflare Zero Trust / Custom JWT

#### Client Libraries
- **CLI**: Rust with TypeScript wrapper via Bun
- **Web/Extensions**: TypeScript (ES2024+), Vue 3/Astro
- **Desktop**: Tauri (Rust + TypeScript)
- **Mobile**: React Native or Flutter
- **MCP Server**: TypeScript with Muppet runtime

## Development Guidelines

### Rust Development

```rust
// ALWAYS use these patterns in Rust code

// Error handling with thiserror
use thiserror::Error;

#[derive(Error, Debug)]
pub enum BookmarkError {
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),
    
    #[error("Invalid URL: {0}")]
    InvalidUrl(String),
}

// Use async-trait for interfaces
#[async_trait]
pub trait BookmarkRepository {
    async fn create(&self, bookmark: CreateBookmark) -> Result<Bookmark, BookmarkError>;
    async fn find_by_id(&self, id: Uuid) -> Result<Option<Bookmark>, BookmarkError>;
}

// Prefer functional style with iterators
let tags: Vec<String> = bookmark_tags
    .split(',')
    .map(|s| s.trim().to_lowercase())
    .filter(|s| !s.is_empty())
    .collect();

// Use serde for serialization
#[derive(Serialize, Deserialize, Debug)]
#[serde(rename_all = "camelCase")]
pub struct Bookmark {
    pub id: Uuid,
    pub url: String,
    pub title: Option<String>,
    #[serde(with = "chrono::serde::ts_seconds")]
    pub created_at: DateTime<Utc>,
}
```

### TypeScript/JavaScript Development

```typescript
// ALWAYS use these patterns in TypeScript

// Use Zod for validation
import { z } from 'zod';

const BookmarkSchema = z.object({
  id: z.string().uuid(),
  url: z.string().url(),
  title: z.string().optional(),
  tags: z.array(z.string()),
  createdAt: z.string().datetime(),
});

// Prefer functional composition
const normalizedTags = tags
  .split(',')
  .map(tag => tag.trim().toLowerCase())
  .filter(Boolean)
  .sort();

// Use modern ES2024+ features
const bookmarks = await Promise.all(
  urls.map(async url => {
    const response = await fetch(url);
    return response.json();
  })
);

// Always handle errors properly
try {
  const bookmark = await createBookmark(data);
  return { success: true, bookmark };
} catch (error) {
  console.error('Bookmark creation failed:', error);
  return { success: false, error: error.message };
}
```

### Database Schema

```sql
-- D1 SQLite schema
CREATE TABLE bookmarks (
  id TEXT PRIMARY KEY,
  url TEXT NOT NULL,
  normalized_url TEXT NOT NULL,
  title TEXT,
  description TEXT,
  content TEXT,  -- Cached page content
  tags TEXT,     -- JSON array
  metadata TEXT, -- JSON object
  embedding BLOB, -- Vector embedding
  
  -- Timestamps
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  accessed_at INTEGER,
  
  -- Sync metadata
  device_id TEXT,
  sync_version INTEGER DEFAULT 0,
  deleted INTEGER DEFAULT 0,
  
  -- Indexes
  UNIQUE(normalized_url, device_id)
);

CREATE INDEX idx_bookmarks_url ON bookmarks(normalized_url);
CREATE INDEX idx_bookmarks_created ON bookmarks(created_at);
CREATE INDEX idx_bookmarks_sync ON bookmarks(sync_version, device_id);
CREATE VIRTUAL TABLE bookmarks_fts USING fts5(
  title, description, content, tags,
  content=bookmarks
);
```

### API Design

All APIs should follow RESTful principles with these conventions:

```typescript
// Request/Response format
interface ApiResponse<T> {
  success: boolean;
  data?: T;
  error?: {
    code: string;
    message: string;
    details?: unknown;
  };
  pagination?: {
    page: number;
    limit: number;
    total: number;
    hasMore: boolean;
  };
}

// Standard query parameters
interface QueryParams {
  page?: number;      // 1-based pagination
  limit?: number;     // Max 100
  sort?: string;      // field:asc or field:desc
  filter?: string;    // JSON filter object
  fields?: string;    // Comma-separated field list
}

// Error codes
enum ErrorCode {
  INVALID_REQUEST = 'INVALID_REQUEST',
  NOT_FOUND = 'NOT_FOUND',
  UNAUTHORIZED = 'UNAUTHORIZED',
  RATE_LIMITED = 'RATE_LIMITED',
  INTERNAL_ERROR = 'INTERNAL_ERROR',
}
```

### Testing Strategy

```rust
// Rust tests
#[cfg(test)]
mod tests {
    use super::*;
    
    #[tokio::test]
    async fn test_create_bookmark() {
        let repo = MockBookmarkRepository::new();
        let bookmark = CreateBookmark {
            url: "https://example.com".to_string(),
            tags: vec!["test".to_string()],
        };
        
        let result = repo.create(bookmark).await;
        assert!(result.is_ok());
    }
}
```

```typescript
// TypeScript tests using Bun
import { describe, it, expect } from 'bun:test';

describe('BookmarkService', () => {
  it('should create bookmark with valid URL', async () => {
    const service = new BookmarkService();
    const bookmark = await service.create({
      url: 'https://example.com',
      tags: ['test'],
    });
    
    expect(bookmark).toBeDefined();
    expect(bookmark.url).toBe('https://example.com');
  });
});
```

## Common Patterns

### Caching Strategy
```typescript
// Use KV for hot data, R2 for cold storage
const cacheKey = `bookmark:${id}`;
const cached = await env.KV.get(cacheKey, 'json');
if (cached) return cached;

const bookmark = await fetchFromD1(id);
await env.KV.put(cacheKey, JSON.stringify(bookmark), {
  expirationTtl: 3600, // 1 hour
});
return bookmark;
```

### Vector Search
```typescript
// Generate embeddings for semantic search
const embedding = await generateEmbedding(text);
const results = await env.VECTORIZE.query(embedding, {
  topK: 10,
  namespace: 'bookmarks',
});
```

### Queue Processing
```typescript
// Async processing with Cloudflare Queues
await env.QUEUE.send({
  type: 'FETCH_CONTENT',
  bookmarkId: id,
  priority: 'low',
});
```

## Performance Considerations

1. **Batch Operations**: Always batch database operations when possible
2. **Lazy Loading**: Load content on-demand, not eagerly
3. **Edge Caching**: Leverage Cloudflare's cache API aggressively
4. **Streaming**: Use streams for large data transfers
5. **Pagination**: Never return unbounded result sets

## Security Best Practices

1. **Input Validation**: Validate all inputs with Zod schemas
2. **SQL Injection**: Use parameterized queries exclusively
3. **XSS Prevention**: Sanitize all user content
4. **CORS**: Configure strict CORS policies
5. **Rate Limiting**: Implement per-user rate limits
6. **Encryption**: Encrypt sensitive data at rest

## Deployment

```bash
# Development
wrangler dev --local

# Staging
wrangler deploy --env staging

# Production
wrangler deploy --env production

# Migrations
wrangler d1 migrations apply bitmarks-db
```

## Monitoring

Key metrics to track:
- Request latency (p50, p95, p99)
- Error rates by endpoint
- Database query performance
- Cache hit rates
- Queue processing times
- Storage usage (KV, R2, D1)

## Common Issues and Solutions

### Issue: Slow search performance
**Solution**: Ensure FTS5 indexes are properly configured and consider using Vectorize for semantic search.

### Issue: Sync conflicts
**Solution**: Implement CRDT-based conflict resolution or last-write-wins with version vectors.

### Issue: Large content storage
**Solution**: Store content in R2, metadata in D1, and use KV for hot cache.

## AI Integration Guidelines

When implementing AI features:
1. Use Cloudflare Workers AI for embeddings
2. Implement streaming responses for LLM interactions
3. Cache embeddings aggressively
4. Batch embedding generation
5. Provide fallback for AI service failures

## Contributing Workflow

1. Create feature branch from `main`
2. Write tests first (TDD)
3. Implement feature
4. Ensure all tests pass
5. Run linter and formatter
6. Update documentation
7. Create pull request with clear description
8. Address review feedback
9. Squash and merge when approved

## Resources

- [Cloudflare Workers Docs](https://developers.cloudflare.com/workers/)
- [Rust WASM Guide](https://rustwasm.github.io/docs/book/)
- [D1 Database Docs](https://developers.cloudflare.com/d1/)
- [Vectorize Docs](https://developers.cloudflare.com/vectorize/)
- [Tauri Docs](https://tauri.app/v1/guides/)
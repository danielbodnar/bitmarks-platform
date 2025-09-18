# Bitmarks

> A high-performance, privacy-first bookmark management ecosystem built on Cloudflare Workers with Rust

[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Built with Rust](https://img.shields.io/badge/Built%20with-Rust-orange.svg)](https://www.rust-lang.org/)
[![Powered by Cloudflare Workers](https://img.shields.io/badge/Powered%20by-Cloudflare%20Workers-f38020.svg)](https://workers.cloudflare.com/)

## Overview

Bitmarks is a modern, distributed bookmark management system that unifies browser history, bookmarks, and GitHub stars into a single, searchable, AI-enhanced knowledge base. Built with performance and privacy at its core, Bitmarks leverages Cloudflare's global edge network to provide lightning-fast access to your personal web archive from anywhere in the world.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         Clients                              │
├──────┬──────┬──────┬──────┬──────┬──────┬──────┬───────────┤
│ CLI  │ Ext  │ Web  │Tauri │ iOS  │ Droid│ MCP  │  API      │
└──┬───┴──┬───┴──┬───┴──┬───┴──┬───┴──┬───┴──┬───┴──┬────────┘
   │      │      │      │      │      │      │      │
   └──────┴──────┴──────┴──────┴──────┴──────┴──────┘
                            │
                   ┌────────▼────────┐
                   │ Cloudflare Edge  │
                   │   Workers (Rust)  │
                   └────────┬────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
   ┌────▼────┐       ┌─────▼─────┐      ┌─────▼─────┐
   │   R2    │       │    D1     │      │ Durable   │
   │ Storage │       │ Database  │      │ Objects   │
   └─────────┘       └───────────┘      └───────────┘
        │                   │                   │
   ┌────▼────┐       ┌─────▼─────┐      ┌─────▼─────┐
   │  Cache  │       │  Vectors  │      │   Queue   │
   │  (KV)   │       │ (Vectorize)│     │ (Queues)  │
   └─────────┘       └───────────┘      └───────────┘
```

## Core Components

### bitmarks-api
The core Rust-based API server running on Cloudflare Workers, handling all CRUD operations, authentication, and synchronization.

### bitmarks-cli
Command-line interface for managing bookmarks, performing searches, and bulk operations.

### bitmarks-ext
Browser extensions for Chrome, Firefox, Safari, and Edge that integrate seamlessly with browser bookmarks and history.

### bitmarks-app
Cross-platform desktop application built with Tauri, providing a native experience for bookmark management.

### bitmarks-web
Web interface for accessing and managing bookmarks from any browser.

### bitmarks-mobile
Native mobile applications for iOS and Android.

### bitmarks-mcp
Model Context Protocol server for AI integration and enhanced search capabilities.

### bitmarks-crawler
Background service for fetching, parsing, and indexing bookmark content.

### bitmarks-cache
Intelligent caching layer for offline access and performance optimization.

### bitmarks-ai
AI service for embeddings generation, semantic search, and content analysis.

## Features

### 🔍 Advanced Search
- **Full-text search** with Typesense-like capabilities
- **Semantic search** using vector embeddings
- **Hybrid search** combining keyword and semantic approaches
- **Faceted search** with tags, domains, and metadata filtering

### 🔄 Synchronization
- **Bidirectional sync** with browser bookmarks and history
- **GitHub stars** integration and synchronization
- **Conflict resolution** with intelligent merge strategies
- **Real-time updates** across all connected devices

### 🤖 AI-Powered
- **Automatic tagging** and categorization
- **Content summarization** for quick previews
- **Duplicate detection** with similarity matching
- **Smart suggestions** based on browsing patterns

### 🛡️ Privacy & Security
- **End-to-end encryption** option for sensitive bookmarks
- **Zero-knowledge architecture** available
- **GDPR compliant** with data export/import capabilities
- **Self-hostable** option for complete control

### ⚡ Performance
- **Edge computing** with sub-50ms response times globally
- **Offline-first** architecture with local caching
- **Incremental sync** for minimal bandwidth usage
- **WebAssembly** powered for native-like performance

## Quick Start

### Installation

```bash
# Install CLI
cargo install bitmarks-cli

# Or use bun for the TypeScript client
bun add -g @bitmarks/cli

# Initialize configuration
bitmarks init

# Import existing bookmarks
bitmarks import --source chrome --source firefox
```

### Basic Usage

```bash
# Add a bookmark
bitmarks add https://example.com --tags "example,demo"

# Search bookmarks
bitmarks search "rust programming"

# Semantic search
bitmarks search --semantic "tutorials for learning async programming"

# Sync with GitHub stars
bitmarks sync github --token $GITHUB_TOKEN
```

## API Reference

### REST Endpoints

```http
GET    /api/bookmarks              # List bookmarks with pagination
POST   /api/bookmarks              # Create new bookmark
GET    /api/bookmarks/:id          # Get specific bookmark
PUT    /api/bookmarks/:id          # Update bookmark
DELETE /api/bookmarks/:id          # Delete bookmark
POST   /api/bookmarks/search       # Advanced search
GET    /api/bookmarks/export       # Export bookmarks
POST   /api/bookmarks/import       # Import bookmarks

GET    /api/sync/status            # Sync status
POST   /api/sync/trigger           # Trigger sync
GET    /api/sync/conflicts         # Get sync conflicts
POST   /api/sync/resolve           # Resolve conflicts

POST   /api/ai/embeddings          # Generate embeddings
POST   /api/ai/summarize           # Summarize content
POST   /api/ai/suggest-tags        # Suggest tags
```

## Development

### Prerequisites

- Rust 1.75+
- Bun 1.2+
- Cloudflare Workers account
- Wrangler CLI

### Setup

```bash
# Clone repository
git clone https://github.com/danielbodnar/bitmarks
cd bitmarks

# Install dependencies
bun install
cargo build

# Configure environment
cp .env.example .env
# Edit .env with your Cloudflare credentials

# Deploy to Cloudflare Workers
wrangler deploy
```

### Project Structure

```
bitmarks/
├── packages/
│   ├── bitmarks-api/        # Core API (Rust)
│   ├── bitmarks-cli/        # CLI tool (Rust/TypeScript)
│   ├── bitmarks-ext/        # Browser extensions
│   ├── bitmarks-app/        # Tauri desktop app
│   ├── bitmarks-web/        # Web interface
│   ├── bitmarks-mobile/     # Mobile apps
│   ├── bitmarks-mcp/        # MCP server
│   ├── bitmarks-crawler/    # Content crawler
│   ├── bitmarks-cache/      # Caching layer
│   └── bitmarks-ai/         # AI services
├── shared/                  # Shared types and utilities
├── docs/                    # Documentation
├── scripts/                 # Build and deployment scripts
└── tests/                   # Integration tests
```

## Configuration

Bitmarks uses a hierarchical configuration system:

1. Environment variables (highest priority)
2. Configuration file (`~/.config/bitmarks/config.toml`)
3. Default values

Example configuration:

```toml
[api]
endpoint = "https://api.bitmarks.io"
timeout = 30

[sync]
interval = 300  # seconds
strategy = "merge"  # or "replace", "manual"

[search]
default_limit = 20
enable_semantic = true

[cache]
max_size = "100MB"
ttl = 86400  # seconds
```

## Contributing

We welcome contributions! Please see our [Contributing Guide](CONTRIBUTING.md) for details.

### Development Philosophy

This project adheres to the Unix Philosophy:
- Each component does one thing well
- Components are designed to work together
- Text streams are the universal interface
- Simplicity and clarity over complexity

## License

MIT License - see [LICENSE](LICENSE) for details.

## Acknowledgments

- Built on [Cloudflare Workers](https://workers.cloudflare.com/)
- Powered by [Rust](https://www.rust-lang.org/) and [WebAssembly](https://webassembly.org/)
- Search powered by Cloudflare [Vectorize](https://developers.cloudflare.com/vectorize/) and [D1](https://developers.cloudflare.com/d1/)
- Desktop apps built with [Tauri](https://tauri.app/)

## Links

- [Documentation](https://docs.bitmarks.io)
- [API Reference](https://api.bitmarks.io/docs)
- [Discord Community](https://discord.gg/bitmarks)
- [GitHub](https://github.com/danielbodnar/bitmarks)

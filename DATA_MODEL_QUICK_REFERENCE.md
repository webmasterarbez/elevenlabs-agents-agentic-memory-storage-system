# OpenMemory Data Model - Quick Reference

## Table of Contents
1. [Core Tables](#core-tables)
2. [Data Types](#data-types)
3. [Sector Types](#sector-types)
4. [API Endpoints](#api-endpoints)
5. [Common Queries](#common-queries)

---

## Core Tables

### memories
Main storage for all memory content.

```sql
CREATE TABLE memories (
    id              UUID PRIMARY KEY,
    user_id         TEXT,
    content         TEXT NOT NULL,
    primary_sector  TEXT NOT NULL,
    salience        DOUBLE,
    created_at      BIGINT,
    last_seen_at    BIGINT
);
```

**Key Fields:**
- `id`: Unique memory identifier
- `user_id`: For multi-tenant isolation
- `content`: The actual memory text
- `primary_sector`: episodic | semantic | procedural | emotional | reflective
- `salience`: Importance score (0.0 - 1.0)

### vectors
Embeddings for semantic search.

```sql
CREATE TABLE vectors (
    id      UUID,
    sector  TEXT,
    v       BYTEA NOT NULL,
    dim     INTEGER NOT NULL,
    PRIMARY KEY (id, sector)
);
```

**Key Fields:**
- `id`: References memories.id
- `sector`: Which cognitive sector
- `v`: Binary vector data
- `dim`: Vector dimension (typically 1536)

### waypoints
Graph connections between memories.

```sql
CREATE TABLE waypoints (
    src_id      TEXT,
    dst_id      TEXT NOT NULL,
    user_id     TEXT,
    weight      DOUBLE NOT NULL,
    PRIMARY KEY (src_id, user_id)
);
```

**Key Fields:**
- `src_id`: Source memory
- `dst_id`: Target memory
- `weight`: Connection strength (0.0 - 1.0)

### users
User profiles and summaries.

```sql
CREATE TABLE users (
    user_id            TEXT PRIMARY KEY,
    summary            TEXT,
    reflection_count   INTEGER,
    created_at         BIGINT
);
```

---

## Data Types

### TypeScript/JavaScript

```typescript
// Core Memory Type
type Memory = {
    id: string
    content: string
    primary_sector: 'episodic' | 'semantic' | 'procedural' | 'emotional' | 'reflective'
    sectors: string[]
    salience: number
    created_at: number
    user_id?: string
}

// Query Match
type QueryMatch = Memory & {
    score: number
    path: string[]
}

// Add Request
type AddMemoryRequest = {
    content: string
    tags?: string[]
    metadata?: Record<string, any>
    user_id?: string
}

// Query Request
type QueryRequest = {
    query: string
    k?: number
    filters?: {
        sector?: string
        user_id?: string
        min_score?: number
    }
}
```

---

## Sector Types

| Sector | Description | Decay Rate | Example |
|--------|-------------|------------|---------|
| **episodic** | Events & experiences | 0.015 | "I went to the store yesterday" |
| **semantic** | Facts & knowledge | 0.005 | "Paris is the capital of France" |
| **procedural** | How-to & processes | 0.008 | "To start the server, run npm start" |
| **emotional** | Feelings & sentiment | 0.020 | "I'm so excited about this project!" |
| **reflective** | Meta insights | 0.001 | "User frequently asks about React" |

**Decay Rate Interpretation:**
- Lower = slower decay (facts persist longer)
- Higher = faster decay (emotions fade quickly)

---

## API Endpoints

### Memory Operations

#### Add Memory
```http
POST /memory/add
Content-Type: application/json

{
  "content": "User prefers dark mode",
  "user_id": "user123",
  "tags": ["preference", "ui"]
}

Response: {
  "id": "uuid-here",
  "primary_sector": "semantic",
  "sectors": ["semantic", "procedural"]
}
```

#### Query Memories
```http
POST /memory/query
Content-Type: application/json

{
  "query": "What are the user's preferences?",
  "k": 5,
  "filters": {
    "user_id": "user123"
  }
}

Response: {
  "query": "...",
  "matches": [
    {
      "id": "uuid",
      "content": "...",
      "score": 0.95,
      "salience": 0.8,
      "path": ["direct"]
    }
  ]
}
```

#### Get All Memories
```http
GET /memory/all?l=100&u=0&user_id=user123

Response: {
  "items": [...]
}
```

#### Reinforce Memory
```http
POST /memory/reinforce
Content-Type: application/json

{
  "id": "memory-uuid",
  "boost": 0.1
}

Response: {
  "ok": true
}
```

#### Update Memory
```http
PATCH /memory/:id
Content-Type: application/json

{
  "content": "Updated content",
  "tags": ["new", "tags"]
}

Response: {
  "id": "uuid",
  "updated": true
}
```

#### Delete Memory
```http
DELETE /memory/:id

Response: {
  "ok": true
}
```

### User Operations

#### Get User Summary
```http
GET /users/:user_id/summary

Response: {
  "user_id": "user123",
  "summary": "50 memories, 12 patterns | active | avg_sal=0.67",
  "reflection_count": 5,
  "created_at": 1699564800000
}
```

#### List Users
```http
GET /users

Response: {
  "users": [
    {
      "user_id": "user123",
      "memory_count": 50,
      "last_active": 1699564800000
    }
  ]
}
```

### System Operations

#### Health Check
```http
GET /health

Response: {
  "ok": true,
  "uptime": 3600,
  "version": "2.1.0"
}
```

#### Statistics
```http
GET /stats

Response: {
  "totalMemories": 1000,
  "sectors": {
    "episodic": 300,
    "semantic": 400,
    "procedural": 200,
    "emotional": 50,
    "reflective": 50
  },
  "avgSalience": 0.65
}
```

### LangGraph Integration

#### Store Node Memory
```http
POST /lgm/store
Content-Type: application/json

{
  "node": "agent_1",
  "content": "User asked about pricing",
  "namespace": "default",
  "graph_id": "conversation_123"
}

Response: {
  "memory_id": "uuid",
  "node": "agent_1"
}
```

#### Retrieve Node Memories
```http
POST /lgm/retrieve
Content-Type: application/json

{
  "node": "agent_1",
  "query": "pricing",
  "namespace": "default",
  "limit": 5
}

Response: {
  "memories": [...]
}
```

### IDE Integration

#### Track IDE Event
```http
POST /api/ide/events
Content-Type: application/json

{
  "event_type": "save",
  "file_path": "src/index.ts",
  "content": "function main() {...}",
  "session_id": "session_123",
  "metadata": {
    "project": "my-app",
    "lang": "typescript"
  }
}

Response: {
  "success": true,
  "memory_id": "uuid",
  "primary_sector": "procedural"
}
```

#### Query IDE Context
```http
POST /api/ide/context
Content-Type: application/json

{
  "query": "database connection",
  "session_id": "session_123",
  "k": 5
}

Response: {
  "success": true,
  "memories": [...],
  "total": 5
}
```

---

## Common Queries

### SQL Queries

#### Get all memories for a user
```sql
SELECT * FROM memories
WHERE user_id = 'user123'
ORDER BY created_at DESC
LIMIT 100;
```

#### Get memories by sector
```sql
SELECT * FROM memories
WHERE primary_sector = 'semantic'
  AND user_id = 'user123'
ORDER BY salience DESC;
```

#### Get connected memories (via waypoints)
```sql
SELECT m.* FROM memories m
JOIN waypoints w ON w.dst_id = m.id
WHERE w.src_id = 'memory-uuid'
  AND w.user_id = 'user123'
ORDER BY w.weight DESC;
```

#### Get top salience memories
```sql
SELECT * FROM memories
WHERE user_id = 'user123'
  AND salience > 0.7
ORDER BY salience DESC, last_seen_at DESC
LIMIT 10;
```

#### Count memories per sector
```sql
SELECT primary_sector, COUNT(*) as count
FROM memories
WHERE user_id = 'user123'
GROUP BY primary_sector;
```

### JavaScript/TypeScript SDK

#### Initialize Client
```typescript
import { OpenMemory } from '@openmemory/sdk'

const client = new OpenMemory({
  baseUrl: 'http://localhost:8080',
  apiKey: 'your-api-key'
})
```

#### Add Memory
```typescript
const memory = await client.add({
  content: 'User likes TypeScript',
  user_id: 'user123',
  tags: ['programming', 'preference']
})
console.log(memory.id, memory.primary_sector)
```

#### Query Memories
```typescript
const results = await client.query({
  query: 'What does the user like?',
  k: 5,
  filters: { user_id: 'user123' }
})

results.matches.forEach(match => {
  console.log(`${match.content} (score: ${match.score})`)
})
```

#### Get Memory by ID
```typescript
const memory = await client.get('memory-uuid')
console.log(memory.content)
```

#### Update Memory
```typescript
await client.update('memory-uuid', {
  content: 'Updated content',
  tags: ['new-tag']
})
```

#### Delete Memory
```typescript
await client.delete('memory-uuid')
```

#### List All Memories
```typescript
const { items } = await client.list({
  limit: 100,
  offset: 0,
  sector: 'semantic',
  user_id: 'user123'
})
```

---

## Data Relationships Cheat Sheet

```
User (1) ──< (N) Memories
              │
              ├──< (N) Vectors (one per sector)
              │
              └──< (N) Waypoints (graph edges)
                        │
                        └──> (1) Other Memory
```

**Key Relationships:**
- 1 Memory → Multiple Sectors (episodic, semantic, etc.)
- 1 Memory → Multiple Vectors (one per sector)
- 1 Memory → 0-1 Waypoint (single strongest link)
- 1 User → Many Memories
- 1 User → 1 Summary (auto-generated)

---

## Salience & Decay

### Salience Scoring
```
New Memory Salience = Base (0.5) + Adjustments

Adjustments:
- Has tags: +0.1
- Has metadata: +0.05
- High word count (>100): +0.1
- Emotional sector: +0.15
- Explicit salience: Use provided value
```

### Decay Formula
```
New Salience = Current Salience × e^(-λ × Δt)

Where:
- λ = decay_lambda (per sector)
- Δt = time since last_seen_at
```

### Memory Tiers
```
Hot:   Recently accessed + (coactivations > 5 OR salience > 0.7)
Warm:  Recently accessed OR salience > 0.4
Cold:  Infrequently accessed + salience ≤ 0.4

Decay rates:
- Hot:  λ = 0.005
- Warm: λ = 0.02
- Cold: λ = 0.05
```

---

## Composite Query Scoring

When querying memories, the final score is:

```
Final Score = 0.6×similarity + 0.2×salience + 0.1×recency + 0.1×link_weight

Where:
- similarity: Cosine similarity of embeddings
- salience: Memory importance
- recency: e^(-(now - last_seen_at) / time_unit)
- link_weight: Waypoint connection strength
```

---

## Environment Variables Reference

```bash
# Database
OM_METADATA_BACKEND=sqlite          # or 'postgres'
OM_DB_PATH=./data/openmemory.sqlite # SQLite path
OM_PG_HOST=localhost                # Postgres host
OM_PG_PORT=5432                     # Postgres port
OM_PG_USER=openmemory               # Postgres user
OM_PG_PASSWORD=password             # Postgres password
OM_PG_DB=openmemory                 # Postgres database
OM_PG_SCHEMA=public                 # Postgres schema

# Embeddings
OM_EMBEDDING_PROVIDER=openai        # openai, gemini, ollama, local, synthetic
OM_OPENAI_API_KEY=sk-...            # OpenAI API key
OM_GEMINI_API_KEY=...               # Gemini API key
OM_OLLAMA_URL=http://localhost:11434 # Ollama endpoint

# Decay
OM_DECAY_INTERVAL=300               # Decay cycle interval (seconds)
OM_DECAY_THREADS=3                  # Parallel decay threads
OM_DECAY_COLD_THRESHOLD=0.25        # Salience threshold for cold tier
OM_REGENERATION_ENABLED=true        # Enable vector compression

# User Summary
OM_USER_SUMMARY_INTERVAL=30         # Summary update interval (minutes)

# Server
OM_PORT=8080                        # Server port
OM_API_KEY=your-secret-key          # API authentication key
```

---

## Error Codes

| Code | Description | Solution |
|------|-------------|----------|
| 400 | Bad request (missing required fields) | Check request body matches schema |
| 401 | Unauthorized (invalid API key) | Set OM_API_KEY and send in x-api-key header |
| 403 | Forbidden (user_id mismatch) | Ensure user_id in request matches memory owner |
| 404 | Not found (memory doesn't exist) | Verify memory ID is correct |
| 500 | Internal server error | Check server logs |

---

## Performance Tips

1. **Always use user_id**: Reduces search space by 90%+
2. **Filter by sector**: Narrows vector search to 1-2 sectors
3. **Limit k parameter**: Default k=8, max k=50 for performance
4. **Batch operations**: Use bulk insert for >10 memories
5. **Enable compression**: Set OM_REGENERATION_ENABLED=true
6. **Monitor decay**: Tune OM_DECAY_INTERVAL based on memory count
7. **Use tags wisely**: 3-5 tags per memory optimal
8. **Set appropriate salience**: Important memories should start at 0.8+

---

## Migration Quick Start

### From Zep
```bash
cd migrate
node index.js --from zep --api-key YOUR_ZEP_KEY --verify
```

### From Mem0
```bash
cd migrate
node index.js --from mem0 --api-key YOUR_MEM0_KEY --verify
```

### From Supermemory
```bash
cd migrate
node index.js --from supermemory --api-key YOUR_SM_KEY --verify
```

---

**Last Updated:** 2025-11-09
**Version:** 2.1.0

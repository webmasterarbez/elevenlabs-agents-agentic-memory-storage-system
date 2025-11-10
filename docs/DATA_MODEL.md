# OpenMemory Data Model

## Overview

OpenMemory uses a multi-layered data architecture that separates concerns across database, service, API, and UI layers. The core design principle is the **Hierarchical Semantic Graph (HSG)** which organizes memories by cognitive sectors and builds associations through waypoints.

---

## Architecture Layers

### 1. Database Layer (Persistence)
### 2. Service Layer (Business Logic)
### 3. API/SDK Layer (Integration)
### 4. UI Layer (Presentation)

---

## 1. Database Schema

OpenMemory supports both **SQLite** and **PostgreSQL** as storage backends.

### Core Tables

#### `memories`
The central table storing all memory content and metadata.

| Column | Type | Description |
|--------|------|-------------|
| `id` | TEXT/UUID | Primary key (UUID format) |
| `user_id` | TEXT | User identifier for multi-tenant isolation |
| `segment` | INTEGER | Memory segment for partitioning (default: 0) |
| `content` | TEXT | The actual memory content (required) |
| `simhash` | TEXT | Similarity hash for deduplication |
| `primary_sector` | TEXT | Primary cognitive sector (episodic, semantic, procedural, emotional, reflective) |
| `tags` | TEXT | JSON array of tags |
| `meta` | TEXT | JSON metadata object |
| `created_at` | INTEGER/BIGINT | Unix timestamp of creation |
| `updated_at` | INTEGER/BIGINT | Unix timestamp of last update |
| `last_seen_at` | INTEGER/BIGINT | Unix timestamp of last access/query |
| `salience` | REAL/DOUBLE | Memory importance score (0.0 - 1.0) |
| `decay_lambda` | REAL/DOUBLE | Decay rate parameter per memory |
| `version` | INTEGER | Version counter for optimistic locking |
| `mean_dim` | INTEGER | Dimension of mean vector |
| `mean_vec` | BLOB/BYTEA | Mean embedding vector across sectors |
| `compressed_vec` | BLOB/BYTEA | Compressed vector for low-memory states |
| `feedback_score` | REAL/DOUBLE | User feedback score (default: 0) |

**Indexes:**
- `idx_memories_sector` on `primary_sector`
- `idx_memories_segment` on `segment`
- `idx_memories_simhash` on `simhash`
- `idx_memories_ts` on `last_seen_at`
- `idx_memories_user` on `user_id`

#### `vectors`
Stores sector-specific embeddings for each memory.

| Column | Type | Description |
|--------|------|-------------|
| `id` | TEXT/UUID | Memory ID (foreign key to memories.id) |
| `sector` | TEXT | Sector type (episodic, semantic, etc.) |
| `user_id` | TEXT | User identifier |
| `v` | BLOB/BYTEA | Embedding vector binary data |
| `dim` | INTEGER | Vector dimension |

**Primary Key:** (`id`, `sector`)

**Indexes:**
- `idx_vectors_user` on `user_id`

#### `waypoints`
Graph edges representing associations between memories.

| Column | Type | Description |
|--------|------|-------------|
| `src_id` | TEXT | Source memory ID |
| `dst_id` | TEXT | Destination memory ID |
| `user_id` | TEXT | User identifier |
| `weight` | REAL/DOUBLE | Association strength (0.0 - 1.0) |
| `created_at` | INTEGER/BIGINT | Creation timestamp |
| `updated_at` | INTEGER/BIGINT | Last update timestamp |

**Primary Key:** (`src_id`, `user_id`)

**Indexes:**
- `idx_waypoints_src` on `src_id`
- `idx_waypoints_dst` on `dst_id`
- `idx_waypoints_user` on `user_id`

#### `users`
User profiles and generated summaries.

| Column | Type | Description |
|--------|------|-------------|
| `user_id` | TEXT | Primary key |
| `summary` | TEXT | Auto-generated user summary |
| `reflection_count` | INTEGER | Number of reflections performed |
| `created_at` | INTEGER/BIGINT | Account creation timestamp |
| `updated_at` | INTEGER/BIGINT | Last update timestamp |

#### `embed_logs`
Tracks embedding operation status for async processing.

| Column | Type | Description |
|--------|------|-------------|
| `id` | TEXT | Log entry ID (primary key) |
| `model` | TEXT | Embedding model name |
| `status` | TEXT | Status: 'pending', 'completed', 'failed' |
| `ts` | INTEGER/BIGINT | Timestamp |
| `err` | TEXT | Error message if failed |

#### `stats`
System maintenance operation statistics.

| Column | Type | Description |
|--------|------|-------------|
| `id` | INTEGER | Auto-increment primary key |
| `type` | TEXT | Operation type: 'decay', 'reflect', 'consolidate' |
| `count` | INTEGER | Operation count |
| `ts` | INTEGER/BIGINT | Timestamp |

**Indexes:**
- `idx_stats_ts` on `ts`
- `idx_stats_type` on `type`

---

## 2. Service Layer Models

### Memory Sectors (Cognitive Architecture)

OpenMemory categorizes memories into 5 cognitive sectors:

| Sector | Description | Decay Lambda | Weight | Use Case |
|--------|-------------|--------------|--------|----------|
| **episodic** | Event memories - temporal data | 0.015 | 1.2 | "I went to the store yesterday" |
| **semantic** | Facts & preferences - factual data | 0.005 | 1.0 | "Paris is the capital of France" |
| **procedural** | Habits, triggers - action patterns | 0.008 | 1.1 | "How to install Node.js" |
| **emotional** | Sentiment states - tone analysis | 0.020 | 1.3 | "I'm so excited about this!" |
| **reflective** | Meta memory & logs - audit trail | 0.001 | 0.8 | System-generated insights |

### HSG Memory Model

```typescript
interface hsg_mem {
    id: string
    content: string
    primary_sector: string
    sectors: string[]              // Multiple sectors per memory
    tags?: string
    meta?: any
    created_at: number
    updated_at: number
    last_seen_at: number
    salience: number               // Importance score
    decay_lambda: number           // Individual decay rate
    version: number
}
```

### Query Result Model

```typescript
interface hsg_q_result {
    id: string
    content: string
    score: number                  // Composite similarity score
    sectors: string[]
    primary_sector: string
    path: string[]                 // Waypoint traversal path
    salience: number
    last_seen_at: number
}
```

### Waypoint Model

```typescript
interface waypoint {
    src_id: string
    dst_id: string
    weight: number                 // Association strength
    created_at: number
    updated_at: number
}
```

### Decay Tier System

Memories are classified into tiers based on access patterns:

```typescript
type MemoryTier = 'hot' | 'warm' | 'cold'

// Tier determination:
// - hot: Recently accessed AND (high coactivations OR high salience)
// - warm: Recently accessed OR moderate salience
// - cold: Infrequently accessed, low salience
```

**Decay rates by tier:**
- **Hot:** λ = 0.005
- **Warm:** λ = 0.02
- **Cold:** λ = 0.05

### User Summary Model

User summaries are auto-generated by clustering memories:

```typescript
interface UserSummary {
    total_memories: number
    pattern_count: number
    activity_level: 'active' | 'moderate' | 'low'
    avg_salience: number
    top_patterns: ClusterPattern[]
}

interface ClusterPattern {
    sector: string
    count: number
    salience: number
    snippet: string
}
```

---

## 3. API/SDK Layer Models

### Request Models

#### AddMemoryRequest
```typescript
interface AddMemoryRequest {
    content: string                // Required
    tags?: string[]
    metadata?: Record<string, unknown>
    salience?: number
    decay_lambda?: number
    user_id?: string
}
```

#### QueryRequest
```typescript
interface QueryRequest {
    query: string                  // Required
    k?: number                     // Number of results (default: 8)
    filters?: {
        tags?: string[]
        min_score?: number
        sector?: SectorType
        user_id?: string
        min_salience?: number
    }
}
```

#### IngestRequest
```typescript
interface IngestRequest {
    source: 'file' | 'link' | 'connector'
    content_type: 'pdf' | 'docx' | 'html' | 'md' | 'txt' | 'audio'
    data: string                   // Base64 or raw content
    metadata?: Record<string, unknown>
    config?: {
        force_root?: boolean
        sec_sz?: number            // Section size
        lg_thresh?: number         // Large section threshold
    }
    user_id?: string
}
```

### Response Models

#### Memory (Response)
```typescript
interface Memory {
    id: string
    content: string
    primary_sector: SectorType
    sectors: SectorType[]
    tags?: string
    metadata?: Record<string, unknown>
    created_at: number
    updated_at: number
    last_seen_at: number
    salience: number
    decay_lambda: number
    version: number
}
```

#### QueryMatch (extends Memory)
```typescript
interface QueryMatch extends Memory {
    score: number                  // Similarity score
    path: string[]                 // Waypoint path from query
}
```

#### QueryResponse
```typescript
interface QueryResponse {
    query: string
    matches: QueryMatch[]
}
```

### LangGraph Integration Models

```typescript
interface LangGraphStoreRequest {
    node: string                   // Node identifier
    content: string
    tags?: string[]
    metadata?: Record<string, unknown>
    namespace?: string
    graph_id?: string
    reflective?: boolean
}

interface LangGraphRetrieveRequest {
    node: string
    query?: string
    namespace?: string
    graph_id?: string
    limit?: number
    include_metadata?: boolean
}
```

### IDE Integration Models

```typescript
interface IDEEventRequest {
    event: 'edit' | 'open' | 'close' | 'save' | 'refactor' | 'comment' |
           'pattern_detected' | 'api_call' | 'definition' | 'reflection'
    file?: string
    snippet?: string
    comment?: string
    metadata: {
        project?: string
        lang?: string
        user?: string
        timestamp?: number
        [key: string]: unknown
    }
    session_id?: string
}

interface IDEContextQueryRequest {
    query: string
    k?: number
    session_id?: string
    file_filter?: string
    include_patterns?: boolean
    include_knowledge?: boolean
}
```

---

## 4. UI Layer Models (Dashboard)

### Dashboard Metrics

```typescript
interface DashboardStats {
    totalMemories: number
    avgSalience: number
    recentMemories: number          // Last 24h
    totalRequests: number
    errors: number
    errorRate: string
}

interface QPSStats {
    peakQps: number
    avgQps: number
    total: number
    errors: number
    cacheHit: number
}

interface SystemHealth {
    activeSegments: number
    maxActive: number
    uptime: number
    memoryUsage: string
}
```

### Activity Log

```typescript
interface ActivityLog {
    time: string                    // Formatted timestamp
    event: string                   // Event type
    sector: string
    salience: number
    level: 'Critical' | 'Warning' | 'Info'
}
```

### Memory (UI Model)

```typescript
interface UIMemory {
    id: string
    content: string
    primary_sector: string
    tags: string[]
    metadata?: any
    created_at: number
    updated_at?: number
    last_seen_at?: number
    salience: number
    decay_lambda?: number
    version?: number
}
```

### Sector Statistics

```typescript
interface SectorStats {
    sector: SectorType
    count: number
    avg_salience: number
}
```

---

## Entity Relationships

### Key Relationships

1. **memories ↔ vectors** (1:N)
   - One memory has multiple vectors (one per sector)
   - Relationship: `memories.id = vectors.id`

2. **memories ↔ waypoints** (N:N via graph)
   - Memories connected via waypoints (directed graph)
   - Relationship: `waypoints.src_id → memories.id`, `waypoints.dst_id → memories.id`

3. **users ↔ memories** (1:N)
   - One user has many memories
   - Relationship: `users.user_id = memories.user_id`

4. **users ↔ waypoints** (1:N)
   - Waypoints are user-scoped
   - Relationship: `users.user_id = waypoints.user_id`

5. **memories ↔ embed_logs** (1:1)
   - Tracking embedding operations
   - Relationship: `memories.id = embed_logs.id`

### Cascading Behavior

- **Delete Memory**: Also deletes associated vectors, waypoints (src and dst)
- **User Isolation**: All operations filtered by `user_id` when provided
- **Sector Independence**: Each sector maintains separate embeddings

---

## Query Flow & Data Transformations

### Add Memory Flow
```
1. Client sends AddMemoryRequest
2. Service classifies into sectors (primary + additional)
3. Generate embeddings per sector
4. Insert into memories table
5. Insert vectors per sector
6. Update user summary (async)
7. Return Memory response
```

### Query Flow
```
1. Client sends QueryRequest
2. Service classifies query into sectors (2-3 types)
3. Generate query embeddings per sector
4. Vector search in each sector
5. Aggregate results across sectors
6. Expand via waypoints (one-hop)
7. Composite scoring:
   - 60% similarity
   - 20% salience
   - 10% recency
   - 10% link weight
8. Return top-K QueryMatch results
```

### Decay Flow (Scheduled)
```
1. Cron job triggers decay cycle
2. Load all memories
3. Classify into tiers (hot/warm/cold)
4. Apply decay per tier
5. Compress vectors for cold memories
6. Prune low-weight waypoints
7. Log stats
```

---

## Data Lifecycle

### Memory States

1. **Creation**: New memory with initial salience
2. **Active**: Frequently accessed, salience maintained/increased
3. **Warm**: Moderate access, gradual decay
4. **Cold**: Rare access, compressed, fast decay
5. **Archived**: Very low salience, candidates for deletion

### Reinforcement

Memories can be reinforced through:
- **Query hits**: Increment coactivation count
- **Explicit reinforcement**: API call to boost salience
- **Waypoint creation**: Association with other active memories

### Compression

Cold memories undergo:
- **Vector compression**: Reduce dimensionality
- **Summary extraction**: Keep keywords instead of full content
- **Metadata pruning**: Remove non-essential metadata

---

## Indexing Strategy

### Vector Search Optimization

OpenMemory uses **sector-based sharding**:
- Each sector has its own vector space
- Queries only search relevant sectors (2-3 max)
- Reduces search space by ~60-80%

### Graph Traversal Optimization

Waypoints use **single-waypoint linking**:
- Each memory → one strongest waypoint
- One-hop expansion during query
- Prevents graph explosion

### User Isolation

All queries filtered by `user_id`:
- Index on `user_id` in memories, vectors, waypoints
- Ensures tenant data isolation
- No cross-user leakage

---

## Configuration

### Environment Variables

Key configuration affecting data model:

- `OM_METADATA_BACKEND`: 'sqlite' or 'postgres'
- `OM_DB_PATH`: SQLite file path
- `OM_PG_*`: PostgreSQL connection params
- `OM_DECAY_THREADS`: Parallel decay processing
- `OM_DECAY_COLD_THRESHOLD`: Salience threshold for cold tier
- `OM_MAX_VECTOR_DIM`: Maximum vector dimension
- `OM_MIN_VECTOR_DIM`: Minimum compressed dimension

---

## Migration & Versioning

### Schema Migrations

OpenMemory uses auto-migration on startup:
- Tables created if not exist
- Indexes added if missing
- Backward compatible with v1.x data

### Data Migration

See `MIGRATION.md` for migrating from:
- Zep
- Mem0
- Supermemory

### Version Field

`memories.version` tracks:
- Content updates
- Metadata changes
- Used for optimistic locking

---

## Best Practices

### Data Modeling

1. **Use user_id**: Always scope to users for multi-tenancy
2. **Tag appropriately**: Tags improve filtering and clustering
3. **Set salience**: Important memories should start with high salience
4. **Metadata structure**: Keep metadata JSON-serializable

### Performance

1. **Limit query k**: Default k=8, max recommended k=50
2. **Use sector filters**: Narrow search to 1-2 sectors
3. **Batch operations**: Use bulk insert for large datasets
4. **Monitor decay**: Tune decay_lambda per use case

### Security

1. **User isolation**: Never query without user_id filter in multi-tenant
2. **API keys**: Require authentication for write operations
3. **Content encryption**: Use AES-GCM for sensitive content
4. **PII scrubbing**: Remove PII before storage

---

## Appendix: Type Definitions

```typescript
type SectorType = 'episodic' | 'semantic' | 'procedural' | 'emotional' | 'reflective'

type MemoryTier = 'hot' | 'warm' | 'cold'

type EventType = 'decay' | 'reflect' | 'consolidate'

type EmbedStatus = 'pending' | 'completed' | 'failed'

type ContentType = 'pdf' | 'docx' | 'html' | 'md' | 'txt' | 'audio'

type IDEEvent = 'edit' | 'open' | 'close' | 'save' | 'refactor' |
                'comment' | 'pattern_detected' | 'api_call' |
                'definition' | 'reflection'
```

---

**Last Updated:** 2025-11-09
**Version:** 2.1.0

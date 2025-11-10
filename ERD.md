# OpenMemory Entity Relationship Diagram

## Database Schema ERD

```mermaid
erDiagram
    memories ||--o{ vectors : "has embeddings"
    memories ||--o{ waypoints : "source of"
    memories ||--o{ waypoints : "target of"
    users ||--o{ memories : "owns"
    users ||--o{ waypoints : "owns"
    memories ||--o| embed_logs : "tracks"

    memories {
        uuid id PK
        text user_id FK
        integer segment
        text content "NOT NULL"
        text simhash
        text primary_sector "NOT NULL"
        text tags "JSON array"
        text meta "JSON object"
        bigint created_at
        bigint updated_at
        bigint last_seen_at
        double salience
        double decay_lambda
        integer version
        integer mean_dim
        bytea mean_vec
        bytea compressed_vec
        double feedback_score
    }

    vectors {
        uuid id PK,FK
        text sector PK
        text user_id
        bytea v "NOT NULL"
        integer dim "NOT NULL"
    }

    waypoints {
        text src_id PK,FK
        text dst_id FK
        text user_id PK,FK
        double weight "NOT NULL"
        bigint created_at
        bigint updated_at
    }

    users {
        text user_id PK
        text summary
        integer reflection_count
        bigint created_at
        bigint updated_at
    }

    embed_logs {
        text id PK
        text model
        text status
        bigint ts
        text err
    }

    stats {
        integer id PK
        text type "NOT NULL"
        integer count
        bigint ts "NOT NULL"
    }
```

## Service Layer Domain Model

```mermaid
erDiagram
    HSGMemory ||--o{ SectorEmbedding : "has multiple"
    HSGMemory ||--o{ Waypoint : "links via"
    HSGMemory }o--|| Sector : "classified into"
    HSGMemory }o--|| MemoryTier : "belongs to"
    User ||--o{ HSGMemory : "owns"
    User ||--|| UserSummary : "has"
    UserSummary ||--o{ ClusterPattern : "contains"

    HSGMemory {
        string id
        string content
        string primary_sector
        array sectors
        array tags
        object metadata
        number created_at
        number updated_at
        number last_seen_at
        number salience
        number decay_lambda
        number version
    }

    SectorEmbedding {
        string memory_id
        string sector_type
        array vector
        number dimension
    }

    Waypoint {
        string src_id
        string dst_id
        number weight
        number created_at
        number updated_at
    }

    Sector {
        string name
        string description
        number decay_lambda
        number weight
        array patterns
    }

    MemoryTier {
        string tier_name
        number lambda
        string compression_level
    }

    User {
        string user_id
        number created_at
    }

    UserSummary {
        string user_id
        number total_memories
        number pattern_count
        string activity_level
        number avg_salience
    }

    ClusterPattern {
        string sector
        number count
        number salience
        string snippet
    }
```

## API/SDK Layer Model

```mermaid
erDiagram
    AddMemoryRequest ||--|| AddMemoryResponse : "creates"
    QueryRequest ||--|| QueryResponse : "returns"
    QueryResponse ||--o{ QueryMatch : "contains"
    Memory ||--|| QueryMatch : "extends"
    IngestRequest ||--|| IngestResponse : "produces"

    AddMemoryRequest {
        string content "REQUIRED"
        array tags
        object metadata
        number salience
        number decay_lambda
        string user_id
    }

    AddMemoryResponse {
        string id
        string primary_sector
        array sectors
    }

    QueryRequest {
        string query "REQUIRED"
        number k
        object filters
    }

    QueryResponse {
        string query
        array matches
    }

    Memory {
        string id
        string content
        string primary_sector
        array sectors
        string tags
        object metadata
        number created_at
        number updated_at
        number last_seen_at
        number salience
        number decay_lambda
        number version
    }

    QueryMatch {
        string id
        string content
        number score
        array path
        number salience
    }

    IngestRequest {
        string source
        string content_type
        string data
        object metadata
        object config
        string user_id
    }

    IngestResponse {
        number count
        array memory_ids
        string status
    }
```

## LangGraph Integration Model

```mermaid
erDiagram
    LangGraphNode ||--o{ LangGraphMemory : "stores"
    LangGraphMemory }o--|| HSGMemory : "backed by"
    LangGraphGraph ||--o{ LangGraphNode : "contains"
    LangGraphNamespace ||--o{ LangGraphGraph : "groups"

    LangGraphNode {
        string node_id
        string graph_id
        string namespace
    }

    LangGraphMemory {
        string memory_id
        string node_id
        string content
        array tags
        object metadata
        boolean reflective
    }

    LangGraphGraph {
        string graph_id
        string namespace
        number created_at
    }

    LangGraphNamespace {
        string namespace
        string description
    }

    HSGMemory {
        string id
        string content
        object metadata
    }
```

## IDE Integration Model

```mermaid
erDiagram
    IDESession ||--o{ IDEEvent : "contains"
    IDEEvent ||--|| HSGMemory : "creates"
    IDESession ||--|| SessionSummary : "generates"
    IDEEvent }o--|| EventType : "is of"

    IDESession {
        string session_id
        string user_id
        string project_name
        string ide_name
        number start_time
        number end_time
    }

    IDEEvent {
        string event_id
        string session_id
        string event_type
        string file_path
        string content
        object metadata
        number timestamp
    }

    EventType {
        string type_name
        string description
    }

    SessionSummary {
        string session_id
        number total_events
        object sectors_distribution
        array files_touched
        number unique_files
    }

    HSGMemory {
        string id
        string content
        object metadata
    }
```

## UI Dashboard Data Flow

```mermaid
erDiagram
    Dashboard ||--o{ StatCard : "displays"
    Dashboard ||--o{ ActivityLog : "shows"
    Dashboard ||--o{ HealthMetric : "monitors"
    MemoryList ||--o{ UIMemory : "renders"
    UIMemory }o--|| Memory : "represents"

    Dashboard {
        object stats
        array logs
        object health
        array qps_data
    }

    StatCard {
        string label
        number value
        string unit
        string trend
    }

    ActivityLog {
        string time
        string event
        string sector
        number salience
        string level
    }

    HealthMetric {
        string metric_name
        number value
        string status
        string threshold
    }

    MemoryList {
        array memories
        string filter
        number page
        string search
    }

    UIMemory {
        string id
        string content
        string primary_sector
        array tags
        number salience
        number created_at
    }

    Memory {
        string id
        string content
        object metadata
    }
```

## Complete System Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        SDK[SDK/Client]
        Dashboard[Dashboard UI]
        IDE[IDE Extension]
    end

    subgraph "API Layer"
        REST[REST API]
        MCP[MCP Server]
        LG[LangGraph API]
    end

    subgraph "Service Layer"
        HSG[HSG Core]
        Decay[Decay Engine]
        Summary[User Summary]
        Embed[Embedding Service]
        Classify[Sector Classifier]
    end

    subgraph "Data Layer"
        Memories[(Memories Table)]
        Vectors[(Vectors Table)]
        Waypoints[(Waypoints Table)]
        Users[(Users Table)]
        Logs[(Embed Logs)]
        Stats[(Stats Table)]
    end

    SDK --> REST
    Dashboard --> REST
    IDE --> REST

    REST --> HSG
    REST --> Summary
    MCP --> HSG
    LG --> HSG

    HSG --> Classify
    HSG --> Embed
    HSG --> Decay

    HSG --> Memories
    HSG --> Vectors
    HSG --> Waypoints
    Embed --> Vectors
    Embed --> Logs
    Summary --> Users
    Decay --> Memories
    Decay --> Stats

    Classify -.-> Embed
    Decay -.-> HSG
```

## Data Flow: Add Memory

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant HSG
    participant Classifier
    participant Embedder
    participant DB

    Client->>API: POST /memory/add {content, user_id}
    API->>HSG: add_hsg_memory()
    HSG->>Classifier: classify_sectors(content)
    Classifier-->>HSG: {primary, additional}

    loop For each sector
        HSG->>Embedder: generate_embedding(content, sector)
        Embedder-->>HSG: vector
    end

    HSG->>DB: INSERT memories
    HSG->>DB: INSERT vectors (per sector)
    HSG->>DB: UPDATE waypoints
    DB-->>HSG: success

    HSG-->>API: {id, primary_sector, sectors}
    API-->>Client: Memory response

    API->>Summary: update_user_summary(user_id) [async]
```

## Data Flow: Query Memory

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant HSG
    participant Classifier
    participant VectorDB
    participant Graph

    Client->>API: POST /memory/query {query, k}
    API->>HSG: hsg_query()
    HSG->>Classifier: classify_sectors(query)
    Classifier-->>HSG: [sector1, sector2]

    loop For each sector
        HSG->>VectorDB: vector_search(query_vec, sector)
        VectorDB-->>HSG: top_k matches
    end

    HSG->>HSG: aggregate_results()
    HSG->>Graph: expand_waypoints(matches)
    Graph-->>HSG: expanded_matches

    HSG->>HSG: composite_scoring()
    HSG-->>API: sorted QueryMatch[]
    API-->>Client: {query, matches}
```

## Data Flow: Decay Cycle

```mermaid
sequenceDiagram
    participant Scheduler
    participant Decay
    participant DB
    participant Compressor

    Scheduler->>Decay: run_decay_cycle()
    Decay->>DB: SELECT all memories
    DB-->>Decay: memory[]

    loop For each memory
        Decay->>Decay: classify_tier(memory)
        alt Hot
            Decay->>DB: UPDATE salience (λ=0.005)
        else Warm
            Decay->>DB: UPDATE salience (λ=0.02)
        else Cold
            Decay->>DB: UPDATE salience (λ=0.05)
            Decay->>Compressor: compress_vector()
            Compressor-->>Decay: compressed_vec
            Decay->>DB: UPDATE compressed_vec
        end
    end

    Decay->>DB: DELETE waypoints WHERE weight < threshold
    Decay->>DB: INSERT stats {type: 'decay', count}
    Decay-->>Scheduler: {updated: N}
```

---

## Entity Cardinalities

### One-to-Many Relationships

1. **users → memories**: One user has many memories
2. **users → waypoints**: One user has many waypoints
3. **memories → vectors**: One memory has multiple sector embeddings
4. **memories → waypoints (src)**: One memory is source of many waypoints
5. **memories → waypoints (dst)**: One memory is destination of many waypoints

### One-to-One Relationships

1. **memories ↔ embed_logs**: Optional tracking log per memory
2. **users ↔ user summary**: One generated summary per user

### Many-to-Many Relationships

1. **memories ↔ memories (via waypoints)**: Memories form a directed graph
2. **memories ↔ sectors**: Memories can belong to multiple sectors

---

## Indexes and Performance

### Primary Indexes
- `memories(id)` - UUID primary key
- `vectors(id, sector)` - Composite primary key
- `waypoints(src_id, user_id)` - Composite primary key
- `users(user_id)` - Text primary key

### Secondary Indexes
- `memories(user_id)` - User isolation
- `memories(primary_sector)` - Sector filtering
- `memories(simhash)` - Deduplication
- `memories(last_seen_at)` - Recency queries
- `vectors(user_id)` - User isolation
- `waypoints(src_id)` - Graph traversal
- `waypoints(dst_id)` - Reverse graph traversal
- `waypoints(user_id)` - User isolation
- `stats(type, ts)` - Analytics queries

### Query Optimization Patterns

1. **Sector-based sharding**: Partition vector search by sector
2. **User scoping**: Always filter by user_id in multi-tenant
3. **Waypoint pruning**: Delete low-weight edges periodically
4. **Vector compression**: Reduce cold memory dimensions
5. **Batch operations**: Process decay in parallel chunks

---

## Constraints

### Primary Key Constraints
- `memories.id` must be unique UUID
- `vectors(id, sector)` must be unique pair
- `waypoints(src_id, user_id)` must be unique pair
- `users.user_id` must be unique

### Foreign Key Constraints (Logical)
- `vectors.id` references `memories.id`
- `waypoints.src_id` references `memories.id`
- `waypoints.dst_id` references `memories.id`
- `memories.user_id` references `users.user_id`
- `waypoints.user_id` references `users.user_id`

### Data Constraints
- `memories.content` NOT NULL
- `memories.primary_sector` NOT NULL
- `vectors.v` NOT NULL (embedding vector)
- `vectors.dim` NOT NULL
- `waypoints.weight` between 0.0 and 1.0
- `memories.salience` between 0.0 and 1.0

### Business Logic Constraints
- Each memory must have at least 1 sector embedding
- Each memory has exactly 1 primary sector
- Waypoints are single-per-source (one strongest link)
- User summaries regenerated every 30 minutes

---

**Diagram Format:** Mermaid
**Last Updated:** 2025-11-09
**Version:** 2.1.0

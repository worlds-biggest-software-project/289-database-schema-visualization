# Data Model Suggestion 4: Graph-Based Storage (Property Graph)

> Project: Database Schema Visualization (Candidate #289)
> Approach: Property graph database for schema relationships and ERD traversal

## Summary

This approach stores the schema metadata itself as a property graph, where database tables are nodes and relationships (foreign keys, inferred references, data lineage) are edges. A graph database is a natural fit for a schema visualization tool because the core domain -- entities connected by relationships -- is literally a graph. ERD rendering, relationship traversal, impact analysis, and "which tables are involved in X?" queries all map directly to graph traversal operations.

The design uses a graph database (Neo4j, Apache AGE on PostgreSQL, or Memgraph) as the primary store for schema metadata and relationships, with PostgreSQL handling user management, project configuration, collaboration, and versioning. This hybrid architecture plays to each database's strengths: graphs for the domain model, relational for the operational model.

---

## Key Entities and Relationships

### Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                  PostgreSQL                          │
│  organizations, users, projects, db_connections,     │
│  collaboration (comments, activity), layouts,        │
│  schema_snapshots (version metadata only)            │
└──────────────────────┬──────────────────────────────┘
                       │ project_id / snapshot_id references
                       v
┌─────────────────────────────────────────────────────┐
│              Graph Database (Neo4j / AGE)             │
│  :Schema, :Table, :Column, :Index, :Constraint       │
│  -[:HAS_TABLE]-> -[:HAS_COLUMN]-> -[:REFERENCES]->   │
│  -[:DEPENDS_ON]-> -[:LINEAGE]-> -[:SIMILAR_TO]->     │
└─────────────────────────────────────────────────────┘
```

### Graph Model (Nodes and Edges)

```
Node Labels:
  (:Project       { id, name, engine })
  (:Schema        { id, project_id, snapshot_version, name })
  (:Table         { id, project_id, snapshot_version, schema_name, name, type, 
                    description, ai_description, row_count_est })
  (:Column        { id, table_id, name, data_type, ordinal, nullable, 
                    default_value, is_pk, is_unique, description, ai_description })
  (:Index         { id, table_id, name, type, is_unique, columns })
  (:Constraint    { id, table_id, name, type, definition })
  (:Tag           { id, project_id, name, color })
  (:QueryExample  { id, project_id, sql_text, description, avg_duration_ms })

Relationship Types:
  (:Project)-[:HAS_SCHEMA { snapshot_version }]->(:Schema)
  (:Schema)-[:HAS_TABLE]->(:Table)
  (:Table)-[:HAS_COLUMN]->(:Column)
  (:Table)-[:HAS_INDEX]->(:Index)
  (:Table)-[:HAS_CONSTRAINT]->(:Constraint)
  
  -- The core value: foreign key and inferred relationships as first-class edges
  (:Column)-[:REFERENCES { constraint_name, on_delete, on_update, is_inferred }]->(:Column)
  
  -- Derived / AI-detected relationships
  (:Table)-[:DEPENDS_ON { reason }]->(:Table)         -- data dependency
  (:Table)-[:SIMILAR_TO { similarity_score }]->(:Table) -- AI-detected structural similarity
  (:Column)-[:LINEAGE { transform }]->(:Column)        -- data lineage tracking
  (:Table)-[:TAGGED]->(:Tag)
  (:QueryExample)-[:INVOLVES]->(:Table)
  
  -- Schema evolution edges (cross-snapshot)
  (:Table)-[:EVOLVED_FROM { changes }]->(:Table)       -- same table across versions
  (:Column)-[:EVOLVED_FROM { changes }]->(:Column)     -- same column across versions
```

### Cypher Examples (Neo4j)

```cypher
// 1. Load the full ERD for a project snapshot (all tables and their relationships)
MATCH (s:Schema {project_id: $projectId, snapshot_version: $version})-[:HAS_TABLE]->(t:Table)
OPTIONAL MATCH (t)-[:HAS_COLUMN]->(c:Column)
OPTIONAL MATCH (c)-[r:REFERENCES]->(tc:Column)<-[:HAS_COLUMN]-(tt:Table)
RETURN t, collect(DISTINCT c) AS columns, 
       collect(DISTINCT {rel: r, target_col: tc, target_table: tt}) AS relationships

// 2. Find all tables reachable from 'orders' within 3 hops (impact analysis)
MATCH path = (start:Table {name: 'orders', project_id: $projectId})
              -[:HAS_COLUMN]->(:Column)-[:REFERENCES*1..3]->(:Column)<-[:HAS_COLUMN]-(related:Table)
RETURN DISTINCT related.name AS table_name, length(path) AS distance
ORDER BY distance

// 3. "Which tables are involved in the order fulfilment flow?" (AI-assisted)
MATCH (t:Table {project_id: $projectId})
WHERE t.name IN ['orders', 'order_items', 'shipments', 'payments', 'invoices']
MATCH path = (t)-[:HAS_COLUMN]->(:Column)-[:REFERENCES]-(:Column)<-[:HAS_COLUMN]-(connected:Table)
WHERE connected.project_id = $projectId
RETURN DISTINCT t, connected, relationships(path)

// 4. Detect orphan tables (no incoming or outgoing foreign keys)
MATCH (t:Table {project_id: $projectId, snapshot_version: $version})
WHERE NOT (t)-[:HAS_COLUMN]->(:Column)-[:REFERENCES]->()
  AND NOT (:Column)-[:REFERENCES]->(:Column)<-[:HAS_COLUMN]-(t)
RETURN t.schema_name, t.name AS orphan_table

// 5. Track how a table evolved across schema versions
MATCH path = (current:Table {name: 'users', snapshot_version: $currentVersion})
              -[:EVOLVED_FROM*]->(ancestor:Table)
WHERE current.project_id = $projectId
RETURN [node IN nodes(path) | {version: node.snapshot_version, name: node.name}] AS evolution,
       [rel IN relationships(path) | rel.changes] AS changes

// 6. Find tables with potential many-to-many relationships (anti-pattern detection)
MATCH (t:Table {project_id: $projectId})-[:HAS_COLUMN]->(c:Column)-[:REFERENCES]->(tc:Column)
WITH t, count(DISTINCT tc) AS fk_count
WHERE fk_count >= 2
  AND NOT EXISTS { MATCH (t)-[:HAS_COLUMN]->(non_fk:Column) 
                   WHERE non_fk.is_pk = false AND NOT (non_fk)-[:REFERENCES]->() }
RETURN t.name AS junction_table, fk_count

// 7. Data lineage: trace a column's origin across tables
MATCH path = (c:Column {name: 'total_revenue'})<-[:HAS_COLUMN]-(t:Table)
              -[:DEPENDS_ON*1..5]->(source:Table)-[:HAS_COLUMN]->(sc:Column)
WHERE t.project_id = $projectId
RETURN nodes(path), relationships(path)
```

### PostgreSQL Side (Operational Data)

```sql
-- Projects, users, and collaboration remain in PostgreSQL
-- (same as Suggestion 1 for these tables)

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255),
    auth_provider   VARCHAR(50) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    database_engine VARCHAR(50),
    graph_db_ref    VARCHAR(255),             -- reference to the graph database/namespace
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Schema snapshot metadata (the actual schema data lives in the graph)
CREATE TABLE schema_snapshot_meta (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    version         INTEGER NOT NULL,
    label           VARCHAR(255),
    source          VARCHAR(50) NOT NULL,
    table_count     INTEGER,
    relationship_count INTEGER,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, version)
);

-- Layouts stored in PostgreSQL (visual state, not schema structure)
CREATE TABLE layouts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    snapshot_version INTEGER,
    name            VARCHAR(255) NOT NULL,
    visual_state    JSONB NOT NULL DEFAULT '{}',  -- positions, canvas settings, groups
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Comments, annotations, activity log in PostgreSQL
CREATE TABLE comments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    target_path     VARCHAR(500),
    parent_id       UUID REFERENCES comments(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    body            TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE annotations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    target_path     VARCHAR(500) NOT NULL,
    annotation_type VARCHAR(50) NOT NULL,
    content         TEXT NOT NULL,
    severity        VARCHAR(20),
    is_ai_generated BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Apache AGE Alternative (Graph on PostgreSQL)

For teams that want graph capabilities without a separate database, Apache AGE adds graph query support directly inside PostgreSQL:

```sql
-- Create a graph namespace for a project
SELECT create_graph('project_289');

-- Load a table node
SELECT * FROM cypher('project_289', $$
    CREATE (t:Table {
        id: 'uuid-here',
        snapshot_version: 1,
        schema_name: 'public',
        name: 'users',
        description: 'User accounts',
        row_count_est: 150000
    })
    RETURN t
$$) AS (t agtype);

-- Create a REFERENCES edge between columns
SELECT * FROM cypher('project_289', $$
    MATCH (sc:Column {name: 'user_id', table_name: 'orders'}),
          (tc:Column {name: 'id', table_name: 'users'})
    CREATE (sc)-[:REFERENCES {
        constraint_name: 'fk_orders_user',
        on_delete: 'CASCADE',
        is_inferred: false
    }]->(tc)
$$) AS (result agtype);

-- Impact analysis: find all tables affected by changing 'users'
SELECT * FROM cypher('project_289', $$
    MATCH (start:Table {name: 'users'})-[:HAS_COLUMN]->(c:Column)
          <-[:REFERENCES*1..4]-(affected:Column)<-[:HAS_COLUMN]-(t:Table)
    RETURN DISTINCT t.name AS affected_table, 
           length(relationships(shortestPath((start)-[*]-(t)))) AS distance
    ORDER BY distance
$$) AS (affected_table agtype, distance agtype);
```

---

## Pros

- **Domain-native data model**: A database schema IS a graph (entities connected by relationships). Storing it as a graph eliminates the impedance mismatch between the domain and the storage layer.
- **Relationship traversal is first-class**: "Find all tables reachable from X" is a single Cypher query, not a recursive CTE. Multi-hop traversal, impact analysis, and dependency graphs are natural operations.
- **Powerful AI-integration surface**: Graph structure enables AI features like anti-pattern detection (find junction tables, detect circular dependencies), relationship inference (suggest missing foreign keys), and natural-language schema exploration.
- **Data lineage**: Lineage tracking (column A in table X derives from column B in table Y) is a graph edge, not a complex relational mapping.
- **Cross-version evolution**: `EVOLVED_FROM` edges connect the same logical table across schema versions, enabling rich change visualization without snapshot diffing.
- **Flexible schema extension**: Adding new relationship types (e.g., `:SIMILAR_TO` for AI-detected structural similarity) requires no schema migration -- just create edges with a new label.
- **Visual rendering alignment**: The graph model directly maps to ERD visualization. Nodes become table boxes, edges become relationship lines. The rendering pipeline reads graph data natively.

## Cons

- **Operational complexity**: Running two databases (PostgreSQL + Neo4j/Memgraph) increases deployment, monitoring, and backup complexity significantly.
- **Consistency across stores**: Keeping PostgreSQL project metadata and graph schema data in sync requires careful transactional coordination or eventual consistency.
- **Smaller talent pool**: Cypher/GQL is less widely known than SQL. Onboarding developers takes longer.
- **Limited aggregation**: Graph databases are optimized for traversal, not aggregation. Queries like "count tables per schema across all projects" are faster in PostgreSQL.
- **Cost**: Neo4j Enterprise or Aura is more expensive than PostgreSQL alone. Memgraph and Apache AGE reduce cost but have smaller ecosystems.
- **Backup and migration**: Graph database backup/restore tooling is less mature than PostgreSQL's. Migrating between graph vendors is non-trivial despite the emerging GQL standard.
- **Apache AGE maturity**: If using the single-database AGE approach, AGE is still a relatively young extension with a smaller community than Neo4j.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Graph database | **Option A**: Neo4j Community/Aura (most mature, largest ecosystem) |
|                | **Option B**: Apache AGE on PostgreSQL (single database, simpler ops) |
|                | **Option C**: Memgraph (in-memory, fastest traversal, Cypher-compatible) |
| Relational database | PostgreSQL 16+ for operational data |
| Graph ORM | `neo4j-driver` (official JS/Python), or `ogm` (Neo4j Object Graph Mapper) |
| Query language | Cypher (Neo4j, AGE, Memgraph) or emerging GQL (ISO/IEC 39075) standard |
| Visualization | Direct graph-to-SVG rendering via D3.js or yFiles (graph data maps 1:1 to visual elements) |
| Sync layer | Application-level dual writes, or Change Data Capture (Debezium) from PostgreSQL to graph |
| Hosting | Neo4j Aura (managed); or PostgreSQL + AGE on any PostgreSQL host |

---

## Migration and Scaling Considerations

- **Migration from relational**: Export tables as nodes and foreign keys as edges using a bulk import script. Neo4j's `neo4j-admin database import` handles millions of nodes/edges efficiently. For AGE, use Cypher `CREATE` statements wrapped in a migration script.
- **Scaling the graph**: Neo4j Aura scales vertically to very large graphs (billions of nodes). For horizontal scaling, consider Neo4j Fabric (federated queries across shards) or partition by project using separate graph namespaces.
- **Cross-database transactions**: Use the Saga pattern or outbox pattern to coordinate writes between PostgreSQL and the graph database. For AGE, this is unnecessary since both live in the same PostgreSQL instance.
- **Fallback strategy**: The graph data can always be materialized back to relational tables (Suggestion 1) or JSONB documents (Suggestion 3) if the graph database introduces unacceptable operational burden. The PostgreSQL side already stores all non-schema data relationally.
- **GQL standard adoption**: The ISO GQL standard (ISO/IEC 39075) is emerging as the SQL equivalent for graph databases. Building on Cypher today provides a migration path to GQL as implementations mature.
- **Apache AGE as a stepping stone**: Start with AGE to validate the graph model within the existing PostgreSQL deployment. If graph query performance becomes a bottleneck at scale, migrate the graph layer to a dedicated Neo4j or Memgraph instance while keeping the same Cypher queries.

# Data Model Suggestion 2: Event-Sourced / CQRS Approach

> Project: Database Schema Visualization (Candidate #289)
> Approach: Event Sourcing with Command Query Responsibility Segregation

## Summary

This approach stores every change to a schema visualization project as an immutable event in an append-only event store. The current state of any project, schema snapshot, or layout is derived by replaying the event stream. Read-optimized projections (materialized views) are maintained separately for fast diagram rendering, search, and analytics.

This model is a natural fit for a schema visualization tool because:
1. **Schema change history is a first-class feature** -- the tool needs to show what changed between versions, when, and by whom.
2. **Time-travel visualization** is a stated backlog feature -- replaying events to any point in time is trivially supported.
3. **Collaboration** benefits from event streams -- concurrent edits resolve naturally via event ordering rather than row-level locks.
4. **AI changelogs** are generated directly from event data rather than computed by diffing snapshots.

---

## Key Entities and Relationships

### Architecture Overview

```
                  ┌───────────────┐
  Commands ──────>│  Command Bus  │
                  └───────┬───────┘
                          │ validates & produces
                          v
                  ┌───────────────┐         ┌─────────────────────┐
                  │  Event Store  │────────> │  Event Handlers /   │
                  │  (append-only)│         │  Projectors          │
                  └───────────────┘         └──────────┬──────────┘
                                                       │ materialize
                                                       v
                                            ┌─────────────────────┐
                                            │  Read Models (CQRS) │
                                            │  - ERD Projection   │
                                            │  - Search Index     │
                                            │  - Change Timeline  │
                                            │  - Analytics View   │
                                            └─────────────────────┘
```

### Event Store Schema

```sql
-- Core event store (append-only, never updated or deleted)
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,               -- aggregate root ID (project_id)
    stream_type     VARCHAR(100) NOT NULL,        -- 'Project', 'SchemaSnapshot', 'Layout'
    event_type      VARCHAR(200) NOT NULL,        -- e.g., 'TableAdded', 'ColumnRenamed', 'RelationshipDetected'
    event_version   INTEGER NOT NULL,             -- per-stream sequence number
    payload         JSONB NOT NULL,               -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',  -- causation_id, correlation_id, user_id, ip, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, event_version)
);

-- Optimistic concurrency: the UNIQUE constraint on (stream_id, event_version)
-- prevents two concurrent writers from appending at the same position.

CREATE INDEX idx_events_stream ON events(stream_id, event_version);
CREATE INDEX idx_events_type ON events(event_type);
CREATE INDEX idx_events_created ON events(created_at);

-- Snapshot store (for fast aggregate rehydration -- avoids replaying thousands of events)
CREATE TABLE event_snapshots (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(100) NOT NULL,
    snapshot_version INTEGER NOT NULL,           -- event_version at time of snapshot
    state           JSONB NOT NULL,              -- serialized aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);
```

### Event Types Catalog

```
Project Domain:
  ProjectCreated          { name, org_id, database_engine }
  ProjectRenamed          { old_name, new_name }
  ProjectDeleted          { }
  MemberAdded             { user_id, role }
  MemberRemoved           { user_id }
  ConnectionConfigured    { connection_id, engine, host, database }

Schema Domain:
  SchemaImported          { source, table_count, snapshot_version }
  TableAdded              { schema, table_name, table_type, columns[] }
  TableDropped            { schema, table_name }
  TableRenamed            { schema, old_name, new_name }
  ColumnAdded             { table_ref, column_name, data_type, nullable, default }
  ColumnDropped           { table_ref, column_name }
  ColumnModified          { table_ref, column_name, changes: { type?, nullable?, default? } }
  ColumnRenamed           { table_ref, old_name, new_name }
  RelationshipAdded       { source_table, source_column, target_table, target_column, type }
  RelationshipRemoved     { relationship_ref }
  IndexAdded              { table_ref, index_name, columns[], index_type, is_unique }
  IndexDropped            { table_ref, index_name }
  ConstraintAdded         { table_ref, constraint_name, constraint_type, definition }
  ConstraintDropped       { table_ref, constraint_name }

Layout Domain:
  LayoutCreated           { layout_name, algorithm }
  TablePositioned         { layout_ref, table_ref, x, y, width, height }
  TableCollapsed          { layout_ref, table_ref }
  TableExpanded           { layout_ref, table_ref }
  CanvasSettingsChanged   { zoom, pan_x, pan_y, grid }

Annotation Domain:
  AnnotationAdded         { target_type, target_ref, content, severity }
  AnnotationRemoved       { annotation_ref }
  TagCreated              { tag_name, color }
  TableTagged             { table_ref, tag_name }
  TableUntagged           { table_ref, tag_name }

AI Domain:
  AiDescriptionGenerated  { target_type, target_ref, description, model_version }
  AntiPatternDetected     { pattern_type, table_refs[], description, severity }
  MigrationPlanGenerated  { plan_id, steps[], description }
  SchemaComparisonGenerated { from_version, to_version, changelog_text }

Collaboration Domain:
  CommentAdded            { target_type, target_ref, body, parent_comment_id? }
  CommentEdited           { comment_ref, new_body }
  CommentDeleted          { comment_ref }
```

### Read Model Projections

```sql
-- PROJECTION 1: Current schema state (materialized for fast ERD rendering)
CREATE TABLE proj_current_schema (
    project_id      UUID NOT NULL,
    schema_name     VARCHAR(255) NOT NULL,
    table_name      VARCHAR(255) NOT NULL,
    table_type      VARCHAR(50),
    columns         JSONB NOT NULL,          -- [{name, type, nullable, pk, description}, ...]
    row_count_est   BIGINT,
    description     TEXT,
    ai_description  TEXT,
    tags            TEXT[],
    last_event_version INTEGER NOT NULL,     -- for projection consistency tracking
    PRIMARY KEY (project_id, schema_name, table_name)
);

-- PROJECTION 2: Relationships (materialized for diagram edge rendering)
CREATE TABLE proj_relationships (
    project_id          UUID NOT NULL,
    source_schema       VARCHAR(255),
    source_table        VARCHAR(255) NOT NULL,
    source_column       VARCHAR(255) NOT NULL,
    target_schema       VARCHAR(255),
    target_table        VARCHAR(255) NOT NULL,
    target_column       VARCHAR(255) NOT NULL,
    relationship_type   VARCHAR(50) NOT NULL,
    is_inferred         BOOLEAN NOT NULL DEFAULT false,
    constraint_name     VARCHAR(255),
    last_event_version  INTEGER NOT NULL,
    PRIMARY KEY (project_id, source_table, source_column, target_table, target_column)
);

-- PROJECTION 3: Layout state (materialized for canvas rendering)
CREATE TABLE proj_layouts (
    project_id      UUID NOT NULL,
    layout_id       UUID NOT NULL,
    layout_name     VARCHAR(255),
    positions       JSONB NOT NULL,          -- { "public.users": {x, y, w, h, collapsed}, ... }
    canvas_settings JSONB,
    last_event_version INTEGER NOT NULL,
    PRIMARY KEY (project_id, layout_id)
);

-- PROJECTION 4: Change timeline (materialized for history view)
CREATE TABLE proj_change_timeline (
    project_id      UUID NOT NULL,
    event_id        UUID NOT NULL,
    event_type      VARCHAR(200) NOT NULL,
    summary         TEXT NOT NULL,           -- human-readable: "Added column 'email' to users"
    user_id         UUID,
    user_name       VARCHAR(255),
    occurred_at     TIMESTAMPTZ NOT NULL,
    affected_tables TEXT[],
    PRIMARY KEY (project_id, occurred_at, event_id)
);

-- PROJECTION 5: Search index
CREATE TABLE proj_search_index (
    project_id      UUID NOT NULL,
    object_type     VARCHAR(50) NOT NULL,    -- 'table', 'column', 'relationship'
    object_name     VARCHAR(500) NOT NULL,
    qualified_name  VARCHAR(1000) NOT NULL,  -- 'public.users.email'
    description     TEXT,
    search_vector   TSVECTOR,
    PRIMARY KEY (project_id, object_type, qualified_name)
);

CREATE INDEX idx_search_vector ON proj_search_index USING GIN(search_vector);
```

---

## Pros

- **Perfect for change history**: Schema diff, time-travel visualization, and AI changelogs are derived directly from the event stream. No need to compute diffs by comparing snapshot copies.
- **Complete audit trail**: Every action is recorded with who, what, when, and why. Compliance-ready by default.
- **Time-travel is free**: Replaying events to any point reconstructs the schema state at that moment. The "time-travel schema visualization" backlog item is trivially implemented.
- **Concurrent collaboration**: Multiple users editing different parts of a schema produce independent events that merge naturally through event ordering. No row-level lock contention.
- **AI-friendly**: The event stream is a structured narrative of what happened. AI models can read recent events to generate contextual annotations, detect anti-patterns from change patterns, and produce changelogs.
- **Flexible projections**: New read models can be added without changing the write path. Need a "most-modified tables" dashboard? Build a new projector over the existing event stream.
- **Undo/redo**: Compensating events enable natural undo at the domain level.

## Cons

- **Complexity**: Event sourcing adds significant architectural complexity compared to CRUD. The team needs to understand event design, projector consistency, and eventual consistency between write and read sides.
- **Eventual consistency**: Read models lag behind writes (typically milliseconds, but it must be designed for). Users may briefly see stale data after making a change.
- **Event schema evolution**: Events are immutable and stored forever. Changing an event's schema requires upcasting (transforming old events to a new format on read), which is the highest-cost maintenance operation.
- **Storage growth**: The event store grows monotonically. A busy project with frequent schema changes accumulates thousands of events. Periodic snapshots and event archival are necessary.
- **Debugging difficulty**: The current state is computed, not stored. Investigating a bug requires tracing through the event stream rather than querying a row.
- **Projection rebuild time**: If a projector has a bug or a new projection is added, rebuilding from the full event stream can take significant time for large projects.
- **Overkill for small installations**: A single-user desktop deployment gains little from CQRS and pays the full complexity cost.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Event store | PostgreSQL (simple, with the schema above) or EventStoreDB/Kurrent (purpose-built) |
| Message bus | PostgreSQL LISTEN/NOTIFY for simple setups; NATS or Apache Kafka for high-throughput |
| Projector framework | Custom projectors in TypeScript/Python, or Marten (.NET) |
| Read database | PostgreSQL for projections + Meilisearch/Typesense for full-text search |
| Caching | Redis for hot projection data (current layout, active schema) |
| Serialization | JSON (JSONB in PostgreSQL) for event payloads; Avro or Protobuf for high-volume streams |
| Hosting | PostgreSQL on Supabase/Neon; Kafka on Confluent Cloud or Redpanda if needed |

---

## Migration and Scaling Considerations

- **Initial migration from relational**: The current relational state can be captured as a set of "genesis events" (e.g., `SchemaImported` containing the full initial state). All subsequent changes are then appended as granular events.
- **Event store partitioning**: Partition the `events` table by `stream_id` (project) for large multi-tenant installations. Each project's event stream is independent.
- **Projection scaling**: Projections can be rebuilt independently. If the search projection falls behind, it can be rebuilt from the event store without affecting the ERD projection.
- **Event archival**: Events older than a configurable threshold (e.g., 2 years) can be archived to cold storage (S3/GCS) after creating a snapshot. The snapshot serves as the new starting point for replay.
- **Multi-region**: Event streams can be replicated across regions using PostgreSQL logical replication or Kafka mirroring. Each region maintains its own projections.
- **Gradual adoption**: CQRS can be adopted incrementally -- start with a normalized model (Suggestion 1) and add event logging as a secondary write. Once proven, flip to event-sourced as the source of truth.

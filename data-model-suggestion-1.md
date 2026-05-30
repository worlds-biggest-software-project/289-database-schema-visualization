# Data Model Suggestion 1: Normalized Relational (PostgreSQL)

> Project: Database Schema Visualization (Candidate #289)
> Approach: Traditional normalized relational schema using PostgreSQL

## Summary

This approach stores all schema visualization metadata in a fully normalized PostgreSQL schema. Every concept -- projects, database connections, schemas, tables, columns, relationships, layout positions, annotations, versions, and collaboration artifacts -- is modeled as a dedicated table with strict foreign key constraints. This is the most straightforward approach and mirrors how tools like Vertabelo and DrawSQL store their internal state.

The normalized model treats the schema-visualization tool's own internal database as a "schema about schemas" (a meta-schema), using the same relational principles the tool helps users visualize.

---

## Key Entities and Relationships

### Entity Overview

```
organizations ─┬─> projects ─┬─> schema_snapshots ─┬─> db_tables ──┬─> db_columns
                │             │                      │               └─> db_indexes
                │             │                      ├─> db_relationships
                │             │                      └─> db_constraints
                │             ├─> db_connections
                │             ├─> layouts ──────────> layout_positions
                │             └─> annotations
                │
                └─> users ──> project_members
                            > comments
                            > change_history
```

### Core Schema (DDL)

```sql
-- Multi-tenancy and user management
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255),
    avatar_url      TEXT,
    auth_provider   VARCHAR(50) NOT NULL,     -- 'github', 'google', 'email'
    auth_provider_id VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization_members (
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',  -- 'owner', 'admin', 'member', 'viewer'
    PRIMARY KEY (organization_id, user_id)
);

-- Projects (one per database/schema being visualized)
CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    database_engine VARCHAR(50),  -- 'postgresql', 'mysql', 'sqlserver', 'mongodb', etc.
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE project_members (
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'editor',
    PRIMARY KEY (project_id, user_id)
);

-- Database connections (encrypted credentials)
CREATE TABLE db_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    label           VARCHAR(255),
    engine          VARCHAR(50) NOT NULL,
    host            TEXT,                    -- encrypted at rest
    port            INTEGER,
    database_name   VARCHAR(255),
    username        TEXT,                    -- encrypted at rest
    encrypted_password BYTEA,               -- AES-256 encrypted
    ssl_mode        VARCHAR(50) DEFAULT 'require',
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Schema snapshots (versioned captures of a database schema)
CREATE TABLE schema_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    version         INTEGER NOT NULL,
    label           VARCHAR(255),           -- e.g., 'v2.3.1', 'post-migration-2026-05'
    source          VARCHAR(50) NOT NULL,   -- 'introspection', 'ddl_import', 'dbml_import', 'manual'
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, version)
);

-- Tables within a snapshot
CREATE TABLE db_tables (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    snapshot_id     UUID NOT NULL REFERENCES schema_snapshots(id) ON DELETE CASCADE,
    schema_name     VARCHAR(255) NOT NULL DEFAULT 'public',
    table_name      VARCHAR(255) NOT NULL,
    table_type      VARCHAR(50) DEFAULT 'table',  -- 'table', 'view', 'materialized_view'
    description     TEXT,                   -- user or AI-generated
    ai_description  TEXT,                   -- AI-inferred description
    row_count_est   BIGINT,                 -- estimated row count from statistics
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (snapshot_id, schema_name, table_name)
);

-- Columns within a table
CREATE TABLE db_columns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    table_id        UUID NOT NULL REFERENCES db_tables(id) ON DELETE CASCADE,
    column_name     VARCHAR(255) NOT NULL,
    ordinal_position INTEGER NOT NULL,
    data_type       VARCHAR(255) NOT NULL,
    is_nullable     BOOLEAN NOT NULL DEFAULT true,
    default_value   TEXT,
    is_primary_key  BOOLEAN NOT NULL DEFAULT false,
    is_unique       BOOLEAN NOT NULL DEFAULT false,
    description     TEXT,
    ai_description  TEXT,
    UNIQUE (table_id, column_name)
);

-- Relationships (foreign keys and inferred references)
CREATE TABLE db_relationships (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    snapshot_id         UUID NOT NULL REFERENCES schema_snapshots(id) ON DELETE CASCADE,
    source_table_id     UUID NOT NULL REFERENCES db_tables(id) ON DELETE CASCADE,
    source_column_id    UUID NOT NULL REFERENCES db_columns(id) ON DELETE CASCADE,
    target_table_id     UUID NOT NULL REFERENCES db_tables(id) ON DELETE CASCADE,
    target_column_id    UUID NOT NULL REFERENCES db_columns(id) ON DELETE CASCADE,
    relationship_type   VARCHAR(50) NOT NULL,  -- 'one_to_one', 'one_to_many', 'many_to_many'
    constraint_name     VARCHAR(255),
    is_inferred         BOOLEAN NOT NULL DEFAULT false,  -- true if AI-detected, not a real FK
    on_delete           VARCHAR(50),           -- 'CASCADE', 'SET NULL', 'RESTRICT', etc.
    on_update           VARCHAR(50),
    description         TEXT
);

-- Indexes
CREATE TABLE db_indexes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    table_id        UUID NOT NULL REFERENCES db_tables(id) ON DELETE CASCADE,
    index_name      VARCHAR(255) NOT NULL,
    index_type      VARCHAR(50),            -- 'btree', 'hash', 'gin', 'gist', etc.
    is_unique       BOOLEAN NOT NULL DEFAULT false,
    column_ids      UUID[] NOT NULL          -- ordered array of column IDs
);

-- Constraints
CREATE TABLE db_constraints (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    table_id        UUID NOT NULL REFERENCES db_tables(id) ON DELETE CASCADE,
    constraint_name VARCHAR(255) NOT NULL,
    constraint_type VARCHAR(50) NOT NULL,   -- 'primary_key', 'unique', 'check', 'exclusion'
    definition      TEXT                     -- CHECK expression text
);

-- Layouts (visual diagram arrangements)
CREATE TABLE layouts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    snapshot_id     UUID REFERENCES schema_snapshots(id),
    name            VARCHAR(255) NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    layout_algorithm VARCHAR(50),            -- 'force_directed', 'hierarchical', 'manual'
    canvas_settings JSONB,                   -- zoom level, pan offset, grid settings
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE layout_positions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    layout_id       UUID NOT NULL REFERENCES layouts(id) ON DELETE CASCADE,
    table_id        UUID NOT NULL REFERENCES db_tables(id) ON DELETE CASCADE,
    x               DOUBLE PRECISION NOT NULL,
    y               DOUBLE PRECISION NOT NULL,
    width           DOUBLE PRECISION,
    height          DOUBLE PRECISION,
    is_collapsed    BOOLEAN NOT NULL DEFAULT false,
    color           VARCHAR(20),
    UNIQUE (layout_id, table_id)
);

-- Annotations and tags
CREATE TABLE annotations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    table_id        UUID REFERENCES db_tables(id) ON DELETE CASCADE,
    column_id       UUID REFERENCES db_columns(id) ON DELETE CASCADE,
    annotation_type VARCHAR(50) NOT NULL,    -- 'note', 'warning', 'anti_pattern', 'ai_suggestion'
    content         TEXT NOT NULL,
    severity        VARCHAR(20),             -- 'info', 'warning', 'error'
    created_by      UUID REFERENCES users(id),
    is_ai_generated BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE tags (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,
    color           VARCHAR(20),
    UNIQUE (project_id, name)
);

CREATE TABLE table_tags (
    table_id        UUID NOT NULL REFERENCES db_tables(id) ON DELETE CASCADE,
    tag_id          UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (table_id, tag_id)
);

-- Collaboration
CREATE TABLE comments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    table_id        UUID REFERENCES db_tables(id) ON DELETE CASCADE,
    column_id       UUID REFERENCES db_columns(id) ON DELETE CASCADE,
    relationship_id UUID REFERENCES db_relationships(id) ON DELETE CASCADE,
    parent_id       UUID REFERENCES comments(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    body            TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Change history (diff between snapshots)
CREATE TABLE schema_changes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    from_snapshot_id UUID REFERENCES schema_snapshots(id),
    to_snapshot_id   UUID NOT NULL REFERENCES schema_snapshots(id),
    change_type     VARCHAR(50) NOT NULL,    -- 'table_added', 'table_dropped', 'column_added', etc.
    object_type     VARCHAR(50) NOT NULL,    -- 'table', 'column', 'index', 'relationship'
    object_name     VARCHAR(500) NOT NULL,   -- 'public.users.email'
    old_definition  TEXT,
    new_definition  TEXT,
    ai_summary      TEXT,                    -- AI-generated human-readable change description
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Query examples associated with tables
CREATE TABLE query_examples (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    sql_text        TEXT NOT NULL,
    description     TEXT,
    tables_involved UUID[] NOT NULL,         -- array of db_tables IDs
    source          VARCHAR(50),             -- 'user_submitted', 'pg_stat_statements', 'ai_generated'
    avg_duration_ms DOUBLE PRECISION,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Key Indexes

```sql
CREATE INDEX idx_db_tables_snapshot ON db_tables(snapshot_id);
CREATE INDEX idx_db_columns_table ON db_columns(table_id);
CREATE INDEX idx_db_relationships_snapshot ON db_relationships(snapshot_id);
CREATE INDEX idx_db_relationships_source ON db_relationships(source_table_id);
CREATE INDEX idx_db_relationships_target ON db_relationships(target_table_id);
CREATE INDEX idx_layout_positions_layout ON layout_positions(layout_id);
CREATE INDEX idx_annotations_project ON annotations(project_id);
CREATE INDEX idx_schema_changes_project ON schema_changes(project_id);
CREATE INDEX idx_comments_project ON comments(project_id);
```

---

## Pros

- **Referential integrity**: Foreign keys enforce consistency between tables, columns, relationships, and layouts. Orphan records are impossible.
- **Well-understood tooling**: Every ORM (Prisma, Drizzle, SQLAlchemy, TypeORM) works naturally with this model. Migration tooling is mature.
- **Efficient queries for common operations**: Fetching "all columns for a table" or "all relationships in a snapshot" maps directly to indexed JOINs.
- **Strong schema versioning**: The snapshot model with integer versions makes "time-travel" between schema states simple (`WHERE snapshot_id = ?`).
- **Audit and compliance**: Every row has `created_at` and foreign keys to `users`, supporting full traceability.
- **Mature ecosystem**: PostgreSQL is the standard backend for SaaS tools with excellent hosting (Supabase, Neon, RDS, CloudSQL).

## Cons

- **Schema rigidity**: Adding a new database-engine-specific feature (e.g., PostgreSQL partitions, MySQL engines) requires ALTER TABLE or new junction tables.
- **Complex queries for graph traversal**: Finding multi-hop relationship chains (e.g., "all tables reachable from `orders`") requires recursive CTEs, which are less natural than graph queries.
- **Large JOIN chains**: Rendering a full ERD requires joining `schema_snapshots -> db_tables -> db_columns + db_relationships + layout_positions`, which can become 5-6 table joins.
- **Duplication across snapshots**: Each snapshot stores a full copy of all tables and columns, even if most did not change between versions. Storage grows linearly with snapshot count.
- **Layout data is weakly structured**: Canvas settings and visual properties are inherently freeform, making strict relational columns feel forced (hence the `JSONB` escape hatch on `canvas_settings`).

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Database | PostgreSQL 16+ (for JSON improvements and performance) |
| ORM | Prisma or Drizzle ORM (TypeScript), or SQLAlchemy (Python) |
| Migrations | Prisma Migrate, or standalone tools like Flyway / golang-migrate |
| Connection pooling | PgBouncer or Supabase connection pooler |
| Hosting | Supabase, Neon, or AWS RDS |
| Full-text search | PostgreSQL `tsvector` for table/column name search |
| Encryption at rest | `pgcrypto` for credential fields; TDE via hosting provider |

---

## Migration and Scaling Considerations

- **Horizontal scaling**: PostgreSQL handles millions of rows across these tables comfortably. If multi-tenant load grows, use Citus for distributed PostgreSQL or partition `schema_snapshots` and child tables by `project_id`.
- **Snapshot deduplication**: To reduce storage, introduce a content-addressable layer -- hash each table/column definition and only store unique definitions, with snapshots referencing them. This is an optimization for v2.
- **Read replicas**: Diagram rendering is read-heavy. Use read replicas for the visualization frontend and direct writes only to the primary.
- **Backup and PITR**: Standard PostgreSQL WAL-based point-in-time recovery protects against data loss. Snapshot versioning provides application-level recovery for schema data.
- **Migration path**: This model migrates cleanly to the hybrid JSONB model (Suggestion 3) by adding JSONB columns alongside relational ones, or to an event-sourced model (Suggestion 2) by replaying current state into an event log.

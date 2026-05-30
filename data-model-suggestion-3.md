# Data Model Suggestion 3: Hybrid Relational + JSONB/Document Approach

> Project: Database Schema Visualization (Candidate #289)
> Approach: PostgreSQL with strategic JSONB columns for flexible schema metadata

## Summary

This approach combines the referential integrity of a normalized relational model with the flexibility of PostgreSQL JSONB columns for data that is inherently semi-structured or varies by database engine. Core entities (projects, users, snapshots, layouts) use strict relational columns, while schema metadata (columns, indexes, constraints, engine-specific features) is stored as JSONB documents within relational rows.

This is inspired by how tools like Azimutt and ChartDB actually store schemas internally -- as JSON documents that represent the introspected schema -- while wrapping them in relational tables for multi-tenancy, versioning, and collaboration. It avoids the rigidity of full normalization (Suggestion 1) without the complexity of event sourcing (Suggestion 2).

---

## Key Entities and Relationships

### Design Principles

1. **Relational for structure**: Projects, users, organizations, snapshots, layouts, and collaboration entities use normalized tables with foreign keys.
2. **JSONB for schema metadata**: Tables, columns, indexes, and relationships within a schema snapshot are stored as a JSONB document. This reflects the fact that different database engines expose different metadata (PostgreSQL partitions, MySQL engines, MongoDB validators, etc.).
3. **JSONB for visual state**: Layout positions, canvas settings, and diagram styling are stored as JSONB since they are consumed as a single document by the frontend renderer.
4. **Relational for change tracking**: Schema diffs and change history use relational tables to enable efficient querying and filtering.

### Core Schema (DDL)

```sql
-- ============================================================
-- RELATIONAL LAYER: Multi-tenancy, auth, and project management
-- ============================================================

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',  -- org-level preferences
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255),
    avatar_url      TEXT,
    auth_provider   VARCHAR(50) NOT NULL,
    preferences     JSONB NOT NULL DEFAULT '{}',  -- theme, default notation, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization_members (
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    PRIMARY KEY (organization_id, user_id)
);

CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    database_engine VARCHAR(50),
    settings        JSONB NOT NULL DEFAULT '{}',  -- notation style, default layout algo, etc.
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE db_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    label           VARCHAR(255),
    engine          VARCHAR(50) NOT NULL,
    connection_config JSONB NOT NULL,         -- {host, port, database, username, ssl_mode, ...}
                                              -- password stored separately in encrypted vault
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- HYBRID LAYER: Schema snapshots with JSONB schema documents
-- ============================================================

CREATE TABLE schema_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    version         INTEGER NOT NULL,
    label           VARCHAR(255),
    source          VARCHAR(50) NOT NULL,     -- 'introspection', 'ddl_import', 'dbml_import', 'manual'
    
    -- THE KEY HYBRID DESIGN: full schema stored as a JSONB document
    schema_doc      JSONB NOT NULL,
    /*
    schema_doc structure:
    {
      "tables": [
        {
          "schema": "public",
          "name": "users",
          "type": "table",
          "description": "User accounts",
          "ai_description": "Core user identity table...",
          "row_count_est": 150000,
          "columns": [
            {
              "name": "id",
              "type": "uuid",
              "nullable": false,
              "default": "gen_random_uuid()",
              "primary_key": true,
              "description": "Primary identifier",
              "ai_description": "Auto-generated UUID..."
            },
            ...
          ],
          "indexes": [
            {
              "name": "idx_users_email",
              "type": "btree",
              "unique": true,
              "columns": ["email"]
            }
          ],
          "constraints": [
            {
              "name": "users_email_check",
              "type": "check",
              "definition": "email ~* '^[^@]+@[^@]+$'"
            }
          ],
          "engine_specific": {
            "postgresql": { "tablespace": "pg_default", "has_rls": true },
            "mysql": { "engine": "InnoDB", "collation": "utf8mb4_unicode_ci" }
          }
        }
      ],
      "relationships": [
        {
          "source": { "schema": "public", "table": "orders", "column": "user_id" },
          "target": { "schema": "public", "table": "users", "column": "id" },
          "type": "many_to_one",
          "constraint_name": "fk_orders_user",
          "on_delete": "CASCADE",
          "is_inferred": false
        }
      ],
      "enums": [...],
      "extensions": [...],
      "metadata": {
        "database_version": "PostgreSQL 16.2",
        "introspected_at": "2026-05-25T10:30:00Z",
        "table_count": 42,
        "relationship_count": 67
      }
    }
    */
    
    -- Extracted fields for efficient relational queries
    table_count     INTEGER GENERATED ALWAYS AS (jsonb_array_length(schema_doc->'tables')) STORED,
    
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, version)
);

-- GIN index for fast JSONB queries within schema documents
CREATE INDEX idx_schema_doc_gin ON schema_snapshots USING GIN(schema_doc jsonb_path_ops);

-- Extracted table index for table-name search across snapshots
CREATE INDEX idx_schema_tables ON schema_snapshots 
    USING GIN ((schema_doc->'tables') jsonb_path_ops);

-- ============================================================
-- HYBRID LAYER: Layouts with JSONB visual state
-- ============================================================

CREATE TABLE layouts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    snapshot_id     UUID REFERENCES schema_snapshots(id),
    name            VARCHAR(255) NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    layout_algorithm VARCHAR(50),
    
    -- Full layout state as a single JSONB document
    visual_state    JSONB NOT NULL DEFAULT '{}',
    /*
    visual_state structure:
    {
      "canvas": {
        "zoom": 1.0,
        "pan": { "x": 0, "y": 0 },
        "grid": { "enabled": true, "size": 20 },
        "background": "#1a1a2e"
      },
      "positions": {
        "public.users": { "x": 100, "y": 200, "width": 280, "height": null, "collapsed": false, "color": "#4CAF50" },
        "public.orders": { "x": 450, "y": 200, "width": 280, "height": null, "collapsed": false, "color": "#2196F3" }
      },
      "groups": [
        {
          "name": "Auth Domain",
          "tables": ["public.users", "public.sessions", "public.roles"],
          "color": "#FF9800",
          "bounds": { "x": 50, "y": 150, "width": 600, "height": 400 }
        }
      ],
      "hidden_tables": ["public.schema_migrations", "public.ar_internal_metadata"],
      "relationship_styles": {
        "fk_orders_user": { "path_type": "curved", "color": "#666" }
      }
    }
    */
    
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- RELATIONAL LAYER: Annotations, tags, and AI results
-- ============================================================

CREATE TABLE annotations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    snapshot_id     UUID REFERENCES schema_snapshots(id),
    target_path     VARCHAR(500) NOT NULL,     -- JSONPath-style: 'public.users', 'public.users.email'
    target_type     VARCHAR(50) NOT NULL,      -- 'table', 'column', 'relationship'
    annotation_type VARCHAR(50) NOT NULL,      -- 'note', 'warning', 'anti_pattern', 'ai_suggestion'
    content         TEXT NOT NULL,
    severity        VARCHAR(20),
    ai_metadata     JSONB,                     -- model version, confidence, pattern details
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

CREATE TABLE object_tags (
    tag_id          UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    target_path     VARCHAR(500) NOT NULL,     -- 'public.users', 'public.orders.status'
    PRIMARY KEY (tag_id, target_path)
);

-- ============================================================
-- RELATIONAL LAYER: Change tracking (computed diffs)
-- ============================================================

CREATE TABLE schema_diffs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    from_version    INTEGER NOT NULL,
    to_version      INTEGER NOT NULL,
    
    -- Diff stored as a structured JSONB document
    diff_doc        JSONB NOT NULL,
    /*
    diff_doc structure:
    {
      "tables_added": ["public.notifications"],
      "tables_dropped": [],
      "tables_modified": [
        {
          "table": "public.users",
          "columns_added": [{"name": "phone", "type": "varchar(20)"}],
          "columns_dropped": [],
          "columns_modified": [{"name": "email", "changes": {"type": {"from": "varchar(100)", "to": "varchar(255)"}}}]
        }
      ],
      "relationships_added": [...],
      "relationships_dropped": [...],
      "summary_stats": { "total_changes": 5, "breaking_changes": 0 }
    }
    */
    
    ai_changelog    TEXT,                     -- AI-generated human-readable summary
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, from_version, to_version)
);

-- ============================================================
-- RELATIONAL LAYER: Collaboration
-- ============================================================

CREATE TABLE comments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    target_path     VARCHAR(500),              -- 'public.users.email' or NULL for project-level
    parent_id       UUID REFERENCES comments(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    body            TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE query_examples (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    sql_text        TEXT NOT NULL,
    description     TEXT,
    tables_involved TEXT[] NOT NULL,           -- ['public.users', 'public.orders']
    source          VARCHAR(50),
    performance     JSONB,                     -- {avg_ms, p99_ms, rows_returned, plan_summary}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- RELATIONAL LAYER: Activity log
-- ============================================================

CREATE TABLE activity_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id),
    action          VARCHAR(100) NOT NULL,     -- 'schema_imported', 'layout_saved', 'comment_added'
    target_path     VARCHAR(500),
    details         JSONB,                     -- action-specific context
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_activity_project ON activity_log(project_id, created_at DESC);
```

### Example: Querying the Hybrid Model

```sql
-- Find all tables with a column named 'email' across all snapshots of a project
SELECT s.version, t->>'name' AS table_name, c->>'type' AS email_type
FROM schema_snapshots s,
     jsonb_array_elements(s.schema_doc->'tables') AS t,
     jsonb_array_elements(t->'columns') AS c
WHERE s.project_id = '...'
  AND c->>'name' = 'email';

-- Get the latest schema snapshot as a single document (one query, one round trip)
SELECT schema_doc
FROM schema_snapshots
WHERE project_id = '...'
ORDER BY version DESC
LIMIT 1;

-- Count tables per schema namespace in the latest snapshot
SELECT t->>'schema' AS ns, count(*)
FROM schema_snapshots s,
     jsonb_array_elements(s.schema_doc->'tables') AS t
WHERE s.id = '...'
GROUP BY t->>'schema';

-- Find tables with more than 50 columns (potential anti-pattern)
SELECT t->>'schema' AS schema_name, t->>'name' AS table_name,
       jsonb_array_length(t->'columns') AS col_count
FROM schema_snapshots s,
     jsonb_array_elements(s.schema_doc->'tables') AS t
WHERE s.project_id = '...'
  AND s.version = (SELECT MAX(version) FROM schema_snapshots WHERE project_id = s.project_id)
  AND jsonb_array_length(t->'columns') > 50;
```

---

## Pros

- **Single-document schema loading**: The entire schema for a snapshot is loaded in one query (`SELECT schema_doc`), which is exactly what the frontend ERD renderer needs. No multi-table JOINs.
- **Database-engine flexibility**: Engine-specific metadata (PostgreSQL RLS, MySQL engines, MongoDB validators) is stored in `engine_specific` JSONB fields without schema changes. Adding support for a new database engine requires zero ALTER TABLE statements.
- **Natural import/export**: Schema import from SQL DDL, DBML, or INFORMATION_SCHEMA maps directly to the JSONB document structure. Export reverses the same transformation.
- **Efficient versioning**: Each snapshot stores a self-contained schema document. Comparing versions is a JSONB diff operation.
- **Familiar relational foundation**: Projects, users, comments, and collaboration use standard relational patterns with foreign keys and joins. ORMs work normally for these entities.
- **Good query performance**: GIN indexes on JSONB enable fast searches within schema documents. Generated columns extract commonly queried values (like `table_count`).
- **Balanced complexity**: Simpler than event sourcing, more flexible than full normalization. A pragmatic middle ground that matches how schema data naturally flows through the system.

## Cons

- **No referential integrity within JSONB**: A typo in a relationship's `target.table` reference inside the JSON document is not caught by the database. Validation must be done in application code.
- **Partial update inefficiency**: Changing a single column description requires reading the entire `schema_doc`, modifying the JSON in application code, and writing it back. PostgreSQL's `jsonb_set()` helps but is limited for deeply nested changes.
- **Duplication across snapshots**: Like the normalized model, each snapshot stores a full copy. The JSONB document may actually be larger than equivalent normalized rows due to repeated key names.
- **Complex JSONB queries**: While powerful, JSONB path queries are harder to write and debug than standard SQL JOINs. Team members need familiarity with `jsonb_array_elements`, `jsonb_path_query`, and GIN indexing.
- **Mixed paradigm**: Developers must understand which data lives in relational tables vs. JSONB documents. The boundary can feel arbitrary and requires clear documentation.
- **Limited JSONB schema validation**: PostgreSQL CHECK constraints on JSONB are basic. Full document validation requires application-level JSON Schema validation or triggers.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Database | PostgreSQL 16+ (JSONB improvements, IS JSON predicates, generated columns) |
| ORM | Prisma (with `Json` type support) or Drizzle ORM; SQLAlchemy with `JSONB` column type |
| Validation | Zod or AJV (TypeScript) for JSONB document schema validation at the application layer |
| Migrations | Prisma Migrate for relational tables; custom scripts for JSONB document schema evolution |
| Search | PostgreSQL full-text search with `tsvector` generated from JSONB, or Meilisearch for richer search |
| Hosting | Supabase (native JSONB support, row-level security) or Neon |

---

## Migration and Scaling Considerations

- **From normalized (Suggestion 1)**: Migrate by SELECT-ing tables + columns + relationships into a JSONB document per snapshot. The relational tables for projects, users, and collaboration remain unchanged.
- **To event-sourced (Suggestion 2)**: The activity_log table already captures a partial event stream. Promoting it to a full event store is an incremental step.
- **JSONB document size**: A schema with 500 tables, 5,000 columns, and 1,000 relationships produces a JSONB document of approximately 2-5 MB. PostgreSQL handles this efficiently, but TOAST compression should be verified. For schemas with 2,000+ tables, consider splitting into sub-documents per schema namespace.
- **Read scaling**: The single-document read pattern is cache-friendly. CDN or Redis caching of the rendered schema document is straightforward.
- **Write scaling**: Schema imports (the main write path) are infrequent compared to reads. A single PostgreSQL primary handles the write load for most installations.
- **Multi-tenant partitioning**: Partition `schema_snapshots` by `project_id` using PostgreSQL declarative partitioning for large SaaS deployments.

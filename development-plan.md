# Database Schema Visualization — Phased Development Plan

> Project: 289-database-schema-visualization · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesizes `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files. The product is an AI-native, open-source, self-hostable ERD platform that reverse-engineers live databases into interactive, annotated, query-aware schema diagrams with full change history.

The data model adopts **Suggestion 3 (Hybrid Relational + JSONB)** as the canonical store: relational tables for multi-tenancy, auth, versioning, and collaboration; a single JSONB `schema_doc` per snapshot for the introspected schema. Graph-style traversal queries (impact analysis, reachability) — the strength of Suggestion 4 — are served by computing a normalized in-memory graph from `schema_doc` and, where DB-side traversal is needed, recursive CTEs over an extracted edge table. This avoids running a second database while preserving graph features. Suggestion 2's event log is incorporated as a lightweight `activity_log` rather than full event sourcing.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | TypeScript (Node 22 LTS) end-to-end | The product is API + heavy interactive frontend; a single language across the introspection engine, REST API, and React canvas removes a type boundary. The richest schema-as-code ecosystem (`@holistics/dbml-core`, `@azimutt/aml`, Mermaid, JSON Schema, GraphQL) is npm-native. |
| Monorepo tooling | pnpm workspaces + Turborepo | Shared `@dbviz/core` types between API and web; incremental builds. |
| API framework | Fastify 5 + `@fastify/swagger` | High throughput, first-class JSON Schema validation that doubles as the OpenAPI 3.1 source of truth (standards.md OpenAPI requirement). |
| API schema/validation | Zod + `zod-to-openapi` | One Zod schema validates requests and generates the OpenAPI document; matches Suggestion 3's recommendation to validate JSONB documents in app code. |
| Application database | PostgreSQL 16 | Suggestion 3 requires JSONB `jsonb_path_ops` GIN indexes, generated columns, and `IS JSON` predicates. Also the default self-host target (Supabase/Neon). |
| ORM / query builder | Drizzle ORM | First-class `jsonb` column typing, generated columns, and raw SQL escape hatch for recursive CTEs and `jsonb_array_elements` queries that an ORM cannot express. |
| Migrations | drizzle-kit | Generates SQL migrations from the Drizzle schema; checked into `migrations/`. |
| Introspection drivers | `pg`, `mysql2`, `mssql`, `better-sqlite3` | Cover the four engines named in standards.md (ISO/IEC 9075 Information Schema introspection); each implements a common `Introspector` interface. |
| Task queue | BullMQ on Redis | Introspection of large/live DBs, AI annotation, and diff computation are long-running and must not block API requests. |
| Cache / pub-sub | Redis 7 | BullMQ backend, hot `schema_doc` cache, and WebSocket fan-out for collaboration. |
| LLM provider | Anthropic SDK (`@anthropic-ai/sdk`), Claude | All AI features (annotation, anti-pattern detection, NL query, migration plans, changelogs). Provider abstracted behind an `LLMClient` interface so an OpenAI adapter can be added. |
| AI tool surface | MCP server (`@modelcontextprotocol/sdk`) | standards.md flags MCP as the key unexploited differentiator — expose the schema graph as MCP resources/tools so coding assistants query data models directly. |
| Frontend framework | React 19 + Vite | SPA dashboard + canvas; Vite for fast HMR on the diagram code. |
| Diagram canvas | React Flow (`@xyflow/react`) + `elkjs` (ELK layout) | React Flow gives pan/zoom/nodes/edges out of the box; ELK provides the layered/force layout algorithms (features.md "automatic layout"). |
| Schema-as-code libs | `@holistics/dbml-core`, `@azimutt/aml`, `mermaid` | Import/export DBML, AML, Mermaid ERD per standards.md. |
| Styling | Tailwind CSS + Radix UI primitives | Dark mode (MVP requirement) via Tailwind `dark:`; accessible primitives. |
| Auth | Lucia-style sessions + Arctic OAuth (GitHub, Google) | OAuth 2.0 + OIDC per standards.md; PKCE Authorization Code flow. |
| Credential encryption | AES-256-GCM via Node `crypto`, key from `DBVIZ_MASTER_KEY` | Live-DB credentials encrypted at rest (Suggestion 1/3 requirement); never returned to the client. |
| Testing | Vitest (unit/integration) + Playwright (E2E) + Testcontainers | Vitest across packages; Testcontainers spins real Postgres/MySQL for introspection integration tests; Playwright drives the canvas. |
| Code quality | Biome (lint + format) + `tsc --noEmit` | Single fast tool for lint/format; strict TypeScript. |
| Containerisation | Docker + docker-compose | Self-host is the primary deployment mode (README). Compose bundles api, web, worker, postgres, redis. |
| CI | GitHub Actions | Lint, typecheck, test, build, docker build per phase Definition of Done. |

### Project Structure

```
dbviz/
├── package.json                    # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json
├── biome.json
├── tsconfig.base.json
├── docker-compose.yml              # postgres, redis, api, worker, web
├── Dockerfile.api
├── Dockerfile.worker
├── Dockerfile.web
├── .env.example
├── migrations/                     # drizzle-kit generated SQL
│   └── 0000_init.sql
├── packages/
│   ├── core/                       # @dbviz/core — shared, framework-free
│   │   └── src/
│   │       ├── schema-doc/         # SchemaDoc type, Zod validator, builders
│   │       │   ├── types.ts
│   │       │   ├── schema.ts        # Zod schema for schema_doc
│   │       │   └── graph.ts         # build adjacency graph + traversal helpers
│   │       ├── diff/                # snapshot diff engine
│   │       │   └── diff.ts
│   │       ├── anti-patterns/       # rule-based detectors
│   │       │   └── rules.ts
│   │       ├── formats/             # import/export adapters
│   │       │   ├── ddl.ts
│   │       │   ├── dbml.ts
│   │       │   ├── aml.ts
│   │       │   ├── mermaid.ts
│   │       │   └── json-schema.ts
│   │       └── index.ts
│   ├── introspect/                 # @dbviz/introspect — live-DB readers
│   │   └── src/
│   │       ├── introspector.ts      # Introspector interface + factory
│   │       ├── postgres.ts
│   │       ├── mysql.ts
│   │       ├── mssql.ts
│   │       └── sqlite.ts
│   ├── llm/                        # @dbviz/llm — AI client + prompts
│   │   └── src/
│   │       ├── client.ts            # LLMClient interface + Anthropic impl
│   │       └── prompts/
│   │           ├── annotate.ts
│   │           ├── anti-pattern.ts
│   │           ├── nl-query.ts
│   │           ├── migration-plan.ts
│   │           └── changelog.ts
│   ├── db/                          # @dbviz/db — Drizzle schema + repositories
│   │   └── src/
│   │       ├── schema.ts            # Drizzle table definitions
│   │       ├── client.ts
│   │       └── repositories/
│   ├── api/                         # @dbviz/api — Fastify app
│   │   └── src/
│   │       ├── server.ts
│   │       ├── plugins/             # auth, db, redis, swagger, errors
│   │       ├── routes/
│   │       │   ├── auth.ts
│   │       │   ├── projects.ts
│   │       │   ├── connections.ts
│   │       │   ├── snapshots.ts
│   │       │   ├── diagram.ts
│   │       │   ├── diffs.ts
│   │       │   ├── annotations.ts
│   │       │   ├── comments.ts
│   │       │   ├── ai.ts
│   │       │   └── export.ts
│   │       ├── queue/               # BullMQ producers
│   │       └── ws/                  # collaboration websocket
│   ├── worker/                      # @dbviz/worker — BullMQ consumers
│   │   └── src/
│   │       ├── index.ts
│   │       └── jobs/
│   │           ├── introspect.job.ts
│   │           ├── diff.job.ts
│   │           └── ai-annotate.job.ts
│   ├── mcp/                         # @dbviz/mcp — MCP server
│   │   └── src/server.ts
│   └── web/                         # @dbviz/web — React SPA
│       └── src/
│           ├── main.tsx
│           ├── api/                 # generated client from OpenAPI
│           ├── routes/
│           ├── canvas/              # React Flow ERD canvas
│           │   ├── ErdCanvas.tsx
│           │   ├── TableNode.tsx
│           │   ├── RelationshipEdge.tsx
│           │   └── layout.ts        # elkjs integration
│           ├── components/
│           └── store/               # zustand state
└── tests/
    └── e2e/                         # Playwright specs + fixtures
```

---

## Phase 1: Foundation & Project Scaffolding

### Purpose
Establish the monorepo, shared types, database connection, migration tooling, and a running (empty) API server with health checks and OpenAPI generation. After this phase the team has a deployable skeleton with CI green, onto which every later phase adds routes and packages without restructuring.

### Tasks

#### 1.1 — Monorepo & tooling setup

**What**: Initialize the pnpm/Turborepo workspace with Biome, shared tsconfig, and the empty package directories.

**Design**:
- Root `package.json` with workspaces `packages/*`; scripts `dev`, `build`, `test`, `lint`, `typecheck` delegating to `turbo`.
- `tsconfig.base.json`: `"strict": true`, `"moduleResolution": "bundler"`, `"target": "ES2023"`, project references per package.
- `biome.json`: tab indentation off, double quotes, recommended ruleset, organize imports on.
- `.env.example` enumerating: `DATABASE_URL`, `REDIS_URL`, `DBVIZ_MASTER_KEY` (32-byte base64), `ANTHROPIC_API_KEY`, `GITHUB_CLIENT_ID/SECRET`, `GOOGLE_CLIENT_ID/SECRET`, `SESSION_SECRET`, `PUBLIC_API_URL`.

**Testing**:
- `Unit: pnpm install resolves all workspace links` (CI step asserts exit 0).
- `Unit: pnpm -r typecheck passes on empty packages` (each package exports a stub `index.ts`).
- `Unit: biome check passes` on committed files.

#### 1.2 — Database schema & migrations (Drizzle)

**What**: Implement the full Suggestion-3 hybrid schema as Drizzle table definitions and generate the initial migration.

**Design**:
Define these tables in `packages/db/src/schema.ts` (DDL summarized; full columns per `data-model-suggestion-3.md`):
- `organizations`, `users`, `organization_members`, `projects` (with `settings jsonb`).
- `db_connections` — credentials split: non-secret `connection_config jsonb`; secret bytes in `encrypted_credentials bytea` (host/user/password ciphertext + GCM nonce/tag).
- `schema_snapshots` — `schema_doc jsonb NOT NULL`, generated column `table_count`, GIN index `idx_schema_doc_gin USING gin(schema_doc jsonb_path_ops)`, `UNIQUE(project_id, version)`.
- `layouts` (`visual_state jsonb`), `annotations` (`target_path`, `target_type`, `is_ai_generated`), `tags`, `object_tags`, `schema_diffs` (`diff_doc jsonb`, `ai_changelog`), `comments`, `query_examples`, `activity_log`.

Drizzle column example:
```ts
export const schemaSnapshots = pgTable("schema_snapshots", {
  id: uuid("id").primaryKey().defaultRandom(),
  projectId: uuid("project_id").notNull().references(() => projects.id, { onDelete: "cascade" }),
  version: integer("version").notNull(),
  label: varchar("label", { length: 255 }),
  source: varchar("source", { length: 50 }).notNull(), // 'introspection'|'ddl_import'|'dbml_import'|'aml_import'|'manual'
  schemaDoc: jsonb("schema_doc").$type<SchemaDoc>().notNull(),
  createdBy: uuid("created_by").references(() => users.id),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
}, (t) => ({ uq: unique().on(t.projectId, t.version) }));
```

**Testing**:
- `Integration (Testcontainers Postgres): run migration → all tables exist` (query `information_schema.tables`).
- `Integration: insert snapshot with 3-table schema_doc → table_count generated column == 3`.
- `Integration: insert two snapshots same (project_id, version) → unique violation`.
- `Integration: GIN index present` (query `pg_indexes` for `idx_schema_doc_gin`).

#### 1.3 — Fastify server, plugins, health & OpenAPI

**What**: Boot a Fastify server with db/redis/swagger/error plugins and a `/healthz` endpoint; serve `/openapi.json` and `/docs`.

**Design**:
- `plugins/db.ts` decorates `app.db` (Drizzle client over a `pg` pool).
- `plugins/redis.ts` decorates `app.redis` and `app.queues`.
- `plugins/errors.ts`: maps thrown `AppError{ code, statusCode, message }` to RFC 9457 problem+json; Zod errors → 422 with field paths.
- `plugins/swagger.ts`: register `@fastify/swagger` with OpenAPI 3.1; route schemas authored as Zod via `zod-to-openapi`.
- `GET /healthz` → `{ status: "ok", db: boolean, redis: boolean }`, 200 if both reachable else 503.

**Testing**:
- `Integration: GET /healthz with db+redis up → 200, {status:"ok"}`.
- `Integration (mocked db down): GET /healthz → 503, db:false`.
- `Unit: GET /openapi.json → valid OpenAPI 3.1 doc (parses with @readme/openapi-parser)`.
- `Unit: unknown route → 404 problem+json with type/title/status`.

---

## Phase 2: SchemaDoc Core Model & Format Conversion

### Purpose
Define the canonical in-memory `SchemaDoc` structure (the heart of the system) and the bidirectional converters between SQL DDL, DBML, AML, Mermaid ERD, and JSON Schema. This is framework-free in `@dbviz/core` so it is reused by the introspection engine, importers, exporters, and the MCP server. After this phase a schema can be parsed from text formats and re-emitted, independent of any live database.

### Tasks

#### 2.1 — Canonical SchemaDoc type & Zod validator

**What**: Define the `SchemaDoc` TypeScript type plus a Zod schema that validates any document before it is stored in `schema_doc`.

**Design**:
```ts
export interface ColumnDoc {
  name: string; type: string; nullable: boolean;
  default?: string | null; primaryKey: boolean; unique: boolean;
  description?: string; aiDescription?: string;
}
export interface IndexDoc { name: string; type: string; unique: boolean; columns: string[]; }
export interface ConstraintDoc { name: string; type: "check"|"unique"|"primary_key"|"exclusion"; definition: string; }
export interface TableDoc {
  schema: string; name: string; type: "table"|"view"|"materialized_view";
  description?: string; aiDescription?: string; rowCountEst?: number;
  columns: ColumnDoc[]; indexes: IndexDoc[]; constraints: ConstraintDoc[];
  engineSpecific?: Record<string, unknown>;
}
export interface RelationshipDoc {
  source: { schema: string; table: string; column: string };
  target: { schema: string; table: string; column: string };
  type: "one_to_one"|"one_to_many"|"many_to_one"|"many_to_many";
  constraintName?: string; onDelete?: string; onUpdate?: string; isInferred: boolean;
}
export interface SchemaDoc {
  tables: TableDoc[]; relationships: RelationshipDoc[];
  enums?: { name: string; values: string[] }[];
  extensions?: string[];
  metadata: { databaseVersion?: string; introspectedAt?: string; tableCount: number; relationshipCount: number };
}
export const schemaDocSchema: z.ZodType<SchemaDoc>; // mirrors the above
```
Provide `validateSchemaDoc(doc): { ok: true } | { ok: false; errors: string[] }` and `normalizeSchemaDoc(doc)` (sort tables/columns deterministically, fully qualify relationship endpoints, recompute `metadata` counts). Referential checks (Suggestion 3 con): every relationship endpoint must resolve to an existing table+column, else error.

**Testing**:
- `Unit: valid doc → validate ok:true`.
- `Unit: relationship target.table not in tables[] → ok:false, error names the dangling ref`.
- `Unit: duplicate column name in a table → ok:false`.
- `Unit: normalizeSchemaDoc → tables sorted by schema.name, metadata counts recomputed`.

#### 2.2 — Schema graph & traversal helpers

**What**: Build an adjacency-list graph from a `SchemaDoc` and provide reachability/impact queries (the graph features from Suggestion 4 without a graph DB).

**Design**:
```ts
export interface SchemaGraph {
  nodes: Map<string, TableDoc>;            // key: "schema.table"
  outEdges: Map<string, RelationshipDoc[]>;
  inEdges: Map<string, RelationshipDoc[]>;
}
export function buildGraph(doc: SchemaDoc): SchemaGraph;
export function reachableFrom(g: SchemaGraph, table: string, maxHops: number): Array<{ table: string; distance: number }>;
export function impactedBy(g: SchemaGraph, table: string, maxHops: number): Array<{ table: string; distance: number }>; // inbound
export function orphanTables(g: SchemaGraph): string[];      // no in/out edges
export function junctionTables(g: SchemaGraph): string[];     // ≥2 FK, only PK/FK columns
export function findCycles(g: SchemaGraph): string[][];
```
BFS for reachability/impact; Tarjan SCC for cycles.

**Testing**:
- `Unit: orders→users, order_items→orders → reachableFrom(order_items,2) = [orders@1, users@2]`.
- `Unit: standalone audit_log table → orphanTables includes audit_log`.
- `Unit: user_roles(user_id FK, role_id FK, PK both) → junctionTables includes user_roles`.
- `Unit: a→b→a → findCycles returns [[a,b]]`.

#### 2.3 — DDL parser/generator

**What**: Parse `CREATE TABLE`/`ALTER TABLE` DDL into a `SchemaDoc` and emit dialect-specific DDL from a `SchemaDoc`.

**Design**:
- Parse using `node-sql-parser` (multi-dialect). Map AST → `TableDoc`/`ColumnDoc`/`ConstraintDoc`; foreign keys → `RelationshipDoc`.
- `generateDdl(doc, dialect: "postgres"|"mysql"|"mssql"|"sqlite"): string` — deterministic, FK constraints emitted after all tables; types mapped per dialect table.
- Unsupported constructs collected into `warnings: string[]` rather than throwing.

**Testing**:
- `Unit: CREATE TABLE users(id uuid pk, email varchar(255) unique not null) → 1 table, email.unique=true`.
- `Unit: FK with ON DELETE CASCADE → RelationshipDoc.onDelete="CASCADE"`.
- `Fixture: round-trip — parse(generateDdl(doc,"postgres")) ≈ doc (normalized equal)`.
- `Unit: unknown dialect feature → captured in warnings, no throw`.

#### 2.4 — DBML, AML, Mermaid, JSON Schema adapters

**What**: Import/export adapters for the schema-as-code formats named in standards.md.

**Design**:
- DBML: `@holistics/dbml-core` for `parseDbml(text)→SchemaDoc` and `toDbml(doc)`.
- AML: `@azimutt/aml` parse/generate; supports nested columns (mapped to dotted column names + `engineSpecific.nested`).
- Mermaid ERD: emit-only `toMermaid(doc)` producing `erDiagram` with Crow's Foot cardinality; import via a small line parser (best-effort).
- JSON Schema (Draft 2020-12): `fromJsonSchema(doc)` maps `properties`→columns, `$ref`→relationships; `toJsonSchema(doc)`.
- Common interface: `interface FormatAdapter { parse?(text): ImportResult; generate?(doc): string }` where `ImportResult = { doc: SchemaDoc; warnings: string[] }`.

**Testing**:
- `Fixture: sample.dbml → SchemaDoc with N tables → toDbml → re-parse equal`.
- `Unit: toMermaid emits "USERS ||--o{ ORDERS : has" for one_to_many`.
- `Unit: JSON Schema with $ref array item → many_to_one RelationshipDoc`.
- `Fixture: AML with nested document column → dotted column names preserved`.

---

## Phase 3: Live Database Introspection

### Purpose
Connect to a user's live database and produce a `SchemaDoc` via the ISO/IEC 9075 Information Schema (and engine-specific catalogs where richer). This is the core "instant import" value proposition. After this phase the system can turn any reachable Postgres/MySQL/SQL Server/SQLite database into a stored, validated snapshot.

### Tasks

#### 3.1 — Introspector interface & engine factory

**What**: Define a common introspection contract and a factory that selects the driver by engine.

**Design**:
```ts
export interface ConnectionParams {
  engine: "postgres"|"mysql"|"mssql"|"sqlite";
  host?: string; port?: number; database: string;
  username?: string; password?: string; sslMode?: "disable"|"require"|"verify-full"; filePath?: string;
}
export interface Introspector {
  testConnection(p: ConnectionParams): Promise<{ ok: boolean; serverVersion?: string; error?: string }>;
  introspect(p: ConnectionParams, opts?: { schemas?: string[]; includeRowCounts?: boolean }): Promise<SchemaDoc>;
}
export function getIntrospector(engine: ConnectionParams["engine"]): Introspector;
```
All connections are **read-only** (statement timeout 30s, `default_transaction_read_only=on` for Postgres). Credentials are never logged.

**Testing**:
- `Unit: getIntrospector("postgres") → PostgresIntrospector instance`.
- `Unit: getIntrospector("oracle") → throws UnsupportedEngineError`.

#### 3.2 — Postgres introspector (reference implementation)

**What**: Query Information Schema + `pg_catalog` to build a complete `SchemaDoc`.

**Design**:
- Tables/views: `information_schema.tables` filtered to requested schemas (default exclude `pg_catalog`, `information_schema`).
- Columns: `information_schema.columns` (name, ordinal, type, nullable, default).
- Primary/unique/check: `information_schema.table_constraints` + `key_column_usage`.
- Foreign keys: `information_schema.referential_constraints` joined to key usage → `RelationshipDoc` with `onDelete/onUpdate`.
- Indexes: `pg_indexes` / `pg_index` for index type + uniqueness.
- Row count estimate: `pg_class.reltuples` (cheap, when `includeRowCounts`).
- Enums: `pg_type`/`pg_enum`; extensions: `pg_extension`.
- `engineSpecific.postgresql`: `{ hasRls, tablespace }`.
- Relationship `type` inferred: unique/PK on FK column → `one_to_one`, else `many_to_one`.

**Testing**:
- `Integration (Testcontainers postgres:16, seeded with sample schema): introspect → table/column/FK counts match seed`.
- `Integration: FK orders.user_id→users.id → RelationshipDoc present, isInferred=false`.
- `Integration: enum type seeded → appears in doc.enums`.
- `Integration: testConnection bad password → { ok:false, error }` (no throw, no credential in message).

#### 3.3 — MySQL, SQL Server, SQLite introspectors

**What**: Implement the remaining three engines against the shared interface.

**Design**:
- MySQL: `information_schema.TABLES/COLUMNS/KEY_COLUMN_USAGE/STATISTICS`; engine + collation → `engineSpecific.mysql`.
- SQL Server: `INFORMATION_SCHEMA` + `sys.foreign_keys`/`sys.indexes`; schema defaults to `dbo`.
- SQLite: `PRAGMA table_info`, `PRAGMA foreign_key_list`, `PRAGMA index_list`; no real schemas → single `main` schema.
- Each maps native types to a normalized type string while preserving the raw type in `ColumnDoc.type`.

**Testing**:
- `Integration (Testcontainers mysql:8): introspect seeded db → matches DDL adapter parse of the seed`.
- `Integration (sqlite file fixture): PRAGMA-based FK detection → relationships present`.
- `Integration (mssql via Testcontainers): dbo schema tables introspected`.
- `Unit: each introspector returns a doc that passes validateSchemaDoc`.

#### 3.4 — Connection management API + encryption

**What**: REST endpoints to create/test/list connections with encrypted credential storage.

**Design**:
- `crypto.ts`: `encrypt(plain): Buffer` (AES-256-GCM, random 12-byte IV, key from `DBVIZ_MASTER_KEY`), `decrypt(buf): string`. Stored layout: `IV(12) || authTag(16) || ciphertext`.
- `POST /projects/:id/connections` body `{ engine, label, host, port, database, username, password, sslMode }` → encrypts secret fields, stores non-secret in `connection_config`, returns connection **without** secrets.
- `POST /projects/:id/connections/:cid/test` → decrypts, runs `testConnection`, returns `{ ok, serverVersion }`.
- `GET /projects/:id/connections` → list, secrets redacted.

**Testing**:
- `Unit: encrypt then decrypt round-trips; ciphertext differs each call (random IV)`.
- `Unit: tampered authTag → decrypt throws`.
- `Integration: POST connection → response contains no password/host secret fields`.
- `Integration (mocked introspector): POST .../test ok → 200 {ok:true}`.

#### 3.5 — Introspection job (BullMQ) → snapshot

**What**: Asynchronous introspection that writes a new versioned snapshot.

**Design**:
- `POST /projects/:id/snapshots:introspect { connectionId, schemas? }` → enqueues `introspect` job, returns `202 { jobId }`.
- `introspect.job.ts`: decrypt connection → `introspect()` → `validateSchemaDoc` → `normalizeSchemaDoc` → insert `schema_snapshots` with `version = max(version)+1`, `source="introspection"` → write `activity_log`.
- `GET /jobs/:jobId` → `{ state: "waiting"|"active"|"completed"|"failed", result?: { snapshotId }, error? }`.
- Failure → job marked failed with sanitized error; no partial snapshot written.

**Testing**:
- `Integration: enqueue introspect against Testcontainers db → snapshot row created with correct version`.
- `Integration: second introspect → version increments to 2`.
- `Integration (introspector throws): job state=failed, no snapshot row`.
- `Unit: GET /jobs/:id maps BullMQ states correctly`.

---

## Phase 4: Auth, Projects & Multi-Tenancy

### Purpose
Add user identity (OAuth/OIDC), organizations, projects, membership, and authorization so the tool is usable by teams and connections/snapshots are access-controlled. After this phase every prior endpoint is protected and scoped to an organization. (Depends only on Phase 1; can be built in parallel with Phases 2–3.)

### Tasks

#### 4.1 — OAuth/OIDC login & sessions

**What**: GitHub and Google sign-in via Authorization Code + PKCE; server-side sessions.

**Design**:
- `GET /auth/:provider/login` → redirect with PKCE challenge + state (stored in Redis, 10-min TTL).
- `GET /auth/:provider/callback` → exchange code, fetch profile, upsert `users` (`auth_provider`, `auth_provider_id`), create session (opaque token in Redis, `Set-Cookie HttpOnly Secure SameSite=Lax`, 30-day sliding).
- `POST /auth/logout` → delete session.
- `GET /auth/me` → current user or 401.

**Testing**:
- `Integration (mocked provider): callback with valid code+state → session cookie set, user upserted`.
- `Integration: callback with mismatched state → 400, no session`.
- `Integration: GET /auth/me without cookie → 401`.
- `Unit: PKCE verifier/challenge pair validates`.

#### 4.2 — Organizations, projects & membership

**What**: CRUD for organizations and projects, member invitation with roles.

**Design**:
- `roles`: `owner` > `admin` > `editor` > `viewer`.
- `POST /orgs`, `GET /orgs`, `POST /orgs/:id/members { userId|email, role }`.
- `POST /orgs/:id/projects { name, description, databaseEngine }`, `GET /orgs/:id/projects`, `GET /projects/:id`, `PATCH`, `DELETE`.
- Creator becomes `owner`/`editor` respectively.

**Testing**:
- `Integration: create org → creator is owner member`.
- `Integration: viewer tries POST project → 403`.
- `Integration: list projects only returns projects in caller's orgs`.

#### 4.3 — Authorization middleware

**What**: A reusable guard that resolves the caller's role for the target project/org and enforces a minimum.

**Design**:
```ts
function requireRole(min: Role): preHandler; // reads :id param, joins membership, sets req.role
```
- Applied to all project-scoped routes from Phases 3, 5–9. Write routes require `editor`; destructive require `admin`; member management requires `owner`.
- Returns 403 problem+json on failure, 404 if project not visible to caller (avoid leaking existence).

**Testing**:
- `Integration: editor POST snapshot → allowed; viewer → 403`.
- `Integration: non-member GET project → 404 (not 403)`.
- `Unit: role ordering comparator (viewer<editor<admin<owner)`.

---

## Phase 5: Diagram API & Interactive Canvas

### Purpose
Deliver the headline feature: render a stored snapshot as an interactive ERD with auto-layout, browsing, search, dark mode, and persisted layouts. This is the primary user-facing surface and the MVP's center of gravity. (Requires Phases 2, 3, 4.)

### Tasks

#### 5.1 — Diagram read API

**What**: Endpoints that return a snapshot's `schema_doc` plus a saved layout in one round trip (Suggestion 3 single-document advantage).

**Design**:
- `GET /projects/:id/snapshots` → version list with metadata + counts.
- `GET /projects/:id/snapshots/:version/diagram?layoutId=` → `{ schemaDoc, layout: visual_state | null, relationships }`. Served from Redis cache keyed `diagram:{snapshotId}:{layoutId}` (invalidated on layout/annotation write).
- `GET /projects/:id/search?q=` → searches table/column names within latest `schema_doc` via `jsonb_array_elements` (per Suggestion 3 example query), returns ranked hits with `target_path`.

**Testing**:
- `Integration: GET diagram → schemaDoc matches stored, single SQL round trip`.
- `Integration: search "email" → returns every table.column path containing email`.
- `Integration: cache hit on second call (assert no DB query via spy)`.

#### 5.2 — Layout persistence & auto-layout

**What**: Save/load `visual_state`; compute automatic layouts with ELK.

**Design**:
- `POST /projects/:id/layouts { name, snapshotId, algorithm }`, `PUT /layouts/:id { visualState }` (positions, canvas, groups, hidden_tables per Suggestion 3 `visual_state`).
- Auto-layout runs **client-side** with `elkjs`: algorithms `layered` (hierarchical) and `force`. A `POST /layouts/:id/auto-layout { algorithm }` server endpoint also exists for headless/export use, returning computed positions.
- `is_default` per project enforced (only one default).

**Testing**:
- `Integration: PUT layout → reload returns identical visual_state`.
- `Integration: set second layout default → previous default cleared`.
- `Unit (elkjs): 5-node graph → all nodes assigned non-overlapping x/y`.

#### 5.3 — React ERD canvas

**What**: The interactive diagram UI: pan/zoom, table nodes with columns, relationship edges with cardinality, collapse, search-to-highlight, dark mode.

**Design**:
- `ErdCanvas.tsx` wraps React Flow; `TableNode.tsx` renders header + column rows (PK/FK badges, types); `RelationshipEdge.tsx` renders Crow's Foot markers based on `RelationshipDoc.type`.
- Zustand store holds `schemaDoc`, `layout`, `selection`, `highlightSet`. Position changes debounced (500ms) → `PUT /layouts/:id`.
- Search box → highlight matching nodes, dim others. Collapse hides column rows. Dark mode via Tailwind `dark` class + persisted user preference.
- Toolbar: notation toggle (Crow's Foot / IDEF1X markers — standards.md), auto-layout button, export menu.

**Testing**:
- `E2E (Playwright): load project → ERD renders N table nodes and M edges`.
- `E2E: drag a table → reload page → position persisted`.
- `E2E: type "users" in search → users node highlighted, others dimmed`.
- `E2E: toggle dark mode → body has dark class, persists across reload`.
- `Unit: RelationshipEdge maps one_to_many → correct Crow's Foot marker ends`.

---

## Phase 6: Export & Documentation Generation

### Purpose
Let users get schemas out of the tool: DDL/JSON/DBML/AML/Mermaid export, an SVG/PNG diagram export, and generated Markdown/HTML documentation. Satisfies MVP "SQL export", "schema export (SQL, JSON)", and "basic documentation generation". (Requires Phases 2, 5.)

### Tasks

#### 6.1 — Schema export endpoints

**What**: Export a snapshot in any supported format.

**Design**:
- `GET /projects/:id/snapshots/:version/export?format=ddl|json|dbml|aml|mermaid|jsonschema&dialect=postgres|mysql|mssql|sqlite` → uses Phase 2 adapters; sets `Content-Disposition` and correct MIME (`application/sql`, `application/json`, `text/plain`).
- Diagram export: `GET .../export?format=svg|png&layoutId=` → render React Flow nodes to SVG server-side via `@xyflow` serialization helper; PNG via `sharp`.

**Testing**:
- `Integration: export format=ddl&dialect=mysql → parses back to equivalent SchemaDoc`.
- `Integration: export format=mermaid → output starts with "erDiagram"`.
- `Integration: unsupported format → 400 problem+json`.
- `E2E: export menu → DBML download has correct filename`.

#### 6.2 — Documentation generation

**What**: Generate browsable Markdown/HTML docs per table (SchemaSpy-style) including descriptions, columns, relationships, and an embedded Mermaid diagram.

**Design**:
- `generateDocs(doc, layout?, template): { files: Record<string,string> }` in `@dbviz/core` using a Handlebars template set (`docs-template/`); default template lists tables with anchored sections, column tables, inbound/outbound relationships, and a per-table Mermaid sub-diagram.
- `GET /projects/:id/snapshots/:version/docs?template=default&format=html|md` → returns a zip (`archiver`) of generated files.

**Testing**:
- `Unit: generateDocs over 3-table doc → one section per table, relationships listed both directions`.
- `Unit: AI descriptions present → included; absent → omitted gracefully`.
- `Integration: docs?format=html → zip contains index.html + per-table pages`.

---

## Phase 7: Versioning, Diffs & Change History

### Purpose
Turn the snapshot sequence into a first-class change-history feature: compute structured diffs between any two versions, store them, and render an annotated timeline. This is research.md's emerging differentiator (change history overlaid on the diagram). (Requires Phases 2, 5.)

### Tasks

#### 7.1 — Diff engine

**What**: Compute a structured `diff_doc` between two `SchemaDoc`s.

**Design**:
```ts
export interface SchemaDiff {
  tablesAdded: string[]; tablesDropped: string[];
  tablesModified: Array<{ table: string;
    columnsAdded: ColumnDoc[]; columnsDropped: string[];
    columnsModified: Array<{ name: string; changes: Record<string,{from:unknown;to:unknown}> }> }>;
  relationshipsAdded: RelationshipDoc[]; relationshipsDropped: RelationshipDoc[];
  summaryStats: { totalChanges: number; breakingChanges: number };
}
export function diffSchemas(from: SchemaDoc, to: SchemaDoc): SchemaDiff;
```
- Match tables/columns by qualified name. Breaking changes: dropped table/column, narrowed type, added NOT NULL without default, dropped FK.

**Testing**:
- `Unit: add table → tablesAdded; drop column → breakingChanges≥1`.
- `Unit: widen varchar(100)→varchar(255) → modified, not breaking`.
- `Unit: identical docs → totalChanges=0`.
- `Fixture: two real snapshot fixtures → expected diff snapshot test`.

#### 7.2 — Diff API & storage

**What**: Persist computed diffs and expose them.

**Design**:
- `GET /projects/:id/diffs?from=&to=` → returns cached `schema_diffs` row or computes, stores, returns. `UNIQUE(project_id, from_version, to_version)`.
- `GET /projects/:id/timeline` → merge `schema_snapshots` + `activity_log` into a reverse-chronological feed.

**Testing**:
- `Integration: GET diffs first call computes+stores; second call reads stored (spy on diff engine)`.
- `Integration: timeline → snapshots and activity interleaved by time desc`.

#### 7.3 — Diff visualization on canvas

**What**: Overlay a diff on the ERD — added (green), dropped (red), modified (amber) tables/columns; a side panel listing changes.

**Design**:
- `GET diagram?compareWith=<version>` augments node/column data with `changeStatus`. Canvas applies status colors; legend shown.
- Side `DiffPanel.tsx` lists changes grouped by table, click → focus node.

**Testing**:
- `E2E: select v1 vs v2 → added table node green, dropped table red`.
- `E2E: click a change in DiffPanel → canvas pans to and selects that node`.

---

## Phase 8: AI-Native Capabilities

### Purpose
Deliver the differentiating AI layer: plain-English annotations, anti-pattern detection, natural-language schema queries, migration-plan generation, and AI changelogs. These are the features no incumbent unifies (README AI-Native Advantage). (Requires Phases 2, 3, 5, 7.)

### Tasks

#### 8.1 — LLM client abstraction & prompt library

**What**: A provider-agnostic `LLMClient` with structured-output helpers and versioned prompt templates.

**Design**:
```ts
export interface LLMClient {
  complete(opts: { system: string; user: string; maxTokens?: number }): Promise<string>;
  completeJson<T>(opts: { system: string; user: string; schema: z.ZodType<T> }): Promise<T>; // validates, 1 retry on parse fail
}
```
- Anthropic implementation; `completeJson` instructs the model to return JSON and validates with Zod.
- Prompts in `@dbviz/llm/prompts`, each exporting `{ version, system, buildUser(input) }`. Never send live data rows to the LLM — only schema metadata.

**Testing**:
- `Unit (mocked SDK): completeJson valid JSON → typed object`.
- `Unit: first response invalid JSON, retry valid → returns parsed; both invalid → throws`.

#### 8.2 — Entity annotation

**What**: Generate plain-English descriptions for tables/columns from names, types, and FK context.

**Design**:
- `ai-annotate.job.ts`: for each table, build prompt with the table's columns + inbound/outbound relationships (from the graph) → `completeJson` returning `{ table: { description }, columns: [{ name, description }] }` → write back into `schema_doc` (`aiDescription` fields) as a new snapshot OR patch in place (configurable; default: update current snapshot's doc).
- `POST /projects/:id/snapshots/:version/annotate` → enqueues job, `202`.

**Testing**:
- `Unit (mocked LLM): annotate job → schema_doc tables gain aiDescription`.
- `Integration: prompt payload contains no sample data rows`.
- `Unit: malformed LLM output for one table → that table skipped, others annotated, job succeeds`.

#### 8.3 — Anti-pattern detection (rules + AI)

**What**: Detect normalization violations and anti-patterns; store as `annotations` with severity.

**Design**:
- Rule-based first (`@dbviz/core/anti-patterns`): missing index on FK column, EAV shape (entity/attribute/value column pattern), implicit many-to-many (junction via `junctionTables`), wide tables (>50 cols), missing PK, circular FK (`findCycles`).
- AI pass adds explanation + refactor suggestion per finding via `anti-pattern` prompt.
- `POST /projects/:id/snapshots/:version/analyze` → runs rules, optionally AI-enriches, writes `annotations` (`annotation_type='anti_pattern'`, `is_ai_generated`).

**Testing**:
- `Unit: table with FK and no covering index → "missing_fk_index" finding`.
- `Unit: table with no primary key → "missing_pk" finding`.
- `Unit: EAV table (entity_id, attribute, value) → "eav" finding`.
- `Integration (mocked LLM): analyze → annotations rows created with severity`.

#### 8.4 — Natural-language schema query

**What**: Answer questions like "which tables are involved in order fulfilment?" with a highlighted sub-diagram.

**Design**:
- `POST /projects/:id/snapshots/:version/ask { question }` → prompt includes compact schema summary (table+column names, relationships) → `completeJson` returns `{ tables: string[], relationships: string[], explanation }`.
- Response drives canvas highlight (reuse Phase 5 `highlightSet`). Validate returned table names exist; drop unknowns.

**Testing**:
- `Unit (mocked LLM): returns table list → unknown table names filtered out`.
- `E2E: ask question → matching nodes highlighted, explanation shown`.

#### 8.5 — Migration-plan generation & AI changelog

**What**: From a plain-English change request, produce ordered `ALTER TABLE` steps with rollback; and generate human-readable changelogs from a `SchemaDiff`.

**Design**:
- `POST /projects/:id/migration-plan { description, fromVersion }` → prompt with current schema + request → `completeJson` `{ steps: [{ forward: string; rollback: string; rationale: string }] }`. Steps are SQL strings; not executed (planning only) — labeled clearly.
- Changelog: `diff.job.ts` calls `changelog` prompt over `diff_doc` → stores `schema_diffs.ai_changelog`.

**Testing**:
- `Unit (mocked LLM): migration-plan → each step has forward+rollback`.
- `Unit: changelog generated from diff fixture → mentions added/dropped tables`.
- `Integration: GET diffs includes ai_changelog after job runs`.

---

## Phase 9: MCP Server & Collaboration

### Purpose
Expose the schema graph over MCP (the key unexploited differentiator from standards.md) so AI coding assistants can query data models directly, and add real-time collaboration (comments, presence). (Requires Phases 2, 4, 5, 8.)

### Tasks

#### 9.1 — MCP server

**What**: An MCP server exposing a project's schema as resources and tools.

**Design**:
- `@dbviz/mcp` using `@modelcontextprotocol/sdk`, transport: stdio (local) and streamable HTTP (hosted), authed via API token.
- Resources: `schema://{projectId}/latest` → `schema_doc`; `schema://{projectId}/v{n}`.
- Tools: `list_tables(projectId)`, `describe_table(projectId, table)`, `find_related(projectId, table, hops)` (uses Phase 2 graph), `search_schema(projectId, query)`, `get_changelog(projectId, from, to)`.
- API-token auth: `Authorization: Bearer <token>`; tokens issued per user, scoped to readable projects.

**Testing**:
- `Integration (MCP test client): list_tables → returns latest snapshot tables`.
- `Integration: find_related(orders,2) → matches graph.reachableFrom`.
- `Integration: token without project access → tool returns authorization error`.

#### 9.2 — Comments & annotations API

**What**: Threaded comments and user annotations anchored to `target_path`.

**Design**:
- `POST /projects/:id/comments { targetPath?, parentId?, body }`, `GET /projects/:id/comments?targetPath=`, `PATCH`, `DELETE` (author or admin).
- Annotations CRUD (`note`/`warning`), distinct from AI-generated ones.

**Testing**:
- `Integration: post reply (parentId) → thread retrievable`.
- `Integration: non-author edit → 403; admin edit → allowed`.

#### 9.3 — Real-time collaboration (WebSocket)

**What**: Live presence and broadcast of layout/comment changes within a project room.

**Design**:
- `ws/collab.ts`: `@fastify/websocket`; clients join room `project:{id}` after session auth. Redis pub/sub fans out across API instances.
- Messages: `presence` (cursor, user), `layout_updated` (broadcast after PUT), `comment_added`. Last-write-wins for positions (per Suggestion 2 note that diagram edits merge by ordering).

**Testing**:
- `Integration: two clients in room; one moves table → other receives layout_updated`.
- `Integration: unauthenticated ws connect → closed with 4401`.
- `E2E: two browser contexts → presence cursors visible to each other`.

---

## Phase 10: Packaging, Deployment & Hardening

### Purpose
Make the system deployable, observable, and secure for self-hosting. After this phase a user runs `docker compose up` and has a working instance; CI publishes images. (Requires all prior phases.)

### Tasks

#### 10.1 — Docker & compose

**What**: Production Dockerfiles and a one-command compose stack.

**Design**:
- Multi-stage `Dockerfile.api`, `Dockerfile.worker`, `Dockerfile.web` (web served by nginx, API URL injected at runtime).
- `docker-compose.yml`: `postgres:16`, `redis:7`, `api`, `worker`, `web`, with healthchecks and a one-shot `migrate` service running drizzle migrations before `api` starts.

**Testing**:
- `E2E (CI): docker compose up → /healthz returns 200 within 60s`.
- `Integration: migrate service applies all migrations to fresh postgres`.

#### 10.2 — Observability & rate limiting

**What**: Structured logging, metrics, and request throttling.

**Design**:
- `pino` JSON logs with request IDs; `/metrics` Prometheus endpoint (`prom-client`): request latency histogram, job durations, LLM token counters.
- `@fastify/rate-limit` on auth and AI endpoints (AI endpoints stricter; per-user token budget tracked in Redis).

**Testing**:
- `Integration: exceed AI rate limit → 429 problem+json`.
- `Unit: /metrics exposes http_request_duration_seconds`.

#### 10.3 — Security hardening

**What**: Apply security controls before release.

**Design**:
- Helmet headers, CORS allowlist from `PUBLIC_API_URL`, CSRF token for cookie-auth state-changing routes (or rely on `SameSite=Lax` + custom header check).
- Introspection connections forced read-only; SSRF guard rejects connecting to internal metadata IP ranges unless explicitly allowed.
- Secrets never logged; `DBVIZ_MASTER_KEY` rotation procedure documented in README.
- Dependency audit (`pnpm audit`) gate in CI.

**Testing**:
- `Integration: connection host = 169.254.169.254 → rejected by SSRF guard`.
- `Integration: state-changing request without CSRF/custom header → 403`.
- `Unit: log redaction strips password fields from connection objects`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Scaffolding         ─── required by everything
   ├── Phase 2: SchemaDoc Core & Formats   ─── requires P1; parallel with P3, P4
   ├── Phase 3: Live DB Introspection       ─── requires P1, P2; parallel with P4
   └── Phase 4: Auth, Projects, Tenancy     ─── requires P1; parallel with P2, P3
                     │
Phase 5: Diagram API & Canvas              ─── requires P2, P3, P4
   ├── Phase 6: Export & Documentation      ─── requires P2, P5; parallel with P7
   └── Phase 7: Versioning, Diffs, History  ─── requires P2, P5; parallel with P6
                     │
Phase 8: AI-Native Capabilities            ─── requires P2, P3, P5, P7
                     │
Phase 9: MCP & Collaboration               ─── requires P2, P4, P5, P8
                     │
Phase 10: Packaging, Deploy, Hardening     ─── requires all prior phases
```

Parallelism: Phases 2, 3, 4 can proceed concurrently after Phase 1 (different packages, shared types from `@dbviz/core` defined first in 2.1). Phases 6 and 7 can proceed concurrently after Phase 5.

MVP cut line (features.md "Must-have"): Phases 1–6 deliver the full MVP (interactive ERD, browsing, relationship detection, SQL/JSON export, sharing via auth, basic docs, dark mode). Phases 7–9 deliver the "Should-have"/AI differentiators; Phase 10 is release engineering.

---

## Definition of Done (per phase)

Every phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests for the phase pass (`pnpm test`); coverage of new modules ≥ 80%.
3. Integration tests requiring real databases run against Testcontainers and pass in CI.
4. `biome check` (lint + format) passes with no errors.
5. `pnpm -r typecheck` (`tsc --noEmit`) passes with no errors.
6. New API endpoints appear in the generated `/openapi.json` (OpenAPI 3.1) and validate.
7. New Zod request/response schemas validate both happy-path and at least one edge case.
8. Database changes are captured as a checked-in drizzle migration and apply cleanly to a fresh database.
9. New environment variables are added to `.env.example` and documented.
10. The feature works end-to-end (Playwright E2E green for user-facing phases).
11. `docker compose up` still reaches a healthy state (`/healthz` 200) — verified from Phase 10 onward.
12. No secrets (DB credentials, API keys, session tokens) appear in logs or API responses.
```

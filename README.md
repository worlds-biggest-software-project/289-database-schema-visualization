# Database Schema Visualization

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source ERD tool that turns any live database into an interactive, annotated, and query-aware schema diagram with change history.

Database Schema Visualization is an interactive ERD platform for backend developers, data engineers, and DBAs who need to understand, document, and evolve database schemas. It combines reverse engineering of live databases with change history and query examples, then layers AI-driven annotation, anti-pattern detection, and migration planning on top.

---

## Why Database Schema Visualization?

- Incumbents fragment the workflow: DbSchema is desktop-first, dbdiagram.io has limited reverse engineering from a live database, and DBeaver treats ERD as a secondary feature.
- Collaboration-focused SaaS tools (DrawSQL, Vertabelo) charge $7–$179/mo per team and still lack deep schema-sync or analytics features.
- SchemaSpy is free and automated but produces static HTML with no editing or AI annotation.
- pgModeler is Postgres-only; most teams run polyglot stacks and need multi-engine support.
- Schema-as-code formats (DBML, Mermaid ERD) are growing, but no incumbent unifies live-DB introspection, version history, query examples, and AI-assisted refactoring in one open-source tool.

---

## Key Features

### Core Visualization & Browsing

- Interactive Entity-Relationship Diagram (ERD) visualization
- Table and column browsing
- Relationship detection and visualization
- Table search and filtering
- Dark mode support

### Schema Export & Documentation

- SQL script generation
- Schema export (SQL, JSON)
- Basic documentation generation
- Documentation templates
- Sample data display

### Versioning & Collaboration

- Version control and change history
- Collaboration and commenting
- User management and sharing
- Data lineage tracking

### Query & Performance Insight

- Query visualization and explanation
- Automatic layout algorithms
- Performance metrics display

### AI-Augmented Capabilities (backlog)

- Automatic schema optimization suggestions
- Security analysis and exposure detection
- Query performance prediction
- Multi-database schema federation
- AI-powered schema design suggestions
- Data profiling and anomaly detection
- Time-travel schema visualization

---

## AI-Native Advantage

AI is used to annotate diagram entities with plain-English descriptions inferred from column names, foreign keys, and query history, and to detect normalisation violations and anti-patterns such as EAV tables, implicit many-to-many relationships, and missing indexes on foreign keys. A natural-language interface lets users ask questions like "which tables are involved in the order fulfilment flow?" and receive a highlighted sub-diagram. AI also generates ordered ALTER TABLE migration plans with rollback steps from a plain-English description, and produces human-readable changelogs that compare two schema snapshots.

---

## Tech Stack & Deployment

The project targets reverse engineering from live databases via the SQL Information Schema (ISO/IEC 9075), with import/export through standard DDL. Schema-as-code interoperability is supported through DBML and Mermaid / PlantUML ERD for embedding diagrams in Markdown and wikis. IDEF1X notation is supported for enterprise data modelling. Deployment modes and SDKs are not yet specified.

---

## Market Context

The broader database management tools market is estimated at USD 8.4 billion in 2025, with schema visualisation a fragmented sub-segment often bundled into database clients. Incumbent pricing ranges from free tiers (ChartDB, SchemaSpy, DbSchema Community) to team plans at $7–$59/mo and DBeaver Pro at $199/yr. Primary buyers are backend developers onboarding to unfamiliar codebases, data engineers documenting warehouse schemas, DBAs reviewing migrations, and product managers needing business-readable data models.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.

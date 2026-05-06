# Standards & API Reference

> Project: Database Schema Visualization · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 9075:2023 — SQL (Information technology — Database languages SQL)**
- Official URL: https://www.iso.org/standard/76583.html
- The overarching SQL standard (16 parts). Part 11 — SQL/Schemata — defines the `INFORMATION_SCHEMA` views that expose table, column, constraint, and key metadata. Every reverse-engineering and schema-import tool queries these standard views to introspect live databases in a database-agnostic way; they are foundational to any tool that connects to a live database.

**ISO/IEC 9075-11:2023 — SQL/Schemata (Information and Definition Schemas)**
- Official URL: https://www.iso.org/standard/76583.html (part of the 2023 family)
- Specifies the Information Schema and Definition Schema including `INFORMATION_SCHEMA.TABLES`, `INFORMATION_SCHEMA.COLUMNS`, `INFORMATION_SCHEMA.TABLE_CONSTRAINTS`, `INFORMATION_SCHEMA.REFERENTIAL_CONSTRAINTS`. These views are the primary introspection surface for schema visualisation tools when connecting to PostgreSQL, MySQL, SQL Server, MariaDB, and SQLite.

---

### Data Model & Diagram Notation Standards

**IDEF1X (Integration DEFinition for Information Modeling)**
- Published as FIPS Publication 184 (NIST, 1993); adopted as IEEE standard in 2012.
- Official URL: https://www.idef.com/idef1x-data-modeling-method/
- A formal data-modelling notation used extensively in US government and enterprise settings. Defines independent/dependent entity notation, identifying/non-identifying relationships, and cardinality markers. Supported by DbSchema and Vertabelo. A schema visualisation tool targeting enterprise users should support IDEF1X rendering alongside Chen and Crow's Foot notations.

**DBML — Database Markup Language**
- Open source; maintained by Holistics. Not a formal ISO/IEEE standard but de facto industry DSL.
- Specification URL: https://dbml.dbdiagram.io/docs/
- GitHub: https://github.com/holistics/dbml
- A concise, human-readable text format for defining database schemas (tables, columns, indexes, references). Popularised by dbdiagram.io with 2.5 M+ docs created. Provides an npm library for programmatic DDL ↔ DBML conversion. A schema visualisation tool should support DBML import/export as a primary schema-as-code workflow.

**AML — Azimutt Markup Language**
- Open source (MIT); maintained by the Azimutt project.
- Specification URL: https://azimutt.app/docs/aml
- npm package: `@azimutt/aml`
- YAML-based DSL for database schema definition with support for nested columns (document databases), polymorphic relations, and custom properties. Positioned as a more expressive alternative to DBML, particularly for non-relational schemas.

**Mermaid ERD Syntax**
- Open source (MIT). Part of the Mermaid diagramming-as-code ecosystem.
- Official syntax reference: https://mermaid.js.org/syntax/entityRelationshipDiagram.html
- Lightweight text-to-diagram format natively rendered by GitHub, GitLab, Notion, and Obsidian. Supports entity definitions, cardinality (zero-or-one through one-or-more), identifying vs. non-identifying relationships, and attribute-level primary key notation. Increasingly used to embed schema diagrams in README files and wikis.

**PlantUML ER and IE Diagrams**
- Open source (Apache 2.0).
- Official URL: https://plantuml.com/ie-diagram (Information Engineering); https://plantuml.com/er-diagram (Chen notation)
- Supports both Information Engineering notation (extend of class diagrams with mandatory-attribute markers) and Chen's original ER notation. Useful for generating schema diagrams inside CI/CD pipelines and documentation builders.

---

### Data Interchange & API Specifications

**OpenAPI Specification 3.2.0**
- Maintained by the OpenAPI Initiative (Linux Foundation).
- Official URL: https://spec.openapis.org/oas/v3.2.0.html; https://www.openapis.org/
- Latest stable version (September 2025). The standard format for describing REST APIs via JSON or YAML. Any schema visualisation tool that exposes an HTTP API for schema import/export or embedding should publish an OpenAPI document. Also directly relevant as a visualisation target — OpenAPI's `components/schemas` block is itself a schema graph that visualisation tools may want to render.

**JSON Schema (Draft 2020-12)**
- Maintained by the JSON Schema project.
- Official URL: https://json-schema.org/specification; https://json-schema.org/draft/2020-12
- The vocabulary for describing and validating JSON data structures. Used as the schema-exchange wire format by several database tools and by OpenAPI 3.1+. A schema visualiser should be able to ingest JSON Schema documents as an input source (alongside SQL DDL and DBML).

**GraphQL Schema Definition Language (SDL) & Introspection**
- Maintained by the GraphQL Foundation.
- SDL spec: https://graphql.org/learn/schema/
- Introspection spec: https://graphql.org/learn/introspection/
- GraphQL APIs expose their schema via the `__schema` and `__type` introspection fields, making schema discovery programmatic and language-agnostic. Tools such as GraphQL Voyager visualise the resulting graph. A full-stack schema visualiser should treat GraphQL SDL and introspection results as first-class import formats alongside relational DDL.

---

### Security & Authentication Standards

**OAuth 2.0 — RFC 6749**
- IETF RFC. Official URL: https://www.rfc-editor.org/rfc/rfc6749.html
- The standard authorisation framework used by SaaS schema tools (Azimutt, Vertabelo, dbdiagram.io) for delegated database credential management and team member authentication. Any tool offering cloud-hosted schema storage or team access must implement OAuth 2.0 / Authorization Code + PKCE flow.

**OpenID Connect 1.0 (OIDC)**
- Built on OAuth 2.0. Official URL: https://openid.net/connect/
- Adds identity assertions to OAuth 2.0 token flows. Used by SSO integrations in enterprise schema tools. Required for GitHub/Google/Okta sign-in, which is expected in any team-facing tool.

---

### MCP Server Specifications

**Model Context Protocol (MCP) — 2025-11-25 Specification**
- Maintained by Anthropic. Official URL: https://modelcontextprotocol.io/specification/2025-11-25
- GitHub: https://github.com/modelcontextprotocol/modelcontextprotocol
- MCP is an open protocol for exposing tools and data sources to AI agents. Multiple database MCP servers already exist (Azure PostgreSQL MCP, AWS database MCP, db-mcp-server). A schema visualisation tool that exposes its schema graph over MCP would allow AI coding assistants to answer questions about data models directly. This is a significant AI-native integration surface and a key differentiator for any new entrant.

---

## Similar Products — Developer Documentation & APIs

### Azimutt
- **Description:** Open-source database explorer and analyser supporting ERD visualisation, data navigation, documentation, and schema analysis across 12+ database engines.
- **API Documentation:** https://azimutt.app/docs/api
- **SDKs/Libraries:** `@azimutt/aml` (npm) for AML parsing/generation; GitHub Actions workflow for automated schema push
- **Developer Guide:** https://github.com/azimuttapp/azimutt
- **Standards:** REST/JSON HTTP API; AML (own DSL); SQL DDL; DBML import
- **Authentication:** API token (bearer); OAuth planned for enterprise edition

### ChartDB
- **Description:** Open-source, browser-based database diagram editor that imports schemas via a single introspection SQL query and exports DDL in any target dialect using AI.
- **API Documentation:** https://docs.chartdb.io/docs/welcome
- **SDKs/Libraries:** Self-hostable via Docker; no standalone SDK published
- **Developer Guide:** https://github.com/chartdb/chartdb
- **Standards:** REST/JSON; SQL DDL import; OpenAPI export planned
- **Authentication:** No authentication in open-source self-hosted variant; cloud (chartdb.io) uses email/OAuth

### dbdiagram.io / dbdocs.io (Holistics)
- **Description:** Browser-based ERD design tool (dbdiagram.io) and database documentation builder (dbdocs.io) driven by DBML syntax; 2.5 M+ schemas created.
- **API Documentation:** https://docs.dbdiagram.io/
- **SDKs/Libraries:** `@holistics/dbml-core` (npm) — parse DBML, export SQL DDL for PostgreSQL, MySQL, MSSQL, etc.
- **Developer Guide:** https://dbml.dbdiagram.io/docs/
- **Standards:** DBML DSL; SQL DDL import/export; REST/JSON
- **Authentication:** GitHub OAuth; email sign-in

### Vertabelo
- **Description:** Cloud-based visual database modeller with forward/reverse engineering, team workspaces, and SQL generation for multiple engines.
- **API Documentation:** https://vertabelo.com/documentation; https://vertabelo.com/blog/new-public-api/
- **SDKs/Libraries:** Document-oriented HTTP API (token-authenticated); exports XML, SQL, PDF, HTML, DOCX
- **Developer Guide:** https://support.vertabelo.com/
- **Standards:** REST/JSON; SQL DDL; XML schema export; OpenAPI not documented
- **Authentication:** API token (bearer); Google/GitHub OAuth for UI login

### DBeaver / CloudBeaver
- **Description:** Universal database client (DBeaver desktop; CloudBeaver web) supporting 70+ databases with ERD generation, SQL editor, data viewer, and team features.
- **API Documentation:** https://dbeaver.com/docs/cloudbeaver/API-and-integration/; https://github.com/dbeaver/cloudbeaver/wiki/API-and-integration
- **SDKs/Libraries:** CloudBeaver exposes a GraphQL API; MCP server (introduced in DBeaver 26.x)
- **Developer Guide:** https://dbeaver.com/docs/dbeaver/
- **Standards:** GraphQL API; JDBC driver integration; REST via CData adapters
- **Authentication:** LDAP; SSO/SAML; local user management; OAuth planned

### SchemaSpy
- **Description:** Open-source Java tool generating interactive HTML documentation of database schemas with ER diagrams from any JDBC-connected database.
- **API Documentation:** https://schemaspy.readthedocs.io/en/latest/started.html
- **SDKs/Libraries:** Java CLI / Docker image; JDBC driver plug-in model; configuration via `schemaspy.properties`
- **Developer Guide:** https://github.com/schemaspy/schemaspy
- **Standards:** JDBC (any driver); SQL `INFORMATION_SCHEMA` introspection; HTML/SVG output (Graphviz); no HTTP API
- **Authentication:** Database credentials passed via config file; no user auth layer (output is static HTML)

### DrawSQL (Atlassian)
- **Description:** Collaborative ERD tool acquired by Atlassian in 2022; supports MySQL, PostgreSQL, SQL Server DDL import/export and Laravel migration generation.
- **API Documentation:** https://drawsql.app/docs/export-to-sql-ddl; https://drawsql.app/docs/import-from-ddl
- **SDKs/Libraries:** No public SDK; DDL import/export via UI; JSON export available
- **Developer Guide:** https://drawsql.app/docs/
- **Standards:** SQL DDL (MySQL, PostgreSQL, SQL Server); Laravel migration format; JSON export
- **Authentication:** Email; Google OAuth; Atlassian SSO for enterprise plans

### GraphQL Voyager
- **Description:** Open-source tool that renders any GraphQL API schema as an interactive graph using introspection data.
- **API Documentation:** https://graphql.org/learn/introspection/
- **SDKs/Libraries:** React component (`graphql-voyager` npm package); embeddable in any web application
- **Developer Guide:** https://github.com/graphql-kit/graphql-voyager
- **Standards:** GraphQL SDL & introspection (spec: graphql.org/learn/schema/); REST/JSON for introspection transport
- **Authentication:** Passes custom HTTP headers to introspection endpoint (supports bearer tokens, API keys)

---

## Notes

- **MCP as a first-class integration surface:** The MCP ecosystem is growing rapidly in 2025–2026, with AWS, Azure, and Anthropic all publishing database MCP servers. A schema visualisation tool that exposes its schema graph as an MCP resource would allow AI coding assistants to answer data-model questions without any additional setup — a differentiator no current ERD tool has exploited fully.
- **Schema-as-code convergence:** DBML, AML, Mermaid ERD, and PlantUML IE all pursue the same goal — schema stored as text in version control — but none has achieved universal adoption. An AI-native tool could auto-convert between all four formats plus SQL DDL, acting as a universal schema language bridge.
- **GraphQL schema visualisation is underserved:** Most tools focus on relational (SQL) schemas. GraphQL SDL and introspection produce rich, browsable schema graphs but only GraphQL Voyager addresses this niche; integration with relational schema tools is largely absent.
- **OpenAPI 3.2 / 4.0 trajectory:** OpenAPI 4.0 ("Moonwalk") is in early development. Tooling that targets OpenAPI schema components as a visualisation source should track this closely to remain compatible.

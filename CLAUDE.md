# ERD Generation Guide for Autoscal Services

This guide documents the architecture, design decisions, and step-by-step workflow for generating interactive Entity Relationship Diagrams (ERDs) and normalized graph JSON models from the service database architecture documentation in `current-db-arch/{SERVICE_NAME}.md`.

Following this guide ensures any service (e.g., `job-service-node`, `attendance-management-service-django`, `auth-service-node`, `finance-service-node`, `calendar-schedular-backend`, `notification`) produces a consistent, production-grade ERD.

## Migration Scope

- `spark-assist-ai` is **out of scope** for the monolith migration. It stays as a separate service with its own database for now.
- Do not merge its tables into the consolidated schema, and do not count them in consolidation totals or mapping registers.
- Its ERD and architecture doc stay for reference only. Treat its tables as an external service when designing the single schema (cross-service ids such as `api_tenant.external_id` and `api_user.external_id` remain logical references).

---

## 1. Deliverables & Output Specification

For any target service configured via `SERVICE_NAME`:

- **Input Architecture Document**:
  `current-db-arch/{SERVICE_NAME}.md`
- **Output Directory**:
  `current-db-arch/erds/`
- **Required Deliverables**:
  1. `current-db-arch/erds/{SERVICE_NAME}-erd.json` (Normalized graph data)
  2. `current-db-arch/erds/{SERVICE_NAME}-erd.html` (Single-file interactive viewer)
  3. `current-db-arch/erds/{SERVICE_NAME}-erd.svg` (Optional static diagram if Graphviz is available)

---

## 2. Core Architectural & UI Decisions

These decisions were established during development and must be preserved across all services:

### A. Zero External Dependencies & Offline Operation
- **Single-File Deliverable**: The HTML diagram must open directly from the local filesystem (`file://`) without requiring a development server, bundler, or runtime network access.
- **Inlined Visualization Engine**: Cytoscape.js (v3.28.1 minified, ~357 KB) is embedded directly inside the `<script>` tag of the HTML file.
- **Inlined Graph Data**: The complete JSON graph payload is embedded directly into the HTML (`const graphData = {...};`).

### B. Theme: Light Mode by Default
- The default interface theme is **Light Mode** (`<html lang="en" data-theme="light">`).
- A prominent theme toggle button (`🌙 Dark Mode` / `☀️ Light Mode`) in the header lets users seamlessly toggle between a clean light canvas (white/slate background, high contrast dark text) and dark mode (navy/slate background with glowing accents).
- High-resolution PNG export automatically matches the active theme.

### C. Layout & Overlap Protection
- **Canvas Wrapper Isolation**: The Diagram Legend and floating zoom controls are children of `.canvas-wrapper`, positioned strictly relative to the canvas viewport. They never overlap the left sidebar or obscure module filters regardless of scroll position or sidebar expansion.
- **Collapsible Legend**: The legend includes an expand/collapse toggle (`▼` / `▲`) to allow minimizing into a compact bar when maximum canvas visibility is needed.
- **Force-Directed COSE Layout**: Default physics layout groups interconnected tables naturally, with optional switchers for Concentric (connectivity-based), Breadthfirst (hierarchical), and Grid layouts.

### D. Deep Drawer Navigation: History Stack & Breadcrumbs
Navigating through foreign key relationships inside the right-hand inspection drawer can lead several hops deep (e.g. `team_members` -> `position` -> `departments`). To avoid losing context:
- **Back & Forward Buttons**: Header of the inspector drawer includes `← Back (<table_name>)` and `Forward →` buttons bound to an exploration history stack.
- **Interactive Breadcrumb Path**: A breadcrumb strip (`table1 › table2 › table3`) renders below the navigation bar. Clicking any ancestor table immediately re-centers the canvas, highlights that table's neighborhood, and updates the drawer.

### E. Graph Model & Visual Density Rules
- **Node Contents**: Diagram nodes display **only** the table name (bold) and a concise 1-2 sentence description. Full column dumps must NOT be placed inside canvas nodes (detailed column tables belong in the markdown architecture documents).
- **Edge Direction**: Directed edges run from **Parent (PK)** to **Child (FK)** (e.g., `projects` -> `timesheets`), matching the Mermaid diagrams in the architecture documents.
- **Physical vs Conceptual Relationships**:
  - **Physical Foreign Keys**: Rendered as solid lines with cardinality labels (`1:1`, `1:n`, `m:n`), enabled by default.
  - **Conceptual Many-to-Many Overlays**: Extracted from explicit junction tables (e.g. `team_member_projects` -> `projects` <-> `team_members`). Rendered as distinct dashed purple lines (`m:n`), toggled via relationship filters.
- **Isolated Tables**: Tables with zero foreign keys (inbound or outbound) are included in the graph, styled with distinct dotted borders, and can be filtered or highlighted using the "Show Isolated Tables" toggle.

---

## 3. Parsing Architecture Markdown Documents

The architecture documents in `current-db-arch/` follow one of two formatting styles. Parsers must accommodate both styles:

### Format A: Bullet-List Relationships (e.g., `job-service-node.md`)
- Relationships are formatted as bullet items under `**Relationships**`:
  `- Out: \`col\` → \`target_table\`.\`target_col\`; cardinality; FK constraint \`name\`; onDelete info`
  `- In: \`source_table\`.\`source_col\`; cardinality; FK constraint \`name\``
- **Critical Regex Gotcha**: Module names in parentheses can contain nested parentheses, such as `(Recruitment (ATS))`. Avoid restrictive regex like `\([^)]*\)`. Use non-semicolon matching:
  ```javascript
  /Out:\s*`([^`]+)`\s*→\s*`([^`]+)`\.`([^`]+)`(?:[^\r\n;]*);\s*([^;]+);\s*([^;]+)(?:;\s*onDelete\s+(.+))?/
  ```

### Format B: Table-Based Relationships (e.g., `attendance-management-service-django.md`, `auth-service-node.md`)
- Relationships are formatted as a markdown table under `**Relationships**`:
  ```markdown
  | Direction | Related table | Columns | Cardinality | Enforcement | Notes |
  | Outbound | [account_employee](#t-account_employee) | adjusted_by_id -> employee_id | N:1 | DB FK ... | ... |
  ```
- Parse rows where Direction is `Outbound` or `Out`. Extract source column, target table, target column, cardinality, and constraint info.

### Cardinality Normalization
Normalize raw document descriptions into explicit notation:
```javascript
function inferCardinality(desc) {
  if (/1\s*[→→]\s*(?:1|0\.\.1)/i.test(desc) || /one-to-one/i.test(desc) || /1:1/.test(desc) || /0\.\.1:1/.test(desc)) return '1:1';
  if (/many-to-many/i.test(desc) || /m2m/i.test(desc) || /m:n/i.test(desc)) return 'm:n';
  if (/many\s*[→→]\s*1/i.test(desc) || /many-to-one/i.test(desc) || /N:1/i.test(desc)) return '1:n';
  if (/1\s*[→→]\s*many/i.test(desc) || /one-to-many/i.test(desc) || /1:N/i.test(desc)) return '1:n';
  if (/many\s*[→→]\s*0\.\.1/i.test(desc)) return '1:n';
  return '1:n';
}
```

### Table & Module Extraction
1. **Table Names**: Parse from `#### <table_name>` or `#### \`<table_name>\`` headers, cross-checked against the `Table Inventory` / `Full table register` table.
2. **Table Status**: `active`, `legacy`, `apparently_unused`, or `uncertain`.
3. **Module Assignment**: Parse from the `Module summary` or module table indexes. Any legacy/framework tables without an application module must be assigned to a baseline module (e.g., `Django legacy baseline (SQL only)`).
4. **Descriptions**: Extract from table `Purpose:` bullet point, `Schema comment:` or fallback summaries.

---

## 4. Normalized JSON Schema Specification

The companion JSON file (`current-db-arch/erds/{SERVICE_NAME}-erd.json`) must adhere to this schema:

```json
{
  "service": "job-service-node",
  "generatedAt": "2026-10-15T06:22:34.930Z",
  "stats": {
    "totalTables": 167,
    "physicalRelationships": 256,
    "conceptualRelationships": 24,
    "inferredRelationships": 0,
    "totalRelationships": 280,
    "modules": 21,
    "isolatedTables": 27,
    "cardinalityBreakdown": {
      "1:1": 8,
      "1:n": 238,
      "m:n": 10
    },
    "statusBreakdown": {
      "active": 148,
      "legacy": 14,
      "apparently_unused": 2,
      "uncertain": 3
    }
  },
  "modules": ["People and org", "Recruitment (ATS)", "..."],
  "nodes": [
    {
      "id": "public.team_members",
      "name": "team_members",
      "description": "Employee master records; central hub table referenced by 74 other tables across 17 modules.",
      "modules": ["People and org", "Recruitment (ATS)"],
      "status": "active",
      "evidence": ["current-db-arch/job-service-node.md#team_members"]
    }
  ],
  "edges": [
    {
      "id": "team_members_buddy_id_fkey",
      "source": "public.team_members",
      "target": "public.team_members",
      "cardinality": "1:n",
      "kind": "physical_foreign_key",
      "sourceColumn": "id",
      "targetColumn": "buddy_id",
      "enforced": true,
      "constraintName": "team_members_buddy_id_fkey",
      "onDelete": "SET NULL",
      "evidence": ["current-db-arch/job-service-node.md#team_members"]
    },
    {
      "id": "m2m_projects_team_members_via_team_member_projects",
      "source": "public.projects",
      "target": "public.team_members",
      "cardinality": "m:n",
      "kind": "conceptual_many_to_many",
      "junctionTable": "team_member_projects",
      "sourceColumn": "id",
      "targetColumn": "id",
      "enforced": false,
      "constraintName": null,
      "onDelete": null,
      "evidence": ["current-db-arch/job-service-node.md#team_member_projects"]
    }
  ]
}
```

---

## 5. Step-by-Step ERD Generation Workflow

When generating an ERD for a service:

### Step 1: Read the Architecture Markdown
Locate `current-db-arch/{SERVICE_NAME}.md`. Read the Service-Level Metrics table to note:
- Total distinct tables
- Active / Legacy / Apparently unused / Uncertain breakdown
- Declared physical foreign key counts
- Isolated tables count
- Total modules

### Step 2: Build the Extraction Script
Write a Node.js script (e.g. `scratch/generate_{SERVICE_NAME}.js`) that:
1. Parses all table definitions and metadata.
2. Extracts all physical foreign keys using the child table's outbound references.
3. Detects junction tables and builds conceptual many-to-many overlay edges.
4. Validates edge endpoints so zero edges point to non-existent nodes.
5. Saves `current-db-arch/erds/{SERVICE_NAME}-erd.json`.

### Step 3: Generate the Standalone HTML
1. Ensure the Cytoscape minified library is available (cached in scratch or fetched from `https://cdnjs.cloudflare.com/ajax/libs/cytoscape/3.28.1/cytoscape.min.js`).
2. Generate `current-db-arch/erds/{SERVICE_NAME}-erd.html` with:
   - Light mode set as default (`data-theme="light"`).
   - Theme toggle button (`🌙 Dark Mode` / `☀️ Light Mode`).
   - Legend placed inside `.canvas-wrapper` (outside sidebar).
   - Right drawer with back/forward navigation and breadcrumb path.
   - Cytoscape and graph data fully inlined.

### Step 4: Validation Checklist
Verify the following before committing:
1. **Table Count Match**: JSON `stats.totalTables` equals the count in the architecture markdown.
2. **Zero Missing Endpoints**: All edge `source` and `target` IDs resolve to existing node IDs.
3. **Zero Dangling Tables**: No undefined or phantom nodes.
4. **All 21 Modules Visible**: Module filter list in sidebar scrolls freely with zero element overlap.
5. **Drawer History**: Clicking through relationships in the drawer updates the breadcrumb and enables the Back button.
6. **Browser Verification**: Open the file in a browser session to confirm visual rendering and console error freedom.

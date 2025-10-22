AGENTS.md — Perceptual (VB) → Nuxt · Supabase · Netlify

Goal

Rebuild the legacy Visual Basic administrative system on a modern stack. Match behavior and flows, not pixel parity. The agent thinks with repo context and then acts.

Ground Rules
	•	No fixed UI scaffolds. The agent proposes screens and only creates what its plan justifies.
	•	Use Nuxt UI for components.
	•	APIs live under Nuxt Server Routes: /server/api/v1/* (Nitro compiles to Netlify functions on deploy).
	•	Database changes are Supabase migrations in supabase/migrations/ and applied via Supabase CLI.
	•	Email is sent via Make.com webhook.

Inputs
	•	VB sources root: /Perceptual (contains .vbp, .frm, .frx, .bas, .cls, and any .mdb, .mdw).
	•	MDB dump root: /nuxt-app/legacy/db/mdb-dump/ (contains *.schema.sql, e.g., database.schema.sql, parameters.schema.sql).

The orchestrator runs in /nuxt-app and treats /Perceptual as an external read‑only source.

Target Architecture
	•	Frontend: Nuxt 3/4 + Nuxt UI + Pinia.
	•	API: Nuxt Server Routes /server/api/v1/* (JWT from Supabase Auth).
	•	DB: Supabase Postgres + RLS.
	•	Auth/RBAC: Derived from legacy PerAdmin.mdw if available.
	•	Integrations: Barcode, Mail (Make webhook), Reporting (SQL + PDF).
	•	Observability: Nitro logs + Supabase logs + custom audit tables.
	•	MCP: Connected to Nuxt and Nuxt UI for code actions the plan requests.

Catalogs produced by thinking (agent writes these)
	•	catalog/db.tables.json — tables, columns, types, PK/FK, indexes.
	•	catalog/db.relations.json — edges, cardinalities, inferred constraints.
	•	catalog/endpoints.plan.json — chosen REST actions per entity.
	•	catalog/views.plan.json — proposed screens, routes, and user flows.

These catalogs are outputs of the plan. They are not required upfront.

Agents (roles)

Agent	Purpose	Inputs	Outputs	Acts On
orchestrator	Think → plan → act loop; prioritizes modules and tasks	repo tree, env, mdb-dump/**/*.schema.sql	task graph, the four catalogs	Repo (files), CLI
discovery	Locate VB/MDB/MDW; confirm dumps present	filesystem	catalog/sources.json	Repo
db-modeler	Unify all *.schema.sql into a coherent model	schema files	catalog/db.tables.json, catalog/db.relations.json	Repo
migration-writer	Emit Supabase migrations from the model	DB catalogs	supabase/migrations/<ts>__schema.sql	Repo
rbac-rls	If PerAdmin.mdw exists, derive roles + RLS	mdw export	supabase/migrations/<ts>__roles.sql, __rls.sql	Repo
endpoint-architect	Decide REST surface	DB catalogs + flows	catalog/endpoints.plan.json, /server/api/v1/* stubs	Repo
ui-designer	Propose screens and interactions (no canned scaffolds)	DB + endpoints + domain	catalog/views.plan.json, PRs/edits under /pages & /components	Repo via MCP
reporting	Reports and exports	domain + SQL	/reports/*, /server/api/reports/*	Repo
observability	Logging and audit	all	/lib/log.ts, audit tables	Repo
task-logger	Append progress logs	events	tasks/backend/*, tasks/views/*	Repo

Execution Flow (single run; repeatable)
	1.	Discover inputs under legacy/db/mdb-dump/. If VB sources appear, add them to context.
	2.	Think & Model DB. Produce catalog/db.tables.json and catalog/db.relations.json from all *.schema.sql files.
	3.	Write Migrations. Generate supabase/migrations/<ts>__schema.sql (and roles/RLS if applicable). Apply with supabase db push.
	4.	Plan Endpoints. Write catalog/endpoints.plan.json, then create only the Nuxt server routes that the plan selects.
	5.	Plan UI. Write catalog/views.plan.json. Using MCP, implement the agreed screens and components. No table→page rule; flows drive decisions.
	6.	Reporting & Integrations. Add Make webhook mail endpoint and initial reports if in scope.
	7.	Log Tasks. Views/features → tasks/views/*. DB/API → tasks/backend/*.
	8.	Iterate per module until coverage targets met (inventory, purchasing, AR/AP, cash, pricing, barcode, mail, reporting).

Files & Folders the agent creates

catalog/
  db.tables.json
  db.relations.json
  endpoints.plan.json
  views.plan.json
server/api/v1/            # only endpoints chosen in endpoints.plan.json
supabase/migrations/      # <ts>__schema.sql, <ts>__roles.sql, <ts>__rls.sql
supabase/seed.sql         # optional if data export is available
pages/                    # only screens chosen in views.plan.json
components/               # UI parts as needed by views
reports/                  # if reporting is in the first increment
tasks/backend/            # BK-*.md logs
tasks/views/              # VW-*.md logs

Security & Policies
	•	Secrets via Netlify/Supabase env. Never in repo.
	•	RLS is mandatory for write endpoints.
	•	Avoid exposing sequential IDs in URLs; optionally use a salt for signed IDs.

Setup (what you provide once)

/Perceptual/                  # legacy VB project (read‑only)
/nuxt-app/                    # working repo (contains AGENTS.md)
  legacy/db/mdb-dump/         # your *.schema.sql files
  supabase/migrations/        # empty; agent will fill
  tasks/{backend,views}/      # agent writes logs

Env (CI or local):

SUPABASE_URL=...
SUPABASE_ANON_KEY=...
SUPABASE_SERVICE_ROLE=...
SUPABASE_PROJECT_REF=...
MAKE_WEBHOOK_URL=...

How to run

You trigger the orchestrator once. It inspects the repo and proceeds through the flow above. No manual code generators.

Acceptance per increment
	•	DB: migrations apply cleanly; tables/indexes match db.tables.json.
	•	API: chosen endpoints exist and pass smoke tests.
	•	UI: screens in views.plan.json open and complete their flows.
	•	Logs: tasks written in the right folders with statuses.

FAQ
	•	Do I need to supply the catalogs? No. The agent writes them after analysis.
	•	What if there are no VB sources? The agent proceeds DB‑first using the dump.
	•	How are emails sent? The API calls MAKE_WEBHOOK_URL with JSON payloads.
	•	Where do ActiveX features go? The agent proposes web-native replacements in views.plan.json and endpoints.plan.json before implementing.

Changelog
	•	2025‑10‑21: Rewritten to agent‑first, catalog‑driven flow.
---
name: index-project
description: Scans a code repository and produces a project overview used to make finding information in the project easier in the future.
---

# Index Project

You are producing a light weight project index. The goal is to get a deep understanding of the structure of the project: folder structure, abstractions, orchestration, programming languages used.

# What you produce

A markdown file `project_index.md` containing a description of the project and a table with all folders/subfolders in the first column, and their contents in the second column. The goal of this file is that if this would be added to an agent's context it will find the correct files to solve a problem faster.

# Step 1

Read the following first:

- README.md or docs/ overview
- Root config files (package.json, pyproject.toml, requirements.txt, docker-compose.yml)
- Top-level directory listing

Get the first impression of the tech stack and the project shape. Don't read the source files yet.

# Step 2

Start scanning subfolders recursively. For each folder, follow these steps to determine its role:

1. List its files and subfolders
2. Look for the following signals to infer the folder's purpose without reading files:
   - **Entry points** (`main.*`, `index.*`, `app.*`, `handler.*`) — read these first if present
   - **Config files** (`*.yml`, `*.toml`, `*.json`) — likely configuration or infrastructure
   - **Test files** (`test_*`, `*_test.*`, `*spec*`) — test layer
   - **Schema / model definitions** (`*.sql`, `schema.yml`, `*.proto`) — data layer
   - **SQL Server project files** (`*.sqlproj`) — SSDT project entry point; read first if present
   - **SQL Server deployment profiles** (`*.publish.xml`, `publish.xml`) — deployment configuration
   - **SQL Server compiled artifacts** (`*.dacpac`, `*.bacpac`) — treat as binary, do not read
3. If the purpose is still not clear from filenames and signals alone, open and skim 1–3 representative files — prefer the smallest non-test source file
4. Once the role is clear, move on — do not read more files than necessary

**Scan depth** — use the following defaults:
- Standard projects: scan up to **3 levels deep**
- Large projects, monorepos, or projects with deep domain structure (e.g. MySQL, large Java/C++ codebases): scan up to **4 levels deep**
- Stop descending into a folder early if it contains only generated, binary, or data files (apply the exclusion list first)

When in doubt, prefer shallower scanning and note in the output that deeper folders were not indexed.

**Exclusion list** — skip the following folders entirely, do not list or describe them in the output:

**General / dependency / build artifacts:**
- `node_modules`
- `.git`
- `__pycache__`
- `.mypy_cache`
- `.pytest_cache`
- `dist`
- `build`
- `out`
- `target`
- `vendor`
- `.venv` / `venv` / `env`
- `.next`
- `.nuxt`
- `coverage`
- `*.egg-info`

**SQL Server database project specific:**
- `bin/` / `obj/` — SSDT/MSBuild output directories (compiled artifacts)
- `.vs/` — Visual Studio local workspace settings
- `DATA/` / `LOG/` — SQL Server runtime data and log file directories
- `BACKUP/` / `Backup/` — database backup files (`.bak`, `.trn`)
- `master/` / `model/` / `msdb/` / `tempdb/` — system database runtime folders (not source scripts)
- `packages/` — NuGet package restore folder (common in SSDT solutions)

**Scan signals specific to SQL Server database projects:**
- `Pre-Deployment/` and `Post-Deployment/` folders under `Scripts/` — always index these; they contain ordered deployment scripts
- `Tables/`, `Views/`, `StoredProcedures/`, `Functions/`, `Schemas/`, `Security/` — standard SSDT object-type folders; index at one level, no need to descend further unless the folder contains subfolders per schema

**MySQL / database project specific:**
- `data/` / `datadir/` — raw MySQL data directory (binary storage files)
- `binlog/` / `relay-log/` — binary and relay log files
- `tmp/` / `tmpdir/` — temporary files created during query execution
- `innodb_temp/` — InnoDB temporary tablespace files
- `undo/` — InnoDB undo tablespace files
- `#innodb_redo/` — InnoDB redo log directory
- `performance_schema/` — internal performance schema database folder
- `mysql/` (when it is a system schema data folder, not source code)
- `sys/` (when it is a system schema data folder, not source code)
- `test/` / `mtr/` — MySQL Test Run (MTR) result and temporary output folders
- `var/` — test suite runtime output (logs, data, pids)
- `*.dSYM/` — macOS debug symbol bundles generated during compilation
- `CMakeFiles/` — CMake internal build cache directories
- `generated/` — auto-generated source or protobuf output folders


# Step 3

Write the file `project_index.md` at the root of the project. This file should contain a header, summary and a table with subfolders and a description of their contents.

If `project_index.md` already exists, update it in place and set the `date-updated` field in the header to today's date. Do not change the `project` field unless the project name has changed.

**For each folder, the contents description should cover:**
- The architectural role (e.g. API layer, domain logic, storage engine, staging models, ETL ingestion)
- The business or technical domain it relates to (e.g. user auth, query parsing, customer orders)
- 1–3 notable files if they are entry points, public interfaces, or particularly important
- Keep descriptions to 1–2 sentences maximum

**Examples:**

| Project type | Path | Contents |
|---|---|---|
| API / backend | `src/api/auth/` | API layer handling user authentication and session management. Key file: `auth_handler.py`. |
| dbt | `models/staging/orders/` | Staging models that clean and recast raw orders data from the source. Key file: `schema.yml`. |
| dbt | `models/marts/finance/` | Finance mart exposing revenue and billing metrics for BI consumption. |
| MySQL source | `storage/innobase/buf/` | InnoDB buffer pool implementation (storage engine infrastructure). Manages in-memory page caching. Key file: `buf0buf.cc`. |
| ETL pipeline | `pipelines/ingest/shopify/` | Ingestion layer pulling raw order and product data from the Shopify API. Key file: `shopify_connector.py`. |
| SQL Server (SSDT) | `Tables/dbo/` | DDL definitions for all tables in the `dbo` schema. One `.sql` file per table. |
| SQL Server (SSDT) | `StoredProcedures/` | Business logic implemented as T-SQL stored procedures, organised by schema. |
| SQL Server (SSDT) | `Scripts/Pre-Deployment/` | Scripts executed before schema deployment, e.g. compatibility guards or drop statements. Key file: `Script.PreDeployment.sql`. |
| SQL Server (SSDT) | `Scripts/Post-Deployment/` | Scripts executed after schema deployment, typically reference data population or permission grants. Key file: `Script.PostDeployment.sql`. |

The actual output table should only have two columns — `Path` and `Contents` — without the `Project type` column.

The file should have the form of:

``` md
---
project: <project-name>
updated: <datetime-updated>
---

# Description

<Description of the project generated in step 1 and refined in step 2.>

# Contents

<table containing subfolders paths and a description of their contents.>
```



---
description: Document a SQL Server 2016 column — orchestrates a scan subtask then a writing subtask to produce a standardized column doc
subtask: false
---

# Document Column — Orchestrator

You are coordinating the full documentation pipeline for a single SQL Server 2016 column.
Your job is to run two subtasks in sequence and verify the final output exists.

## Input

Column to document: **`$ARGUMENTS`**

---

## Step 1 — Read project context

Before delegating anything, read the project index so you can pass accurate context to both subtasks:

@project_index.md

Extract and hold in memory:
- The table name this repo is scoped to
- Where the schema is defined (SSDT `.sqlproj`, Flyway, Liquibase, raw `.sql` scripts, or ORM models)
- What languages and frameworks are used in the application layer
- Whether extended properties (`sp_addextendedproperty`), a dbt layer, YAML catalog, or inline T-SQL comments are the documentation source

---

## Step 2 — Run the scan subtask

Invoke `/document-column-scan $ARGUMENTS` as a subtask.

Wait for it to complete. It will produce:

`docs/columns/<table>/$ARGUMENTS.findings.md`

Verify the file exists before proceeding. If it failed or is empty, report the error and stop.

---

## Step 3 — Run the writing subtask

Invoke `/document-column-write $ARGUMENTS` as a subtask.

Wait for it to complete. It will read the findings file and produce:

`docs/columns/<table>/$ARGUMENTS.md`

---

## Step 4 — Report back

Once both subtasks are done, report:
- ✅ Findings file: `docs/columns/<table>/$ARGUMENTS.findings.md`
- ✅ Column doc: `docs/columns/<table>/$ARGUMENTS.md`
- Any ⚠️ conflicts or gaps flagged by either subtask
- A one-line summary of what the column does (taken from the doc's opening sentence)

Do not modify any source files. Do not write any output yourself — both files are produced by the subtasks.

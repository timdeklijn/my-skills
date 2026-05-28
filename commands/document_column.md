---
description: Document the <TABLE> SQL Server 2016 column — orchestrates a scan subtask then a writing subtask to produce a standardized column doc
subtask: false
---

# Document Column — Orchestrator

You are coordinating the full documentation pipeline for a single SQL Server 2016 column in the `<TABLE>` table.
Your job is to run two subtasks in sequence and verify the final output exists.

## Input

Column to document: **`$ARGUMENTS`**

## Hardcoded context

- **Table**: `<TABLE>`

---

## Step 1 — Read project context

Read the project index for repo conventions — schema tooling, languages, and documentation source:

@project_index.md

The table name is already known (`<TABLE>`). You do not need to derive it from this file.

---

## Step 2 — Run the scan subtask

Invoke `/document-column-scan $ARGUMENTS` as a subtask.
The table name is `<TABLE>` — pass this explicitly to the subtask.

Wait for it to complete. It will produce:

`docs/columns/<TABLE>/$ARGUMENTS.findings.md`

Verify the file exists before proceeding. If it failed or is empty, report the error and stop.

---

## Step 3 — Run the writing subtask

Invoke `/document-column-write $ARGUMENTS` as a subtask.
The table name is `<TABLE>` — pass this explicitly to the subtask.

Wait for it to complete. It will produce:

`docs/columns/<TABLE>/$ARGUMENTS.md`

---

## Step 4 — Report back

Once both subtasks are done, report:
- ✅ Findings file: `docs/columns/<TABLE>/$ARGUMENTS.findings.md`
- ✅ Column doc: `docs/columns/<TABLE>/$ARGUMENTS.md`
- Any ⚠️ conflicts or gaps flagged by either subtask
- A one-line summary of what the column does (taken from the doc's opening sentence)

Do not modify any source files. Do not write any output yourself — both files are produced by the subtasks.
